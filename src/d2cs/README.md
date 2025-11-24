# PvPGN D2CS - Diablo II Character Server

## Overview

The `src/d2cs` folder contains the **Diablo II Character Server (D2CS)**, a critical component of the PvPGN server architecture that manages Diablo II character data, game creation, and coordinates communication between the Battle.net daemon (bnetd), game clients, and Diablo II Game Servers (D2GS).

## Purpose

D2CS serves as the intermediary layer in the Diablo II closed Battle.net realm system:

1. **Character Management**: Store, create, delete, and validate Diablo II character files
2. **Game Coordination**: Create and manage game lobbies, coordinate player entry
3. **Server Routing**: Route clients to appropriate D2GS instances for gameplay
4. **Ladder System**: Maintain and serve ladder rankings for competitive play
5. **Authentication Relay**: Forward authentication requests between clients and bnetd
6. **Load Balancing**: Distribute games across multiple D2GS instances

## Architecture

D2CS operates as a **standalone daemon** that communicates with three types of entities:

```
                    ┌──────────────┐
                    │    bnetd     │
                    │  (Auth/Chat) │
                    └──────┬───────┘
                           │
              ┌────────────┴────────────┐
              │                         │
         Auth │                         │ Game State
      Session │                         │ Updates
              │                         │
        ┌─────▼─────┐             ┌─────▼─────┐
        │   D2CS    │◄───────────►│   D2DBS   │
        │ (Character│             │ (Database)│
        │   Server) │             └───────────┘
        └─────┬─────┘
              │
      ┌───────┼───────┐
      │       │       │
   Create  Ladder  Char
    Game    Info   List
      │       │       │
┌─────▼───────▼───────▼─────┐
│  Diablo II Clients (TCP)  │
└───────────┬───────────────┘
            │
         Join Game
            │
      ┌─────▼─────┐
      │   D2GS    │
      │  (Game    │
      │  Server)  │
      └───────────┘
```

### Component Interaction Flow

**Login Flow**:
1. Client connects to bnetd, authenticates account
2. Client connects to D2CS with session token
3. D2CS validates session with bnetd
4. Client selects/creates character
5. D2CS loads character from disk, validates integrity

**Game Creation Flow**:
1. Client requests game creation via D2CS
2. D2CS selects available D2GS instance
3. D2CS forwards game creation request to D2GS
4. D2GS creates game, returns game ID and token
5. D2CS returns game info to client
6. Client connects directly to D2GS for gameplay

**Game Join Flow**:
1. Client requests game list from D2CS
2. Client selects game and requests join
3. D2CS validates character eligibility (level, difficulty, hardcore/softcore)
4. D2CS adds client to game queue if game is full
5. When space available, D2CS provides D2GS address and token
6. Client connects to D2GS with token

## Core Components

### 1. Connection Management (`connection.h/cpp`)

Manages all network connections to D2CS.

**Connection Types**:
```cpp
typedef enum {
    conn_class_none,
    conn_class_init,     // Initializing connection
    conn_class_d2cs,     // Diablo II client
    conn_class_d2gs,     // Game server
    conn_class_bnetd     // Battle.net daemon
} t_conn_class;
```

**Connection States**:
```cpp
typedef enum {
    conn_state_none         = 0x00,
    conn_state_init         = 0x01,  // Initial connection
    conn_state_connecting   = 0x02,  // TCP handshake
    conn_state_connected    = 0x04,  // Connected, not authenticated
    conn_state_authed       = 0x08,  // Account authenticated
    conn_state_char_authed  = 0x10,  // Character authenticated
    conn_state_destroy      = 0x20,  // Marked for deletion
    conn_state_destroying   = 0x40   // Being destroyed
} t_conn_state;
```

**Connection Structure**:
```cpp
typedef struct {
    char const*              charname;        // Selected character name
    char const*              account;         // Account name
    int                      sock;            // Socket descriptor
    int                      fdw_idx;         // fdwatch index
    unsigned int             socket_flag;     // Read/write flags
    unsigned int             addr;            // Remote IP address
    unsigned int             local_addr;      // Local IP address
    unsigned short           port;            // Remote port
    unsigned short           local_port;      // Local port
    unsigned int             last_active;     // Last activity timestamp
    t_conn_class             cclass;          // Connection class
    t_conn_state             state;           // Current state
    unsigned int             sessionnum;      // D2CS session number
    t_queue*                 outqueue;        // Outgoing packet queue
    unsigned int             outsize;         // Bytes in queue
    unsigned int             outsizep;        // Pending bytes
    t_packet*                inqueue;         // Incoming packet buffer
    unsigned int             insize;          // Bytes received
    t_d2charinfo_summary*    charinfo;        // Character information
    unsigned int             d2gs_id;         // Assigned D2GS ID
    t_gq*                    gamequeue;       // Game queue entry
    unsigned int             bnetd_sessionnum;// bnetd session number
    unsigned int             sessionnum_hash; // Hash for lookup
    unsigned int             charname_hash;   // Hash for lookup
} t_connection;
```

