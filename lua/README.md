# Lua - PvPGN Scripting Engine

## Overview

The `lua` folder contains the complete Lua scripting system for PvPGN (Player vs Player Gaming Network) servers. This powerful extension framework enables server administrators to customize server behavior, add new features, implement anti-cheat systems, create interactive games, and integrate with external game hosting bots—all without recompiling the core server.

Lua scripts run within the bnetd server process and have full access to server events, user management, game control, and channel operations through a comprehensive C++ API bridge.

## Architecture

### System Design

```
┌──────────────────────────────────────────────────────────────┐
│                    PvPGN Server (bnetd)                       │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  C++ Core                         Lua VM                     │
│  ┌─────────────────────┐         ┌──────────────────────┐   │
│  │ Server Events       │────────▶│ Event Handlers       │   │
│  │  - Client Login     │         │  handle_user_login() │   │
│  │  - Game Create      │         │  handle_game_create()│   │
│  │  - Command Execute  │         │  handle_command()    │   │
│  │  - Channel Message  │         │  handle_channel_msg()│   │
│  │  - Memory Read      │         │  handle_readmemory() │   │
│  └─────────────────────┘         └──────────────────────┘   │
│           │                                │                 │
│           │                                │                 │
│  ┌─────────────────────┐         ┌──────────────────────┐   │
│  │ Lua Interface API   │◀────────│ Lua Scripts          │   │
│  │  api.message_*      │         │  - Commands          │   │
│  │  api.server_*       │         │  - AntiHack          │   │
│  │  api.game_*         │         │  - Quiz Game         │   │
│  │  api.client_*       │         │  - GHost Integration │   │
│  │  api.account_*      │         │  - Custom Logic      │   │
│  └─────────────────────┘         └──────────────────────┘   │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### Directory Structure

```
lua/
├── Core System
│   ├── main.lua                - Entry point, initialization
│   ├── config.lua              - Configuration variables
│   └── CMakeLists.txt          - Build/installation rules
│
├── Event Handlers
│   ├── handle_server.lua       - Server lifecycle events
│   ├── handle_client.lua       - Client protocol events
│   ├── handle_user.lua         - User login/disconnect/whisper
│   ├── handle_channel.lua      - Channel messages/join/leave
│   ├── handle_game.lua         - Game create/join/end/report
│   └── handle_command.lua      - Command dispatcher
│
├── Commands (command/)
│   ├── ping.lua                - /ping - GHost latency check
│   ├── w3motd.lua              - /w3motd - Warcraft 3 MOTD
│   ├── stats.lua               - /stats - Player statistics
│   └── redirect.lua            - /redirect - Server redirect
│
├── Features
│   ├── quiz/                   - Interactive quiz game system
│   │   ├── quiz.lua            - Quiz engine
│   │   ├── command.lua         - /quiz commands
│   │   ├── helper.lua          - Utilities
│   │   ├── records.lua         - High score tracking
│   │   └── questions/          - Question databases
│   │
│   ├── antihack/               - Anti-cheat system
│   │   └── starcraft.lua       - Starcraft map hack detection
│   │
│   └── ghost/                  - GHost++ bot integration
│       ├── ghost.lua           - Bot state management
│       ├── handle.lua          - Event integration
│       ├── command.lua         - /host, /swap, /start, etc.
│       ├── command_callback.lua - Bot protocol handlers
│       ├── helper.lua          - Utilities
│       └── maplist.txt         - Custom map definitions
│
├── Extensions (extend/)
│   ├── account.lua             - Account helper functions
│   ├── account_wrap.lua        - Account API wrappers
│   ├── channel.lua             - Channel utilities
│   ├── game.lua                - Game manipulation
│   ├── message.lua             - Message formatting
│   ├── eventlog.lua            - Logging helpers
│   └── enum/                   - Constant definitions
│
└── Utilities (include/)
    ├── common.lua              - Debug helpers (__FUNCTION__)
    ├── string.lua              - String manipulation
    ├── table.lua               - Table operations
    ├── math.lua                - Math utilities
    ├── file.lua                - File I/O helpers
    ├── timer.lua               - Timer system
    ├── bitwise.lua             - Bitwise operations
    └── convert.lua             - Data conversion
