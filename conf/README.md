# conf - PvPGN Server Configuration Files

## Overview

The `conf` folder contains all configuration files required for PvPGN (Player vs Player Gaming Network) server operation. These files control every aspect of server behavior, from network settings and user authentication to game policies, channels, and localization. Configuration files use the `.in` extension during development and are processed by CMake during build to generate the final runtime configuration files.

PvPGN supports three main server components, each with dedicated configuration:
- **bnetd** - Main Battle.net server (game clients)
- **d2cs** - Diablo II Character Server (character management)
- **d2dbs** - Diablo II Database Server (character storage)

## Architecture

### Configuration Processing

```
Development            Build Phase                Runtime
┌──────────────┐      ┌──────────────┐          ┌──────────────┐
│ *.conf.in    │──────│ CMake        │──────────│ *.conf       │
│ *.json.in    │      │ Configure    │          │ *.json       │
│ *.txt.in     │      │              │          │ *.txt        │
│              │      │ - Replace    │          │              │
│ Variables:   │      │   ${VARS}    │          │ Final files  │
│ ${SYSCONFDIR}│      │ - Set paths  │          │ installed to │
│ ${LOCALSTATEDIR}    │ - Generate   │          │ /etc/pvpgn   │
│              │      │   line ends  │          │              │
└──────────────┘      └──────────────┘          └──────────────┘
```

### Directory Structure

```
conf/
├── Core Server Configuration
│   ├── bnetd.conf.in           - Main server settings
│   ├── bnetd.conf.win32        - Windows-specific defaults
│   ├── d2cs.conf.in            - Diablo II Character Server
│   ├── d2cs.conf.win32         - Windows D2CS defaults
│   ├── d2dbs.conf.in           - Diablo II Database Server
│   └── d2dbs.conf.win32        - Windows D2DBS defaults
│
├── Network & Security
│   ├── address_translation.conf.in  - NAT/firewall IP translation
│   ├── bnban.conf.in               - IP ban list
│   ├── realm.conf.in               - Diablo II realm definitions
│   └── versioncheck.json.in        - Client version validation
│
├── User & Authentication
│   ├── bnetd_default_user.plain.in - Default account template
│   ├── command_groups.conf.in      - Command permissions
│   ├── sql_DB_layout.conf.in       - Database schema
│   └── supportfile.conf.in         - Support ticket system
│
├── Channels & Games
│   ├── channel.conf.in             - Permanent channel definitions
│   ├── topics.conf.in              - Channel topics
│   ├── bnmaps.conf.in              - Map list and checksums
│   ├── anongame_infos.conf.in      - Anonymous matchmaking URLs
│   ├── tournament.conf.in          - Tournament settings
│   ├── bnxplevel.conf.in           - W3 experience/level table
│   └── bnxpcalc.conf.in            - W3 XP calculation formulas
│
├── Content & Display
│   ├── ad.json.in                  - Advertisement banners
│   ├── bnissue.txt.in              - Login banner (MOTD)
│   ├── bnalias.conf.in             - Command aliases
│   ├── icons.conf.in               - Custom user icons
│   └── autoupdate.conf.in          - Client auto-update files
│
├── Localization (i18n/)
│   ├── common.xml                  - Shared localized strings
│   ├── bnmotd.txt                  - Message of the day
│   ├── w3motd.txt                  - Warcraft 3 MOTD
│   ├── news.txt                    - News feed
│   ├── bnhelp.conf                 - Command help text
│   ├── termsofservice.txt          - Terms of service
│   ├── newaccount.txt              - New account welcome
│   ├── chathelp-war3.txt           - W3 chat help
│   └── [Language Folders]          - Per-language overrides
│       ├── enUS/  (English US)
│       ├── deDE/  (German)
│       ├── frFR/  (French)
│       ├── esES/  (Spanish)
│       ├── ruRU/  (Russian)
│       ├── koKR/  (Korean)
│       ├── zhCN/  (Chinese Simplified)
│       ├── zhTW/  (Chinese Traditional)
│       ├── itIT/  (Italian)
│       ├── jaJA/  (Japanese)
│       ├── plPL/  (Polish)
│       ├── ptBR/  (Portuguese Brazilian)
│       ├── nlNL/  (Dutch)
│       ├── csCZ/  (Czech)
│       ├── svSE/  (Swedish)
│       └── bgBG/  (Bulgarian)
│
└── Build System
    └── CMakeLists.txt              - Configuration installation rules
```

## Core Configuration Files

### 1. bnetd.conf - Main Server Configuration

The primary configuration file for the PvPGN Battle.net server.

**Key Sections**

#### Storage Configuration

```ini
# File-based storage (default)
storage_path = "file:mode=plain;dir=${LOCALSTATEDIR}/users;clan=${LOCALSTATEDIR}/clans;team=${LOCALSTATEDIR}/teams;default=${SYSCONFDIR}/bnetd_default_user.plain"

# MySQL storage
storage_path = "sql:mode=mysql;host=127.0.0.1;name=PVPGN;user=pvpgn;pass=password;default=0;prefix=pvpgn_"

# PostgreSQL storage
storage_path = "sql:mode=pgsql;host=127.0.0.1;name=pvpgn;user=pvpgn;pass=password;default=0;prefix=pvpgn_"

# SQLite storage
storage_path = "sql:mode=sqlite3;name=${LOCALSTATEDIR}/users.db;default=0;prefix=pvpgn_"

# ODBC storage (Windows)
storage_path = "sql:mode=odbc;name=PVPGN;prefix=pvpgn_"
```