**API**:
```cpp
// Connection list management
t_hashtable* d2cs_connlist(void);
int d2cs_connlist_create(void);
int d2cs_connlist_destroy(void);
int d2cs_connlist_reap(void);  // Remove destroyed connections

// Connection lookup
t_connection* d2cs_connlist_find_connection_by_sessionnum(unsigned int sessionnum);
t_connection* d2cs_connlist_find_connection_by_charname(char const* charname);

// Connection operations
int conn_check_multilogin(t_connection const* c, char const* charname);
int conn_set_state(t_connection* c, t_conn_state state);
int conn_destroy(t_connection* c, t_elem** elem);
```

### 2. Character File Management (`d2charfile.h/cpp`, `d2charlist.h/cpp`)

Handles Diablo II `.d2s` character save files.

**Character Status Flags**:
```cpp
#define D2CHARINFO_STATUS_FLAG_INIT       0x01  // Character initialized
#define D2CHARINFO_STATUS_FLAG_EXPANSION  0x20  // Lord of Destruction
#define D2CHARINFO_STATUS_FLAG_LADDER     0x40  // Ladder character
#define D2CHARINFO_STATUS_FLAG_HARDCORE   0x04  // Hardcore mode
#define D2CHARINFO_STATUS_FLAG_DEAD       0x08  // Character is dead
```

**Character Classes**:
```cpp
#define D2CHAR_CLASS_AMAZON       0x00
#define D2CHAR_CLASS_SORCERESS    0x01
#define D2CHAR_CLASS_NECROMANCER  0x02
#define D2CHAR_CLASS_PALADIN      0x03
#define D2CHAR_CLASS_BARBARIAN    0x04
#define D2CHAR_CLASS_DRUID        0x05  // Expansion
#define D2CHAR_CLASS_ASSASSIN     0x06  // Expansion
```

**Character Summary Structure**:
```cpp
typedef struct {
    bn_int     experience;      // Character experience
    bn_int     charstatus;      // Status flags
    bn_short   charlevel;       // Character level
    bn_byte    charclass;       // Character class
    bn_byte    portrait[30];    // Portrait/appearance data
    char       charname[16];    // Character name
} t_d2charinfo_summary;
```

**File Format Offsets**:
```cpp
// Version 1.09 and later
#define D2CHARSAVE_CHECKSUM_OFFSET       0x0C
#define D2CHARSAVE_CHARNAME_OFFSET_109   0x14
#define D2CHARSAVE_STATUS_OFFSET_109     0x24
#define D2CHARSAVE_CLASS_OFFSET_109      0x28
```

**API**:
```cpp
// Character file operations
int d2char_create(char const* account, char const* charname,
                  unsigned char chclass, unsigned short status);
int d2char_delete(char const* account, char const* charname);
int d2char_find(char const* account, char const* charname);
int d2char_convert(char const* account, char const* charname);  // Classic to LOD

// Character information
int d2char_get_summary(char const* account, char const* charname,
                       t_d2charinfo_summary* charinfo);
int d2char_get_portrait(char const* account, char const* filename,
                        t_d2charinfo_portrait* portrait);
int d2charinfo_load(char const* account, char const* charname,
                    t_d2charinfo_file* data);
int d2charinfo_check(t_d2charinfo_file* data);  // Validate checksum

// Character list operations
int d2charlist_create(char const* account);
int d2charlist_find_character(char const* account, char const* charname);
unsigned int d2charlist_get_charnum(char const* account);
```

**File Locations**:
- **Save Files**: `var/charsave/<account>/<charname>.d2s`
- **Info Files**: `var/charinfo/<account>/<charname>`
- **Backup**: `var/bak/charsave/<account>/<charname>.d2s`
- **Newbie Templates**: `files/newbie.save` (for character creation)

**Character Integrity**:
- Checksum validation prevents tampering
- Level/status/class validation
- Hardcore death flag enforcement
- Expansion/Classic compatibility checking

### 3. Game Management (`game.h/cpp`)

Manages game lobbies and player lists.