```

## Key Features

### 1. Event-Driven Architecture

Lua scripts receive callbacks from the C++ core when significant events occur, allowing custom logic to execute at precise moments.

**Event Categories**

```lua
-- Server Events
handle_server_mainloop()         -- Called every second
handle_server_rehash()            -- Called on Lua VM restart

-- User Events
handle_user_login(account)        -- User logs in
handle_user_disconnect(account)   -- User disconnects
handle_user_whisper(src, dst, text) -- Private message
handle_user_icon(account, iconinfo) -- Icon update

-- Channel Events
handle_channel_message(channel, account, text, type)
handle_channel_userjoin(channel, account)
handle_channel_userleft(channel, account)

-- Game Events
handle_game_create(game)          -- Game created
handle_game_userjoin(game, account) -- Player joins game
handle_game_userleft(game, account) -- Player leaves game
handle_game_end(game)             -- Game ends
handle_game_report(game)          -- Game stats reported
handle_game_destroy(game)         -- Game destroyed
handle_game_changestatus(game)    -- Game status changes
handle_game_list(account)         -- Custom game list filter

-- Client Events
handle_client_readmemory(account, request_id, data)
handle_client_extrawork(account, gametype, length, data)

-- Command Events
handle_command(account, text)     -- User executes command
handle_command_before(account, text) -- Pre-command hook
```

**Event Handler Return Values**

```lua
-- Most handlers:
return 0   -- Block default C++ behavior
return 1   -- Allow default C++ behavior to continue
return -1  -- Silently prevent execution

-- handle_command_before (flood control):
return 0   -- Apply flood protection
return 1   -- Ignore flood protection
return -1  -- Prevent command execution
```

### 2. Command System

Custom Lua commands extend server functionality without recompilation.

**Command Registration** (`handle_command.lua`)

```lua
local lua_command_table = {
    [1] = {  -- Command group 1 (all users)
        ["/w3motd"] = command_w3motd,
        ["/quiz"] = command_quiz,
        ["/ping"] = command_ping,
        ["/stats"] = command_stats,
    },
    [8] = {  -- Command group 8 (admins)
        ["/redirect"] = command_redirect,
    },
}

function handle_command(account, text)
    for cg, cmdlist in pairs(lua_command_table) do
        for cmd, func in pairs(cmdlist) do
            if string.starts(text, cmd) then
                -- Check permissions
                if math_and(account_get_auth_command_groups(account.name), cg) == 0 then
                    api.message_send_text(account.name, message_type_error, 
                        account.name, "This command is reserved for admins.")
                    return -1
                end
                return func(account, text)
            end
        end
    end
    return 1  -- Pass to C++ handler
end
```

**Command Implementation Example** (`command/w3motd.lua`)

```lua
function command_w3motd(account, text)
    -- Restrict to Warcraft 3 clients
    if not (account.clienttag == CLIENTTAG_WAR3XP or 
            account.clienttag == CLIENTTAG_WARCRAFT3) then
        return 1
    end

    username = account.name
    file_load(config.motdw3file, w3motd_sendline_callback)
    
    return 0  -- Block default behavior
end

function w3motd_sendline_callback(line)
    api.message_send_text(username, message_type_info, nil, line)
end
```

**Adding New Commands**

```lua
-- 1. Create file: command/mycommand.lua
function command_mycommand(account, text)
    local args = split_command(text, 2)  -- Split into arguments
    
    if args[1] == "help" then
        api.message_send_text(account.name, message_type_info, nil,
            "Usage: /mycommand <option>")
        return 0
    end
    
    -- Implementation
    api.message_send_text(account.name, message_type_info, nil,
        "You executed: " .. args[1])
    
    return 0
