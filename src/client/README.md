# PvPGN Client Tools

## Overview

The `src/client` folder contains a collection of command-line client utilities for interacting with Battle.net servers (PvPGN/bnetd). These tools emulate various aspects of official Battle.net game clients, enabling developers to test server functionality, administrators to perform server management tasks, and users to access Battle.net services from the Unix/Linux command line without needing the full game clients.

## Purpose

The client tools serve multiple purposes:

1. **Server Testing**: Validate server configuration and protocol implementations
2. **Administration**: Perform server administration from command-line interfaces
3. **Protocol Analysis**: Debug and analyze Battle.net protocol behavior
4. **Automation**: Script interactions with Battle.net servers for bots and automation
5. **File Transfer**: Download game files, patches, and resources from servers
6. **Statistics Retrieval**: Query player statistics and ladder rankings
7. **Chat Operations**: Participate in Battle.net chat channels without game clients

## Client Tools

### 1. bnchat - Battle.net Chat Client

**Purpose**: Full-featured text-based chat client for Battle.net servers.

**File**: `bnchat.cpp` (2169 lines)

**Features**:
- Interactive chat in Battle.net channels
- User listing and presence information
- Private messaging (`/whisper`, `/msg`)
- Channel operations (`/join`, `/whoami`, `/who`, `/whois`)
- Game creation and joining
- Account management (create, password change)
- Clan operations (create, invite, motd)
- Friend and ignore lists
- ANSI terminal color support (VT100 compatible)
- Multiple client emulation (see Client Emulation section)

**Key Capabilities**:
```cpp
// Supported modes
mode_chat         // Normal chat interaction
mode_command      // Command input
mode_waitstat     // Waiting for statistics
mode_waitgames    // Waiting for game list
mode_waitladder   // Waiting for ladder ranking
mode_gamename     // Creating game
mode_gamepass     // Setting game password
mode_gamewait     // In game lobby
mode_claninvite   // Clan invitation
```

**Usage**:
```bash
# Basic usage
bnchat [servername] [port]

# With ANSI colors
bnchat -a localhost 6112

# Emulate specific client
bnchat -b                          # Brood War
bnchat -s                          # Starcraft (default)
bnchat -d                          # Diablo
bnchat -w                          # Warcraft II BNE
bnchat --client=D2DV               # Diablo II
bnchat --client=WAR3               # Warcraft III

# Account operations
bnchat -n                          # Create new account
bnchat -c                          # Change password

# Authentication
bnchat -k CDKEY -o OWNER           # Specify CD key and owner
```

**Interactive Commands**:
- `/join <channel>` - Join a channel
- `/whisper <user> <msg>` - Send private message
- `/who` - List users in current channel
- `/whois <user>` - Query user information
- `/stats <user>` - View player statistics
- `/games` - List active games
- `/ladder <type>` - View ladder rankings
- `/away <msg>` - Set away status
- `/dnd <msg>` - Set Do Not Disturb
- `/quit` - Exit client

**Default Channels by Client Type**:
| Client | Channel |
|--------|---------|
| BNCHATBOT | Chat |
| STAR | Starcraft |
| SEXP | Brood War |
| SSHR | Starcraft Shareware |
| DRTL | Diablo Retail |
| DSHR | Diablo Shareware |
| W2BN | War2BNE |
| D2DV/D2XP | Diablo II |
| WAR3/W3XP | W3 |

### 2. bnftp - Battle.net File Transfer Client

**Purpose**: Download files from Battle.net servers (MPQ archives, TOS files, banners).

**File**: `bnftp.cpp` (851 lines)

**Features**:
- Download game update files (MPQ archives)
- Retrieve Terms of Service (`tos.txt`)
- Download ad banners
- Resume interrupted downloads
- File existence handling (ask/overwrite/backup/resume)
- Custom starting offset for partial downloads
- Multiple client emulation

**Usage**:
```bash
# Basic usage
bnftp [servername] [port]

# Specify file directly
bnftp --file=tos.txt localhost

# Handle existing files
bnftp --exists=Overwrite            # Overwrite without asking
bnftp --exists=Resume               # Resume partial download
bnftp --exists=Backup               # Create .bak file
bnftp --exists=Ask                  # Prompt user (default)

# Resume from specific offset
bnftp --startoffset=1024 --file=ver-IX86-1.mpq

# Emulate specific client
bnftp -b --file=IX86ver1.mpq        # Brood War
bnftp --client=D2DV --file=patch.mpq

# Architecture specification
bnftp --arch=IX86                   # Windows x86 (default)
bnftp --arch=PMAC                   # PowerPC Mac
bnftp --arch=XMAC                   # Mac OS X

# Debug mode
bnftp --hexdump=packets.hex localhost
```