**Game Structure**:
```cpp
typedef struct {
    unsigned int    id;                 // Unique game ID
    char const*     name;               // Game name
    char const*     pass;               // Password (NULL if public)
    char const*     desc;               // Game description
    int             create_time;        // Creation timestamp
    int             lastaccess_time;    // Last access time
    unsigned int    created;            // Creation flag
    unsigned int    gameflag;           // Game settings flags
    unsigned int    charlevel;          // Character level restriction
    unsigned int    leveldiff;          // Level difference allowed
    unsigned int    d2gs_gameid;        // D2GS-assigned game ID
    unsigned int    maxchar;            // Max players (usually 8)
    unsigned int    currchar;           // Current player count
    t_list*         charlist;           // List of players in game
    t_d2gs*         d2gs;               // Assigned game server
} t_game;
```

**Game Flags**:
```cpp
#define D2_GAMEFLAG_EXPANSION   0x00100000  // Lord of Destruction
#define D2_GAMEFLAG_LADDER      0x00200000  // Ladder game
#define D2_GAMEFLAG_HARDCORE    0x00000800  // Hardcore mode
#define D2_GAMEFLAG_SINGLE      0x00000010  // No party allowed

// Difficulty levels (stored in bits 12-14)
#define DIFFICULTY_NORMAL       0
#define DIFFICULTY_NIGHTMARE    1
#define DIFFICULTY_HELL         2
```

**Game Flag Macros**:
```cpp
#define gameflag_get_difficulty(gameflag)   ((gameflag >> 0x0C) & 0x07)
#define gameflag_get_expansion(gameflag)    ((gameflag & D2_GAMEFLAG_EXPANSION) != 0)
#define gameflag_get_ladder(gameflag)       ((gameflag & D2_GAMEFLAG_LADDER) != 0)
#define gameflag_get_hardcore(gameflag)     ((gameflag & D2_GAMEFLAG_HARDCORE) != 0)

#define gameflag_create(l,e,h,d) \
    (0x04 | (e?D2_GAMEFLAG_EXPANSION:0) | (h?D2_GAMEFLAG_HARDCORE:0) | \
     ((d & 0x07) << 0x0c) | (l?D2_GAMEFLAG_LADDER:0))
```

**API**:
```cpp
// Game list management
t_list* d2cs_gamelist(void);
int d2cs_gamelist_create(void);
int d2cs_gamelist_destroy(void);
void d2cs_gamelist_check_voidgame(void);  // Remove empty games

// Game operations
t_game* d2cs_game_create(char const* gamename, char const* gamepass,
                         char const* gamedesc, unsigned int gameflag);
int game_destroy(t_game* game, t_elem** elem);
t_game* d2cs_gamelist_find_game(char const* gamename);
t_game* gamelist_find_game_by_id(unsigned int id);
t_game* gamelist_find_game_by_d2gs_and_id(unsigned int d2gs_id,
                                           unsigned int d2gs_gameid);

// Player management
int game_add_character(t_game* game, char const* charname,
                       unsigned char chclass, unsigned char level);
int game_del_character(t_game* game, char const* charname);

// Game properties
t_d2gs* game_get_d2gs(t_game const* game);
int game_set_d2gs(t_game* game, t_d2gs* gs);
unsigned int game_get_d2gs_gameid(t_game const* game);
unsigned int game_get_id(t_game const* game);
char const* game_get_name(t_game const* game);
```

**Game Lifecycle**:
1. Client requests game creation
2. D2CS validates request (character level, flags)
3. D2CS selects available D2GS
4. D2CS creates game entry, forwards to D2GS
5. D2GS creates game world, returns game ID
6. D2CS stores game-to-D2GS mapping
7. Client joins via D2GS address and token
8. Game removed when all players leave or timeout

### 4. D2GS Management (`d2gs.h/cpp`)

Manages connections to Diablo II Game Servers.

**D2GS Structure**:
```cpp
typedef struct d2gs {
    unsigned int    ip;         // Game server IP address
    unsigned int    id;         // Unique server ID
    unsigned int    flag;       // Status flags
    unsigned int    active;     // Connection active flag
    unsigned int    token;      // Authentication token
    t_d2gs_state    state;      // Connection state
    unsigned int    maxgame;    // Maximum concurrent games
    unsigned int    gamenum;    // Current game count
    t_connection*   connection; // Connection to D2GS
} t_d2gs;
```

**D2GS States**:
```cpp
typedef enum {
    d2gs_state_none,        // Uninitialized
    d2gs_state_connected,   // TCP connected
    d2gs_state_authed       // Authenticated
} t_d2gs_state;
```