**Storage Path Components**:
- `mode` - Driver: plain (files), mysql, pgsql, sqlite3, odbc
- `dir` - User account directory (file mode)
- `clan` - Clan data directory (file mode)
- `team` - Team data directory (file mode)
- `default` - Template for new accounts
- `host` - Database server IP (SQL modes)
- `port` - Database TCP port (optional)
- `name` - Database name
- `user` - Database username
- `pass` - Database password
- `prefix` - Table name prefix (prevents conflicts)

#### File Paths

```ini
filedir     = "${LOCALSTATEDIR}/files"      # MPQ files, updates
scriptdir   = "${LOCALSTATEDIR}/lua"        # Lua scripts
reportdir   = "${LOCALSTATEDIR}/reports"    # Game reports
chanlogdir  = "${LOCALSTATEDIR}/chanlogs"   # Channel logs
userlogdir  = "${LOCALSTATEDIR}/userlogs"   # User chat logs
i18ndir     = "${SYSCONFDIR}/i18n"          # Localization files
logfile     = "${LOCALSTATEDIR}/bnetd.log"  # Server log
ladderdir   = "${LOCALSTATEDIR}/ladders"    # Ladder rankings
statusdir   = "${LOCALSTATEDIR}/status"     # Server status
```

**Path Variables** (replaced by CMake):
- `${SYSCONFDIR}` - Configuration directory (e.g., `/etc/pvpgn`)
- `${LOCALSTATEDIR}` - Variable data directory (e.g., `/var/pvpgn`)

#### Network Settings

```ini
# Server listening addresses
servaddrs = 0.0.0.0:6112

# Multiple addresses (comma-separated)
servaddrs = 0.0.0.0:6112, 192.168.1.100:6112

# IPv6 support
servaddrs = [::]:6112

# IRC server
irc_addrs = 0.0.0.0:6667

# Telnet administration
telnet_addrs = 127.0.0.1:8888

# Warcraft 3 routing (PG/AT games)
w3route_addr = 0.0.0.0:6200
```

#### Client Policy

```ini
# Allowed game clients (comma-separated or "all")
allowed_clients = all
# Or specific: war3,w3xp,star,sexp,drtl,dshr,d2dv,d2xp,w2bn

# Version checking
allow_bad_version = true          # Allow failed checksums
allow_unknown_version = true      # Allow unlisted versions

# Account creation
new_accounts = true               # Allow new registrations
max_accounts = 0                  # Maximum accounts (0 = unlimited)

# Login behavior
kick_old_login = true             # Kick existing session on re-login

# Game visibility
hide_pass_games = true            # Hide password-protected games
hide_started_games = true         # Hide already-started games
```

#### Timing Settings

```ini
# Account synchronization
usersync = 300                    # Save accounts every 5 minutes
userflush = 3600                  # Unload inactive accounts after 1 hour
userstep = 100                    # Accounts to check per sync cycle
userflush_connected = true        # Flush connected users

# Network keepalive
latency = 600                     # Latency test interval (10 minutes)
nullmsg = 120                     # Keepalive packet interval (2 minutes)

# Shutdown
shutdown_delay = 300              # Delay before shutdown (5 minutes)
shutdown_decr = 60                # Reduce delay per signal (1 minute)
```

#### Logging

```ini
# Log levels (comma-separated)
loglevels = fatal,error,warn,info,debug,trace

# Production setting
loglevels = fatal,error,warn,info

# Log detail options
log_commands = true               # Log all commands
log_command_groups = 1            # Command groups to log (bitmask)
```

#### Localization

```ini
# Localized content files
localizefile = common.xml         # Shared strings
motdfile = bnmotd.txt             # Message of the day
motdw3file = w3motd.txt           # Warcraft 3 MOTD
newsfile = news.txt               # News feed
helpfile = bnhelp.conf            # Command help
tosfile = termsofservice.txt      # Terms of service

# Localization method
localize_by_country = true        # By country (false = by game language)
```

### 2. d2cs.conf - Diablo II Character Server

Manages Diablo II character data and coordinates with game servers.

**Key Settings**

```ini
# Realm identifier (must match realm.conf)
realmname = D2CS

# Server listening address
servaddrs = 0.0.0.0:6113

# Diablo II Game Server IPs (comma-separated)
# WARNING: Do NOT use 127.0.0.1 or localhost
gameservlist = 192.168.1.100,192.168.1.101

# Main bnetd server address
bnetdaddr = 192.168.1.50:6112

# Connection limits
max_connections = 1000

# Realm type
# 0 = Classic only
# 1 = Expansion (LOD) only
# 2 = Both (default)
lod_realm = 2

# Character conversion (Classic → Expansion)
allow_convert = 0                 # 0 = disabled, 1 = enabled

# Account name symbols (must match bnetd.conf)
account_allowed_symbols = "-_[]"
```

**Network Topology**

```
Client → bnetd:6112 → d2cs:6113 → d2gs (game server)
                  ↓                  ↓
              Character           Character
              Selection           Storage
                                 (d2dbs:6114)
```

### 3. d2dbs.conf - Diablo II Database Server

Stores Diablo II character files and ladder data.

**Key Settings**