**Common Files to Download**:
- `tos.txt` - Terms of Service
- `IX86ver1.mpq` - Version checking data
- `ver-IX86-1.mpq` - Version archive
- `ad000001.smk`/`ad000002.smk` - Ad banners

**File Protocol**: Uses Battle.net file transfer protocol with resume support and integrity checking.

### 3. bnstat - Battle.net Statistics Client

**Purpose**: Query and display player statistics, ladder rankings, and game information.

**File**: `bnstat.cpp` (1242 lines)

**Features**:
- View player win/loss records
- Query character class information (Diablo)
- Display ladder rankings
- Show detailed game statistics
- Parse binary statistic codes
- Support for all Battle.net game types

**Usage**:
```bash
# Basic usage
bnstat [servername] [port]

# Query specific player
bnstat localhost 6112
# Then enter username when prompted

# Emulate different clients
bnstat -s                  # Starcraft (default)
bnstat -b                  # Brood War
bnstat -d                  # Diablo
bnstat -w                  # Warcraft II BNE
bnstat --client=D2DV       # Diablo II

# Authentication
bnstat -k CDKEY -o OWNER
```

**Statistics Displayed**:

**Starcraft/Brood War**:
- Normal games (wins/losses/disconnects)
- Ladder rating
- High ladder rating
- Rank (if ranked)
- Iron Man ladder rating

**Diablo**:
- Character level
- Character class (warrior/rogue/sorcerer)
- Equipment strength
- Dots (game completion indicator)

**Warcraft II**:
- Normal games
- Iron Man games
- Ladder ratings

**Player Information Classes**:
```cpp
PLAYERINFO_DRTL_CLASS_WARRIOR   // Diablo Warrior
PLAYERINFO_DRTL_CLASS_ROGUE     // Diablo Rogue  
PLAYERINFO_DRTL_CLASS_SORCERER  // Diablo Sorcerer
```

### 4. bnbot - Battle.net Bot Client

**Purpose**: Simple bot interface for scripting and automation.

**File**: `bnbot.cpp` (311 lines)

**Features**:
- Raw bot protocol access
- Stdin/stdout interface for scripting
- Compatible with scripting languages (expect, bash, Python)
- Test bot login permissions
- Simple automation framework

**Usage**:
```bash
# Basic usage
bnbot [servername] [port]

# Pipe input/output
echo "/join channel" | bnbot localhost

# Use with expect for automation
expect bot_script.exp

# Test bot account permissions
bnbot pvpgn.example.com 6112
```

**Bot Protocol**:
1. Send `^C` (Control-C) character before any input
2. Input from stdin is sent to server
3. Output from server is sent to stdout
4. No terminal manipulation (unlike bnchat)

**Example Bot Script** (bash):
```bash
#!/bin/bash
(
    echo -ne '\x03'  # Control-C
    sleep 1
    echo "Bot Login"
    sleep 1
    echo "/join support"
    sleep 1
    echo "/who"
    sleep 2
) | bnbot localhost 6112
```

**Use Cases**:
- Automated channel moderation
- Welcome bots
- Game announcements
- Statistics tracking
- Chat logging

## Shared Components

### client.cpp/client.h

**Purpose**: Common client functionality shared across all tools.

**Key Functions**:
```cpp
// Blocking packet I/O
client_blocksend_packet(sock, packet);
client_blockrecv_packet(sock, packet);
client_blockrecv_raw_packet(sock, packet, len);

// Terminal operations
client_get_termsize(fd, &width, &height);  // Get terminal dimensions
client_get_comm(prompt, text, maxlen, ...); // Readline-like input
```

**Features**:
- Blocking packet send/receive
- Terminal size detection (TIOCGWINSZ ioctl)
- Command-line editing with cursor positioning
- Text scrolling for long inputs
- Environment variable fallback (COLUMNS, LINES)

### client_connect.cpp/client_connect.h

**Purpose**: Battle.net connection establishment and authentication.

**Key Functions**:
```cpp
// Connection handling
client_connect(...)        // Establish connection with authentication
client_get_termsize(...)  // Terminal capability detection
key_interpret(...)        // CD key validation/parsing
```

**Authentication Flow**:
```
1. TCP Connection (port 6112)
2. CLIENT_INITCONN (protocol handshake)
3. CLIENT_VERSIONCHECK (version validation)
4. CLIENT_AUTHREQ (authentication request)
   - CD key hashing (if required)
   - Password hashing (old/new protocols)
5. SERVER_AUTHREPLY (success/failure)
6. Join default channel
```

