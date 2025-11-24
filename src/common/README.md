# PvPGN Common Library

## Overview

The `src/common` folder contains the foundational shared library that provides core functionality used throughout the entire PvPGN server project. This library implements fundamental data structures, protocol definitions, networking primitives, cryptographic functions, and utility routines that form the backbone of all PvPGN server components (bnetd, d2cs, d2dbs, and client tools).

## Purpose

The common library serves as:

1. **Foundation Layer**: Provides basic building blocks for all PvPGN components
2. **Protocol Definitions**: Defines all Battle.net and Diablo protocol structures
3. **Cross-Platform Abstraction**: Handles platform-specific differences (Windows/Unix/macOS)
4. **Performance Optimization**: Implements efficient I/O multiplexing and data structures
5. **Security**: Provides cryptographic hashing and authentication primitives
6. **Code Reuse**: Eliminates duplication across server daemons and client tools

## Architecture

The common library is organized into several functional categories:

### 1. Data Structures

#### Linked Lists (`list.h/cpp`)
Generic doubly-linked list implementation.

**Key Features**:
- O(1) prepend/append operations
- Safe traversal with element removal
- Reference counting support
- Debug mode with leak detection

**API**:
```cpp
t_list* list_create();
int list_destroy(t_list* list);
int list_prepend_data(t_list* list, void* data);
int list_append_data(t_list* list, void* data);
t_elem* list_get_first(t_list* list);
void* elem_get_data(t_elem* elem);

// Safe traversal macros
LIST_TRAVERSE(list, curr)
LIST_TRAVERSE_CONST(list, curr)
```

**Usage Example**:
```cpp
t_list* users = list_create();
list_append_data(users, user_ptr);
LIST_TRAVERSE(users, curr) {
    t_user* user = elem_get_data(curr);
    // Process user
}
```

#### Hash Tables (`hashtable.h/cpp`)
High-performance hash table with chaining.

**Key Features**:
- Configurable bucket count
- O(1) average insert/lookup/delete
- Zombie entry management (lazy deletion)
- Hash-based iteration
- Automatic memory management

**API**:
```cpp
t_hashtable* hashtable_create(unsigned int num_rows);
int hashtable_insert_data(t_hashtable* ht, void* data, unsigned int hash);
t_entry* hashtable_get_entry_by_data(t_hashtable* ht, void* data, unsigned int hash);
int hashtable_remove_data(t_hashtable* ht, void* data, unsigned int hash);
```

**Typical Usage**:
- Account storage (hash by username)
- Game listings (hash by game ID)
- Connection tracking (hash by socket FD)

#### Queues (`queue.h/cpp`)
Ring buffer-based FIFO queue for packets.

**Key Features**:
- Dynamic resizing (powers of 2)
- Packet reference counting
- O(1) push/pop operations
- Efficient memory usage

**API**:
```cpp
void queue_push_packet(t_queue** queue, t_packet* packet);
t_packet* queue_pull_packet(t_queue** queue);
t_packet* queue_peek_packet(t_queue const* const* queue);
int queue_get_length(t_queue const* const* queue);
void queue_clear(t_queue** queue);
```

**Use Cases**:
- Outgoing packet buffering
- Incoming packet assembly
- Asynchronous I/O queuing

#### Embedded Lists (`elist.h`)
Intrusive linked lists for zero-allocation traversal.

**Key Features**:
- No dynamic allocation for list nodes
- Cache-friendly memory layout
- C++ template-based
- Type-safe iteration

**Usage**:
```cpp
struct MyStruct {
    int data;
    elist_node<MyStruct> list_link;
};

elist<MyStruct> mylist;
mylist.push_back(my_struct_instance);
```

### 2. Networking

#### Packet System (`packet.h/cpp`)
Universal packet abstraction for all protocols.