**API**:
```cpp
// D2GS list management
t_list* d2gslist(void);
int d2gslist_create(void);
int d2gslist_destroy(void);
int d2gslist_reload(char const* gslist);

// D2GS operations
t_d2gs* d2gs_create(char const* ip);
int d2gs_destroy(t_d2gs* gs, t_elem** curr);
t_d2gs* d2gslist_find_gs(unsigned int id);
t_d2gs* d2gslist_find_gs_by_ip(unsigned int ip);
t_d2gs* d2gslist_choose_server(void);  // Load balancing

// D2GS properties
unsigned int d2gs_get_id(t_d2gs const* gs);
unsigned int d2gs_get_ip(t_d2gs const* gs);
unsigned int d2gs_get_maxgame(t_d2gs const* gs);
unsigned int d2gs_get_gamenum(t_d2gs const* gs);
int d2gs_add_gamenum(t_d2gs* gs, int number);

// D2GS authentication
unsigned int d2gs_calc_checksum(t_connection* c);
unsigned int d2gs_make_token(t_d2gs* gs);

// D2GS maintenance
int d2gs_keepalive(void);
int d2gs_restart_all_gs(void);
int d2gs_active(t_d2gs* gs, t_connection* c);
int d2gs_deactive(t_d2gs* gs, t_connection* c);
```

**Load Balancing**:
D2CS selects the D2GS with:
1. Lowest `gamenum / maxgame` ratio (least loaded)
2. State is `d2gs_state_authed`
3. `flag & D2GS_FLAG_VALID` is set

### 5. Ladder System (`d2ladder.h/cpp`)

Maintains and serves ladder rankings.

**Ladder Structure**:
```cpp
typedef struct {
    unsigned int                 type;      // Ladder type
    unsigned int                 len;       // Total entries
    unsigned int                 curr_len;  // Current entries
    t_d2cs_client_ladderinfo*    info;      // Ladder data array
} t_d2ladder;
```

**Ladder Types**:
```cpp
// Standard ladders (per difficulty)
D2LADDER_NORMAL_CLASSIC_STANDARD
D2LADDER_NORMAL_EXPANSION_STANDARD
D2LADDER_NIGHTMARE_CLASSIC_STANDARD
D2LADDER_NIGHTMARE_EXPANSION_STANDARD
D2LADDER_HELL_CLASSIC_STANDARD
D2LADDER_HELL_EXPANSION_STANDARD

// Hardcore ladders
D2LADDER_NORMAL_CLASSIC_HARDCORE
D2LADDER_NORMAL_EXPANSION_HARDCORE
// ... (same difficulty breakdown)

// Per-class ladders
D2LADDER_EXPANSION_AMAZON
D2LADDER_EXPANSION_SORCERESS
D2LADDER_EXPANSION_NECROMANCER
D2LADDER_EXPANSION_PALADIN
D2LADDER_EXPANSION_BARBARIAN
D2LADDER_EXPANSION_DRUID
D2LADDER_EXPANSION_ASSASSIN
```

**Ladder Entry**:
```cpp
typedef struct {
    bn_int      experience;      // Character experience
    bn_int      status;          // Character status
    bn_short    level;           // Character level
    bn_byte     charclass;       // Character class
    char        charname[16];    // Character name
} t_d2cs_client_ladderinfo;
```

**API**:
```cpp
// Ladder management
int d2ladder_init(void);
int d2ladder_refresh(void);  // Reload from D2DBS
int d2ladder_destroy(void);

// Ladder queries
int d2ladder_get_ladder(unsigned int* from, unsigned int* count,
                        unsigned int type,
                        t_d2cs_client_ladderinfo const** info);
int d2ladder_find_character_pos(unsigned int type, char const* charname);
```

**Ladder Refresh**:
- Periodic refresh from D2DBS (default: every 3600 seconds)
- Requests specific ladder type from D2DBS
- Updates in-memory ladder cache
- Serves cached data to clients

### 6. Game Queue (`gamequeue.h/cpp`)

Manages waiting lists for full games.

**Game Queue Entry**:
```cpp
typedef struct gq {
    unsigned int    gameid;      // Target game ID
    t_connection*   connection;  // Waiting client
    unsigned int    position;    // Queue position
} t_gq;
```