**Version Information**:
Maintains default version data for each client:
```cpp
CLIENT_VERSIONID_STAR   // Starcraft version ID
CLIENT_GAMEVERSION_STAR // Game version number
CLIENT_EXEINFO_STAR     // EXE info string
CLIENT_CHECKSUM_STAR    // Version checksum
```

### udptest.cpp/udptest.h

**Purpose**: UDP connectivity testing for NAT/firewall troubleshooting.

**Key Functions**:
```cpp
client_udptest_setup(progname, &port)     // Create UDP socket
client_udptest_recv(progname, lsock, ...)  // Receive UDP test packets
```

**UDP Test Process**:
1. Bind to port range (BNETD_MIN_TEST_PORT - BNETD_MAX_TEST_PORT)
2. Inform server of UDP port
3. Wait for server echo packet
4. Verify connectivity

Used by `bnchat` and `bnstat` to verify UDP functionality for games and chat features.

### ansi_term.h

**Purpose**: ANSI terminal control for colors and cursor positioning.

**Capabilities**:
- Screen/line clearing
- Cursor movement (up/down/left/right)
- Cursor positioning (absolute, save/restore)
- Text colors (8 colors for foreground/background)
- Text styling (bold, underline, blink, inverse)

**Macros**:
```cpp
ansi_screen_clear()              // Clear entire screen
ansi_cursor_move(row, col)       // Move to position
ansi_text_color_fore(color)      // Set foreground color
ansi_text_color_back(color)      // Set background color
ansi_text_style(style)           // Set text style
ansi_text_reset()                // Reset all attributes
```

**Color Constants**:
```cpp
ansi_text_color_black
ansi_text_color_red
ansi_text_color_green
ansi_text_color_yellow
ansi_text_color_blue
ansi_text_color_magenta
ansi_text_color_cyan
ansi_text_color_white
```

**Requirements**: VT100-compatible terminal (xterm, most Linux terminals)

## Client Emulation

All client tools can emulate different Battle.net game clients. This affects:
- Version checking
- Default channels
- Available statistics
- Protocol features

**Supported Client Tags**:

| Tag | Short Option | Game | Description |
|-----|--------------|------|-------------|
| DRTL | `-d` | Diablo | Diablo Retail |
| DSHR | | Diablo | Diablo Shareware |
| STAR | `-s` | Starcraft | Starcraft (default) |
| SSHR | | Starcraft | Starcraft Shareware |
| SEXP | `-b` | Starcraft | Brood War (expansion) |
| W2BN | `-w` | Warcraft II | Warcraft II BNE |
| D2DV | | Diablo II | Diablo II |
| D2XP | | Diablo II | Diablo II: Lord of Destruction |
| WAR3 | | Warcraft III | Warcraft III: Reign of Chaos |
| W3XP | | Warcraft III | Warcraft III: The Frozen Throne |

**Architecture Tags** (for bnftp):
- `IX86` - Windows (Intel x86) - default
- `PMAC` - Macintosh (PowerPC)
- `XMAC` - Mac OS X

## Protocol Details

### Battle.net Protocol

The clients implement the Battle.net binary protocol:

**Packet Structure**:
```
┌────────────────────────────────────────┐
│  0xFF (protocol marker)       1 byte  │
│  Packet Type                  1 byte  │
│  Packet Length                2 bytes │
│  Payload                      N bytes │
└────────────────────────────────────────┘
```

**Key Packet Types**:
- `CLIENT_INITCONN (0x01)` - Initial connection
- `CLIENT_VERSIONCHECK (0x07)` - Version validation
- `CLIENT_AUTHREQ (0x08)` - Authentication request
- `CLIENT_CHATCOMMAND (0x0e)` - Chat command
- `CLIENT_MESSAGE (0x0f)` - Chat message
- `SERVER_MESSAGE (0x0f)` - Server chat message
- `SERVER_CHANNELLIST (0x0b)` - Channel list
- `SERVER_STATSREPLY (0x26)` - Statistics response

### File Transfer Protocol

Used by `bnftp`:
```
CLIENT_FILE_REQ → [filename]
← SERVER_FILE_REPLY [size, timestamp]
← [file data chunks]
```

### Bot Protocol

Used by `bnbot`:
```
CLIENT: ^C [control character]
CLIENT: [text line]\n
SERVER: [text line]\n
```

## Building

The client tools are built as part of the PvPGN project:

```bash
cd build
cmake ..
make bnchat bnftp bnbot bnstat
```

