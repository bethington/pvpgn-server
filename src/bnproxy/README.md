# bnproxy - Battle.net Proxy Daemon

## Overview

`bnproxy` is a transparent network proxy daemon that relays client connections to a Battle.net server (such as bnetd/PvPGN). It acts as an intermediary between Battle.net clients and the actual game server, providing packet inspection, logging capabilities, and protocol debugging features. The proxy is designed to help with network troubleshooting, protocol analysis, and potentially NAT traversal scenarios.

## Purpose

The primary purposes of bnproxy are:

1. **Protocol Analysis**: Capture and analyze Battle.net protocol traffic by logging all packets in hexadecimal/ASCII format
2. **Network Debugging**: Monitor client-server communication for troubleshooting connectivity or protocol issues
3. **Development Aid**: Assist developers in understanding Battle.net protocol implementations
4. **Transparent Relay**: Forward client connections to Battle.net servers without modification (in normal mode)
5. **NAT Traversal**: (Planned) Help clients behind NAT firewalls connect to Battle.net servers

## Architecture

### Core Components

#### 1. Virtual Connection (`virtconn.c/h`)

The virtual connection system abstracts the bidirectional proxy connection:

- **`t_virtconn`**: Represents a single proxied connection with:
  - `csd`: Client-side socket descriptor
  - `ssd`: Server-side socket descriptor
  - `class`: Connection type (bnet/file/bot)
  - `state`: Connection state (empty/initial/connecting/connected)
  - `udpaddr`/`udpport`: Real UDP endpoint for datagram forwarding
  - Separate packet queues for each direction (client↔server)

**Connection Classes**:
- `virtconn_class_bnet`: Normal Battle.net protocol (structured packets)
- `virtconn_class_file`: File transfer protocol (MPQ downloads)
- `virtconn_class_bot`: Bot/chat protocol (line-based text)
- `virtconn_class_none`: Uninitialized connection

**Connection States**:
- `virtconn_state_empty`: Connection destroyed
- `virtconn_state_initial`: Newly accepted, awaiting first packet
- `virtconn_state_connecting`: Connecting to remote server (async)
- `virtconn_state_connected`: Fully established bidirectional relay

#### 2. Main Proxy Loop (`bnproxy.c`)

The proxy operates using a classic select-based event loop:

```
┌──────────────────────────────────────────────────────────┐
│                    Listen Socket (TCP)                    │
│                      UDP Socket (6112)                     │
└────────────┬─────────────────────────────────┬────────────┘
             │                                 │
    ┌────────▼─────────┐              ┌────────▼─────────┐
    │   Client Socket   │             │   UDP Datagram    │
    │   (accept new)    │             │   (relay bidir)   │
    └────────┬──────────┘              └───────────────────┘
             │
    ┌────────▼─────────┐
    │  Create virtconn │
    │   csd + ssd      │
    └────────┬──────────┘
             │
    ┌────────▼─────────┐
    │ Connection Init  │
    │ (detect class)   │
    └────────┬──────────┘
             │
    ┌────────▼─────────────────────────────────────────────┐
    │             Bidirectional Relay Loop                  │
    │  ┌──────────────┐         ┌──────────────┐          │
    │  │ Client→Proxy │─ queue ─│ Proxy→Server │          │
    │  └──────────────┘         └──────────────┘          │
    │  ┌──────────────┐         ┌──────────────┐          │
    │  │ Server→Proxy │─ queue ─│ Proxy→Client │          │
    │  └──────────────┘         └──────────────┘          │
    └───────────────────────────────────────────────────────┘
```

### Connection Initialization Flow

When a client connects, `init_virtconn()` performs protocol detection:

1. **Wait for First Packet**: Read initial packet from client
2. **Detect Connection Class**:
   - Check for `CLIENT_INITCONN` packet → `virtconn_class_bnet`
   - Check for `CLIENT_FILE_REQ` packet → `virtconn_class_file`
   - Otherwise → `virtconn_class_bot`
3. **Establish Server Connection**: Create non-blocking TCP socket to actual server
4. **Begin Async Connect**: Initiate connection, set state to `virtconn_state_connecting`
5. **Queue Initial Packet**: Store client's first packet for forwarding after connection completes

### Packet Processing

#### TCP Packet Flow

```c
// Reading from client
net_recv_packet(csocket, packet, &currsize)
  → Determine packet size based on connection class
  → Read data into packet buffer
  → When complete: queue_push_packet(serverout_queue)

// Writing to server  
net_send_packet(ssocket, packet, &currsize)
  → Send queued packet to server
  → Track bytes sent for partial writes
  → When complete: packet_del_ref(), remove from queue

// Same logic reversed for server→client direction
```

