# PvPGN Server - Player vs Player Gaming Network

PvPGN is a free and open source cross-platform server software that supports Battle.net and and Westwood Online game clients. PvPGN-PRO is a fork of the official PvPGN project, whose development stopped in 2011, and aims to provide continued maintenance and additional features for PvPGN.

[![License (GPL version 2)](https://img.shields.io/badge/license-GNU%20GPL%20version%202-blue.svg?style=flat-square)](http://opensource.org/licenses/GPL-2.0)
![Language (C++)](https://img.shields.io/badge/powered_by-C++-brightgreen.svg?style=flat-square)
[![Language (Lua)](https://img.shields.io/badge/powered_by-Lua-red.svg?style=flat-square)](https://lua.org)
[![Github Releases (by Release)](https://img.shields.io/github/downloads/pvpgn/pvpgn-server/1.99.7.2.1/total.svg?maxAge=2592000)]()

[![Compiler (Microsoft Visual C++)](https://img.shields.io/badge/compiled_with-Microsoft%20Visual%20C++-yellow.svg?style=flat-square)](https://msdn.microsoft.com/en-us/vstudio/hh386302.aspx)
[![Compiler (LLVM/Clang)](https://img.shields.io/badge/compiled_with-LLVM/Clang-lightgrey.svg?style=flat-square)](http://clang.llvm.org/)
[![Compiler (GCC)](https://img.shields.io/badge/compiled_with-GCC-yellowgreen.svg?style=flat-square)](https://gcc.gnu.org/)

[![Build Status](https://travis-ci.org/pvpgn/pvpgn-server.svg?branch=master)](https://travis-ci.org/pvpgn/pvpgn-server)
[![Build status](https://ci.appveyor.com/api/projects/status/dqoj9lkvhfwthmn6)](https://ci.appveyor.com/project/HarpyWar/pvpgn)

---

## Overview

PvPGN (Player vs Player Gaming Network) is a free, open-source, cross-platform server software that emulates Battle.net and Westwood Online gaming services. It allows players to continue enjoying classic multiplayer games even after official server support has ended.

**PvPGN-PRO** is a community-maintained fork of the original PvPGN project (which ceased development in 2011), providing:
- ✅ Continued maintenance and bug fixes
- ✅ Support for modern operating systems and compilers
- ✅ New features and enhancements
- ✅ Active community development
- ✅ Compatibility with 30+ classic games

### Key Features

- **Multi-Game Support**: Host servers for Blizzard Entertainment and Westwood Studios games
- **Cross-Platform**: Runs on Windows, Linux, macOS, and BSD systems
- **Flexible Storage**: File-based, MySQL, PostgreSQL, SQLite3, or ODBC database storage
- **Lua Scripting**: Extend functionality with custom Lua scripts
- **Localization**: Support for 16+ languages
- **IRC/Telnet**: Remote administration capabilities
- **Clan/Team System**: Built-in clan and team management
- **Ladder/Tournament**: Automated ladder rankings and tournament systems
- **Anti-Cheat**: Version checking and memory scanning capabilities

[Deleaker](http://www.deleaker.com/) helps us find memory leaks.

---

---

## Architecture

PvPGN consists of three main server components:

### Server Components

```
┌─────────────────────────────────────────────────────────────────┐
│                        PvPGN Server Suite                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌────────────────┐  ┌─────────────┐  ┌──────────────┐         │
│  │     bnetd      │  │    d2cs     │  │    d2dbs     │         │
│  │  Main Server   │  │ D2 Char Srv │  │  D2 DB Srv   │         │
│  │                │  │             │  │              │         │
│  │ • Battle.net   │  │ • Character │  │ • Character  │         │
│  │ • WOL          │  │   Selection │  │   Storage    │         │
│  │ • Chat/Lobby   │  │ • Game Mgmt │  │ • Ladders    │         │
│  │ • Accounts     │  │ • Routing   │  │ • Backups    │         │
│  │ • Channels     │  │             │  │              │         │
│  │ • Games        │  │             │  │              │         │
│  │ • Ladders      │  │             │  │              │         │
│  └────────────────┘  └─────────────┘  └──────────────┘         │
│         │                   │                  │                │
│         └───────────────────┴──────────────────┘                │
│                             │                                   │
│                   ┌─────────┴─────────┐                        │
│                   │   Storage Layer   │                        │
│                   │ • File (Plain/CDB)│                        │
│                   │ • MySQL/PostgreSQL│                        │
│                   │ • SQLite3/ODBC    │                        │
│                   └───────────────────┘                        │
└─────────────────────────────────────────────────────────────────┘
```

**bnetd** - The main Battle.net/WOL server handling:
- Client authentication and account management
- Chat channels and lobbies
- Game creation and matchmaking
- Clan and team systems
- IRC and Telnet interfaces
- Lua scripting engine

**d2cs** (Diablo II Character Server) - Manages Diablo II gameplay:
- Character selection and validation
- Game server routing
- Realm coordination
- Character conversion (Classic ↔ Expansion)

**d2dbs** (Diablo II Database Server) - Stores Diablo II data:
- Character save files
- Character information
- Ladder rankings
- Automated backups

### Directory Structure

```
pvpgn-server/
├── src/                    # Source code
│   ├── bnetd/             # Main server
│   ├── d2cs/              # Diablo II Character Server
│   ├── d2dbs/             # Diablo II Database Server
│   ├── client/            # Client utilities (bnchat, bnbot, etc.)
│   ├── bnproxy/           # Battle.net proxy
│   ├── bntrackd/          # Server tracker
│   ├── common/            # Shared utilities
│   ├── compat/            # Compatibility layer
│   └── win32/             # Windows-specific code
├── conf/                   # Configuration files
├── lua/                    # Lua scripting system
├── files/                  # Game files (MPQs, icons, ads)
├── scripts/                # Administrative scripts
├── docs/                   # Documentation
└── lib/                    # Third-party libraries
```

---

## Tracking
By default, tracking is enabled and is only used for the purpose of sending informational data (e.g. server description, homepage, uptime, amount of users) to tracking servers. To disable tracking, set ````track = 0```` in ````conf/bnetd.conf````.

---

---

## Supported Games & Clients

### Blizzard Entertainment Games

#### Warcraft Series
- **WarCraft 2: Battle.net Edition**: 2.02a, 2.02b
- **WarCraft 3: Reign of Chaos**\*: 1.13a, 1.13b, 1.14a, 1.14b, 1.15a, 1.16a, 1.17a, 1.18a, 1.19a, 1.19b, 1.20a, 1.20b, 1.20c, 1.20d, 1.20e, 1.21a, 1.21b, 1.22a, 1.23a, 1.24a, 1.24b, 1.24c, 1.24d, 1.24e, 1.25b, 1.26a, 1.27a, 1.27b, 1.28, 1.28.1, 1.28.2, 1.28.4, 1.28.5
- **WarCraft 3: The Frozen Throne**\*: 1.13a, 1.13b, 1.14a, 1.14b, 1.15a, 1.16a, 1.17a, 1.18a, 1.19a, 1.19b, 1.20a, 1.20b, 1.20c, 1.20d, 1.20e, 1.21a, 1.21b, 1.22a, 1.23a, 1.24a, 1.24b, 1.24c, 1.24d, 1.24e, 1.25b, 1.26a, 1.27a, 1.27b, 1.28, 1.28.1, 1.28.2, 1.28.4, 1.28.5
- **StarCraft**: 1.08, 1.08b, 1.09, 1.09b, 1.10, 1.11, 1.11b, 1.12, 1.12b, 1.13, 1.13b, 1.13c, 1.13d, 1.13e, 1.13f, 1.14, 1.15, 1.15.1, 1.15.2, 1.15.3, 1.16, 1.16.1, 1.17.0, 1.18.0
- **StarCraft: Brood War**: 1.08, 1.08b, 1.09, 1.09b, 1.10, 1.11, 1.11b, 1.12, 1.12b, 1.13, 1.13b, 1.13c, 1.13d, 1.13e, 1.13f, 1.14, 1.15, 1.15.1, 1.15.2, 1.15.3, 1.16, 1.16.1, 1.17.0, 1.18.0
- **Diablo**: 1.09, 1.09b
- **Diablo 2**: 1.10, 1.11, 1.11b, 1.12a, 1.13c, 1.14a, 1.14b, 1.14c, 1.14d
- **Diablo 2: Lord of Destruction**: 1.10, 1.11, 1.11b, 1.12a, 1.13c, 1.14a, 1.14b, 1.14c, 1.14d
- **Westwood Chat Client**: 4.221
- **Command & Conquer**: Win95 1.04a (using Westwood Chat)
- **Command & Conquer: Red Alert**: Win95 2.00 (using Westwood Chat), Win95 3.03
- **Command & Conquer: Red Alert 2**: 1.006
- **Command & Conquer: Tiberian Sun**: 2.03 ST-10
- **Command & Conquer: Tiberian Sun Firestorm**: 2.03 ST-10
- **Command & Conquer: Yuri's Revenge**: 1.001
- **Command & Conquer: Renegade**: 1.037
- **Nox**: 1.02b
- **Nox Quest**: 1.02b
- **Dune 2000**: 1.06
- **Emperor: Battle for Dune**: 1.09

### Important Notes

\* **WarCraft 3**: Clients are unable to connect to PvPGN servers without a client-side modification through tools such as [W3L](https://github.com/w3lh/w3l) to disable server signature verification.

\* **StarCraft 1.18+**: Clients beginning with patch 1.18 will not be supported by PvPGN-PRO due to protocol changes. A 1.18.0 versioncheck entry is included for compatibility with bot software.

---

---

## Getting Started

### Quick Start

1. **Download**: Get the latest release or clone the repository
2. **Build**: Compile using CMake (see [Building](#building) section)
3. **Configure**: Edit configuration files in `conf/` directory
4. **Run**: Start the server with appropriate permissions
5. **Connect**: Point game clients to your server IP

### Requirements

**Compiler Support**:
- **Visual Studio**: 2015 or later (VC++ 14+)
- **GCC**: 5.1 or later
- **Clang**: 3.7 or later

**Build Tools**:
- **CMake**: 3.1.0 or later
- **C++11** compliant compiler

**Optional Dependencies**:
- **Lua**: 5.1+ (for scripting support)
- **MySQL**: 5.x+ (for MySQL storage)
- **PostgreSQL**: 9.x+ (for PostgreSQL storage)
- **SQLite3**: 3.x+ (for SQLite storage)

### Platform Support

PvPGN has been tested and confirmed working on:

| Platform | Architecture | Status | Compiler |
|----------|-------------|---------|----------|
| Windows 10/11 | x86_64 | ✅ Working | MSVC 14+, Clang |
| Ubuntu 14.04+ | x86_64 | ✅ Working | GCC 5.1+, Clang |
| Debian 8+ | x86_64 | ✅ Working | GCC, Clang |
| CentOS 7+ | x86_64 | ✅ Working | GCC 5.3+ |
| Fedora 25+ | x86_64 | ✅ Working | GCC 6.2+ |
| macOS 10.11+ | x86_64 | ✅ Working | Apple Clang |
| FreeBSD 11+ | x86_64 | ✅ Working | Clang |
| Arch Linux | x86_64 | ✅ Working | GCC, Clang |

See [docs/ports.md](https://github.com/pvpgn/pvpgn-server/blob/master/docs/ports.md) for detailed platform compatibility.

---

## Support
[Create an issue](https://github.com/pvpgn/pvpgn-server/issues) if you have any questions, suggestions, or anything else to say about PvPGN-PRO. Please note that D2GS is not part of the PvPGN project and is therefore unsupported here.
Set `loglevels = fatal,error,warn,info,debug,trace` in `bnetd.conf` before obtaining logs and posting them.

## Development
Submit pull requests to contribute to this project. Utilize C++11 features and adhere to the [C++ Core Guidelines](https://github.com/isocpp/CppCoreGuidelines/blob/master/CppCoreGuidelines.md) whenever possible.

---

## Building

The CMake files have been configured to reject compilers older than Visual Studio 2015 and GCC 5.1 to ensure C++11 compliance.

### Build Options

```bash
# Core components (enabled by default)
-DWITH_BNETD=ON          # Build main bnetd server
-DWITH_D2CS=ON           # Build Diablo II Character Server
-DWITH_D2DBS=ON          # Build Diablo II Database Server

# Optional features
-DWITH_LUA=ON            # Enable Lua scripting support
-DWITH_WIN32_GUI=ON      # Enable Windows GUI (Windows only)

# Storage backends
-DWITH_MYSQL=ON          # MySQL database support
-DWITH_SQLITE3=ON        # SQLite3 database support
-DWITH_PGSQL=ON          # PostgreSQL database support
-DWITH_ODBC=ON           # ODBC database support

# Installation paths
-DCMAKE_INSTALL_PREFIX=/usr/local/pvpgn
```

### Windows

#### Option 1: Magic Builder (Recommended)
Use [Magic Builder](https://github.com/pvpgn/pvpgn-magic-builder) for automated Windows builds.

#### Option 2: Visual Studio
```bash
# Generate Visual Studio solution
cmake -G "Visual Studio 14 2015" -H./ -B./build

# Or for Visual Studio 2019/2022
cmake -G "Visual Studio 16 2019" -H./ -B./build
cmake -G "Visual Studio 17 2022" -H./ -B./build

# This generates .sln in build/ directory
# Open the solution in Visual Studio and build
```

#### Option 3: Command Line
```bash
# Navigate to build directory
cd build

# Build Release configuration
cmake --build . --config Release

# Build Debug configuration
cmake --build . --config Debug
```

### Linux

#### Ubuntu 16.04, 18.04, 20.04, 22.04
```bash
sudo apt-get -y install build-essential git cmake zlib1g-dev
git clone https://github.com/pvpgn/pvpgn-server.git
cmake -D CMAKE_INSTALL_PREFIX=/usr/local/pvpgn -D WITH_MYSQL=true -D WITH_LUA=true ../
make
make install
```

#### Ubuntu 16.04+ with All Features
```bash
sudo apt-get -y install build-essential git cmake zlib1g-dev
# Lua support
sudo apt-get install liblua5.1-0-dev
# MySQL support
sudo apt-get install mysql-server mysql-client libmysqlclient-dev
# PostgreSQL support
sudo apt-get install postgresql libpq-dev
# SQLite3 support
sudo apt-get install libsqlite3-dev

git clone https://github.com/pvpgn/pvpgn-server.git
cd pvpgn-server
cmake -G "Unix Makefiles" \
  -DCMAKE_INSTALL_PREFIX=/usr/local/pvpgn \
  -DWITH_MYSQL=ON \
  -DWITH_SQLITE3=ON \
  -DWITH_PGSQL=ON \
  -DWITH_LUA=ON \
  -H./ -B./build
cd build && make
sudo make install
```

#### Ubuntu 14.04 (Legacy)
```bash
sudo apt-get -y install build-essential zlib1g-dev git
sudo add-apt-repository -y ppa:ubuntu-toolchain-r/test
sudo apt-get -y update
sudo apt-get -y install gcc-5 g++-5
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-5 60 --slave /usr/bin/g++ g++ /usr/bin/g++-5
sudo add-apt-repository -y ppa:george-edison55/cmake-3.x
sudo apt-get update
sudo apt-get -y install cmake
git clone https://github.com/pvpgn/pvpgn-server.git
cd pvpgn-server && cmake -G "Unix Makefiles" -H./ -B./build
cd build && make
```

#### Debian 8+ with Clang
```bash
sudo apt-get -y install build-essential zlib1g-dev clang libc++-dev git
wget https://cmake.org/files/v3.7/cmake-3.7.1-Linux-x86_64.tar.gz
tar xvfz cmake-3.7.1-Linux-x86_64.tar.gz
git clone https://github.com/pvpgn/pvpgn-server.git
cd pvpgn-server && CC=/usr/bin/clang CXX=/usr/bin/clang++ ../cmake-3.7.1-Linux-x86_64/bin/cmake -G "Unix Makefiles" -H./ -B./build
cd build && make
```

#### CentOS 7+
```bash
sudo yum -y install epel-release centos-release-scl
sudo yum -y install git zlib-devel cmake3 devtoolset-4-gcc*
sudo ln -s /usr/bin/cmake3 /usr/bin/cmake
git clone https://github.com/pvpgn/pvpgn-server.git
cd pvpgn-server
CC=/opt/rh/devtoolset-4/root/usr/bin/gcc CXX=/opt/rh/devtoolset-4/root/usr/bin/g++ cmake -G "Unix Makefiles" -H./ -B./build
cd build && make
```

#### Fedora 25+
```bash
sudo dnf -y install gcc-c++ gcc make zlib-devel cmake git
git clone https://github.com/pvpgn/pvpgn-server.git
cd pvpgn-server
cmake -G "Unix Makefiles" -H./ -B./build
cd build && make
```

### macOS

```bash
# Install Xcode Command Line Tools
xcode-select --install

# Install CMake (via Homebrew)
brew install cmake

# Clone and build
git clone https://github.com/pvpgn/pvpgn-server.git
cd pvpgn-server
cmake -G "Unix Makefiles" -H./ -B./build
cd build && make
sudo make install
```

### FreeBSD 11+
```bash
sudo pkg install -y git cmake
git clone https://github.com/pvpgn/pvpgn-server.git
cd pvpgn-server
cmake -G "Unix Makefiles" -H./ -B./build
cd build && make
sudo make install
```

### Additional Resources
Full installation instructions: [Русский](http://harpywar.com/?a=articles&b=2&c=1&d=74) | [English](http://harpywar.com/?a=articles&b=2&c=1&d=74&lang=en)

---

---

## Configuration

After building, you need to configure PvPGN before running it.

### Basic Configuration

1. **Navigate to configuration directory**:
   ```bash
   cd /usr/local/etc/pvpgn  # Linux/macOS
   # or
   cd C:\Program Files\PvPGN\etc  # Windows
   ```

2. **Edit main configuration** (`bnetd.conf`):
   - Set `storage_path` for account storage (file, MySQL, PostgreSQL, SQLite3, ODBC)
   - Configure `servaddrs` for network binding
   - Set `loglevels` for debugging
   - Review other options as needed

3. **Configure storage backend**:

   **File-based storage** (default):
   ```ini
   storage_path = "file:mode=plain;dir=/var/pvpgn/users;clan=/var/pvpgn/clans;default=/etc/pvpgn/bnetd_default_user.plain"
   ```

   **MySQL storage**:
   ```ini
   storage_path = "sql:mode=mysql;host=127.0.0.1;name=pvpgn;user=pvpgn;pass=password;default=0;prefix=pvpgn_"
   ```

   **PostgreSQL storage**:
   ```ini
   storage_path = "sql:mode=pgsql;host=127.0.0.1;name=pvpgn;user=pvpgn;pass=password;default=0;prefix=pvpgn_"
   ```

   **SQLite3 storage**:
   ```ini
   storage_path = "sql:mode=sqlite3;name=/var/pvpgn/users.db;default=0;prefix=pvpgn_"
   ```

4. **Create required directories** (file-based storage):
   ```bash
   sudo mkdir -p /var/pvpgn/{users,files,reports,chanlogs,charsave,charinfo,ladders}
   sudo chown pvpgn:pvpgn /var/pvpgn -R
   ```

### Advanced Configuration

See the [conf/](conf/) directory for detailed documentation on:
- **Channel configuration** (`channel.conf`)
- **Command permissions** (`command_groups.conf`)
- **Version checking** (`versioncheck.json`)
- **Realm setup** (`realm.conf`) for Diablo II
- **Address translation** (`address_translation.conf`) for NAT/firewall
- **Localization** (`i18n/` directory) for multi-language support

### Diablo II Setup

For Diablo II support, additional configuration is required:

1. **Configure d2cs** (`d2cs.conf`):
   ```ini
   realmname = D2CS
   gameservlist = <d2gs-server-ip>
   bnetdaddr = <bnetd-server-ip>:6112
   ```

2. **Configure d2dbs** (`d2dbs.conf`):
   ```ini
   gameservlist = <d2gs-server-ip>
   charsavedir = /var/pvpgn/charsave
   ```

3. **Setup realm** (`realm.conf`):
   ```
   "D2CS"  "PvPGN Realm"  <d2cs-ip>:6113
   ```

---

## Network Configuration

### Hosting on LAN or VPS with Private IP

Some VPS providers do not assign your server a direct public IP. If hosting behind NAT or with a private IP, you need to configure route translation in `address_translation.conf`.

#### Why This Matters

When clients search for games, PvPGN pushes the server's route address to game clients. Without correct address translation:
- ❌ Long game search times
- ❌ Unable to join games
- ❌ Connection errors

#### Configuration Example

**Scenario**: Server has internal IP `192.168.1.100` and external IP `1.2.3.4`

Edit `address_translation.conf`:
```
# Format: internal_ip:port  external_ip:port  exclude_network  include_network
0.0.0.0:6112      1.2.3.4:6112      192.168.0.0/24    ANY
0.0.0.0:6200      1.2.3.4:6200      192.168.0.0/24    ANY
```

This tells PvPGN to:
- Send external IP (`1.2.3.4`) to clients outside `192.168.0.0/24`
- Send internal IP (`0.0.0.0`) to clients inside `192.168.0.0/24`

**Note**: If your network interface is directly bound to a public IP, PvPGN can auto-detect it and this step is unnecessary.

### Firewall Configuration

Open required ports:

**bnetd (main server)**:
- TCP 6112 (Battle.net)
- TCP 6113 (d2cs communication)
- TCP 6200 (Warcraft 3 routing)
- TCP 6667 (IRC)
- TCP 8888 (Telnet admin)

**d2cs (Diablo II Character Server)**:
- TCP 6113 (client connections)

**d2dbs (Diablo II Database Server)**:
- TCP 6114 (database operations)

```bash
# Linux (iptables)
sudo iptables -A INPUT -p tcp --dport 6112 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 6113 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 6114 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 6200 -j ACCEPT

# Linux (firewalld)
sudo firewall-cmd --permanent --add-port=6112/tcp
sudo firewall-cmd --permanent --add-port=6113/tcp
sudo firewall-cmd --permanent --add-port=6114/tcp
sudo firewall-cmd --permanent --add-port=6200/tcp
sudo firewall-cmd --reload
```

---

## Running PvPGN

### Linux

```bash
# Start bnetd
sudo /usr/local/sbin/bnetd

# Start d2cs (if using Diablo II)
sudo /usr/local/sbin/d2cs

# Start d2dbs (if using Diablo II)
sudo /usr/local/sbin/d2dbs

# Check if running
ps aux | grep bnetd
```

### Linux (systemd service)

Create `/etc/systemd/system/pvpgn.service`:
```ini
[Unit]
Description=PvPGN Battle.net Server
After=network.target

[Service]
Type=simple
User=pvpgn
ExecStart=/usr/local/sbin/bnetd -f
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Enable and start:
```bash
sudo systemctl daemon-reload
sudo systemctl enable pvpgn
sudo systemctl start pvpgn
sudo systemctl status pvpgn
```

### Windows

#### GUI Mode
1. Navigate to installation directory
2. Double-click `bnetd.exe`
3. GUI window will open showing server status

#### Service Mode
```cmd
# Install service
bnetd.exe -i

# Start service
net start pvpgn

# Stop service
net stop pvpgn

# Uninstall service
bnetd.exe -u
```

#### Console Mode
```cmd
# Run in console
bnetd.exe -f
```

### Testing Connection

```bash
# Using telnet
telnet localhost 6112

# Using bnchat client (included)
./bnchat localhost

# Check logs
tail -f /var/log/pvpgn/bnetd.log
```

---

## Documentation

Comprehensive documentation is available in the [docs/](docs/) and component directories:

### Core Documentation
- **[docs/readme.md](docs/readme.md)** - Documentation index
- **[docs/storage.txt](docs/storage.txt)** - Storage backend configuration
- **[docs/ports.md](docs/ports.md)** - Platform compatibility matrix
- **[docs/versioncheck.md](docs/versioncheck.md)** - Client version validation

### Component Documentation
- **[src/bnetd/](src/bnetd/)** - Main server architecture
- **[src/d2cs/](src/d2cs/)** - Diablo II Character Server
- **[src/d2dbs/](src/d2dbs/)** - Diablo II Database Server
- **[conf/](conf/)** - Configuration file reference
- **[lua/](lua/)** - Lua scripting system
- **[scripts/](scripts/)** - Administrative utilities

### Quick Links
- **Configuration Files**: [conf/README.md](conf/README.md)
- **Lua Scripting**: [lua/README.md](lua/README.md)
- **Admin Scripts**: [scripts/README.md](scripts/README.md)
- **Compilation Guide**: [docs/compile.visualstudio2015.md](docs/compile.visualstudio2015.md)

---

## Troubleshooting

### Server Won't Start

**Check logs**:
```bash
tail -f /var/log/pvpgn/bnetd.log
```

**Common issues**:
- Port already in use (another instance running)
- Permission denied (run as root/administrator or fix permissions)
- Missing configuration files
- Invalid storage path

### Clients Can't Connect

**Verify server is listening**:
```bash
netstat -tlnp | grep 6112
# Should show: 0.0.0.0:6112 or your IP:6112
```

**Check firewall**:
```bash
sudo iptables -L -n | grep 6112
```

**Test locally first**:
```bash
telnet localhost 6112
```

### Database Connection Errors

**MySQL**:
```bash
# Test connection
mysql -h localhost -u pvpgn -p pvpgn

# Check user privileges
SHOW GRANTS FOR 'pvpgn'@'localhost';
```

**PostgreSQL**:
```bash
# Test connection
psql -h localhost -U pvpgn -d pvpgn
```

### Version Check Failures

If clients report version check failures:

1. Set in `bnetd.conf`:
   ```ini
   allow_bad_version = true
   allow_unknown_version = true
   ```

2. Or add proper entries to `versioncheck.json`

### Debug Mode

Enable verbose logging in `bnetd.conf`:
```ini
loglevels = fatal,error,warn,info,debug,trace
```

---

## Contributing

We welcome contributions! Here's how to get started:

### Development Guidelines

1. **C++11 Standard**: Use modern C++11 features
2. **Code Style**: Follow [C++ Core Guidelines](https://github.com/isocpp/CppCoreGuidelines/blob/master/CppCoreGuidelines.md)
3. **Testing**: Test changes on multiple platforms when possible
4. **Documentation**: Update docs for any user-facing changes

### Contribution Process

1. **Fork** the repository
2. **Create** a feature branch (`git checkout -b feature/amazing-feature`)
3. **Commit** your changes (`git commit -m 'Add amazing feature'`)
4. **Push** to the branch (`git push origin feature/amazing-feature`)
5. **Open** a Pull Request

### Areas for Contribution

- 🐛 **Bug fixes** - Help improve stability
- ✨ **New features** - Enhance functionality
- 📝 **Documentation** - Improve guides and examples
- 🌐 **Localization** - Add/update translations
- 🧪 **Testing** - Platform compatibility testing
- 🎨 **Lua scripts** - Community scripts and tools

### Reporting Issues

When reporting bugs, please include:
- PvPGN version
- Operating system and version
- Compiler version
- Configuration files (sanitized)
- Log output with `trace` level enabled
- Steps to reproduce

---

## Community & Support

### Getting Help

- **GitHub Issues**: [Report bugs or request features](https://github.com/pvpgn/pvpgn-server/issues)
- **Documentation**: Check [docs/](docs/) directory
- **Community Forums**: Connect with other PvPGN users

### Important Notes

- **D2GS**: Diablo II Game Server (D2GS) is **not** part of PvPGN and is unsupported here
- **Client Mods**: Some games (War3) require client-side modifications to connect
- **Version Support**: StarCraft 1.18+ uses different protocols and is not supported

---

## License

This program is free software; you can redistribute it and/or
modify it under the terms of the GNU General Public License
as published by the Free Software Foundation; either version 2
of the License, or (at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License
along with this program; if not, write to the Free Software
Foundation, Inc., 51 Franklin Street, Fifth Floor, Boston, MA 02110-1301, USA.

See [LICENSE](LICENSE) for full license text.

---

## Credits

PvPGN is made possible by the contributions of many developers over the years. See [CREDITS](CREDITS) for a comprehensive list of contributors.

**Original BNETD Project**: 1998-2001  
**PvPGN Project**: 2002-2011  
**PvPGN-PRO Fork**: 2011-Present

### Special Thanks

- All contributors who have helped develop and maintain PvPGN
- The gaming community for keeping classic games alive
- [Deleaker](http://www.deleaker.com/) for memory leak detection tools

---

**Maintained by the PvPGN Community**  
**Website**: http://pvpgn.pro  
**Repository**: https://github.com/pvpgn/pvpgn-server