**Individual Targets**:
```bash
make bnchat    # Chat client
make bnftp     # File transfer client
make bnbot     # Bot client
make bnstat    # Statistics client
```

**CMake Configuration** (`CMakeLists.txt`):
```cmake
add_executable(bnchat bnchat.cpp client.cpp client_connect.cpp 
               udptest.cpp)
target_link_libraries(bnchat common compat fmt ${NETWORK_LIBRARIES})
```

## Dependencies

**Common Libraries**:
- `common/packet.h` - Packet abstraction layer
- `common/bnet_protocol.h` - Battle.net protocol definitions
- `common/init_protocol.h` - Connection initialization
- `common/bn_type.h` - Battle.net data types (byte order)
- `common/bnethash.h` - Password/CD key hashing
- `common/network.h` - Network I/O helpers
- `common/tag.h` - Client tag definitions

**Compatibility Layer**:
- `compat/psock.h` - Portable socket API
- `compat/termios.h` - Terminal control
- `compat/strcasecmp.h` - Case-insensitive string comparison

**System Libraries**:
- POSIX sockets
- Terminal I/O (`termios.h`)
- Standard C++ library

## Terminal Capabilities

### Assumed Keybindings

The clients assume the following terminal keybindings:

| Key | Action |
|-----|--------|
| `^H` (Backspace) | Delete character left of cursor |
| `^J` (Line Feed) | Accept current line |
| `^M` (Return) | Accept current line |
| `^T` | Transpose last two characters |
| `^W` | Delete word left of cursor |
| `^U` | Delete entire line |
| `^[` (Escape) | Cancel current input |
| `^?` (Delete) | Delete character left of cursor |

### Terminal Requirements

For best experience:
- VT100-compatible terminal emulator
- ANSI color support (for `bnchat -a`)
- Minimum 80x24 character display
- UTF-8 encoding support (optional, for extended characters)

### Terminal Size Detection

Clients detect terminal size via:
1. `ioctl(TIOCGWINSZ)` (preferred)
2. `COLUMNS` and `LINES` environment variables
3. Defaults: 80x24

## Usage Examples

### Example 1: Interactive Chat Session

```bash
# Connect with ANSI colors
bnchat -a -s localhost 6112

# At prompts:
Username: testuser
Password: testpass

# In chat:
/join support
Hello, this is a test
/whisper admin Need help with server
/stats testuser
/quit
```

### Example 2: Download Game Files

```bash
# Download Terms of Service
bnftp --file=tos.txt localhost

# Download version check file (Starcraft)
bnftp -s --file=IX86ver1.mpq --exists=Resume

# Download with specific architecture
bnftp --arch=XMAC --client=WAR3 --file=ver-XMAC-1.mpq
```

### Example 3: Query Player Statistics

```bash
# Check Starcraft stats
bnstat -s localhost 6112
# Enter username: ProGamer
# View displayed statistics

# Check Diablo character
bnstat -d localhost
# Enter character name: WarriorKing
# View level, class, equipment
```

### Example 4: Bot Automation

```bash
# Simple echo bot
#!/bin/bash
while true; do
    read line
    echo "You said: $line"
done | bnbot localhost 6112
```

**Python Bot Example**:
```python
#!/usr/bin/env python3
import socket
import sys

sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.connect(('localhost', 6112))

# Send control character
sock.send(b'\x03')

# Bot loop
while True:
    data = sock.recv(1024)
    if not data:
        break
    
    line = data.decode('latin-1').strip()
    print(f"Received: {line}")
    
    # Send response
    if "hello" in line.lower():
        sock.send(b"Hello to you too!\n")
```

## Testing and Debugging

### Debug Mode

Enable debug output by compiling with `CLIENTDEBUG`:
```bash
cmake -DCLIENTDEBUG=ON ..
make
```

Debug output includes:
- Packet type and length
- Protocol handshake details
- Authentication flow
- Raw packet data

### Packet Inspection

Use `--hexdump` option (where available):
```bash
bnftp --hexdump=packets.hex --file=tos.txt localhost
```

Generates hex dump:
```
0000: FF 01 08 00 00 00 00 00    ........
0008: FF 07 36 00 00 00 00 00    ..6.....
...
```

### Common Issues

**Connection Refused**:
- Verify server is running
- Check firewall rules
- Confirm correct port (default: 6112)

**Authentication Failed**:
- Verify account exists
- Check password correctness
- Ensure client tag matches server configuration

**UDP Test Failure**:
- NAT/firewall blocking UDP
- Port range 6112-6119 unavailable
- Server not responding to UDP probes