```ini
# Server listening address
servaddrs = 0.0.0.0:6114

# Game server IPs (must match d2cs.conf)
gameservlist = 192.168.1.100,192.168.1.101

# Storage directories
charsavedir = "${LOCALSTATEDIR}/charsave"      # Character saves
charinfodir = "${LOCALSTATEDIR}/charinfo"      # Character info
ladderdir = "${LOCALSTATEDIR}/ladders"         # Ladder data
bak_charsavedir = "${LOCALSTATEDIR}/bak/charsave"  # Backups
bak_charinfodir = "${LOCALSTATEDIR}/bak/charinfo"  # Backups

# Ladder settings
laddersave_interval = 3600        # Save ladder every hour
ladderinit_time = 0               # Ladder init delay
XML_ladder_output = 0             # Export XML ladder (0=off, 1=on)

# Connection management
idletime = 300                    # Idle timeout (5 minutes)
keepalive_interval = 60           # Keepalive interval
timeout_checkinterval = 60        # Timeout check interval
```

## Network & Security Configuration

### address_translation.conf - NAT/Firewall Translation

Critical for servers behind NAT or firewalls. Translates internal IPs to external IPs for clients outside the local network.

**Use Cases**:
- Server inside LAN with external players
- Multiple network interfaces
- Port forwarding through router
- VPN or proxy configurations

**Format**

```
# input (ip:port)   output (ip:port)   exclude (ip/netmask)    include (ip/netmask)
```

**Examples**

```ini
# W3 route server translation
# Internal: 0.0.0.0:6200
# External: 1.2.3.4:6200 (public IP)
# Exclude: 192.168.0.0/24 (local LAN)
0.0.0.0:6200      1.2.3.4:6200      192.168.0.0/24           ANY

# Multiple subnets
0.0.0.0:6112      10.0.1.5:6112     192.168.0.0/24,10.0.0.0/8  ANY

# VPN users get different IP
0.0.0.0:6112      172.16.1.1:6112   NONE                     10.8.0.0/24
```

**Network Specification**:
- `NONE` - No networks (0.0.0.0/32)
- `ANY` - All networks (0.0.0.0/0)
- `x.y.z.t/bitlen` - CIDR notation (e.g., 192.168.0.0/24)

**Processing Order**:
1. Lines searched top to bottom
2. Exclude networks checked first
3. Include networks checked second
4. First match wins
5. No match = return input unchanged

### versioncheck.json - Client Version Validation

Validates game client versions and checksums to prevent cheating and ensure compatibility.

**Structure**

```json
{
    "WAR3": {                           // Client tag
        "IX86": {                       // Architecture (IX86, PMAC, XMAC)
            "0x1c": {                   // Version code (hex)
                "checkRevisionFile": "ver-IX86-1.mpq",
                "equation": "A=1239576727 C=1604096186 B=4198521212 4 A=A+S B=B-C C=C^A A=A+B",
                "entries": [
                    {
                        "title": "Warcraft III - ROC 1.28.2",
                        "version": "1.28.2.227",
                        "hash": "0x2dc96bf5",
                        "fileMetadata": "War3.exe 05/20/17 13:25:29 558568",
                        "versionTag": "WAR3_1282"
                    }
                ]
            }
        }
    }
}
```

**Components**:
- **checkRevisionFile** - MPQ file for version check (must exist in filedir)
- **equation** - Checksum formula (used by client)
- **hash** - Expected checksum result
- **fileMetadata** - File signature (name, date, time, size)
- **versionTag** - Identifier for autoupdate.conf

**Supported Client Tags**:
- `WAR3` - Warcraft III: Reign of Chaos
- `W3XP` - Warcraft III: The Frozen Throne
- `STAR` - Starcraft
- `SEXP` - Starcraft: Brood War
- `D2DV` - Diablo II
- `D2XP` - Diablo II: Lord of Destruction
- `W2BN` - Warcraft II: Battle.net Edition
- `DRTL` - Diablo Retail
- `DSHR` - Diablo Shareware

**Architecture Tags**:
- `IX86` - Intel x86 (Windows)
- `PMAC` - PowerPC Mac
- `XMAC` - Intel Mac

### bnban.conf - IP Ban List

Manages IP address bans with optional expiration times.

**Format**

```
# Permanent ban
192.168.1.100

# Temporary ban (expires at Unix timestamp)
192.168.1.101 1735689600

# Subnet ban (CIDR notation)
10.0.0.0/8

# Range ban
192.168.1.1-192.168.1.254
```

**Ban Types**:
- **Permanent** - Single IP, no expiration
- **Temporary** - Single IP with timestamp
- **Subnet** - CIDR block (e.g., /24 = 256 IPs)
- **Range** - IP range with start-end

**Commands**:
```
/ipban add <ip> [duration]    - Add ban
/ipban del <ip>               - Remove ban
/ipban list                   - List all bans
/ipban check <ip>             - Check if IP is banned
```

## User & Authentication

### command_groups.conf - Command Permission System

Defines which commands are available to different user privilege levels.

**Group System**

```
Group 1: Regular users (default)
Group 2-7: Custom operator/moderator tiers
Group 8: Administrators (full access)
```

**Bitmask Authorization**: Groups use bitwise flags (1, 2, 4, 8, 16, 32, 64, 128)

**Format**

```
# <group>  <command>  [<command2>  <command3>  ...]
```

**Examples**

