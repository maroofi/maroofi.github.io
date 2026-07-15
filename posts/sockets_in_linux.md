---
layout: posts
---


### Some Socket Notes

This is just a reference to keep here so that I don't check the man pages every time I want to do something.

Sockets are actually files. On opening one, the kernel gives back an integer (fd) and you can `read`, `write` and `close` it like other files. However, instead of using `read()` and `write()`,
we use `recv()` and `send()` (and other functions) to have more control over the operation.

Opening a regular socket is like:

```c
#include <sys/socket.h>

int socket(int domain, int type, int protocol);
```

what are the possible values for `domain`, `type` and `protocol` highly depends on the operation we want to do, and which part should be done by user and/or kernel.

The most important used values for `domain` is as follows:

1. **AF_INET**: Address family for IPv4 networking
2. **AF_INET6**: Address family for IPv6 networking
3. **AF_UNIX**: Local Inter-process Communication
4. **AF_PACKET**: raw link-layer (ethernet) access. This, I usually use for sniffing packets.
5. **AF_UNSPEC**: I haven't used this one in `socket()` function but I know it is used in `getaddrinfo()` function.

Possible values for `type` depends on the value chosen for `domain`:

**domain=[AF_INET/AF_INET6]**:
  - **SOCK_STREAM**: This is used for TCP
  - **SOCK_DGRAM**: This is used for UDP
  - **SOCK_RAW**: niether TCP, nor UDP. This can be used for example for ICMP.

**domain=[AF_UNIX]**:
  - **SOCK_STREAM**: local stream but like TCP
  - **SOCK_DGRAM**: local datagram messaging (like UDP)

**domain=[AF_PACKET]**:
  - **SOCK_RAW**: you should do the full ethernet fram including header!
  - **SOCK_DGRAM**: kernel build/strinp the header for us

and again, possible values for `protocol` depends on both `domain` and `type`:

**domain=AF_INET/AF_INET6 & type=SOCK_STREAM**
  - **IPPROTO_TCP** (or just 0 which means the same): TCP connection

**domain=AF_INET/AF_INET6 & type=SOCK_DGRAM**
  - **IPPROTO_UDP** (or just specify 0 which means the same): UDP connection

**domain=AF_INET/AF_INET6 & type=SOCK_RAW**
  - **IPPROTO_ICMP**: build/receive raw ICMP
  - **IPPROTO_ICMPV6**: IMCPV6
  - **IPPROTO_RAW**: for sending raw IP packet. This means you must build the IP packet headers
  - **IPPROTO_IGMP**: IGMP

**domain=AF_UNIX & type=**_whatever_
  - 0: always specify 0. I don't know if there is any other value we can pass

**domain=AF_PACKET & type=SOCK_RAW**
  - **htons(ETH_P_ALL)**: This will capture everything (the full ethernet packet with header and stuff)
  - **htons(ETH_P_IP)**: This will capture everything from IP layer (In capturing, you will see IP header and in sending, you should build IP header)
  - **htons(ETH_P_ARP)**: This will capture everything but related to ARP messages (capture all ARP with headers, and you should make ARP headers)
  - **htons(ETH_P_IPV6)**: This is the same as `htons(ETH_P_IP)` but for IPv6.

**domain=AF_PACKET & type=SOCK_DGRAM**
  - **htons(ETH_P_ALL)**: you won't capture/build the ethernet headers. Already stripped in capturing and the kernel will build it for you in sending mode.
  - **htons(ETH_P_IP)**: you capture everything from IP layer but the kernel already stripped IP header and just deliver you the payload. For sending, you don't need to make IP header.
  - **htons(ETH_P_ARP)**: The same as `htons(ETH_P_IP)` but for ARP
  - **htons(ETH_P_IPV6)**: The same as `htons(ETH_P_IP)` but for IPv6.


So basically, with `domain`, you tell the kernel which OSI layer you want to work on and which stack (IPv4 or IPv6), if it's local or Internet level.
However, using `type`, you basically tell the kernel which protocol you want the kernel to wrap your payload with (in sending/receiving mode) or to strip from the payload.
Finally, the `protocol` just acts as a filter.

