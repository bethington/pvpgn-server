# D2DBS - Diablo II Database Server

## Overview

The Diablo II Database Server (D2DBS) is the persistent storage backend for the PvPGN server's Diablo II closed realm system. It provides reliable character data storage, ladder rankings management, and character locking mechanisms to ensure data integrity across the Diablo II gaming infrastructure.

D2DBS acts as the authoritative data store, interfacing with Diablo II Game Servers (D2GS) to save and retrieve character files, prevent multi-login exploits, detect item duplication, and maintain competitive ladder rankings.

## Architecture

### System Position

```
┌─────────────────────────────────────────────────────────────┐
│                     PvPGN D2 Architecture                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Client ──► BNETD ──► D2CS ──┬─► D2GS (Game Server)         │
│              (Auth)   (Char   │    │                         │
│                       Server) │    │                         │
│                               │    │ Character Operations    │
│                               │    │ (Save/Load/Lock)        │
│                               │    ▼                         │
│                               └─► D2DBS ◄────────────────┐   │
│                                    (Database Server)     │   │
│                                    │                     │   │
│                                    ├─ Character Storage  │   │
│                                    ├─ Ladder Rankings    │   │
│                                    ├─ Character Locking  │   │
│                                    └─ Dupe Detection     │   │
│                                                          │   │
│                    Filesystem Storage                    │   │
│                    └────────────────────────────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Core Components

```
d2dbs/
├── Server Management
│   ├── main.cpp              - Entry point, daemon setup
│   ├── dbserver.h/cpp        - Server loop, connection handling
│   └── prefs.h/cpp           - Configuration management
│
├── Character Management
│   ├── charlock.h/cpp        - Character locking system
│   └── dbsdupecheck.h/cpp    - Item duplication detection
│
├── Ladder System
│   └── d2ladder.h/cpp        - Ranking system (35 ladder types)
│
├── Protocol Layer
│   ├── dbspacket.h/cpp       - D2GS ↔ D2DBS protocol
│   └── d2gs.h                - Game server interface
│
└── Storage Layer
    └── dbspacket.cpp         - File I/O with checksums, backups
```

## Key Features

### 1. Character Storage System

D2DBS provides reliable persistent storage for Diablo II character files with data integrity guarantees:

**Character Save Files (.d2s)**
- Binary format containing character state (level, inventory, skills, stats)
- Checksum validation on every save operation
- Atomic file operations (write to `.tmp`, then rename)
- Automatic backup to `bak_charsavedir` before overwriting
- Character name normalization (lowercase) for consistency

**Character Info Files**
- Portrait data and metadata storage
- Separate from save files for efficient access
- Same atomic save and backup mechanisms

**Storage Structure**
```
charsave/
  ├── account1/
  │   ├── Character1.d2s
  │   └── Character2.d2s
  └── account2/
      └── Character3.d2s

charinfo/
  ├── account1/
  │   ├── Character1
  │   └── Character2
  └── account2/
      └── Character3

bak/
  ├── charsave/          # Automatic backups
  └── charinfo/
```

### 2. Character Locking System

Prevents characters from logging into multiple game servers simultaneously, which could lead to item duplication exploits:

**Hash Table Implementation**
- Default capacity: 65,000 buckets (`DEFAULT_HASHTBL_LEN`)
- Case-insensitive character name hashing
- Collision handling via linked lists

**Locking Operations**
```
Lock:   D2GS → D2DBS: "Lock character 'Barbarian' for GSID 5"
        D2DBS: Hash("barbarian") → bucket 42538
               Store: {charname: "Barbarian", realm: "USEast", gsid: 5}

Query:  D2GS → D2DBS: "Is 'Barbarian' locked?"
        D2DBS: Check bucket 42538 → Found, locked to GSID 5

Unlock: D2GS → D2DBS: "Unlock 'Barbarian' from GSID 5"
        D2DBS: Remove from hash table bucket 42538
```

**Crash Recovery**
- When D2GS disconnects, D2DBS calls `cl_unlock_all_char_by_gsid(gsid)`
- Releases all character locks for that game server
- Secondary hash table (`gsqtbl`) maintains locks by game server ID
- Prevents characters from remaining permanently locked after server crashes

### 3. Ladder Ranking System

Maintains competitive ladder rankings across 35 different categories:

**Ladder Types** (D2LADDER_MAXTYPE = 35)
```
Standard (Non-Expansion):
  - Overall Standard (1000 entries)
  - Amazon, Sorceress, Necromancer, Paladin, Barbarian (200 each)
  - Overall Hardcore (1000 entries)  
  - Amazon HC, Sorceress HC, Necro HC, Paladin HC, Barbarian HC (200 each)

