# Changelog

All notable changes to exif-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0 — 2026-11-03

### Changed

- Internal: ladders rewritten as `match`, the byte and thumbnail lookups answer an optional instead of -1, and errors are propagated with `!`. The `-1` that public functions and fields hold for no value is unchanged.

This release requires novo 0.23.0 or later.

## [0.1.0] — 2026-10-11

**The bodies.** Every `todo()` under `src/` is gone; the interface
0.0.1 published is now the thing itself. `novo test` is green over the
five suites, and `novo pkg publish --dry-run` reports 99% of 1540
instrumented lines covered, with the eight regions no test can reach
excused by a marker.

### Added

- `tests/exif_nv_edge_tests.nv` — thirty tests the four module suites
  do not reach. The hand-built TIFF is written byte for byte in BOTH
  byte orders and read in both; all twelve type codes are parsed and
  written back; `1/10` and `1/3` — which have no exact binary fraction
  — are carried through a parse and a write and compared as two
  integers and as the whole file byte for byte; all eight orientations
  are applied, inverted and composed, and the ones that are their own
  inverse are applied twice; every refusal arm is asserted for its
  code, its message and its position; and the container walks are
  driven off a truncated segment, a JFIF APP1 in place of an Exif one,
  a PNG chunk whose length runs past the end, a JPEG with no APP1 at
  all, and bytes that are no container.

### Changed

- The README's test-vector paragraph says five suites and names what
  the fifth adds.
- `novo.lint` reported 30 `char-literal` findings in `exifread` and
  `exifwrite`: `byte_or(source, i, 0 - 1) == 0x50` where `'P'` reads
  better.  All 30 are written as the character they are, which is what
  the rest of the shelf does, and lint is clean over every file this
  release touched.

### Known

- **The `+` join workaround.** A `Bytes + Bytes` join written inside a
  match arm leaks a cell whether or not the arm runs, so
  `without_range` — which cuts a range out of a buffer and puts the
  rest back — builds the answer into a `bytes.zeros(n)` with two
  `bytes.copy_into` calls instead of joining two slices.  The filing
  is
  `language/a-join-in-a-match-arm-is-leaked-even-when-the-arm-is-not-taken`,
  named at the site.
- **A tuple field read off a call.** `list.get(p.dropped, 0).0`
  generates a pointer where an `Int` is expected and the LLVM backend
  refuses it, so the suite binds the tuple first.  The filing is
  `codegen-llvm/a-tuple-field-read-off-a-call-expression-generates-a-pointer`,
  and the shape is a row in `../BRIEF.md` § 3.
- **A write into a TIFF is the metadata block and nothing else.** The
  strip and tile tags (0x0111, 0x0117, 0x0144, 0x0145) name picture
  data that lives outside the block, so they are dropped and reported:
  a caller that rewrites the EXIF of a TIFF and writes the answer back
  over the file has replaced the picture with its metadata.  A JPEG
  and a PNG keep every other segment and chunk, because there the
  block is a guest.
- **`to_tiff_block` leaves the thumbnail's place zeroed.** The
  thumbnail's bytes live in a buffer that function was not given.
  `write(source, d, ExifContainerTiff)` copies them in from the
  source.
- **`plan.dropped` names a directory by the enum's order** — 0 the 0th
  IFD, 1 the thumbnail's, 2 the Exif one, 3 GPS, 4 interoperability —
  as `(directory, tag)` pairs, which is the vocabulary the reader and
  the writer already share.
- **A NaN loses its payload.** Every `FLOAT` and `DOUBLE` NaN is
  written canonically (`0x7FC00000`, `0x7FF8000000000000`): a NaN is
  its exponent and its quiet bit, and a `Float` cannot carry the rest.
  A negative NaN keeps its sign.  `-0.0` survives exactly, because the
  sign is read from the bit pattern rather than from a comparison
  against zero.
- **`set_orientation` edits an entry that is already there.** A file
  whose 0th IFD carries no 0x0112 is answered `ExifNoExif` rather than
  given one: a caller editing a file in place should be told the tag is
  missing, not have it invented.  `exifdata.with_orientation` is how a
  caller adds one.
- **`maker_note_span` answers `None` for every file.** The note's
  bytes are read into the entry like any other `UNDEFINED` value, but
  `ExifEntry` does not carry where the value sat in the caller's
  buffer, so there is no span to hand back.  `value(d, ExifIfdExif,
  0x927C)` is the entry.
- **`gps_degrees` needs both the coordinate and its reference tag.**
  Either pair missing is `Ok(None)` rather than a guess, and a
  coordinate of fewer than three rationals is `None` too.
- **`date_fields` requires the shape the specification fixes**:
  nineteen bytes of `YYYY:MM:DD HH:MM:SS` with no time zone.  Any other
  shape, including one whose digits are not digits, is `None`.
- **Eight regions are excused by `// cov: skip`**, each with its
  reason on the marker: a double's exponent ceiling and its mantissa
  carry, a second `log2` correction no host reaches, the entry-count
  ceiling a two-byte field cannot exceed, the note comparison whose
  TIFF offset was already found, two range checks the walk that found
  the range has already made, and the third arm of a three-arm enum.

## 0.0.2 — 2026-10-25

A file that names a sibling module's type, function or constant now has the `use` line for that module (E2033, E2003).  No signature and no behaviour changed.

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
