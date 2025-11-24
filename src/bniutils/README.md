# BNIUTILS - Battle.net Icon Utilities

## Overview

`bniutils` is a collection of command-line utilities for working with Battle.net Icon (BNI) files and Targa (TGA) image files. These tools enable developers and server administrators to inspect, extract, build, and convert the icon files used by Battle.net game clients for displaying user avatars, ranks, and status indicators in chat channels.

## Purpose

Battle.net clients display small icons next to usernames in chat channels to indicate:
- **Player Race/Faction** (StarCraft, Warcraft 3)
- **Ladder Rank** (Bronze, Silver, Gold, etc.)
- **Win Count Tiers** (different icons for milestone achievements)
- **Special Status** (operators, Blizzard employees, bots)
- **Tournament Participation**

BNI files are Blizzard's proprietary container format that stores multiple small icons as a single TGA (Targa) image with metadata describing each icon's location and dimensions within that image. The bniutils package provides tools to work with these files outside of the game client.

## File Format Background

### BNI File Structure

A BNI file consists of:

1. **Header** (16 bytes base + variable icon entries)
   - `unknown1` (4 bytes): Always `0x00000010` - likely format version
   - `unknown2` (4 bytes): Always `0x00000001` - unknown purpose
   - `numicons` (4 bytes): Number of icons in the file
   - `dataoffset` (4 bytes): Byte offset where TGA image data begins

2. **Icon Entries** (16 or 20 bytes each)
   - `id` (4 bytes): Icon identifier (0 = use tag instead)
   - `x` (4 bytes): Icon width in pixels
   - `y` (4 bytes): Icon height in pixels
   - `tag` (4 bytes): 4-character tag (only if id == 0), e.g., "STAR" for StarCraft
   - `unknown` (4 bytes): Unknown flags, typically `0x00000000`

3. **TGA Image Data** (starts at `dataoffset`)
   - Single vertical strip containing all icons stacked
   - Typically 28 pixels wide (one icon width)
   - Height = sum of all icon heights
   - Format: 24-bit true-color, RLE-compressed
   - Each icon occupies a rectangular region in the strip

### TGA Format Support

The utilities support various TGA (Targa) image formats:
- **Uncompressed**: mapped, true-color, monochrome
- **RLE-compressed**: mapped, true-color, monochrome (most common for BNI)
- **Color depths**: 8, 15, 16, 24, 32 bits per pixel
- **Attributes**: Alpha channel support
- **Orientation**: Top-down or bottom-up, left-right or right-left

## Utilities

### 1. bnilist

**Purpose**: Inspect and validate BNI file structure

**Usage**:
```bash
bnilist [options] [BNI_file]
```

**Description**:
Reads a BNI file and displays detailed information about its header and icon entries without extracting the actual image data. Performs validation checks to ensure the file format is correct.

**Output Example**:
```
BNIHeader: id=0x00000010 unknown=0x00000001 icons=0x0000000C datastart=0x000000C4
Icon[0]: id=0x00000000 x=28 y=28 tag=0x52415453("STAR") flags=0x00000000
Icon[1]: id=0x00000001 x=28 y=28 tag=0x00000000 flags=0x00000000
...
Check: Expected 28x336 TGA, got 28x336. OK.
Check: Expected 24bit color depth TGA, got 24bit. OK.
Check: Expected ImageType 10, got 10. OK.
```

**Use Cases**:
- Verify BNI file integrity before deployment
- Inspect icon count and dimensions
- Debug custom icon files
- Validate TGA format compatibility

**Options**:
- `-h, --help, --usage`: Show usage information
- `-v, --version`: Print version number
- `--`: End of options marker

---

### 2. bni2tga

**Purpose**: Extract TGA image data from BNI file

**Usage**:
```bash
bni2tga [options] [BNI_file [TGA_file]]
```

**Description**:
Extracts the raw TGA image data from a BNI file, discarding the icon metadata. The resulting TGA contains all icons as a vertical strip. Use this when you need to view or edit the icons as a single image.

