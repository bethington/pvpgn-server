# PvPGN Compatibility Library

## Overview

The `src/compat` folder contains the **portability and compatibility layer** for the PvPGN server project. This library provides a uniform programming interface across multiple operating systems (Windows, Linux, Unix, BSD, macOS) and compilers by implementing missing POSIX functions, wrapping platform-specific APIs, and providing fallback implementations for systems lacking modern features.

## Purpose

The compat library serves as:

1. **Cross-Platform Abstraction**: Hide differences between Windows, Unix, and BSD systems
2. **POSIX Emulation**: Implement missing POSIX functions on non-POSIX systems (primarily Windows)
3. **Legacy System Support**: Provide modern functions for older compilers/systems
4. **API Normalization**: Present consistent interfaces regardless of underlying implementation
5. **Build Portability**: Enable compilation on diverse platforms without code duplication

## Architecture

The compatibility layer is implemented as a **static library** that conditionally compiles implementations based on detected system capabilities. CMake's feature detection (`ConfigureChecks.cmake`) determines which functions/headers are available, and the compat library provides substitutes for missing ones.

### Detection Strategy

```cmake
# CMake checks for function availability
check_function_exists(strcasecmp HAVE_STRCASECMP)
check_function_exists(gettimeofday HAVE_GETTIMEOFDAY)
check_include_file_cxx(sys/utsname.h HAVE_SYS_UTSNAME_H)
```

**Conditional Compilation**:
```cpp
#ifndef HAVE_STRCASECMP
    // Provide custom implementation
    int strcasecmp(const char* s1, const char* s2);
#endif
```

## Component Categories

### 1. Socket and Network Abstraction

#### Socket Operations (`psock.h/cpp`)
Unified socket interface hiding Windows Winsock vs. Unix sockets differences.

**Key Problems Solved**:
- Windows uses `closesocket()`, Unix uses `close()`
- Windows uses `ioctlsocket()`, Unix uses `fcntl()` or `ioctl()`
- Different error codes and initialization requirements
- Non-blocking I/O configuration differs

**API**:
```cpp
// Initialization (Windows only, no-op on Unix)
int psock_init(void);      // Call WSAStartup() on Windows
int psock_deinit(void);    // Call WSACleanup() on Windows

// Set socket to non-blocking mode
int psock_ctl(int sd, int mode);  // Uses ioctlsocket() or fcntl()

// Cross-platform error checking
int psock_errno(void);     // Returns WSAGetLastError() or errno

// Socket close
int psock_close(int sock); // Uses closesocket() or close()

// Socket selection
int psock_select(int n, fd_set* readfds, fd_set* writefds, 
                 fd_set* exceptfds, struct timeval* timeout);
```

**Constants**:
```cpp
// Protocol families
PSOCK_PF_INET       // PF_INET

// Socket types
PSOCK_SOCK_STREAM   // TCP
PSOCK_SOCK_DGRAM    // UDP

// Socket options
PSOCK_SOL_SOCKET    // SOL_SOCKET
PSOCK_SO_REUSEADDR  // SO_REUSEADDR
PSOCK_SO_KEEPALIVE  // SO_KEEPALIVE

// Control flags
PSOCK_NONBLOCK      // FIONBIO or O_NONBLOCK

// Error codes
PSOCK_EWOULDBLOCK   // WSAEWOULDBLOCK or EWOULDBLOCK
PSOCK_EINPROGRESS   // WSAEINPROGRESS or EINPROGRESS
PSOCK_ECONNRESET    // WSAECONNRESET or ECONNRESET
```

**Implementation Details**:

*Windows (Winsock2)*:
```cpp
int psock_init(void) {
    WSADATA wsaData;
    return WSAStartup(MAKEWORD(2, 2), &wsaData);
}

int psock_ctl(int sd, int mode) {
    unsigned long nParam = 1;
    return ioctlsocket(sd, mode, &nParam);
}
```

*Unix (POSIX)*:
```cpp
int psock_init(void) {
    return 0;  // No-op
}

int psock_ctl(int sd, long int mode) {
    long int oldmode = fcntl(sd, F_GETFL);
    return fcntl(sd, F_SETFL, oldmode | mode);
}
```