end

-- 2. Register in handle_command.lua
["/mycommand"] = command_mycommand,
```

### 3. Quiz Game System

Interactive trivia game system with scoring, streaks, and competitive rankings.

**Features**
- Multiple question databases (misc, dota, warcraft)
- Real-time hints with progressive difficulty
- Streak tracking (consecutive correct answers)
- Persistent leaderboards
- Competitive mode (points stealing)
- Configurable timing and scoring

**Configuration** (`config.lua`)

```lua
quiz = true,
quiz_filelist = "misc, dota, warcraft",
quiz_competitive_mode = true,  -- Top players lose points
quiz_max_questions = 100,      -- Questions per game
quiz_question_delay = 5,       -- Seconds between questions
quiz_hint_delay = 20,          -- Seconds between hints
quiz_users_in_top = 15,        -- Leaderboard size
```

**Commands**

```
/quiz start <name>    - Start quiz in current channel (misc/dota/warcraft)
/quiz stop            - Stop current quiz
/quiz next            - Skip to next question
/quiz records         - Show all-time leaderboard
```

**Question Format** (`quiz/questions/misc.txt`)

```
What is the capital of France?=Paris
Who painted the Mona Lisa?=Leonardo da Vinci
What year did World War II end?=1945
```

**Gameplay Flow**

```
1. User: /quiz start dota
   Bot: Quiz "dota" started!

2. Bot: [Question 1/100] Which hero has the ability "Rearm"?
   (5 second delay)

3. Bot: Hint: T _ _ _ _ r
   (20 second delay)

4. Bot: Hint: T i _ k e r
   (20 second delay)

5. User: Tinker
   Bot: Correct! Player1 answered in 32 seconds. [+10 points, 1 streak]

6. (Next question after 5 seconds)
```

**Scoring System**

```lua
-- Base points
points = 10

-- Time bonus (faster = more points)
if answer_time < 10 then
    points = points + 5
end

-- Streak multiplier
if streak > 2 then
    points = points * (1 + (streak * 0.1))
end

-- Competitive mode: 
-- Last place steals half their points from top players
```

**Question File Structure**

```
quiz/
└── questions/
    ├── misc.txt       - General trivia
    ├── dota.txt       - DotA game questions
    └── warcraft.txt   - Warcraft lore questions
```

### 4. Anti-Cheat System (AntiHack)

Memory scanning system to detect and ban map hack users in Starcraft: Brood War.

**How It Works**

```
1. Timer runs every 60 seconds (configurable)
2. Scan all active Starcraft: Brood War games
3. For each player in game:
   - Send SID_READMEMORY request
   - Target: 0x0047FD12 (map visibility flag)
   - Expected value: 139 (normal)
   
4. Client responds with memory contents
5. If value != 139:
   - Lock account
   - Notify all players in game
   - Kick cheater
   - Log incident
```

**Configuration** (`config.lua`)

```lua
ah = true,                  -- Enable antihack
ah_interval = 60,           -- Check every 60 seconds
```

**Implementation** (`antihack/starcraft.lua`)

```lua
-- Detection parameters
ah_mh_request_id = 99              -- Request identifier
ah_mh_offset = 0x0047FD12          -- Memory offset (map visible)
ah_mh_value = 139                  -- Normal value (no hack)

function ah_timer_tick(options)
    -- Iterate all games
    for i, game in pairs(api.server_get_games()) do
        if game.clienttag == CLIENTTAG_BROODWARS then
            if substr_count(game.players, ",") > -1 then
                -- Check each player
                for username in string.split(game.players, ",") do
                    api.client_readmemory(username, ah_mh_request_id, 
                                          ah_mh_offset, 2)
                end
            end
        end
    end
end