**Packet Classes**:
```cpp
packet_class_none      // Uninitialized
packet_class_init      // Connection initialization
packet_class_bnet      // Battle.net protocol
packet_class_file      // File transfer protocol
packet_class_raw       // Raw data (UDP, bot protocol)
packet_class_udp       // UDP-specific
packet_class_d2game    // Diablo II game protocol
packet_class_d2gs      // Diablo II game server
packet_class_d2cs      // Diablo II character server
packet_class_d2cs_bnetd // D2CS<->bnetd communication
packet_class_w3route   // Warcraft III routing
packet_class_wolgameres // Westwood Online game results
```

**Packet Structure**:
```cpp
typedef struct {
    unsigned int   ref;      // Reference count
    t_packet_class pclass;   // Packet class
    unsigned int   flags;    // User-defined flags
    unsigned int   len;      // Data length
    union {
        char data[MAX_PACKET_SIZE];
        t_bnet_generic   bnet;
        t_file_generic   file;
        t_udp_generic    udp;
        // ... protocol-specific structures
    };
} t_packet;
```

**API**:
```cpp
t_packet* packet_create(t_packet_class pclass);
void packet_del_ref(t_packet* packet);
unsigned int packet_get_size(t_packet* packet);
void packet_set_size(t_packet* packet, unsigned int size);
unsigned short packet_get_type(t_packet* packet);
void* packet_get_raw_data(t_packet* packet, unsigned int offset);
```

#### Network I/O (`network.h/cpp`)
Low-level socket I/O with partial read/write handling.

**Functions**:
```cpp
int net_recv(int sock, void* buff, int len);
int net_send(int sock, const void* buff, int len);
int net_recv_packet(int sock, t_packet* packet, unsigned int* currsize);
int net_send_packet(int sock, t_packet const* packet, unsigned int* currsize);
```

**Returns**:
- `-1`: Error (connection closed or fatal error)
- `0`: Partial operation (would block, continue later)
- `1`: Complete operation

#### File Descriptor Watching (`fdwatch.h/cpp`)
Abstraction layer for I/O multiplexing.

**Backends** (automatically selected):
1. **epoll** (`fdwatch_epoll.h/cpp`) - Linux, best performance
2. **kqueue** (`fdwatch_kqueue.h/cpp`) - BSD/macOS, excellent performance
3. **poll** (`fdwatch_poll.h/cpp`) - POSIX, good performance
4. **select** (`fdwatch_select.h/cpp`) - Universal fallback, limited scalability

**API**:
```cpp
int fdwatch_init(int maxcons);
int fdwatch_add_fd(int fd, unsigned rw, fdwatch_handler h, void* data);
int fdwatch_update_fd(int idx, unsigned rw);
int fdwatch_del_fd(int idx);
int fdwatch(long timeout_msecs);
void fdwatch_handle();
```

**Handler Signature**:
```cpp
typedef int(*fdwatch_handler)(void* data, t_fdwatch_type);
```

**Usage Pattern**:
```cpp
// Initialize
fdwatch_init(1024);

// Register socket
fdwatch_add_fd(sock, fdwatch_type_read, on_readable, conn_data);

// Main loop
while (running) {
    fdwatch(1000); // 1 second timeout
    fdwatch_handle(); // Call registered handlers
}
```

**Performance Characteristics**:
| Backend | Connections | Complexity | Platform |
|---------|-------------|------------|----------|
| epoll | 100,000+ | O(1) | Linux 2.6+ |
| kqueue | 100,000+ | O(1) | FreeBSD, macOS |
| poll | 10,000+ | O(n) | POSIX |
| select | 1,024 | O(n) | Universal |

#### Address Handling (`addr.h/cpp`)
IP address abstraction supporting IPv4 and IPv6.

**Functions**:
```cpp
// Address parsing and formatting
char const* addr_num_to_addr_str(unsigned int addr, unsigned short port);
char const* addr_num_to_ip_str(unsigned int addr);
int addr_get_addr_str(t_addr const* addr, char* dst, unsigned int size);

// Address comparison
int addr_cmp_addr(t_addr const* addr1, t_addr const* addr2);
```

