# BNETD - Battle.net Daemon

## Overview

`bnetd` is the core game server daemon of the PvPGN (Player vs Player Gaming Network) project. It provides Battle.net protocol emulation and Westwood Online support, allowing classic Blizzard and Westwood games to connect for multiplayer gaming, chat, and matchmaking services.

## Purpose

BNETD serves as a drop-in replacement for Blizzard's Battle.net servers and Westwood's online gaming servers, enabling:

- **Private Server Hosting**: Run your own gaming network for LAN parties or online communities
- **Game Preservation**: Keep classic games playable after official server shutdowns
- **Extended Features**: Add custom functionality beyond what original servers provided
- **Protocol Interoperability**: Support multiple connection protocols (Battle.net, IRC, Telnet, Westwood Online)

## Supported Games

### Blizzard Entertainment
- **StarCraft / Brood War** (1.08 - 1.18.0)
- **Warcraft II: Battle.net Edition** (2.02a, 2.02b)
- **Warcraft III: Reign of Chaos / The Frozen Throne** (1.13a - 1.28.5)
- **Diablo** (1.09, 1.09b)
- **Diablo II / Lord of Destruction** (1.10 - 1.14d)

### Westwood Studios
- **Command & Conquer** series
- **Red Alert** series
- **Tiberian Sun / Firestorm**
- **Yuri's Revenge**
- **Renegade**
- **Nox / Nox Quest**
- **Dune 2000**
- **Emperor: Battle for Dune**

## Architecture

### Main Components

#### Core Server (`server.cpp`, `main.cpp`)
- **Event Loop**: Main select()/poll() loop using `fdwatch` abstraction
- **Signal Handling**: POSIX signal support for graceful shutdown, restart, and save operations
- **Network Management**: Multi-protocol listener setup (Battle.net, IRC, Telnet, UDP)
- **Timer System**: Periodic tasks for user sync, ladder updates, tracker reports
- **Connection Lifecycle**: Accept, process, reap dead connections

#### Connection Management (`connection.cpp`, `connection.h`)
Manages individual client connections with:
- **Connection Classes**: `init`, `bnet`, `file`, `bot`, `telnet`, `irc`, `wol` (Westwood Online), `d2cs_bnetd`, `w3route`
- **Connection States**: `empty`, `initial`, `connected`, `loggedin`, `destroy`, `bot_username`, `bot_password`, `untrusted`
- **Packet Queues**: Input/output buffering for network data
- **Rate Limiting**: Quota management to prevent abuse

#### Protocol Handlers
Each protocol has dedicated packet handlers:
- `handle_bnet.cpp` - Battle.net binary protocol
- `handle_irc.cpp` - IRC text-based protocol
- `handle_telnet.cpp` - Telnet admin interface
- `handle_wol.cpp` - Westwood Online (IRC-based)
- `handle_file.cpp` - File transfer protocol (FTP-like)
- `handle_bot.cpp` - Bot protocol
- `handle_udp.cpp` - UDP packets (game discovery, etc.)
- `handle_d2cs.cpp` - Diablo II Character Server communication
- `handle_anongame.cpp` - Anonymous game matchmaking (AT games)

### Data Management

#### Account System (`account.cpp`, `account.h`)
- User authentication and authorization
- Profile storage (wins, losses, stats, ladder rankings)
- Friends lists and social features
- Clan membership tracking
- Attribute-based storage abstraction

#### Storage Backends (`storage.cpp`, `storage_*.cpp`)
Pluggable storage architecture supporting:
- **Plain Files** (`storage_file.cpp`) - Traditional file-based storage
- **SQL Databases** (`storage_sql.cpp`) - MySQL, PostgreSQL, SQLite3, ODBC
- **Abstraction Layer** - Unified API regardless of backend

#### Channel System (`channel.cpp`, `channel.h`)
- Public/private chat channels
- Moderated channels
- Clan channels
- Country-specific channels
- Game lobbies
- Channel flags: `public`, `moderated`, `restricted`, `official`, `permanent`, `clan`, etc.

#### Game Management (`game.cpp`, `game.h`)
- Game creation and lifecycle
- Game types: `melee`, `ffa`, `ladder`, `topvbot`, `ctf`, `team*`, etc.
- Game status tracking: `open`, `full`, `started`, `loaded`, `done`
- Result reporting for ladder systems
- Diablo 2 Open/Closed game support
- Anonymous matchmaking integration