function ah_handle_client(account, request_id, data)
    if request_id == ah_mh_request_id then
        local value = bytes_to_int(data, 0, 2)
        
        if value ~= ah_mh_value then
            -- Ban cheater
            account_set_auth_lock(account.name, true)
            account_set_auth_lockreason(account.name, 
                "we do not like cheaters")
            
            -- Notify game
            local game = api.game_get_by_id(account.game_id)
            for username in string.split(game.players, ",") do
                api.message_send_text(username, message_type_error, nil,
                    account.name .. " was banned by the antihack system.")
            end
            
            -- Kick
            api.client_kill(account.name)
            INFO(account.name .. " was banned by the antihack system.")
        end
    end
end
```

**Adding More Checks**

```lua
-- Define new check
ah_speedhack_request_id = 100
ah_speedhack_offset = 0x00XXXXXX
ah_speedhack_value = YYY

-- Add to timer
function ah_timer_tick(options)
    for username in string.split(game.players, ",") do
        api.client_readmemory(username, ah_mh_request_id, 
                              ah_mh_offset, 2)
        api.client_readmemory(username, ah_speedhack_request_id,
                              ah_speedhack_offset, 4)
    end
end

-- Handle response
function ah_handle_client(account, request_id, data)
    if request_id == ah_mh_request_id then
        -- Map hack check
    elseif request_id == ah_speedhack_request_id then
        -- Speed hack check
        local value = bytes_to_int(data, 0, 4)
        if value ~= ah_speedhack_value then
            is_cheater = true
        end
    end
end
```

### 5. GHost++ Integration

Seamless integration with GHost++ bot for automated Warcraft 3 game hosting.

**Features**
- Custom game commands (/host, /swap, /start, /abort)
- Player ping optimization
- DotA statistics tracking
- Map list customization
- Bot user management
- Persistent state saving

**Configuration** (`config.lua`)

```lua
ghost = true,                       -- Enable GHost commands
ghost_bots = { "hostbot1", "hostbot2" },  -- Authorized bots
ghost_dota_server = true,           -- Use DotA stats
ghost_ping_expire = 90,             -- Ping cache timeout
```

**Commands**

```
/host <map> [gamename]   - Host game (admin only)
/chost <mapcode>         - Host from custom maplist
/unhost                  - Close current game
/ping                    - Show latency to all bots
/swap <player1> <player2> - Swap team slots
/open <slot>             - Open slot
/close <slot>            - Close slot
/start                   - Start game
/abort                   - Cancel game
/pub                     - Make game public
/priv                    - Make game private
/stats <player>          - Show player stats
```

**Custom Map List** (`ghost/maplist.txt`)

```
# Format: code | map name | map filename
dota | DotA v6.88 | Maps\Download\DotA v6.88.w3x
td | Element TD | Maps\Download\Element TD 5.0.w3x
footmen | Footmen Frenzy | Maps\Download\Footmen Frenzy v5.5.w3x
```

**Usage Example**

```
User: /ping
Bot: Your latency to bots:
Bot:   hostbot1: 45ms (Los Angeles)
Bot:   hostbot2: 120ms (New York)

User: /chost dota
Bot: Hosting DotA v6.88 on hostbot1 (best ping: 45ms)

User: /swap 1 2
Bot: Swapped slot 1 with slot 2

User: /start
Bot: Starting game in 10 seconds...
```

**Ping System**

```lua
-- Store ping data per user
account_set_botping(username, "hostbot1", 45)
account_set_botping(username, "hostbot2", 120)

-- Retrieve pings
local pings = account_get_botping(username)
-- Returns: { hostbot1 = 45, hostbot2 = 120 }

-- Sort game list by best ping
function handle_game_list(account)
    local glist = server_get_games()
    local pings = account_get_botping(account.name)
    
    for i, game in pairs(glist) do
        if gh_is_bot(game.owner) then
            glist[i].ping = pings[game.owner] or 1000
        end
    end
    
    -- Client shows games sorted by lowest ping
    return glist