**Usage Example**:
```cpp
#include "compat/psock.h"

// Initialize sockets (Windows)
psock_init();

// Create socket
int sock = socket(PSOCK_PF_INET, PSOCK_SOCK_STREAM, PSOCK_IPPROTO_TCP);

// Set non-blocking
psock_ctl(sock, PSOCK_NONBLOCK);

// Check for errors
if (errno == PSOCK_EWOULDBLOCK) {
    // Would block, retry later
}

// Close socket
psock_close(sock);

// Cleanup (Windows)
psock_deinit();
```

#### Socket Headers (`socket.h`)
Single header that includes appropriate socket headers for the platform.

**Windows**: Includes `<winsock2.h>`  
**Unix**: Includes `<sys/socket.h>`, `<netinet/in.h>`, `<arpa/inet.h>`, `<netdb.h>`

#### Network Constants (`netinet_in.h`)
Defines missing network constants.

```cpp
#ifndef INADDR_LOOPBACK
#define INADDR_LOOPBACK 0x7f000001  // 127.0.0.1
#endif

#ifndef INADDR_ANY
#define INADDR_ANY 0x00000000       // 0.0.0.0
#endif
```

#### Socket I/O (`send.h`, `recv.h`)
Ensure `send()` and `recv()` are available.

**Fallback**: Use `sendto()` and `recvfrom()` with NULL address if `send()`/`recv()` not available.

```cpp
#ifndef HAVE_SEND
# define send(s, b, l, f) sendto(s, b, l, f, NULL, NULL)
#endif

#ifndef HAVE_RECV
# define recv(s, b, l, f) recvfrom(s, b, l, f, NULL, NULL)
#endif
```

### 2. String Functions

#### Case-Insensitive Comparison

**`strcasecmp.h/cpp`** - Compare strings ignoring case (POSIX function missing on some systems).

**Implementation**:
```cpp
int strcasecmp(char const* str1, char const* str2) {
    for (int i = 0; str1[i] && str2[i]; i++) {
        int a = isupper(str1[i]) ? tolower(str1[i]) : str1[i];
        int b = isupper(str2[i]) ? tolower(str2[i]) : str2[i];
        if (a < b) return -1;
        if (a > b) return +1;
    }
    return (str1[i] ? -1 : (str2[i] ? +1 : 0));
}
```

**Fallback Strategy**:
```cpp
#ifdef HAVE_STRICMP
# define strcasecmp(s1, s2) stricmp(s1, s2)  // Use MSVC's stricmp()
#else
  // Custom implementation
#endif
```

**`strncasecmp.h/cpp`** - Compare first N characters ignoring case.

Similar to `strcasecmp()` but with length limit:
```cpp
int strncasecmp(char const* str1, char const* str2, size_t len);
```

**Fallback**: Uses `_strnicmp()` on MSVC.

#### String Duplication

**`strdup.h/cpp`** - Duplicate a string (POSIX function).

```cpp
char* strdup(const char* str) {
    char* copy = (char*)malloc(strlen(str) + 1);
    if (copy) strcpy(copy, str);
    return copy;
}
```

**Note**: Returns `NULL` on allocation failure. Caller must `free()` the result.

#### Error Messages

**`strerror.h/cpp`** - Convert errno to error string (C89 standard, but may be missing).

```cpp
char const* strerror(int errornum);
```

**Fallback**: Uses static buffer with `sprintf()` if `strerror()` unavailable.

#### String Token Separator

**`strsep.h/cpp`** - Extract tokens from string (BSD function, missing on some systems).

```cpp
char* strsep(char** stringp, char const* delim);
```

**Behavior**:
- Modifies `*stringp` to point past the delimiter
- Returns pointer to extracted token
- Returns `NULL` when no more tokens

**Usage**:
```cpp
char* str = strdup("foo:bar:baz");
char* token;
while ((token = strsep(&str, ":")) != NULL) {
    printf("%s\n", token);
}
```

### 3. Time Functions

#### High-Resolution Time (`gettimeofday.h/cpp`)
Get current time with microsecond precision (POSIX function missing on Windows).

**Structure**:
```cpp
struct timeval {
    long tv_sec;   // Seconds
    long tv_usec;  // Microseconds
};

struct timezone {
    int tz_minuteswest;  // Minutes west of GMT
    int tz_dsttime;      // Daylight saving time type
};
```

