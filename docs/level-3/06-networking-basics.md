# 06 · Networking Basics

Everything up to this point has run inside one process. Networking is what
lets two separate processes — potentially on different machines — exchange
bytes over a connection. The standard library has no networking API of its
own (that arrives with C++23's `std::net` in some implementations, but isn't
yet universal), so this module uses the **POSIX sockets API** — the C
interface that underlies almost every networking library on Linux and macOS.
On Windows the same concepts apply through **Winsock**, with `WSAStartup()`
required first and header names differing; the ideas below transfer
directly even though the exact function names don't.

## The core idea: a socket is a file descriptor for a connection

A socket is created, configured, and used through a small, consistent set of
calls: `socket()` to create it, then either `connect()` (client) or
`bind()`+`listen()`+`accept()` (server), then `send()`/`recv()` to exchange
bytes, then `close()` — the RAII lesson from
[Module 4](04-raii-deep-dive.md) applies directly here: a raw file
descriptor is exactly the kind of non-RAII resource worth wrapping in a
class that closes it in its destructor.

```cpp
#include <iostream>
#include <unistd.h>

// A minimal RAII wrapper around a POSIX socket file descriptor.
class Socket {
public:
    explicit Socket(int fd) : fd_(fd) {}
    ~Socket() { if (fd_ >= 0) ::close(fd_); }

    Socket(const Socket&) = delete;
    Socket& operator=(const Socket&) = delete;

    Socket(Socket&& other) noexcept : fd_(other.fd_) { other.fd_ = -1; }

    int get() const { return fd_; }

private:
    int fd_;
};

int main() {
    std::cout << "Socket is just an int file descriptor under the hood." << std::endl;
}
// Socket is just an int file descriptor under the hood.
```

## A TCP echo server

The server side: create a socket, bind it to a port, listen for incoming
connections, accept one, and echo back whatever it reads.

```cpp
#include <iostream>
#include <cstring>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <unistd.h>

int runEchoServer(int port) {
    int serverFd = socket(AF_INET, SOCK_STREAM, 0);   // IPv4, TCP
    if (serverFd < 0) { std::cerr << "socket() failed" << std::endl; return 1; }

    int yes = 1;
    setsockopt(serverFd, SOL_SOCKET, SO_REUSEADDR, &yes, sizeof(yes));

    sockaddr_in address{};
    address.sin_family = AF_INET;
    address.sin_addr.s_addr = INADDR_ANY;      // listen on all local interfaces
    address.sin_port = htons(port);            // host-to-network byte order

    if (bind(serverFd, reinterpret_cast<sockaddr*>(&address), sizeof(address)) < 0) {
        std::cerr << "bind() failed" << std::endl;
        return 1;
    }

    listen(serverFd, /*backlog=*/1);
    std::cout << "server listening on port " << port << std::endl;

    sockaddr_in clientAddr{};
    socklen_t clientLen = sizeof(clientAddr);
    int clientFd = accept(serverFd, reinterpret_cast<sockaddr*>(&clientAddr), &clientLen);
    std::cout << "server accepted a connection" << std::endl;

    char buffer[1024];
    ssize_t bytesRead = read(clientFd, buffer, sizeof(buffer) - 1);
    if (bytesRead > 0) {
        buffer[bytesRead] = '\0';
        std::cout << "server received: " << buffer << std::endl;
        write(clientFd, buffer, bytesRead);   // echo it straight back
    }

    close(clientFd);
    close(serverFd);
    return 0;
}
```

Note `htons(port)` — "host to network short." Network byte order is
big-endian regardless of the machine's own endianness, so any multi-byte
value going onto the wire (a port number, an IPv4 address) must be converted
with `htons`/`htonl` when sending and `ntohs`/`ntohl` when receiving. Skipping
this is a classic bug that only shows up cross-platform, since a little-
endian machine talking to itself never notices the mismatch.

## A TCP client