end
```

**State Persistence**

```lua
-- On server start
function gh_load()
    local filename = config.vardir() .. "ghost_users.dmp"
    if file_exists(filename) then
        gh_load_userbots(filename)  -- Restore bot/user associations
    end
end

-- On server shutdown or /rehash
function gh_unload()
    local filename = config.vardir() .. "ghost_users.dmp"
    gh_save_userbots(filename)  -- Save current state
end
```

### 6. Timer System

Flexible timer system for scheduled/recurring tasks.

**Timer API** (`include/timer.lua`)

```lua
-- Create timer
timer_add(name, interval, callback, options)

-- Delete timer
timer_del(name)

-- Timer automatically ticks in handle_server_mainloop()
```

**Examples**

```lua
-- One-time timer (5 seconds)
timer_add("quiz_hint", 5, function()
    channel_send_message(config.quiz_channel, "Hint: " .. hint)
end)

-- Recurring timer (every 60 seconds)
timer_add("ah_timer", 60, ah_timer_tick, { recurring = true })

-- Timer with custom data
timer_add("announce", 300, function(options)
    for username in pairs(api.server_get_users()) do
        api.message_send_text(username, message_type_info, nil,
            options.message)
    end
end, { recurring = true, message = "Server restart in 10 minutes" })

-- Cancel timer
timer_del("quiz_hint")
```

### 7. Lua API Reference

**Message Functions**

```lua
-- Send message to user
api.message_send_text(username, message_type, from, text)
    -- message_type: message_type_info, message_type_error, 
    --               message_type_whisper, message_type_emote

-- Send to channel
channel_send_message(channelname, text, message_type)
```

**Account Functions**

```lua
-- Get account info
local account = api.account_get(username)
-- Returns: { name, clienttag, archtag, game_id, channel, ... }

-- Authentication
account_get_auth_command_groups(username)  -- Command permissions
account_set_auth_lock(username, locked)    -- Lock/unlock account
account_set_auth_lockreason(username, reason)

-- Custom attributes
account_set_botping(username, botname, ping_ms)
local pings = account_get_botping(username)
```

**Server Functions**

```lua
-- Get all users
for i, account in pairs(api.server_get_users()) do
    print(account.name)
end

-- Get all games
for i, game in pairs(api.server_get_games()) do
    print(game.name)
end

-- Get specific game
local game = api.game_get_by_id(game_id)
```

**Client Functions**

```lua
-- Read client memory
api.client_readmemory(username, request_id, offset, length)

-- Send extra work request
api.client_requiredwork(username, "IX86ExtraWork.mpq")

-- Disconnect user
api.client_kill(username)
```

**Game Functions**

```lua
-- Game object properties
game.id              -- Unique game ID
game.name            -- Game name
game.owner           -- Owner username
game.type            -- Game type
game.status          -- Game status
game.clienttag       -- Game client tag
game.players         -- Comma-separated player list
game.mapname         -- Map filename
```

**File Functions**

```lua
-- Check file exists
if file_exists(filename) then ... end

-- Read file line by line
file_load(filename, process_callback, line_callback)

-- Callback examples
function process_callback(line)
    -- Return processed line
    return string.upper(line)
end

function line_callback(processed_line)
    -- Use processed line
    print(processed_line)
end
```

**Utility Functions**

```lua
-- String
string.starts(text, prefix)
string.ends(text, suffix)
string.split(text, delimiter)
string:trim()
string:empty()

-- Table
table.clear(t)
table.copy(t)
table.count(t)

-- Math
math_and(a, b)   -- Bitwise AND
math_or(a, b)    -- Bitwise OR

-- Logging
DEBUG(message)   -- Debug level
INFO(message)    -- Info level
WARN(message)    -- Warning level
ERROR(message)   -- Error level
TRACE(message)   -- Trace level
```

**Constants**

```lua
-- Client tags
CLIENTTAG_STARCRAFT
CLIENTTAG_BROODWARS
CLIENTTAG_WARCRAFT2
CLIENTTAG_DIABLO2DV
CLIENTTAG_DIABLO2XP
CLIENTTAG_STARJAPAN
CLIENTTAG_WARCRAFT3
CLIENTTAG_WAR3XP