**Input/Output**:
- If `BNI_file` is `-` or omitted: reads from stdin
- If `TGA_file` is `-` or omitted: writes to stdout
- Suitable for piping in shell scripts

**Example**:
```bash
# Extract icons.bni to icons.tga
bni2tga icons.bni icons.tga

# Use with pipes
cat icons.bni | bni2tga > icons.tga

# View immediately with ImageMagick
bni2tga icons.bni - | display -
```

**Use Cases**:
- Quick preview of all icons in one image
- Batch processing BNI files
- Converting for use in image editors
- Creating backups of icon artwork

---

### 3. bniextract

**Purpose**: Extract individual icon images from BNI file

**Usage**:
```bash
bniextract [options] BNI_file output_directory
```

**Description**:
Fully decomposes a BNI file into individual TGA files for each icon, plus an index file describing the icon layout. This is the most detailed extraction mode, preserving all metadata.

**Output Files**:
- `bniindex.lst`: Text file describing icon layout
- `XXXX.tga` or `NNNNNNNN.tga`: Individual icon files

**Index File Format** (`bniindex.lst`):
```
unknown1 00000010
unknown2 00000001
icon !STAR 28 28 00000000
icon #00000001 28 28 00000000
icon #00000002 28 28 00000000
...
```

**Icon Naming Scheme**:
- **Tag-based** (id == 0): `STAR.tga`, `SEXP.tga`, `W3XP.tga`
- **ID-based** (id != 0): `00000001.tga`, `00000002.tga`, etc.

**Example**:
```bash
# Extract all icons to ./icons/ directory
mkdir icons
bniextract icons_STAR.bni ./icons/

# Result:
# ./icons/bniindex.lst
# ./icons/STAR.tga
# ./icons/00000001.tga
# ./icons/00000002.tga
# ...
```

**Use Cases**:
- Edit individual icons in image editor
- Create custom icon sets
- Analyze specific icon artwork
- Prepare icons for bnibuild reconstruction

---

### 4. bnibuild

**Purpose**: Build BNI file from individual icon images

**Usage**:
```bash
bnibuild [options] index_file input_directory output_BNI_file
```

**Description**:
The reverse of bniextract - takes an index file and directory of TGA images and combines them into a single BNI file. Used for creating custom icon files or rebuilding after editing.

**Input Requirements**:
- **Index file**: Text file specifying icon metadata (see bniextract format)
- **Input directory**: Folder containing TGA files referenced in index
- **Icon TGAs**: Must match dimensions specified in index file

**Index File Commands**:
```
unknown1 <hex>        # Set unknown1 field (default: 00000010)
unknown2 <hex>        # Set unknown2 field (default: 00000001)
icon !XXXX w h flags  # Add tag-based icon (XXXX = 4-char tag)
icon #NNNNNNNN w h f  # Add ID-based icon (NNNNNNNN = hex ID)
```

**Example**:
```bash
# Create custom icons.bni from edited icons
bnibuild bniindex.lst ./icons/ custom_icons.bni
```

**Workflow**:
1. Extract existing BNI: `bniextract icons.bni ./edit/`
2. Edit individual icon TGA files in image editor
3. Rebuild BNI: `bnibuild ./edit/bniindex.lst ./edit/ new_icons.bni`
4. Deploy to PvPGN: Copy `new_icons.bni` to `files/` directory

**Use Cases**:
- Create custom icon sets for private servers
- Add new icons for ladder ranks or special events
- Modify existing icons for branding
- Experiment with different icon artwork

---

### 5. tgainfo

**Purpose**: Display TGA file metadata

**Usage**:
```bash
tgainfo [options] [TGA_file]
```

**Description**:
Reads and displays detailed information about a TGA image file without loading the actual pixel data. Useful for verifying TGA file properties before using in BNI operations.

**Output Information**:
- Image dimensions (width x height)
- Color depth (bits per pixel)
- Image type (uncompressed, RLE-compressed, etc.)
- Color map information
- Attribute bits
- Origin and orientation
- Extension/developer area offsets