### 3. Protocol Definitions

#### Battle.net Protocol (`bnet_protocol.h`)
**4101 lines** - Complete Battle.net protocol specification.

**Packet Categories**:
- Connection initialization
- Authentication and account creation
- Chat and channels
- Game listing and creation
- Statistics and ladder
- File requests
- Friend lists and clans
- News and MOTD
- Version checking
- Tournament systems

**Example Structures**:
```cpp
typedef struct {
    t_bnet_header h;
    bn_int        sessionkey;
    bn_int        sessionnum;
} t_client_initconn;

typedef struct {
    t_bnet_header h;
    bn_int        ticks;
    /* NUL-terminated message text follows */
} t_server_message;
```

**Packet Types** (selected):
- `CLIENT_INITCONN (0x01)` - Initial connection
- `CLIENT_CHATCOMMAND (0x0e)` - Chat commands
- `CLIENT_MESSAGE (0x0f)` - Chat messages
- `SERVER_MESSAGE (0x0f)` - Server messages
- `CLIENT_STATSREQ (0x26)` - Statistics request
- `CLIENT_GAMELISTREQ (0x09)` - Game list request

#### Other Protocols
- `init_protocol.h` - Connection establishment
- `file_protocol.h` - File transfer (MPQ, icons)
- `bot_protocol.h` - Bot/telnet text protocol
- `udp_protocol.h` - UDP game traffic
- `irc_protocol.h` - IRC gateway protocol
- `d2cs_protocol.h` - Diablo II character server
- `d2game_protocol.h` - Diablo II game protocol
- `d2cs_d2gs_protocol.h` - D2CS to D2GS communication
- `d2cs_bnetd_protocol.h` - D2CS to bnetd communication
- `anongame_protocol.h` - Anonymous matchmaking
- `wol_gameres_protocol.h` - Westwood Online game results

#### Field Sizes (`field_sizes.h`)
Constants for protocol limits:
```cpp
#define MAX_PACKET_SIZE      2048
#define MAX_MESSAGE_LEN      256
#define MAX_USERNAME_LEN     16
#define MAX_CHATNAME_LEN     32
#define MAX_CHANNELNAME_LEN  32
#define MAX_GAMENAME_LEN     32
#define MAX_GAMEDESC_LEN     128
```

### 4. Cryptography and Hashing

#### Battle.net Hashing (`bnethash.h/cpp`)
Modified SHA-1 for password and CD key hashing.

**Functions**:
```cpp
int bnet_hash(t_hash* hashout, unsigned int size, void const* data);
int sha1_hash(t_hash* hashout, unsigned int size, void const* data);
int hash_eq(t_hash const h1, t_hash const h2);
char const* hash_get_str(t_hash const hash);
int hash_set_str(t_hash* hash, char const* str);
```

**Hash Type**:
```cpp
typedef std::uint32_t[5] t_hash; // 160-bit hash
```

**Usage**:
```cpp
t_hash password_hash;
bnet_hash(&password_hash, strlen(password), password);
if (hash_eq(password_hash, stored_hash)) {
    // Password match
}
```

#### Hash Conversion (`bnethashconv.h/cpp`)
Convert between hash formats (old/new Battle.net).

#### SRP-3 Authentication (`bnetsrp3.h/cpp`)
Secure Remote Password protocol version 3.

**Features**:
- Zero-knowledge password proof
- Mutual authentication
- Session key derivation
- Protects against replay attacks

#### Westwood Online Hash (`wolhash.h/cpp`)
Hashing for Command & Conquer games.

#### Big Integer Math (`bigint.h/cpp`)
Arbitrary precision integer arithmetic for SRP-3.

### 5. Data Type Handling

#### Battle.net Types (`bn_type.h/cpp`)
Network byte order conversion for Battle.net integers.

