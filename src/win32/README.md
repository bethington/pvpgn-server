# Win32 - Windows Platform Support Layer

## Overview

The `src/win32` folder provides Windows-specific implementations and GUI components for the PvPGN server suite. This module enables PvPGN servers (bnetd, d2cs, d2dbs) to run natively on Windows with full platform integration including:

- **Windows Services**: Install and run servers as background Windows services
- **GUI Interface**: Rich graphical user interface with real-time monitoring and management
- **Console Redirection**: Proper console I/O for Windows applications
- **Crash Handling**: Minidump generation for debugging crashes
- **POSIX Emulation**: Directory API implementation (dirent.h) for Windows
- **Resource Management**: Icons, dialogs, and menus for GUI applications

This layer bridges the gap between the cross-platform PvPGN codebase and Windows-specific requirements, providing seamless integration with the Windows ecosystem.

## Architecture

### Component Structure

```
win32/
├── Service Layer
│   ├── service.h/cpp         - Windows Service Control Manager integration
│   ├── winmain.cpp           - BNETD GUI application (WIN32_GUI)
│   ├── d2cs_winmain.cpp      - D2CS GUI application
│   └── d2dbs_winmain.cpp     - D2DBS GUI application
│
├── Console & I/O
│   ├── console_output.h/cpp  - Console allocation and I/O redirection
│   └── dirent.h              - POSIX directory API for Windows
│
├── Debugging
│   ├── windump.h/cpp         - Crash handler with minidump generation
│   └── winmain.h             - GUI utility functions
│
└── Resources
    ├── resource.h/rc         - BNETD menu, dialogs, icons
    ├── d2cs_resource.h/rc    - D2CS dialogs and icons
    ├── d2dbs_resource.h/rc   - D2DBS dialogs and icons
    ├── console_resource.h/rc - Console window resources
    └── logo01.ico            - Application icon
```

### Integration with PvPGN

```
┌────────────────────────────────────────────────────────┐
│                 PvPGN Server (bnetd/d2cs/d2dbs)        │
├────────────────────────────────────────────────────────┤
│                                                         │
│  main.cpp                                               │
│    │                                                    │
│    ├─► #ifdef WIN32_GUI                                │
│    │     └─► winmain.cpp (GUI Mode)                    │
│    │           ├─► guiThread() - Window creation       │
│    │           ├─► guiWndProc() - Event handling       │
│    │           └─► app_main() - Server logic thread    │
│    │                                                    │
│    ├─► #ifdef WIN32 (--service flag)                   │
│    │     └─► service.cpp                               │
│    │           ├─► Win32_ServiceInstall()              │
│    │           ├─► Win32_ServiceRun()                  │
│    │           └─► ServiceMain() - SCM callback        │
│    │                                                    │
│    └─► Default: Console mode                           │
│          └─► console_output.cpp                        │
│                └─► RedirectIOToConsole()               │
│                                                         │
│  Crash Handler (Always Active on Win32)                │
│    └─► windump.cpp                                     │
│          └─► unhandled_handler() - EXCEPTION_POINTERS  │
│                └─► make_minidump() - .dmp file         │
│                                                         │
│  File System Operations                                │
│    └─► dirent.h                                        │
│          ├─► opendir() - FindFirstFile wrapper         │
│          ├─► readdir() - FindNextFile wrapper          │
│          └─► closedir() - FindClose wrapper            │
│                                                         │
└────────────────────────────────────────────────────────┘
```

## Key Features

### 1. Windows Service Support

Enables PvPGN servers to run as Windows background services with full Service Control Manager (SCM) integration.

**Service API** (`service.h/cpp`)

```cpp
// Install service in SCM database
void Win32_ServiceInstall(void);

// Uninstall service from SCM
void Win32_ServiceUninstall(void);

// Run server as a service
void Win32_ServiceRun(void);
```

**Service Lifecycle**

```
1. Installation:
   sc create PvPGN binPath= "C:\PvPGN\bnetd.exe --service"
   
2. Service Control Manager calls ServiceMain():
   ├─ Initialize SERVICE_STATUS structure
   ├─ Register ServiceControlHandler callback
   ├─ Set working directory to executable path
   ├─ Set status: SERVICE_RUNNING
   └─ Call main() or app_main() to run server

3. Service Control:
   ├─ SERVICE_CONTROL_STOP → Graceful shutdown
   ├─ SERVICE_CONTROL_PAUSE → Pause operations (g_ServiceStatus = 2)
   ├─ SERVICE_CONTROL_CONTINUE → Resume operations
   └─ SERVICE_CONTROL_SHUTDOWN → System shutdown

4. Cleanup:
   ├─ Server main loop exits
   ├─ Set status: SERVICE_STOPPED
   └─ SCM removes service from active list
```

**Service Installation Details**

