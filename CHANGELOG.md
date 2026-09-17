# Changelog

All notable changes to exif-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## [0.0.1] — 2026-09-17

**The interface, published before anyone implements it.** Every public
type and function carries its full signature, its effect row and its
doc comment; every body is `todo()`; the release is recorded
`implemented = false`.

### Added

- `exifvalue` — the load-bearing interface. A RATIONAL is
  `ExifRational { numerator, denominator }` and STAYS two integers.
  `1/250` is what a camera displays, what a photographer writes and
  what the file contains, and a library that hands back a `Float` has
  destroyed it: `10/2000` and `1/200` are the same number and
  different files. `format_ratio` does not reduce, `to_float` is asked
  for by name, and a zero denominator is a refusal rather than an
  infinity. `ratio_from_float` is the continued-fraction convergent,
  because a writer that multiplies by a million writes
  `4000/1000000` where a camera writes `1/250`. Every `ExifValue` arm
  holds a LIST, because a TIFF entry carries a count and the format
  has no scalar case.
- `exiftag` — the table, keyed by DIRECTORY AND NUMBER. Tag 0x0001 is
  `GPSLatitudeRef` in the GPS IFD and `InteroperabilityIndex` in the
  Interoperability IFD, so a table keyed by number alone answers the
  wrong name for a GPS file. `tag_at` answering `None` is not a
  refusal: the reader keeps an unknown entry with its number, its type
  and its bytes, because a library that dropped what it did not
  recognise would destroy metadata on every save.
  `is_offset_fragile` is true for MakerNote, which is the fact a
  writer has to consult.
- `exiforient` — the eight values as named transforms, with the same
  names image-io-nv gave them so the two read alike in one program.
  `swaps_dimensions` is the one derived fact a caller needs before it
  has decoded anything. `mirrors` is true for the four most
  implementations forget. `after_applying` says what a tool that
  rotated the pixels writes back.
- `exifdata` — the parsed metadata. The thumbnail and the MakerNote
  are SPANS into the caller's own buffer, not copies: a thumbnail is
  10 to 40 kilobytes and the overwhelming majority of callers wanted
  the exposure time. `orientation` answers `ExifUpright` for a file
  that carries no tag, and `has_orientation` is there for the few
  callers that must tell absent from 1. `gps_degrees` is the one
  derived value computed here, because the sign lives in a separate
  tag from the coordinate and a reader that takes one and not the
  other puts every southern photograph in the northern hemisphere.
  `date_fields` hands back six numbers rather than an instant,
  because the field carries no time zone and therefore names no
  moment.
- `exifread` — three containers, each with a named reader and a
  sniffing one. `read` answers `?ExifData`, because a photograph with
  no metadata is an ordinary photograph. `read_report` carries the
  non-fatal findings beside the data, so a viewer ignores what a
  validator prints. `tiff_offset` is public because every offset in
  the structure is relative to the TIFF header and that is the fact
  that produces the most wrong EXIF code.
- `exifwrite` — `plan` before `write`. Rewriting EXIF is not safe: a
  MakerNote that moves stops parsing, and a JPEG APP1 segment cannot
  exceed 65533 bytes. The plan says both before anything is written.
  `strip` is a separate function from writing empty metadata, because
  the two differ: one removes the block and the other leaves an empty
  one.
- `exiffault` — fifteen refusals, each naming its tag and its offset.
  `is_fatal` separates the four that make the structure unreadable
  from the mismatches that spoil one entry, which is what lets a
  reader read the large share of real photographs that carry a type
  disagreeing with the specification.

### Known

- `novo test` is red, and that is the release's expected state: every
  assertion in the API suite reaches `not implemented:
  exif-nv.<module>.<fn>`.
- **NO DEPENDENCIES, deliberately.** EXIF is a TIFF header, a chain of
  directories and a table of tag numbers; reading it needs no image
  codec, no colour space and no calendar.
- **HEIF and AVIF are not read.** Both carry EXIF in an ISO base media
  file format box, which is a container walk this package does not do.
  The plan's one-line note named HEIF; the fuller brief named JPEG,
  PNG and the raw TIFF, and those are what shipped. HEIF is a later
  release and the README says so.
- **A MakerNote is not parsed**, only located and flagged as
  position-dependent.
