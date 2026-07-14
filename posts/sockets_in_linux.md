
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