```cpp
SC_HANDLE service = CreateServiceA(
    serviceControlManager,
    serviceName,              // "PvPGN"
    serviceLongName,          // "PvPGN Battle.net Server"
    SERVICE_ALL_ACCESS,
    SERVICE_WIN32_OWN_PROCESS,
    SERVICE_AUTO_START,       // Start on boot
    SERVICE_ERROR_IGNORE,
    "C:\\PvPGN\\bnetd.exe --service",
    NULL,                     // No load order group
    NULL,                     // No tag identifier
    NULL,                     // No dependencies
    NULL,                     // LocalSystem account
    NULL                      // No password
);

// Set service description (Windows 2000+)
SERVICE_DESCRIPTION sdBuf;
sdBuf.lpDescription = "PvPGN Battle.net emulation server";
ChangeServiceConfig2A(service, SERVICE_CONFIG_DESCRIPTION, &sdBuf);
```

**Global Service Status Variable**

```cpp
extern int g_ServiceStatus;
// 0 = Stopping
// 1 = Running
// 2 = Paused
```

Used by server main loops to check service state and respond to SCM commands.

### 2. Graphical User Interface (WIN32_GUI)

Rich Windows application with real-time server monitoring and administrative controls.

**GUI Features** (`winmain.cpp`, `d2cs_winmain.cpp`, `d2dbs_winmain.cpp`)

```
┌─────────────────────────────────────────────────────────────┐
│ PvPGN Server - Battle.net Emulation                    [_][□][X] │
├─────────────────────────────────────────────────────────────┤
│ File  Server  AdminCommands  View  ServerConfiguration  Help  │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────────────────┬───────────────────────────┐│
│  │ Server Log                  │ Connected Users           ││
│  │                             │                           ││
│  │ [INFO] Server started       │ • Player1 (192.168.1.10)  ││
│  │ [INFO] Listening on 6112    │ • Player2 (192.168.1.15)  ││
│  │ [INFO] Connection from      │ • Player3 (192.168.1.20)  ││
│  │        192.168.1.10         │                           ││
│  │ [INFO] User "Player1"       │ Total: 3 users            ││
│  │        logged in            │                           ││
│  │                             │                           ││
│  │                             │                           ││
│  └─────────────────────────────┴───────────────────────────┘│
│                                                              │
│  Status: Running | Port: 6112 | Uptime: 1h 23m             │
└─────────────────────────────────────────────────────────────┘
```

**Window Components**

```cpp
struct gui_struc {
    HWND    hwnd;                   // Main window handle
    HMENU   hmenuTray;              // System tray context menu
    HWND    hwndUsers;              // User list control
    HWND    hwndUserCount;          // User count static text
    HWND    hwndUserEditButton;     // User management button
    HWND    hwndTree;               // Channel tree view (future)
    HWND    ghwndConsole;           // RichEdit log window
    int     y_ratio;                // Vertical splitter position
    int     x_ratio;                // Horizontal splitter position
    RECT    rectHDivider;           // Horizontal splitter area
    RECT    rectVDivider;           // Vertical splitter area
    RECT    rectConsole;            // Log window area
    RECT    rectUsers;              // User list area
    BOOL    main_finished;          // Server shutdown flag
    int     mode;                   // Splitter drag mode
};
```

**Menu Items** (`resource.rc`)

```
Server Menu:
  - Save Accounts           (IDM_SAVE)
  - Restart Lua VM          (IDM_RESTART_LUA)
  - Restart                 (IDM_RESTART)
  - Shutdown                (IDM_SHUTDOWN)
  - Exit                    (IDM_EXIT)

AdminCommands Menu:
  - Announce                (IDM_ANN)
  - User Actions
    ├─ Admin Control Panel  (ID_USERACTIONS_KICKUSER)
    └─ Edit Profile         (ID_USERACTIONS_VIEWPROFILE)

View Menu:
  - Clear Window            (IDM_CLEAR)
  - Update Userlist         (IDM_USERLIST)

Server Configuration Menu:
  - Edit Config File        (ID_SERVERCONFIGURATION_EDITTHESERVERCONFIGFILE)

Help Menu:
  - About                   (IDM_ABOUT)
  - HomePage                (ID_HELP_CHECKFORUPDATES)
```

**GUI Thread Architecture**