#### UDP Packet Relay

UDP datagrams are handled separately:

- **From Server**: Forwarded to client's real UDP endpoint (`udpaddr:udpport`)
- **From Client**: Forwarded to server's address
- Currently supports single-client mode (multi-client UDP routing is broken)

### Packet Queue System

Each virtual connection maintains four queues:

1. **`cinqueue`**: Client incoming (partial packets being received)
2. **`coutqueue`**: Client outgoing (packets waiting to be sent to client)
3. **`sinqueue`**: Server incoming (partial packets being received)
4. **`soutqueue`**: Server outgoing (packets waiting to be sent to server)

This allows non-blocking I/O with buffering for partial reads/writes.

### Protocol Classes

The proxy adapts its packet handling based on detected protocol:

| Class | Packet Structure | Read Strategy | Use Case |
|-------|------------------|---------------|----------|
| `bnet` | Fixed header + variable payload | Read header, determine size, read body | Battle.net game protocol |
| `file` | File header + raw data chunks | Read file header for size, then stream chunks | MPQ file downloads |
| `bot` | Line-based text (CR/LF terminated) | Read char-by-char until newline | Chat bots, telnet |

## Features

### 1. Hexadecimal Packet Logging

When `-d <file>` is specified, all packets are logged in this format:

```
<socket>: <direction> class=<class>[0x<hex>] type=<type>[0x<hex>] length=<size>
<hexdump with ASCII representation>
```

Example:
```
1234: cli class=bnet[0x0001] type=CLIENT_INITCONN[0x01] length=38
0000  01 00 00 00  00 00 00 00  36 38 58 49  56 44 32 50  ........68XIVD2P
0010  00 00 00 00  00 00 00 00  00 00 00 00  00 00        ..............
```

### 2. Event Logging

Standard operational logging via `eventlog` system:
- Connection acceptance/closure
- Protocol detection results  
- Error conditions
- Debug tracing (when enabled)

### 3. Dual Protocol Support

Simultaneous TCP and UDP proxying:
- TCP for structured packets (game commands, file transfers)
- UDP for latency-sensitive data (game state updates)

### 4. Non-Blocking I/O

All sockets are set to non-blocking mode with `PSOCK_NONBLOCK`, preventing the proxy from stalling on slow connections.

### 5. Graceful Connection Handling

- Partial packet buffering
- Proper cleanup on disconnect
- Error recovery for aborted connections

## Usage

### Command Line

```bash
bnproxy [options] [servername] [serverport]
```

**Arguments**:
- `servername`: Target Battle.net server hostname (default: `localhost`)
- `serverport`: Target server TCP port (default: `6112`)

