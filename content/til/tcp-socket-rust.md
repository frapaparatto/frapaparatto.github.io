---
title: "Sockets and TCP, from system calls to Rust's `TcpListener`"
date: "2026-11-20"
description: "What the kernel does when a Rust program binds, accepts and reads from a TCP socket, seen with strace."
tags:
  - rust
---

I am writing a [key-value store](https://github.com/frapaparatto/kv-store) in Rust while studying that programming language, so I need to build understanding from the ground up: starting from what sockets are under the hood (just enough to understand how Rust applies all of that), some basic TCP notions, and then use them to understand what the Rust standard library provides.

So, let's start by understanding the building blocks.

## Sockets and file descriptors

In Unix, everything is a file. Files, pipes, terminals, network connections: you access all of them the same way, through an integer that identifies the resource inside your process. That integer is the **file descriptor** (fd).

There are OS concepts behind this, like kernel space and user space, but I don't want to go deep on them now. What matters here: your program never touches the network directly. It holds an fd and asks the kernel, through system calls, to do things with the resource behind it.

To get an fd for network communication you use the `socket` system call. It creates an **endpoint** for communication and returns the fd for that endpoint.

```c
int socket(int domain, int type, int protocol);
```

- `domain`: the address family, for example `AF_INET` for IPv4.
- `type`: for example `SOCK_STREAM` for a stream socket.
- `protocol`: usually `0`, the default protocol for that type.

A socket is the endpoint, not the channel. I picture a TCP connection as a tube with a socket at each end. Actually two tubes, one per direction, because a connection carries bytes both ways.

You talk to a socket with `send` and `recv`:

```c
ssize_t send(int sockfd, const void *buf, size_t len, int flags);
ssize_t recv(int sockfd, void *buf, size_t len, int flags);
```

You pass the socket's fd, a buffer, how many bytes, and some flags. `send` copies your bytes into the kernel's send buffer and returns: it doesn't wait for them to arrive. `recv` copies bytes out of the kernel's receive buffer, and if it's empty it blocks until something arrives. It returns `0` when the other side has closed the connection.

There are different kinds of sockets. The ones I need are **stream sockets**: two-way connected communication streams that get their reliability from **TCP**.

## TCP in brief

TCP, the Transmission Control Protocol, is a protocol of the **transport layer**. It delivers a stream of bytes reliably and in order. I don't care about the other layers right now.

TCP gives you a stream of bytes, not messages. There are no boundaries: one `recv` can return half a line or two lines together. Knowing where a message ends is the job of the protocol built on top of it.

### Phases of a TCP connection
A TCP connection has three phases: opening, data transmission and closing.

**Opening: the three-way handshake.**
1. The client sends a **SYN** (synchronize): "I want to connect, and this is my starting sequence number".
2. The server answers with **SYN-ACK**: its own starting number, plus the acknowledgment of the client's.
3. The client sends an **ACK**.

**Data transmission.** TCP splits the stream into chunks called **segments**. The receiver answers with **ACKs**, and an ACK carries the next byte it expects. If a segment goes missing, the sender finds out from repeated ACKs or from a missing ACK, and resends it.

**Closing.** Each side sends a **FIN** to say it has finished sending, and the other side answers with an ACK.

### Data integrity and error detection

**Numbered segments.** Every byte of the stream has a sequence number, and each segment's header carries the number of its first byte. This is what makes the rest work: the receiver knows which bytes arrived, which are missing and in what order they go, and, through its ACKs, the sender learns which ones to send again.

**Error detection.** The sender computes a **checksum**, a 16-bit number calculated from the TCP header, the payload and the IP addresses involved. The receiver computes the same checksum on what arrived and compares them. If they don't match, the segment is corrupted.

## Server side

The client side has its own system call, `connect`. I skipped it for now: today I focused on the server, the side that waits for connections. A client usually doesn't even choose its port: the kernel assigns it one when it connects.

The server needs three more system calls after `socket`.

```c
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
int listen(int sockfd, int backlog);
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
```

**`bind`** associates the socket with an address and a port on my machine. The kernel uses the port to deliver incoming packets to the right socket, and so to the right process.

**`listen`** turns the socket into a **listening socket**. The `backlog` is how many connections can wait in the incoming queue. A connection that completed the handshake waits in this queue until it's accepted.

**`accept`** takes a pending connection out of the queue and returns a **new fd**, just for that connection. The kernel also writes the client's address into the buffer you pass.

So there are two kinds of fd on the server:
- the listening socket's fd, which never carries data. It's only used to accept new connections;
- one new fd per accepted connection. It's like a private tunnel with that client, ready for `send` and `recv`.

Now that I have a mental model of what the kernel provides, let's see how Rust uses it.

## How Rust handles them

I used `strace` to track the syscalls and see what happens from the kernel perspective.

### Creating the listening socket

First, a listening socket in Rust:

```rust
use std::net::TcpListener;

fn main() {
    let listener = TcpListener::bind("127.0.0.1:7878").unwrap();
}
```

`TcpListener` creates a TCP socket server, listening for incoming connections. Under the hood, `bind` calls the following system calls: `socket`, `setsockopt`, `bind`, `listen`.

We can confirm it with `strace` (tracking only the syscalls related to the network or to closing a socket):

```sh
strace -f -e trace=network,close ./target/debug/<binary_file>
```

- `strace` runs a program and prints every system call it makes: the name, the arguments, and after the `=` the value the kernel returned.
- `-f` follows the threads and the child processes too.
- `-e trace=network,close` keeps only the network syscalls and `close`, otherwise the output gets very long.
- I run the binary directly and not with `cargo run`, otherwise I would trace cargo instead of my program.

```sh
close(3)                                = 0
close(3)                                = 0
close(3)                                = 0
close(3)                                = 0
socket(AF_INET, SOCK_STREAM|SOCK_CLOEXEC, IPPROTO_IP) = 3
setsockopt(3, SOL_SOCKET, SO_REUSEADDR, [1], 4) = 0
bind(3, {sa_family=AF_INET, sin_port=htons(7878), sin_addr=inet_addr("127.0.0.1")}, 16) = 0
listen(3, 128)                          = 0
close(3)                                = 0
+++ exited with 0 +++
```

Reading it line by line:
- the first `close(3)` lines happen before my `main` even starts: the program loader and Rust's startup code opening and closing the files they need. I ignore them;
- `socket(...) = 3`: the kernel creates the socket and gives me **fd 3**. From now on, every call uses that number;
- `setsockopt(3, ...) = 0`: on fd 3, turn on `SO_REUSEADDR`, which allows restarting the server on the same port right away;
- `bind(3, ...) = 0`: fd 3 gets address `127.0.0.1` and port `7878`. `htons` means the port is converted to network byte order;
- `listen(3, 128) = 0`: fd 3 is now a listening socket, with a queue of up to 128 connections;
- `close(3) = 0`: `main` ends and the fd is closed. I never wrote `close` in my code, I'll come back to this.

These calls return `0` or a new fd on success. On failure the kernel returns `-1`, and Rust turns it into an error.

The listener is ready, now I need a client.

### Waiting for a client

I added a call to `accept()`:

```rust
use std::net::TcpListener;

fn main() {
    let listener = TcpListener::bind("127.0.0.1:7878").unwrap();

    let (stream, addr) = listener.accept().unwrap();
    println!("connection from {addr}");
}
```

`accept()` returns a tuple containing a `TcpStream` and a `SocketAddr`. I want to focus on the stream, because `TcpListener` and `TcpStream` are different things: the first is a socket that listens for incoming connections, not for data. The second is the established connection, with its own new file descriptor, and you can read and write on it.

Under the hood, `accept()` calls `accept4` (the Linux version of `accept`) on the listening fd, waits until a connection is in the queue, and wraps the new fd into a `TcpStream`.

I ran `strace` again and connected from another terminal with `nc` (netcat), a small command line tool that opens a TCP connection and connects it to the terminal:

```sh
nc localhost 7878
```

Before `nc` connects, the last line is incomplete:

```sh
accept4(3
```

The call hasn't returned yet: the program is blocked inside the kernel, waiting. As soon as `nc` connects, the line is completed:

```sh
...
listen(3, 128)                          = 0
accept4(3, {sa_family=AF_INET, sin_port=htons(34694), sin_addr=inet_addr("127.0.0.1")}, [128 => 16], SOCK_CLOEXEC) = 4
close(4)                                = 0
close(3)                                = 0
+++ exited with 0 +++
```

The `...` hides the lines I already explained. The new one is `accept4`:
- `accept4(3, ...)`: the call is made on fd 3, the listening socket;
- inside the braces there's the client's address: `127.0.0.1`, port `34694`. Not 7878: it's the port the kernel assigned to `nc` when it connected;
- `[128 => 16]`: Rust told the kernel "you have 128 bytes of space for the address", and the kernel answered "I used 16". That's the size of an IPv4 address;
- `= 4`: the new fd. **fd 4 is the connection**, the private tunnel with this client. fd 3 keeps listening.

Then `main` ends: `close(4)`, `close(3)`, and the program exits with code 0.

One thing I noticed while trying this: the client can connect even before the server calls `accept`. I made my program wait before `accept`, opened two `nc` in other terminals, and both connected. With `ss -tln` (it lists the listening sockets) the `Recv-Q` column of my port went up by one for each `nc`. The handshake is done by the kernel: the connections just wait in the queue. `accept` only takes them out.

I have a connection, now I want to send something through it.

### Talking through the connection

I extended the program to read from the connection and write back what it receives:

```rust
use std::io::{Read, Write};
use std::net::TcpListener;

fn main() {
    let listener = TcpListener::bind("127.0.0.1:7878").unwrap();

    let (mut stream, addr) = listener.accept().unwrap();
    println!("connection from {addr}");

    let mut buf = [0u8; 1024];
    let n = stream.read(&mut buf).unwrap();
    stream.write_all(&buf[..n]).unwrap();
}
```

The first line brings the reading and writing operations into scope. Then:
- `buf` is a buffer of 1024 bytes;
- `read` waits for the client to send something, copies it into `buf`, and returns `n`, how many bytes arrived;
- `write_all` sends back exactly those bytes: `&buf[..n]` means "the first `n` bytes of `buf`", not the whole buffer.

So the server sends back what it receives: an echo.

I typed `hello world` in `nc` and pressed Enter:

```sh
...
accept4(3, {sa_family=AF_INET, sin_port=htons(51238), sin_addr=inet_addr("127.0.0.1")}, [128 => 16], SOCK_CLOEXEC) = 4
recvfrom(4, "hello world\n", 1024, 0, NULL, NULL) = 12
sendto(4, "hello world\n", 12, MSG_NOSIGNAL, NULL, 0) = 12
close(4)                                = 0
close(3)                                = 0
+++ exited with 0 +++
```

Two new lines, after `accept4`:
- `recvfrom(4, "hello world\n", 1024, ...) = 12`: on fd 4, read up to 1024 bytes. 12 arrived: `hello world` plus the newline;
- `sendto(4, "hello world\n", 12, MSG_NOSIGNAL, ...) = 12`: on fd 4, send those 12 bytes back. `MSG_NOSIGNAL` is a flag Rust always adds: if the client has already gone away, the call returns an error instead of killing the program.

And in the `nc` terminal, the line comes back:

```sh
hello world
hello world
```

The first one is what I typed, the second one is the echo from the server.

The names are different from `recv` and `send`, but they're the same calls: on my machine (and on x86-64 and arm64 in general), Linux has no separate `recv` and `send` syscalls, libc implements them with `recvfrom` and `sendto`. And the fd is always 4: data travels on the connection, never on the listening socket.

### Closing: who owns the fd

The last lines of the same output are `close(4)` and `close(3)`. I never wrote `close` in my code, Rust does it for me.

In Rust every value has an **owner**, the variable that holds it. When the owner goes out of scope (here, at the end of `main`), the value is **dropped**, and for a `TcpStream` or a `TcpListener` being dropped means closing its fd. The order is the reverse of the declarations: `stream` was created after `listener`, so fd 4 is closed first, then fd 3.

On the other side, `nc` sees that the server has closed the connection.

In C you have to remember to call `close` yourself, and if you forget, the fd stays open. In Rust the `TcpStream` owns the fd, so the fd lives exactly as long as the `TcpStream`.

### Accepting forever: `incoming()`

A server that handles one client and exits is not very useful. `incoming()` is `accept` called in a loop: each turn gives a new connection.

```rust
use std::io::{Read, Write};
use std::net::TcpListener;

fn main() {
    let listener = TcpListener::bind("127.0.0.1:7878").unwrap();

    for stream in listener.incoming() {
        let mut stream = stream.unwrap();

        let mut buf = [0u8; 1024];
        let n = stream.read(&mut buf).unwrap();
        stream.write_all(&buf[..n]).unwrap();
    }
}
```

It never stops on its own: the loop ends only if the program breaks out of it or is killed. Each `stream` is dropped at the end of its turn, so in `strace` the same cycle repeats for every `nc`: `accept4`, `recvfrom`, `sendto`, `close(4)`. And the new connection often gets fd 4 again, because the kernel reuses the lowest free number.

The clients are handled one at a time: while the server is busy with one, the others connect anyway and wait in the queue.

## Resources

- [Beej's Guide to Network Programming](https://beej.us/guide/bgnet/)
- [RFC 9293: Transmission Control Protocol](https://www.rfc-editor.org/rfc/rfc9293)
- [Internet protocol suite (Wikipedia)](https://en.wikipedia.org/wiki/Internet_protocol_suite)
- Man pages: [`socket(2)`](https://man7.org/linux/man-pages/man2/socket.2.html), [`bind(2)`](https://man7.org/linux/man-pages/man2/bind.2.html), [`listen(2)`](https://man7.org/linux/man-pages/man2/listen.2.html), [`accept(2)`](https://man7.org/linux/man-pages/man2/accept.2.html), [`send(2)`](https://man7.org/linux/man-pages/man2/send.2.html), [`recv(2)`](https://man7.org/linux/man-pages/man2/recv.2.html), [`ss(8)`](https://man7.org/linux/man-pages/man8/ss.8.html), [`strace(1)`](https://man7.org/linux/man-pages/man1/strace.1.html)
- Rust: [`TcpListener`](https://doc.rust-lang.org/std/net/struct.TcpListener.html), [`TcpStream`](https://doc.rust-lang.org/std/net/struct.TcpStream.html), [Building a Single-Threaded Web Server](https://doc.rust-lang.org/book/ch21-01-single-threaded.html)