```cpp
// Main function spawns GUI thread
int WINAPI WinMain(HINSTANCE hInstance, ...) {
    gui.event_ready = CreateEvent(...);
    _beginthread(guiThread, 0, (void*)hInstance);
    WaitForSingleObject(gui.event_ready, INFINITE);
    
    // Run server in main thread
    return app_main(argc, argv);
}

// GUI thread runs message loop
static void guiThread(void* param) {
    // Load RichEdit control library
    LoadLibrary("RichEd20.dll");
    
    // Register window class
    WNDCLASSEX wc;
    wc.lpfnWndProc = guiWndProc;
    wc.lpszClassName = "BnetdWndClass";
    RegisterClassEx(&wc);
    
    // Create main window
    gui.hwnd = CreateWindowEx(...);
    ShowWindow(gui.hwnd, SW_SHOW);
    SetEvent(gui.event_ready);
    
    // Message pump
    MSG msg;
    while (GetMessage(&msg, NULL, 0, 0)) {
        TranslateMessage(&msg);
        DispatchMessage(&msg);
    }
}
```

**Real-Time User List** (`guiOnUpdateUserList`)

```cpp
void guiOnUpdateUserList(void) {
    // Clear list view
    ListView_DeleteAllItems(gui.hwndUsers);
    
    // Iterate all connections
    t_connection* c;
    LIST_TRAVERSE_CONST(connlist(), c) {
        if (account = conn_get_account(c)) {
            char const* username = account_get_name(account);
            char const* ip = addr_num_to_ip_str(conn_get_addr(c));
            
            // Add to ListView
            LVITEM lvi;
            lvi.iItem = i++;
            lvi.pszText = (char*)username;
            ListView_InsertItem(gui.hwndUsers, &lvi);
            ListView_SetItemText(gui.hwndUsers, lvi.iItem, 1, (char*)ip);
        }
    }
    
    // Update count
    char buf[64];
    sprintf(buf, "Users: %d", i);
    SetWindowText(gui.hwndUserCount, buf);
}
```

**Admin Control Panel Dialog** (`IDD_KICKUSER`)

```cpp
INT_PTR KickDlgProc(HWND hwnd, UINT Message, WPARAM wParam, LPARAM lParam) {
    switch (Message) {
        case WM_COMMAND:
            if (LOWORD(wParam) == IDC_KICK_EXECUTE) {
                char username[128];
                GetDlgItemText(hwnd, IDC_EDITKICK, username, sizeof(username));
                
                BOOL kick = IsDlgButtonChecked(hwnd, IDC_CHECKKICK);
                BOOL ban = IsDlgButtonChecked(hwnd, IDC_CHECKBAN);
                BOOL admin = IsDlgButtonChecked(hwnd, IDC_CHECKADMIN);
                
                if (kick) {
                    // Execute /kick command
                    handle_command(server_conn, "/kick", username);
                }
                if (ban) {
                    // Execute /ban command
                    handle_command(server_conn, "/ban", username);
                }
                if (admin) {
                    // Promote to admin
                    account_set_auth(account, "admin");
                }
                
                EndDialog(hwnd, 0);
            }
            break;
    }
    return FALSE;
}
```

**System Tray Integration** (`WM_SHELLNOTIFY`)

```cpp
// Add icon to system tray
NOTIFYICONDATA nid;
nid.cbSize = sizeof(NOTIFYICONDATA);
nid.hWnd = hwnd;
nid.uID = ID_TRAY;
nid.uFlags = NIF_ICON | NIF_MESSAGE | NIF_TIP;
nid.uCallbackMessage = WM_SHELLNOTIFY;
nid.hIcon = LoadIcon(hInstance, MAKEINTRESOURCE(IDI_ICON1));
strcpy(nid.szTip, "PvPGN Server - Running");
Shell_NotifyIcon(NIM_ADD, &nid);

// Handle tray events
case WM_SHELLNOTIFY:
    if (lParam == WM_RBUTTONUP) {
        // Show context menu
        HMENU menu = LoadMenu(hInstance, MAKEINTRESOURCE(IDR_TRAYMENU));
        TrackPopupMenu(GetSubMenu(menu, 0), TPM_RIGHTBUTTON, x, y, 0, hwnd, NULL);
    }
    break;
```

**Color-Coded Logging**

```cpp
static void guiAddText(const char* text, COLORREF color) {
    // Get RichEdit control
    HWND console = ghwndConsole;
    
    // Select end of text
    CHARRANGE cr;
    cr.cpMin = -1;
    cr.cpMax = -1;
    SendMessage(console, EM_EXSETSEL, 0, (LPARAM)&cr);
    
    // Set color
    CHARFORMAT2 cf;
    cf.cbSize = sizeof(CHARFORMAT2);
    cf.dwMask = CFM_COLOR;
    cf.crTextColor = color;
    cf.dwEffects = 0;
    SendMessage(console, EM_SETCHARFORMAT, SCF_SELECTION, (LPARAM)&cf);
    
    // Append text
    SendMessage(console, EM_REPLACESEL, FALSE, (LPARAM)text);
    
    // Auto-scroll to bottom
    SendMessage(console, EM_SCROLLCARET, 0, 0);
}

// Log level colors
COLORREF colors[] = {
    RGB(128, 128, 128),  // TRACE - Gray
    RGB(0, 0, 255),      // DEBUG - Blue
    RGB(0, 0, 0),        // INFO - Black
    RGB(255, 165, 0),    // WARN - Orange
    RGB(255, 0, 0),      // ERROR - Red
    RGB(139, 0, 0)       // FATAL - Dark Red
};
```