```ini
# Group 1 - All users
1  /help /time /uptime /stats
1  /whisper /w /m /msg
1  /join /channel /rejoin
1  /who /whois /whereis
1  /away /dnd /ignore /squelch
1  /friends /f /clan /c

# Group 2 - Moderators
2  /kick /ban /unban
2  /announce
2  /moderate /unmoderate

# Group 8 - Admins
8  /shutdown /rehash
8  /admin /op /deop
8  /ipban /accounts
```

**Managing Permissions**

```bash
# List user's groups
/commandgroups list <username>
/cg list <username>

# Add groups (2, 3, 4)
/commandgroups add <username> 234
/cg add <username> 234

# Remove groups (2, 4)
/commandgroups del <username> 24
/cg del <username> 24
```

**Storage**:
- **File mode**: `"BNET\\auth\\command_groups"="255"` in user file
- **SQL mode**: `UPDATE BNET SET auth_command_groups='255' WHERE uid='userid'`

### bnetd_default_user.plain - Default Account Template

Template for new user accounts (file storage mode).

**Structure**

```ini
# Account metadata
"BNET\\acct\\username"="NewUser"
"BNET\\acct\\userid"="1"
"BNET\\acct\\ctime"="0"

# Authentication
"BNET\\auth\\admin"="false"
"BNET\\auth\\normallogin"="true"
"BNET\\auth\\changepass"="true"
"BNET\\auth\\changeprofile"="true"
"BNET\\auth\\botlogin"="false"
"BNET\\auth\\operator"="false"
"BNET\\auth\\command_groups"="1"
"BNET\\auth\\lock"="false"
"BNET\\auth\\mute"="false"

# Game statistics (all start at 0)
"Record\\STAR\\0\\wins"="0"
"Record\\STAR\\0\\losses"="0"
"Record\\W3XP\\solo\\wins"="0"
"Record\\W3XP\\solo\\xp"="0"
```

**Important**: Default users must have `command_groups="1"` to use basic commands.

### sql_DB_layout.conf - Database Schema

Defines SQL table structure for database storage modes.

**Tables**

```
${prefix}BNET      - User accounts and authentication
${prefix}WOL       - Westwood Online data
${prefix}friend    - Friend lists
${prefix}profile   - User profiles
${prefix}Record    - Game statistics
${prefix}clan      - Clan information
${prefix}team      - Team data
```

**Schema Example**

```ini
[${prefix}BNET]
"uid int NOT NULL PRIMARY KEY","'0'"
"acct_username varchar(32)","NULL"
"username varchar(32)","NULL" && "UPDATE ${prefix}BNET SET username = lower(acct_username)"
"acct_passhash1 varchar(40)","NULL"
"acct_email varchar(128)","NULL"
"auth_admin varchar(6)","'false'"
"auth_command_groups varchar(16)","'1'"
"acct_lastlogin_time int","'0'"
:"CREATE UNIQUE INDEX username2 ON ${prefix}BNET (username)"
```

**Syntax**:
- `"column definition","default value"` - Create column
- `&& "sql"` - Execute on success
- `|| "sql"` - Execute on failure
- `:"sql"` - Execute unconditionally

**Prefix**: Configured in `storage_path` (e.g., `prefix=pvpgn_`)

## Channels & Games

### channel.conf - Permanent Channel Definitions

Defines channels that exist when the server starts.

**Format**

```
# special_name    short_name    cltag  bots  ops  log  ctry  realm  max  mod
```

**Columns**:
1. **special_name** - Display name (or NONE for auto-generated)
2. **short_name** - Internal identifier
3. **cltag** - Client tag restriction (NULL = all)
4. **bots** - Allow bots (true/false)
5. **ops** - Auto-grant operator status (true/false)
6. **log** - Enable channel logging (true/false)
7. **ctry** - Country code (NULL = all)
8. **realm** - Realm restriction (NULL = all)
9. **max** - Maximum users (-1 = unlimited)
10. **mod** - Moderated channel (true/false)

**Examples**

```
"The Void"                  "The Void"    NULL   true   false  false  NULL  NULL  -1   true
NONE                        "Starcraft"   STAR   true   false  false  NULL  NULL  -1   false
"Warcraft 3 Frozen Throne"  "W3"          W3XP   true   false  false  NULL  NULL  -1   false
"Moderated Support"         "Support"     NULL   true   false  false  NULL  NULL  -1   true
```

**Special Name Formats**:
- `NONE` - Auto-generate as "shortname-ctry num realm" (e.g., "D2CS-Starcraft USA-1")
- `"Custom Name"` - Fixed display name

**Westwood Online Channels**

WOL channels use special `"Lob xx x"` format:
- `xx` = Game type number
- `x` = Channel number (starts at 0)

```
"Tiberian Sun-1"   "Lob 18 0"   TSUN   true   false  false  NULL  NULL  -1   false
"Red Alert 2-1"    "Lob 33 0"   RAL2   true   false  false  NULL  NULL  -1   false
"Yuri's Revenge-1" "Lob 41 0"   YURI   true   false  false  NULL  NULL  -1   false
```

**Game Type Numbers**:
- 12 = Renegade
- 14 = Dune 2000
- 16 = Nox
- 18 = Tiberian Sun
- 21 = Red Alert 1
- 31 = Emperor: Battle for Dune
- 33 = Red Alert 2
- 37 = Nox Quest
- 41 = Yuri's Revenge

### anongame_infos.conf - Anonymous Matchmaking

Configures ladder URLs and game type descriptions for Warcraft 3 matchmaking.

**Sections**