**Function**:
```cpp
int gettimeofday(struct timeval* tv, struct timezone* tz);
```

**Windows Implementation**:
Uses `GetSystemTimeAsFileTime()` and converts Windows FILETIME to Unix timeval:
```cpp
int gettimeofday(struct timeval* tv, struct timezone* tz) {
    FILETIME ft;
    GetSystemTimeAsFileTime(&ft);
    
    // Convert 100-nanosecond intervals to microseconds
    unsigned __int64 tmpres = 0;
    tmpres |= ft.dwHighDateTime;
    tmpres <<= 32;
    tmpres |= ft.dwLowDateTime;
    
    tmpres /= 10;  // Convert to microseconds
    tmpres -= DELTA_EPOCH_IN_MICROSECS;  // Adjust epoch
    
    tv->tv_sec = (long)(tmpres / 1000000UL);
    tv->tv_usec = (long)(tmpres % 1000000UL);
    
    return 0;
}
```

### 4. System Information

#### System Name (`uname.h/cpp`)
Get system information (POSIX function missing on Windows).

**Structure**:
```cpp
struct utsname {
    char sysname[SYS_NMLN + 1];    // OS name (e.g., "Windows NT")
    char nodename[SYS_NMLN + 1];   // Network node hostname
    char release[SYS_NMLN + 1];    // OS release (e.g., "10.0")
    char version[SYS_NMLN + 1];    // OS version string
    char machine[SYS_NMLN + 1];    // Hardware type (e.g., "x86_64")
    char domainname[SYS_NMLN + 1]; // Domain name (NIS/YP)
};
```

**Function**:
```cpp
int uname(struct utsname* buf);
```

**Windows Implementation**:
Uses `RtlGetVersion()` from ntdll.dll to detect Windows version:
```cpp
// Detects: Windows 10, 8.1, 8, 7, Vista, XP, Server editions
// Returns architecture: x86, x64, ARM, ARM64
// Example output:
//   sysname: "Windows NT"
//   release: "10.0"
//   version: "Windows 10 Enterprise"
//   machine: "x86_64"
```

**Unix**: Passes through to system `uname()`.

#### Process ID (`pgetpid.h`)
Get process ID (POSIX function, location varies by platform).

**Unix**: `<unistd.h>` - `getpid()`  
**Windows**: `<process.h>` - `_getpid()` or `GetCurrentProcessId()`

**Provides**: Correct header includes for `getpid()`.

#### Host Name (`gethostname.h`)
Get local hostname (POSIX function, different headers).

**Unix**: `<unistd.h>` - `gethostname()`  
**Windows**: `<winsock2.h>` - `gethostname()`

### 5. Directory Operations

#### Directory Enumeration (`pdir.h/cpp`)
C++ RAII wrapper for directory traversal.

**Hides**:
- Unix: `opendir()`, `readdir()`, `closedir()`
- Windows: `_findfirst()`, `_findnext()`, `_findclose()`

**Class Definition**:
```cpp
namespace pvpgn {

class Directory {
public:
    class OpenError : public std::runtime_error { };
    
    // Constructor - opens directory
    Directory(const std::string& path, bool lazyread = false);
    
    // Destructor - closes directory
    ~Directory() throw();
    
    // Iteration
    std::string read();      // Read next entry
    void rewind();           // Reset to beginning
    
    // Query
    bool empty() const;      // Check if directory is empty
    
    // Get all entries
    std::vector<std::string> getFilenames();
    
private:
    std::string path;
    bool lazyread;
    
#ifdef WIN32
    long lFindHandle;
    struct _finddata_t fileinfo;
    int status;
#else
    DIR* dir;
#endif
};

}
```

**Usage Example**:
```cpp
try {
    pvpgn::Directory dir("conf/");
    std::string filename;
    
    while (!(filename = dir.read()).empty()) {
        if (filename != "." && filename != "..") {
            std::cout << filename << std::endl;
        }
    }
} catch (pvpgn::Directory::OpenError& e) {
    std::cerr << "Cannot open directory: " << e.what() << std::endl;
}
```

**Alternatives**:
```cpp
// Get all filenames at once
std::vector<std::string> files = dir.getFilenames();
```

#### Directory Creation (`mkdir.h`)
Create directory (POSIX function with varying signatures).