Expansion (Lord of Destruction):
  - Overall Expansion (1000 entries)
  - Amazon, Sorc, Necro, Paladin, Barb, Druid, Assassin (200 each)
  - Overall Expansion HC (1000 entries)
  - Amazon HC, Sorc HC, Necro HC, Paladin HC, Barb HC, Druid HC, Assassin HC (200 each)
```

**Ladder Data Structure**
```cpp
typedef struct {
    bn_int      experience;     // Primary ranking metric
    bn_short    status;         // Hardcore, expansion, ladder flags
    bn_short    level;          // Character level
    bn_byte     chclass;        // 0-6: Amazon, Sorc, Necro, Pala, Barb, Druid, Assassin
    bn_byte     charstatus;     // Additional status flags
    char        charname[MAX_CHARNAME_LEN];  // Character name
    char        realmname[MAX_REALMNAME_LEN]; // Realm identifier
} t_d2ladder_info;
```

**Update Algorithm**
1. D2GS sends UPDATE_LADDER packet after character level-up or significant progress
2. D2DBS determines appropriate ladder type based on:
   - Hardcore flag (`charstatus_get_hardcore`)
   - Expansion flag (`charstatus_get_expansion`)
   - Character class (0-6)
3. Finds insertion position via binary search by experience
4. Shifts entries down if necessary (kicked out if > max entries)
5. Increments `d2ladder_change_count` for dirty tracking

**Persistence**
- Periodic saves every `laddersave_interval` seconds (default: 3600 = 1 hour)
- Binary file format: header + index table + all ladder entries
- Checksum validation on load
- Optional XML export for external tools (`XML_ladder_output`)

### 4. D2GS Communication Protocol

Binary packet protocol for game server communication:

**Packet Structure**
```cpp
struct t_d2dbs_d2gs_header {
    bn_short    size;      // Total packet size
    bn_short    type;      // Packet type code
    bn_int      seqno;     // Sequence number
};
```

**Packet Types**

| Type | Code | Direction | Purpose |
|------|------|-----------|---------|
| `D2DBS_D2GS_SAVE_DATA_REQUEST` | 0x30 | D2GS → D2DBS | Save character data |
| `D2DBS_D2GS_SAVE_DATA_REPLY` | 0x30 | D2DBS → D2GS | Save result (success/fail) |
| `D2DBS_D2GS_GET_DATA_REQUEST` | 0x31 | D2GS → D2DBS | Load character data |
| `D2DBS_D2GS_GET_DATA_REPLY` | 0x31 | D2DBS → D2GS | Character data response |
| `D2DBS_D2GS_UPDATALADDER` | 0x32 | D2GS → D2DBS | Update ladder ranking |
| `D2DBS_D2GS_CHARLOCK` | 0x33 | D2GS ↔ D2DBS | Character lock/unlock |

**Data Types**
```cpp
#define D2GS_DATA_CHARSAVE    0x01  // Character save file (.d2s)
#define D2GS_DATA_PORTRAIT    0x02  // Character portrait/info
```

**Example: Save Character**
```
D2GS → D2DBS:
  Header: {size: 1024, type: 0x30, seqno: 42}
  Body: {
    datatype: 0x01,           // CHARSAVE
    datalen: 896,             // .d2s file size
    AccountName: "player123",
    CharName: "Barbarian",
    data: [... binary character data ...]
  }

D2DBS Processing:
  1. Validate checksum (d2charsave_checksum)
  2. Open "charsave/player123/Barbarian.tmp" for writing
  3. Write data atomically
  4. If "Barbarian.d2s" exists:
     - Move to "bak/charsave/player123/Barbarian.d2s"
  5. Rename "Barbarian.tmp" → "Barbarian.d2s"
  6. Log operation: "saved charsave data for character Barbarian"

D2DBS → D2GS:
  Header: {size: 8, type: 0x30, seqno: 42}
  Body: {result: 0}  // 0 = success, 1 = failed