```ini
[URL]
server_URL  = "http://your-site.com"
player_URL  = "http://your-site.com/?user="
clan_URL    = "http://your-site.com/?clan="

ladder_PG_1v1_URL  = "http://your-site.com/?type=solo"
ladder_AT_2v2_URL  = "http://your-site.com/?type=at2v2"

[DEFAULT_DESC]
ladder_PG_1v1_desc = "Solo Games"
gametype_1v1_short = "One vs. One"
gametype_1v1_long  = "Two players fight to the death"

[deDE]  # German localization
ladder_PG_1v1_desc = "Solo-Spiele"
gametype_1v1_short = "Einer gegen Einen"
```

**Ladder Types**:
- **PG** - Played Games (solo, FFA, team)
- **AT** - Arranged Team (2v2, 3v3, 4v4)
- **Clan** - Clan wars (1v1, 2v2, 3v3, 4v4)

### tournament.conf - Tournament System

Configures automated tournament schedules and rules.

**Settings**

```ini
# Time format: "MM/DD/YYYY HH:MM:SS"
start_preliminary = "08/01/2003 08:00:00"
end_signup        = "08/01/2003 09:00:00"
end_preliminary   = "08/01/2003 10:00:00"
start_round_1     = "08/01/2003 11:00:00"
tournament_end    = "08/01/2003 15:00:00"

# Game selection: 1=PG, 2=AT
game_selection = 1

# Game type: 1=1v1, 2=2v2, 3=3v3, 4=4v4
game_type = 1

# Game client: 1=RoC, 2=FT
game_client = 2

# Available races (quotes required)
races = "HONUR"  # H=Human, O=Orc, N=Night Elf, U=Undead, R=Random

# Tournament info
format  = "PvPGN 1v1 Tournament"
sponsor = "The PvPGN Team"
icon    = "H3"  # See file for icon codes
```

**Icon Codes** (Warcraft 3):
- `R` = Random
- `H2` = Rifleman, `H3` = Sorceress, `H4` = Spellbreaker
- `O2` = Troll, `O3` = Shaman, `O4` = Spirit Walker
- `U2` = Crypt Fiend, `U3` = Banshee, `U4` = Destroyer
- `N2` = Huntress, `N3` = Druid of Talon, `N4` = Dryad

### bnxplevel.conf - Warcraft 3 Experience Table

Defines XP requirements and level progression for W3 anonymous matchmaking.

**Format**

```
# <level>  <starting_XP>  <XP_needed>  <loss_factor>  <min_games>
```

**Example**

```
1  0      100   0.10   0
2  100    100   0.11   0
3  200    200   0.11   0
10 2500   500   1.00   3
20 7500   500   1.00   6
50 22500  500   1.00   15
```

**Columns**:
- **level** - Player level
- **starting_XP** - Cumulative XP to reach this level
- **XP_needed** - XP to advance to next level
- **loss_factor** - XP penalty multiplier on loss
- **min_games** - Minimum games required for level

**XP Calculation**:
- **Win**: Base XP + performance bonus
- **Loss**: -XP (XP_needed × loss_factor)
- **Level Up**: When cumulative XP ≥ next level's starting_XP

## Content & Display

### ad.json - Advertisement Banners

Defines clickable ad banners displayed in game clients.

**Format**

```json
{
    "ads": [
        {
            "filename": "ad000001.png",
            "url": "http://pvpgn.pro",
            "client": "W3XP",
            "lang": "NULL"
        },
        {
            "filename": "ad000002.smk",
            "url": "http://example.com",
            "client": "NULL",
            "lang": "enUS"
        }
    ]
}
```

**Properties**:
- **filename** - Ad file in `filedir` (png, smk, mng)
- **url** - Click destination URL
- **client** - Client tag restriction (`NULL` = all)
- **lang** - Language restriction (`NULL` = all)

**Supported Formats**:
- `.png` - PNG image (W3XP only)
- `.smk` - Smacker video (legacy clients)
- `.mng` - MNG animation (W3XP)

**File Locations**: Place ad files in `${LOCALSTATEDIR}/files/`

### autoupdate.conf - Client Auto-Update

Maps client versions to update MPQ files.

**Format**

```
# archtag  clienttag  versiontag  -----update_file-----
```

**Examples**

```ini
# Warcraft 3: Reign of Chaos update
IX86   WAR3   WAR3_124B   WAR3_IX86_123a_124b.mpq

# Starcraft: Brood War update
IX86   SEXP   SEXP_118   BWAI_IX86_117_118.mpq

# Diablo II update
IX86   D2DV   D2DV_110   D2DV_IX86_109_110.mpq
```

**Warcraft 3 Localized Updates**

War3/W3XP automatically appends language code:

```ini
# Entry:
IX86  W3XP  W3XP_127B  W3XP_IX86_127a_127b.mpq

# Actual files needed:
W3XP_IX86_127a_127b_enUS.mpq  (English)
W3XP_IX86_127a_127b_deDE.mpq  (German)
W3XP_IX86_127a_127b_frFR.mpq  (French)
W3XP_IX86_127a_127b_ruRU.mpq  (Russian)
```

**Language Codes**: enUS, deDE, esES, frFR, itIT, jaJA, koKR, plPL, ruRU, zhCN, zhTW, csCZ

**Westwood Online Updates**

WOL uses FTP protocol with SKU-specific files:

```ini
# Entry:
IX86  TSUN  65536  131075_65536.rtp  tibsun

# Actual files:
131075_65536_4608.rtp  (SKU 4608)
131075_65536_4610.rtp  (SKU 4610)
```