The client side connects to that address and port, sends a message, and
reads the echoed reply:

```cpp
#include <iostream>
#include <cstring>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <unistd.h>

int runEchoClient(int port) {
    int sockFd = socket(AF_INET, SOCK_STREAM, 0);

    sockaddr_in serverAddr{};
    serverAddr.sin_family = AF_INET;
    serverAddr.sin_port = htons(port);
    inet_pton(AF_INET, "127.0.0.1", &serverAddr.sin_addr);   // loopback: same machine

    if (connect(sockFd, reinterpret_cast<sockaddr*>(&serverAddr), sizeof(serverAddr)) < 0) {
        std::cerr << "connect() failed" << std::endl;
        return 1;
    }

    const char* message = "hello from client";
    write(sockFd, message, strlen(message));

    char buffer[1024];
    ssize_t bytesRead = read(sockFd, buffer, sizeof(buffer) - 1);
    if (bytesRead > 0) {
        buffer[bytesRead] = '\0';
        std::cout << "client received echo: " << buffer << std::endl;
    }

    close(sockFd);
    return 0;
}
```

Running the server in one process (or background thread) and the client in
another against `127.0.0.1` (the loopback address — "this same machine")
produces:

```
server listening on port 5555
server accepted a connection
server received: hello from client
client received echo: hello from client
```

`connect()` performs the three-way TCP handshake before returning, so by the
time `write()` runs, the server is guaranteed to already be past `accept()`
— but *starting* the server before the client tries to connect is still the
caller's responsibility; `connect()` fails immediately with "connection
refused" if nothing is listening yet.

## Cheat sheet

| Call | Role | Notes |
|------|------|-------|
| `socket()` | Create an endpoint | `AF_INET` + `SOCK_STREAM` = IPv4 TCP |
| `bind()` | Server: attach to a local port | Needed before `listen()` |
| `listen()` | Server: start accepting connections | Second arg is the pending-connection backlog |
| `accept()` | Server: block until a client connects | Returns a *new* fd for that client |
| `connect()` | Client: initiate a connection | Blocks until the handshake completes or fails |
| `send()`/`write()` | Send bytes | May send fewer bytes than requested — check the return value |
| `recv()`/`read()` | Receive bytes | Returns 0 on a graceful close, negative on error |
| `htons`/`htonl`, `ntohs`/`ntohl` | Byte-order conversion | Required for any multi-byte value on the wire |
| `close()` | Release the socket | Wrap in an RAII type, as shown above |

## Traps

**TCP is a byte stream, not a message stream.** A single `write()` on one
side is not guaranteed to arrive as a single `read()` on the other — a large
message can be split across multiple reads, and small messages can be
coalesced into one. Real protocols prefix each message with a fixed-size
length field (or use a delimiter) so the reader knows where one message ends
and the next begins; this echo example works only because both sides do one
`write` and one `read` and stop.

**Forgetting `htons`/`htonl`** silently works on same-endianness round trips
(a Mac talking to itself) and silently breaks talking to a big-endian
device or a value inspected by a packet sniffer on the wire — always convert
multi-byte fields, even when testing locally makes it look unnecessary.

**Leaking file descriptors.** Every `socket()`/`accept()` call that succeeds
must eventually reach a matching `close()` on every code path, including
error returns — exactly the RAII problem from Module 4, and exactly why the
`Socket` wrapper above exists.

## Exercise

Turn the `Socket` class above into a complete RAII wrapper with `bind()`,
`listen()`, `accept()` (returning another `Socket`), `connect()`, `send()`,
and `recv()` member functions that wrap the raw POSIX calls and throw
`std::runtime_error` on failure. Then extend the echo server to loop —
`accept()` and handle client connections repeatedly instead of exiting after
one — and add a length-prefixed framing scheme (send a 4-byte big-endian
length before the payload, and have the reader loop on `read()` until it has
that many bytes) so a message longer than one `read()` call's buffer arrives
intact.