**API**:
```cpp
// Queue management
t_list* gqlist(void);
int gqlist_create(void);
int gqlist_destroy(void);

// Queue operations
t_gq* gq_create(unsigned int gameid, t_connection* c);
int gq_destroy(t_gq* gq, t_elem** curr);
int gqlist_check_timeout(void);

// Queue queries
unsigned int gqlist_get_length(void);
unsigned int gqlist_get_game_length(unsigned int gameid);
unsigned int gq_get_position(t_gq const* gq);
```

**Queue Flow**:
1. Client attempts to join full game
2. D2CS adds client to game queue
3. D2CS returns queue position to client
4. When player leaves game, D2CS notifies next queued client
5. Client removed from queue when:
   - Successfully joins game
   - Cancels join request
   - Timeout expires
   - Game is destroyed

### 7. Server Queue (`serverqueue.h/cpp`)

Tracks pending requests to bnetd/D2GS awaiting replies.

**Server Queue Entry**:
```cpp
typedef struct serverqueue {
    unsigned int    seqno;       // Sequence number
    unsigned int    ctime;       // Creation time
    unsigned int    clientid;    // Client session number
    t_packet*       packet;      // Original request packet
    unsigned int    gameid;      // Game ID (if applicable)
    unsigned int    gametoken;   // Game token (if applicable)
} t_sq;
```

**API**:
```cpp
// Server queue management
t_list* sqlist(void);
int sqlist_create(void);
int sqlist_destroy(void);
int sqlist_check_timeout(void);

// Queue operations
t_sq* sq_create(unsigned int clientid, t_packet* packet,
                unsigned int gameid);
int sq_destroy(t_sq* sq, t_elem** curr);
t_sq* sqlist_find_sq(unsigned int seqno);

// Queue properties
unsigned int sq_get_clientid(t_sq const* sq);
unsigned int sq_get_gameid(t_sq const* sq);
unsigned int sq_get_seqno(t_sq const* sq);
t_packet* sq_get_packet(t_sq const* sq);
```

**Purpose**:
- Match asynchronous replies to original requests
- Timeout stale requests
- Forward responses to correct clients
- Track multi-step operations

### 8. bnetd Communication (`bnetd.h/cpp`)

Manages connection to Battle.net daemon.

**API**:
```cpp
// bnetd connection
int bnetd_init(void);
int bnetd_check(void);
t_connection* bnetd_conn(void);
int bnetd_destroy(t_connection* c);
int bnetd_set_connection(t_connection* c);
int bnetd_keepalive(void);
```

**Communication Protocol** (D2CS ↔ bnetd):
```cpp
// D2CS → bnetd
D2CS_BNETD_ACCOUNTLOGINREQ      // Validate client session
D2CS_BNETD_CHARLOGINREQ         // Notify character login
D2CS_BNETD_KEEPALIVE            // Keepalive ping

// bnetd → D2CS
BNETD_D2CS_ACCOUNTLOGINREPLY    // Session validation result
BNETD_D2CS_CHARLOGINREPLY       // Character login acknowledgment
BNETD_D2CS_AUTHREQ              // Authentication request
```

**Session Validation Flow**:
1. Client connects to D2CS with session token from bnetd
2. D2CS forwards session token to bnetd
3. bnetd validates session, returns account name
4. D2CS completes client login

### 9. Packet Handlers

**Client Protocol Handlers** (`handle_d2cs.h/cpp`):
```cpp
// Authentication
on_client_loginreq              // Account session validation
on_client_charloginreq          // Character selection

// Character management
on_client_createcharreq         // Create new character
on_client_deletecharreq         // Delete character
on_client_charlistreq           // Request character list
on_client_convertcharreq        // Convert Classic to LOD

// Game operations
on_client_creategamereq         // Create new game
on_client_joingamereq           // Join existing game
on_client_cancelcreategame      // Cancel game creation

// Information requests
on_client_gamelistreq           // Request game list
on_client_gameinforeq           // Request game details
on_client_ladderreq             // Request ladder data
on_client_charladderreq         // Request character ladder rank
on_client_motdreq               // Request message of the day
```

**D2GS Protocol Handlers** (`handle_d2gs.h/cpp`):
```cpp
// D2GS management
handle_d2gs_init                // D2GS authentication
handle_d2gs_authreq             // D2GS auth request
handle_d2gs_authreply           // D2GS auth reply
handle_d2gs_setinitinfo         // D2GS capabilities

// Game management
handle_d2gs_creategamereply     // Game creation result
handle_d2gs_joingamereply       // Game join result
handle_d2gs_updategameinfo      // Game state update
handle_d2gs_echoreq             // D2GS keepalive
```