#### Ladder System (`ladder.cpp`, `ladder_calc.cpp`)
- Ranking calculations for competitive play
- War3 solo/team ladder support
- StarCraft ladder tracking
- Periodic ladder updates and saves

### Feature Modules

#### Clans (`clan.cpp`)
- Clan creation and management
- Member ranks and permissions
- Clan channels and messaging

#### Friends System (`friends.cpp`)
- Friend list management
- Online status tracking
- Friend-to-friend messaging

#### Character Management (`character.cpp`)
- Diablo II character storage
- Character validation

#### Version Checking (`versioncheck.cpp`)
- Client version validation
- Anti-cheat mechanisms
- CheckRevision (CR) support
- Game exe hash verification

#### Auto-Update (`autoupdate.cpp`)
- MPQ file distribution
- Client patching support

#### Ad Banners (`adbanner.cpp`)
- In-client advertising system
- Banner rotation

#### Mail System (`mail.cpp`)
- Offline messaging
- In-game mail delivery

#### News/MOTD (`news.cpp`)
- Message of the day
- Server announcements

#### Realms (`realm.cpp`)
- Diablo II realm support
- Diablo II Character Server (D2CS) integration

#### Tournament Support (`tournament.cpp`)
- Organized competition features

#### IP Banning (`ipban.cpp`)
- IP-based access control
- Temporary and permanent bans

#### Command System (`command.cpp`, `command_groups.cpp`)
- In-game slash commands
- Permission-based command access
- Alias support (`alias_command.cpp`)

### Extension Systems

#### Lua Scripting (`luainterface.cpp`, `luafunctions.cpp`)
- Server-side scripting support
- Event hooks for:
  - Client connections
  - Channel joins
  - Command execution
  - Game events
  - User actions
- Custom game logic
- Anti-cheat plugins
- Quiz/minigame systems

#### Internationalization (`i18n.cpp`)
- Multi-language support
- Locale-based messaging

#### User Logging (`userlog.cpp`)
- Activity tracking
- Audit trails

### Network Architecture

#### Address Management
- Multi-homing support (listen on multiple IPs/interfaces)
- IPv4 support (IPv6 planned)
- Port configuration for each protocol
- Address translation for NAT scenarios

#### Event-Driven I/O
- `fdwatch` abstraction layer (select/poll/epoll/kqueue)
- Non-blocking socket I/O
- Efficient handling of thousands of concurrent connections

#### Packet Processing
- Binary protocol parsing
- Packet validation and security checks
- Queue management for reliable delivery

### Configuration

Main configuration file: `conf/bnetd.conf`

Key settings:
- **Storage backend selection** (file, SQL)
- **Database connection strings**
- **Listen addresses and ports**
- **Security settings** (max connections, timeouts)
- **Feature toggles** (tracking, Lua, ladder, clans)
- **File paths** (motd, helpfiles, maps, MPQs)
- **Logging levels**

Additional config files:
- `channel.conf` - Channel definitions
- `command_groups.conf` - Command permissions
- `realm.conf` - Diablo II realm configuration
- `versioncheck.json` - Client version validation
- `tournament.conf` - Tournament rules
- Various game-specific configs

## Build System

Uses CMake for cross-platform compilation:

```cmake
# CMakeLists.txt compiles bnetd executable
add_executable(bnetd ${BNETD_SOURCES})

# Links against:
# - common (shared utilities)
# - compat (portability layer)
# - fmt (formatting library)
# - Optional: MySQL, PostgreSQL, SQLite3, ODBC, Lua
```

Compiler requirements:
- C++11 compliant compiler
- Visual Studio 2015+ / GCC 5.1+ / Clang 3.4+

## Runtime Flow

### Startup Sequence
1. Parse command line arguments (`cmdline.cpp`)
2. Load configuration (`prefs.cpp`)
3. Initialize storage backend (`storage.cpp`)
4. Load accounts, channels, clans, realms
5. Initialize Lua environment (if enabled)
6. Setup network listeners (Battle.net, IRC, Telnet, UDP)
7. Install signal handlers
8. Enter main event loop (`_server_mainloop`)

### Main Event Loop
```cpp
for (;;) {
    // Update current time
    now = time(NULL);
    
    // Check for shutdown signal
    if (should_exit) break;
    
    // Handle reload/restart signals
    if (do_restart) reload_configs();
    if (do_save) save_accounts();
    
    // Periodic tasks
    if (time_for_user_sync) save_accounts();
    if (time_for_ladder_update) update_ladders();
    if (time_for_tracker_report) send_tracker_report();
    
    // Wait for network events
    fdwatch(timeout);
    
    // Handle ready sockets
    fdwatch_handle();
    
    // Cleanup dead connections
    connlist_reap();
}
```