### bnalias.conf - Command Aliases

Creates custom commands and shortcuts.

**Format**

```
@
aliasname [shortcut1 [shortcut2 ...]]
[!]command or text
[!]%Iinfo message
[!]%Eerror message
```

**Argument Matching**:
- `[0]` - No arguments
- `[1]` - Exactly 1 argument
- `[2+]` - 2 or more arguments
- `[*]` - Any number of arguments
- `[+]` - At least 1 argument

**Argument Substitution**:
- `$*` - All arguments
- `$0` - Command name
- `$1, $2, $3...` - Individual arguments
- `${1-}` - Arguments 1 to end
- `${2-5}` - Arguments 2 through 5

**Examples**

```
@
//doubt //dt //d
[0]/me looks with doubt
[1+]/me looks at ${1-} with doubt

@
//statsme //sm
[0]/stats %I

@
//announce //ann
[*]/announce $*

@
//askban //ab
[2+]/w $1 Please ban ${2-}.
```

**Format Codes**:
- `%a` - Account count
- `%u` - Users online
- `%g` - Games online
- `%c` - Channels online
- `%h` - Server hostname
- `%I` - Current username
- `%v` - Server version

### icons.conf - Custom User Icons

Assigns custom icons to specific users.

**Format**

```
# username  icon_code
```

**Examples**

```
admin       WCAD  # Admin icon
moderator   WCMO  # Moderator icon
developer   W3C1  # Custom icon 1
```

**Icon Codes** vary by game client. Check client documentation for available codes.

## Localization System

### i18n/ Directory Structure

PvPGN supports 16 languages with automatic content delivery based on client locale.

**Supported Languages**

| Code | Language | Code | Language |
|------|----------|------|----------|
| enUS | English (US) | deDE | German |
| esES | Spanish | frFR | French |
| ruRU | Russian | itIT | Italian |
| koKR | Korean | jaJA | Japanese |
| zhCN | Chinese (Simplified) | zhTW | Chinese (Traditional) |
| plPL | Polish | ptBR | Portuguese (Brazilian) |
| nlNL | Dutch | csCZ | Czech |
| svSE | Swedish | bgBG | Bulgarian |

### Core Localized Files

**common.xml** - Shared localized strings

XML format with string IDs and translations:

```xml
<strings>
    <string id="welcome_message">
        <enUS>Welcome to PvPGN!</enUS>
        <deDE>Willkommen bei PvPGN!</deDE>
        <frFR>Bienvenue sur PvPGN!</frFR>
    </string>
</strings>
```

**bnmotd.txt** - Message of the Day

Displayed when users log in:

```
%--------------------------------------------------------------------------------
% Welcome to PvPGN Server!
%--------------------------------------------------------------------------------
% Server: %h
% Uptime: %v
% Users Online: %u
%--------------------------------------------------------------------------------
```

**w3motd.txt** - Warcraft 3 MOTD

Special MOTD for Warcraft 3 clients:

```
Warcraft 3 Server
Type /help for commands
Visit: http://pvpgn.pro
```

**bnhelp.conf** - Command Help

Multi-command help system:

```
%whois whereis where
--------------------------------------------------------
/whois <player> (aliases: /where /whereis)
    Displays where a <player> is on the server.
    
    Example: /whois nomad

%whisper w m msg
--------------------------------------------------------
/whisper <player> <message> (aliases: /w /m /msg)
    Sends a private <message> to <player>
```

**Format**:
- `%command1 command2 command3` - Commands that share this help entry
- Following lines until next `%` - Help text

**news.txt** - News Feed

Server announcements displayed periodically:

```
[2025-01-15] New ladder season started!
[2025-01-10] Server maintenance completed
[2025-01-05] Happy New Year from PvPGN Team!
```

**termsofservice.txt** - Terms of Service

Legal agreement shown at account creation:

```
TERMS OF SERVICE

By creating an account, you agree to:
1. Not use cheats or hacks
2. Be respectful to other players
3. Follow server rules
...
```

### Localization Priority

1. **Language-specific file**: `i18n/deDE/bnmotd.txt`
2. **Default file**: `i18n/bnmotd.txt`

**Selection Method**:
- `localize_by_country = true` - Use client's country/language setting
- `localize_by_country = false` - Use game client's built-in language

### Creating Translations

```bash
# 1. Create language directory
mkdir i18n/deDE

# 2. Copy English template
cp i18n/bnmotd.txt i18n/deDE/bnmotd.txt

# 3. Translate content
nano i18n/deDE/bnmotd.txt

# 4. Repeat for all files
cp i18n/news.txt i18n/deDE/news.txt
cp i18n/bnhelp.conf i18n/deDE/bnhelp.conf
```

## Installation & Deployment

### Build Process

Configuration files are processed during CMake build:

```bash
cd pvpgn-server
mkdir build && cd build

# Configure with custom paths
cmake -DCMAKE_INSTALL_PREFIX=/usr/local \
      -DSYSCONFDIR=/etc/pvpgn \
      -DLOCALSTATEDIR=/var/pvpgn \
      ..

# Build
make

# Install (requires root)
sudo make install
```

**Default Paths**:
- **SYSCONFDIR**: `/usr/local/etc` (or `/etc/pvpgn`)
- **LOCALSTATEDIR**: `/usr/local/var` (or `/var/pvpgn`)

### Post-Installation Setup

**1. Review Core Configuration**