**Types**:
```cpp
typedef struct { std::uint8_t  data[1]; } bn_byte;
typedef struct { std::uint8_t  data[2]; } bn_short;
typedef struct { std::uint8_t  data[3]; } bn_int3;
typedef struct { std::uint8_t  data[4]; } bn_int;
typedef struct { std::uint8_t  data[8]; } bn_long;
```

**Functions**:
```cpp
// Get (network -> host)
std::uint16_t bn_short_get(bn_short src);
std::uint32_t bn_int_get(bn_int src);
std::uint64_t bn_long_get(bn_long src);

// Network byte order get
std::uint16_t bn_short_nget(bn_short src);
std::uint32_t bn_int_nget(bn_int src);

// Set (host -> network)
void bn_short_set(bn_short* dst, std::uint16_t val);
void bn_int_set(bn_int* dst, std::uint32_t val);
void bn_long_set(bn_long* dst, std::uint64_t val);
```

**Purpose**: Handle endianness differences between x86 (little-endian) clients and network byte order (big-endian).

#### Tags (`tag.h/cpp`)
Client, architecture, and language tags.

**Tag Types**:
```cpp
typedef std::uint32_t t_tag;
typedef t_tag t_clienttag;  // e.g., "STAR", "SEXP", "D2DV"
typedef t_tag t_archtag;    // e.g., "IX86", "PMAC", "XMAC"
typedef t_tag t_gamelang;   // e.g., "enUS", "deDE", "frFR"
```

**Functions**:
```cpp
t_tag tag_str_to_uint(char const* tag_str);
char const* tag_uint_to_str(t_tag tag);
char const* clienttag_uint_to_str(t_clienttag clienttag);
char const* archtag_uint_to_str(t_archtag archtag);
```

**Common Tags**:
- `CLIENTTAG_STARCRAFT` - Starcraft
- `CLIENTTAG_BROODWARS` - Brood War
- `CLIENTTAG_DIABLORTL` - Diablo
- `CLIENTTAG_DIABLO2DV` - Diablo II
- `CLIENTTAG_WARCRAFT3` - Warcraft III
- `ARCHTAG_WINX86` - Windows x86
- `ARCHTAG_MACPPC` - Macintosh PowerPC

### 6. Time and Date

#### Battle.net Time (`bnettime.h/cpp`)
Battle.net-specific timestamp conversion.

**Functions**:
```cpp
bn_long bnettime_add_tzbias(bn_long bnettime);
bn_long bnettime_from_time_t(std::time_t stdtime);
std::time_t bnettime_to_time_t(bn_long bnettime);
char const* bnettime_get_str(bn_long bnettime);
```

**Format**: Seconds since January 1, 1970 00:00:00 GMT (like Unix time, but stored in Battle.net byte order).

### 7. Configuration

#### Configuration System (`conf.h/cpp`)
Generic configuration entry framework.

**Entry Structure**:
```cpp
typedef struct {
    const char* name;
    int (*set)(const char* valstr);  // Set value from string
    const char* (*get)(void);         // Get value as string
    int (*setdef)(void);              // Set default value
} t_conf_entry;
```

**Utility Functions**:
```cpp
int conf_set_bool(unsigned* pbool, const char* valstr, unsigned def);
int conf_set_int(unsigned* pint, const char* valstr, unsigned def);
int conf_set_str(const char** pstr, const char* valstr, const char* def);
int conf_load_file(const char* filename, const t_conf_entry* entries, int n);
```

**Usage Pattern**:
```cpp
static unsigned myconfig_port;
static const char* myconfig_logfile;

static int _set_port(const char* valstr) {
    return conf_set_int(&myconfig_port, valstr, 6112);
}

static const char* _get_port() {
    static char buf[16];
    snprintf(buf, sizeof(buf), "%u", myconfig_port);
    return buf;
}

static t_conf_entry conf_table[] = {
    { "port", _set_port, _get_port, NULL },
    { "logfile", _set_logfile, _get_logfile, NULL }
};
```

### 8. Logging

#### Event Logging (`eventlog.h/cpp`)
Flexible logging system with severity levels.