### 3. Console Output Redirection

Allocates a Windows console for GUI applications to enable proper stdout/stderr handling.

**Console Class** (`console_output.h/cpp`)

```cpp
namespace pvpgn {
    class Console {
    public:
        Console();
        ~Console() throw();
        void RedirectIOToConsole();
    private:
        bool initialised;
    };
}
```

**Implementation**

```cpp
void Console::RedirectIOToConsole() {
    const WORD MAX_CONSOLE_LINES = 500;
    const WORD CONSOLE_COLUMNS = 140;
    
    if (initialised)
        return;
    
    // Create console window
    AllocConsole();
    
    // Set buffer size
    CONSOLE_SCREEN_BUFFER_INFO coninfo;
    GetConsoleScreenBufferInfo(GetStdHandle(STD_OUTPUT_HANDLE), &coninfo);
    coninfo.dwSize.X = CONSOLE_COLUMNS;
    coninfo.dwSize.Y = MAX_CONSOLE_LINES;
    SetConsoleScreenBufferSize(GetStdHandle(STD_OUTPUT_HANDLE), coninfo.dwSize);
    
    // Redirect stdin
    long lStdHandle = (long)GetStdHandle(STD_INPUT_HANDLE);
    int hConHandle = _open_osfhandle(lStdHandle, _O_TEXT);
    FILE* fp = _fdopen(hConHandle, "r");
    *stdin = *fp;
    setvbuf(stdin, NULL, _IONBF, 0);
    
    // Redirect stdout
    lStdHandle = (long)GetStdHandle(STD_OUTPUT_HANDLE);
    hConHandle = _open_osfhandle(lStdHandle, _O_TEXT);
    fp = _fdopen(hConHandle, "w");
    *stdout = *fp;
    setvbuf(stdout, NULL, _IONBF, 0);
    
    // Redirect stderr
    lStdHandle = (long)GetStdHandle(STD_ERROR_HANDLE);
    hConHandle = _open_osfhandle(lStdHandle, _O_TEXT);
    fp = _fdopen(hConHandle, "w");
    *stderr = *fp;
    setvbuf(stderr, NULL, _IONBF, 0);
    
    // Sync C++ streams
    std::ios::sync_with_stdio();
    initialised = true;
}
```

**Usage**

```cpp
#ifdef WIN32
    Console console;
    console.RedirectIOToConsole();
#endif

// Now printf, std::cout, etc. work properly
printf("Server starting...\n");
std::cout << "Port: 6112" << std::endl;
```

### 4. Crash Dump Generation

Automatic minidump creation on unhandled exceptions for post-mortem debugging.

**Crash Handler** (`windump.h/cpp`)

```cpp
// Register exception handler
LONG WINAPI unhandled_handler(struct _EXCEPTION_POINTERS* e);

// Install handler
SetUnhandledExceptionFilter(unhandled_handler);
```

**Minidump Creation**

```cpp
void make_minidump(struct _EXCEPTION_POINTERS* e) {
    // Load dbghelp.dll dynamically
    HMODULE hDbgHelp = LoadLibrary("dbghelp");
    if (hDbgHelp == nullptr)
        return;
    
    auto pMiniDumpWriteDump = (decltype(&MiniDumpWriteDump))
        GetProcAddress(hDbgHelp, "MiniDumpWriteDump");
    if (pMiniDumpWriteDump == nullptr)
        return;
    
    // Generate timestamped filename
    char name[MAX_PATH];
    auto nameEnd = name + GetModuleFileName(GetModuleHandle(0), name, MAX_PATH);
    SYSTEMTIME t;
    GetSystemTime(&t);
    wsprintf(nameEnd - strlen(".exe"),
             "_%4d%02d%02d_%02d%02d%02d.dmp",
             t.wYear, t.wMonth, t.wDay, t.wHour, t.wMinute, t.wSecond);
    
    // Create dump file
    HANDLE hFile = CreateFile(name, GENERIC_WRITE, FILE_SHARE_READ,
                              0, CREATE_ALWAYS, FILE_ATTRIBUTE_NORMAL, 0);
    if (hFile == INVALID_HANDLE_VALUE)
        return;
    
    // Write minidump
    MINIDUMP_EXCEPTION_INFORMATION exceptionInfo;
    exceptionInfo.ThreadId = GetCurrentThreadId();
    exceptionInfo.ExceptionPointers = e;
    exceptionInfo.ClientPointers = FALSE;
    
    pMiniDumpWriteDump(
        GetCurrentProcess(),
        GetCurrentProcessId(),
        hFile,
        MINIDUMP_TYPE(MiniDumpWithIndirectlyReferencedMemory | MiniDumpScanMemory),
        e ? &exceptionInfo : nullptr,
        nullptr,
        nullptr
    );
    
    CloseHandle(hFile);
}

LONG WINAPI unhandled_handler(struct _EXCEPTION_POINTERS* e) {
    make_minidump(e);
    return EXCEPTION_CONTINUE_SEARCH;  // Let system handler run
}
```