```bash
# Edit main server config
sudo nano /etc/pvpgn/bnetd.conf

# Key settings to review:
# - storage_path (file vs SQL)
# - servaddrs (listening IPs)
# - allowed_clients
# - loglevels
```

**2. Configure Diablo II (if used)**

```bash
# Set realm name
sudo nano /etc/pvpgn/d2cs.conf
# realmname = D2CS

# Configure game servers
# gameservlist = <d2gs-IP>
# bnetdaddr = <bnetd-IP>:6112

# Set storage paths
sudo nano /etc/pvpgn/d2dbs.conf
# gameservlist = <d2gs-IP>
```

**3. Set Up Network Translation**

```bash
sudo nano /etc/pvpgn/address_translation.conf

# Example for server at 192.168.1.100 with external IP 1.2.3.4
0.0.0.0:6112      1.2.3.4:6112      192.168.0.0/24    ANY
0.0.0.0:6200      1.2.3.4:6200      192.168.0.0/24    ANY
```

**4. Configure Firewall**

```bash
# Open required ports
sudo firewall-cmd --permanent --add-port=6112/tcp  # bnetd
sudo firewall-cmd --permanent --add-port=6113/tcp  # d2cs
sudo firewall-cmd --permanent --add-port=6114/tcp  # d2dbs
sudo firewall-cmd --permanent --add-port=6200/tcp  # w3route
sudo firewall-cmd --reload

# Or iptables:
sudo iptables -A INPUT -p tcp --dport 6112 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 6113 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 6114 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 6200 -j ACCEPT
```

**5. Create Data Directories**

```bash
# Create necessary directories
sudo mkdir -p /var/pvpgn/{users,charsave,charinfo,ladders,files,lua,reports,chanlogs,bak}
sudo chown -R pvpgn:pvpgn /var/pvpgn
sudo chmod 750 /var/pvpgn
```

**6. Initialize Database (SQL mode)**

```bash
# MySQL example
mysql -u root -p

CREATE DATABASE pvpgn;
CREATE USER 'pvpgn'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON pvpgn.* TO 'pvpgn'@'localhost';
FLUSH PRIVILEGES;

# Update bnetd.conf
storage_path = "sql:mode=mysql;host=localhost;name=pvpgn;user=pvpgn;pass=password;prefix=pvpgn_"
```

### Realm Configuration (Diablo II)

**1. Define Realm** (`realm.conf`)

```
"D2CS"  "PvPGN Closed Realm"  192.168.1.100:6113
```

**2. Match in d2cs.conf**

```
realmname = D2CS
```

**3. Verify Game Server**

```bash
# d2cs.conf
gameservlist = 192.168.1.100

# d2dbs.conf  
gameservlist = 192.168.1.100
```

## Configuration Examples

### Example 1: Basic Public Server

**bnetd.conf**

```ini
storage_path = "file:mode=plain;dir=/var/pvpgn/users;clan=/var/pvpgn/clans;team=/var/pvpgn/teams;default=/etc/pvpgn/bnetd_default_user.plain"
servaddrs = 0.0.0.0:6112
allowed_clients = all
new_accounts = true
loglevels = fatal,error,warn,info
```

### Example 2: Private LAN Server

**bnetd.conf**

```ini
servaddrs = 192.168.1.100:6112
allowed_clients = war3,w3xp,star,sexp
new_accounts = false  # Pre-created accounts only
max_accounts = 100
allow_bad_version = true  # Allow LAN versions
```

### Example 3: SQL-Based Server

**bnetd.conf**

```ini
storage_path = "sql:mode=mysql;host=localhost;name=pvpgn;user=pvpgn;pass=SecurePass123;prefix=pvpgn_"
servaddrs = 0.0.0.0:6112
usersync = 60  # Save frequently with SQL
userflush_connected = false  # Keep in memory
```

### Example 4: NAT/Firewall Server

**bnetd.conf**

```ini
servaddrs = 0.0.0.0:6112
w3route_addr = 0.0.0.0:6200
```

**address_translation.conf**

```
0.0.0.0:6112  1.2.3.4:6112  192.168.0.0/24  ANY
0.0.0.0:6200  1.2.3.4:6200  192.168.0.0/24  ANY
```

### Example 5: Multi-Language Server

**bnetd.conf**

```ini
localize_by_country = true
i18ndir = /etc/pvpgn/i18n
```

**Create translations**:

```bash
mkdir -p /etc/pvpgn/i18n/{enUS,deDE,frFR,esES,ruRU}
# Copy and translate bnmotd.txt, news.txt, bnhelp.conf to each
```

## Troubleshooting

### Configuration Loading Issues

**Problem**: Server won't start, "config file not found"

```bash
# Check file exists
ls -la /etc/pvpgn/bnetd.conf

# Verify permissions
sudo chmod 644 /etc/pvpgn/*.conf

# Check syntax (look for tabs, special characters)
cat -A /etc/pvpgn/bnetd.conf | head -20
```

**Problem**: "Invalid storage path"

```bash
# File mode: Ensure directories exist
sudo mkdir -p /var/pvpgn/users /var/pvpgn/clans
sudo chown pvpgn:pvpgn /var/pvpgn/users

# SQL mode: Test database connection
mysql -h localhost -u pvpgn -p pvpgn
```

### Network Configuration Issues

**Problem**: Can't connect from outside LAN