**Log Levels**:
```cpp
eventlog_level_none   // No logging
eventlog_level_trace  // Trace execution
eventlog_level_debug  // Debug information
eventlog_level_info   // Informational messages
eventlog_level_warn   // Warnings
eventlog_level_error  // Errors
eventlog_level_fatal  // Fatal errors
```

**API**:
```cpp
// Setup
void eventlog_set(std::FILE* fp);
int eventlog_open(char const* filename);
int eventlog_close();

// Level control
void eventlog_clear_level();
int eventlog_add_level(char const* levelname);
int eventlog_del_level(char const* levelname);

// Logging (uses fmt library)
template <typename... Args>
void eventlog(t_eventlog_level level, const char* module, 
              fmt::string_view format_str, const Args&... args);

// Hex dump utility
void eventlog_hexdump_data(void const* data, unsigned int len);
```

**Usage Example**:
```cpp
// Initialization
eventlog_add_level("info");
eventlog_add_level("warn");
eventlog_add_level("error");
eventlog_open("/var/log/pvpgn/bnetd.log");

// Logging
eventlog(eventlog_level_info, __FUNCTION__, 
         "User {} logged in from {}", username, ipaddr);
eventlog(eventlog_level_error, __FUNCTION__,
         "Failed to open file: {}", strerror(errno));
```

### 9. String and Text Processing

#### Safe String (`xstring.h/cpp`)
Safe string operations avoiding buffer overflows.

**Functions**:
```cpp
char* xstrdup(const char* str);
int xstrcpy(char* dest, const char* src, size_t size);
int xstrcat(char* dest, const char* src, size_t size);
```

#### String Utilities (`xstr.h/cpp`, `lstr.h`)
String manipulation and length-prefixed strings.

#### Formatted Printing (`asnprintf.h/cpp`)
Safe snprintf with automatic memory allocation.

#### Token Parsing (`token.h/cpp`)
Tokenize strings for command parsing.

**Functions**:
```cpp
char* token_first(char* str, char delim);
char* token_next(char* str, char delim);
```

### 10. Utility Functions

#### General Utilities (`util.h/cpp`)
Miscellaneous helper functions.

**String Operations**:
```cpp
int strstart(char const* full, char const* part);
char* file_get_line(std::FILE* fp);
char* strreverse(char* str);
char* str_skip_space(char* str);
char* str_skip_word(char* str);
```

**Conversions**:
```cpp
int str_to_uint(char const* str, unsigned int* num);
int str_to_ushort(char const* str, unsigned short* num);
int str_get_bool(char const* str);
int clockstr_to_seconds(char const* clockstr, unsigned int* totsecs);
int timestr_to_time(char const* timestr, std::time_t* ptime);
```

**Formatting**:
```cpp
char const* seconds_to_timestr(unsigned int totsecs);
char* escape_fs_chars(char const* in, unsigned int len);
char* escape_chars(char const* in, unsigned int len);
char* unescape_chars(char const* in);
void str_to_hex(char* target, char const* data, int datalen);
int hex_to_str(char const* source, char* data, int datalen);
```

#### Memory Allocation (`xalloc.h/cpp`)
Safe memory allocation with error handling.

**Functions**:
```cpp
void* xmalloc(size_t size);
void* xrealloc(void* ptr, size_t size);
void* xcalloc(size_t nmemb, size_t size);
void xfree(void* ptr);
```

**Behavior**: Like standard malloc/free but never returns NULL; calls error handler on failure.

#### Hex Dumping (`hexdump.h/cpp`)
Format binary data as hexadecimal with ASCII representation.

**Function**:
```cpp
void hexdump(std::FILE* stream, void const* data, unsigned int len);
```

**Output Format**:
```
0000: 01 02 03 04 05 06 07 08  09 0A 0B 0C 0D 0E 0F 10  ................
0010: 11 12 13 14 15 16 17 18  19 1A 1B 1C 1D 1E 1F 20  ...............
```