**Output Example**

```
# Crash at 2025-11-24 15:42:10
bnetd_20251124_154210.dmp

# Analyze with WinDbg or Visual Studio
File → Open → Crash Dump → bnetd_20251124_154210.dmp
```

**Minidump Contents**

- Exception context (registers, stack pointer)
- Stack trace for all threads
- Memory referenced by stack (local variables, parameters)
- Module list (loaded DLLs, addresses)
- System information (OS version, CPU)

### 5. POSIX Directory API (dirent.h)

Windows implementation of POSIX directory enumeration functions.

**API Implementation** (`dirent.h`)

```cpp
typedef struct dirent {
    char d_name[MAX_PATH + 1];  // Filename
    WIN32_FIND_DATAA data;      // File attributes
} dirent;

typedef struct DIR {
    dirent current;             // Current entry
    int cached;                 // Entry buffered flag
    HANDLE search_handle;       // FindFirstFile handle
    char patt[MAX_PATH + 3];    // Search pattern
} DIR;

// Open directory for reading
static DIR* opendir(const char* dirname);

// Read next directory entry
static struct dirent* readdir(DIR* dirp);

// Close directory handle
static int closedir(DIR* dirp);

// Reset to beginning
static void rewinddir(DIR* dirp);
```

**Implementation Details**

```cpp
static DIR* opendir(const char* dirname) {
    DIR* dirp = (DIR*)malloc(sizeof(struct DIR));
    if (dirp != NULL) {
        // Copy directory name
        strncpy(dirp->patt, dirname, sizeof(dirp->patt));
        dirp->patt[MAX_PATH] = '\0';
        
        // Append wildcard pattern
        char* p = strchr(dirp->patt, '\0');
        if (dirp->patt < p && *(p-1) != '\\' && *(p-1) != ':') {
            *p++ = '\\';
        }
        *p++ = '*';
        *p = '\0';
        
        // Open search handle
        dirp->search_handle = FindFirstFileA(dirp->patt, &dirp->current.data);
        if (dirp->search_handle == INVALID_HANDLE_VALUE) {
            free(dirp);
            return NULL;
        }
        
        dirp->cached = 1;  // First entry already read
    }
    return dirp;
}

static struct dirent* readdir(DIR* dirp) {
    // Return cached entry first
    if (dirp->cached) {
        dirp->cached = 0;
        strncpy(dirp->current.d_name, 
                dirp->current.data.cFileName, 
                sizeof(dirp->current.d_name));
        return &dirp->current;
    }
    
    // Read next entry
    if (FindNextFileA(dirp->search_handle, &dirp->current.data)) {
        strncpy(dirp->current.d_name,
                dirp->current.data.cFileName,
                sizeof(dirp->current.d_name));
        return &dirp->current;
    }
    
    return NULL;  // No more entries
}

static int closedir(DIR* dirp) {
    if (dirp->search_handle != INVALID_HANDLE_VALUE) {
        FindClose(dirp->search_handle);
    }
    free(dirp);
    return 0;
}
```

**Usage Example**

```cpp
#ifdef WIN32
# include "win32/dirent.h"
#else
# include <dirent.h>
#endif

// Cross-platform directory listing
DIR* dir = opendir("./accounts");
if (dir) {
    struct dirent* entry;
    while ((entry = readdir(dir)) != NULL) {
        if (strcmp(entry->d_name, ".") != 0 &&
            strcmp(entry->d_name, "..") != 0) {
            printf("File: %s\n", entry->d_name);
        }
    }
    closedir(dir);
}
```

**Why This Exists**

Windows doesn't provide POSIX `opendir()`, `readdir()`, `closedir()` functions. PvPGN uses these extensively for:
- Loading account files from directories
- Scanning configuration directories
- Enumerating ladder files
- Processing map directories

This header-only implementation wraps Windows `FindFirstFile/FindNextFile` API to provide compatible functionality.

## Building

The win32 library is built as a static library linked into server executables when targeting Windows.

### CMake Configuration

```cmake
# src/win32/CMakeLists.txt
set(WIN32_SOURCES
    service.cpp service.h
    console_output.h console_output.cpp
    dirent.h
    windump.cpp windump.h
)

add_library(win32 STATIC ${WIN32_SOURCES})
```