**Unix**: `int mkdir(const char* path, mode_t mode)`  
**Windows**: `int mkdir(const char* path)` or `int _mkdir(const char* path)`

**Macro**:
```cpp
#ifdef MKDIR_TAKES_ONE_ARG
# define p_mkdir(path, mode) mkdir(path)    // Windows
#else
# define p_mkdir(path, mode) mkdir(path, mode)  // Unix
#endif
```

**Usage**:
```cpp
p_mkdir("var/userdata", 0755);  // Works on all platforms
```

### 6. File System Operations

#### File Access (`access.h`)
Check file accessibility (POSIX function, different headers).

**Unix**: `<unistd.h>` - `access(path, mode)`  
**Windows**: `<io.h>` - `_access(path, mode)`

**Constants**:
```cpp
#define R_OK 4  // Test read permission
#define W_OK 2  // Test write permission
#define X_OK 1  // Test execute permission
#define F_OK 0  // Test existence
```

**Usage**:
```cpp
if (access("config.conf", R_OK) == 0) {
    // File exists and is readable
}
```

#### File Rename (`rename.h`)
Rename file (standard C function, but included for completeness).

#### File Read (`read.h`)
Include appropriate header for `read()` function.

**Unix**: `<unistd.h>`  
**Windows**: `<io.h>`

#### Standard File Descriptors (`stdfileno.h`)
Define stdin/stdout/stderr file descriptor numbers.

```cpp
#define STDINFD  0  // STDIN_FILENO
#define STDOUTFD 1  // STDOUT_FILENO
#define STDERRFD 2  // STDERR_FILENO
```

#### File Stat Macros (`statmacros.h`)
Portable macros for checking file types from `stat()` results.

**Macros**:
```cpp
S_ISREG(mode)   // Is regular file?
S_ISDIR(mode)   // Is directory?
S_ISCHR(mode)   // Is character device?
S_ISBLK(mode)   // Is block device?
S_ISFIFO(mode)  // Is FIFO/pipe?
S_ISLNK(mode)   // Is symbolic link?
S_ISSOCK(mode)  // Is socket?
```

**Purpose**: Some systems have broken `S_IS*` macros; this header provides working versions.

#### Memory-Mapped Files (`mmap.h/cpp`)
Map files into memory (POSIX function missing on Windows and some systems).

**Function**:
```cpp
void* pmmap(void* addr, unsigned len, int prot, int flags, 
            int fd, unsigned offset);
int pmunmap(void* addr, unsigned len);
```

**Constants**:
```cpp
PROT_NONE   // No access
PROT_READ   // Read access
PROT_WRITE  // Write access
PROT_EXEC   // Execute access

MAP_SHARED  // Share changes
MAP_PRIVATE // Private copy-on-write
MAP_FAILED  // Return value on error
```

**Implementations**:

*Unix with mmap()*:
```cpp
#define pmmap(a,b,c,d,e,f) mmap(a,b,c,d,e,f)
#define pmunmap(a,b) munmap(a,b)
```

*Windows (CreateFileMapping)*:
```cpp
void* pmmap(void* addr, unsigned len, int prot, int flags, 
            int fd, unsigned offset) {
    HANDLE hFile = (HANDLE)_get_osfhandle(fd);
    HANDLE hMapping = CreateFileMapping(hFile, NULL, PAGE_READONLY, 0, 0, NULL);
    return MapViewOfFile(hMapping, FILE_MAP_READ, 0, 0, 0);
}
```

*Fallback (no mmap)*:
```cpp
void* pmmap(...) {
    void* mem = malloc(len);
    read(fd, mem, len);  // Read entire file into memory
    return mem;
}
```

**Usage**:
```cpp
int fd = open("data.bin", O_RDONLY);
void* data = pmmap(NULL, filesize, PROT_READ, MAP_PRIVATE, fd, 0);
// Use data...
pmunmap(data, filesize);
close(fd);
```

### 7. Command-Line Parsing

#### GNU getopt (`pgetopt.h/cpp`)
Full implementation of GNU getopt_long for systems lacking it.

**Functions**:
```cpp
int getopt(int argc, char** argv, const char* optstring);
int getopt_long(int argc, char** argv, const char* optstring,
                const struct option* longopts, int* longindex);
```

**Globals**:
```cpp
extern char* optarg;   // Option argument
extern int optind;     // Next argv index
extern int opterr;     // Print errors?
extern int optopt;     // Last processed option
```