### 11. System Operations

#### Resource Limits (`rlimit.h/cpp`)
Set system resource limits (open files, memory, CPU).

**Functions**:
```cpp
int rlimit_set_nofile(int limit);
int rlimit_set_memsize(int limit);
int rlimit_set_cputime(int limit);
```

#### Privilege Management (`give_up_root_privileges.h/cpp`)
Safely drop root privileges after binding privileged ports.

**Function**:
```cpp
int give_up_root_privileges(const char* username);
```

**Usage**:
```cpp
// Bind to port 80 (requires root)
bind(sock, ...);
// Drop to unprivileged user
give_up_root_privileges("pvpgn");
```

#### Error Handling (`systemerror.h/cpp`)
System error utilities and errno handling.

### 12. Diablo II Support

#### D2 Character Checksum (`d2char_checksum.h/cpp`)
Validate Diablo II character file checksums.

#### D2 Character File (`d2char_file.h`)
Diablo II `.d2s` character file format structures.

#### D2 Ladder Protocol (`d2cs_d2dbs_ladder.h`)
D2CS to D2DBS ladder communication.

### 13. XML Parsing

#### PugiXML (`pugixml.h/cpp`, `pugiconfig.h`)
Lightweight XML parser for configuration files.

**Use Cases**:
- Advanced configuration files
- Data import/export
- Web service integration

### 14. Advanced Structures

#### Smart Pointers (`scoped_ptr.h`, `scoped_array.h`)
RAII-based automatic resource management (pre-C++11).

**Usage**:
```cpp
scoped_ptr<MyObject> obj(new MyObject());
// Automatically deleted when scope exits
```

#### Hash Tuple (`hash_tuple.hpp`)
Hash function for std::tuple (C++ template utilities).

#### Integer Rotation (`introtate.h`)
Bit rotation macros for hash functions.

### 15. Translation and Localization

#### Translation System (`trans.h/cpp`)
Multi-language support for server messages.

**Functions**:
```cpp
char const* trans_string(char const* tag, const char* def);
int trans_load_from_file(const char* filename);
```

#### Peer Chat (`peerchat.h/cpp`)
IRC-style peer communication protocol.

### 16. Platform Abstraction

#### Setup Headers (`setup_before.h`, `setup_after.h`)
Configure compiler-specific settings and feature detection.

**Purpose**:
- Detect platform capabilities
- Enable/disable features based on availability
- Handle compiler differences (MSVC, GCC, Clang)
- Include order management

#### Program Information (`proginfo.h/cpp`)
Store program name and version for error messages.

#### GUI Support (`gui_printf.h/cpp`)
Printf-like output for Windows GUI applications.

### 17. Miscellaneous

#### Tracking Protocol (`tracker.h`)
Server tracker packet definitions (see bntrackd).

#### RCM (`rcm.h/cpp`)
Reliable Connection Management (connection state tracking).

#### Flags (`flags.h`)
Bit flag manipulation utilities.

## Building

The common library is built as a static library:

```cmake
# CMakeLists.txt
add_library(common STATIC ${COMMON_SOURCES})
target_link_libraries(common PRIVATE fmt)
```

All PvPGN components link against this library:

```cmake
target_link_libraries(bnetd PRIVATE common compat fmt ${NETWORK_LIBRARIES})
target_link_libraries(d2cs PRIVATE common compat fmt ${NETWORK_LIBRARIES})
target_link_libraries(bnchat PRIVATE common compat fmt ${NETWORK_LIBRARIES})
```

## Dependencies

**Internal**:
- Self-contained (no internal dependencies outside common/)

**External**:
- **fmt**: Modern C++ formatting library (replaces sprintf/printf)
- **Standard C++ Library**: C++11 or later
- **Platform Libraries**: System sockets, threading (varies by OS)

## Design Patterns

### 1. Opaque Pointers
Many structures use opaque pointers to hide implementation details:

```cpp
// Public header
typedef struct list t_list;  // Forward declaration

// Internal header (LIST_INTERNAL_ACCESS)
struct list {
    unsigned int len;
    t_elem* head;
    t_elem* tail;
};
```

**Benefits**:
- Binary compatibility
- Implementation flexibility
- Reduced compilation dependencies

### 2. Reference Counting
Packets use reference counting for safe sharing:

```cpp
t_packet* packet = packet_create(...);
packet_add_ref(packet);  // Increment
queue_push_packet(queue, packet);
packet_del_ref(packet);  // Decrement, frees when 0
```

### 3. Factory Pattern
Creation functions allocate and initialize:

```cpp
t_list* list_create();      // Allocates and initializes
int list_destroy(t_list*);  // Cleanup and deallocation
```

### 4. Callback Functions
Event-driven architecture with function pointers:

```cpp
int fdwatch_add_fd(int fd, unsigned rw, 
                   fdwatch_handler callback, void* data);
```

### 5. RAII (Resource Acquisition Is Initialization)
Smart pointers ensure cleanup:

```cpp
scoped_ptr<Resource> res(acquire_resource());
// Automatically released on scope exit
```

## Performance Considerations

### Memory Management
- **Packet Pool**: Reuse packet structures to reduce allocations
- **Reference Counting**: Avoid unnecessary copies
- **Ring Buffers**: Efficient queue implementation
- **Hash Tables**: O(1) lookups for large datasets

### I/O Multiplexing
- **epoll (Linux)**: Best performance, O(1) scalability
- **kqueue (BSD)**: Excellent performance, O(1) scalability  
- **poll**: Good performance, O(n) scalability
- **select**: Fallback, limited to 1024 connections

### Network Optimization
- **Non-blocking I/O**: Prevent thread blocking
- **Partial Read/Write**: Handle EAGAIN/EWOULDBLOCK gracefully
- **Buffering**: Queue packets to reduce syscalls
- **Zero-Copy**: Direct packet access without memcpy where possible

## Thread Safety

**Warning**: The common library is **NOT thread-safe** by default.

**Thread-Safe Components**:
- None (single-threaded design)

**Unsafe Components**:
- All data structures (list, hashtable, queue)
- Packet system
- Event logging (shared FILE* pointer)

**Recommendations**:
- Use separate data structures per thread
- Protect shared state with mutexes
- Use message passing between threads
- Consider lock-free data structures for high-performance scenarios

## Error Handling

### Return Value Conventions

**Success/Failure**:
- `0`: Success
- `-1`: Failure
- `errno` set on system call failures

**Packet I/O**:
- `-1`: Error (connection closed)
- `0`: Partial operation (continue)
- `1`: Complete operation

**Pointer Functions**:
- Valid pointer: Success
- `NULL`: Failure

### Fatal Errors
Functions like `xmalloc()` never return NULL:
- Call `eventlog(eventlog_level_fatal, ...)` on failure
- Exit program with error status

### Non-Fatal Errors
Most functions log errors but continue:
```cpp
if (operation() < 0) {
    eventlog(eventlog_level_error, __FUNCTION__, "Operation failed");
    return -1;  // Propagate error
}
```

## Coding Conventions

### Naming
- **Functions**: `module_action()` (e.g., `list_create()`)
- **Types**: `t_typename` (e.g., `t_packet`)
- **Macros**: `UPPER_CASE` (e.g., `MAX_PACKET_SIZE`)
- **Constants**: `module_CONSTANT` (e.g., `CLIENT_MESSAGE`)

### Header Guards
```cpp
#ifndef INCLUDED_MODULE_TYPES
#define INCLUDED_MODULE_TYPES
// Type definitions
#endif

#ifndef JUST_NEED_TYPES
#ifndef INCLUDED_MODULE_PROTOS
#define INCLUDED_MODULE_PROTOS
// Function prototypes
#endif
#endif
```

### Namespaces
All code in `namespace pvpgn`:
```cpp
namespace pvpgn {
    // Declarations
}
```