### Build Options

**Console Mode** (Default)
```bash
mkdir build && cd build
cmake ..
cmake --build .
```

**GUI Mode** (WIN32_GUI)
```bash
cmake -DWIN32_GUI=ON ..
cmake --build .
```

**Service Mode**
```bash
# Built automatically, activated via --service flag
bnetd.exe --service
```

### Linking

```cmake
# Server executable CMakeLists.txt
target_link_libraries(bnetd
    win32      # Windows support layer
    common     # Shared utilities
    compat     # Cross-platform compatibility
    # ... other libraries
)

# GUI mode requires additional libraries
if(WIN32_GUI)
    target_link_libraries(bnetd
        comctl32   # Common controls (ListView, etc.)
        shell32    # System tray API
    )
endif()
```

## Usage

### Running as Console Application

```bash
# Windows Command Prompt
C:\PvPGN> bnetd.exe -c conf\bnetd.conf

# PowerShell
PS C:\PvPGN> .\bnetd.exe -c .\conf\bnetd.conf

# With console redirection
C:\PvPGN> bnetd.exe -c conf\bnetd.conf > log.txt 2>&1
```

### Running as Windows Service

**Installation**

```bash
# Using sc command
sc create PvPGN binPath= "C:\PvPGN\bnetd.exe --service" start= auto
sc description PvPGN "PvPGN Battle.net emulation server"

# Or using built-in install function
bnetd.exe --install
```

**Management**

```bash
# Start service
sc start PvPGN
net start PvPGN

# Stop service
sc stop PvPGN
net stop PvPGN

# Query status
sc query PvPGN

# Uninstall
sc delete PvPGN
# Or
bnetd.exe --uninstall
```

**Service Configuration**

Services run with LocalSystem account by default. To use a different account:

```bash
sc config PvPGN obj= ".\ServiceUser" password= "password123"
```

**Service Logs**

Since services can't write to console, configure file logging:

```ini
# bnetd.conf
logfile = "C:\PvPGN\logs\bnetd.log"
loglevels = fatal,error,warn,info
```

### Running with GUI (WIN32_GUI Build)

**Launch**

```bash
# Double-click bnetd.exe in Windows Explorer
# Or from command line
C:\PvPGN> bnetd.exe
```

**GUI Commands**

- **File → Save Accounts**: Force account database flush
- **File → Restart**: Restart server without closing GUI
- **File → Shutdown**: Graceful shutdown (waits for connections)
- **AdminCommands → Announce**: Broadcast message to all users
- **AdminCommands → User Actions → Admin Control Panel**: Kick/ban/promote users
- **View → Update Userlist**: Refresh connected users list
- **View → Clear Window**: Clear log window

**Keyboard Shortcuts**

```
Ctrl+S - Save accounts
Ctrl+R - Restart server
Ctrl+Q - Quit
F5 - Refresh user list
```

**System Tray**

- Minimize to tray: Window minimizes to system tray icon
- Right-click tray icon: Quick access menu
  - Restore - Bring window to foreground
  - Shutdown - Graceful stop
  - Exit - Force close

### Crash Dump Analysis

**Automatic Generation**

When a crash occurs, a minidump is automatically created:

```
bnetd_20251124_154210.dmp
d2cs_20251124_160530.dmp
d2dbs_20251124_162015.dmp
```

**Analysis with WinDbg**

```bash
# Install Windows Debugging Tools
winget install Microsoft.WinDbg

# Open dump
windbg -z bnetd_20251124_154210.dmp

# In WinDbg command window:
!analyze -v        # Automatic analysis
k                  # Stack trace
lm                 # List loaded modules
~*k                # All thread stacks
.ecxr              # Exception context
```

**Analysis with Visual Studio**

```
1. File → Open → File → bnetd_20251124_154210.dmp
2. Click "Debug with Native Only"
3. View → Call Stack (shows crash location)
4. View → Locals (shows variable values)
5. Debug → Windows → Modules (shows loaded DLLs)
```

**Common Crash Patterns**

```
Access Violation (0xC0000005):
  - NULL pointer dereference
  - Buffer overflow
  - Use-after-free

Stack Overflow (0xC00000FD):
  - Infinite recursion
  - Large stack allocation

Assertion Failure:
  - Debug assertion triggered
  - Check assertion message in dump
```

## API Reference

### Service API

```cpp
// Install service in Service Control Manager
void Win32_ServiceInstall(void);

// Remove service from Service Control Manager
void Win32_ServiceUninstall(void);

// Run application as a service
void Win32_ServiceRun(void);

// Global status variable (read by main loop)
extern int g_ServiceStatus;
// 0 = Stopping, 1 = Running, 2 = Paused
```

### Console API