**Long Option Structure**:
```cpp
struct option {
    const char* name;      // Long option name
    int has_arg;           // no_argument(0), required_argument(1), optional_argument(2)
    int* flag;             // Where to store value (NULL = return val)
    int val;               // Value to return/store
};
```

**Usage Example**:
```cpp
static struct option long_options[] = {
    {"help",    no_argument,       NULL, 'h'},
    {"version", no_argument,       NULL, 'v'},
    {"config",  required_argument, NULL, 'c'},
    {NULL, 0, NULL, 0}
};

int c;
while ((c = getopt_long(argc, argv, "hvc:", long_options, NULL)) != -1) {
    switch (c) {
        case 'h': print_help(); break;
        case 'v': print_version(); break;
        case 'c': config_file = optarg; break;
    }
}
```

**Fallback**: Only compiled if system lacks `getopt()`.

### 8. Terminal I/O

#### Terminal Control (`termios.h`)
Terminal I/O control for systems without termios (primarily Windows).

**Structure**:
```cpp
struct termios {
    int c_lflag;    // Local flags
    int c_cc[1];    // Control characters
};
```

**Constants**:
```cpp
ECHO    // Echo input characters
ICANON  // Canonical mode (line buffering)
VMIN    // Minimum characters to read
VTIME   // Timeout value
```

**Functions (stubs on non-termios systems)**:
```cpp
int tcgetattr(int fd, struct termios* termios_p);
int tcsetattr(int fd, int optional_actions, const struct termios* termios_p);
```

**Purpose**: Allow code using termios to compile on Windows (functionality disabled).

### 9. Runtime Library Loading

#### Dynamic Library Loading (`runtime_libs.h`)
Unified interface for loading shared libraries at runtime.

**API Abstraction**:

*Windows*:
```cpp
#define OpenLibrary(l)     LoadLibrary(l)
#define GetFunction(h, f)  GetProcAddress((HINSTANCE)h, f)
#define CloseLibrary(h)    FreeLibrary((HINSTANCE)h)
```

*Unix (POSIX dl)*:
```cpp
#define OpenLibrary(l)     dlopen(l, RTLD_LOCAL | RTLD_LAZY)
#define GetFunction(h, f)  dlsym(h, f)
#define CloseLibrary(h)    dlclose(h)
```

**Library Names**:
```cpp
// Database drivers
#define MYSQL_LIB    "libmysql.dll" / "libmysqlclient.so"
#define PGSQL_LIB    "libpq.dll" / "libpq.so"
#define SQLITE3_LIB  "sqlite3.dll" / "libsqlite3.so"
#define ODBC_LIB     "odbc32.dll" / "libodbc.so"
```

**Usage**:
```cpp
void* handle = OpenLibrary(MYSQL_LIB);
if (handle) {
    mysql_init_func init = (mysql_init_func)GetFunction(handle, "mysql_init");
    if (init) {
        // Use function
    }
    CloseLibrary(handle);
}
```

**Purpose**: Load database drivers dynamically to avoid hard dependencies.

## Build Integration

The compat library is built as a **static library** linked by all PvPGN components:

```cmake
# src/compat/CMakeLists.txt
add_library(compat STATIC ${COMPAT_SOURCES})
target_link_libraries(compat PRIVATE common fmt)
```

**Linked by**:
```cmake
target_link_libraries(bnetd PRIVATE compat common fmt ${NETWORK_LIBRARIES})
target_link_libraries(d2cs PRIVATE compat common fmt ${NETWORK_LIBRARIES})
target_link_libraries(bnchat PRIVATE compat common fmt ${NETWORK_LIBRARIES})
```

## Feature Detection

The build system (`ConfigureChecks.cmake`) detects available features:

### Headers Checked
```cmake
check_include_file_cxx(arpa/inet.h HAVE_ARPA_INET_H)
check_include_file_cxx(dirent.h HAVE_DIRENT_H)
check_include_file_cxx(sys/socket.h HAVE_SYS_SOCKET_H)
check_include_file_cxx(sys/utsname.h HAVE_SYS_UTSNAME_H)
check_include_file_cxx(termios.h HAVE_TERMIOS_H)
check_include_file_cxx(winsock2.h HAVE_WINSOCK2_H)
check_include_file_cxx(unistd.h HAVE_UNISTD_H)
```