**bnetd Protocol Handlers** (`handle_bnetd.h/cpp`):
```cpp
handle_bnetd_init               // bnetd handshake
handle_bnetd_authreply          // Auth handshake reply
handle_bnetd_accountloginreply  // Session validation result
handle_bnetd_charloginreply     // Character login ack
```

### 10. Server Operations (`server.h/cpp`)

Main server loop and event processing.

**API**:
```cpp
int d2cs_server_process(void);                      // Main event loop
int d2cs_server_handle_tcp(void*, t_fdwatch_type);  // Handle TCP events
int d2cs_server_handle_accept(void*, t_fdwatch_type); // Accept connections
```

**Event Loop**:
```cpp
while (running) {
    fdwatch(timeout);           // Wait for I/O events
    fdwatch_handle();           // Dispatch event handlers
    
    d2cs_connlist_reap();       // Remove destroyed connections
    d2gslist_check_voidgame();  // Remove empty games
    sqlist_check_timeout();     // Timeout server queue entries
    gqlist_check_timeout();     // Timeout game queue entries
    
    bnetd_keepalive();          // Send bnetd keepalive
    d2gs_keepalive();           // Send D2GS keepalives
}
```

## Configuration

D2CS is configured via `conf/d2cs.conf`:

### Critical Settings

```conf
# Realm name (must match realm.conf in bnetd)
realmname = D2CS

# D2CS listening address and port
servaddrs = 0.0.0.0:6113

# Battle.net daemon address
bnetdaddr = 192.168.1.100:6112

# Game server list (comma-separated IPs)
gameservlist = 192.168.1.101,192.168.1.102

# Maximum connections
max_connections = 1000
```

### Character Storage

```conf
# Character save files directory
charsave_dir = "var/charsave"

# Character info files directory  
charinfo_dir = "var/charinfo"

# Backup directories
bak_charsave_dir = "var/bak/charsave"
bak_charinfo_dir = "var/bak/charinfo"

# Newbie character templates
charsave_newbie_amazon = "files/newbie.save"
# ... (other classes)
```

### Realm Settings

```conf
# Realm type
# 0 = Classic only
# 1 = LOD only
# 2 = Both (default)
lod_realm = 2

# Allow Classic to LOD conversion
allow_convert = 0

# Maximum characters per account
maxchar = 18
```

### Game Settings

```conf
# Maximum games in list
maxgamelist = 200

# Maximum game lifetime (seconds)
game_maxlifetime = 0  # 0 = unlimited

# Maximum game idle time (seconds)
max_game_idletime = 0  # 0 = unlimited

# Maximum character level restriction
game_maxlevel = 255
```

### Ladder Settings

```conf
# Ladder data directory
ladder_dir = "var/ladders"

# Ladder refresh interval (seconds)
d2ladder_refresh_interval = 3600

# Ladder list entry count
ladderlist_count = 100

# Ladder start time (Unix timestamp)
# ladder_start_time = 0
```

### Security

```conf
# Check for multi-login
check_multilogin = 1

# Hide password-protected games from list
hide_pass_games = 0

# Show all games in list (ignore character restrictions)
allow_gamelist_showall = 0

# D2GS authentication password
d2gs_password = ""

# Allowed account name symbols
account_allowed_symbols = "-_[]"
```

### Timeouts

```conf
# Client idle timeout (seconds)
idletime = 300

# Server-to-server timeouts
s2s_timeout = 90
s2s_retryinterval = 5
s2s_keepalive_interval = 60

# Server queue timeout
sq_timeout = 300
sq_checkinterval = 60

# Game queue check interval
gamequeue_checkinterval = 60
```

### Logging

```conf
# Log file location
logfile = "var/logs/d2cs.log"

# Log levels (comma-separated)
# trace,debug,info,warn,error,fatal
loglevels = "info,warn,error"

# PID file
pidfile = "var/d2cs.pid"
```

### Advanced

```conf
# D2GS checksum validation
d2gs_checksum = 0

# D2GS version requirement
d2gs_version = 0

# D2GS restart delay after crash (seconds)
d2gs_restart_delay = 10

# Character expiration time (seconds)
# 0 = never expire
char_expire_time = 0

# Character list sorting
# name, level, experience, class, date
charlist_sort = "level"
charlist_sort_order = "DESC"  # ASC or DESC

# Shutdown delay (seconds)
shutdown_delay = 300
shutdown_decr = 60
```

## Network Protocols

### Client → D2CS Protocol

**Packet Format**:
```cpp
typedef struct {
    bn_short    size;   // Packet size (including header)
    bn_short    type;   // Packet type
    // Payload follows
} t_d2cs_client_header;
```