```cpp
namespace pvpgn {
    class Console {
    public:
        // Allocate console and redirect I/O
        void RedirectIOToConsole();
    };
}
```

### GUI API

```cpp
// Update user list in GUI (called from server thread)
extern void guiOnUpdateUserList(void);

// Add colored text to log window
static void guiAddText(const char* text, COLORREF color);

// Handle for console output
extern HWND ghwndConsole;
```

### Crash Handler API

```cpp
// Unhandled exception filter
LONG WINAPI unhandled_handler(struct _EXCEPTION_POINTERS* e);

// Install handler (called automatically)
SetUnhandledExceptionFilter(unhandled_handler);
```

### Directory API (dirent.h)

```cpp
// Open directory for reading
DIR* opendir(const char* dirname);

// Read next directory entry
struct dirent* readdir(DIR* dirp);

// Close directory
int closedir(DIR* dirp);

// Reset to first entry
void rewinddir(DIR* dirp);

// Directory entry structure
struct dirent {
    char d_name[MAX_PATH + 1];
    WIN32_FIND_DATAA data;
};
```

## Troubleshooting

### Service Issues

**Service won't install**

```bash
# Check if running as Administrator
# Right-click cmd.exe → "Run as administrator"

# Check service name conflicts
sc query PvPGN

# View detailed error
sc create PvPGN binPath= "C:\PvPGN\bnetd.exe --service"
# Error 1073: Service already exists
# Error 5: Access denied (need admin)
```

**Service won't start**

```bash
# Check Event Viewer
eventvwr.msc
# → Windows Logs → Application
# Look for PvPGN service errors

# Common issues:
# 1. Config file not found
#    → Service runs from System32, use absolute paths
# 2. Port already in use
#    → Check netstat -ano | findstr :6112
# 3. Missing DLLs
#    → Use Dependency Walker: depends.exe bnetd.exe
```

**Service stops unexpectedly**

```cpp
// Check g_ServiceStatus in main loop
while (g_ServiceStatus) {  // Must check this!
    // Server loop
}

// Log service state changes
eventlog(eventlog_level_info, __FUNCTION__, 
         "Service status changed to {}", g_ServiceStatus);
```

### GUI Issues

**GUI window won't appear**

```bash
# Check if WIN32_GUI was defined during build
cmake -DWIN32_GUI=ON ..

# Verify resource compilation
# Resources must be linked into executable
```

**RichEdit control errors**

```
"Could not load RichEd20.dll"

Solution:
1. Install Windows Desktop Runtime
2. Copy riched20.dll to application directory
3. Register DLL: regsvr32 riched20.dll
```

**User list not updating**

```cpp
// Ensure guiOnUpdateUserList() is called
// From server connection handler:

void connection_add(t_connection* conn) {
    // ... add connection ...
    
#ifdef WIN32_GUI
    guiOnUpdateUserList();  // Update GUI
#endif
}
```

### Console Redirection Issues

**printf output not appearing**

```cpp
// Ensure console is allocated
#ifdef WIN32
    Console console;
    console.RedirectIOToConsole();
#endif

// Flush output
printf("Message\n");
fflush(stdout);

// Or use unbuffered I/O
setvbuf(stdout, NULL, _IONBF, 0);
```

**stderr redirected incorrectly**

```cpp
// Both stdout and stderr should redirect
fprintf(stderr, "Error: %s\n", message);

// Check file descriptors
printf("stdout fileno: %d\n", fileno(stdout));
printf("stderr fileno: %d\n", fileno(stderr));
```

### Crash Dump Issues

**Minidumps not generated**

```cpp
// Ensure handler is installed
#ifdef WIN32
    SetUnhandledExceptionFilter(unhandled_handler);
#endif

// Check dbghelp.dll exists
// Should be in System32, but can bundle with app
// Copy from: C:\Windows\System32\dbghelp.dll
```

**Dump files too large**

```cpp
// Reduce dump type in windump.cpp
pMiniDumpWriteDump(
    GetCurrentProcess(),
    GetCurrentProcessId(),
    hFile,
    MiniDumpNormal,  // Smaller dump (only stacks, no memory)
    e ? &exceptionInfo : nullptr,
    nullptr,
    nullptr
);
```

### Directory Enumeration Issues

**opendir returns NULL**

```cpp
DIR* dir = opendir("./accounts");
if (!dir) {
    DWORD err = GetLastError();
    printf("opendir failed: %lu\n", err);
    // ERROR_FILE_NOT_FOUND (2): Directory doesn't exist
    // ERROR_PATH_NOT_FOUND (3): Path invalid
    // ERROR_ACCESS_DENIED (5): Permission denied
}
```

**Missing files in readdir**