-- Architecture tags
ARCHTAG_WINX86
ARCHTAG_MACX86
ARCHTAG_MACPPC

-- Message types
message_type_info
message_type_error
message_type_whisper
message_type_emote
message_type_talk
```

## Configuration

### config.lua

Central configuration file with customizable settings.

```lua
config = {
    -- Paths
    vardir = function()
        return string.replace(config.statusdir, "status", "")
    end,
    
    -- Flood protection exemptions
    flood_immunity_users = { "admin", "moderator" },
    
    -- Quiz settings
    quiz = true,
    quiz_filelist = "misc, dota, warcraft",
    quiz_competitive_mode = true,
    quiz_max_questions = 100,
    quiz_question_delay = 5,
    quiz_hint_delay = 20,
    quiz_users_in_top = 15,
    
    -- AntiHack settings
    ah = true,
    ah_interval = 60,
    
    -- GHost settings
    ghost = false,
    ghost_bots = { "hostbot1" },
    ghost_dota_server = true,
    ghost_ping_expire = 90,
}
```

### bnetd.conf Integration

```ini
# Enable Lua scripting
luafile = "lua/main.lua"

# Lua automatically reads these config values:
statusdir = "/var/pvpgn/status"
motdw3file = "/etc/pvpgn/w3motd.txt"
```

## Installation

### Prerequisites

PvPGN must be compiled with Lua support enabled.

```bash
# Check if Lua is enabled
bnetd --version | grep Lua

# If not enabled, recompile with Lua
cd pvpgn-server/build
cmake -DWITH_LUA=ON ..
make && sudo make install
```

### Deployment

```bash
# Lua files are installed automatically with PvPGN
# Default location: /usr/local/var/lua

# Verify installation
ls -la /usr/local/var/lua
# Should show: main.lua, config.lua, handle_*.lua, etc.

# Enable in bnetd.conf
sudo nano /etc/pvpgn/bnetd.conf
# Add: luafile = "lua/main.lua"

# Restart server
sudo service bnetd restart

# Check logs for Lua initialization
tail -f /var/log/bnetd.log
# Should see: Starcraft Antihack activated
#             Loaded GHost state from ...
```

### Customization

```bash
# Edit configuration
sudo nano /usr/local/var/lua/config.lua

# Modify settings
config.quiz = true
config.ah = true
config.ghost = false

# Reload Lua VM without restarting server
bnchat localhost
/admin
/rehash

# Or restart server
sudo service bnetd restart
```

## Usage Examples

### Adding a Custom Command

**1. Create command file** (`command/uptime.lua`)

```lua
function command_uptime(account, text)
    local uptime = os.time() - server_start_time
    local days = math.floor(uptime / 86400)
    local hours = math.floor((uptime % 86400) / 3600)
    local mins = math.floor((uptime % 3600) / 60)
    
    local msg = string.format("Server uptime: %dd %dh %dm", days, hours, mins)
    api.message_send_text(account.name, message_type_info, nil, msg)
    
    return 0
end
```

**2. Register command** (`handle_command.lua`)

```lua
local lua_command_table = {
    [1] = {
        ["/uptime"] = command_uptime,  -- Add this line
        ["/w3motd"] = command_w3motd,
        -- ... other commands
    },
}
```

**3. Reload Lua**

```bash
/admin
/rehash
```

**4. Test**

```
User: /uptime
Bot: Server uptime: 5d 12h 34m
```

### Creating a Welcome System

**Edit** `handle_user.lua`:

```lua
function handle_user_login(account)
    -- Send welcome message
    api.message_send_text(account.name, message_type_info, nil,
        "Welcome to our server, " .. account.name .. "!")
    
    -- Check if first login
    local login_count = account_get_login_count(account.name)
    if login_count == 1 then
        api.message_send_text(account.name, message_type_info, nil,
            "This is your first time here! Type /help for commands.")
        
        -- Grant bonus
        account_set_normal_wins(account.name, 
            account_get_normal_wins(account.name) + 10)
    end
    
    -- Announce to admins
    for username in pairs(api.server_get_users()) do
        if account_get_auth_admin(username) then
            api.message_send_text(username, message_type_whisper, nil,
                account.name .. " has logged in.")
        end
    end