Here are some examples:

1. _socket(AF_PACKET, SOCK_RAW, htons(ETH_P_ALL))_: with this, we can capture all the packets of layer 2 with the ethernet frame headers. So basically, we can make any payload and wrap it
in the ethernet frame and pass it to the kernel. It will be sent by the NIC but we must specify everything.

2. _socket(AF_INET, SOCK_RAW, IPPROTO_ICMP)_: In receiving, the kernel only filter ICMP packets. In sending mode, we must make the ICMP packet but the kernel will make the IP header and ethernet header.

3. _socket(AF_INET, SOCK_RAW, IPPROTO_RAW)_: the kernel only build the ethernet header. The whole IP header is up to us, in both sending and receiving.

4. _socket(AF_INET, SOCK_STREAM, 0)_: Kernel handles all ethernet, IP and TCP packet headers and routing and everything. You just do the freaking payload.

5. _socket(AF_INET, SOCK_DGRAM, 0)_: The same as 4, but for UDP.

---------------------------------------

### Sending data to the destination

Function like `connect()`, `bind()`, `accept()` and `sendto()` need a destination address to send data. However, the destination address can be different based on the stack that is used
(e.g., IPv4 or IPv6) and/or the OSI layer we are working on. For example, a destination address of IPv4 needs a 4 bytes of IPv4 address and 2 bytes of port number while the destination
with an IPv6 needs 16 bytes of address and two bytes of port number. At the link-layer, a destination is a MAC address which needs 6 bytes for the mac address.

As an example, let's consider the `connect()` function (used for TCP connection in clients). Here is the syntax:

```c
#include <sys/socket.h>

int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```

The connect function needs to know the type, as well as the address of the destination. This data is passed through `sockaddr` structure. However, the function needs to know if it's IPv4 or
IPv6. C lang has no polymorphism so we can have only one structure passed to the function but we need different types/sizes for different stacks!