### Packed Structures
Protocol structures use `PACKED_ATTR()`:
```cpp
typedef struct {
    bn_short type;
    bn_short size;
} PACKED_ATTR() t_bnet_header;
```

## Testing

### Debug Modes
Enable debug features during development:
```cmake
cmake -DCLIENTDEBUG=ON ..
make
```

**Debug Features**:
- List traversal validation
- Hashtable consistency checks
- Reference count tracking
- Memory leak detection

### Hex Dumping
Debug protocol issues:
```cpp
eventlog_hexdump_data(packet_data, packet_size);
```

### Assertions
Critical invariants:
```cpp
assert(packet != NULL);
assert(list_get_length(list) > 0);
```

## Common Usage Patterns

### Connection Handling
```cpp
// Accept connection
int client_sock = accept(listen_sock, ...);

// Register with fdwatch
fdwatch_add_fd(client_sock, fdwatch_type_read, 
               on_client_readable, conn_data);

// Create packet queue
t_queue* outqueue = NULL;

// Send packet
t_packet* packet = packet_create(packet_class_bnet);
// ... populate packet
queue_push_packet(&outqueue, packet);
packet_del_ref(packet);

// On writable
t_packet* out_packet = queue_pull_packet(&outqueue);
unsigned int sent = 0;
if (net_send_packet(client_sock, out_packet, &sent) == 1) {
    packet_del_ref(out_packet);
}
```

### Account Lookup
```cpp
t_hashtable* accounts = hashtable_create(1024);

// Insert
unsigned int hash = account_get_hash(username);
hashtable_insert_data(accounts, account, hash);

// Lookup
t_entry* entry = hashtable_get_entry_by_data(accounts, username, hash);
if (entry) {
    t_account* account = entry_get_data(entry);
    // Use account
}
```

### Configuration Loading
```cpp
static t_conf_entry conf_table[] = {
    { "port", conf_set_port, conf_get_port, NULL },
    { "logfile", conf_set_logfile, conf_get_logfile, NULL }
};

conf_load_file("bnetd.conf", conf_table, 
               sizeof(conf_table)/sizeof(t_conf_entry));
```

## Future Enhancements

Potential improvements for the common library:

1. **Thread Safety**: Add mutex protection for shared data structures
2. **C++11 Features**: Replace custom smart pointers with std::unique_ptr
3. **Async I/O**: Add io_uring support (Linux)
4. **Memory Pools**: Reduce allocator overhead
5. **JSON Support**: Replace pugixml with modern JSON parser
6. **IPv6**: Complete IPv6 support throughout
7. **TLS/SSL**: Add encrypted connection support
8. **Compression**: Packet compression for bandwidth reduction
9. **Metrics**: Built-in performance monitoring
10. **Unit Tests**: Comprehensive test suite

## Related Components

The common library is used by:

- **bnetd**: Main Battle.net server (`src/bnetd`)
- **d2cs**: Diablo II character server (`src/d2cs`)
- **d2dbs**: Diablo II database server (`src/d2dbs`)
- **bntrackd**: Server tracker daemon (`src/bntrackd`)
- **Client Tools**: bnchat, bnftp, bnstat, bnbot (`src/client`)
- **bnproxy**: Network proxy (`src/bnproxy`)

## Documentation

- **Protocol Specs**: Extensive comments in `*_protocol.h` files
- **Man Pages**: Some utilities have man pages in `man/` directory
- **Code Comments**: Inline documentation throughout source

## Authors

The common library has been developed by numerous contributors:
- Mark Baysinger (mbaysing@ucsd.edu) - Original packet system
- Ross Combs (rocombs@cs.nmsu.edu, ross@bnetd.org) - Core infrastructure
- Dizzy - Configuration system
- Many others (see individual file headers)

## License

GNU General Public License v2 or later (see `LICENSE` file in project root).

---

*This README documents the PvPGN common library. Last updated: November 2024*
