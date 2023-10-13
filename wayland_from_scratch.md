<link rel="stylesheet" type="text/css" href="main.css">
<link rel="stylesheet" href="https://unpkg.com/@highlightjs/cdn-assets@11.8.0/styles/default.min.css">
<script src="https://unpkg.com/@highlightjs/cdn-assets@11.8.0/highlight.min.js"></script>
<script src="https://unpkg.com/@highlightjs/cdn-assets@11.8.0/languages/x86asm.min.js"></script>
<script>
window.addEventListener("load", (event) => {
  hljs.highlightAll();
});
</script>

<a href="/blog">All articles</a>

# Learn Wayland by writing a GUI from scratch

[Wayland](https://wayland.freedesktop.org/) is all the rage those days. Distributions left and right switch to it, many readers of my previous article on [writing a X11 GUI from scratch in x86_64 assembly](/blog/x11_x64.html) asked for a follow-up article about Wayland, and I now run Waland on my desktop. So here we go, let's write a (very simple) GUI program with Wayland, without any libraries, this time in C. 

Here is what we are working towards:


![Result](wayland-screenshot-tiled1.png)

We display the Wayland logo in its own window (we can see the mountain wallpaper in the background since we use a fixed size buffer). It's not quite Visual Studio yet, I know, but it's a good fundation for more in future articles, perhaps.

Why not in assembly again you ask? Well, the Wayland protocol has some peculiarities that necessitate the use of some C standard library macros to make it work reliably on different platforms (Linux, FreeBSD, etc): namely, sending a file descriptor over a UNIX socket. Maybe it could be done in assembly, but it would be much more tedious. Also, the Wayland protocol is completely asynchronous by nature, whereas the X11 protocol was more of a request-(maybe) response chatter, and as such, we have to keep track of some state in our program, and C makes it easier.

Now, if you want to follow along and translate the C snippets into assembly, go for it, it is doable, just tedious.


> If you spot an error, please open a [Github issue](https://github.com/gaultier/blog)!

**Table of Contents**

## What do we need?

Not much: We'll use C99 so any C compiler of the last 20 years will do. Having a Wayland desktop to test the application will also greatly help.

Note that I have only run it on Linux; it should work (meaning: compile and run) on other platforms running Wayland such as FreeBSD, it's just that I have not tried.

*Note that the code in this article has not been written in the most robust way, it simply exits when things are not how they should be for example. So, not production ready, but still a good learnign resource and a good fundation for more.*

## Wayland basics

Wayland is a protocol specification for GUI applications (and more), in short. We will write the client side, while the server side is a compositor which understands our protocol. If you have a Wayland desktop right now, a Wayland compositor is already running so there is nothing to do.

Much like X11, a client opens a UNIX socket, sends some commands in a specific format (which are different from the X11 ones), to open a window and the server can also send messages to notify the client to resize the window, that there is some keyboard input, etc. It's important to note that contrary to X11, in Wayland, the client only has access to its own window.

It is also interesting to note that Wayland is quite a limited protocol and any GUI will have to use extension protocols.


Most client applications use `libwayland` which is a library composed of C files that are autogenerated from a XML file describing the protocol.
The same goes for extension protocols: they simply are one XML file that is turned into C files, which are then compiled and linked to a GUI application.

Now, we will not do any of this: we will instead write our own serialization and deserialization functions, which is really not a lot of work as you will see.

There are many advantages:
- No need to link to external libraries: no build system complexities, no dynamic linking issues, and so on.
- We do not have to use the callback system that `libwayland` requires.
- We can use the polling mechanism we wish to listen to incoming messages: blocking, `poll`, `select`, `epoll`, `io_uring`, `kqueue` on some systems, etc. Here, we will use blocking calls for simplicity but the world is your oyster.
- Easy troubleshooting: 100% of the code is our own.
- No XML
- The protocols we will use are stable so the numeric values on the wire should not change underneath us, but in the event they do, we simply have to fix them in our code and compile again.


So at this point you might be thinking: this is going to be so much work! Well, not really. Here are **all** of the Wayland protocol numeric values we will need, including the extension protocols:

```c
static const uint32_t wayland_display_object_id = 1;
static const uint16_t wayland_wl_display_event_delete_id = 1;
static const uint16_t wayland_wl_registry_event_global = 0;
static const uint16_t wayland_shm_pool_event_format = 0;
static const uint16_t wayland_wl_buffer_event_release = 0;
static const uint16_t wayland_xdg_wm_base_event_ping = 0;
static const uint16_t wayland_xdg_toplevel_event_configure = 0;
static const uint16_t wayland_xdg_toplevel_event_close = 1;
static const uint16_t wayland_xdg_surface_event_configure = 0;
static const uint16_t wayland_wl_display_get_registry_opcode = 1;
static const uint16_t wayland_wl_registry_bind_opcode = 0;
static const uint16_t wayland_wl_compositor_create_surface_opcode = 0;
static const uint16_t wayland_xdg_wm_base_pong_opcode = 3;
static const uint16_t wayland_xdg_surface_ack_configure_opcode = 4;
static const uint16_t wayland_wl_shm_create_pool_opcode = 0;
static const uint16_t wayland_xdg_wm_base_get_xdg_surface_opcode = 2;
static const uint16_t wayland_wl_shm_pool_create_buffer_opcode = 0;
static const uint16_t wayland_wl_surface_attach_opcode = 1;
static const uint16_t wayland_xdg_surface_get_toplevel_opcode = 1;
static const uint16_t wayland_wl_surface_commit_opcode = 6;
static const uint32_t wayland_format_xrgb8888 = 1;
static const uint32_t wayland_header_size = 8;
static const uint32_t color_channels = 4;
```

So, not that much!


## Opening a socket

The first step is opening a UNIX domain socket. Note that this step is exactly the same as for X11, save for the path of the socket. Also, X11 is designed to be used over the network so it does not have to be a UNIX domain socket, on the same machine - but everybody does so on their desktop machine anyway.

To craft the socket path, we follow these simple steps:

- If `$WAYLAND_DISPLAY` is set, attempt to connect to `$XDG_RUNTIME_DIR/$WAYLAND_DISPLAY`
- Otherwise, attempt to connect to `$XDG_RUNTIME_DIR/wayland-0`
- Otherwise, fail

Here goes:

```c
static int wayland_display_connect() {
  char *xdg_runtime_dir = getenv("XDG_RUNTIME_DIR");
  if (xdg_runtime_dir == NULL)
    return EINVAL;

  uint64_t xdg_runtime_dir_len = strlen(xdg_runtime_dir);

  struct sockaddr_un addr = {.sun_family = AF_UNIX};
  assert(xdg_runtime_dir_len <= cstring_len(addr.sun_path));
  uint64_t socket_path_len = 0;

  memcpy(addr.sun_path, xdg_runtime_dir, xdg_runtime_dir_len);
  socket_path_len += xdg_runtime_dir_len;

  addr.sun_path[socket_path_len++] = '/';

  char *wayland_display = getenv("WAYLAND_DISPLAY");
  if (wayland_display == NULL) {
    char wayland_display_default[] = "wayland-0";
    uint64_t wayland_display_default_len = cstring_len(wayland_display_default);

    memcpy(addr.sun_path + socket_path_len, wayland_display_default,
           wayland_display_default_len);
    socket_path_len += wayland_display_default_len;
  } else {
    uint64_t wayland_display_len = strlen(wayland_display);
    memcpy(addr.sun_path + socket_path_len, wayland_display,
           wayland_display_len);
    socket_path_len += wayland_display_len;
  }

  int fd = socket(AF_UNIX, SOCK_STREAM, 0);
  if (fd == -1)
    exit(errno);

  if (connect(fd, (struct sockaddr *)&addr, sizeof(addr)) == -1)
    exit(errno);

  return fd;
}
```


In Wayland, there is no connection setup to do, such as sending some special messages, so there is nothing more to do.

## Creating a registry

Now, to do anything useful, we want to create a registry: it is an object that allows us to query at runtime the capabilities of the compositor.

In Wayland, to create an object, we simply send the right message with an id of our own. Ids should be unique so we simply increment a number each time we want to create a new resource. After this is done, we will remember this number to be able to refer to it in later messages:

This is coincidentally our first message we send, so let's briefly go over the structure of a Wayland message. It is basically a RPC mechanism. All bytes are in the host endianness so there is nothing special to do about it:

- 4 bytes containing the id of the resource ('object') we want to call a method on
- 2 bytes containing the opcode of the method we want to call
- 2 bytes containing the size of the message
- Depending on the method, arguments in their wire format follow

The object id is `1`, which is the singleton `wl_display` that already exists.
The method is: `get_registry(u32 new_id)` whose opcode we listed before.
The sole argument takes 4 bytes and is this incremental number we keep track of client-side.
It does not necessarily have to be incremental, but that's what `libwayland` does and also it's the easiest. 

For convenience and efficiency, we craft the message on the stack and do not allocate dynamic memory.


We first introduce a few utility functions to read and write parts of messages:

```c
static void buf_write_u32(char *buf, uint64_t *buf_size, uint64_t buf_cap,
                          uint32_t x) {
  assert(*buf_size + sizeof(x) <= buf_cap);
  assert(((size_t)buf + *buf_size) % sizeof(x) == 0);

  *(uint32_t *)(buf + *buf_size) = x;
  *buf_size += sizeof(x);
}

static void buf_write_u16(char *buf, uint64_t *buf_size, uint64_t buf_cap,
                          uint16_t x) {
  assert(*buf_size + sizeof(x) <= buf_cap);
  assert(((size_t)buf + *buf_size) % sizeof(x) == 0);

  *(uint16_t *)(buf + *buf_size) = x;
  *buf_size += sizeof(x);
}

static void buf_write_string(char *buf, uint64_t *buf_size, uint64_t buf_cap,
                             char *src, uint32_t src_len) {
  assert(*buf_size + src_len <= buf_cap);

  buf_write_u32(buf, buf_size, buf_cap, src_len);
  memcpy(buf + *buf_size, src, roundup_4(src_len));
  *buf_size += roundup_4(src_len);
}

static uint32_t buf_read_u32(char **buf, uint64_t *buf_size) {
  assert(*buf_size >= sizeof(uint32_t));
  assert((size_t)*buf % sizeof(uint32_t) == 0);

  uint32_t res = *(uint32_t *)(*buf);
  *buf += sizeof(res);
  *buf_size -= sizeof(res);

  return res;
}

static uint16_t buf_read_u16(char **buf, uint64_t *buf_size) {
  assert(*buf_size >= sizeof(uint16_t));
  assert((size_t)*buf % sizeof(uint16_t) == 0);

  uint16_t res = *(uint16_t *)(*buf);
  *buf += sizeof(res);
  *buf_size -= sizeof(res);

  return res;
}

static void buf_read_n(char **buf, uint64_t *buf_size, char *dst, uint64_t n) {
  assert(*buf_size >= n);

  memcpy(dst, *buf, n);

  *buf += n;
  *buf_size -= n;
}
```

And we finally can send our first message:

```c
static uint32_t wayland_wl_display_get_registry(int fd) {
  uint64_t msg_size = 0;
  char msg[128] = "";
  buf_write_u32(msg, &msg_size, sizeof(msg), wayland_display_object_id);

  buf_write_u16(msg, &msg_size, sizeof(msg),
                wayland_wl_display_get_registry_opcode);

  uint16_t msg_announced_size =
      wayland_header_size + sizeof(wayland_current_id);
  assert(roundup_4(msg_announced_size) == msg_announced_size);
  buf_write_u16(msg, &msg_size, sizeof(msg), msg_announced_size);

  wayland_current_id++;
  buf_write_u32(msg, &msg_size, sizeof(msg), wayland_current_id);

  if ((int64_t)msg_size != send(fd, msg, msg_size, MSG_DONTWAIT))
    exit(errno);

  printf("-> wl_display@%u.get_registry: wl_registry=%u\n",
         wayland_display_object_id, wayland_current_id);

  return wayland_current_id;
}
```


And by calling it, we have created our very first Wayland resource!


## Shared memory: the frame buffer

To avoid drawing  a frame in our application, and having to send all of the bytes over the socket to the compositor, there is smarter approach: the buffer should be shared between the two processes, so that no copying is required.

We need to synchronize the access between the two so that presenting the frame does not happen while we are still drawing it, and Wayland has that built-in.

First, we need to create this buffer. We are going to make it easier for us by using a fixed size. Wayland is going to send us 'resize' events, which we will acknowledge but without changing a thing. This is done here just to simplify a bit the article, obviously in a real application, you would resize the buffer.

First, we introduce a struct that will hold all of the client-side state so that we remember which resources have created so far. We also need a super simple state machine for later to track whether the surface (i.e. the 'frame' data) should be drawn to, as mentioned:

```c
typedef enum state_state_t state_state_t;
enum state_state_t {
  STATE_NONE,
  STATE_SURFACE_ACKED_CONFIGURE,
  STATE_SURFACE_ATTACHED,
};

typedef struct state_t state_t;
struct state_t {
  uint32_t wl_registry;
  uint32_t wl_shm;
  uint32_t wl_shm_pool;
  uint32_t wl_buffer;
  uint32_t xdg_wm_base;
  uint32_t xdg_surface;
  uint32_t wl_compositor;
  uint32_t wl_surface;
  uint32_t xdg_toplevel;
  uint32_t stride;
  uint32_t w;
  uint32_t h;
  uint32_t shm_pool_size;
  int shm_fd;
  uint8_t *shm_pool_data;

  state_state_t state;
};
```

We use it so in `main()`:


```c
  state_t state = {
      .wl_registry = wayland_wl_display_get_registry(fd),
      .w = 117,
      .h = 150,
      .stride = 117 * color_channels,
  };

  // Single buffering.
  state.shm_pool_size = state.h * state.stride;
```

The window is a rectangle, of width `w` and height `h`. We will use the color format `xrgb` which is 4 color channels, each taking one bytes, so 4 bytes per pixel. This is one of the two formats that is guaranteed to be supported by the compositor per the specification. The stride counts how many bytes is a horizontal row: `w * 4`.

And so, our buffer size for the frame is : `w * h * 4`. We use single buffering again for simplicity and also because we want to display a static image. 

We could choose to use double or even triple buffering, thus respectively doubling or tripling the buffer size. The compositor is none the wiser - we would simply keep a counter client-side that increments each time we render a frame (and wraps around back to 0 when reaching the number of buffers), and we would draw in the right location of this big buffer (i.e. at an offset). All the Wayland calls would still remain the same.

Alright, time to really create this buffer, and not only keep track of its size:

```c
static void create_shared_memory_file(uint64_t size, state_t *state) {
  char name[255] = "/";
  for (uint64_t i = 1; i < cstring_len(name); i++) {
    name[i] = ((double)rand()) / (double)RAND_MAX * 26 + 'a';
  }

  int fd = shm_open(name, O_RDWR | O_EXCL | O_CREAT, 0600);
  if (fd == -1)
    exit(errno);

  assert(shm_unlink(name) != -1);

  if (ftruncate(fd, size) == -1)
    exit(errno);

  state->shm_pool_data =
      mmap(NULL, size, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
  assert(state->shm_pool_data != NULL);
  state->shm_fd = fd;
}
```

We use `shm_open(3)` to create a POSIX shared memory object, so that we later can send the corresponding file descriptor to the compositor so that the latter also has access to it. The flags mean:

- `O_RDWR`: Read-write.
- `O_CREAT`: If the file does not exist, create it.
- `O_EXCL`: Return an error if the shared memory object with this name already exists (we do not want that another running instance of the application gets by mistake the same memory buffer).

We alternatively could use `memfd_create(2)` which spares us from crafting a unique path but this is Linux specific.

We craft a unique, random path to avoid clashes with other running applications.

Right after, we removed the file on the filesystem with `shm_unlink` to not leave any traces when the program finishes. Note that the file descriptor remains valid since our process still has the file open (there is a reference counting mechanism in the kernel behind the scenes).

We then resize with `ftruncate` and memory map this file with `mmap(2)`, effectively allocating memory, with the `MAP_SHARED` flag to allow the compositor to also read this memory.

Alright, we now have some memory to draw our frame to, but the compositor does not know of it yet. Let's tackle that now.

## Chatting with the compositor


We are going to exchange messages back and forth over the socket with the compositor. Let's use plain old blocking calls in `main` like it's the 70's:

```c
  while (1) {
    char read_buf[4096] = "";
    int64_t read_bytes = recv(fd, read_buf, sizeof(read_buf), 0);
    if (read_bytes == -1)
      exit(errno);

    char *msg = read_buf;
    uint64_t msg_len = (uint64_t)read_bytes;

    while (msg_len > 0)
      wayland_handle_message(fd, &state, &msg, &msg_len);
    }
  }
```

And `wayland_handle_message` reads the header part of every message as described in the beginning, and reacts to known opcodes:


```c
static void wayland_handle_message(int fd, state_t *state, char **msg,
                                   uint64_t *msg_len) {
  assert(*msg_len >= 8);

  uint32_t object_id = buf_read_u32(msg, msg_len);
  assert(object_id <= wayland_current_id);

  uint16_t opcode = buf_read_u16(msg, msg_len);

  uint16_t announced_size = buf_read_u16(msg, msg_len);
  assert(roundup_4(announced_size) <= announced_size);

  uint32_t header_size =
      sizeof(object_id) + sizeof(opcode) + sizeof(announced_size);
  assert(announced_size <= header_size + *msg_len);

  if (object_id == state->wl_registry &&
      opcode == wayland_wl_registry_event_global) {
      // TODO
  }
  // Following: Lots of `if (opcode == ...) {... } else if (opcode = ...) { ... } [...]`
}
```