# exif-nv

EXIF is the metadata a camera writes into a photograph: the exposure
time, the lens, the date, the orientation, sometimes the place. It is
specified in
[EXIF 2.32](https://www.cipa.jp/std/documents/e/DC-008-Translation-2019-E.pdf)
(CIPA DC-008-2019, also published as JEITA CP-3451D), and its structure
is [TIFF 6.0](https://www.adobe.io/open/standards/TIFF.html). The
reference implementations this package is measured against are the Rust
crate [`kamadak-exif`](https://docs.rs/kamadak-exif) and the Python
package [`piexif`](https://piexif.readthedocs.io/). This package reads
EXIF out of a JPEG, a PNG or a TIFF and writes it back.

**Status: NOT IMPLEMENTED — interface only.** Every function is
declared with its full signature, but every body is a `todo()` that
panics when called. The package is published so its design can be
reviewed and depended on before it is implemented. Version 0.1.0 will
be the first working release.

## What EXIF is

EXIF is a TIFF file with no image in it, carried inside another file. It
begins with an eight-byte **TIFF header**: two bytes saying which end of
a number comes first (`II` for least significant byte first, `MM` for
most significant), the number 42, and the offset of the first directory.

An **image file directory**, or **IFD**, is a count followed by that
many twelve-byte **entries**, followed by the offset of the next
directory. An entry is a **tag number**, a **type**, a **count**, and
then either the value itself or an offset to it — the entry has four
bytes for the value, and anything longer lives elsewhere and is pointed
at.

Every offset in the structure is counted from the start of the TIFF
header. Not from the start of the file, and not from the start of the
block that contains it.

EXIF uses five directories.

| Directory | Reached by | Holds |
| --- | --- | --- |
| 0th | The TIFF header | The main image: make, model, orientation, resolution |
| 1st | The 0th's next-directory offset | The thumbnail, and where its bytes are |
| Exif | Tag 0x8769 in the 0th | Exposure, aperture, ISO, dates, the lens |
| GPS | Tag 0x8825 in the 0th | Where the picture was taken |
| Interoperability | Tag 0xA005 in the Exif IFD | Which interoperability rules the file claims |

Nesting through a tag is what makes EXIF a tree rather than a chain. A
tag number means nothing without its directory: tag 0x0001 is
`GPSLatitudeRef` in the GPS IFD and `InteroperabilityIndex` in the
Interoperability IFD.

TIFF 6.0 defines twelve types. Two of them matter more than the rest.

| Code | Type | Bytes each |
| --- | --- | --- |
| 1 | BYTE | 1 |
| 2 | ASCII | 1 |
| 3 | SHORT | 2 |
| 4 | LONG | 4 |
| 5 | RATIONAL | 8 |
| 7 | UNDEFINED | 1 |
| 10 | SRATIONAL | 8 |

A **RATIONAL** is two unsigned 32-bit integers, a numerator and a
denominator. **SRATIONAL** is the signed form. EXIF uses them for every
number a photographer reads: the exposure time, the aperture, the focal
length, each part of a GPS coordinate.

The metadata travels in one of three **containers**. In a JPEG it is an
APP1 marker segment whose payload begins with the six bytes `Exif`,
NUL, NUL. In a PNG it is an `eXIf` chunk, which carries the TIFF
structure with no such prefix. In a TIFF it is the file.

## Install

```
novo pkg add exif-nv
```

## Example

```novo
use std.bytes
use exifread
use exiforient
use exifvalue
use exifdata

fn main() [io]
    // The file's bytes, which the caller already holds. This package
    // opens nothing.
    let src = bytes.zeros(64)

    match exifread.read(src, exifread.default_options())
        Err(e) => println("cannot read: ${e.message()}")
        Ok(found) =>
            match found
                None => println("no metadata")
                Some(d) =>
                    // Orientation is one of eight named transforms, and
                    // a file that carries none is upright.
                    println(exiforient.orientation_name(exifdata.orientation(d)))

                    // The exposure time stays the fraction the file
                    // holds: `1/250`, not `0.004`.
                    match exifdata.value(d, ExifIfdExif, 0x829A)
                        None => println("no exposure time")
                        Some(v) =>
                            match exifvalue.first_ratio(v)
                                Some(r) => println(exifvalue.format_ratio(r))
                                None    => println("not a rational")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: exif-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `exiffault` | Every reason a read or a write fails, with the tag and the offset it was about. |
| `exifvalue` | The twelve TIFF types, the two byte orders, and the rational forms with their exact arithmetic. |
| `exiftag` | The five directories and the table of what a tag number means in each. |
| `exiforient` | The eight orientations, as named transforms. |
| `exifdata` | The parsed metadata: entries, accessors, the thumbnail and the MakerNote as spans, and the GPS position. |
| `exifread` | Finding the block in a JPEG, a PNG or a TIFF, and reading it. |
| `exifwrite` | Writing it back, and saying first what that would cost. |

## How to choose an entry point

**`exifread.read` sniffs the container.** Use it for a file whose
format you do not already know.

**`exifread.read_jpeg`, `read_png` and `read_tiff` do not.** Use one
when you already know what you have: it refuses bytes that are not that
container, where `read` would silently accept a different one.

**`exifread.read_orientation` reads one tag.** Use it in a thumbnail
pipeline. It follows the 0th IFD only and allocates nothing else.

**`exifread.read_report` answers the findings as well as the data.**
Use it in a validator. `read` discards them, which is right for a
viewer.

**`exifwrite.plan` answers what a write would cost, and `write` does
it.** Call the first before the second on any file you did not create.

**`exifwrite.set_orientation` edits two bytes in place.** Use it when
that is the only change: nothing moves, so nothing can be lost.

**`exifwrite.strip` removes the metadata entirely.** Use it before
publishing. Writing empty metadata is not the same thing: it leaves an
empty block behind.

## The rules a user needs

1. **A rational is two integers and must stay two integers.** The file
   says `1/250`. That is what a camera displays and what a photographer
   writes. `10/2000` and `1/200` are the same number and different
   files, and a tool that normalises one into the other has edited the
   file. `exifvalue.to_float` converts when a caller asks by name.

2. **A denominator of zero is legal in the file.** It is
   `ExifZeroDenominator` when a caller asks for the number, not an
   infinity.

3. **Every offset is relative to the TIFF header.** Not to the file and
   not to the `Exif\0\0` identifier (EXIF 2.32 section 4.5.4).
   `ExifData.tiff_span` carries where that header was.

4. **A tag number means nothing without its directory.** Tag 0x0001 is
   two different tags in two directories (section 4.6.6).

5. **Orientation has eight values and four of them mirror.** Handling
   1, 3, 6 and 8 and ignoring 2, 4, 5 and 7 is the usual bug; scanners
   and some phone front cameras write the others (section 4.6.4).

6. **A tool that rotates the pixels must clear the tag.** A file whose
   pixels were straightened and whose tag still says `rotate-90` is
   shown sideways by the next program that opens it.

7. **Moving a MakerNote can destroy it.** Several vendors wrote
   absolute file offsets inside their private note. If the Exif IFD
   moves, the note points at nothing and no error is raised anywhere.
   `exifwrite.plan` reports whether it would move.

8. **A JPEG can carry 65533 bytes of EXIF and no more.** The APP1
   segment's length field is two bytes. A file with a large thumbnail
   is already near that ceiling.

9. **A date field names no moment.** `DateTimeOriginal` is
   `"YYYY:MM:DD HH:MM:SS"` with no time zone (section 4.6.5). EXIF 2.31
   added `OffsetTimeOriginal` and almost nothing writes it.
   `exifdata.date_fields` hands back the six numbers.

10. **GPS needs four tags.** The coordinate is three rationals and the
    sign is in a separate reference tag. A reader that takes the first
    and not the second puts every southern photograph in the northern
    hemisphere. `exifdata.gps_degrees` reads all four.

11. **An unknown tag is kept.** A private tag, a vendor's tag and a tag
    from a newer EXIF than this table all survive a read and a write. A
    library that dropped them would destroy metadata on every save.

## What is not included

- **Parsing a MakerNote.** Every vendor's is different, most are
  undocumented, and several are position-dependent.
  `exifdata.maker_note_span` hands back where it is, and
  `exiftag.is_offset_fragile` says that moving it is dangerous.
- **XMP and IPTC.** Both are other metadata standards that travel in
  other segments of the same files. Neither is EXIF, and each is its
  own package.
- **HEIF and AVIF.** Both carry EXIF in an ISO base media file format
  box, which is a container this package does not walk. JPEG, PNG and
  TIFF are the three it reads. HEIF is a later release.
- **Decoding the image.** This package reads metadata. image-nv,
  png-nv and gif-nv decode pictures.
- **Turning a date into an instant.** The field carries no time zone,
  so nothing can. chrono-nv and tz-nv are the packages that build one
  when a caller supplies the offset.
- **Turning a thumbnail into pixels.** `exifdata.thumbnail_bytes`
  hands back the bytes; they are usually a JPEG, and image-nv decodes
  them.
- **Repairing a broken file.** A structure that cannot be walked is
  refused. Recovering entries from a damaged directory is a different
  program with different rules.

## Related packages

**image-io-nv** decides what can decode a file and reads the
orientation out of its head. It names this package as the one that
reads the rest of the metadata, and its `IioOrientation` has the same
eight values under the same names.

**image-nv**, **png-nv** and **gif-nv** decode pictures. This package
never looks at one.

**chrono-nv** and **tz-nv** turn a date and an offset into an instant.

## Test vectors

The suite is written against the specifications themselves.

- **TIFF 6.0 section 2** fixes the twelve type codes and their sizes,
  the two byte-order marks, the magic number 42, and the four-byte
  inline value field. Each is asserted directly.
- **EXIF 2.32 section 4.6.4** fixes the eight orientations, and each of
  the three derived facts — whether the dimensions are exchanged,
  whether the image is mirrored, how far it turns — is asserted for all
  eight.
- **EXIF 2.32 section 4.6.6** fixes the tag tables. The two tags
  numbered 0x0001 in different directories are asserted as two
  different tags.
- **EXIF 2.32 section 4.5.4** fixes where the TIFF header sits in a
  JPEG APP1 segment, which is what every offset is relative to.
- The rational arithmetic is asserted on the fractions cameras actually
  write: `1/250`, `10/2000`, `37/1`.

Today every one of those assertions reaches a `not implemented` panic.

## Implementation status

| Area | Status |
| --- | --- |
| TIFF types, byte orders, rationals | declared, not implemented |
| The five directories and the tag table | declared, not implemented |
| Orientation | declared, not implemented |
| Reading from JPEG, PNG and TIFF | declared, not implemented |
| Writing back, with a plan first | declared, not implemented |
| Stripping | declared, not implemented |
| MakerNote parsing | not declared |
| HEIF and AVIF containers | not declared — a later release |
| XMP and IPTC | not declared — other standards |

## Licence

Apache-2.0. See [LICENSE](LICENSE).