**Key Packet Types**:
```cpp
CLIENT_D2CS_LOGINREQ            = 0x01  // Login with session
CLIENT_D2CS_CREATECHARREQ       = 0x02  // Create character
CLIENT_D2CS_CREATEGAMEREQ       = 0x03  // Create game
CLIENT_D2CS_JOINGAMEREQ         = 0x04  // Join game
CLIENT_D2CS_GAMELISTREQ         = 0x05  // Request game list
CLIENT_D2CS_GAMEINFOREQ         = 0x06  // Request game info
CLIENT_D2CS_CHARLOGINREQ        = 0x07  // Select character
CLIENT_D2CS_DELETECHARREQ       = 0x0A  // Delete character
CLIENT_D2CS_LADDERREQ           = 0x11  // Request ladder
CLIENT_D2CS_MOTDREQ             = 0x12  // Request MOTD
CLIENT_D2CS_CANCELCREATEGAME    = 0x13  // Cancel game creation
CLIENT_D2CS_CHARLADDERREQ       = 0x16  // Character ladder position
CLIENT_D2CS_CHARLISTREQ         = 0x17  // Character list (v1.09)
CLIENT_D2CS_CONVERTCHARREQ      = 0x18  // Convert to expansion
CLIENT_D2CS_CHARLISTREQ_110     = 0x19  // Character list (v1.10+)
```

### D2CS → Client Replies

```cpp
D2CS_CLIENT_LOGINREPLY          = 0x01  // Login result
D2CS_CLIENT_CREATECHARREPLY     = 0x02  // Character creation result
D2CS_CLIENT_CREATEGAMEREPLY     = 0x03  // Game creation result
D2CS_CLIENT_JOINGAMEREPLY       = 0x04  // Game join result
D2CS_CLIENT_GAMELISTREPLY       = 0x05  // Game list data
D2CS_CLIENT_GAMEINFOREPLY       = 0x06  // Game info data
D2CS_CLIENT_CHARLOGINREPLY      = 0x07  // Character selection result
D2CS_CLIENT_DELETECHARREPLY     = 0x0A  // Delete result
D2CS_CLIENT_LADDERREPLY         = 0x11  // Ladder data
D2CS_CLIENT_MOTDREPLY           = 0x12  // MOTD text
D2CS_CLIENT_CREATEGAMEWAIT      = 0x13  // Game creation queued
D2CS_CLIENT_CHARLADDERREPLY     = 0x16  // Character ladder rank
D2CS_CLIENT_CHARLISTREPLY       = 0x17  // Character list
```

### D2CS ↔ D2GS Protocol

```cpp
// D2CS → D2GS
D2CS_D2GS_AUTHREQ               // Authentication request
D2CS_D2GS_CREATEGAMEREQ         // Create game
D2CS_D2GS_JOINGAMEREQ           // Join game request
D2CS_D2GS_ECHOREPLY             // Keepalive reply

// D2GS → D2CS
D2GS_D2CS_AUTHREPLY             // Auth result
D2GS_D2CS_SETINITINFO           // Server capabilities
D2GS_D2CS_CREATEGAMEREPLY       // Game creation result
D2GS_D2CS_JOINGAMEREPLY         // Join result
D2GS_D2CS_UPDATEGAMEINFO        // Game state update
D2GS_D2CS_ECHOREQ               // Keepalive ping
```

### D2CS ↔ bnetd Protocol

```cpp
// D2CS → bnetd
D2CS_BNETD_ACCOUNTLOGINREQ      // Validate session
D2CS_BNETD_CHARLOGINREQ         // Character login notification
D2CS_BNETD_KEEPALIVE            // Keepalive

// bnetd → D2CS
BNETD_D2CS_AUTHREPLY            // Handshake reply
BNETD_D2CS_ACCOUNTLOGINREPLY    // Session validation result
BNETD_D2CS_CHARLOGINREPLY       // Character login ack
```

## Building and Running

### Build

```bash
# Configure
cmake -B build -DCMAKE_BUILD_TYPE=Release -DWITH_BNETD=ON

# Build
cmake --build build --target d2cs

# Install
cmake --install build
```

### Run

```bash
# Foreground mode
./d2cs -f -c conf/d2cs.conf

# Background daemon (Unix)
./d2cs -c conf/d2cs.conf

# Windows service
d2cs.exe --service
```

### Command-Line Options

```
-c, --config <file>     Configuration file path
-f, --foreground        Run in foreground (don't daemonize)
-l, --log <file>        Override log file
-h, --help              Show help
-v, --version           Show version
```