**Options**:
- `-d FILE, --hexdump=FILE`: Enable packet hex dumps to FILE
- `-l FILE, --logfile=FILE`: Write event logs to FILE
- `-p PORT, --port=PORT`: Listen on PORT for client connections (default: `6112`)
- `-f, --foreground`: Run in foreground (don't daemonize)
- `-h, --help, --usage`: Show usage information
- `-v, --version`: Print version and exit

### Examples

**Basic proxy to localhost**:
```bash
bnproxy
```

**Proxy to remote server with logging**:
```bash
bnproxy -l proxy.log -d packets.hex server.bnetd.org 6112
```

**Listen on alternate port**:
```bash
bnproxy -p 6113 -f localhost
```

**Debug mode with hex dumps**:
```bash
bnproxy -f -d /tmp/proxy.hex -l /tmp/proxy.log remote.pvpgn.pro
```

### Configuration

Client Configuration:
1. Start bnproxy on your local machine
2. Configure game client to connect to `127.0.0.1` (or proxy machine IP)
3. Proxy forwards traffic to actual server

## Dependencies

- **Common Libraries**:
  - `common/packet.h`: Packet abstraction layer
  - `common/queue.h`: FIFO packet queue implementation
  - `common/list.h`: Virtual connection list management
  - `common/eventlog.h`: Logging infrastructure
  - `compat/psock.h`: Portable socket API

- **System Libraries**:
  - POSIX sockets (`sys/socket.h`, `netinet/in.h`)
  - Host resolution (`netdb.h`)
  - Standard C library

## Building

bnproxy is built as part of the PvPGN project:

```bash
cd build
cmake ..
make bnproxy
```

The binary is typically installed to `bin/bnproxy`.

## Known Issues

### Multi-Client UDP Routing

From man page:
> Support for multiple clients is broken because the server becomes confused as to which client to relay UDP server traffic to.

**Current Behavior**: UDP packets from server are always forwarded to the first client (`list_get_first_const(virtconnlist())`).

**Impact**: Only one client can reliably use UDP features when multiple clients connect through the same proxy.

### No Dynamic UDP Address Detection

> No support for dynamic UDP address determination (SESSIONADDR[12] packets) has been added.

**Current Behavior**: UDP addresses are derived from connection parameters, not from `SERVER_SESSIONADDR1`/`SERVER_SESSIONADDR2` packets.

**Impact**: May not work correctly with complex NAT setups where UDP address differs from TCP address.

### File Transfer Handling

The file transfer class (`virtconn_class_file`) has special logic for tracking download progress:
- Reads `SERVER_FILE_REPLY.filelen` to determine total bytes
- Tracks `fileleft` counter for remaining data
- May not handle all edge cases (reconnects, resume)

## Code Structure

```
src/bnproxy/
├── bnproxy.c          (1049 lines) - Main proxy loop, connection handling
├── virtconn.c         (408 lines)  - Virtual connection management
└── virtconn.h         - Virtual connection types and prototypes
```

### Key Functions

**`bnproxy.c`**:
- `main()`: Argument parsing, initialization, daemonization
- `proxy_process()`: Main select() event loop
- `init_virtconn()`: Protocol detection and server connection
- `usage()`: Command-line help

**`virtconn.c`**:
- `virtconn_create()`: Allocate and initialize virtual connection
- `virtconn_destroy()`: Clean up connection and close sockets
- `virtconn_get/set_*()`: Accessor methods for connection properties
- `virtconnlist_*()`: Global connection list management

## Protocol Detection Algorithm

```c
if (packet_class == packet_class_bnet && 
    packet_type == CLIENT_INITCONN)
{
    virtconn_set_class(vc, virtconn_class_bnet);
}
else if (packet_class == packet_class_file && 
         packet_type == CLIENT_FILE_REQ)
{
    virtconn_set_class(vc, virtconn_class_file);
}
else
{
    virtconn_set_class(vc, virtconn_class_bot);
}
```

## Debugging

Enable detailed logging by modifying event log levels in `main()`:

```c
eventlog_add_level("debug");  // Already added if logfile specified
```

Debug messages include:
- Socket FD set membership
- Connection state transitions
- Packet queue operations
- Select() readiness checks

## Performance Considerations

- **Select-Based**: Single-threaded, limited by `FD_SETSIZE` (typically 1024)
- **Packet Copying**: All data copied through proxy buffers (not zero-copy)
- **Blocking UDP**: UDP sends use blocking loop (see `for(;;)` hacks in code)
- **Memory**: Each connection allocates 4 packet queues

For high-performance scenarios, consider:
- Switching to `epoll`/`kqueue` for large connection counts
- Implementing zero-copy forwarding
- Making UDP sends truly non-blocking

## Security Considerations

⚠️ **Warning**: bnproxy provides no authentication or encryption.

- Anyone who can connect to the listen port can proxy through your server
- All traffic is visible in plaintext (battle.net protocol is unencrypted)
- Suitable for trusted networks only (localhost, private LAN)
- Not recommended for production use without additional security measures

## Future Enhancements

Potential improvements (from code TODOs and comments):

1. **Fix Multi-Client UDP**: Implement proper UDP endpoint tracking per client
2. **SESSIONADDR Support**: Parse and use dynamic UDP addresses from server
3. **Connection Pooling**: Reuse server connections for multiple clients
4. **Statistics**: Connection counts, bytes transferred, latency measurements
5. **Access Control**: IP-based client filtering
6. **SSL/TLS**: Encrypted proxy connections (though battle.net itself is plaintext)

## Integration with PvPGN

bnproxy works seamlessly with the bnetd server:

```
[Game Client] ← Battle.net Protocol → [bnproxy] ← Battle.net Protocol → [bnetd]
    TCP/UDP                              TCP/UDP                           TCP/UDP
   (6112/6112)                         (any/any)                        (6112/6112)
```

Useful for:
- Capturing protocol traces for bnetd development
- Testing client behavior in controlled environment  
- Debugging authentication/packet handling issues
- Network simulation (add delays/drops in proxy code)

## References

- **Man Page**: `man/bnproxy.1` - Official manual page
- **Packet System**: `src/common/packet.h` - Packet abstraction used by proxy
- **Socket Compatibility**: `src/compat/psock.h` - Cross-platform socket API

## Author

Ross Combs (rocombs@cs.nmsu.edu, ross@bnetd.org)

## License

GNU General Public License v2 or later (see `LICENSE` file in project root).

---

*This README documents bnproxy as part of the PvPGN server project. Last updated: 2024*
