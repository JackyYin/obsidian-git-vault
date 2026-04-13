# Scanline Rendering in `ppu_render_scanline()`

## Overview

The Game Boy PPU renders the screen **one horizontal line at a time**, called a **scanline**. `ppu_render_scanline(line)` is called once per scanline (lines 0–143) during the PPU's active drawing period. Each call fills exactly 160 pixels into the framebuffer.

```
Frame = 160 × 144 pixels
        ┌─────────────────────────────── 160px ────────────────────────────────┐
   Y=0  │░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░│
   Y=1  │░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░│
   ...  │                                                                      │
  Y=143 │░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░│
        └──────────────────────────────────────────────────────────────────────┘
        Each row is one call to ppu_render_scanline(Y)
```

---

## Step 1: Read I/O Registers

Before drawing anything, the function reads the current state of the PPU's I/O registers:

| Register | Address  | Name | Purpose                                      |
|----------|----------|------|----------------------------------------------|
| `lcdc`   | `0xFF40` | LCDC | Master control — enable/disable layers       |
| `bgp`    | `0xFF47` | BGP  | Background palette                           |
| `obp0`   | `0xFF48` | OBP0 | Object (sprite) palette 0                    |
| `obp1`   | `0xFF49` | OBP1 | Object (sprite) palette 1                    |
| `scx`    | `0xFF43` | SCX  | Background horizontal scroll                 |
| `scy`    | `0xFF42` | SCY  | Background vertical scroll                   |
| `wy`     | `0xFF4A` | WY   | Window Y position                            |
| `wx`     | `0xFF4B` | WX   | Window X position + 7                        |

### LCDC Bit Breakdown

```
  Bit 7  Bit 6  Bit 5  Bit 4  Bit 3  Bit 2  Bit 1  Bit 0
┌──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┐
│  LCD │  WIN │  WIN │ TILE │  BG  │  OBJ │  OBJ │  BG/ │
│  EN  │  MAP │  EN  │ DATA │  MAP │ SIZE │  EN  │  WIN │
│ bit7 │ bit6 │ bit5 │ bit4 │ bit3 │ bit2 │ bit1 │  EN  │
└──────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┘
   │       │      │      │      │      │      │
   │       │      │      │      │      │      └─ BG/WIN enable (not used here)
   │       │      │      │      │      └──────── OBJ enable
   │       │      │      │      └─────────────── OBJ size: 0=8×8, 1=8×16
   │       │      │      └────────────────────── BG tile map: 0=0x9800, 1=0x9C00
   │       │      └───────────────────────────── Tile data: 0=0x9000 signed, 1=0x8000 unsigned
   │       └──────────────────────────────────── Window tile map: 0=0x9800, 1=0x9C00
   └──────────────────────────────────────────── LCD enable (if 0, return immediately)
```

---

## Step 2: Background & Window — Pixel Loop (x = 0..159)

For each of the 160 horizontal pixels, the function decides whether to draw from the **Window** layer or the **Background** layer.

```
For each pixel X in [0, 159]:

  Is window enabled AND line >= WY AND x >= (WX-1)?
         │
    YES ─┴─► Draw from WINDOW layer
    NO  ────► Draw from BACKGROUND layer
```

### The Tile Map and Tile Data System

The Game Boy's background is a 32×32 grid of **tiles** (each tile is 8×8 pixels), forming a 256×256 pixel virtual canvas. The screen is a 160×144 viewport into this space.

```
256px Virtual BG Canvas (32×32 tiles)
┌────────────────────────────────────────────────┐
│  tile(0,0)  tile(1,0)  ...  tile(31,0)         │
│  tile(0,1)  tile(1,1)  ...  tile(31,1)         │
│    ...                                         │
│                ┌─────────────────────┐         │
│         SCY───►│  Screen Viewport    │         │
│                │    160×144 px       │         │
│         SCX───►│                     │         │
│                └─────────────────────┘         │
│  tile(0,31) ...             tile(31,31)        │
└────────────────────────────────────────────────┘
```

### Background: Coordinate Mapping

To handle scrolling, screen coordinates are wrapped into the 256×256 BG canvas:

```
Screen pixel (X, line)
        │
        ▼
wrapped_x = (X + SCX) mod 256       ← wraps horizontally
wrapped_y = (line + SCY) mod 256     ← wraps vertically
        │
        ▼
Which tile?
  tile_x = wrapped_x / 8   (0..31)
  tile_y = wrapped_y / 8   (0..31)
        │
        ▼
Tile map address = BG_TILE_MAP + tile_y * 32 + tile_x
   (BG_TILE_MAP = 0x9800 or 0x9C00, from LCDC bit 3)
        │
        ▼
tile_num = VRAM[tile_addr]   ← 1-byte tile index
```

```
Tile Map (32×32 bytes at 0x9800 or 0x9C00)
offset: 0    1    2    3  ...  31
      ┌────┬────┬────┬────┬────────┬────┐
row 0 │ 05 │ 12 │ 00 │ 3A │  ...   │ 1B │
row 1 │ 07 │ 00 │ 00 │ 05 │  ...   │ 3C │
...   │    │    │    │    │        │    │
row31 │ 00 │ 01 │ 02 │ 03 │  ...   │ FF │
      └────┴────┴────┴────┴────────┴────┘
         ▲
         tile_num (e.g. 0x05) → index into tile data
```

### Tile Data: Unsigned vs Signed Addressing

Two addressing modes exist depending on LCDC bit 4:

```
LCDC bit 4 = 1 (unsigned / "8000 mode"):
  tile_data_addr = 0x8000 + tile_num * 16
  tile_num is treated as uint8_t (0–255)

LCDC bit 4 = 0 (signed / "8800 mode"):
  tile_data_addr = 0x9000 + (int8_t)tile_num * 16
  tile_num is treated as int8_t (-128..127, where 0=block at 0x9000)

Tile Data Memory Layout (8000 mode):
0x8000 ┌────────────────┐ tile 0   (16 bytes per tile)
       │ row0: byte1    │
       │ row0: byte2    │
       │ row1: byte1    │
       │ row1: byte2    │
       │  ...           │
       │ row7: byte1    │
       │ row7: byte2    │
0x8010 ├────────────────┤ tile 1
       │ ...            │
0x8FFF └────────────────┘ tile 255
```

### Tile Row Decoding: 2bpp Format

Each tile row is stored as **2 bytes** (2 bits per pixel = 4 possible colors).

```
Tile row at line_in_tile = wrapped_y % 8:

  byte1 = VRAM[tile_data_addr + line_in_tile * 2]
  byte2 = VRAM[tile_data_addr + line_in_tile * 2 + 1]

  bit_x = 7 - (wrapped_x % 8)   ← MSB is leftmost pixel

  bit1 = (byte1 >> bit_x) & 1   ← low bit of color index
  bit2 = (byte2 >> bit_x) & 1   ← high bit of color index

  color_idx = (bit2 << 1) | bit1   → value in {0, 1, 2, 3}
```

**Visual example** — decoding all 8 pixels of a tile row:

```
byte1 = 0b10110100
byte2 = 0b01101100

Pixel:     0    1    2    3    4    5    6    7
byte1 bit: 1    0    1    1    0    1    0    0
byte2 bit: 0    1    1    0    1    1    0    0
color_idx: 1    2    3    2    1    3    0    0
           ↑                             ↑
        dark grey                   white (transparent for sprites)
```

### Background Palette Mapping

`color_idx` (0–3) is mapped through the **BGP** register to a final shade:

```
BGP register (0xFF47):
  Bits [1:0] → shade for color index 0
  Bits [3:2] → shade for color index 1
  Bits [5:4] → shade for color index 2
  Bits [7:6] → shade for color index 3

  pal_idx = (bgp >> (color_idx * 2)) & 0x03

Shade table:
  0 → 0xFFFFFFFF  (white)
  1 → 0xFFAAAAAA  (light grey)
  2 → 0xFF555555  (dark grey)
  3 → 0xFF000000  (black)
```