**Example Output**:
```
TGA Header Information:
  ID Length: 0
  Color Map Type: 0 (no color map)
  Image Type: 10 (RLE true-color)
  Color Map: First=0, Length=0, Entry Size=0
  Image Spec: X=0, Y=0, Width=28, Height=336
  Pixel Depth: 24 bits
  Image Descriptor: 0x00
```

**Use Cases**:
- Validate TGA files before adding to BNI
- Debug image format issues
- Verify color depth and compression
- Check orientation settings

---

## Common Use Cases

### Inspecting Official Icon Files

```bash
# Check what's in the StarCraft icon file
bnilist files/icons_STAR.bni

# Extract for detailed viewing
bniextract files/icons_STAR.bni ./starcraft_icons/

# Count the icons
ls -1 ./starcraft_icons/*.tga | wc -l
```

### Creating Custom Icons

```bash
# 1. Extract base icon set
bniextract files/icons.bni ./base_icons/

# 2. Create new icon in image editor (28x28 pixels, 24-bit TGA)
#    Save as base_icons/0000000D.tga

# 3. Add entry to bniindex.lst
echo "icon #0000000D 28 28 00000000" >> ./base_icons/bniindex.lst

# 4. Rebuild BNI file
bnibuild ./base_icons/bniindex.lst ./base_icons/ files/custom_icons.bni

# 5. Update bnetd.conf
#    iconfile = "files/custom_icons.bni"
```

### Batch Converting Icons

```bash
# Extract all BNI files in directory
for bni in *.bni; do
    name="${bni%.bni}"
    mkdir "$name"
    bniextract "$bni" "$name/"
done

# Convert all icons to PNG
for tga in */*.tga; do
    convert "$tga" "${tga%.tga}.png"
done
```

### Validating Icon Sets

```bash
# Check if BNI is valid
bnilist custom.bni > /dev/null && echo "Valid BNI" || echo "Invalid BNI"

# Verify TGA format for all icons
for tga in icons/*.tga; do
    echo "Checking $tga..."
    tgainfo "$tga" | grep "Image Type"
done
```

## Library Components

### fileio.cpp/h - Binary I/O Helpers

Low-level file reading/writing functions for binary data:

**Functions**:
- `file_rpush(FILE*)` / `file_rpop()`: Read context stack
- `file_wpush(FILE*)` / `file_wpop()`: Write context stack
- `file_readb()`: Read 8-bit unsigned byte
- `file_readw_le()`, `file_readw_be()`: Read 16-bit word (little/big endian)
- `file_readd_le()`, `file_readd_be()`: Read 32-bit dword (little/big endian)
- `file_writeb()`, `file_writew_*()`, `file_writed_*()`: Write functions

**Purpose**: Abstracts platform-specific binary I/O and endianness handling

**Design Pattern**: Stack-based context for tracking file position across nested operations

---

### bni.cpp/h - BNI File Operations

Core BNI format parsing and writing:

**Data Structures**:
```cpp
typedef struct {
    unsigned int id;       // Icon ID or 0 for tag-based
    unsigned int x, y;     // Width and height
    unsigned int tag;      // 4-char tag (if id == 0)
    unsigned int unknown;  // Flags (usually 0)
} t_bniicon;

typedef struct {
    unsigned int unknown1;    // Always 0x00000010
    unsigned int unknown2;    // Always 0x00000001
    unsigned int numicons;    // Icon count
    unsigned int dataoffset;  // TGA data offset
    struct bni_iconlist_struct *icons;
} t_bnifile;
```

**Functions**:
- `load_bni(FILE*)`: Parse BNI file, returns `t_bnifile*`
- `write_bni(FILE*, t_bnifile*)`: Write BNI structure to file

**Notes**:
- Maximum icons: `BNI_MAXICONS` (default: 4096)
- Validates magic numbers
- Detects garbage data between header and TGA

---

### tga.cpp/h - TGA Image Operations