```bash
# Verify listening address
netstat -tlnp | grep 6112

# Should show: 0.0.0.0:6112 (not 127.0.0.1:6112)

# Check address_translation.conf
grep -v '^#' /etc/pvpgn/address_translation.conf

# Test firewall
sudo iptables -L -n | grep 6112
```

**Problem**: D2 clients can't see realm

```bash
# Verify realm defined
grep -v '^#' /etc/pvpgn/realm.conf

# Match d2cs realmname
grep realmname /etc/pvpgn/d2cs.conf

# Check bnetd → d2cs connection
netstat -tn | grep :6113
```

### Version Check Issues

**Problem**: "Version check failed" for valid clients

```bash
# Enable bypass
nano /etc/pvpgn/bnetd.conf
allow_bad_version = true
allow_unknown_version = true

# Or add version to versioncheck.json
nano /etc/pvpgn/versioncheck.json
```

**Problem**: Autoupdate not working

```bash
# Ensure MPQ file exists
ls -la /var/pvpgn/files/*.mpq

# Check autoupdate.conf entry
grep "WAR3_127B" /etc/pvpgn/autoupdate.conf

# Match versionTag in versioncheck.json
grep "WAR3_127B" /etc/pvpgn/versioncheck.json
```

### Permission Issues

**Problem**: Commands don't work

```bash
# Check command_groups.conf
grep "^1 " /etc/pvpgn/command_groups.conf

# Verify user has group 1
# File mode:
grep "command_groups" /var/pvpgn/users/<username>
# Should contain: "BNET\\auth\\command_groups"="1"

# SQL mode:
SELECT username, auth_command_groups FROM pvpgn_BNET WHERE username='<name>';
```

**Problem**: Default users can't use commands

```bash
# Check default account template
grep "command_groups" /etc/pvpgn/bnetd_default_user.plain
# Must have: "BNET\\auth\\command_groups"="1"

# SQL: Check default UID (usually 0)
SELECT auth_command_groups FROM pvpgn_BNET WHERE uid='0';
```

### Localization Issues

**Problem**: Wrong language displayed

```bash
# Check localize_by_country setting
grep localize_by_country /etc/pvpgn/bnetd.conf

# Verify language files exist
ls -la /etc/pvpgn/i18n/deDE/

# Test with specific file
cat /etc/pvpgn/i18n/deDE/bnmotd.txt
```

## Security Best Practices

### 1. File Permissions

```bash
# Config files: readable by server only
sudo chmod 640 /etc/pvpgn/*.conf
sudo chown root:pvpgn /etc/pvpgn/*.conf

# Data directories: writable by server
sudo chmod 750 /var/pvpgn
sudo chown pvpgn:pvpgn /var/pvpgn
```

### 2. Database Security

```sql
-- Dedicated user with limited privileges
CREATE USER 'pvpgn'@'localhost' IDENTIFIED BY 'StrongPassword123!';
GRANT SELECT, INSERT, UPDATE, DELETE ON pvpgn.* TO 'pvpgn'@'localhost';

-- No CREATE, DROP, or admin privileges
```

### 3. Network Security

```ini
# Bind to specific interface (not 0.0.0.0) if possible
servaddrs = 192.168.1.100:6112

# Use telnet only on localhost
telnet_addrs = 127.0.0.1:8888

# Enable IP bans
ipbanfile = /etc/pvpgn/bnban.conf
```

### 4. Account Security

```ini
# Disable anonymous logins
new_accounts = false

# Limit account creation
max_accounts = 1000

# Require strong passwords (in bnetd.conf)
# (enforced by client, server stores hash only)
```

### 5. Version Enforcement

```ini
# Strict version checking
allow_bad_version = false
allow_unknown_version = false

# Maintain updated versioncheck.json
# Add all legitimate client versions
```

## Related Documentation

- **bnetd Server**: `src/bnetd/README.md`
- **d2dbs Server**: `src/d2dbs/README.md`
- **Lua Scripting**: `lua/README.md`
- **Scripts & Tools**: `scripts/README.md`
- **PvPGN Documentation**: https://pvpgn.pro/pvpgn_wiki

## Maintenance & Updates

### Regular Tasks

**Weekly**:
- Review IP ban list (`bnban.conf`)
- Check error logs for configuration issues
- Monitor account creation rate

**Monthly**:
- Update `versioncheck.json` for new game patches
- Add corresponding MPQ files to `autoupdate.conf`
- Backup configuration files
- Review and update localized content

**As Needed**:
- Adjust channel configuration based on usage
- Update MOTD/news for events
- Modify XP tables for balance
- Add tournament schedules

### Configuration Backup

```bash
# Backup all configuration
sudo tar -czf pvpgn-conf-$(date +%F).tar.gz /etc/pvpgn/

# Backup SQL schema
mysqldump -u pvpgn -p pvpgn > pvpgn-db-$(date +%F).sql

# Automated backup (crontab)
0 2 * * * tar -czf /backup/pvpgn-conf-$(date +\%F).tar.gz /etc/pvpgn/
```

## Contributing

When modifying configuration files:

1. **Test changes** on development server first
2. **Document** any non-obvious settings
3. **Validate syntax** (especially JSON files)
4. **Backup** before making changes
5. **Version control** configuration in git

## License

Configuration files are part of the PvPGN project and licensed under the GPL.

Copyright (C) 1999-2025 PvPGN Project and contributors.

---

**Last Updated**: November 24, 2025  
**PvPGN Version**: Latest development branch  
**Configuration Format Version**: 2.x