```
Example: BGP = 0xE4 = 0b11100100
  color_idx 0 → bits[1:0] = 00 → shade 0 → WHITE
  color_idx 1 → bits[3:2] = 01 → shade 1 → LIGHT GREY
  color_idx 2 → bits[5:4] = 10 → shade 2 → DARK GREY
  color_idx 3 → bits[7:6] = 11 → shade 3 → BLACK
  (This is the "natural" mapping)
```

### Window Layer

The Window is an alternative tile layer rendered on top of the BG. It uses its own tile map and its own local coordinate system:

```
Screen space:
  win_x = x - (WX - 1)      ← WX=7 means window starts at screen x=0
  win_y = line - WY          ← WY=0 means window starts at screen y=0

Window tile map starts at:
  0x9800 (LCDC bit 6 = 0)
  0x9C00 (LCDC bit 6 = 1)

The window does NOT scroll — it is always anchored to its WX/WY position.
```

```
Screen (160×144)
┌──────────────────────────────────────────┐
│▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓│ ← BG only (line < WY)
│▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓│
│▓▓▓▓▓▓▓▓▓▓▓▓┌──────────────────────────┐▓▓│ ← WY: window starts here
│▓▓▓▓▓▓▓▓▓▓▓▓│░░░░░░░░ WINDOW ░░░░░░░░░░│▓▓│
│▓▓▓▓▓▓▓▓▓▓▓▓│░░░░░░░░░░░░░░░░░░░░░░░░░░│▓▓│
│▓▓▓▓▓▓▓▓▓▓▓▓└──────────────────────────┘▓▓│
└──────────────────────────────────────────┘
             ▲
             WX-1 (screen x where window starts)

  ▓ = Background layer
  ░ = Window layer (covers BG underneath)
```

---

## Step 3: Framebuffer Write

After computing the palette index, the pixel is written to the framebuffer:

```c
mmu->framebuffer[line * SCREEN_W + x] = gb_colors[pal_idx];
```

```
Framebuffer layout (160 × 144 uint32_t RGBA values):

index = line * 160 + x

  x=0    x=1   x=2      ...      x=159
┌──────┬──────┬──────┬──────────┬──────┐
│ [0]  │ [1]  │ [2]  │  ...     │[159] │  ← line 0
├──────┼──────┼──────┼──────────┼──────┤
│[160] │[161] │[162] │  ...     │[319] │  ← line 1
├──────┼──────┼──────┼──────────┼──────┤
│ ...  │      │      │          │      │
├──────┼──────┼──────┼──────────┼──────┤
│[143*160]                      │[last]│  ← line 143
└──────┴──────┴──────┴──────────┴──────┘
```

---

## Step 4: Sprite Rendering

After the background/window pass, sprites (OBJs) are rendered on top.

### OAM Scan: Which Sprites Hit This Line?

There are 40 entries in OAM. Each is 4 bytes:

```
OAM Entry (4 bytes at 0xFE00 + i*4):
  Byte 0: Y position  (actual screen Y = sprite_y - 16)
  Byte 1: X position  (actual screen X = sprite_x - 8)
  Byte 2: Tile number
  Byte 3: Flags

Flags byte:
  Bit 7: Priority (0=above BG, 1=behind BG colors 1-3)
  Bit 6: Y-flip
  Bit 5: X-flip
  Bit 4: Palette (0=OBP0, 1=OBP1)
```

A sprite hits scanline `line` if:

```
actual_y = sprite_y - 16
sprite_h = 8 (8×8 mode) or 16 (8×16 mode)

line >= actual_y  AND  line < actual_y + sprite_h

      actual_y ─► ┌────────┐
                  │        │  (sprite, 8 or 16 rows tall)
  ←─ scanline ─── │════════│ ← line hits here
                  │        │
actual_y+h-1 ─►   └────────┘
```

Up to **10 sprites per scanline** are collected (hardware limit), then sorted by X position (lower X = higher priority; ties broken by OAM index).

### 8×16 Sprite Mode

```
8×16 sprites use two consecutive tiles. The tile number's LSB is forced to 0:
  tile_num &= 0xFE   → top tile = tile N, bottom tile = tile N+1

  ┌────────┐  ← tile N   (top 8 rows)
  │        │
  │        │
  ├────────┤  ← tile N+1 (bottom 8 rows)
  │        │
  │        │
  └────────┘
```