TGA format reading, writing, and manipulation:

**Data Structure**:
```cpp
typedef struct {
    uint8_t idlen;          // ID field length
    uint8_t cmaptype;       // Color map type
    uint8_t imgtype;        // Image type (RLE, uncompressed, etc.)
    uint16_t cmapfirst;     // Color map offset
    uint16_t cmaplen;       // Color map length
    uint8_t cmapes;         // Color map entry size
    uint16_t xorigin, yorigin;
    uint16_t width, height;
    uint8_t bpp;            // Bits per pixel
    uint8_t desc;           // Descriptor (attributes, orientation)
    uint8_t* data;          // Pixel data
    uint32_t extareaoff;    // Extension area offset
    uint32_t devareaoff;    // Developer area offset
} t_tgaimg;
```

**Image Types** (`t_tgaimgtype`):
- `tgaimgtype_empty` (0): No image data
- `tgaimgtype_uncompressed_mapped` (1): Color-mapped
- `tgaimgtype_uncompressed_truecolor` (2): RGB/RGBA
- `tgaimgtype_uncompressed_monochrome` (3): Grayscale
- `tgaimgtype_rlecompressed_*` (9-11): RLE variants (most common)
- `tgaimgtype_huffman_*` (32-33): Rare compression

**Functions**:
- `new_tgaimg(width, height, bpp, type)`: Allocate new image
- `load_tga(FILE*)`: Read TGA file
- `load_tgaheader()`: Read header only (no pixel data)
- `write_tga(FILE*, t_tgaimg*)`: Write TGA file
- `getpixelsize(t_tgaimg*)`: Calculate bytes per pixel
- `print_tga_info(t_tgaimg*, FILE*)`: Debug output
- `destroy_img(t_tgaimg*)`: Free memory

**Internal Helpers**:
- RLE decompression/compression
- Pixel format conversion
- Orientation handling

---

## Building

### CMake Configuration

```cmake
add_executable(bnilist bnilist.cpp fileio.cpp fileio.h tga.cpp tga.h)
add_executable(bni2tga bni2tga.cpp fileio.cpp fileio.h)
add_executable(bniextract bniextract.cpp fileio.cpp fileio.h tga.cpp tga.h bni.cpp bni.h)
add_executable(bnibuild bnibuild.cpp fileio.cpp fileio.h bni.cpp bni.h tga.cpp tga.h)
add_executable(tgainfo tgainfo.cpp fileio.cpp fileio.h tga.cpp tga.h)

# All utilities link against:
target_link_libraries(<utility> PRIVATE common compat)
```

### Dependencies

- **common**: PvPGN common utilities (`xalloc`, `version`, etc.)
- **compat**: Platform compatibility layer (`mkdir`, `stat` macros)
- **Standard C/C++**: stdio, stdlib, string, errno, sys/stat

### Compilation

```bash
# From project root
cmake -B build
cmake --build build --target bnilist bni2tga bniextract bnibuild tgainfo

# Binaries will be in: build/src/bniutils/
```

## Icon Files in PvPGN

### Standard Icon Files

PvPGN uses different BNI files for different games:

| File | Purpose | Typical Size |
|------|---------|--------------|
| `icons.bni` | General icons (Diablo, Warcraft II) | 28x280-336px |
| `icons_STAR.bni` | StarCraft/Brood War icons | 28x336px (12 icons) |
| `icons-WAR3.bni` | Warcraft III icons | 28x variable |

### Icon Configuration

In `conf/bnetd.conf`:
```
# Default icon file for most clients
iconfile = "files/icons.bni"

# StarCraft-specific icons
star_iconfile = "files/icons_STAR.bni"

# Warcraft III-specific icons  
war3_iconfile = "files/icons-WAR3.bni"
```

### Icon Selection Logic

The server sends different icon files based on:
1. **Client tag** (STAR, SEXP, WAR3, W3XP, etc.)
2. **User rank/rating** (ladder position)
3. **Win count** (milestones: 25, 150, 350, 750, 1500+ wins)
4. **Race/faction** (Human, Orc, Undead, Night Elf, Random)
5. **Special status** (operator, bot, employee)