```cpp
// Windows hidden/system files are included
struct dirent* entry;
while ((entry = readdir(dir)) != NULL) {
    // Check file attributes
    if (entry->data.dwFileAttributes & FILE_ATTRIBUTE_HIDDEN) {
        continue;  // Skip hidden files
    }
    
    printf("File: %s\n", entry->d_name);
}
```

## Integration Points

### Main Entry Points

**Console Application** (`main.cpp`)

```cpp
#ifdef WIN32
# include "win32/service.h"
#endif

int main(int argc, char* argv[]) {
#ifdef WIN32
    // Install crash handler
    SetUnhandledExceptionFilter(unhandled_handler);
    
    // Check for service mode
    if (cmdline_has_flag(argv, argc, "--service")) {
        Win32_ServiceRun();
        return 0;
    }
    
    if (cmdline_has_flag(argv, argc, "--install")) {
        Win32_ServiceInstall();
        return 0;
    }
    
    if (cmdline_has_flag(argv, argc, "--uninstall")) {
        Win32_ServiceUninstall();
        return 0;
    }
#endif
    
    // Normal server startup
    return server_main(argc, argv);
}
```

**GUI Application** (`winmain.cpp`)

```cpp
#ifdef WIN32_GUI
int WINAPI WinMain(HINSTANCE hInstance, HINSTANCE hPrevInstance,
                   LPSTR lpCmdLine, int nCmdShow) {
    // Parse command line
    int argc;
    char** argv = CommandLineToArgvA(lpCmdLine, &argc);
    
    // Start GUI thread
    gui.event_ready = CreateEvent(NULL, TRUE, FALSE, NULL);
    _beginthread(guiThread, 0, (void*)hInstance);
    WaitForSingleObject(gui.event_ready, INFINITE);
    
    // Run server in main thread
    return app_main(argc, argv);
}
#endif
```

### Server Main Loop Integration

```cpp
int server_main_loop() {
    while (1) {
#ifdef WIN32
        // Check service status
        if (!g_ServiceStatus) {
            eventlog(eventlog_level_info, "main", "Service stop requested");
            break;
        }
        
        if (g_ServiceStatus == 2) {
            // Service paused
            Sleep(100);
            continue;
        }
#endif
        
        // Normal server processing
        fdwatch_handle_events();
        
#ifdef WIN32_GUI
        // Update GUI periodically
        static time_t last_gui_update = 0;
        if (time(NULL) - last_gui_update > 5) {
            guiOnUpdateUserList();
            last_gui_update = time(NULL);
        }
#endif
    }
    
    return 0;
}
```

## Platform-Specific Considerations

### Windows Version Support

- **Minimum**: Windows XP SP3
- **Recommended**: Windows 7 SP1 or later
- **Optimal**: Windows 10/11 (better debugging tools)

**API Requirements**

```cpp
// Windows 2000+: Service descriptions
ChangeServiceConfig2A()  // Requires ADVAPI32.DLL

// Windows XP+: RichEdit 2.0
LoadLibrary("RichEd20.dll")

// Windows XP+: Minidumps
MiniDumpWriteDump()  // Requires dbghelp.dll
```

### Visual Studio Compatibility

```
VS 2015 (v140) - Minimum supported
VS 2017 (v141) - Recommended
VS 2019 (v142) - Fully tested
VS 2022 (v143) - Latest, fully supported
```

### 64-bit Support

```cmake
# Build 32-bit (x86)
cmake -A Win32 ..

# Build 64-bit (x64)
cmake -A x64 ..
```

Both architectures fully supported. 64-bit recommended for servers with >2GB memory.

## Related Components

- **src/common**: Event logging, GUI printf integration
- **src/compat**: Cross-platform socket and file operations
- **src/bnetd**: Battle.net server (primary GUI user)
- **src/d2cs**: D2 Character Server (separate GUI)
- **src/d2dbs**: D2 Database Server (separate GUI)

## References

- **Microsoft Service Documentation**: https://docs.microsoft.com/windows/win32/services
- **Windows Debugging Tools**: https://docs.microsoft.com/windows-hardware/drivers/debugger
- **RichEdit Control**: https://docs.microsoft.com/windows/win32/controls/rich-edit-controls
- **Minidump Format**: https://docs.microsoft.com/windows/win32/api/minidumpapiset

## Credits

- **Service Layer**: PvPGN Development Team
- **GUI Interface**: Erik Latoshek [forester] (laterk@inbox.lv)
- **Crash Handler**: HarpyWar (harpywar@gmail.com)
- **Console Redirection**: Richard Moore, Olaf Freyer
- **dirent.h**: Toni Ronkko (MIT License)
- **D2CS/D2DBS GUI**: CreepLord (creeplord@pvpgn.org)

## License

GNU General Public License v2 or later (except dirent.h which is MIT licensed).

---

**Status**: Production-ready Windows support layer used by PvPGN deployments worldwide.