### Connection Handling
1. `handle_accept()` - New TCP connection
2. `conn_create()` - Allocate connection structure
3. Protocol detection → set `conn_class`
4. State machine progression:
   - `conn_state_initial` → Protocol handshake
   - `conn_state_connected` → Authentication
   - `conn_state_loggedin` → Active session
5. `handle_tcp()` / `handle_udp()` - Process packets
6. Route to protocol-specific handler
7. `conn_destroy()` - Cleanup on disconnect

### Packet Routing
```cpp
handle_tcp() 
  → packet_parse()
  → switch (conn_class)
       case conn_class_bnet: → handle_bnet_packet()
       case conn_class_irc:  → handle_irc_packet()
       case conn_class_telnet: → handle_telnet_packet()
       // etc.
```

### Shutdown Sequence
1. Set exit time via signal or command
2. Announce shutdown to users
3. Wait for connections to close gracefully
4. Force save all accounts
5. Cleanup resources
6. Close storage backend
7. Shutdown network listeners
8. Exit

## Key Namespaces

All bnetd code lives in:
```cpp
namespace pvpgn {
    namespace bnetd {
        // All classes, functions, types
    }
}
```

## Threading Model

BNETD is **single-threaded** with:
- Event-driven architecture
- Non-blocking I/O
- No mutexes or locks needed
- Simplifies debugging and development

## Security Features

- **Rate Limiting**: Prevent flooding attacks
- **IP Banning**: Block malicious hosts
- **Version Checking**: Validate client integrity
- **Password Hashing**: Secure credential storage
- **Command Permissions**: Role-based access control
- **Connection Limits**: Prevent resource exhaustion

## Extensibility

### Adding New Protocol Support
1. Create `handle_newproto.cpp`
2. Add `conn_class_newproto` to connection types
3. Implement packet handler
4. Register in `server.cpp` listen setup
5. Update configuration for new port

### Adding New Commands
1. Add function to `command.cpp`
2. Register in command table
3. Set permissions in `command_groups.conf`
4. Add help text to `helpfile`

### Adding Lua Hooks
1. Extend `luafunctions.cpp` with new C++ → Lua bindings
2. Expose events in `luainterface.cpp`
3. Implement handlers in `lua/*.lua`

## Debugging

Enable comprehensive logging in `bnetd.conf`:
```
loglevels = fatal,error,warn,info,debug,trace
```

Logs include:
- Connection lifecycle events
- Packet send/receive (with hexdump)
- Game creation/destruction
- Account operations
- Error conditions

## Integration Points

### Diablo II Character Server (D2CS)
- BNETD handles authentication and chat
- D2CS manages characters and game servers
- Communication via `conn_class_d2cs_bnetd`

### Diablo II Database Server (D2DBS)
- Character storage backend
- Accessed by D2CS, not directly by BNETD

### Tracking Servers
- Optional telemetry reporting
- Server population statistics
- Can be disabled in config

## Performance Characteristics

- **Connections**: Handles thousands of concurrent connections
- **Memory**: Efficient per-connection overhead (~few KB each)
- **CPU**: Minimal when idle, scales with active users
- **I/O**: Network-bound, not CPU-bound

## Development Guidelines

### Code Style
- Follow C++ Core Guidelines
- Use C++11 features where appropriate
- Match existing code formatting
- No code duplication - refactor into common functions

### Memory Management
- Allocate and free in same function when possible
- Use RAII patterns
- Avoid manual memory management where possible
- Check all allocation returns

### Error Handling
- Use `eventlog()` for all logging
- Return error codes consistently
- Clean up resources on error paths

### Testing
- Test with actual game clients
- Verify protocol compatibility
- Check edge cases (disconnect during game, etc.)

## Related Documentation

- **Root README.md** - Project overview and build instructions
- **README.DEV** - General development guidelines
- **docs/** - Protocol documentation, storage formats
- **conf/** - Configuration file examples and templates

## Contributors

BNETD originated from the BNETD project and has been extensively developed by the PvPGN community. See CREDITS file for full attribution.

## License

GNU General Public License v2.0 - See LICENSE file for details.
