# libwebp-roi — ROI Encoding Feature Documentation

> Fork of [webmproject/libwebp](https://github.com/webmproject/libwebp) v1.6.0

---

## Table of Contents

1. [Overview](#overview)
2. [Mechanism](#mechanism)
3. [Usage](#usage)
   - [Rectangular ROI (`-roi`)](#rectangular-roi--roi)
   - [Non-rectangular ROI (`-roi_mb`)](#non-rectangular-roi--roi_mb)
   - [How to find macroblock indices](#how-to-find-macroblock-indices)
4. [VP8 Specification Mapping](#vp8-specification-mapping)
5. [Modified Files](#modified-files)
6. [Limitations](#limitations)
7. [Credits](#credits)

---

## Overview

This fork adds two ROI (Region of Interest) encoding options to `cwebp`.
Macroblocks inside the ROI are encoded at maximum quality; all others at minimum quality.
This allows significant file size reduction while preserving the visual quality of
the important region.
The output is a standard WebP file viewable by any ordinary image viewer without
any custom decoder.

---

## Mechanism

WebP lossy encoding (VP8-based) divides an image into **16×16-pixel macroblocks**,
grouped into up to 4 **segments**, each with its own quantization parameter.

This ROI feature forces two segments with the following settings:

```
segment 0 (non-ROI): quant=127, fstrength=63  →  lowest quality
segment 1 (ROI):     quant=0,   fstrength=0   →  highest quality
```

Each macroblock is assigned to a segment based on its position relative to the ROI:

```
Input image example (80x48px = 5x3 macroblocks)

  0   16   32   48   64   80  px
  |    |    |    |    |    |
0 +----+----+----+----+----+
  |  0 |  1 |  2 |  3 |  4 |
16+----+#####+#####+----+----+
  |  5 |# 6#|# 7#|  8 |  9 |    <- -roi 16 16 47 47
32+----+#####+#####+----+----+      or  -roi_mb 6,7
  | 10 | 11 | 12 | 13 | 14 |
48+----+----+----+----+----+

  #### = segment 1 (ROI, high quality)
  [  ] = segment 0 (non-ROI, low quality)
```

#### Segment assignment flow

```
SetupFilterStrength()
  |
  +-- segment 0: quant=127, fstrength=63  (non-ROI: lowest quality)
  +-- segment 1: quant=0,   fstrength=0   (ROI:     highest quality)
  |
  v
SimplifySegments()
  |
  +-- if num_segments reduced to 1, force back to 2
  |   (prevents invalid segment reference)
  |
  +-- if roi_mb_mask != NULL  (-roi_mb mode):
  |     mb_info[i].segment = roi_mb_mask[i] ? 1 : 0
  |
  +-- else if roi == 1  (-roi mode):
  |     col = i % mb_w,  row = i / mb_w
  |     mb_info[i].segment = (col in [x1..x2] AND row in [y1..y2]) ? 1 : 0
  |
  v
VP8 encoder applies per-segment quantization to each macroblock
```

---

## Usage

### Rectangular ROI (`-roi`)

Specify a rectangular region of interest by pixel coordinates.

```
cwebp -roi <x1> <y1> <x2> <y2> input.png -o output.webp
```

| Argument | Description |
|---|---|
| `x1`, `y1` | Top-left corner of ROI in pixels (inclusive) |
| `x2`, `y2` | Bottom-right corner of ROI in pixels (inclusive) |

Coordinates are internally floored to the 16px macroblock grid.

**Example:**
```
cwebp -roi 64 64 192 192 photo.png -o photo_roi.webp
```

---

### Non-rectangular ROI (`-roi_mb`)

Directly specify macroblock indices as a comma-separated list.
Any arbitrary shape can be expressed.

```
cwebp -roi_mb <idx1>,<idx2>,... input.png -o output.webp
```

| Argument | Description |
|---|---|
| `idx1,idx2,...` | 0-indexed macroblock indices to mark as ROI, comma-separated |

When `-roi_mb` is specified, it takes precedence over `-roi`.

**Example:**
```
# For a 5x3 macroblock image (80x48px): set center 2 blocks as ROI
cwebp -roi_mb 6,7 photo.png -o photo_roi.webp
```

---

### How to find macroblock indices

Macroblocks are numbered left-to-right, top-to-bottom, starting from 0.

```
For an image W x H pixels:

  mb_w = ceil(W / 16)   (number of columns)
  mb_h = ceil(H / 16)   (number of rows)

  Macroblock index = row * mb_w + col

Example: 80x48px (mb_w=5, mb_h=3)

   col:  0    1    2    3    4
row 0: [  0][  1][  2][  3][  4]
row 1: [  5][  6][  7][  8][  9]
row 2: [ 10][ 11][ 12][ 13][ 14]

Formula to get macroblock index from pixel coordinate (px, py):

  idx = (py / 16) * mb_w + (px / 16)
```

---

## VP8 Specification Mapping

This implementation uses VP8's segment-based quantization feature directly (RFC 6386).
No changes to the decoder are required.

### WebP → VP8 structure

```
File (RIFF format)
└── VP8 chunk
    ├── Uncompressed header (10 bytes)  frame_type / version / size
    └── VP8 bitstream (boolean arithmetic coding)
        ├── Frame header          <- segment definitions
        └── Macroblock data       <- per-MB segment IDs
```

WebP lossy is a VP8 key frame wrapped in a RIFF container.
All segment functionality is defined by the VP8 specification.

### Frame header: segment definitions (RFC 6386 §9.3 / §9.6)

The compressed frame header contains the following fields:

```
segmentation_enabled          L(1)  = 1    <- always 1 when ROI is active
update_mb_segmentation_map    L(1)  = 1    <- write per-MB segment IDs
update_segment_feature_data   L(1)  = 1    <- write per-segment quant values
  segment_feature_mode        L(1)  = 0    <- absolute mode (not delta)

  segment[0].yac_qi           L(7)  = 127  <- non-ROI (lowest quality)
  segment[1].yac_qi           L(7)  = 0    <- ROI (highest quality)
```

`yac_qi` is the Y-plane AC quantization index (0 = highest quality, 127 = lowest).

Corresponding code in `SetupFilterStrength()`:

```c
m->quant = (i == 0) ? 127 : (i == 1) ? 0 : m->quant;
//          ^-- segment[0].yac_qi=127   ^-- segment[1].yac_qi=0
```

### Macroblock data: segment ID (RFC 6386 §19.3)

After the frame header, macroblocks are coded in raster-scan order.
Each macroblock begins with an entropy-coded segment ID:

```
segment_id  in {0, 1, 2, 3}   <- coded via mb_segment_tree probability tree
```

`ApplyROISegments()` forces this to one of two values:

```c
enc->mb_info[i].segment = roi_mb_mask[i] ? 1 : 0;
//                         ^-- segment_id for macroblock i
```

### Loop filter (RFC 6386 §9.4)

VP8 segments control loop filter strength in addition to quantization:

```
filter_strength[0] = 63   <- non-ROI: strong filter (reduces block artifacts)
filter_strength[1] = 0    <- ROI: no filter (preserves edges as-is)
```

No filter is needed at high quality; heavy filtering is appropriate at low quality.

### ApplyROISegments() design rationale

`SimplifySegments()` is a VP8 encoder optimization that merges equivalent segments.
It may be called multiple times across encoding passes. ROI assignment is therefore
extracted into a dedicated function applied **after** `SimplifySegments()` each time:

```
VP8SetSegmentParams()
  ├── SetupFilterStrength()   <- set quant / fstrength for segment[0] and [1]
  ├── SimplifySegments()      <- merge equivalent segments (does not touch ROI)
  └── ApplyROISegments()      <- assign final per-MB segment IDs for ROI
```

This ordering ensures that ROI assignment is always the final state,
regardless of what SimplifySegments() reduces the segment count to.

---

## Modified Files

| File | Change |
|---|---|
| `src/webp/encode.h` | Added `roi`, `roi_x1/x2/y1/y2`, `roi_mb_mask` to `WebPConfig` |
| `src/enc/quant_enc.c` | Added ROI quantization settings in `SetupFilterStrength()`. Extracted ROI segment assignment into `ApplyROISegments()`, called explicitly from `VP8SetSegmentParams()` after `SimplifySegments()` |
| `examples/cwebp.c` | Added `-roi` and `-roi_mb` CLI options with help text |

---

## Limitations

**1. Rectangle mode coordinate precision**

`-roi` coordinates are floored to the 16px macroblock grid.
Sub-macroblock precision is not available.

```
Specified: x1=10  ->  Actual: x1=0   (10/16 = 0, i.e. 0px)
Specified: x1=20  ->  Actual: x1=1   (20/16 = 1, i.e. 16px)
```

`-roi_mb` directly specifies macroblock units and does not have this limitation.

**2. Binary quality only**

Only two quality levels are supported: maximum inside ROI, minimum outside.
Gradual falloff or intermediate quality levels are not implemented.

```
Quality

 high |####|
      |####|
      |    |________
 low  |
      +----+---------> distance from ROI edge
      ROI  outside
```

**3. Conflict with `-q` / `-sns`**

When ROI mode is active, `-q` (quality) and `-sns` (spatial noise shaping)
are overridden internally by the ROI segment settings.

**4. Lossy encoding only**

ROI encoding applies to lossy WebP only.
The `-roi` and `-roi_mb` options are ignored when `-lossless` is specified.

---

## Credits

Part of the implementation and code review was done using [Claude Code](https://claude.ai/code) by Anthropic.
