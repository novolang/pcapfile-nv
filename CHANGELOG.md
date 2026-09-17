# Changelog

All notable changes to pcapfile-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.1 — 2026-09-17

**The interface, published before anyone implements it.** Every public
type and function carries its full signature, its effect row and its
doc comment; every body is `todo()`; the release is recorded
`implemented = false`.

### Added

- `pcapread` — the load-bearing interface, and two decisions in it.
  **The first is that `feed` drains.** Every codec of this shape
  answers one event per call and offers a second call for the next one;
  that is wrong here, because a payload event names octets INSIDE THE
  CHUNK THE CALLER JUST PASSED, and a second call producing another
  such event would be naming octets it was never given. So `feed`
  returns every event the chunk completed, the spans all point into
  that one chunk, and they are valid until the next call. The reader
  holds a partial header and the body of a block it decodes whole, both
  bounded by `PcapLimits`; it never holds a payload, so a packet split
  across three chunks is three `PcapBytes` events. **The second is that
  both formats drain into one stream of packets.** A classic record, an
  Enhanced Packet Block and a Simple Packet Block all arrive as
  `PcapPacketStart`, some `PcapBytes` and a `PcapBodyDone`, so a
  program that counts, filters or forwards packets is written once and
  reads both. What differs is in `PcapPacket.origin`, for the programs
  that must care. `PcapBodyDone` carries the options because an
  Enhanced Packet Block writes them after its octets, which is a fact
  about the format and not a convenience.
- `pcaphdr` — the four magic numbers as four arms, because they carry
  TWO independent facts. `magic_order` and `magic_resolution` are two
  functions over one value: a reader that matched one number and then
  decided the byte order by comparing against its own machine's word
  gets both answers from one comparison, and both are wrong on half the
  files in the world. `PcapSpan` carries both coordinates — where the
  octets are in the chunk, where they are in the stream — because a
  caller slices in the first and reports in the second. `PcapPacket` is
  the one packet value, with `has_timestamp` and `length_is_exact`
  saying what a Simple Packet Block does not know.
- `pcapts` — a timestamp is a count of ticks and does not carry its own
  tick size, so every conversion takes the resolution. `PcapResolution`
  has a decimal arm and a binary arm rather than one exponent, because
  `if_tsresol`'s top bit picks the base and a reader that ignored it
  reads 2^-16 seconds as 10^-16. `is_exact` says whether an instant
  survives a resolution before the conversion loses it, and `to_unix`
  truncates rather than rounding so that a rewritten capture stays in
  order.
- `pcapbyte` — there is no host byte order in this package. Every read
  and every write takes a `PcapByteOrder` that came from the file.
  `read_u32` never answers a negative number and `read_u64` answers
  `None` for a value a signed `Int` cannot hold, which is the
  0xFFFFFFFFFFFFFFFF an unknown section length is written as.
- `pcaplink` — six named link types and `PcapLinkOther` for the rest,
  because the registry has over three hundred entries and an enum that
  held them all would turn every new one into a build failure for a
  consumer that only wanted to skip it.
  `family_is_in_file_order` is the quiet rule: a LINKTYPE_NULL frame's
  leading four octets are an address family in the FILE's byte order,
  not in network order.
- `pcapngopt` — `option_name` takes the block kind as well as the code,
  because code 2 is `shb_hardware`, `if_name`, `epb_flags` and
  `isb_starttime` in the four blocks that define one. A lookup that
  took the code alone would hand its callers that collision, and
  reading a four-octet flags word as an interface name gives a short
  string of control characters that nothing complains about.
  `padding_for` is the 32-bit alignment rule written once, for both
  formats.
- `pcapngblock` — the block frame, and the three blocks that hold
  something other than a packet. `PcapngSection.length` is `?Int` so
  that the -1 meaning "the writer did not know" cannot be read as a
  number. `PcapngInterface` applies the specification's defaults —
  microseconds, an offset of zero, no name — because that is what an
  absent option MEANS, and a caller that guessed differently would
  misdate a whole capture.
- `pcapwrite` — `PcapOutPacket` has no captured length. It holds the
  octets, and their count IS the captured length wherever one is
  written, so the commonest way a hand-written capture writer produces
  an unreadable file is unwritable here. `pcapwrite.block` is the only
  function that emits pcapng block framing, so the trailing length copy
  cannot be left out of a block somebody adds later.
  `pcapwrite.simple_packet` refuses a truncated packet, because a
  Simple Packet Block has nowhere to record the truncation and a reader
  would return its padding as frame octets.
- `pcaperror` — sixteen reasons, every one carrying the offset it
  happened at, because a capture file is repaired by cutting at an
  offset. `is_truncation` separates the file that stopped early from
  the file that says something impossible, and `is_caller_fault`
  separates the calling program's own mistakes from a stranger's file.

### Known

- `novo test` is red, and that is the release's expected state: every
  assertion in the six suites reaches `not implemented:
  pcapfile-nv.<module>.<function>`.
- **The package takes no dependencies, and one of the three candidates
  was refused by a language rule rather than by design.** ipaddr-nv
  would have held a Name Resolution Block's address as an `Ipv4` or an
  `Ipv6`. Both are `@value` structs, a record carries a list of names
  beside its address and so cannot be one, and a boxed struct may not
  hold an unboxed one (E2015). The record carries the octets the block
  wrote, and the README says where to hand them.
- **There is no device claim and no `tests/embedded_probe.nv`.** Every
  module would compile for one — nothing here allocates unboundedly and
  `pcapread.embedded_limits` exists for it — but a microcontroller is
  not where capture files are read, and a claim nobody built is a claim
  nobody checks.
- **The Decryption Secrets Block and the Systemd Journal Export Block
  are not decoded.** Both reach a caller as `PcapSkipped`, naming their
  type and their length.
