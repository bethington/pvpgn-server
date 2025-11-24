# Scripts - PvPGN Administration and Utility Tools

## Overview

The `scripts` folder contains a comprehensive collection of administration, deployment, migration, and utility tools for managing PvPGN (Player vs Player Gaming Network) servers. These scripts automate common tasks such as server deployment, account management, database migration, ladder generation, localization, and system integration.

Written in various languages (Bash, Perl, Python, PHP, C#, AWK), these tools provide flexible solutions for different operating systems and use cases, supporting both Linux/Unix and Windows environments.

## Directory Structure

```
scripts/
├── System Integration (Init Scripts)
│   ├── bnetd.init           - SysV init script (Red Hat/CentOS)
│   ├── rc.bnetd             - BSD/Slackware init script
│   ├── S98bnetd             - System V runlevel script
│   └── bnetd.logrotate      - Log rotation configuration
│
├── Account Management
│   ├── make_testusers.sh    - Mass test account creation
│   ├── login_testusers.sh   - Automated login testing
│   └── lastlogin.pl         - Last login time extraction
│
├── Database Migration
│   ├── flat2cdb.pl          - Plain text → CDB conversion
│   ├── storage/
│   │   ├── plain2sql.pl     - Plain text → SQL conversion
│   │   └── cdb2sql.pl       - CDB → SQL conversion
│   └── sql/
│       └── repairlevels.pl  - SQL level data repair
│
├── Game-Specific Tools
│   ├── convert_w3.pl        - Warcraft III account converter
│   ├── ladder.py            - Diablo II ladder viewer/generator
│   └── fsgs2bnetd.pl        - FSGS→PvPGN account import
│
├── Server Administration
│   ├── announce.sh          - Automated server announcements
│   ├── bnetmasq.sh          - UDP masquerading for NAT
│   └── process.pl           - Account processing utility
│
├── Localization
│   ├── localize/
│   │   ├── pvpgn_localize_generator.cs    - Extract strings for translation
│   │   ├── pvpgn_localize_validator.cs    - Validate translation files
│   │   ├── *.exe            - Compiled C# utilities
│   │   ├── update.bat       - Windows localization update
│   │   └── validate.bat     - Windows validation runner
│   ├── tos.sh               - Generate ToS files for all languages
│   └── tos.bat              - Windows ToS generation
│
├── Security & Hashing
│   ├── pvpgn_hash.inc.php   - PvPGN password hash (PHP)
│   └── pvpgn_wol_hash.inc.php - Westwood Online hash (PHP)
│
└── Development Tools
    ├── cvs2cl.pl            - CVS to ChangeLog converter
    └── reform.awk           - AWK text reformatting
```

## Categories

### 1. System Integration Scripts

#### bnetd.init
SysV init script for Red Hat, CentOS, Fedora systems using the traditional init system.

**Installation**
```bash
# Copy to init.d
sudo cp bnetd.init /etc/rc.d/init.d/bnetd
sudo chmod +x /etc/rc.d/init.d/bnetd

# Enable on boot
sudo chkconfig bnetd on

# Start service
sudo service bnetd start
```

**Usage**
```bash
service bnetd start       # Start server
service bnetd stop        # Stop server
service bnetd restart     # Restart server
service bnetd reload      # Reload (stop + start)
service bnetd status      # Check status
```

**Features**
- Uses `daemon` and `killproc` from `/etc/rc.d/init.d/functions`
- Creates lock file: `/var/lock/subsys/bnetd`
- Chkconfig integration: runlevels 2345, priority 21/79

#### rc.bnetd
BSD-style init script for Slackware, FreeBSD, OpenBSD systems.

**Installation**
```bash
# Copy to rc.d
sudo cp rc.bnetd /etc/rc.d/rc.bnetd
sudo chmod +x /etc/rc.d/rc.bnetd

# Enable in rc.local
echo "/etc/rc.d/rc.bnetd start" >> /etc/rc.local
```

#### S98bnetd
System V runlevel script for automatic startup.

**Installation**
```bash
# Link to appropriate runlevels
sudo ln -s /etc/init.d/bnetd /etc/rc2.d/S98bnetd
sudo ln -s /etc/init.d/bnetd /etc/rc3.d/S98bnetd
sudo ln -s /etc/init.d/bnetd /etc/rc5.d/S98bnetd

# Link for shutdown
sudo ln -s /etc/init.d/bnetd /etc/rc0.d/K01bnetd
sudo ln -s /etc/init.d/bnetd /etc/rc1.d/K01bnetd
sudo ln -s /etc/init.d/bnetd /etc/rc6.d/K01bnetd
```

#### bnetd.logrotate
Log rotation configuration for `/var/log/bnetd/bnetd.log`.

**Installation**
```bash
sudo cp bnetd.logrotate /etc/logrotate.d/bnetd
```

**Features**
- Rotates logs 9 times (keeps 9 old versions)
- `missingok`: Don't error if log missing
- `notifempty`: Don't rotate empty logs
- `postrotate`: Sends HUP signal to reload logs

**Manual Rotation**
```bash
sudo logrotate -f /etc/logrotate.d/bnetd
```

### 2. Account Management

#### make_testusers.sh
Creates massive numbers of dummy test accounts for load testing and development.

**Configuration Variables**
```bash
numaccts=400              # Number of accounts to create
name="bob"                # Account name prefix
pass="bob"                # Password for all accounts
padding=6                 # Zero-padding width (bob000001)
users=/path/to/users      # User directory
bnpass=/path/to/bnpass    # Password hasher
```

**Usage**
```bash
./make_testusers.sh

# Creates accounts:
#   bob000001, bob000002, ..., bob000400
# All with password "bob"
```

**Generated Account Format**
```
# File: users/000001
"BNET\\acct\\username"="bob000001"
"BNET\\acct\\passhash1"="a9993e364706816aba3e25717850c26c9cd0d89d"
"BNET\\acct\\userid"="1"
```

**Use Cases**
- Load testing server capacity
- Stress testing connection handling
- Database performance benchmarking
- Network bandwidth testing

#### login_testusers.sh
Automatically logs in test accounts using `bnchat` for connection testing.

**Usage**
```bash
./login_testusers.sh [server] [num_accounts]

# Example: Login 100 test users to localhost
./login_testusers.sh localhost 100
```

**Requirements**
- `bnchat` binary in PATH
- Test accounts created with `make_testusers.sh`
- Server running and accepting connections

#### lastlogin.pl
Extracts last login timestamps from account files.

**Usage**
```bash
./lastlogin.pl /var/pvpgn/users

# Output:
# bob000001: 2025-11-24 15:42:10
# bob000002: 2025-11-23 08:30:45
# ...
```

**Applications**
- Account activity monitoring
- Inactive account cleanup
- Usage statistics generation

### 3. Database Migration Tools

#### flat2cdb.pl
Converts plain text account files to CDB (Constant Database) format for faster lookups.

**Usage**
```bash
./flat2cdb.pl /var/pvpgn/users /var/pvpgn/users_cdb

# Converts:
#   users/000001 → users_cdb/000001 (CDB format)
```

**CDB Benefits**
- O(1) lookup time (vs linear search)
- Smaller file size (binary format)
- No locking required for reads
- Fast server restarts

**Format Conversion**
```
Plain Text:
  "BNET\\acct\\username"="Player1"
  "BNET\\acct\\userid"="123"

CDB:
  +13,7:BNET\acct\username->Player1
  +11,3:BNET\acct\userid->123
```

**Requirements**
- `cdb` command-line tool (`/usr/local/bin/cdb`)
- Install: `apt-get install tinycdb` or compile from source

#### storage/plain2sql.pl
Migrates plain text accounts to SQL database format.

**Usage**
```bash
./plain2sql.pl /var/pvpgn/users > accounts.sql

# Generates SQL INSERT statements:
# INSERT INTO BNET (uid, acct_username, acct_userid) 
#   VALUES (1, 'Player1', '123');
```

**Supported SQL Backends**
- MySQL
- PostgreSQL
- SQLite

**Output Format**
```sql
-- Account: Player1
INSERT INTO BNET (uid, acct_username, acct_passhash1, acct_userid, acct_lastlogin_time)
VALUES (1, 'Player1', 'a9993e364706816aba3e25717850c26c9cd0d89d', 123, 1732464130);

-- Account: Player2
INSERT INTO BNET (uid, acct_username, acct_passhash1, acct_userid, acct_lastlogin_time)
VALUES (2, 'Player2', 'b8693e364706816aba3e25717850c26c9cd0d89f', 124, 1732464145);
```

**Migration Steps**
```bash
# 1. Generate SQL
./plain2sql.pl /var/pvpgn/users > migrate.sql

# 2. Import to database
mysql -u pvpgn -p pvpgn_db < migrate.sql

# 3. Update bnetd.conf
storage_path = "sql:mode=mysql;host=localhost;name=pvpgn_db;user=pvpgn;pass=secret"

# 4. Restart server
sudo service bnetd restart
```

#### storage/cdb2sql.pl
Converts CDB format accounts to SQL.

**Usage**
```bash
./cdb2sql.pl /var/pvpgn/users_cdb > accounts.sql
```

#### sql/repairlevels.pl
Repairs corrupted character level data in SQL databases.

**Usage**
```bash
./repairlevels.pl --host=localhost --user=pvpgn --pass=secret --db=pvpgn_db

# Fixes:
# - Invalid level values (< 1 or > 99)
# - Mismatched experience/level
# - Corrupted character stats
```

### 4. Game-Specific Tools

#### convert_w3.pl
Converts Warcraft III account data between storage formats.

**Purpose**
Migrates Warcraft III statistics (solo/team/FFA records, race wins) to new schema format with `WAR3` prefix.

**Usage**
```bash
./convert_w3.pl /var/pvpgn/users /var/pvpgn/users_converted
```

**Conversion Example**
```
Before:
  "record\\solo\\wins"="25"
  "team\\2v2\\wins"="18"

After:
  "record\\WAR3\\solo\\wins"="25"
  "team\\WAR3\\2v2\\wins"="18"
```

**Affected Fields**
- `record\solo`, `record\team`, `record\ffa`
- `record\orcs`, `record\humans`, `record\nightelves`, `record\undead`, `record\random`
- All team statistics

#### ladder.py
Diablo II ladder viewer and HTML/ANSI/ASCII generator.

**Features**
- Reads binary D2 ladder files from D2DBS
- Supports 35 ladder types (classes, difficulty, modes)
- Multiple output formats: HTML, ANSI color, ASCII, Python dict
- CGI mode for web integration
- Character status display (hardcore, dead, expansion)

**Usage**

**Command Line**
```bash
# Display ladder in terminal (ANSI colors)
./ladder.py --file=/var/pvpgn/ladders/ladder.D2DV --format=ansi --max=50

# Generate HTML page
./ladder.py --file=/var/pvpgn/ladders/ladder.D2DV --format=html > ladder.html

# ASCII output (no colors)
./ladder.py --file=/var/pvpgn/ladders/ladder.D2DV --format=ascii
```

**CGI Mode**
```python
# In ladder.py, set:
CGI_MODE = 1
FILE = "/opt/bnetd/var/ladders/ladder.D2DV"
MAX = 100

# Apache configuration:
# ScriptAlias /ladder.py /var/www/cgi-bin/ladder.py

# Access: http://yourserver/ladder.py
```

**Output Formats**

**HTML**
```html
<!DOCTYPE html>
<html>
<head><title>D2 Closed Realm Ladder</title></head>
<body bgcolor="#000000" text="#ffff00">
  <h2>D2 Closed Realm Ladder</h2>
  <table>
    <tr><th>#</th><th>Charname</th><th>Level</th><th>Class</th><th>Exp</th></tr>
    <tr><td>1</td><td>Barbarian</td><td>99</td><td>Barbarian</td><td>3520485254</td></tr>
  </table>
</body>
</html>
```

**ANSI (Terminal Colors)**
```
D2 Closed Ladder
===============================================================
Ladder for Standard Expansion Overall
#    Charname             level   class        exp
1    Sorceress            99      Sorceress    3520485254
2    Paladin             98      Paladin      3020485123
```

**Ladder Types Supported**
- Standard: Amazon, Sorceress, Necromancer, Paladin, Barbarian
- Expansion: + Druid, Assassin
- Hardcore versions of all
- Overall ladders (top 1000)
- Class ladders (top 200)

**Character Status**
- Normal: Black text
- Hardcore (alive): Red text
- Hardcore (dead): Orange text
- Expansion: Includes Druid/Assassin

#### fsgs2bnetd.pl
Imports accounts from FSGS (Free Starcraft Game Server) to PvPGN format.

**Usage**
```bash
./fsgs2bnetd.pl /path/to/fsgs/users /var/pvpgn/users
```

**Conversion**
- Translates FSGS account schema to BNET schema
- Preserves usernames and passwords
- Converts statistics and records

### 5. Server Administration

#### announce.sh
Automated periodic server announcements using `bnchat`.

**Usage**
```bash
./announce.sh server account password interval "message"

# Example: Announce every 30 seconds
./announce.sh localhost admin adminpass 30 "Server restart in 10 minutes!"

# One-time announcement (interval=0)
./announce.sh localhost admin adminpass 0 "Welcome to our server!"
```

**Parameters**
- `server`: Server hostname or IP
- `account`: Admin account name
- `password`: Account password
- `interval`: Seconds between announcements (0 = one-time)
- `message`: Text to announce (use \n for newlines)

**Features**
- Uses named pipe for communication with `bnchat`
- Cleans up on SIGINT/SIGTERM
- Requires account with `/announce` or admin permissions
- Supports multi-line messages

**Example**
```bash
./announce.sh localhost admin pass 300 "Attention:\nServer maintenance tonight\nExpected downtime: 30 minutes"

# Output in chat:
# [Announce] Attention:
# [Announce] Server maintenance tonight
# [Announce] Expected downtime: 30 minutes
```

**Process Flow**
1. Create named pipe `/tmp/pipe-bnannounce-$$`
2. Launch `bnchat` connected to pipe
3. Send login commands
4. Loop: Send announce command every `interval` seconds
5. Cleanup pipe on exit

#### bnetmasq.sh
UDP masquerading script for Linux firewalls to enable Battle.net behind NAT.

**Problem**: Battle.net clients behind NAT can't establish peer-to-peer game connections because UDP masquerading requires manual port forwarding.

**Solution**: Automates `ipmasqadm` port forwarding rules for each client.

**Configuration**
```bash
# Port on clients (usually 6112)
BNET_PORT=6112

# List of internal_ip:external_port mappings
REDIR_LIST="192.168.1.10:5000 192.168.1.11:5001 192.168.1.12:5002"

# External interface (eth0, ppp0, etc.)
EXTERNAL_IF=eth0
```

**Usage**
```bash
# Start masquerading
./bnetmasq.sh start

# Stop masquerading
./bnetmasq.sh stop

# Restart
./bnetmasq.sh restart
```

**How It Works**
```
External IP: 203.0.113.50

Client 192.168.1.10 → Forwarded to 203.0.113.50:5000 → Internal 192.168.1.10:6112
Client 192.168.1.11 → Forwarded to 203.0.113.50:5001 → Internal 192.168.1.11:6112
Client 192.168.1.12 → Forwarded to 203.0.113.50:5002 → Internal 192.168.1.12:6112
```

**Installation**
```bash
# Copy to init.d
sudo cp bnetmasq.sh /etc/init.d/bnetmasq
sudo chmod +x /etc/init.d/bnetmasq

# Auto-start on boot
sudo update-rc.d bnetmasq defaults

# For PPP dial-up connections
sudo ln -s /etc/init.d/bnetmasq /etc/ppp/ip-up.d/bnetmasq
sudo ln -s /etc/init.d/bnetmasq /etc/ppp/ip-down.d/bnetmasq
```

**Note**: Not needed with "loose UDP" kernel patch (2.2+ kernels with automatic P2P UDP handling).

#### process.pl
General-purpose account file processing utility.

**Usage**
```bash
./process.pl /var/pvpgn/users [processing_options]
```

### 6. Localization Tools

#### localize/pvpgn_localize_generator.cs
C# utility that extracts translatable strings from PvPGN source code.

**Purpose**
Scans C++, header, and Lua files for `localize(...)` function calls and generates XML translation template.

**Usage**
```bash
# Windows
pvpgn_localize_generator.exe C:\pvpgn\src output.xml

# Update existing translations
pvpgn_localize_generator.exe C:\pvpgn\src i18n\common.xml
```

**Output Format (XML)**
```xml
<?xml version="1.0"?>
<root>
  <item>
    <file>bnetd/server.cpp</file>
    <function>server_process_connection</function>
    <line>428</line>
    <original>Connection from {}</original>
    <translation></translation>
  </item>
  <item>
    <file>bnetd/account.cpp</file>
    <function>account_create</function>
    <line>156</line>
    <original>Account created successfully</original>
    <translation></translation>
  </item>
</root>
```

**Workflow**
1. Run generator to extract strings
2. Translators fill in `<translation>` fields
3. Save as `i18n/language.xml` (e.g., `i18n/russian.xml`)
4. Configure in `bnetd.conf`: `i18nfilename = "i18n/russian.xml"`

#### localize/pvpgn_localize_validator.cs
Validates translation XML files for completeness and syntax.

**Usage**
```bash
# Windows
pvpgn_localize_validator.exe i18n\russian.xml

# Check output:
# ✓ All strings translated
# ✗ Missing translations: 3
# ✗ Invalid XML syntax: line 45
```

**Validations**
- XML well-formedness
- Required fields present (file, function, original, translation)
- Translation completeness percentage
- Duplicate entries
- Orphaned translations (no matching source)

#### localize/update.bat & validate.bat
Windows batch scripts to automate localization workflow.

**update.bat**
```batch
@echo off
pvpgn_localize_generator.exe ..\..\src i18n\common.xml
echo Localization template updated
pause
```

**validate.bat**
```batch
@echo off
for %%f in (i18n\*.xml) do (
    echo Validating %%f
    pvpgn_localize_validator.exe %%f
)
pause
```

#### tos.sh / tos.bat
Generate Terms of Service files for all supported languages.

**Supported Languages**
- RUS (Russian), POL (Polish), CHI (Chinese)
- SIN (Simplified Chinese), HAN (Traditional Chinese)
- KOR (Korean), JPN (Japanese), ITA (Italian)
- BRA (Brazilian Portuguese), POR (Portuguese)
- FRA (French), DEU (German), ESP (Spanish)
- USA (US English), ENU (English)

**Usage**

**Linux/Unix**
```bash
./tos.sh /path/to/files

# Generates in /path/to/files:
#   tos_RUS.txt, tos_POL.txt, ...
#   tos-unicode_RUS.txt, tos-unicode_POL.txt, ...
```

**Windows**
```batch
cd scripts
tos.bat

REM Generates in ..\files:
REM   tos_RUS.txt, tos_POL.txt, ...
REM   tos-unicode_RUS.txt, tos-unicode_POL.txt, ...
```

**File Formats**
- `tos_XXX.txt`: ANSI encoding (legacy clients)
- `tos-unicode_XXX.txt`: UTF-8 encoding (modern clients)

**Customization**
1. Edit `files/tos.txt` (base template)
2. Run `tos.sh` or `tos.bat`
3. Translate each generated file to target language
4. Clients receive correct ToS based on language tag

### 7. Security & Hashing

#### pvpgn_hash.inc.php
PHP implementation of PvPGN's password hashing algorithm (SHA-1 variant).

**Purpose**
Enables external PHP applications (web registration, password reset) to generate compatible password hashes.

**Usage**
```php
<?php
require_once 'pvpgn_hash.inc.php';

$username = "Player1";
$password = "secret123";

// Generate hash
$hash = pvpgn_hash($password);
// Returns: "a9993e364706816aba3e25717850c26c9cd0d89d"

// Store in account file or database
$account_data = [
    'BNET\\acct\\username' => $username,
    'BNET\\acct\\passhash1' => $hash
];
```

**Algorithm**
PvPGN uses a modified SHA-1 algorithm with specific padding and block processing compatible with Battle.net's original implementation.

**Integration Examples**

**Web Registration Form**
```php
<?php
require_once 'pvpgn_hash.inc.php';

if ($_POST['register']) {
    $username = $_POST['username'];
    $password = $_POST['password'];
    
    // Validate input
    if (strlen($username) < 3 || strlen($password) < 6) {
        die("Invalid username or password");
    }
    
    // Generate hash
    $hash = pvpgn_hash($password);
    
    // Create account file
    $uid = get_next_userid();
    $account_file = "/var/pvpgn/users/" . sprintf("%06d", $uid);
    
    $data = [
        '"BNET\\acct\\username"' => '"' . $username . '"',
        '"BNET\\acct\\passhash1"' => '"' . $hash . '"',
        '"BNET\\acct\\userid"' => '"' . $uid . '"',
        '"BNET\\acct\\ctime"' => '"' . time() . '"'
    ];
    
    $content = "";
    foreach ($data as $key => $value) {
        $content .= $key . "=" . $value . "\n";
    }
    
    file_put_contents($account_file, $content);
    echo "Account created successfully!";
}
?>
```

**Password Reset**
```php
<?php
require_once 'pvpgn_hash.inc.php';

function reset_password($username, $new_password) {
    $user_file = find_account_file($username);
    if (!$user_file) {
        return false;
    }
    
    // Read existing account
    $lines = file($user_file);
    $new_hash = pvpgn_hash($new_password);
    
    // Update hash
    foreach ($lines as &$line) {
        if (strpos($line, 'BNET\\acct\\passhash1') !== false) {
            $line = '"BNET\\acct\\passhash1"="' . $new_hash . '"' . "\n";
        }
    }
    
    file_put_contents($user_file, implode("", $lines));
    return true;
}
?>
```

#### pvpgn_wol_hash.inc.php
Westwood Online password hashing for Command & Conquer games.

**Supported Games**
- Command & Conquer: Red Alert
- Command & Conquer: Tiberian Sun
- Command & Conquer: Red Alert 2

**Usage**
```php
<?php
require_once 'pvpgn_wol_hash.inc.php';

$password = "tankrush";
$hash = pvpgn_wol_hash($password);

// Store in WOL account
echo '"WOL\\acct\\passhash"="' . $hash . '"';
?>
```

### 8. Development Tools

#### cvs2cl.pl
Generates ChangeLog from CVS repository history.

**Usage**
```bash
# Generate ChangeLog
./cvs2cl.pl

# Custom date range
./cvs2cl.pl --since "2024-01-01" --until "2024-12-31"

# Specific files/directories
./cvs2cl.pl src/bnetd/

# Output to specific file
./cvs2cl.pl --file CHANGELOG.txt
```

**Output Format**
```
2025-11-24  Developer Name <dev@example.com>

    * src/bnetd/server.cpp: Fixed memory leak in connection handler
    * src/common/eventlog.cpp: Added debug logging level

2025-11-23  Another Dev <dev2@example.com>

    * src/d2cs/charlock.cpp: Improved character locking algorithm
```

**Note**: For modern Git repositories, use `git log` instead:
```bash
git log --pretty=format:"%ad  %an <%ae>%n%n    * %s%n" --date=short > ChangeLog
```

#### reform.awk
AWK script for text reformatting tasks.

**Usage**
```bash
# Reformat text file
awk -f reform.awk input.txt > output.txt
```

## Installation Guide

### Prerequisites

**Linux/Unix**
```bash
# Perl
sudo apt-get install perl libcdb-file-perl  # Debian/Ubuntu
sudo yum install perl perl-CDB_File          # RHEL/CentOS

# Python
sudo apt-get install python3

# PHP
sudo apt-get install php-cli

# C# (Mono)
sudo apt-get install mono-complete
```

**Windows**
```powershell
# Perl (Strawberry Perl)
choco install strawberryperl

# Python
choco install python

# PHP
choco install php

# C# (.NET)
# Executables already compiled in localize/
```

### Script Deployment

**System Scripts**
```bash
# Init scripts
sudo cp bnetd.init /etc/rc.d/init.d/bnetd
sudo chmod +x /etc/rc.d/init.d/bnetd
sudo chkconfig bnetd on

# Log rotation
sudo cp bnetd.logrotate /etc/logrotate.d/bnetd

# Verify
sudo service bnetd status
sudo logrotate -d /etc/logrotate.d/bnetd
```

**Admin Scripts**
```bash
# Install to /usr/local/bin
sudo cp announce.sh /usr/local/bin/pvpgn-announce
sudo cp ladder.py /usr/local/bin/pvpgn-ladder
sudo chmod +x /usr/local/bin/pvpgn-*

# Create symlinks
cd /usr/local/bin
sudo ln -s pvpgn-ladder ladder
```

**Web Scripts**
```bash
# CGI ladder viewer
sudo cp ladder.py /var/www/cgi-bin/ladder.py
sudo chmod +x /var/www/cgi-bin/ladder.py

# Edit ladder.py:
#   CGI_MODE = 1
#   FILE = "/var/pvpgn/ladders/ladder.D2DV"

# Apache configuration
sudo a2enmod cgi
# Add to virtualhost:
#   ScriptAlias /ladder.py /var/www/cgi-bin/ladder.py

sudo systemctl restart apache2
```

## Usage Examples

### Complete Server Setup

**1. Install PvPGN**
```bash
git clone https://github.com/pvpgn/pvpgn-server.git
cd pvpgn-server
mkdir build && cd build
cmake -DCMAKE_INSTALL_PREFIX=/usr/local/pvpgn ..
make && sudo make install
```

**2. Configure Startup**
```bash
cd ../scripts
sudo cp bnetd.init /etc/init.d/bnetd
sudo chmod +x /etc/init.d/bnetd
sudo update-rc.d bnetd defaults
```

**3. Configure Log Rotation**
```bash
sudo cp bnetd.logrotate /etc/logrotate.d/bnetd
```

**4. Create Test Accounts**
```bash
cd /usr/local/pvpgn
./scripts/make_testusers.sh
# Creates bob000001 - bob000400
```

**5. Start Server**
```bash
sudo service bnetd start
```

**6. Test Connections**
```bash
./scripts/login_testusers.sh localhost 50
# Logs in 50 test users
```

### Database Migration

**Plain → SQL Migration**
```bash
# 1. Backup existing accounts
tar -czf accounts_backup.tar.gz /var/pvpgn/users

# 2. Generate SQL
cd scripts/storage
./plain2sql.pl /var/pvpgn/users > migrate.sql

# 3. Create database
mysql -u root -p
CREATE DATABASE pvpgn_db;
GRANT ALL ON pvpgn_db.* TO 'pvpgn'@'localhost' IDENTIFIED BY 'password';
exit

# 4. Import
mysql -u pvpgn -p pvpgn_db < migrate.sql

# 5. Update bnetd.conf
storage_path = "sql:mode=mysql;host=localhost;name=pvpgn_db;user=pvpgn;pass=password"

# 6. Restart
sudo service bnetd restart

# 7. Verify
bnchat localhost
# Login with test account
```

### Ladder Web Page

**Setup**
```bash
# 1. Copy script
sudo cp scripts/ladder.py /var/www/html/d2ladder.py
sudo chmod +x /var/www/html/d2ladder.py

# 2. Edit configuration
sudo nano /var/www/html/d2ladder.py
# Set:
#   CGI_MODE = 1
#   FILE = "/var/pvpgn/ladders/ladder.D2DV"
#   MAX = 100

# 3. Configure web server
# Apache:
sudo a2enmod cgi
# In virtualhost: AddHandler cgi-script .py

# Nginx:
# location ~ \.py$ {
#     include fastcgi_params;
#     fastcgi_pass unix:/var/run/fcgiwrap.socket;
# }

sudo systemctl reload apache2

# 4. Access
# http://yourserver/d2ladder.py
```

**Automated Updates**
```bash
# Cron job to regenerate HTML every 5 minutes
crontab -e

# Add:
*/5 * * * * /var/www/html/d2ladder.py > /var/www/html/ladder.html 2>&1
```

### Localization Workflow

**1. Extract Strings**
```bash
cd scripts/localize
./pvpgn_localize_generator.exe ../../src i18n/template.xml
```

**2. Create Translation**
```bash
cp i18n/template.xml i18n/russian.xml

# Edit russian.xml, fill in <translation> tags
nano i18n/russian.xml
```

**3. Validate**
```bash
./pvpgn_localize_validator.exe i18n/russian.xml
# Check for errors
```

**4. Deploy**
```bash
cp i18n/russian.xml /usr/local/pvpgn/i18n/

# Edit bnetd.conf:
i18nfilename = "i18n/russian.xml"

sudo service bnetd restart
```

**5. Generate ToS Files**
```bash
cd ../
./tos.sh ../files

# Translate each tos_XXX.txt file
nano ../files/tos_RUS.txt
```

## Troubleshooting

### Init Script Issues

**Service won't start**
```bash
# Check permissions
ls -l /etc/init.d/bnetd
# Should be: -rwxr-xr-x root root

# Check binary path
which bnetd
# Edit bnetd.init if path differs

# Check manually
sudo /usr/local/bin/bnetd -f -c /etc/pvpgn/bnetd.conf
# Look for error messages
```

**Service won't stop**
```bash
# Check PID file
cat /var/run/bnetd.pid

# Kill manually
sudo killall -TERM bnetd

# Remove stale lock
sudo rm -f /var/lock/subsys/bnetd
```

### Script Execution Issues

**Permission denied**
```bash
# Add execute bit
chmod +x script.sh

# Check shebang
head -1 script.sh
# Should be: #!/bin/sh or #!/usr/bin/perl
```

**Interpreter not found**
```bash
# /usr/bin/perl: not found
# Install Perl
sudo apt-get install perl

# /usr/bin/env python: not found
# Install Python
sudo apt-get install python3

# Or fix shebang
# Change: #!/usr/bin/env python
# To: #!/usr/bin/python3
```

### Migration Script Errors

**CDB conversion fails**
```bash
# Error: cdb command not found
sudo apt-get install tinycdb

# Error: Can't locate CDB_File.pm
sudo apt-get install libcdb-file-perl
```

**SQL import fails**
```bash
# Syntax error at line X
# Check table schema matches script expectations

# Create tables manually
mysql -u pvpgn -p pvpgn_db
source create_tables.sql
exit

# Retry import
mysql -u pvpgn -p pvpgn_db < migrate.sql
```

### Ladder Script Issues

**ladder.py errors**
```python
# struct.error: unpack requires a buffer of X bytes
# Corrupted ladder file
# Regenerate:
cd /var/pvpgn/ladders
rm ladder.*
sudo service d2dbs restart
# Wait for D2DBS to rebuild ladder
```

**CGI 500 error**
```bash
# Check Apache error log
sudo tail -f /var/log/apache2/error.log

# Common fixes:
# 1. Add shebang: #!/usr/bin/env python3
# 2. Set permissions: chmod +x ladder.py
# 3. Set CGI_MODE = 1 in script
# 4. Check file path: FILE = "/var/pvpgn/ladders/ladder.D2DV"
```

## Best Practices

### Security

**Script Permissions**
```bash
# Admin scripts: root only
sudo chown root:root *.sh
sudo chmod 700 *.sh

# User scripts: restricted
sudo chown pvpgn:pvpgn *.pl
sudo chmod 750 *.pl

# Web scripts: web server user
sudo chown www-data:www-data ladder.py
sudo chmod 755 ladder.py
```

**Password Hashing**
```php
// NEVER store plain text passwords
$password = $_POST['password'];
$hash = pvpgn_hash($password);  // ✓ Good

// DON'T do this:
file_put_contents("account.txt", "password=$password");  // ✗ Bad
```

### Performance

**Optimize Ladder Generation**
```bash
# Cache ladder HTML
*/5 * * * * /usr/local/bin/pvpgn-ladder > /var/www/html/ladder.html

# Don't regenerate on every request
```

**Database Indexing**
```sql
-- Index frequently queried columns
CREATE INDEX idx_username ON BNET(acct_username);
CREATE INDEX idx_userid ON BNET(acct_userid);
CREATE INDEX idx_lastlogin ON BNET(acct_lastlogin_time);
```

### Maintenance

**Regular Backups**
```bash
#!/bin/bash
# Daily backup script

DATE=$(date +%Y%m%d)
BACKUP_DIR=/backup/pvpgn

# Backup accounts
tar -czf $BACKUP_DIR/users-$DATE.tar.gz /var/pvpgn/users

# Backup config
tar -czf $BACKUP_DIR/conf-$DATE.tar.gz /etc/pvpgn

# Backup database (if using SQL)
mysqldump -u pvpgn -p pvpgn_db > $BACKUP_DIR/db-$DATE.sql

# Keep only last 30 days
find $BACKUP_DIR -name "*.tar.gz" -mtime +30 -delete
find $BACKUP_DIR -name "*.sql" -mtime +30 -delete
```

**Log Monitoring**
```bash
# Setup logwatch
sudo apt-get install logwatch

# Add PvPGN logs
sudo nano /etc/logwatch/conf/logfiles/bnetd.conf
```

## Contributing

### Adding New Scripts

1. Choose appropriate category (admin, migration, localization, etc.)
2. Use standard shebang: `#!/bin/sh`, `#!/usr/bin/perl`, `#!/usr/bin/env python3`
3. Include usage documentation in comments
4. Add error handling and validation
5. Test on multiple platforms (if applicable)
6. Update this README with usage examples

### Script Guidelines

**Style**
```bash
#!/bin/bash
# Script description
# Author: Your Name
# Date: YYYY-MM-DD

set -e  # Exit on error
set -u  # Exit on undefined variable

# Configuration
VAR1="value"
VAR2="value"

# Functions
function main() {
    # Implementation
}

# Execute
main "$@"
```

**Error Handling**
```bash
# Check arguments
if [ $# -ne 2 ]; then
    echo "Usage: $0 <arg1> <arg2>" >&2
    exit 1
fi

# Check file existence
if [ ! -f "$FILE" ]; then
    echo "Error: File not found: $FILE" >&2
    exit 1
fi

# Check command availability
if ! command -v bnetd &> /dev/null; then
    echo "Error: bnetd not found in PATH" >&2
    exit 1
fi
```

## Related Documentation

- **Configuration**: `conf/bnetd.conf.in`, `conf/d2cs.conf.in`, `conf/d2dbs.conf.in`
- **Man Pages**: `man/bnetd.1`, `man/bnchat.1`, `man/bnpass.1`
- **Storage**: `docs/storage.txt` - Account storage formats
- **Client Tools**: `src/client/` - bnchat, bnbot, bnpass utilities

## Support

- **GitHub Issues**: https://github.com/pvpgn/pvpgn-server/issues
- **PvPGN Forums**: https://pvpgn.pro/
- **Discord**: https://discord.gg/VpARdYJ

## License

Scripts in this directory are licensed under GNU General Public License v2 or later unless otherwise specified. See individual files for copyright and authorship information.

**Exceptions**:
- `pvpgn_localize_generator.cs`, `pvpgn_localize_validator.cs`: MIT License (HarpyWar)
- `ladder.py`: GPL v2 (Gianluigi Tiesi)

---

**Last Updated**: November 24, 2025  
**PvPGN Version**: Latest development branch