### Custom Icons

PvPGN supports custom icon systems via `conf/icons.conf`:
```
[STAR]
rating_key = "STAR\\0\\rating"

[icons]
0     none    0000
1000  Bronze  0001
1400  Silver  0002
1600  Gold    0003
```

The bniutils allow creating the actual icon graphics to match the rating system.

## Technical Details

### Endianness

BNI files use **little-endian** byte order:
- All multi-byte integers (16-bit, 32-bit) stored least-significant byte first
- `fileio` library handles conversion automatically
- Compatible with x86/x64 platforms (native)
- Cross-platform builds handle big-endian systems

### Memory Management

- Uses PvPGN's `xalloc()` wrappers for allocation
- Proper cleanup with explicit free/destroy functions
- Stack-based file context to prevent leaks
- Error handling returns NULL/failure codes

### Error Handling

All utilities:
- Validate input files exist and are readable
- Check file format magic numbers
- Verify dimensions and offsets
- Report errors to stderr with errno details
- Return `EXIT_SUCCESS` (0) or `EXIT_FAILURE` (1)

### Platform Compatibility

- **POSIX**: Full support (Linux, macOS, BSD)
- **Windows**: Full support with compat layer
- **Paths**: Uses forward slash internally, converted as needed
- **File I/O**: Binary mode enforced (`"rb"`, `"wb"`)

## Limitations

- **Maximum Icons**: 4096 per BNI file (adjustable via `BNI_MAXICONS`)
- **TGA Format**: Limited to formats supported by `tga.cpp`
- **No Animation**: Icons are static images only
- **Fixed Width**: Typically 28 pixels (Blizzard standard)
- **Single Strip**: All icons in one vertical TGA (no sprite sheets)

## Debugging Tips

### BNI File Issues

```bash
# Check if file is valid BNI
file icons.bni  # Should show "data"
hexdump -C icons.bni | head -n 5  # Check header bytes

# Validate structure
bnilist icons.bni 2>&1 | grep -i error

# Compare with working file
bnilist working.bni > working.txt
bnilist broken.bni > broken.txt
diff working.txt broken.txt
```

### TGA Problems

```bash
# Verify TGA is readable
tgainfo icon.tga

# Check dimensions
identify icon.tga  # ImageMagick

# Convert problematic TGA
convert icon.tga -depth 24 -compress rle fixed.tga
```

### Build Issues

```bash
# Rebuild BNI with verbose output
bnibuild bniindex.lst ./icons/ test.bni 2>&1 | tee build.log

# Check icon file exists
cat bniindex.lst | grep "^icon" | while read _ id rest; do
    [ -f "./icons/${id#?}.tga" ] || echo "Missing: ${id#?}.tga"
done
```

## Integration with PvPGN

### Server Startup

When bnetd starts:
1. Loads `iconfile` from configuration
2. Reads BNI header to determine icon count
3. Keeps file handle open for client requests
4. Sends icon file to clients on connection

### Client Icon Requests

1. Client sends `CLIENT_ICONREQ` packet
2. Server responds with file timestamp and name
3. Client checks local cache
4. If outdated, client requests file download via `CLIENT_FILEINFOREQ`
5. Server streams BNI file to client

### Dynamic Icon Assignment

Server code (`icons.cpp`) uses bniutils concepts:
- Reads icon file metadata
- Assigns icon ID based on account stats
- Sends icon code to client (e.g., "3RAW 1 2 100" for Orc rank)
- Client looks up icon in cached BNI file

## Future Enhancements

Possible improvements:
- **PNG Support**: Modern image format input/output
- **Sprite Sheet Mode**: Multiple columns instead of single strip
- **Icon Previewer**: GUI tool to visualize BNI contents
- **Batch Builder**: Create BNI from directory without index file
- **Format Converter**: BNI ↔ ZIP of PNGs with metadata
- **Validation Tool**: Comprehensive BNI file checker