```c
#include <sys/socket.h>
struct sockaddr {
    sa_family_t sa_family;   // address family, e.g. AF_INET
    char        sa_data[14]; // protocol-specific address data
};
```
One possible trick to do the polymorphism is to always pass a generic structure and its size (it's important so that the machine knows how many bytes to read) but for each address type,
the machine read the structure differently. After all, it's just all a bunch of bytes in the memory.

`sa_family_t` is always two bytes (defined in `sockaddr.h`):

```c
/* POSIX.1g specifies this type name for the `sa_family' member.  */
typedef unsigned short int sa_family_t;
```
from `sa_family_t`, the kernel knows which stack we are addressing and how to deal with the rest of the structure. In practice, we never fill `sockaddr` structure. For each type and address,
we have a separate structure to pass to the functions but we cast them to `sockaddr` to trick the kernel!

For example, for the `connect()` function, we use `sockaddr_in`:

```c
#include <netinet/in.h>
struct sockaddr_in {
    sa_family_t    sin_family;  // AF_INET. This is just an informational tag for the kernel.
    in_port_t      sin_port;    // port, network byte order. Destination port, goes to TCP packet header
    struct in_addr sin_addr;    // IPv4 address, network byte order. Destination addr, goes to IP packet header
    char           sin_zero[8]; // padding, unused, must be zeroed
};

struct in_addr {
    uint32_t s_addr;   // the actual 32-bit IPv4 address
};
```
That was for IPv4. For IPv6, we use a different structure, `sockaddr_in6`:

```c
#include <netinet/in.h>
struct sockaddr_in6 {
    sa_family_t     sin6_family;   // AF_INET6
    in_port_t       sin6_port;     // port, network byte order
    uint32_t        sin6_flowinfo; // IPv6 flow label (rarely used)
    struct in6_addr sin6_addr;     // 128-bit IPv6 address
    uint32_t        sin6_scope_id; // scope ID for link-local addresses
};
```

Or to send ether packets, we use `sockaddr_ll`:

```c
#include <linux/if_packet.h>
struct sockaddr_ll {
    sa_family_t sll_family;    // always AF_PACKET
    __be16      sll_protocol;  // EtherType, network byte order. goes directly in ethernet frame header
    int         sll_ifindex;   // interface index (e.g., eth0 → 2)
    unsigned short sll_hatype; // ARP hardware type
    unsigned char  sll_pkttype;// packet type (see below)
    unsigned char  sll_halen;  // length of address below (6 for Ethernet/MAC)
    unsigned char  sll_addr[8];// physical-layer address (MAC uses first 6 bytes). goes to the ethernet frame header
};
```

or a general one that is guaranteed to be big enough to hold all types of addresses: `sockaddr_storage`: This can be used for example in `accept()` function when
we don't know if the client is using IPv4 or IPv6 to connect.

```c
struct sockaddr_storage {
    sa_family_t ss_family;
    // ... padding, guaranteed large enough and aligned for ANY sockaddr_* variant
};
```

Using these structures, we pass all the necesssary information to the system to send a packet to the network. Usually, the addresses are in network byte orders, so we use functions
like `htons()`, `htonl()` to convert from host to network (short and long) order and `ntons()` and `ntohl()` to convert from network to host (short and long). Here is the list of useful
functions:

```c
uint16_t htons(uint16_t hostshort);
// Converts a 16-bit value from host byte order to network byte order.
// Used for ports, since ports are 16 bits.
```

```c
uint32_t htonl(uint32_t hostlong);
// Same idea, but for 32-bit values. Used for IPv4 addresses when you're constructing them manually
// example uint32_t ip = htonl(0xC0A80101); // 192.168.1.1 manually
```

```c
uint16_t ntohs(uint16_t netshort);
// converts a 16-bit value received from the network back into your machine's
// native format so you can actually use/print it correctly.
```

```c
uint32_t ntohl(uint32_t netlong);
// Reverse of htonl(), for 32-bit values.
```

and if we have the address in human readable string (presentation) and need to convert it to network order:

```c
#include <arpa/inet.h>
int inet_pton(int af, const char *src, void *dst);
// Converts a human-readable address string into its binary form, writing the result into dst.
// Works for both IPv4 and IPv6 — you tell it which via the af parameter.
// af parameter can be AF_INET or AF_INET6
```

```c
inet_ntop()   // network to presentation
const char *inet_ntop(int af, const void *src, char *dst, socklen_t size);
//  takes a binary address and produces a human-readable string.
// for example for logging the connection received with accept() function
// socklen_t is the length of the dst
```

Now what if you don't know the address on beforehand (e.g., the user enter the address from command-line). Which function to use?
What if there is no address and instead, we have a domain name? How to get the address and stuff?

**getaddrinfo() solves both problems in one call: it does DNS resolution and produces ready-to-use sockaddr structures, for whichever address families you ask for.**

```c
#include <netdb.h>

int getaddrinfo(const char *node,
                 const char *service,
                 const struct addrinfo *hints,
                 struct addrinfo **res);

// node: either a domain name or IP address
// service: either a port number (e.g., "80") or service name ("https")
// hints: fill this one to give the kernel some hints about the service
// res: a pointer to a link list of results. You just pass the pointer,
// you don't need to allocate memory.You can free the allocated memory
// for the res using freeaddrinfo() function.
```

The result of this function is a linked-list. Why? a domain name may have multiple A records.
The `addrinfo` structure is like:

```c
struct addrinfo {
    int              ai_flags;
    int              ai_family;    // AF_INET, AF_INET6, or AF_UNSPEC
    int              ai_socktype;  // SOCK_STREAM or SOCK_DGRAM
    int              ai_protocol;  // usually 0
    socklen_t        ai_addrlen;
    struct sockaddr *ai_addr;      // ← the actual sockaddr_in/in6, ready to use
    char            *ai_canonname;
    struct addrinfo *ai_next;      // ← linked list to the next candidate
};
```

In the `hints` parameter of `getaddrinfo()`, we can set the `ai_family` to `AF_UNSPEC` which means we don't care about IP version. Basically, we just need to fill in two fiels of
the `hints` param: `ai_family` and `ai_socktype`. We can set the rest all to zero using `memset()` function.

-------------------------

### Server operations: bind(), listen(), and accept()

Most of the time, we use `bind()` for server side operations. Here is the syntax of the bind():

```c
#include <sys/socket.h>

int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```
`bind()` actually assigns a local address (e.g., IP+PORT in AF_INET) to a socket. If you are running a server, it needs to have a specific IP and port. When you are running a client code, you 
still need local IP and port but if you don't call bind (which we almost never do), the kernel will automatically assign us a port and bind to the default interface IP address.

When we want to bind to the default IP address, the constant `INADDR_ANY` can be useful. This is basically the same as 0.0.0.0 address.

```c
// in the file in.h
/* Address to accept any incoming messages.  */
#define INADDR_ANY      ((in_addr_t) 0x00000000)
```

Now one of the very common problem when we write server code is the **address already in use** error. This happens when for example, you are running a TCP server bound to port 8080. You
kill the code with CTRL+c and you want to run it again immediately. Here is the error you get: **bind: Address already in use**.

The reason is the **TIME_WAIT** of TCP. When a TCP connection closes, the side that initiate the close will send **FIN** request and does not immediately free up the connection. It's kind of
a post handshake that will be done between peers and during this time, the server enters **TIME_WAIT** mode. This may take from 60 seconds to few minutes (depend on the OS). One solution
to avoid this is to set a socket option `SO_REUSEADDR`. 

```c
int optval = 1;
setsockopt(sockfd, SOL_SOCKET, SO_REUSEADDR, &optval, sizeof(optval));
```
This tells the kernel: "**allow this socket to bind to an address/port that's in TIME_WAIT, as long as it's not actually actively LISTENing elsewhere.**"

#### What if you want to bind to both IPv4 and IPv6 at the same time?

There are several ways to do this. The second parameter of the `bind()` is the `sockaddr` structure which has a member `sin_family`. This can be either `AF_INET` or `AF_INET6`. This means
that `bind()` can only work in one of the stacks (4 or 6). 

here is the classic example of `bind()`:

```c
int sockfd = socket(AF_INET, SOCK_STREAM, 0);

int yes = 1;
setsockopt(sockfd, SOL_SOCKET, SO_REUSEADDR, &yes, sizeof(yes));

struct sockaddr_in addr;
memset(&addr, 0, sizeof(addr));
addr.sin_family      = AF_INET;
addr.sin_port        = htons(8080);
addr.sin_addr.s_addr = INADDR_ANY;

bind(sockfd, (struct sockaddr *)&addr, sizeof(addr));

listen(sockfd, SOMAXCONN);   // ← socket is now passively listening

// next: accept() in a loop
```
And also we know that `INADDR_ANY` equals to `0.0.0.0` which is IPv4. So if you want to bind to IPv6, here is the equivalent:

```c
struct sockaddr_in6 addr6;
addr6.sin6_family = AF_INET6;
addr6.sin6_addr   = in6addr_any;    // the IPv6 wildcard, "::"
```

But if you want to do the both, one solution (the cleanest) is to `fork()` the process and the new process run the other stack. I like this one because it's clean. for example, the parent runs
on IPv4 and the child runs on IPv6 or you can run two versions of the code to handle things completely separately.

another approach is to use **IPv4-mapped IPv6 address**. In this approach:

```c
int sockfd = socket(AF_INET6, SOCK_STREAM, 0);

struct sockaddr_in6 addr6;
memset(&addr6, 0, sizeof(addr6));
addr6.sin6_family = AF_INET6;
addr6.sin6_addr   = in6addr_any;   // "::" — binds all IPv6 interfaces
addr6.sin6_port   = htons(8080);

int v6only = 0;   // 0 = allow IPv4-mapped connections too (dual-stack)
                  // 1 = IPv6 ONLY, reject IPv4-mapped connections
setsockopt(sockfd, IPPROTO_IPV6, IPV6_V6ONLY, &v6only, sizeof(v6only));

bind(sockfd, (struct sockaddr *)&addr6, sizeof(addr6));
listen(sockfd, SOMAXCONN);
```

This will accept all the IPv6 addresses and maps all the IPv4 addresses to IPv6 (and accept them as well). A IPv4 `192.168.1.1` will be mapped to `::ffff:192.168.1.1`.
(I personally use two separate stacks).

#### Listen to connections

The `listen()` function makes the socket passive. All the functions we call before listen, they just create sockets and (optionally) bind socket to a local address. However, the 
kernel still does not know if we want to send data or receive data. By calling `listen()`, the kernel knows that this socket should receive incoming connections. So it starts listening
to the incoming connections and does the handshake (TCP) for us and queues all the connections, waiting for us to call the `accept()` function. The syntax of the `listen()` function is:

```c
listen(sockfd, backlog);
```

the `backlog` is actually the number of connections that is put in the queue waiting for the `accept()` function to be called. This is not an exact number and its maximum value is
always smaller than the maximum values specified by the kernel. The maximum possible value is stored in `/proc/sys/net/core/somaxconn` (which is usually 4096 for new OSes).

The macro **`SOMAXCONN`** specified in `<sys/socket.h>` specifies the maximum possible value and lets us call the `listen()` function like:
```c
listen(sockfd, SOMAXCONN);
```

#### Let's accept the incoming connections

After listening to the socket, we can start accepting connections and this is where client and server are fully connected and can exchange data. The syntax of the `accept()` function is:

```c
#include <sys/socket.h>

int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
```
We need to allocate memory for `addr` parameter (stack or heap does not matter) and pass it to `accept()` function. Every accepted connection will fill the structure and we will know who is connected to our server. The
structure must be large enough to hold the connection information. If we bind on IPv4, we can pass `sockaddr_in` and if we bind on IPv6, we can pass `sockaddr_in6`. If we bind on both,
since we don't know which stack the client is using, we can pass `sockaddr_storage` which is large enough to keep all the cases. If we don't care about client information, we can pass
NULL like this:

```c
int client_fd = accept(sockfd, NULL, NULL);
```
Every accepted connection will return a new socket file descriptor which can be used to send/recv data to/from client. Whenever we don't need it anymore, we can just simply close it.
We can use `fork()` or `pthreads` for each newly accepted connection.

Here is the small example how to use `fork()` in the server code to deal with newly accepted connections:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <signal.h>
#include <sys/wait.h>
#include <sys/socket.h>
#include <netinet/in.h>

int main(void) {
    int sockfd = socket(AF_INET, SOCK_STREAM, 0);

    int yes = 1;
    setsockopt(sockfd, SOL_SOCKET, SO_REUSEADDR, &yes, sizeof(yes));

    struct sockaddr_in addr;
    memset(&addr, 0, sizeof(addr));
    addr.sin_family      = AF_INET;
    addr.sin_port        = htons(8080);
    addr.sin_addr.s_addr = INADDR_ANY;

    bind(sockfd, (struct sockaddr *)&addr, sizeof(addr));
    listen(sockfd, SOMAXCONN);

    // reap dead children automatically, avoid zombies
    signal(SIGCHLD, SIG_IGN);

    while (1) {
        struct sockaddr_in client_addr;
        socklen_t client_len = sizeof(client_addr);

        int client_fd = accept(sockfd, (struct sockaddr *)&client_addr, &client_len);
        if (client_fd == -1) {
            perror("accept");
            continue;
        }

        pid_t pid = fork();

        if (pid == -1) {
            perror("fork");
            close(client_fd);
            continue;
        }

        if (pid == 0) {
            // ---- CHILD PROCESS ----
            close(sockfd);        // child doesn't need the listening socket
            // handle client_fd here: recv()/send()
            close(client_fd);
            exit(0);
        } else {
            // ---- PARENT PROCESS ----
            close(client_fd);     // parent doesn't need the connected socket
            // loop back to accept() immediately, ready for the next client
        }
    }

    close(sockfd);
    return 0;
}
```

-------------------------------------------

#### Sending and Receiving data

Now when it comes to sending and receiving data, the whole thing is a little bit tricky. We need to first know if we are talking about a specific protocol (e.g., TCP, UDP) or sending a 
raw payload over IP?

Let's start with TCP. In TCP, we have no concept of "**message**". All we have is a stream of data.

Here is the syntax of the two functions:

```c
#include <sys/socket.h>

ssize_t send(int sockfd, const void *buf, size_t len, int flags);
ssize_t recv(int sockfd, void *buf, size_t len, int flags);
```

The return value of the both functions is the actual bytes sent/received or -1 in case of error. When I say, "there is no concept of message", it means that the kernel is free to break
your data from any points it feels like! In one `recv()` call, you may get 1 byte or the whole data or a few bytes of data. You need to have your own (upper level) protocol to know when to 
stop reading data. The same rule applies to `send()` function. When you call, `send()`, you just send the data from your machine to the kernel buffer. It's the kernel who decides when to send
the data. The buffer might be partially full (probably from the previous calls). This means that you may not even be able to send all the data at once. So the return value of `send()` function
can be any value, not necessarily the actual amount you tell it to send. Therefore, for both functions, you must call them in a loop to collect what you expect to collect or kill it when
there is an error (e.g., the other side closes the connection).

The return value of `recv()` function has 3 possibilities:


|Return Value |Meaning|
|----------|-------------------|
| n > 0| Successfully received n bytes|
| n = 0 | The peer closed the connection|
|n == -1| An actual error occurred — check `errno`|

Even though we covered every possibility, still these functions need careful control and a quality code make sure it covers all the possible scenarios like closing connection, 
suddin closing, slow-loris attack. timeout, etc...

I will write a code to cover all these after introducing `poll()` function.

#### Sending data in connectionless mode

In UDP, there is no need to call the `connect()` function as we don't need to connect to someone. We just create the socket and start sending data. This means we need another function and
we can not use `send()` here without calling `connect()`. Therefore, we use `sendto()` function:

```c
#include <sys/socket.h>

ssize_t sendto(int sockfd, const void *buf, size_t len, int flags,
                const struct sockaddr *dest_addr, socklen_t addrlen);
```

`sendto()` function has two more parameters that covers what `connect()` was suppoed to do it for us. We need to specify the destination address in every call. This means that we can 
open a socket one time and send data to different destination by calling `sendto()` functions several times.

the other function for receiving data in UDP is called `recvfrom()`.

```c
ssize_t recvfrom(int sockfd, void *buf, size_t len, int flags,
                  struct sockaddr *src_addr, socklen_t *addrlen);

// you allocate the buffer and kernel fills it for you!
```

In UDP, **unlike** TCP, whatever that is sent by `sendto()` will be received in **one call** of `recvfrom()`. This means that there is a message boundry and it's not just a stream of bytes. 
`sendto()` sends exactly one datagram and `recvfrom()` receives exactly one datagram.


#### Can UDP use connect() function.

Of course it's possible. You can call the `connect()` function before sending and receiving data. In this way, it's possible to call `send()` and `recv()` instead of `sendto()` and 
`recvfrom()`. This also has a small advantage. Calling `recv()` after `connect()` in UDP mode will filter out all UDP packets which don't match your given destination. However, using 
`recvfrom()` you will receive all the packets destined for the given port no matter which IP address they are coming from!

#### Better ways of handling connections: poll() and epoll()

Before getting into details of `poll()` and `epoll()`, it's worth to mention that there is also another function called `select()` which can do the job for us. `select()` is POSIX and
it's available everywhere (Linux, Unix,....). However, it suffers from several limitations and that's why Linux introduced `epoll()`. `poll()` is also POSIX and it handles some of the 
limitations of `select()` but still it's slower than `epoll()` and that's because of the algorithm behind it. We are going to introduce both `poll()` and `epoll()` but it is recommended
to only use `epoll()`. We introduce `poll()` because it's just simply easier to understand and it helps better understanding 'epoll()`.










