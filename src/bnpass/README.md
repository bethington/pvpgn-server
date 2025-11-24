# BNPASS - Battle.net Password Utilities

## Overview

`bnpass` is a collection of command-line utilities for generating password hashes compatible with PvPGN's account authentication system. These tools convert cleartext passwords into hashed formats used by the Battle.net daemon (bnetd) for secure password storage and verification.

## Purpose

PvPGN never stores passwords in plaintext. Instead, it stores cryptographic hashes that can be verified during login but cannot be reversed to obtain the original password. The bnpass utilities enable:

- **Account Creation**: Generate password hashes for manual account creation
- **Password Resets**: Reset forgotten passwords by generating new hashes
- **Batch Operations**: Script automated account creation and management
- **Testing**: Generate test accounts for development and debugging
- **Migration**: Convert passwords when migrating from other systems

## Security Context

### Why Hash Passwords?

1. **Protection Against Breaches**: If the account database is compromised, attackers cannot directly read passwords
2. **One-Way Function**: Hashes cannot be reversed to obtain original passwords
3. **Verification Without Storage**: Server can verify login attempts without knowing the actual password
4. **Salted Hashes**: Modern hashing prevents rainbow table attacks

### Hash Algorithms Used

PvPGN uses two primary hashing algorithms:

1. **BNET Hash** (Blizzard's variant of SHA-1)
   - Used for older clients (StarCraft, Diablo, Warcraft II)
   - Modified SHA-1 with different bit rotation
   - Produces 160-bit (20-byte) hash
   - Stored as 40-character hex string

2. **Standard SHA-1**
   - Used for some authentication scenarios
   - Standard SHA-1 algorithm (RFC 3174)
   - Also produces 160-bit hash
   - Compatible with standard SHA-1 implementations

## Utilities

### 1. bnpass

**Purpose**: Generate BNET password hashes for account creation and management

**Usage**:
```bash
bnpass [options] [cleartext_password]
```

**Description**:
Converts a cleartext password to BNET hash format used in PvPGN account files. The hash is output in the format used by the account storage system.

**Input Methods**:

**Interactive** (secure - password not visible in shell history):
```bash
$ bnpass
Enter password to hash: [type password]
"BNET\\acct\\passhash1"="c3a733e547e41c5f2c7812177a5e8c783f4a6f3a"
```

**Command-line** (convenient but less secure):
```bash
$ bnpass mypassword
"BNET\\acct\\passhash1"="c3a733e547e41c5f2c7812177a5e8c783f4a6f3a"
```

**Output Format**:
```
"BNET\\acct\\passhash1"="<40-character-hex-hash>"
```

This format matches the storage attribute used in:
- File-based accounts: `users/<username>/BNET/acct/passhash1`
- SQL accounts: Column in account attributes table
- Plain text accounts: Field in account file

**Password Normalization**:
- Password is automatically converted to lowercase
- This matches Blizzard's behavior for case-insensitive authentication
- Maximum password length: 255 characters

**Options**:
- `-h, --help, --usage`: Show usage information and exit
- `-v, --version`: Print version number and exit
- `--`: End of options (allows password starting with `-`)

**Examples**:

```bash
# Interactive password entry (most secure)
$ bnpass
Enter password to hash: 
"BNET\\acct\\passhash1"="e5e9fa1ba31ecd1ae84f75caaa474f3a663f05f4"

# Command-line password
$ bnpass password123
"BNET\\acct\\passhash1"="5baa61e4c9b93f3f0682250b6cf8331b7ee68fd8"

# Password starting with dash (use --)
$ bnpass -- -mypassword
"BNET\\acct\\passhash1"="d167b5b9c2e2dbf16c23cc2f0c2cf689e9e2e836"

# Capture hash in variable
HASH=$(bnpass mypass | cut -d'"' -f4)
echo "Hash: $HASH"
```

**Use Cases**:

1. **Manual Account Creation**:
```bash
# Generate hash
HASH=$(bnpass secretpass | cut -d'"' -f4)

# Create account file (file storage)
mkdir -p users/john
echo "$HASH" > users/john/BNET/acct/passhash1

# Or insert into SQL
mysql -e "INSERT INTO accounts VALUES ('john', '$HASH', ...)"
```

2. **Password Reset Script**:
```bash
#!/bin/bash
USERNAME="$1"
PASSWORD="$2"

HASH=$(bnpass "$PASSWORD" | cut -d'"' -f4)

# Update file storage
echo "$HASH" > "users/$USERNAME/BNET/acct/passhash1"

echo "Password reset for $USERNAME"
```

3. **Batch Account Creation**:
```bash
#!/bin/bash
# Create 100 test accounts
for i in {1..100}; do
    USERNAME="test$i"
    PASSWORD="pass$i"
    HASH=$(bnpass "$PASSWORD" | cut -d'"' -f4)
    
    mkdir -p "users/$USERNAME/BNET/acct"
    echo "$HASH" > "users/$USERNAME/BNET/acct/passhash1"
    echo "0" > "users/$USERNAME/BNET/acct/userid"
done
```

---

### 2. sha1hash

**Purpose**: Generate standard SHA-1 hashes

**Usage**:
```bash
sha1hash [options] [cleartext_password]
```

**Description**:
Generates standard SHA-1 hashes (not the BNET variant). Useful for:
- Non-Battle.net authentication scenarios
- Testing and verification
- Comparing with standard SHA-1 implementations
- General-purpose hashing needs

**Input/Output**:

```bash
$ sha1hash
Enter password to hash: [type password]
sha1 hash = 5baa61e4c9b93f3f0682250b6cf8331b7ee68fd8

$ sha1hash password
sha1 hash = 5baa61e4c9b93f3f0682250b6cf8331b7ee68fd8
```

**Differences from bnpass**:
- Uses standard SHA-1 (not BNET variant)
- Output format is simpler (no attribute path)
- No lowercase conversion (hashes input as-is)
- Useful for debugging and comparison

**Options**:
- `-h, --help, --usage`: Show usage information and exit
- `-v, --version`: Print version number and exit
- `--`: End of options marker

**Examples**:

```bash
# Compare BNET hash vs standard SHA-1
$ echo "Testing BNET hash:"
$ bnpass test
"BNET\\acct\\passhash1"="a94a8fe5ccb19ba61c4c0873d391e987982fbbd3"

$ echo "Testing standard SHA-1:"
$ sha1hash test
sha1 hash = a94a8fe5ccb19ba61c4c0873d391e987982fbbd3

# In this case they're the same, but for longer inputs they differ
```

**Use Cases**:

1. **Verification Against Standard Tools**:
```bash
# Compare with openssl
$ echo -n "password" | openssl sha1
(stdin)= 5baa61e4c9b93f3f0682250b6cf8331b7ee68fd8

$ sha1hash password
sha1 hash = 5baa61e4c9b93f3f0682250b6cf8331b7ee68fd8
```

2. **General Hashing**:
```bash
# Hash any data for integrity checks
DATA="important data"
HASH=$(sha1hash "$DATA" | awk '{print $4}')
echo "Data hash: $HASH"
```

3. **Testing Compatibility**:
```bash
# Test PvPGN's SHA-1 implementation
for word in password test admin; do
    echo "Testing: $word"
    sha1hash "$word"
done
```

---

## Architecture

### Code Structure

```
src/bnpass/
├── bnpass.cpp      # BNET password hash utility
├── sha1hash.cpp    # Standard SHA-1 hash utility
└── CMakeLists.txt  # Build configuration
```

Both utilities share common code from `src/common/`:
- `bnethash.cpp/h` - Hash algorithm implementations
- `eventlog.cpp/h` - Logging infrastructure
- `version.h` - Version information
- `xstring.cpp/h` - String manipulation utilities

### Hash Implementation

Located in `src/common/bnethash.cpp`:

**Functions**:
```cpp
// BNET hash (Blizzard's SHA-1 variant)
int bnet_hash(t_hash * hashout, unsigned int size, void const * data);

// Standard SHA-1
int sha1_hash(t_hash * hashout, unsigned int size, void const * data);

// Little-endian SHA-1 variant
int little_endian_sha1_hash(t_hash * hashout, unsigned int size, void const * data);

// Hash comparison
int hash_eq(t_hash const h1, t_hash const h2);

// Hash string conversion
char const * hash_get_str(t_hash const hash);
char const * little_endian_hash_get_str(t_hash const hash);
int hash_set_str(t_hash * hash, char const * str);
```

**Hash Type**:
```cpp
typedef uint32_t t_hash[5];  // 5 x 32-bit = 160 bits
```

### Algorithm Details

#### BNET Hash (Modified SHA-1)

Blizzard's variant differs from standard SHA-1 in the message schedule:

**Standard SHA-1**:
```
W[i] = ROTL(W[i-3] ^ W[i-8] ^ W[i-14] ^ W[i-16], 1)
```

**BNET Hash**:
```
W[i] = ROTL(W[i-13] ^ W[i-8] ^ W[i-2] ^ W[i], 1)
```

This produces different hash values for the same input, making it incompatible with standard SHA-1.

#### Password Processing

1. **Input**: Cleartext password
2. **Normalization**: Convert to lowercase (bnpass only)
3. **Hashing**: Apply BNET hash or SHA-1 algorithm
4. **Output**: 160-bit hash as 40-character hex string

### Integration with PvPGN

#### Account Storage

Password hashes are stored in account attributes:

**File Storage** (`storage_file.cpp`):
```
users/
└── username/
    └── BNET/
        └── acct/
            ├── passhash1    # BNET hash
            ├── salt         # For SRP3 (Warcraft 3)
            └── verifier     # For SRP3 (Warcraft 3)
```

**SQL Storage** (`storage_sql.cpp`):
```sql
CREATE TABLE bnet (
    uid INT PRIMARY KEY,
    acct_username VARCHAR(128),
    acct_passhash1 VARCHAR(40),  -- BNET hash
    acct_salt BLOB,              -- SRP3 salt
    acct_verifier BLOB,          -- SRP3 verifier
    ...
);
```

#### Authentication Flow

1. **Client Login Attempt**:
   - User enters password in game client
   - Client hashes password locally
   - Sends hashed password to server

2. **Server Verification** (older clients):
   - Server retrieves stored hash from account
   - Performs double-hash with session key
   - Compares with client's submitted hash
   - Grants access if match

3. **SRP3 Authentication** (Warcraft 3):
   - Uses Secure Remote Password protocol
   - Requires salt and verifier (not just hash)
   - More secure, prevents replay attacks

#### Hash Usage in Code

**Account Creation** (`account.cpp`):
```cpp
t_account * account_create(const char * username, const char * passhash1)
{
    // ...
    account_set_strattr(account, "BNET\\acct\\passhash1", passhash1);
    // ...
}
```

**Login Verification** (`handle_bnet.cpp`):
```cpp
// Retrieve stored hash
const char * oldstrhash1 = account_get_pass(account);
hash_set_str(&oldpasshash1, oldstrhash1);

// Double-hash with session data
bnet_hash(&oldpasshash2, sizeof(temp), &temp);

// Compare with client submission
if (hash_eq(trypasshash2, oldpasshash2)) {
    // Login successful
}
```

**Password Change** (`handle_bnet.cpp`):
```cpp
t_hash newpasshash1;
bnhash_to_hash(packet->u.client_changepassreq.newpassword_hash1, &newpasshash1);
account_set_pass(account, hash_get_str(newpasshash1));
```

**Command-line Account Creation** (`command.cpp`):
```cpp
// /addacct command handler
t_hash passhash;
bnet_hash(&passhash, strlen(pass), pass);
accountlist_create_account(username, hash_get_str(passhash));
```

---

## Building

### CMake Configuration

```cmake
add_executable(bnpass bnpass.cpp)
target_link_libraries(bnpass PRIVATE common compat fmt)

add_executable(sha1hash sha1hash.cpp)
target_link_libraries(sha1hash PRIVATE common compat fmt)

install(TARGETS bnpass sha1hash DESTINATION ${BINDIR})
```

### Dependencies

- **common**: PvPGN common library (hash functions, logging, etc.)
- **compat**: Portability layer for cross-platform support
- **fmt**: Modern C++ formatting library

### Compilation

```bash
# From project root
cmake -B build
cmake --build build --target bnpass sha1hash

# Binaries in: build/src/bnpass/
```

### Installation

```bash
# Install to system
sudo cmake --install build

# Binaries installed to: /usr/local/bin/ (or configured prefix)
```

---

## Common Workflows

### Resetting a User's Password

```bash
#!/bin/bash
# reset_password.sh - Reset a user's password

USERNAME="$1"
NEW_PASSWORD="$2"

if [ -z "$USERNAME" ] || [ -z "$NEW_PASSWORD" ]; then
    echo "Usage: $0 <username> <new_password>"
    exit 1
fi

# Generate hash
HASH=$(bnpass "$NEW_PASSWORD" | cut -d'"' -f4)

# Update based on storage type
if [ -d "users/$USERNAME" ]; then
    # File storage
    echo "$HASH" > "users/$USERNAME/BNET/acct/passhash1"
    echo "Password reset for $USERNAME (file storage)"
elif command -v mysql &> /dev/null; then
    # SQL storage (MySQL example)
    mysql pvpgn -e "UPDATE bnet SET acct_passhash1='$HASH' WHERE acct_username='$USERNAME'"
    echo "Password reset for $USERNAME (SQL storage)"
else
    echo "Error: Could not determine storage type"
    exit 1
fi
```

### Creating Admin Accounts

```bash
#!/bin/bash
# create_admin.sh - Create admin account with special permissions

USERNAME="$1"
PASSWORD="$2"

HASH=$(bnpass "$PASSWORD" | cut -d'"' -f4)

# Create account directory
mkdir -p "users/$USERNAME/BNET/acct"
mkdir -p "users/$USERNAME/BNET/auth"

# Set password
echo "$HASH" > "users/$USERNAME/BNET/acct/passhash1"

# Set admin permissions
echo "1" > "users/$USERNAME/BNET/auth/admin"
echo "1" > "users/$USERNAME/BNET/auth/command"

# Set other attributes
echo "$USERNAME" > "users/$USERNAME/BNET/acct/username"
echo "0" > "users/$USERNAME/BNET/acct/userid"

echo "Admin account created: $USERNAME"
```

### Migrating Passwords from Another System

```bash
#!/bin/bash
# migrate_accounts.sh - Migrate from plaintext password list

INPUT_FILE="passwords.txt"  # Format: username:password

while IFS=: read -r username password; do
    echo "Migrating: $username"
    
    # Generate BNET hash
    HASH=$(bnpass "$password" | cut -d'"' -f4)
    
    # Create account structure
    mkdir -p "users/$username/BNET/acct"
    echo "$HASH" > "users/$username/BNET/acct/passhash1"
    echo "$username" > "users/$username/BNET/acct/username"
    
done < "$INPUT_FILE"

echo "Migration complete!"
```

### Bulk Account Creation for Testing

```bash
#!/bin/bash
# create_test_accounts.sh - Create multiple test accounts

PREFIX="testuser"
COUNT=50
BASE_PASSWORD="testpass"

for i in $(seq 1 $COUNT); do
    USERNAME="${PREFIX}${i}"
    PASSWORD="${BASE_PASSWORD}${i}"
    
    echo "Creating: $USERNAME"
    
    HASH=$(bnpass "$PASSWORD" | cut -d'"' -f4)
    
    mkdir -p "users/$USERNAME/BNET/acct"
    echo "$HASH" > "users/$USERNAME/BNET/acct/passhash1"
    echo "$USERNAME" > "users/$USERNAME/BNET/acct/username"
    echo "$i" > "users/$USERNAME/BNET/acct/userid"
done

echo "$COUNT test accounts created"
```

### Verifying Password Hashes

```bash
#!/bin/bash
# verify_password.sh - Check if password matches stored hash

USERNAME="$1"
PASSWORD="$2"

# Get stored hash
if [ -f "users/$USERNAME/BNET/acct/passhash1" ]; then
    STORED_HASH=$(cat "users/$USERNAME/BNET/acct/passhash1")
else
    echo "Account not found: $USERNAME"
    exit 1
fi

# Generate hash from password
GENERATED_HASH=$(bnpass "$PASSWORD" | cut -d'"' -f4)

# Compare
if [ "$STORED_HASH" = "$GENERATED_HASH" ]; then
    echo "✓ Password matches for $USERNAME"
    exit 0
else
    echo "✗ Password does NOT match for $USERNAME"
    echo "Stored:    $STORED_HASH"
    echo "Generated: $GENERATED_HASH"
    exit 1
fi
```

---

## Security Considerations

### Strengths

1. **One-Way Function**: Cannot reverse hash to get password
2. **Deterministic**: Same password always produces same hash (for verification)
3. **Fixed Length**: 160-bit output regardless of input length
4. **Collision Resistance**: Difficult to find two inputs with same hash

### Weaknesses

1. **No Salt in Basic BNET Hash**: Rainbow table attacks possible
   - Modern PvPGN uses SRP3 for Warcraft 3 (includes salt)
   - Older games still use unsalted BNET hash

2. **SHA-1 Deprecation**: SHA-1 is considered weak by modern standards
   - Collision attacks demonstrated
   - Not suitable for new cryptographic applications
   - PvPGN maintains it for Blizzard client compatibility

3. **Dictionary Attacks**: Weak passwords still vulnerable
   - Hash doesn't prevent brute-force on weak passwords
   - Server should enforce password complexity

4. **Command-line Exposure**: Password visible in shell history
   - Use interactive mode when possible
   - Clear shell history after using command-line mode

### Best Practices

1. **Use Interactive Mode**:
```bash
# Secure (password not in history)
$ bnpass
Enter password to hash: 

# Less secure (password in shell history)
$ bnpass mypassword
```

2. **Clear Shell History**:
```bash
# After using command-line mode
history -c
# or
rm ~/.bash_history
```

3. **Protect Account Files**:
```bash
# Set proper permissions
chmod 700 users/
chmod 600 users/*/BNET/acct/passhash1
```

4. **Use Strong Passwords**:
```bash
# Generate random passwords
PASSWORD=$(openssl rand -base64 12)
bnpass "$PASSWORD"
```

5. **Rotate Admin Passwords**:
```bash
# Periodically change administrator passwords
for admin in admin moderator root; do
    NEW_PASS=$(openssl rand -base64 16)
    echo "New password for $admin: $NEW_PASS"
    HASH=$(bnpass "$NEW_PASS" | cut -d'"' -f4)
    echo "$HASH" > "users/$admin/BNET/acct/passhash1"
done
```

---

## Troubleshooting

### Common Issues

**1. Hash doesn't match stored value**
```bash
# Ensure lowercase conversion
$ bnpass MyPassword
# Becomes: "mypassword"

# Password is case-insensitive for login
# But hash is case-sensitive in storage
```

**2. Special characters in password**
```bash
# Use quotes for passwords with spaces/special chars
$ bnpass "my password 123!"

# Use -- for passwords starting with -
$ bnpass -- "-mypassword"
```

**3. Output format differs from expected**
```bash
# bnpass output:
"BNET\\acct\\passhash1"="abc123..."

# Extract just the hash:
$ bnpass mypass | cut -d'"' -f4
abc123...
```

**4. Utility not found after installation**
```bash
# Check installation path
$ which bnpass
/usr/local/bin/bnpass

# If not found, check install prefix
$ cmake -L build | grep CMAKE_INSTALL_PREFIX

# Add to PATH if needed
export PATH=$PATH:/usr/local/bin
```

### Debugging

Enable verbose output:
```bash
# The utilities use eventlog internally
# Errors are printed to stderr

# Check for errors:
$ bnpass test 2>&1 | grep -i error

# Test with known values:
$ bnpass password
"BNET\\acct\\passhash1"="5baa61e4c9b93f3f0682250b6cf8331b7ee68fd8"

# This should always produce the same hash
```

---

## Advanced Topics

### Hash Format Internals

**Binary Representation**:
```
t_hash = uint32_t[5]
       = 5 x 32-bit integers
       = 160 bits total
       = 20 bytes
```

**String Representation**:
```
40-character hexadecimal string
Each byte → 2 hex digits
Example: "5baa61e4c9b93f3f0682250b6cf8331b7ee68fd8"
```

**Endianness**:
- Hash internally uses big-endian format
- Storage is platform-independent hex string
- No byte-order issues across systems

### Extending the Utilities

To add new hash algorithms:

1. **Add to bnethash.cpp**:
```cpp
extern int bcrypt_hash(t_hash * hashout, unsigned int size, void const * data) {
    // Implementation
}
```

2. **Create new utility**:
```cpp
// bcrypthash.cpp
#include "common/bnethash.h"

int main(int argc, char * argv[]) {
    char buff[256];
    t_hash hash;
    
    // Get password
    // ...
    
    bcrypt_hash(&hash, strlen(buff), buff);
    printf("bcrypt hash = %s\n", hash_get_str(hash));
    
    return 0;
}
```

3. **Update CMakeLists.txt**:
```cmake
add_executable(bcrypthash bcrypthash.cpp)
target_link_libraries(bcrypthash PRIVATE common compat fmt)
```

### Performance

Hash computation is fast:
```bash
# Benchmark: 10,000 hashes
$ time for i in {1..10000}; do 
    bnpass "password$i" > /dev/null
done

# Typical: ~10-30 seconds (depends on system)
# ~300-1000 hashes/second
```

For bulk operations, consider batching:
```bash
# Instead of calling bnpass 1000 times,
# Write a C++ program using bnet_hash() directly
```

---

## Related Tools

**In PvPGN Project**:
- `bnetd` - Server daemon (uses these hashes for authentication)
- `bnbot` - Bot client (can use bnpass for bot accounts)
- `bnchat` - Chat client (demonstrates password hashing)

**External Tools**:
- `openssl` - Standard cryptographic operations
- `hashcat` - Password cracking (demonstrates hash weakness)
- `john` - Password recovery tool

**Comparison**:
```bash
# PvPGN BNET hash
$ bnpass test
"BNET\\acct\\passhash1"="a94a8fe5ccb19ba61c4c0873d391e987982fbbd3"

# Standard SHA-1
$ echo -n "test" | openssl sha1
a94a8fe5ccb19ba61c4c0873d391e987982fbbd3

# Same for short strings, but BNET differs for longer inputs
```

---

## Authors

- Ross Combs (rocombs@cs.nmsu.edu) - Original implementation
- Descolada - Hash algorithm contribution

## License

GNU General Public License v2.0 - See LICENSE file for details.

## See Also

- **Man Page**: `bnpass(1)` - Manual page
- **Server Code**: `src/bnetd/account.cpp` - Account management
- **Hash Code**: `src/common/bnethash.cpp` - Hash implementations
- **Authentication**: `src/bnetd/handle_bnet.cpp` - Login handling
- **Storage**: `src/bnetd/storage*.cpp` - Account storage backends