## Example Workflows

### Creating Warcraft 3 Custom Ladder Icons

```bash
#!/bin/bash
# Create custom ladder icon set for Warcraft 3

# 1. Extract base icons
bniextract files/icons-WAR3.bni ./w3_icons/

# 2. Create 5 new rank icons (28x28, 24-bit TGA)
gimp # Create: bronze.tga, silver.tga, gold.tga, platinum.tga, diamond.tga

# 3. Add to index
cat >> ./w3_icons/bniindex.lst << EOF
icon #00000100 28 28 00000000
icon #00000101 28 28 00000000
icon #00000102 28 28 00000000
icon #00000103 28 28 00000000
icon #00000104 28 28 00000000
EOF

# 4. Copy new icons
cp bronze.tga ./w3_icons/00000100.tga
cp silver.tga ./w3_icons/00000101.tga
cp gold.tga ./w3_icons/00000102.tga
cp platinum.tga ./w3_icons/00000103.tga
cp diamond.tga ./w3_icons/00000104.tga

# 5. Build new BNI
bnibuild ./w3_icons/bniindex.lst ./w3_icons/ files/custom-war3.bni

# 6. Configure in bnetd.conf
echo "war3_iconfile = files/custom-war3.bni" >> conf/bnetd.conf

# 7. Configure ranks in icons.conf
cat >> conf/icons.conf << EOF
[WAR3]
[icons]
1000  Bronze    00000100
1200  Silver    00000101
1400  Gold      00000102
1600  Platinum  00000103
1800  Diamond   00000104
[/icons]
EOF
```

### Automated Icon Validation

```bash
#!/bin/bash
# Validate all BNI files in files/ directory

validate_bni() {
    local bni="$1"
    echo "Validating $bni..."
    
    # Check format
    if ! bnilist "$bni" > /dev/null 2>&1; then
        echo "  ERROR: Invalid BNI format"
        return 1
    fi
    
    # Extract
    local tmpdir=$(mktemp -d)
    if ! bniextract "$bni" "$tmpdir" 2>/dev/null; then
        echo "  ERROR: Extraction failed"
        rm -rf "$tmpdir"
        return 1
    fi
    
    # Check all TGAs exist
    local icons=$(grep "^icon" "$tmpdir/bniindex.lst" | wc -l)
    local tgas=$(ls -1 "$tmpdir"/*.tga 2>/dev/null | wc -l)
    
    if [ $icons -ne $tgas ]; then
        echo "  ERROR: Icon count mismatch ($icons expected, $tgas found)"
        rm -rf "$tmpdir"
        return 1
    fi
    
    # Rebuild and compare
    local rebuild="$tmpdir/rebuild.bni"
    if ! bnibuild "$tmpdir/bniindex.lst" "$tmpdir" "$rebuild" 2>/dev/null; then
        echo "  ERROR: Rebuild failed"
        rm -rf "$tmpdir"
        return 1
    fi
    
    if ! cmp -s "$bni" "$rebuild"; then
        echo "  WARNING: Rebuild differs (acceptable if only metadata changed)"
    fi
    
    rm -rf "$tmpdir"
    echo "  OK: $icons icons validated"
    return 0
}

# Validate all BNI files
for bni in files/*.bni; do
    validate_bni "$bni"
done
```

## Authors

- Marco Ziech (mmz@gmx.net) - Original implementation
- Ross Combs (rocombs@cs.nmsu.edu) - Additional development

## License

GNU General Public License v2.0 - See LICENSE file for details.

## See Also

- **Man Pages**: `bnilist(1)`, `bni2tga(1)`, `bniextract(1)`, `bnibuild(1)`, `tgainfo(1)`
- **PvPGN Docs**: `docs/` - Server documentation
- **Icons Config**: `conf/icons.conf` - Custom icon configuration
- **Server Code**: `src/bnetd/icons.cpp` - Icon management in bnetd
- **TGA Spec**: Truevision TGA File Format Specification v2.0