### Functions Checked
```cmake
check_function_exists(strcasecmp HAVE_STRCASECMP)
check_function_exists(gettimeofday HAVE_GETTIMEOFDAY)
check_function_exists(getopt HAVE_GETOPT)
check_function_exists(mkdir HAVE_MKDIR)
check_function_exists(mmap HAVE_MMAP)
check_function_exists(uname HAVE_UNAME)
```

### Generated Config
Results written to `config.h`:
```cpp
#define HAVE_STRCASECMP 1
#define HAVE_GETTIMEOFDAY 1
#undef HAVE_UNAME        // Not available on Windows
```

## Platform Support Matrix

| Feature | Windows | Linux | BSD | macOS | Notes |
|---------|---------|-------|-----|-------|-------|
| **Sockets** | Winsock2 | BSD | BSD | BSD | Full abstraction via psock |
| **Non-blocking I/O** | ioctlsocket | fcntl | fcntl | fcntl | psock_ctl() |
| **strcasecmp** | stricmp | native | native | native | Case-insensitive compare |
| **gettimeofday** | Emulated | native | native | native | Microsecond precision |
| **uname** | Emulated | native | native | native | System information |
| **mmap** | CreateFileMapping | native | native | native | Memory-mapped files |
| **getopt_long** | Custom | native | native | native | GNU long options |
| **Directory** | _findfirst | readdir | readdir | readdir | C++ wrapper |
| **termios** | Stub | native | native | native | Terminal control |
| **dlopen** | LoadLibrary | native | native | native | Dynamic loading |

## Design Patterns

### 1. Conditional Compilation
Use feature detection macros to include/exclude code:
```cpp
#ifndef HAVE_STRCASECMP
    // Provide implementation
    int strcasecmp(const char* s1, const char* s2) { ... }
#endif
```

### 2. Platform-Specific Implementations
Same header, different implementations:
```cpp
#ifdef WIN32
    int psock_ctl(int sd, int mode) {
        return ioctlsocket(sd, mode, &param);
    }
#else
    int psock_ctl(int sd, long int mode) {
        return fcntl(sd, F_SETFL, mode);
    }
#endif
```

### 3. Macro Redirection
Map to platform-specific names:
```cpp
#ifdef HAVE_STRICMP
# define strcasecmp(s1, s2) stricmp(s1, s2)
#endif
```

### 4. RAII Wrappers
C++ classes for resource management:
```cpp
class Directory {
    ~Directory() { close(); }  // Automatic cleanup
};
```

### 5. Fallback Implementations
Provide reduced functionality when feature unavailable:
```cpp
#ifndef HAVE_MMAP
    void* pmmap(...) {
        // Read entire file into malloc'd buffer
        void* mem = malloc(len);
        read(fd, mem, len);
        return mem;
    }
#endif
```

## Usage Guidelines

### Including Compat Headers

**Always include after system headers**:
```cpp
// Correct order:
#include "common/setup_before.h"  // Platform detection
#include <string.h>
#include <unistd.h>
#include "compat/strcasecmp.h"    // Compat layer
#include "common/setup_after.h"   // Cleanup
```

### Socket Programming

**Use psock wrappers**:
```cpp
// Initialize
psock_init();

// Create socket
int sock = socket(PSOCK_PF_INET, PSOCK_SOCK_STREAM, 0);

// Set non-blocking
psock_ctl(sock, PSOCK_NONBLOCK);

// Check errors
if (psock_errno() == PSOCK_EWOULDBLOCK) {
    // Retry
}

// Close
psock_close(sock);

// Cleanup
psock_deinit();
```

### String Operations

**Use compat string functions**:
```cpp
#include "compat/strcasecmp.h"

if (strcasecmp(str1, str2) == 0) {
    // Strings equal (case-insensitive)
}
```

### Directory Traversal

**Use Directory class**:
```cpp
#include "compat/pdir.h"

try {
    pvpgn::Directory dir("conf/");
    std::vector<std::string> files = dir.getFilenames();
    
    for (const auto& file : files) {
        process_file(file);
    }
} catch (pvpgn::Directory::OpenError&) {
    // Handle error
}
```

### File System

