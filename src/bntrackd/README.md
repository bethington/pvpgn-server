# bntrackd - Battle.net Tracking Daemon

## Overview

`bntrackd` is a lightweight UDP server that collects and aggregates real-time statistics from multiple PvPGN/bnetd game servers across the internet. It acts as a central registry that receives periodic "heartbeat" announcements from game servers, maintains an active server list with current player counts and metadata, and publishes this information in machine-readable formats (text or XML) for display on websites or other applications.

## Purpose

The primary purposes of bntrackd are:

1. **Server Discovery**: Maintain a centralized directory of active PvPGN servers
2. **Statistics Aggregation**: Collect real-time metrics (users online, games running, channels active)
3. **Server Monitoring**: Track server uptime and automatically expire stale entries
4. **Web Publishing**: Generate HTML-ready output for displaying server lists on websites
5. **Network Health**: Provide visibility into the health of the PvPGN server network

## Architecture

### High-Level Design

```
┌─────────────┐          ┌─────────────┐          ┌─────────────┐
│ PvPGN Server│          │ PvPGN Server│          │ PvPGN Server│
│   (bnetd)   │          │   (bnetd)   │          │   (bnetd)   │
└──────┬──────┘          └──────┬──────┘          └──────┬──────┘
       │                        │                        │
       │ UDP Announce (6114)    │                        │
       │ Every ~5 minutes       │                        │
       └────────────┬───────────┴────────────────────────┘
                    ▼
         ┌─────────────────────┐
         │     bntrackd        │
         │  (Tracking Daemon)  │
         │  Listens on UDP     │
         │  Port 6114          │
         └──────────┬──────────┘
                    │
         ┌──────────▼──────────┐
         │   Server List       │
         │   (In-Memory)       │
         │ ┌─────────────────┐ │
         │ │ Server 1 (Age)  │ │
         │ │ Server 2 (Age)  │ │
         │ │ Server 3 (Age)  │ │
         │ └─────────────────┘ │
         └──────────┬──────────┘
                    │
         ┌──────────▼──────────┐
         │  Periodic Output    │
         │  Every 150 seconds  │
         │ ┌─────────────────┐ │
         │ │ bnetdlist.txt   │ │ (Plain text or XML)
         │ │ bnetdlist.xml   │ │
         │ └─────────────────┘ │
         └──────────┬──────────┘
                    │
         ┌──────────▼──────────┐
         │ Post-Process Hook   │
         │  (scripts/process.pl│
         │   or custom command)│
         │ ┌─────────────────┐ │
         │ │ bnetdlist.html  │ │ (Generated HTML)
         │ └─────────────────┘ │
         └─────────────────────┘
```

### Core Components

#### 1. UDP Listener

- **Bind**: Listens on UDP port 6114 (configurable via `-p`)
- **Non-Blocking**: Uses `select()` with configurable timeout (BNTRACKD_GRANULARITY)
- **Packet Format**: Receives `t_trackpacket` structures from bnetd servers

#### 2. Server List Management (`t_server`)

Each tracked server entry contains:

```cpp
typedef struct {
    struct in_addr address;    // Server IP address
    time_t         updated;    // Last announcement timestamp
    t_trackpacket  info;       // Server metadata and statistics
} t_server;
```

**Lifecycle**:
- **Addition**: When first announcement received from new IP
- **Update**: Refresh `updated` timestamp and `info` on subsequent announcements
- **Expiration**: Remove entry if no announcement within `expire` seconds (default: 600s)
- **Shutdown**: Remove immediately if `TF_SHUTDOWN` flag set in announcement

#### 3. Tracking Packet Structure (`t_trackpacket`)

The announcement packet (defined in `src/common/tracker.h`) contains:

| Field | Type | Size | Description |
|-------|------|------|-------------|
| `packet_version` | `bn_short` | 2 | Protocol version (TRACK_VERSION=2) |
| `port` | `bn_short` | 2 | Server TCP port (network byte order) |
| `flags` | `bn_int` | 4 | Status flags (TF_SHUTDOWN, TF_PRIVATE) |
| `software` | `bn_byte[32]` | 32 | Software name ("PvPGN", "bnetd") |
| `version` | `bn_byte[16]` | 16 | Version string ("2.0.0") |
| `platform` | `bn_byte[32]` | 32 | Operating system ("Linux", "Windows") |
| `server_desc` | `bn_byte[64]` | 64 | Server description |
| `server_location` | `bn_byte[64]` | 64 | Geographic location |
| `server_url` | `bn_byte[96]` | 96 | Website URL |
| `contact_name` | `bn_byte[64]` | 64 | Operator name |
| `contact_email` | `bn_byte[64]` | 64 | Operator email |
| `users` | `bn_int` | 4 | Current users online |
| `channels` | `bn_int` | 4 | Current channels |
| `games` | `bn_int` | 4 | Current active games |
| `uptime` | `bn_int` | 4 | Server uptime (seconds) |
| `total_games` | `bn_int` | 4 | Lifetime games served |
| `total_logins` | `bn_int` | 4 | Lifetime login count |

**Flags**:
- `TF_SHUTDOWN (0x1)`: Server is shutting down (remove from list)
- `TF_PRIVATE (0x2)`: Server is private (currently ignored)

#### 4. Output Generation

The tracker periodically writes server information to disk in two formats:

**Plain Text Format** (default):
```
<IP_ADDRESS>
##
<PORT>
##
<LOCATION>
##
<SOFTWARE>
##
<VERSION>
##
<USERS>
##
...
###
```

**XML Format** (with `-x` option):
```xml
<server>
    <address>192.168.1.1</address>
    <port>6112</port>
    <location>United States</location>
    <software>PvPGN</software>
    <version>2.0.0</version>
    <users>42</users>
    <channels>15</channels>
    <games>8</games>
    <description>Public PvPGN Server</description>
    <platform>Linux</platform>
    <url>http://example.com</url>
    <contact_name>Admin</contact_name>
    <contact_email>admin@example.com</contact_email>
    <uptime>86400</uptime>
    <total_games>12345</total_games>
    <logins>67890</logins>
</server>
```

#### 5. Post-Processing Hook

After writing the output file, bntrackd executes an external command:
- **Default**: `scripts/process.pl` (Perl script generating HTML)
- **Custom**: Specify via `-c` or `--command`
- **Disable**: Pass empty string (`-c ""`)

### Main Event Loop

```cpp
while (true) {
    // 1. Check if it's time to write output
    if (time_since_last_update >= update_interval) {
        // Write server list to outfile
        // Expire stale servers
        // Execute post-processing command
    }
    
    // 2. Select on UDP socket with timeout
    select(sockfd, &rfds, NULL, NULL, &tv);
    
    // 3. Process incoming tracking packet
    if (packet_received) {
        // Validate packet version
        // Sanitize string fields
        // Find existing server entry by IP
        if (found) {
            if (TF_SHUTDOWN) {
                remove_server();
            } else {
                update_server_info();
            }
        } else {
            if (!TF_SHUTDOWN) {
                add_new_server();
            }
        }
    }
}
```

### String Sanitization

The `fixup_str()` function prevents delimiter confusion:
- Replaces consecutive `##` sequences with `#%`
- Ensures fields don't break the delimiter-based output format
- Applied to all string fields in tracking packet

## Features

### 1. Automatic Server Discovery

Servers announce themselves—no manual configuration needed on tracker side.

### 2. Stale Server Expiration

- Default expiration: 600 seconds (10 minutes)
- Configurable via `-e` or `--expire`
- Prevents listing of dead servers

### 3. Periodic Output Updates

- Default interval: 150 seconds (2.5 minutes)
- Configurable via `-u` or `--update`
- Writes output even if no changes occurred

### 4. Dual Output Formats

- **Plain Text**: Delimited format for Perl/custom parsers
- **XML**: Structured format for modern web applications

### 5. Graceful Shutdown Handling

Servers can announce shutdown via `TF_SHUTDOWN` flag, allowing immediate removal from list.

### 6. Debug Logging

Enable verbose logging with `-d` to see:
- Each received packet with full field values
- Packet version and flags
- Server addition/update/removal events

