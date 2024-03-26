# Network Programming

* We can write programs that communicate with each other via network programming.
* There are many good resources on the Internet, but [Beej's Guide to Network
  Programming](https://beej.us/guide/bgnet/) has traditionally been popular.
* Our recommended textbook, The Linux Programming Interface, is also a great resource to get more
  details.

## Basics of the Networking Stack

* Layers: Phy, link (MAC), IP, Transport (TCP/UDP), then applications
* Phy: Hardware control.
    * Generate and receive signals.
        * Need to know how to physically send and receive something.
    * Think Amazon package delivery: need a car and a driver.
* Link (MAC): Local network addressing and routing.
    * Think Amazon package delivery again: need an address and routes (how to get there).
    * This is only for a (small area) local network.
    * E.g., wired local network and wireless local network.
    * This layer uses MAC addresses for addressing.
* IP: Inter-network addressing and routing.
    * What if you want to connect a wired local network and a wireless local network?
    * Still need addressing and routing but it needs to be something common for both wired and
      wireless.
    * This layer uses IP addresses for addressing.
* Transport: What do you do when you have a lot of packages that keep coming and going?
    * If you don't do anything, three problems can occur.
        * Packages can be lost. (Think car crash. Or human errors like losing a package in a
          warehouse.)
        * Packages can arrive out of order. (They may be delivered by different trucks via different
          routes.)
        * Packages can be duplicated. (If the sender mistakenly thinks the package is lost and
          re-sends.)
    * Need a way to control these things.
    * TCP (Transmission Control Protocol) provides protection against these things. No loss, no out
      of order, or duplication.
    * UDP (User Datagram Protocol) doesn't provide any protection.
* HTTP is an app-layer protocol.

## The Socket Interface

* There are five key syscalls
    * `socket()`
    * `bind()`
    * `listen()`
    * `accept()`
    * `connect()`
* Will examine these.

## Socket

* `int socket(int domain, int type, int protocol)`
    * An application can use a socket to communicate with another process (local or remote).
    * Returns a file descriptor.
    * You can use this like a file (read/write).
    * But there are socket-specific calls exist, e.g., send, recv, sendto, recvfrom.
    * `man socket`
    * We need to understand what these parameters mean first.
* Domain (`int domain`)
    * There are many communication protocols and `domain` specifies one to use.
    * What is a protocol?
        * A protocol defines a set of rules that an entity needs to follow to communicate with
          another entity (that follows the same set of rules).
    * Domain examples
        * AF_UNIX: Local communication
        * AF_INET: IPv4 Internet protocols
        * AF_INET6: IPv6 Internet protocols
* Type (`int type`)
    * SOCK_STREAM: TCP. Provides sequenced, reliable, two-way, connection-based byte streams.
        * Connection-based or connection-oriented: will explain later
    * SOCK_DGRAM: UDP. Supports datagrams (connectionless, unreliable messages of a fixed maximum
      length).
        * Connectionless: will explain later.
* Protocol (`int protocol`)
    * Some domains allow different protocols.
    * Not used for common domains, AF_UNIX, AF_INET, and AF_INET6, hence 0.

## Stream Socket Sequence (TCP)

```bash
Server                  Client
------                  ------
socket
bind
listen
accept
                        socket
      <---------------- connect
(accept returns)

read  <---------------- write
write ----------------> read

close                   close
```

* `socket()` creates a socket.
* `bind()` binds the socket to an address.
    * Uses a generic address `struct`.

      ```c
       struct sockaddr {
         sa_family_t sa_family;
         char        sa_data[14]; // size varies. bind requires addrlen.
       }
      ```

    * Different protocols use different `struct`s (with different-yet-similar names).
* `listen()` marks the socket as *passive*, i.e., it's used to wait for a connection to come
  (typically used by a server). By default, a socket is *active*.
* `accept()` accepts a new connection.
    * Returns a *new socket* to use for the new connection.
    * The original socket is only used to accept new connections.
* `connect()` connects to a passive socket.
* "connection-oriented" means we establish a connection first.

## Datagram Socket Sequence (UDP)

```bash
Server                  Client
------                  ------
socket
bind
                        socket
recvfrom <------------- sendto
sendto   -------------> recvfrom

close                   close
```

* No active or passive socket.
* `sendto` needs to specify the receiver's address every time.
* "connectionless" means we don't establish a connection first.

## Activity

* Create two programs, server and client. Implement the socket sequence using `AF_UNIX`. The client
  should be able to send messages typed on the terminal to the server. The server should be able to
  print out the messages.
    * `man unix` for detailed info for `AF_UNIX`.
    * An `AF_UNIX` address uses `struct sockaddr_un`.

      ```c
      struct sockaddr_un {
        sa_family_t sun_family;               /* AF_UNIX */
        char        sun_path[108];            /* Pathname */
      };
      ```

    * Do this for both TCP and UDP.
* A stream server example.

  ```c
  #include <errno.h>
  #include <stdio.h>
  #include <stdlib.h>
  #include <string.h>
  #include <sys/socket.h>
  #include <sys/un.h>
  #include <unistd.h>

  #define BUF_SIZE 64
  #define MY_SOCK_PATH "tmp"
  #define LISTEN_BACKLOG 32

  #define handle_error(msg)                                                      \
    do {                                                                         \
      perror(msg);                                                               \
      exit(EXIT_FAILURE);                                                        \
    } while (0)

  int main() {
    struct sockaddr_un addr;
    int sfd, cfd;
    ssize_t num_read;
    char buf[BUF_SIZE];

    sfd = socket(AF_UNIX, SOCK_STREAM, 0);
    if (sfd == -1)
      handle_error("socket");

    if (remove(MY_SOCK_PATH) == -1 &&
        errno != ENOENT) // No such file or directory
      handle_error("remove");

    memset(&addr, 0, sizeof(struct sockaddr_un));
    addr.sun_family = AF_UNIX;
    strncpy(addr.sun_path, MY_SOCK_PATH, sizeof(addr.sun_path) - 1);

    if (bind(sfd, (struct sockaddr *)&addr, sizeof(struct sockaddr_un)) == -1)
      handle_error("bind");

    if (listen(sfd, LISTEN_BACKLOG) == -1)
      handle_error("listen");

    for (;;) {
      cfd = accept(sfd, NULL, NULL);
      if (cfd == -1)
        handle_error("accept");

      while ((num_read = read(cfd, buf, BUF_SIZE)) > 0) {
        if (write(STDOUT_FILENO, buf, num_read) != num_read)
          handle_error("write");

        if (num_read == -1)
          handle_error("read");
      }
    }

    if (close(cfd) == -1)
      handle_error("close");
  }
  ```

* A stream client example.

  ```c
  #include <stdio.h>
  #include <stdlib.h>
  #include <string.h>
  #include <sys/socket.h>
  #include <sys/un.h>
  #include <unistd.h>

  #define MY_SOCK_PATH "tmp"
  #define BUF_SIZE 64

  #define handle_error(msg)                                                      \
    do {                                                                         \
      perror(msg);                                                               \
      exit(EXIT_FAILURE);                                                        \
    } while (0)

  int main() {
    struct sockaddr_un addr;
    int sfd;
    ssize_t num_read;
    char buf[BUF_SIZE];

    sfd = socket(AF_UNIX, SOCK_STREAM, 0);
    if (sfd == -1)
      handle_error("socket");

    memset(&addr, 0, sizeof(struct sockaddr_un));
    addr.sun_family = AF_UNIX;
    strncpy(addr.sun_path, MY_SOCK_PATH, sizeof(addr.sun_path) - 1);

    if (connect(sfd, (struct sockaddr *)&addr, sizeof(struct sockaddr_un)) == -1)
      handle_error("connect");

    while ((num_read = read(STDIN_FILENO, buf, BUF_SIZE)) > 0)
      if (write(sfd, buf, num_read) != num_read)
        handle_error("write");

    if (num_read == -1)
      handle_error("read");

    exit(EXIT_SUCCESS);
  }
  ```

* A datagram server example.

  ```c
  #include <errno.h>
  #include <stdio.h>
  #include <stdlib.h>
  #include <string.h>
  #include <sys/socket.h>
  #include <sys/un.h>
  #include <unistd.h>

  #define BUF_SIZE 64
  #define MY_SOCK_PATH "tmp"
  #define LISTEN_BACKLOG 32

  #define handle_error(msg)                                                      \
    do {                                                                         \
      perror(msg);                                                               \
      exit(EXIT_FAILURE);                                                        \
    } while (0)

  int main() {
    struct sockaddr_un addr;
    int sfd;
    ssize_t num_read;
    char buf[BUF_SIZE];
    socklen_t len = sizeof(struct sockaddr_un);

    sfd = socket(AF_UNIX, SOCK_DGRAM, 0);
    if (sfd == -1)
      handle_error("socket");

    if (remove(MY_SOCK_PATH) == -1 &&
        errno != ENOENT) // No such file or directory
      handle_error("remove");

    memset(&addr, 0, sizeof(struct sockaddr_un));
    addr.sun_family = AF_UNIX;
    strncpy(addr.sun_path, MY_SOCK_PATH, sizeof(addr.sun_path) - 1);

    if (bind(sfd, (struct sockaddr *)&addr, sizeof(struct sockaddr_un)) == -1)
      handle_error("bind");

    for (;;) {
      num_read = recvfrom(sfd, buf, BUF_SIZE, 0, NULL, &len);
      if (write(STDOUT_FILENO, buf, num_read) != num_read)
        handle_error("write");
    }
  }
  ```

* A datagram client example.

  ```c
  #include <stdio.h>
  #include <stdlib.h>
  #include <string.h>
  #include <sys/socket.h>
  #include <sys/un.h>
  #include <unistd.h>

  #define MY_SOCK_PATH "tmp"
  #define BUF_SIZE 64

  #define handle_error(msg)                                                      \
    do {                                                                         \
      perror(msg);                                                               \
      exit(EXIT_FAILURE);                                                        \
    } while (0)

  int main() {
    struct sockaddr_un addr;
    int sfd;
    ssize_t num_read;
    char buf[BUF_SIZE];

    sfd = socket(AF_UNIX, SOCK_DGRAM, 0);
    if (sfd == -1)
      handle_error("socket");

    memset(&addr, 0, sizeof(struct sockaddr_un));
    addr.sun_family = AF_UNIX;
    strncpy(addr.sun_path, MY_SOCK_PATH, sizeof(addr.sun_path) - 1);

    while ((num_read = read(STDIN_FILENO, buf, BUF_SIZE)) > 0)
      if (sendto(sfd, buf, num_read, 0, (struct sockaddr *)&addr,
                 sizeof(struct sockaddr_un)) != num_read)
        handle_error("sendto");

    exit(EXIT_SUCCESS);
  }
  ```

## `AF_INET` and `AF_INET6` Overview

* AF_INET uses 4 bytes for addresses while AF_INET6 uses 16 bytes for addresses.
* We mainly discuss `AF_INET` here.
* `man 7 ip` & `man 7 ipv6` provide detailed information.
* `AF_INET` uses `struct sockaddr_in` for addresses.

  ```c
  struct in_addr {
    in_addr_t s_addr;
  };

  struct sockaddr_in {
    sa_family_t     sin_family;
    in_port_t       sin_port;
    struct in_addr  sin_addr;
    unsigned char   __pad[X];
  }
  ```

* There are a few important things to examine in the above address `struct`. We examine one by one
  below.

## `sin_addr`

* This represents an IPv4 address. E.g., 192.168.0.1.
* `sin_addr`: 4 bytes
    * In 192.168.0.1, 192 uses one byte, 168 uses one byte, 0 uses one byte, and 1 uses one byte.
* There are functions that convert a human-readable address to a binary address (and vice
  versa).
    * `inet_pton()`: *presentation* to network
    * `inet_ntop()`: network to *presentation*
    * The above two functions handle both IPv4 and IPv6.
* `<netinet/in.h>` defines two constants:
    * `INET_ADDRSTRLEN`
    * `INET6_ADDRSTRLEN`
    * We can use these constants for a buffer that holds an IPv4 or IPv6 address (presentation
      format).
* There are two special addresses to know.
    * loopback address: 127.0.0.1
        * The constant to use for `sin_addr.s_addr` is `INADDR_LOOPBACK`.
        * This is for local communication, similar to the UNIX domain sockets.
        * If we use this address, data sent/received are all local, i.e., nothing goes out the network
    * Wildcard address
        * The constant to use for `sin_addr.s_addr` is `INADDR_ANY`.
        * A machine can have multiple network cards, e.g., a typical laptop has a wireless card and
          a wired (Ethernet) card. If this is the case, each network card gets a different IP
          address.
        * If we `bind()` a socket to the wildcard address, we tell the OS that we want to listen on
          any address.

## `sin_port`

* When you `bind()` a socket to an address, you not only `bind()` to an IP address but also a port
  number.
* We use this port number to identify each socket. I.e., with IP + port, you can uniquely identify a
  socket.
* There are well-known port numbers that are assigned to specific applications. For example, port 22
  is a well-known port number for ssh. Port 80 is a well-known port number for web servers. So your
  ssh server by default binds to your local IP address and port 22. Your web server by default binds
  to your local IP address and port 80. This is so that other machines can connect to your ssh or
  web server via their well-known port numbers.
* If you do not call `bind()` specifically (and you don't need to, e.g., for a TCP client, or for
  UDP peers), then TCP and UDP picks a port number for you and uses that. This is called an
  *ephemeral* port.

## Network Byte Order

* Since different architectures use different byte orders, i.e., little endian and big endian,
  we need to make sure that the data sent over the network uses the same byte order.
    * Little endian: little end goes first ("little end"---the least significant byte; "first"---the
      lowest address).
    * Big endian: big end goes first ("big end"---the most significant byte; "first"---the lowest
      address).
* This is called the network byte order, and it is big endian.
* For example, the port number and the IP address need to be recognizable by any machine on the
  network, so they need to use the network byte order.
* There are functions you can use to translate this correctly.
    * `man byteorder`

      ```c
      #include <arpa/inet.h>
      uint32_t htonl(uint32_t hostlong);
      uint16_t htons(uint16_t hostshort);
      uint32_t ntohl(uint32_t netlong);
      uint16_t ntohs(uint16_t netshort);
      ```

* This is only applicable for multi-byte data. For data sent byte-by-byte, we do not need to worry
  about converting each byte to the network order since it's only for how to interpret data
  represented in multiple bytes.

## Host names

* Often times, we can use a host name instead of an IP address.
* `getaddrinfo()`: given a host name, returns a set of all possible options (structs containing
  an IP and a port number) as it's possible to associate multiple IP&port addresses to a single
  host name.
* `getnameinfo()`: performs reverse---IP to host name.

## Activity

* Create two programs, server and client. Implement the socket sequence using AF_INET. Send messages
  from the client and print them out from the server.
* A stream server example.

  ```c
  #include <arpa/inet.h>
  #include <errno.h>
  #include <stdio.h>
  #include <stdlib.h>
  #include <string.h>
  #include <sys/socket.h>
  #include <unistd.h>

  #define BUF_SIZE 64
  #define PORT 8000
  #define LISTEN_BACKLOG 32

  #define handle_error(msg)                                                      \
    do {                                                                         \
      perror(msg);                                                               \
      exit(EXIT_FAILURE);                                                        \
    } while (0)

  int main() {
    struct sockaddr_in addr;
    int sfd, cfd;
    ssize_t num_read;
    char buf[BUF_SIZE];

    sfd = socket(AF_INET, SOCK_STREAM, 0);
    if (sfd == -1)
      handle_error("socket");

    memset(&addr, 0, sizeof(struct sockaddr_in));
    addr.sin_family = AF_INET;
    addr.sin_port = htons(PORT);
    addr.sin_addr.s_addr = htonl(INADDR_ANY);

    if (bind(sfd, (struct sockaddr *)&addr, sizeof(struct sockaddr_in)) == -1)
      handle_error("bind");

    if (listen(sfd, LISTEN_BACKLOG) == -1)
      handle_error("listen");

    for (;;) {
      cfd = accept(sfd, NULL, NULL);
      if (cfd == -1)
        handle_error("accept");

      while ((num_read = read(cfd, buf, BUF_SIZE)) > 0) {
        if (write(STDOUT_FILENO, buf, num_read) != num_read)
          handle_error("write");

        if (num_read == -1)
          handle_error("read");
      }
    }

    if (close(cfd) == -1)
      handle_error("close");
  }
  ```

* A stream client example.

  ```c
  #include <arpa/inet.h>
  #include <stdio.h>
  #include <stdlib.h>
  #include <string.h>
  #include <sys/socket.h>
  #include <unistd.h>

  #define PORT 8000
  #define BUF_SIZE 64
  #define ADDR "192.168.68.6"

  #define handle_error(msg)                                                      \
    do {                                                                         \
      perror(msg);                                                               \
      exit(EXIT_FAILURE);                                                        \
    } while (0)

  int main() {
    struct sockaddr_in addr;
    int sfd;
    ssize_t num_read;
    char buf[BUF_SIZE];

    sfd = socket(AF_INET, SOCK_STREAM, 0);
    if (sfd == -1)
      handle_error("socket");

    memset(&addr, 0, sizeof(struct sockaddr_in));
    addr.sin_family = AF_INET;
    addr.sin_port = htons(PORT);
    if (inet_pton(AF_INET, ADDR, &addr.sin_addr) <= 0)
      handle_error("inet_pton");

    if (connect(sfd, (struct sockaddr *)&addr, sizeof(struct sockaddr_in)) == -1)
      handle_error("connect");

    while ((num_read = read(STDIN_FILENO, buf, BUF_SIZE)) > 0)
      if (write(sfd, buf, num_read) != num_read)
        handle_error("write");

    if (num_read == -1)
      handle_error("read");

    exit(EXIT_SUCCESS);
  }
  ```

## `recv()` and `send()`

* `ssize_t recv(int sockfd, void *buf, size_t len, int flags);`
    * Similar to `read()` but socket specific.
    * Provides more control, e.g.:
        * `MSG_DONTWAIT`: Non-blocking
        * `MSG_PEEK`: read but don't remove
* `ssize_t send(int sockfd, const void *buf, size_t len, int flags);`
    * Similar to `write()` but socket specific.
    * Provides more control, e.g.:
        * `MSG_DONTWAIT`: Non-blocking

## Multiple Clients

* How can a server handle multiple clients?
* One possibility: a server can handle multiple clients by creating a child process or a thread
  after accepting a connection.
    * Each new process/thread interacts with a client with a new socket.
    * This has the overhead of creating new processes or threads.
* Another possibility: non-blocking socket
    * Non-blocking `accept()` will either accept a new connection or return immediately if there's
      no incoming connection.
    * Non-blocking `read()` and `write()` will either perform I/O or return immediately if I/O is
      not possible.
    * Then we can define a socket array and write an infinite loop that uses the socket array. The
      loop works as follows.
        * We call `accept()` to receive an incoming connection if any. Add the new socket to the
          socket array. If there's no incoming connection, `accept()` will return immediately.
        * We call either `read()` or `write()` or both appropriately (depending on what we want to
          do with each socket) on all sockets in the array, and see if we can read or write
          immediately.
    * This avoids creating new processes/threads, but the loop keeps checking all the sockets. It's
      a busy-wait loop.
* A better way: use non-blocking sockets but let the kernel tell you if something can be done.
    * `select()`, `poll()`, and `epoll()` performs what's known as I/O multiplexing.
    * They provide an ability to monitor multiple file descriptors to check if a file descriptor is
      ready for read or write (or "error" though this doesn't exactly mean errors and we don't cover
      this).
* Generally speaking, this is how these work.
    * We add file descriptors to the monitored list.
    * We indicate what events we want to monitor the file descriptors for, e.g., read and write.
    * We call the function, e.g., `select()`.
    * When it returns, we check which file descriptors can perform I/O.
    * We perform the I/O.
* Using this method, we can monitor the socket for `accept()` and all the sockets for clients
  altogether. If the socket for `accept()` receives a new connection or other sockets for clients
  receives data, etc., we will be notified.
* `epoll()`
    * `man epoll_create`: `epoll_create()` returns an epoll instance. We can think of this as a
      monitor object that maintains the monitoring list.
    * `man epoll_ctl`: `epoll_ctl()` allows us to add, remove, or modify a file descriptor to the
      epoll instance.
        * When you add a file descriptor, you store `struct epoll_event` that later you receive
          when the file descriptor has something new to process. More on this is below.
    * `man epoll_wait`: `epoll_wait()` waits for a file descriptor to be available for I/O.
        * The second argument `struct epoll_event *events` is the most important one.
        * It is a buffer passed to `epoll_wait()`.
        * Each entry is for a file descriptor that has something new to process. There can be
          multiple entries if different file descriptors each have something new to process.
        * This is the same `struct epoll_event` that you store with `epoll_ctl()` with its last
          argument (`struct epoll_event *_Nullable event`).
        * The kernel stores `struct epoll_event` you pass to `epoll_ctl()`, and when the associated
          file descriptor has something new to process, the kernel returns it back to you.
        * Typically, you store at least the file descriptor, so that you know which file
          descriptor has a new event to process.
        * `man epoll_event` explains further about this.
* Server example.

  ```c
  #include <arpa/inet.h>
  #include <errno.h>
  #include <fcntl.h>
  #include <stdio.h>
  #include <stdlib.h>
  #include <string.h>
  #include <sys/epoll.h>
  #include <sys/socket.h>
  #include <unistd.h>

  #define BUF_SIZE 100
  #define PORT 8000
  #define LISTEN_BACKLOG 32
  #define MAX_EVENTS 10

  #define handle_error(msg)                                                      \
    do {                                                                         \
      perror(msg);                                                               \
      exit(EXIT_FAILURE);                                                        \
    } while (0)

  int main() {
    struct sockaddr_in addr, remote_addr;
    int sfd, cfd, epollfd;
    int nfds;
    ssize_t num_read;
    socklen_t addrlen = sizeof(struct sockaddr_in);
    char buf[BUF_SIZE];
    struct epoll_event ev, events[MAX_EVENTS];

    sfd = socket(AF_INET, SOCK_STREAM, 0);
    if (sfd == -1)
      handle_error("socket");

    memset(&addr, 0, sizeof(struct sockaddr_in));
    addr.sin_family = AF_INET;
    addr.sin_port = htons(PORT);
    addr.sin_addr.s_addr = htonl(INADDR_ANY);

    if (bind(sfd, (struct sockaddr *)&addr, sizeof(struct sockaddr_in)) == -1)
      handle_error("bind");

    if (listen(sfd, LISTEN_BACKLOG) == -1)
      handle_error("listen");

    epollfd = epoll_create1(0);
    if (epollfd == -1)
      handle_error("epoll_create1");

    ev.events = EPOLLIN | EPOLLOUT;
    ev.data.fd = sfd; // save the accept socket
    if (epoll_ctl(epollfd, EPOLL_CTL_ADD, sfd, &ev) == -1)
      handle_error("epoll_ctl");

    for (;;) {
      nfds = epoll_wait(epollfd, events, MAX_EVENTS, -1);
      if (nfds == -1)
        handle_error("epoll_wait");

      for (int i = 0; i < nfds; ++i) {
        if (events[i].data.fd == sfd) {
          memset(&remote_addr, 0, sizeof(struct sockaddr_in));
          cfd = accept(sfd, (struct sockaddr *)&remote_addr, &addrlen);
          if (cfd == -1)
            handle_error("accept");

          // Set O_NONBLOCK
          int flags = fcntl(cfd, F_GETFL, 0);
          if (flags == -1)
            handle_error("fcntl");
          flags |= O_NONBLOCK;
          if (fcntl(cfd, F_SETFL, flags) == -1)
            handle_error("fcntl");

          ev.events = EPOLLIN | EPOLLOUT;
          ev.data.fd = cfd;
          if (epoll_ctl(epollfd, EPOLL_CTL_ADD, cfd, &ev) == -1) {
            perror("epoll_ctl: conn_sock");
            exit(EXIT_FAILURE);
          }
        } else {
          while ((num_read = read(events[i].data.fd, buf, BUF_SIZE)) > 0) {
            if (write(STDOUT_FILENO, buf, num_read) != num_read)
              handle_error("write");

            if (num_read == -1)
              handle_error("read");
          }
        }
      }
    }

    if (close(cfd) == -1)
      handle_error("close");
  }
  ```