```

### 5. Server Management

**Connection Structure**
```cpp
typedef struct {
    int             sd;             // Socket descriptor
    char            ipaddr[16];     // Client IP address
    unsigned short  major;          // Protocol major version
    unsigned short  minor;          // Protocol minor version
    unsigned short  type;           // Connection type
    unsigned int    serverid;       // Game server ID (preset)
    int             verified;       // IP verification status
    char*           ReadBuf;        // Incoming data buffer
    char*           WriteBuf;       // Outgoing data buffer
    unsigned int    charnum;        // Character count
    unsigned int    maxcharnum;     // Max characters allowed
    time_t          last_active;    // Last activity timestamp
} t_d2dbs_connection;
```

**Event Loop (select-based)**
```cpp
dbs_server_loop() {
    while (!signal_quit_received) {
        // Setup file descriptor sets
        dbs_server_setup_fdsets(&readfds, &writefds);
        
        // Wait for I/O events
        select(maxfd, &readfds, &writefds, NULL, &timeout);
        
        // Handle new connections
        if (FD_ISSET(listening_socket, &readfds)) {
            accept_connection();
        }
        
        // Handle client I/O
        for (each connection) {
            if (FD_ISSET(connection->sd, &readfds))
                handle_read(connection);
            if (FD_ISSET(connection->sd, &writefds))
                handle_write(connection);
        }
        
        // Periodic maintenance
        check_timeouts();
        check_ladder_save();
    }
}
```

**Daemon Features**
- Unix daemon mode: `fork()`, `setsid()`, `chdir("/")`
- PID file management for single-instance enforcement
- Windows service support via `win32/service.cpp`
- Graceful shutdown with configurable delay
- Signal handling (SIGTERM, SIGINT for shutdown)

### 6. Security Features

**IP Verification**
- Preset Game Server ID system maps allowed IP addresses to server IDs
- Only verified D2GS instances can send character data
- Prevents rogue servers from injecting fake characters

**Checksum Validation**
- Every character save includes a checksum
- Validated on every load/save operation
- Prevents corrupted data from being accepted

**Dupe Detection**
- `dbsdupecheck.h/cpp` implements item duplication detection
- Tracks item signatures across character saves
- Configurable dupe checking thresholds

**Atomic Operations**
- All file writes use `.tmp` extension during writing
- Rename to final filename only after successful write
- Prevents partial saves from corrupting character files

## Configuration

### d2dbs.conf

Key configuration options:

```ini
# Server listening address
servaddrs = 0.0.0.0:6114

# List of allowed D2GS IP addresses (MUST CONFIGURE)
gameservlist = 192.168.1.10,192.168.1.11

# Log files
logfile = /var/pvpgn/d2dbs.log
logfile-gs = /var/pvpgn/d2dbs-gs.log

# Storage directories
charsavedir = /var/pvpgn/charsave
charinfodir = /var/pvpgn/charinfo
ladderdir = /var/pvpgn/ladders
bak_charsavedir = /var/pvpgn/bak/charsave
bak_charinfodir = /var/pvpgn/bak/charinfo

# Ladder settings
laddersave_interval = 3600           # Save every hour
ladderinit_time = 0                  # Time before char enters ladder
XML_ladder_output = 0                # Export XML ladders (0=off, 1=on)

# Connection management
idletime = 300                       # 5 minutes idle timeout
keepalive_interval = 60              # TCP keepalive interval
timeout_checkinterval = 60           # Check timeouts every 60 seconds

# Shutdown behavior
shutdown_delay = 360                 # 6 minutes shutdown delay
shutdown_decr = 60                   # Decrement by 60s per signal
```

## Building

D2DBS is built as part of the PvPGN server suite:

```bash
# From pvpgn-server root
mkdir build && cd build
cmake ..
cmake --build . --target d2dbs

# Executable location
# Windows: build\bin\Debug\d2dbs.exe
# Linux: build/bin/d2dbs
```

### Dependencies

- **common**: Shared PvPGN utilities (eventlog, xalloc, network)
- **compat**: Cross-platform compatibility layer
- **fmt**: Format library for logging
- **win32**: Windows-specific service support (optional)

## Usage

### Starting D2DBS

```bash
# Linux/Unix
./d2dbs -c /etc/pvpgn/d2dbs.conf

# Windows
d2dbs.exe -c "C:\Program Files\PvPGN\conf\d2dbs.conf"

# As daemon (Linux)
./d2dbs -c /etc/pvpgn/d2dbs.conf -D

# Windows Service
sc create D2DBS binPath= "C:\PvPGN\d2dbs.exe -c C:\PvPGN\conf\d2dbs.conf"
sc start D2DBS
```

### Command-Line Options

```
-c <config_file>    Configuration file path
-D                  Run as daemon (Unix)
-h                  Display help
-v                  Display version
```

### Graceful Shutdown

```bash
# Send SIGTERM (or Ctrl+C)
kill -TERM `cat /var/run/d2dbs.pid`

# Shutdown sequence:
# 1. Wait shutdown_delay seconds (default: 360)
# 2. Each SIGTERM decreases delay by shutdown_decr (default: 60)
# 3. Save all ladder data
# 4. Close all connections
# 5. Exit cleanly
```

## API Reference

### Character Locking API

```cpp
// Initialize locking system
int cl_init(unsigned int tbllen, unsigned int maxgs);