**Terminal Display Issues**:
- Disable ANSI colors (`bnchat` without `-a`)
- Check terminal type: `echo $TERM`
- Try different terminal emulator

## Integration with PvPGN Server

The client tools are designed to work with PvPGN servers:

```
[bnchat]  ←→  [PvPGN Server]  ←→  [Game Clients]
[bnftp]   ←→  Port 6112 TCP   ←→  [Starcraft]
[bnstat]  ←→  UDP 6112-6119   ←→  [Diablo II]
[bnbot]   ←→                  ←→  [Warcraft III]
```

**Server Configuration** (bnetd.conf):
```conf
# Allow old clients (if needed)
allowed_clients = all

# Enable bot access
bot_account_access = true
```

## Security Considerations

⚠️ **Warnings**:

- **Plaintext Passwords**: Passwords sent in plaintext or weakly hashed
- **No Encryption**: All traffic unencrypted (Battle.net protocol limitation)
- **CD Key Exposure**: CD keys transmitted to server
- **Account Security**: Don't use valuable accounts with these tools on untrusted servers

**Best Practices**:
- Use test accounts only
- Connect to trusted servers only
- Don't share CD keys used with real Battle.net
- Monitor network traffic on shared networks

## Troubleshooting

### Build Issues

**Missing headers**:
```bash
# Install development packages
# Debian/Ubuntu:
sudo apt-get install build-essential libfmt-dev

# Fedora/RHEL:
sudo dnf install gcc-c++ fmt-devel
```

**Link errors**:
```bash
# Ensure proper library path
cmake -DCMAKE_LIBRARY_PATH=/usr/local/lib ..
```

### Runtime Issues

**Terminal garbled after crash**:
```bash
# Reset terminal
reset
# Or manually:
stty sane
```

**Cannot bind UDP port**:
```bash
# Check for conflicts
netstat -ulnp | grep 6112
# Use different port range (modify BNETD_MIN_TEST_PORT)
```

**Segmentation fault**:
- Check server is running and reachable
- Verify packet structures match server version
- Enable debug mode for detailed output

## Advanced Usage

### Scripted Account Creation

```bash
# Create multiple test accounts
for i in {1..10}; do
    echo -e "testuser$i\npassword$i\npassword$i\n/quit" | \
    bnchat -n localhost 6112
done
```

### Automated File Downloads

```bash
# Download all version files
for file in IX86ver1.mpq ver-IX86-1.mpq; do
    bnftp --exists=Overwrite --file=$file localhost
done
```

### Batch Statistics Query

```bash
# Query multiple users
for user in player1 player2 player3; do
    echo $user | bnstat -s localhost > stats_$user.txt
done
```

### Bot Command Processing

```bash
# Command response bot
(
    echo -ne '\x03'
    while read cmd; do
        case "$cmd" in
            "!help")
                echo "Available commands: !help !time !uptime"
                ;;
            "!time")
                echo "Current time: $(date)"
                ;;
            "!uptime")
                echo "Uptime: $(uptime -p)"
                ;;
        esac
    done
) | bnbot localhost 6112
```

## Performance Considerations

- **Blocking I/O**: All clients use blocking I/O (not suitable for high-performance applications)
- **Single Connection**: One connection per client instance
- **Terminal I/O**: Terminal operations may slow down high-frequency operations
- **Memory Usage**: Minimal (< 10MB per client instance)

## Limitations

- No encryption support (protocol limitation)
- No binary file editing (bnftp downloads only)
- No game hosting (bnchat can create but not actually host games)
- No voice/multimedia features
- No graphical interface
- Limited Unicode support (ASCII/Latin-1 primarily)

## Related Tools

- **bnetd/pvpgn**: Game server these clients connect to
- **bnpass**: Password hashing utility (see `src/bnpass`)
- **bnproxy**: Network proxy for packet inspection (see `src/bnproxy`)
- **telnet**: Can be used similarly to bnbot for raw protocol access

## References

- **Man Pages**: `man/bnchat.1`, `man/bnftp.1`, `man/bnbot.1`, `man/bnstat.1`
- **Protocol Definitions**: `src/common/bnet_protocol.h`
- **Client Tags**: `src/common/tag.h`
- **PvPGN Documentation**: Project wiki and README files

## Authors

- Ross Combs (rocombs@cs.nmsu.edu, ross@bnetd.org) - Primary author
- Oleg Drokin (green@ccssu.ccssu.crimea.ua) - Contributions to bnchat

## License

GNU General Public License v2 or later (see `LICENSE` file in project root).

---

*This README documents the PvPGN client tools. Last updated: November 2024*
