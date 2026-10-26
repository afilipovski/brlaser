# Investigation: brlaser failing on complex documents at 600 DPI for Brother HL-1112E

## Summary

The Brother HL-1112E has only **1 MiB RAM** and uses Brother's PCL "1030"
encoding method (`src/job.cc:110`). In this method, the page is divided into
**bands of `n` lines** that the printer prints atomically. The CAPS Universe
documentation (the independent reverse-engineered reference for this protocol,
at https://caps-printing.org/documentation/EncoderC1030.html) is explicit
about three hard rules the printer firmware enforces:

1. A band must contain the **full** line count — incomplete bands are
   **silently skipped**.
2. A single block must not push lines past the band boundary — overflow lines
   are **silently dropped**.
3. The first line of each block must not be delta-encoded against a reference
   line.

Three issues in the brlaser source code violate (or risk violating) these
rules, and combined with the HL-1112E's tiny memory, explain why complex
documents (pictures) at 600 DPI silently fail.

---

## Issue 1: `lines_per_band` mismatch with Brother's official driver

**File:** `src/job.cc:112-113`

```cpp
// XXX brother driver uses 128 lines per band
const int lines_per_band = 64;
```

The `XXX` is a known TODO marker. Brother's official driver uses **128** lines
per band; brlaser uses 64. While 64 divides 128 evenly (so two brlaser bands
*should* equal one printer band), this is fragile:

- The official CAPS reference (`encode_method_1030.h`) defines
  `C1030_MAX_LINES_IN_ONE_BAND = 256` as a *default*, and notes: "The real
  count depends on the printer device and its available memory. This value is
  a default only and can be overwritten by the `band_line_count_1030` INI
  variable."
- If the HL-1112E's actual band size is anything other than a multiple of 64,
  block boundaries will not align with band boundaries, and lines will be
  silently dropped per rule #2 above.

The brlaser author clearly knew this was a compromise (hence the XXX comment)
but never resolved it.

## Issue 2: No padding for the last band  ← MOST LIKELY CULPRIT

**File:** `src/job.cc:131-133`

```cpp
block.flush(out_);
fputs("1030M\f", out_);
fflush(out_);
```

After the last block, brlaser simply flushes and terminates the page with the
`1030M` + form-feed. There is **no code to pad the final band** with empty
lines so that it reaches the printer's full band line count.

Per the CAPS documentation:

> "Each band must have the full count of lines to be printed, else it gets
> skipped. If the band contains less than the required lines, you need to fill
> it up with empty lines in order to print the intended lines."

The official CAPS implementation has a dedicated `hl_c1030_block_pad()`
function for exactly this purpose. Its documentation reads:

> "The printer only prints in bands of n lines. Bands with less than these n
> lines are skipped entirely (and silently). This hurts at the bottom edge of
> a page, where the vertical dot count might not be a multiple of these n
> lines. This function pads the current data block up to these n lines on
> demand with empty lines. It must be called once at the end of each page if
> you know your last band has less lines than these n lines."

A `grep` of brlaser's `src/` directory for `pad|padding|empty_line|fill.*band`
returns **no matches** — this functionality is entirely missing.

For an A4 page at 600 DPI:
- A4 height = 297 mm = 11.69" → 11.69 × 600 ≈ 7016 lines
- 7016 mod 128 = **104** (assuming the HL-1112E uses 128-line bands)
- The last band has only 104 lines instead of 128 → **silently skipped**

For complex images that compress poorly (the use case the user describes),
this combined with the HL-1112E's tiny 1 MiB memory is often enough to make
the printer reject the job entirely — the LED blinks, the printer returns
"success" to CUPS, and nothing is actually printed. This matches the symptom
reported in GitHub issue #40 for the very similar HL-1210W:
<https://github.com/pdewacht/brlaser/issues/40>

## Issue 3: Variable block line counts for incompressible data

**Files:** `src/block.h:59`, `src/job.cc:115-129`

```cpp
// src/block.h:59
static const unsigned max_block_size_ = 16350;
```

This matches the official `C1030_MAX_BYTES_IN_ONE_BLOCK = 16350`. For A4 at
600 DPI, each raster line is ~620 bytes (4962 dots / 8). For complex images
that don't compress well, a block fills at the byte limit after only ~26
lines — far short of the 64-line force-flush boundary.

The encode loop in `src/job.cc:115-129`:

```cpp
for (int i = 1; i < lines && nextline(line); ++i) {
  std::vector<uint8_t> encoded;
  if (i % lines_per_band == 0) {
    block.flush(out_);
    encoded = encode_line(line);
  } else {
    encoded = encode_line(line, reference);
    if (!block.line_fits(encoded.size())) {
      block.flush(out_);
      encoded = encode_line(line);
    }
  }
  block.add_line(std::move(encoded));
  std::swap(line, reference);
}
```

While `lines_per_band = 64` forces a flush at 64-line boundaries (so pairs of
brlaser bands *should* sum to 128 lines = one printer band), the variable
block line counts within each band are fragile. The cumulative alignment
depends entirely on the printer's actual band size being exactly 128 — if it
isn't, lines get silently dropped per rule #2.

## Issue 4: HL-1112E's extremely tight memory budget

At 600 DPI, an A4 page is ~4.3 MB of raw raster data (4962 × 7016 dots / 8).
Even though brlaser streams band-by-band, the HL-1112E's 1 MiB RAM leaves
very little headroom. Complex images with poor compression ratios at 600 DPI
can exhaust the printer's band buffer, causing it to silently abort the job.

The CAPS Universe documentation for the HL-11 series
(<https://caps-printing.org/documentation/brother-hl11_8c.html>) explicitly
notes:

> "The HL10 series of printers is partially very limited in their available
> resources.
> HL-1112: only 1 MiB RAM, can print at 300x300 DPI, can print at 600x600 DPI
> with encoding method 1030 and 1027"

The 300 DPI mode works for complex documents because the data volume is 4×
smaller and fits comfortably in the printer's memory. At 600 DPI with complex
content, the printer is right at the edge of its capacity, and any protocol
violation (like the missing last-band padding) is enough to push it over.

---

## Git history confirms this is a long-standing issue

The ChangeLog notes for v5 (2019-05-18):

> "Fixed problems with Brother HL-series printers in 600 dpi mode. Thanks to
> Onno Kortmann for the fix."

The actual fix was commit `779f71e` "Reduce max block size to 64 lines (#40)"
(March 2019), which reduced `max_lines_per_block_` from 128 to 64. This
forced block boundaries to align with 64-line band boundaries (instead of
128, which allowed misalignment for incompressible data because the 16350
byte limit closed blocks at ~26 lines mid-band).

Subsequent commits continued the effort:
- `172c3bf` "Don't compress across bands (#39)" — re-introduces non-delta
  encoding for the first line of a new block.
- `9d7ddda` "Arrange blocks in bands of 128 lines" (April 2020) — despite
  the title, the code still uses `lines_per_band = 64` with the XXX TODO.

**Notably, the last-band padding issue was never addressed** in any commit,
which is the most likely remaining cause of failures on memory-constrained
printers like the HL-1112E.

---

## Suggested fixes

### Fix 1: Pad the last band with empty lines (highest impact)

In `src/job.cc`, before the `fputs("1030M\f", out_)` line at line 132, add
logic to pad the final block with empty-line markers (`0xFF`, per
`src/line.cc:137,186`) up to the printer's expected band line count.

A minimal implementation would be something like:

```cpp
// Pad the last band with empty lines so the printer doesn't skip it.
// The printer silently skips bands with fewer than n lines.
const int band_size = 128;  // XXX: Brother's value; make configurable
int lines_in_last_band = ... // track lines added since last forced flush
while (lines_in_last_band % band_size != 0) {
  block.add_line(std::vector<uint8_t>(1, 0xFF));
  lines_in_last_band++;
}
block.flush(out_);
fputs("1030M\f", out_);
```

The driver would need to track how many lines have been added to the current
band (since the last `i % lines_per_band == 0` flush) to know how many
padding lines to add.

### Fix 2: Match Brother's 128 lines per band

Change `src/job.cc:113` from:

```cpp
const int lines_per_band = 64;
```

to:

```cpp
const int lines_per_band = 128;
```

This matches the official Brother driver and eliminates the XXX TODO.
However, this should be validated against the HL-1112E specifically, since
the XXX comment suggests the author tried 128 and it didn't work for some
printers (which is why they reduced to 64 in commit `779f71e`).

### Fix 3: Make the band size configurable per printer

Add a `band_line_count` field to the `page_params` struct in `src/job.h` and
set it per-printer in the PPD file (`brlaser.drv.in`). This would let the
HL-1112E use its actual band size (which may be 128 or some other value)
rather than relying on the hardcoded 64.

---

## References

- CAPS Universe: 1030 encoding method
  <https://caps-printing.org/documentation/EncoderC1030.html>
- CAPS Universe: encode_method_1030.h source reference
  <https://caps-printing.org/documentation/encode__method__1030_8h.html>
- CAPS Universe: Brother HL-11 series (HL-1112 specs)
  <https://caps-printing.org/documentation/brother-hl11_8c.html>
- brlaser issue #40: "Brother HL-1210W; 1200HQ mode"
  <https://github.com/pdewacht/brlaser/issues/40>
- brlaser issue #52: "HL-L2320D can't print this pdf"
  <https://github.com/pdewacht/brlaser/issues/52>
- brlaser issue #111: "HL-1110 DPI only printing certain PDF at 300DPI"
  <https://github.com/pdewacht/brlaser/issues/111>
- brlaser commit 779f71e: "Reduce max block size to 64 lines (#40)"
- brlaser commit 172c3bf: "Don't compress across bands (#39)"
- brlaser commit 9d7ddda: "Arrange blocks in bands of 128 lines"