// Lock character to specific game server
int cl_lock_char(unsigned char* charname, unsigned char* realmname, unsigned int gsid);

// Query if character is locked
int cl_query_charlock_status(unsigned char* charname, unsigned char* realmname, unsigned int* gsid);

// Unlock specific character
int cl_unlock_char(unsigned char* charname, unsigned char* realmname, unsigned int gsid);

// Release all locks for a game server (crash recovery)
int cl_unlock_all_char_by_gsid(unsigned int gsid);

// Cleanup
int cl_destroy(void);
```

### Ladder API

```cpp
// Initialize ladder system
int d2ladder_init(void);

// Update character ladder entry
int d2ladder_update(t_d2ladder_info* pcharladderinfo);

// Rebuild ladder from character files
int d2ladder_rebuild(void);

// Save ladder to disk
int d2ladder_saveladder(void);

// Cleanup
int d2ladder_destroy(void);
```

### Packet Handling API

```cpp
// Process save data request from D2GS
int dbs_packet_savedata(t_d2dbs_connection* conn, t_packet* packet);

// Process load data request from D2GS
int dbs_packet_getdata(t_d2dbs_connection* conn, t_packet* packet);

// Process ladder update
int dbs_packet_updateladder(t_d2dbs_connection* conn, t_packet* packet);

// Process character lock/unlock
int dbs_packet_charlock(t_d2dbs_connection* conn, t_packet* packet);
```

## Troubleshooting

### Common Issues

**1. Characters Not Saving**

Check:
- D2DBS is running and accessible on port 6114
- D2GS IP is listed in `gameservlist`
- `charsavedir` and `charinfodir` exist and are writable
- Disk space is available
- Check logs: `logfile-gs` for D2GS-specific errors

Verify:
```bash
# Check D2DBS is listening
netstat -an | grep 6114

# Check directory permissions
ls -ld /var/pvpgn/charsave

# Check disk space
df -h /var/pvpgn
```

**2. Character Multi-Login**

Symptoms: Players can log same character into multiple game instances

Causes:
- Character locking not working
- D2GS not sending CHAR_LOCK packets
- D2DBS crashed and didn't release locks

Solution:
```bash
# Check D2DBS logs for lock operations
grep "lock" /var/pvpgn/d2dbs-gs.log

# Restart D2DBS to clear all locks
killall -TERM d2dbs
./d2dbs -c /etc/pvpgn/d2dbs.conf -D

# Verify in code: charlock.cpp logs should show lock/unlock operations
```

**3. Ladder Not Updating**

Check:
- `laddersave_interval` not too high
- `ladderdir` exists and is writable
- D2GS is sending UPDATE_LADDER packets
- Character meets threshold: `prefs_get_ladderupdate_threshold()`

Debug:
```cpp
// Enable ladder debug logging in d2ladder.cpp
eventlog(eventlog_level_debug, __FUNCTION__, 
         "ladder update: char={} exp={} class={} type={}", 
         info->charname, info->experience, info->chclass, ladder_type);
```

**4. "D2DBS WON'T WORK PROPERLY"**

Error message in config indicates `gameservlist` not configured.

Fix:
```ini
# Must use actual D2GS IP addresses
gameservlist = 192.168.1.10,192.168.1.11

# DO NOT USE:
# gameservlist = 127.0.0.1,localhost  ❌ Wrong!
```

**5. Character Files Corrupted**

Recovery:
```bash
# Character backups are in bak_charsavedir
cd /var/pvpgn/bak/charsave/accountname

# Copy backup to active directory
cp Character.d2s /var/pvpgn/charsave/accountname/

