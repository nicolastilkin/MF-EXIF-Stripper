# MF - EXIF Stripper

A single HTML file that removes EXIF, GPS, XMP, IPTC, PNG text chunks and vendor
metadata from JPEG and PNG images, in the browser, without re-encoding the
picture.

Open `mf-exif-stripper.html` by double-clicking it. There is no server, no build
step, no install and no network call — it works from `file://`, from a network
share, or from a SharePoint library behind a strict proxy. Files are read into
memory, rewritten in memory, and handed back as downloads. Nothing is uploaded
or stored; close the tab and it is all gone.

## What it does to a file

It never decodes the image. It parses the container — the JPEG marker segment
chain, the PNG chunk sequence — works out which blocks are metadata, and writes
out the survivors in order. The JPEG entropy-coded scan and the PNG `IDAT`
stream are copied across byte-for-byte, so the output decodes to pixels
identical to the input.

This is the difference from the usual browser approach, which draws the image to
a `<canvas>` and re-exports it. That does remove metadata, but it also
re-compresses the picture: generational JPEG loss on every pass, an unrelated
file size, alpha flattened when a PNG goes through a JPEG encoder, and colour
shifts on anything that was not already sRGB.

## What it removes and what it keeps

**JPEG**

| Block | Action |
|---|---|
| `APP1` Exif/TIFF, including the IFD1 thumbnail | removed |
| `APP1` XMP and extended XMP | removed |
| `APP13` Photoshop IRB / IPTC | removed |
| `APP11` JUMBF / C2PA provenance | removed |
| `APP12` Ducky, `APP2` MPF, `APP2` FPXR, `APP0` JFXX thumbnail | removed |
| `COM` comments | removed |
| Undocumented `APPn` segments | removed (option 4) |
| `APP2` `ICC_PROFILE` | kept (option 1) |
| `APP0` JFIF density | kept (option 3) |
| `APP14` `Adobe` | always kept |
| `SOI`, `DQT`, `DHT`, `SOFn`, `DRI`, `DAC`, `SOS` + scan, `EOI` | always kept |

**PNG**

| Chunk | Action |
|---|---|
| `eXIf`, `tEXt`, `zTXt`, `iTXt`, `tIME` | removed |
| Private / unregistered chunks (`mkBF`, `prVW`, `caBX`, vendor blocks) | removed (option 4) |
| `iCCP` | kept (option 1) |
| `pHYs`, `oFFs`, `sCAL` | kept (option 3) |
| `IHDR`, `PLTE`, `IDAT`, `IEND` | always kept |
| `tRNS`, `gAMA`, `cHRM`, `sRGB`, `sBIT`, `bKGD`, `hIST`, `sPLT`, `sTER` | always kept |
| `cICP`, `mDCv`, `cLLi` (HDR signalling) | always kept |
| `acTL`, `fcTL`, `fdAT` (APNG) | always kept |

Two of those are pinned because deleting them breaks the file rather than
cleaning it. `APP14 Adobe` declares the YCCK/CMYK colour transform, and an
Adobe-authored CMYK JPEG comes back inverted without it. `acTL`/`fcTL`/`fdAT`
are ancillary by PNG's naming rules, so a stripper that keeps only critical
chunks flattens every APNG to a single still frame.

Unsupported containers — HEIC/HEIF, WebP, TIFF, GIF, AVIF — are detected by
signature and rejected with a reason rather than mangled.

## The four options

**Keep ICC colour profile.** The profile holds no personal data, and dropping it
makes wide-gamut and CMYK sources render wrong in colour-managed viewers.

**Preserve orientation flag.** Phones write the sensor's landscape frame and set
EXIF tag `0x0112` to say which way up it goes; delete the whole EXIF block and a
portrait photo displays sideways. With this on, the tool writes a fresh 26-byte
TIFF block holding that one tag and nothing else — a 36-byte `APP1` segment on
JPEG, a 38-byte `eXIf` chunk after `IHDR` on PNG. No camera, timestamp, GPS or
serial number survives it.

**Keep density / DPI.** `pHYs` and the JFIF density fields drive physical print
size when the image is placed in InDesign, Word or Acrobat.

**Drop unknown / vendor blocks.** Undocumented `APPn` segments and private PNG
chunks are where Samsung, Apple, Fireworks and C2PA provenance data live. Turn
it off to keep anything not conclusively identified as metadata.

## What you get out

Drop in one file or a folder's worth. Each one gets a report in four views:

- **Findings** — GPS as decimal degrees, camera and lens serial numbers, owner
  and artist names, image unique IDs, capture timestamps, and embedded IFD1
  thumbnails, which are worth flagging separately because several editors do not
  regenerate them after a crop or a redaction.
- **Metadata read** — the full tag dump: byte order, tag count across IFDs,
  every recognised EXIF and GPS tag, PNG text chunk contents.
- **Structure** — every marker or chunk, its byte count, and whether it was
  kept, dropped or added.
- **Audit JSON** — the same as a machine-readable record: bytes in and out per
  file, every block removed and retained with sizes, the GPS fix found, the
  options in force, and a UTC timestamp.

Downloads are per file, or all cleaned files at once as a `.zip`. The archive
uses stored entries rather than deflate, since JPEG and PNG payloads are already
compressed and storing them keeps each member recoverable byte-for-byte.

Two geotagged sample images are generated on load, so the tool is doing
something the moment it opens.

## Limitations

- JPEG and PNG only.
- `zTXt` chunks are removed but not decoded for the preview; the keyword is
  shown and the payload listed as compressed.
- PNG has no native orientation concept, so the rebuilt `eXIf` chunk is honoured
  by viewers that read PNG EXIF and ignored by those that do not. On JPEG the
  flag is universal.
- Orientation is preserved as a flag, not baked into the pixels — that would
  require re-encoding, which is the thing this avoids.
- Processing is synchronous, so a queue of very large files will briefly block
  the tab.
- Filenames and visible content are untouched. `IMG_20190714_Zaventem_site.jpg`
  still says what it says, and stripping metadata does not redact a whiteboard
  or a badge in the frame.