## Deployment Architecture

### Single Server Setup

```
┌─────────────────────────────┐
│      Single Machine         │
│                             │
│  ┌────────┐  ┌────────┐    │
│  │ bnetd  │←→│  D2CS  │    │
│  └────────┘  └───┬────┘    │
│                  │          │
│               ┌──▼──┐       │
│               │ D2GS│       │
│               └─────┘       │
└─────────────────────────────┘
```

### Distributed Setup

```
┌──────────────┐     ┌──────────────┐
│   Machine 1  │     │  Machine 2   │
│              │     │              │
│  ┌────────┐  │     │  ┌────────┐  │
│  │ bnetd  │◄─┼─────┼─►│  D2CS  │  │
│  └────────┘  │     │  └───┬────┘  │
└──────────────┘     └──────┼───────┘
                             │
                    ┌────────┴────────┐
                    │                 │
         ┌──────────▼──┐   ┌──────────▼──┐
         │  Machine 3  │   │  Machine 4  │
         │             │   │             │
         │  ┌──────┐   │   │  ┌──────┐   │
         │  │ D2GS │   │   │  │ D2GS │   │
         │  └──────┘   │   │  └──────┘   │
         └─────────────┘   └─────────────┘
```

### Load Distribution

**Characters**: ~1000 concurrent players per D2CS instance  
**Games**: ~50-100 games per D2GS instance  
**Bandwidth**: ~5 KB/s per player (D2CS), ~20 KB/s per player (D2GS)

## Troubleshooting

### Common Issues

**1. D2CS won't start**
```
ERROR: cannot connect to bnetd at <address>
```
**Solution**: Ensure bnetd is running and address is correct in `d2cs.conf`

**2. Clients can't log in**
```
ERROR: got bad account session
```
**Solution**: Verify bnetd address matches, check firewall rules

**3. Game creation fails**
```
ERROR: no available d2gs
```
**Solution**: Verify D2GS instances are running, check `gameservlist` in config

**4. Character files missing**
```
ERROR: character not found
```
**Solution**: Check `charsave_dir` and `charinfo_dir` paths, verify permissions

**5. Ladder not updating**
```
WARN: ladder refresh failed
```
**Solution**: Ensure D2DBS is running and accessible

### Debug Mode

Enable detailed logging:
```conf
loglevels = "trace,debug,info,warn,error"
```

Check logs:
```bash
tail -f var/logs/d2cs.log
```

## Performance Tuning

### Connection Limits

```conf
max_connections = 2000  # Increase for more concurrent users
```

### Game Limits

```conf
maxgamelist = 500       # More games in browser
list_purgeinterval = 60 # Remove stale games faster
```

### Timeout Adjustments

```conf
idletime = 600          # Longer idle timeout
sq_timeout = 600        # Longer server queue timeout
```

### System Limits (Unix)

```bash
# Increase file descriptor limit
ulimit -n 4096

# Adjust in /etc/security/limits.conf
d2cs soft nofile 4096
d2cs hard nofile 8192
```

## Security Considerations

1. **Multi-Login Prevention**: Enable `check_multilogin` to prevent account sharing
2. **Character Validation**: D2CS validates character files to prevent hacked items
3. **D2GS Authentication**: Use `d2gs_password` to authenticate game servers
4. **Firewall Rules**: Restrict D2CS port (6113) to game clients only
5. **File Permissions**: Set restrictive permissions on character save directories

## Related Components

- **bnetd**: Battle.net daemon (`src/bnetd`) - Authentication and chat
- **D2DBS**: Diablo II Database Server (`src/d2dbs`) - Character and ladder storage
- **D2GS**: Diablo II Game Server (external) - Game simulation
- **common**: Shared libraries (`src/common`) - Protocol definitions, data structures
- **compat**: Compatibility layer (`src/compat`) - Cross-platform support

## Authors

D2CS was developed by:
- **Onlyer** (onlyer@263.net) - Original implementation
- **sousou** (liupeng.cs@263.net) - Ladder system
- **Olaf Freyer** (aaron@cs.tu-berlin.de) - Enhancements
- **Dizzy** - Maintenance and improvements
- PvPGN community contributors

## License

GNU General Public License v2 or later (see `LICENSE` file in project root).

---

*This README documents the D2CS (Diablo II Character Server) component of PvPGN. D2CS is the central coordinator for Diablo II closed realm gameplay, managing character data, game creation, and server routing.*

**Last updated**: November 2024