# Check backup age
ls -l /var/pvpgn/bak/charsave/accountname/
```

Enable debug logging:
```ini
loglevels = fatal,error,warn,info,debug,trace
```

### Log Analysis

**Normal Operation**
```
[info] d2dbs server starting up
[info] listening on 0.0.0.0:6114
[info] connection from 192.168.1.10 (D2GS)
[info] saved charsave data for character Barbarian (account: player123)
[info] ladder updated: Barbarian exp=1250000 class=4
[info] saved ladder file: /var/pvpgn/ladders/ladder.dat
```

**Error Indicators**
```
[error] failed to save character: disk full
[error] checksum mismatch for character Sorceress
[error] unknown game server IP: 10.0.0.5
[warn] character Necromancer already locked to GSID 2
```

## Performance Considerations

### Scalability

- **Connection limit**: Determined by OS file descriptor limit
- **Character locking**: O(1) average case, O(n) worst case with hash collisions
- **Ladder updates**: O(log n) insertion, O(n) shift operations
- **File I/O**: Disk-bound, consider SSD for high-traffic servers

### Optimization Tips

1. **Increase hash table size** for servers with many concurrent characters:
   ```cpp
   // In charlock.h
   #define DEFAULT_HASHTBL_LEN 130000  // Double default
   ```

2. **Reduce ladder save frequency** if ladder not critical:
   ```ini
   laddersave_interval = 7200  # Save every 2 hours
   ```

3. **Use SSD storage** for character directories:
   - Reduces save latency
   - Improves concurrent access

4. **Separate backup directory** to different disk:
   ```ini
   bak_charsavedir = /mnt/backup/pvpgn/charsave
   ```

### Monitoring

Key metrics to track:
- Active connections: Number of D2GS instances connected
- Character saves per second: I/O throughput
- Ladder change count: `d2ladder_change_count` increments
- Lock table utilization: Percentage of hash buckets used
- Backup directory size: Disk space monitoring

## Integration with PvPGN

### Required Configuration

**d2cs.conf** (Character Server)
```ini
# D2DBS connection
d2dbs_host = 192.168.1.100
d2dbs_port = 6114
```

**D2GS Configuration**
```ini
# In D2GS config (game server)
D2DBSHost = 192.168.1.100
D2DBSPort = 6114
```

### Data Flow Example

```
1. Player logs in:
   Client → BNETD: Authentication
   BNETD → D2CS: Character list request
   D2CS → D2DBS: GET_DATA_REQUEST (load character)
   D2DBS → D2CS: GET_DATA_REPLY (character data)
   D2CS → BNETD: Character list
   BNETD → Client: Show character selection

2. Player joins game:
   Client → D2GS: Join game with character "Barbarian"
   D2GS → D2DBS: CHAR_LOCK (lock "Barbarian")
   D2DBS: Check lock table → Available → Lock to GSID 5
   D2DBS → D2GS: CHAR_LOCK reply (success)
   D2GS → Client: Enter game world

3. Player gains level:
   D2GS: Character level up → experience increased
   D2GS → D2DBS: UPDATE_LADDER (new exp, level, class)
   D2DBS: Update ladder ranking (Standard Barbarian ladder)
   D2DBS: d2ladder_change_count++

4. Player quits game:
   Client → D2GS: Leave game
   D2GS → D2DBS: SAVE_DATA_REQUEST (save .d2s file)
   D2DBS: Write to Barbarian.tmp
   D2DBS: Move old to backup
   D2DBS: Rename tmp → Barbarian.d2s
   D2DBS → D2GS: SAVE_DATA_REPLY (success)
   D2GS → D2DBS: CHAR_LOCK unlock "Barbarian"
   D2DBS: Remove from lock table
```

## Development

### Adding New Features

**Example: Add Character Statistics Tracking**

1. Extend protocol in `dbspacket.h`:
   ```cpp
   #define D2DBS_D2GS_CHARSTAT    0x35
   ```

2. Add packet handler in `dbspacket.cpp`:
   ```cpp
   int dbs_packet_charstat(t_d2dbs_connection* conn, t_packet* packet) {
       // Extract statistics
       // Store in database
       // Send reply
   }
   ```

3. Update `dbserver.cpp` dispatch:
   ```cpp
   case D2DBS_D2GS_CHARSTAT:
       return dbs_packet_charstat(conn, packet);
   ```

### Testing

```bash
# Build with debug symbols
cmake -DCMAKE_BUILD_TYPE=Debug ..
cmake --build .

# Run with GDB
gdb --args ./d2dbs -c test.conf

# Set breakpoints
(gdb) break dbs_packet_savedata
(gdb) break cl_lock_char
(gdb) run

# Check ladder state
(gdb) p *d2ladder_list
(gdb) p d2ladder_change_count
```

## Related Components

- **src/d2cs**: Diablo II Character Server (D2DBS client)
- **src/d2dbs**: This component
- **src/common**: Shared utilities (logging, network, data structures)
- **src/compat**: Cross-platform compatibility

## References

- **Configuration**: `conf/d2dbs.conf.in`
- **Protocol Documentation**: `docs/readme.md`
- **Storage Layout**: `docs/storage.txt`
- **Diablo II File Format**: Community documentation (.d2s structure)

## License

GNU General Public License v2 or later. See LICENSE file for details.

## Credits

Original implementation by sousou (liupeng.cs@263.net) and faster (lqx@cic.tsinghua.edu.cn).

---

**Status**: Production-ready component in active use by PvPGN Diablo II servers worldwide.