### 7. Daemonization

On Unix systems (with `DO_DAEMONIZE`):
- Automatically backgrounds process
- Detaches from controlling terminal
- Creates new process group
- Override with `-f` for foreground mode

### 8. PID File Support

Write daemon process ID to file for init scripts:
```bash
bntrackd -P /var/run/bntrackd.pid
```

## Usage

### Command Line

```bash
bntrackd [options]
```

**Options**:

| Option | Long Form | Argument | Description |
|--------|-----------|----------|-------------|
| `-c` | `--command` | `COMMAND` | Execute COMMAND after each update (default: `scripts/process.pl`) |
| `-d` | `--debug` | - | Enable debug logging (packet details) |
| `-e` | `--expire` | `SECS` | Expire servers after SECS seconds without update (default: 600) |
| `-f` | `--foreground` | - | Don't daemonize (stay in foreground) |
| `-l` | `--logfile` | `FILE` | Write event log to FILE (default: `logs/bntrackd.log`) |
| `-o` | `--outfile` | `FILE` | Write server list to FILE (default: `bnetdlist.txt`) |
| `-p` | `--port` | `PORT` | Listen on UDP port PORT (default: 6114) |
| `-P` | `--pidfile` | `FILE` | Write process ID to FILE |
| `-u` | `--update` | `SECS` | Update output every SECS seconds (default: 150) |
| `-x` | `--XML` | - | Write output in XML format instead of plain text |
| `-h` | `--help` | - | Show usage information and exit |
| `-v` | `--version` | - | Print version number and exit |

### Examples

**Basic usage (default settings)**:
```bash
bntrackd
```
- Listens on UDP 6114
- Writes to `bnetdlist.txt` every 150 seconds
- Expires servers after 600 seconds
- Executes `scripts/process.pl` after each update

**Production deployment with XML output**:
```bash
bntrackd -x -o /var/www/servers.xml -l /var/log/bntrackd.log -P /var/run/bntrackd.pid
```
- XML output format
- Custom output location
- Event logging
- PID file for init scripts