### Sprite Pixel Decoding with Flipping

```
pixel_x = x - actual_x       (0..7)
pixel_y = line - actual_y     (0..sprite_h-1)

if X-flip: pixel_x = 7 - pixel_x
if Y-flip: pixel_y = sprite_h - 1 - pixel_y

tile_data_addr = 0x8000 + tile_num * 16   (sprites always use 0x8000 mode)

byte1 = VRAM[tile_data_addr + pixel_y * 2]
byte2 = VRAM[tile_data_addr + pixel_y * 2 + 1]

bit1 = (byte1 >> (7 - pixel_x)) & 1
bit2 = (byte2 >> (7 - pixel_x)) & 1
color_idx = (bit2 << 1) | bit1
```

```
X-flip example:
  Normal:    pixel_x  0  1  2  3  4  5  6  7
  X-flipped:          7  6  5  4  3  2  1  0

Y-flip example (8×8):
  Normal:    pixel_y  0  1  2  3  4  5  6  7
  Y-flipped:          7  6  5  4  3  2  1  0
```

```
  Normal:                X Flipped:              Y Flipped:
  ┌───────┐              ┌───────┐              ┌───────┐
  │0 1 2 3│              │3 2 1 0│              │4 5 6 7│
  │4 5 6 7│              │7 6 5 4│              │0 1 2 3│
  └───────┘              └───────┘              └───────┘
   pixel_x values         pixel_x values         pixel_x values
   (after flip calc)      (after flip calc)      (after flip calc)
```


### Sprite Transparency and Priority

```
color_idx == 0  →  TRANSPARENT, skip (BG shows through)

priority bit set (flag bit 7):
  sprite is behind BG colors 1, 2, 3
  only drawn if the BG pixel at this X was color 0

  if (priority && bg_color_at_x != 0) → skip sprite pixel
```

### Sprite Drawing Order

Sprites are drawn **right-to-left** (x = 159 down to 0), iterating sprites in sorted order. The first opaque sprite pixel found at each X wins (`break` after write):

```
x = 159 ──► x = 0
  For each x:
    For each sprite (sorted by X, low→high):
      Is x within this sprite's 8px width?
        Decode pixel
        If transparent → next sprite
        If priority and BG != 0 → next sprite
        Write pixel → break (this sprite wins)
```

---

## Complete Pipeline Summary

```
ppu_render_scanline(line Y)
         │
         ▼
  Read LCDC, scrolls, palettes
         │
         ├── LCD disabled? ──► RETURN
         │
         ▼
  ┌─────────────────────────────────────────┐
  │  FOR x = 0 to 159:                      │
  │                                         │
  │   Window active at (x, Y)?              │
  │    YES → win coords → tile map lookup   │
  │    NO  → scroll BG coords → tile map    │
  │                    │                    │
  │            tile_num from VRAM           │
  │                    │                    │
  │          tile data address              │
  │       (unsigned 0x8000 or signed 0x9000)│
  │                    │                    │
  │      row bytes: byte1, byte2            │
  │                    │                    │
  │      2bpp decode → color_idx (0–3)      │
  │                    │                    │
  │      BGP palette → pal_idx (0–3)        │
  │                    │                    │
  │      gb_colors[pal_idx] → framebuffer   │
  └─────────────────────────────────────────┘
         │
         ├── OBJ disabled? ──► RETURN
         │
         ▼
  Scan OAM: collect up to 10 sprites on line Y
         │
         ▼
  Sort sprites by X (then OAM index)
         │
         ▼
  ┌─────────────────────────────────────────┐
  │  FOR x = 159 downto 0:                  │
  │    FOR each sprite:                     │
  │      x within sprite width?             │
  │        Decode pixel (with flip)         │
  │        Transparent? → next sprite       │
  │        BG priority conflict? → next     │
  │        Write to framebuffer → BREAK     │
  └─────────────────────────────────────────┘
         │
         ▼
  Line Y complete: 160 pixels written
```