end
```

### Implementing Auto-Announce

**Add to** `handle_server.lua`:

```lua
local announce_counter = 0
local announce_interval = 300  -- 5 minutes
local announcements = {
    "Visit our website at www.example.com",
    "Join our Discord server!",
    "Type /help for available commands",
    "Support the server by voting on TopG!",
}

function handle_server_mainloop()
    -- Tick timers
    for t in pairs(__timers) do
        __timers[t]:tick()
    end
    
    -- Auto-announce
    announce_counter = announce_counter + 1
    if announce_counter >= announce_interval then
        announce_counter = 0
        
        -- Random announcement
        local msg = announcements[math.random(#announcements)]
        
        -- Send to all users
        for i, account in pairs(api.server_get_users()) do
            api.message_send_text(account.name, message_type_info, nil, msg)
        end
    end
end
```

### Channel-Specific Quiz

**Restrict quiz to specific channels** (`quiz/command.lua`):

```lua
function command_quiz(account, text)
    local args = split_command(text, 2)
    local channel = api.channel_get_by_name(account.channel)
    
    -- Only allow in Quiz channel
    if channel.name ~= "Quiz" then
        api.message_send_text(account.name, message_type_error, nil,
            "Quiz commands only work in the Quiz channel!")
        return 0
    end
    
    if args[1] == "start" then
        if not args[2] then
            api.message_send_text(account.name, message_type_error, nil,
                "Usage: /quiz start <" .. config.quiz_filelist .. ">")
            return 0
        end
        quiz:start(channel.name, args[2])
    elseif args[1] == "stop" then
        quiz:stop(account.name)
    end
    
    return 0
end
```

## Development

### Adding New Event Handler

**1. Identify C++ hook point** (check `src/bnetd/luainterface.cpp`)

```cpp
// Example: New packet handler
extern void lua_handle_client_newpacket(t_connection* c, int value) {
    lua_State* L = get_lua_state();
    lua_getglobal(L, "handle_client_newpacket");
    
    // Push account table
    push_account_table(L, c);
    lua_pushinteger(L, value);
    
    // Call Lua function
    lua_pcall(L, 2, 0, 0);
}
```

**2. Create Lua handler** (`handle_client.lua`)

```lua
function handle_client_newpacket(account, value)
    DEBUG(account.name .. " sent packet with value: " .. value)
    
    -- Process packet
    if value > 100 then
        api.client_kill(account.name)
    end
end
```

**3. Recompile server**

```bash
cd pvpgn-server/build
make && sudo make install
sudo service bnetd restart
```

### Debugging

**Enable debug logging** (`config.lua`):

```lua
-- Use DEBUG() function
DEBUG("This is a debug message")
INFO("This is an info message")
WARN("This is a warning")
ERROR("This is an error")
TRACE("Detailed trace data")
```

**Check logs**:

```bash
tail -f /var/log/bnetd.log | grep "lua"
```

**Lua error messages**:

```
[error] lua_handle_command: attempt to call a nil value (field 'message_send_text')
[error] lua_handle_command: main.lua:45: syntax error near 'end'
```

### Best Practices

**Error Handling**

```lua
function command_safe(account, text)
    local success, result = pcall(function()
        -- Risky operation
        local data = api.dangerous_operation()
        return data
    end)
    
    if not success then
        ERROR("Command failed: " .. result)
        api.message_send_text(account.name, message_type_error, nil,
            "An error occurred. Contact an administrator.")
        return 0
    end
    
    -- Use result
    return 0
end
```

**Performance**

```lua
-- Cache frequently accessed data
local user_cache = {}

function get_cached_user(username)
    if not user_cache[username] then
        user_cache[username] = api.account_get(username)
    end
    return user_cache[username]
end

-- Clear cache periodically
function handle_server_mainloop()
    if os.time() % 300 == 0 then  -- Every 5 minutes
        user_cache = {}
    end
end
```

**Memory Management**

```lua
-- Clean up large tables
function quiz:stop()
    table.clear(q_dictionary)
    table.clear(q_records_current)
    collectgarbage("collect")
end
```

## Troubleshooting

### Lua VM Won't Start

```bash
# Check bnetd.conf
grep luafile /etc/pvpgn/bnetd.conf
# Should be: luafile = "lua/main.lua"

# Check file exists
ls -la /usr/local/var/lua/main.lua

# Check Lua compilation support
bnetd --version | grep Lua
# Should see: WITH_LUA

# Check logs
tail -50 /var/log/bnetd.log | grep -i lua
```

### Syntax Errors

```
[error] lua_handle_command: main.lua:45: syntax error near 'end'

# Fix: Check Lua syntax
lua -l main.lua

# Or use luac
luac -p main.lua
```

### Function Not Found

```
[error] attempt to call a nil value (field 'command_uptime')

# Causes:
# 1. Function not defined
# 2. Not registered in command table
# 3. Typo in function name

# Fix: Verify function exists and is registered
grep -r "function command_uptime" lua/
```

### API Function Errors

```
[error] api.message_send_text: invalid username

# Common issues:
# - Username doesn't exist (user logged out)
# - Username is nil (missing parameter)

# Fix: Check username validity
if account and account.name then
    api.message_send_text(account.name, ...)
end
```

### Memory Leaks

```bash
# Monitor Lua memory usage
watch -n 1 'ps aux | grep bnetd'

# If memory grows continuously:
# 1. Add collectgarbage() calls
# 2. Clear unused tables
# 3. Remove global variables after use
```

### Command Not Executing

```
# Check command registration
grep "/mycommand" lua/handle_command.lua

# Check permissions
# Command group 8 requires admin

# Test with simple command
["/test"] = function(account, text)
    api.message_send_text(account.name, message_type_info, nil, "Works!")
    return 0
end
```

## Related Documentation

- **C++ API Bridge**: `src/bnetd/luainterface.cpp`, `src/bnetd/luainterface.h`
- **Configuration**: `conf/bnetd.conf.in`
- **Lua 5.1 Manual**: https://www.lua.org/manual/5.1/
- **PvPGN Wiki**: https://pvpgn.pro/pvpgn_wiki

## Contributing

### Guidelines

1. **Naming Conventions**
   - Functions: `snake_case`
   - Constants: `UPPER_CASE`
   - Private functions: `_prefix`

2. **Code Style**
   - Indent: 4 spaces (tabs)
   - Comments: `-- Single line` or `--[[ Multi-line ]]--`

3. **Documentation**
   - Add function descriptions
   - Include usage examples
   - Document return values

4. **Testing**
   - Test with live server
   - Check for memory leaks
   - Verify error handling

### Submitting Scripts

1. Create feature in new file (e.g., `command/newfeature.lua`)
2. Update `config.lua` if configuration needed
3. Add documentation to this README
4. Test thoroughly
5. Submit pull request to PvPGN repository

## License

Licensed under the same terms as Lua itself (MIT License).

Copyright (C) 2014-2025 HarpyWar (harpywar@gmail.com) and PvPGN Project contributors.

---

**Last Updated**: November 24, 2025  
**PvPGN Version**: Latest development branch  
**Lua Version**: 5.1.x