**Debug mode with custom update interval**:
```bash
bntrackd -f -d -u 60 -e 300 -o servers.txt
```
- Foreground mode (don't daemonize)
- Debug logging enabled
- Update every 60 seconds
- Expire after 300 seconds

**Disable post-processing**:
```bash
bntrackd -c "" -o /tmp/rawlist.txt
```
- No command executed after updates
- Useful for raw data collection

**Custom processing script**:
```bash
bntrackd -c "php generate_serverlist.php" -o /tmp/servers.xml -x
```
- Execute custom PHP script after each update
- XML output format

### Web Publishing Workflow

**1. Run bntrackd**:
```bash
bntrackd -x -o /var/www/data/servers.xml -c ""
```

**2. Use XSL transformation**:
The included `servers.xsl` file transforms XML to HTML:
```xml
<?xml-stylesheet type="text/xsl" href="servers.xsl"?>
```

**3. Or use process.pl**:
```bash
bntrackd -o /tmp/bnetdlist.txt -c "perl scripts/process.pl"
```
Generates styled HTML table with:
- Server addresses (clickable `bnetd://` links)
- Player counts
- Server descriptions
- Contact information
- Sortable by users

## Configuring bnetd Servers

For servers to announce themselves to bntrackd, configure in `bnetd.conf`:

```conf
# Track server list (comma-separated)
trackserv = track.pvpgn.org:6114,backup-track.example.com:6114

# Server description (shown in tracker)
servername = "My PvPGN Server"
location = "United States, California"
url = "http://mypvpgn.example.com"
contact_name = "Admin"
contact_email = "admin@example.com"
```

The bnetd daemon will send UDP announcements to all listed tracking servers every ~5 minutes.

## Integration with PvPGN

bntrackd is designed to work with the PvPGN server ecosystem:

```
[bnetd] ──UDP announce──→ [bntrackd] ──writes──→ [servers.xml]
                                                        │
                                                        ▼
                                                  [Web Server]
                                                        │
                                                        ▼
                                                  [Player Browser]
```

**Typical deployment scenarios**:

1. **Master Tracker**: Central server (e.g., `track.pvpgn.org`) aggregating all public servers
2. **Regional Trackers**: Geographic-specific trackers for reduced latency
3. **Private Networks**: Track internal game servers for LAN parties or private communities

## Output File Processing

### Using process.pl (Default)

The included Perl script generates HTML:

**Configuration** (edit `scripts/process.pl`):
```perl
$infile  = "./bnetdlist.txt";      # Input from bntrackd
$outfile = "./bnetdlist.html";     # Generated HTML
```

**Output features**:
- Sortable HTML table
- Customizable colors/styling
- Player count sorting
- Clickable server links
- Total statistics

### Using XSL Transformation

For XML mode (`-x`):

1. Reference `servers.xsl` in XML header
2. Browser performs client-side transformation
3. CSS styling via `main.css`

**Example**:
```bash
# Generate XML with XSL reference
bntrackd -x -o /var/www/servers.xml

# Serve via HTTP
cd /var/www && python3 -m http.server 8080

# Browser loads: http://localhost:8080/servers.xml
# XSL transforms to HTML automatically
```

### Custom Processing

Write your own processor in any language:

**Python example**:
```python
#!/usr/bin/env python3
import xml.etree.ElementTree as ET

tree = ET.parse('/tmp/servers.xml')
servers = tree.findall('.//server')

for server in servers:
    addr = server.find('address').text
    users = server.find('users').text
    print(f"{addr}: {users} users")
```

Set as post-processor:
```bash
bntrackd -x -c "python3 /path/to/processor.py"
```

## Dependencies

- **Common Libraries**:
  - `common/tracker.h`: Tracking packet structure definition
  - `common/eventlog.h`: Logging infrastructure
  - `common/list.h`: Server list management
  - `common/xalloc.h`: Safe memory allocation
  - `common/bn_type.h`: Battle.net byte-order types

- **System Libraries**:
  - POSIX sockets (`sys/socket.h`, `arpa/inet.h`)
  - Standard C++ library (`<ctime>`, `<cstring>`, `<cstdio>`)

- **Compatibility Layer**:
  - `compat/psock.h`: Cross-platform socket API
  - `compat/pgetpid.h`: Portable `getpid()`

## Building

bntrackd is built as part of the PvPGN project:

```bash
cd build
cmake ..
make bntrackd
```

The binary is typically installed to `sbin/bntrackd`.

## Code Structure

```
src/bntrackd/
├── bntrackd.cpp       (762 lines) - Main daemon implementation
├── servers.xsl        - XSL stylesheet for XML→HTML transformation
└── CMakeLists.txt     - Build configuration

Key Functions:
├── main()             - Initialization, socket setup, daemonization
├── server_process()   - Main event loop (UDP listener + periodic updates)
├── getprefs()         - Command-line argument parsing
└── fixup_str()        - String sanitization (escape ## delimiters)
```

## Security Considerations

⚠️ **Warning**: bntrackd has minimal security features.

- **No Authentication**: Any UDP packet to port 6114 is accepted
- **IP Spoofing**: Malicious actors can announce fake servers
- **DoS Vulnerability**: Flood of packets can consume memory
- **No Rate Limiting**: Accept unlimited announcements per second

**Mitigations**:
- Run behind firewall with IP whitelist
- Use dedicated network interface
- Monitor log file for suspicious activity
- Set low `expire` timeout to limit stale entry impact

**Recommended deployment**:
```bash
# Restrict to trusted network
iptables -A INPUT -p udp --dport 6114 -s 10.0.0.0/8 -j ACCEPT
iptables -A INPUT -p udp --dport 6114 -j DROP
```

## Performance Considerations

- **Memory**: Each server entry ~500 bytes, scales linearly
- **CPU**: Minimal processing (UDP receive + periodic file write)
- **Disk I/O**: Writes output file every `update` seconds
- **Network**: Handles ~thousands of servers with low overhead

**Typical resource usage**:
- 100 servers: ~50KB RAM, negligible CPU
- 1000 servers: ~500KB RAM, <1% CPU
- 10000 servers: ~5MB RAM, <5% CPU

**Optimization tips**:
- Increase `update` interval to reduce disk writes
- Use ramdisk for output file (`/dev/shm/`)
- Disable post-processing command if not needed

## Monitoring and Maintenance

### Check Daemon Status

```bash
# Check if running
ps aux | grep bntrackd

# Check listening port
netstat -ulnp | grep 6114

# View recent log entries
tail -f logs/bntrackd.log
```

### Restart Daemon

```bash
# Using PID file
kill $(cat /var/run/bntrackd.pid)
bntrackd -P /var/run/bntrackd.pid

# Using systemd (if configured)
systemctl restart bntrackd
```

### Monitor Output

```bash
# Watch for updates
watch -n 5 'ls -lh bnetdlist.txt'

# Check server count
grep -c "###" bnetdlist.txt

# XML server count
grep -c "<server>" servers.xml
```

### Debug Connection Issues

```bash
# Test UDP connectivity from bnetd server
echo "test" | nc -u track.example.com 6114

# Capture packets
tcpdump -i any -n udp port 6114

# Enable debug mode
bntrackd -f -d -l /dev/stdout
```

## Troubleshooting

### No servers appearing in list

**Check**:
1. bnetd servers have correct `trackserv` configuration
2. Firewall allows UDP 6114 inbound
3. bntrackd is actually running (`ps aux`)
4. No errors in log file

**Test manually**:
```bash
# From bnetd machine, send test packet
echo -ne '\x00\x02\x00\x00...' | nc -u tracker-ip 6114
```

### Output file not updating

**Check**:
1. bntrackd has write permissions to output directory
2. Update interval not too long
3. Check log for file I/O errors
4. Disk not full

### Post-processing script failing

**Check**:
1. Script has execute permissions (`chmod +x`)
2. Script shebang is correct (`#!/usr/bin/perl`)
3. Required interpreters installed
4. Check stderr output: `bntrackd -c "perl script.pl 2>/tmp/error.log"`

### High memory usage

**Causes**:
- Too many servers in list (unlikely unless thousands)
- Memory leak (report bug if confirmed)

**Solutions**:
- Lower `expire` timeout to purge stale entries faster
- Restart daemon periodically via cron

## Master Tracker Network

PvPGN traditionally used a master tracker at `track.bnetd.org`:

```conf
# In bnetd.conf
trackserv = track.bnetd.org:6114,backup.tracker.net:6114
```

**Current status**: Check PvPGN community for active master trackers.

**Running your own master tracker**:
```bash
# Public server (careful with security!)
bntrackd -x -o /var/www/servers.xml -l /var/log/bntrackd.log \
  -P /var/run/bntrackd.pid -c "xsltproc servers.xsl /var/www/servers.xml > /var/www/index.html"
```

## Future Enhancements

Potential improvements for bntrackd:

1. **Rate Limiting**: Limit announcements per IP to prevent abuse
2. **IP Whitelisting**: Accept announcements only from known servers
3. **Database Backend**: Store history in SQLite/MySQL for analytics
4. **REST API**: JSON endpoint for modern web applications
5. **WebSocket Support**: Real-time server list updates
6. **Geolocation**: Automatic country detection from IP addresses
7. **Health Checks**: Active probing of listed servers
8. **Clustering**: Multiple tracker instances with synchronization

## Related Tools

- **bnetd**: Main PvPGN game server (sends announcements)
- **process.pl**: Default HTML generator script
- **servers.xsl**: XML to HTML transformation stylesheet

## References

- **Man Page**: `man/bntrackd.1` - Official manual page
- **Tracker Protocol**: `src/common/tracker.h` - Packet format definition
- **Processing Script**: `scripts/process.pl` - Perl HTML generator

## Authors

- Mark Baysinger (mbaysing@ucsd.edu) - Original implementation
- Ross Combs (rocombs@cs.nmsu.edu, ross@bnetd.org) - Enhancements and maintenance

## License

GNU General Public License v2 or later (see `LICENSE` file in project root).

---

*This README documents bntrackd as part of the PvPGN server project. Last updated: November 2024*