**Use portable macros**:
```cpp
#include "compat/mkdir.h"
#include "compat/access.h"

// Create directory (mode ignored on Windows)
p_mkdir("var/logs", 0755);

// Check file access
if (access("config.conf", R_OK) == 0) {
    // Readable
}
```

### Memory Mapping

**Use pmmap wrapper**:
```cpp
#include "compat/mmap.h"

void* data = pmmap(NULL, size, PROT_READ, MAP_PRIVATE, fd, 0);
if (data != MAP_FAILED) {
    // Use data
    pmunmap(data, size);
}
```

## Common Pitfalls

### 1. Socket Initialization
**Problem**: Forgetting to initialize Winsock on Windows.  
**Solution**: Always call `psock_init()` at program start and `psock_deinit()` at exit.

### 2. Error Code Differences
**Problem**: Checking `errno == EWOULDBLOCK` fails on Windows.  
**Solution**: Use `psock_errno() == PSOCK_EWOULDBLOCK`.

### 3. Directory Iteration
**Problem**: Windows `_findfirst` returns first entry immediately, Unix `readdir` requires first call.  
**Solution**: Use `Directory` class which hides the difference.

### 4. File Modes
**Problem**: `mkdir()` takes mode argument on Unix, not on Windows.  
**Solution**: Use `p_mkdir()` macro which ignores mode on Windows.

### 5. Header Include Order
**Problem**: Compat headers included before system headers can cause conflicts.  
**Solution**: Always include system headers first, then compat headers.

## Testing

### Compile-Time Testing
CMake detects features and defines macros:
```bash
cmake -DCMAKE_BUILD_TYPE=Debug ..
# Generates config.h with HAVE_* macros
```

### Cross-Platform Testing
Test on multiple platforms:
- **Windows**: MSVC, MinGW
- **Linux**: GCC, Clang
- **BSD**: FreeBSD, OpenBSD, NetBSD
- **macOS**: Clang

### Fallback Testing
Disable features to test fallback implementations:
```cmake
# Force custom implementation
set(HAVE_STRCASECMP OFF)
```

## Performance Considerations

### Socket Operations
- **psock wrappers**: Minimal overhead (inline functions or macros)
- **Initialization**: One-time cost on Windows (WSAStartup)

### String Functions
- **Custom implementations**: Comparable to system versions
- **Case conversion**: Hand-optimized to avoid `tolower()` on already lowercase

### Memory Mapping
- **Native mmap**: Zero-copy file access
- **Fallback**: Full file read into RAM (memory overhead)

### Directory Enumeration
- **C++ wrapper**: RAII ensures cleanup, minimal overhead
- **Vector allocation**: Batch mode allocates all entries at once

## Future Enhancements

1. **C++17 Filesystem**: Migrate to `std::filesystem` when C++17 adopted
2. **IPv6 Support**: Extend socket abstractions for IPv6
3. **Async I/O**: Add compatibility layer for async I/O (IOCP, io_uring)
4. **UTF-16 Support**: Better Windows Unicode path handling
5. **Modern Compilers**: Remove compatibility for pre-C++11 compilers

## Related Components

The compat library is used by:
- **common**: Shared utility library (`src/common`)
- **bnetd**: Main server daemon (`src/bnetd`)
- **d2cs/d2dbs**: Diablo II servers (`src/d2cs`, `src/d2dbs`)
- **Client tools**: bnchat, bnftp, bnstat, bnbot (`src/client`)
- **bnproxy**: Network proxy (`src/bnproxy`)
- **bntrackd**: Server tracker (`src/bntrackd`)

## Authors

The compatibility library has been developed by:
- **Ross Combs** (rocombs@cs.nmsu.edu, ross@bnetd.org) - Core abstractions
- **Philippe Dubois** (pdubois1@hotmail.com) - Socket layer
- **Dizzy** - Directory, mmap implementations
- **Olaf Freyer** (aaron@cs.tu-berlin.de) - Process ID
- **CreepLord** (creeplord@pvpgn.org) - Runtime loading
- Many others (see individual file headers)

## License

GNU General Public License v2 or later (see `LICENSE` file in project root).

---

*This README documents the PvPGN compatibility layer. The compat library enables PvPGN to run on Windows, Linux, BSD, macOS, and other Unix-like systems by providing a uniform POSIX-like interface.*

**Last updated**: November 2024
