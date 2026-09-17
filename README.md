# pcapfile-nv

A packet capture file holds network packets that were recorded as they
passed an interface, together with the time each one was seen. Two file
formats hold them. The older is **pcap**, the format libpcap and
`tcpdump` have written since 1994. The newer is **pcapng**, specified by
the IETF draft
[Introduction and Description of PcapNg](https://www.ietf.org/archive/id/draft-tuexen-opsawg-pcapng-06.html),
which is what Wireshark writes today. This package reads and writes both
in novo-lang.

The reference implementations are the Rust crate
[pcap-file](https://docs.rs/pcap-file) and the Python library
[dpkt](https://dpkt.readthedocs.io/).

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What a capture file is

A capture file is a header followed by packets. Each packet carries the
octets that were saved, how many octets were on the wire before the
capture cut it short, and a timestamp.

**The link type** says what the saved octets are. The same twenty
octets are an Ethernet frame under one link type, an IPv4 datagram
under another, and a Linux cooked-capture header under a third. Nothing
in the octets says which. The numbers come from
[tcpdump.org's link-layer header type registry](https://www.tcpdump.org/linktypes.html).

**The snapshot length** is the most octets the capture was willing to
save from any one packet. A packet longer than it was **truncated**:
its saved octets are a prefix of the packet that was on the wire.

**The timestamp** is an integer, and the size of one tick of it is
recorded elsewhere in the file. A count without its tick size is
meaningless.

### Classic pcap

A classic pcap file is a 24-octet header and then packet records, back
to back, with no padding.

The header begins with four octets called the **magic number**, and
there are exactly four values they may have. The number says whether
the timestamps are in microseconds or nanoseconds. The order the octets
lie in says how every other integer in the file is laid out.

| Octets | Byte order | Timestamp |
| --- | --- | --- |
| `A1 B2 C3 D4` | big-endian | microseconds |
| `D4 C3 B2 A1` | little-endian | microseconds |
| `A1 B2 3C 4D` | big-endian | nanoseconds |
| `4D 3C B2 A1` | little-endian | nanoseconds |

The rest of the header is the version, which is 2.4; a correction in
seconds from the timestamps to UTC, which is zero in every file any
current tool writes; a claimed accuracy, which is also zero; the
snapshot length; and one link type for the whole file.

Each record is a 16-octet header — the whole seconds, the fraction, the
saved length and the original length — and then the saved octets.

### pcapng

A pcapng file is a sequence of **blocks**. Every block is a 32-bit
type, a 32-bit **block total length**, a body, padding to a 32-bit
boundary, and the same total length again at the end. The total length
counts all of that.

The second copy of the length is what lets a reader start at the end of
a file and walk backwards. The first copy is what lets a reader step
over a block whose type it has never seen.

| Block | What it holds |
| --- | --- |
| Section Header (SHB) | Begins a section: its byte order, its version, and its length when the writer knew it |
| Interface Description (IDB) | One interface: its link type, its snapshot length, its name, its timestamp resolution |
| Enhanced Packet (EPB) | A packet: its interface, its timestamp, both lengths, the octets, and options |
| Simple Packet (SPB) | A packet: the original length and the octets, and nothing else |
| Name Resolution (NRB) | Addresses and the names they resolved to when the capture was made |
| Interface Statistics (ISB) | How many packets an interface received and dropped |
| Custom (CB) | Private data, introduced by the enterprise number of whoever defined it |

A file is a sequence of **sections**, each beginning with a Section
Header Block, because appending one capture to another is concatenating
the files. Each section has its own byte order and its own interfaces,
and interface identifiers start at zero again in each one.

Most blocks may end with a list of **options**. An option is a 16-bit
code, a 16-bit length, the value, and padding to a 32-bit boundary. The
length counts the value and not the padding. An option's code means
something different in each kind of block: code 2 is the capturing
hardware in a Section Header Block, the interface name in an Interface
Description Block, and the packet's flags in an Enhanced Packet Block.

The option that carries an interface's timestamp resolution is
`if_tsresol`, one octet whose top bit picks a power of two over a power
of ten. When it is absent the resolution is microseconds.

## Install

```
novo pkg add pcapfile-nv
```

## Example

```novo
use pcaphdr
use pcaplink
use pcapread
use pcapts

fn main() [io]
    // The front of a capture file the caller opened. This package
    // opens nothing: the octets come from a file, a pipe or a socket,
    // and which one is the caller's business.
    let chunk = list.zeros(24)

    // A reader that does not yet know which format it is reading. The
    // first four octets settle it.
    let r = pcapread.reader(pcapread.default_limits())

    // Hand it the octets. It hands back every event they completed.
    match pcapread.feed(r, chunk)
        Err(e)  => println("not a capture file: ${e.message()}")
        Ok(fed) => describe(fed.reader)

// What the reader knows once the front of the file has arrived.
fn describe(r: PcapReader) [io]
    match pcapread.format(r)
        None    => println("not enough octets yet")
        Some(f) => println("this is a ${pcaphdr.format_name(f)} file")

    // The link type is the only thing that says what a packet's octets
    // are.
    match pcapread.link_type(r, 0)
        None    => println("no link type yet")
        Some(l) => println("its packets are ${pcaplink.link_name(l)}")

    // How big one tick of a packet's timestamp is.
    match pcapread.resolution(r, 0)
        None      => println("no timestamp resolution yet")
        Some(res) => println("${pcapts.ticks_per_second(res)} ticks per second")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: pcapfile-nv.<module>.<function>` panic. The tests are
the specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `pcaperror` | Every way a file can be refused, each carrying the offset it happened at. |
| `pcapbyte` | Which way round a file's integers lie, and the reads and writes that take that as an argument. |
| `pcaplink` | The link types, named and numbered, and what is known about each one's link-layer header. |
| `pcapts` | Timestamp resolutions, tick counts, and the conversions between a count and a Unix instant. |
| `pcaphdr` | The four magic numbers, the classic file header, and the packet value both formats produce. |
| `pcapngopt` | A pcapng option, and what its code means in each kind of block. |
| `pcapngblock` | The pcapng block frame, the section, the interface, and the blocks that hold something other than a packet. |
| `pcapread` | The reader: octets in, packets and blocks out. |
| `pcapwrite` | The writer: a header and packets in, octets out. |

## How to choose an entry point

**`pcapread.feed` is the way in for a file of any size.** The caller
reads whatever it can and hands it over; the reader hands back every
event those octets completed. It reads both formats and does not have
to be told which.

**`pcaphdr.parse_file_header` reads a classic header on its own**, for
a caller that already holds the front of a file and wants to know what
it is before deciding anything else.

**`pcapwrite.classic_file` and `pcapwrite.ng_file` write a whole
capture in one call**, for a caller that holds one in memory.

**`pcapwrite.classic_header` with `pcapwrite.classic_record`, and
`pcapwrite.section_header` with `pcapwrite.enhanced_packet`, write a
capture of any size**, one packet at a time, holding one packet at a
time.

## The rules a user needs

1. **A payload span points into the chunk that was just fed, and is
   valid until the next call.** `pcapread.feed` returns `PcapBytes`
   events whose `chunk_at` and `length` index the list the caller
   passed to that same call. The octets are never copied. A caller that
   keeps a span past its next call is reading whatever is there now.
2. **One packet may arrive as several `PcapBytes` events.** How many is
   an artefact of how the caller read its file. The events between a
   `PcapPacketStart` and its `PcapBodyDone` are one packet.
3. **The byte order comes from the file, never from the machine reading
   it.** Every integer read and every integer written takes a
   `PcapByteOrder` argument. A reader that used its own machine's order
   reads every foreign capture backwards, and a snapshot length of
   262144 arrives as 4096.
4. **The four classic magic numbers are two numbers, each written two
   ways.** Matching only `A1 B2 C3 D4` and `D4 C3 B2 A1` reads every
   nanosecond capture as microseconds. The timestamps stay ordered and
   are a thousand times too large.
5. **A timestamp is a count of ticks and does not carry its own tick
   size.** `pcapts.to_unix` takes the resolution, which comes from
   `pcapread.resolution` for the interface the packet was captured on.
6. **A pcapng interface may add a constant to its timestamps.**
   `if_tsoffset` is a number of seconds, zero when absent, and
   `pcapread.time_offset` answers it. It exists so that a capture of a
   long run can use a small counter, and the writers that use it choose
   large numbers.
7. **Interface identifiers are per-section.** An Enhanced Packet Block
   naming interface 1 means the second Interface Description Block of
   its own section. `pcapread.interfaces` is emptied at every Section
   Header Block.
8. **A section header says its length is unknown by writing -1**, which
   `PcapngSection.length` reports as `None`. A capture still being
   written cannot know its length, so this is the common case.
9. **A pcapng block carries its total length twice, and the two must
   agree.** A disagreement is `PcapLengthMismatch`: a file whose
   forward and backward walks disagree is a file where one of them is
   reading different blocks.
10. **An Enhanced Packet Block's options arrive after its octets**,
    because that is where the format writes them. They reach the caller
    on `PcapBodyDone`. A program that needs a packet's flags before it
    looks at the packet has to hold the packet itself.
11. **An option's code means different things in different blocks.**
    `pcapngopt.named` takes the block kind as well as the option, so a
    lookup cannot read one option as another.
12. **A Simple Packet Block has no timestamp and no saved length.**
    `pcaphdr.has_timestamp` is false for one, and its zero timestamp is
    not an instant. Its saved length is derived from the block's total
    length, which includes the padding, so it is exact only when the
    packet was not truncated.
13. **A classic pcap record is not padded.** Padding one, as a pcapng
    block is padded, produces a file that every reader walks off the end
    of.
14. **A truncated packet is a prefix.** `pcaphdr.is_truncated` is true
    when the saved length is below the original length. Reassembly, a
    checksum and a payload read all give the wrong answer on one, and
    traffic volume is counted from the original length.
15. **`pcapread.finish` is what tells a complete file from a damaged
    one.** A stream that ends between blocks answers `PcapStreamEnd`; a
    stream that ends inside one answers `PcapTruncated`, which is what a
    full disk and a killed `tcpdump` leave behind.
16. **`PcapTruncated` is the only refusal that means the rest of the
    file was fine.** `pcaperror.is_truncation` answers which, and every
    refusal carries the offset it happened at.
17. **The limits are the caller's.** `PcapLimits` bounds the saved
    length a record may claim, the size of a block the reader buffers,
    the number of options in one block and the number of interfaces in
    one section. A length field in a capture file is a number from
    somewhere else.
18. **The captured length is not a field a caller writes.**
    `PcapOutPacket` holds the octets, and their count is the captured
    length wherever one is written. The original length is stated
    separately and must not be below it.
19. **A LINKTYPE_NULL frame's leading four octets are in the file's
    byte order.** They are an address family written by the capturing
    machine, not a protocol field, and every other field is in network
    order. `pcaplink.family_is_in_file_order` is the one exception.
20. **A link type this package does not name is not an error.**
    `PcapLinkOther` carries the number, and a consumer that understands
    it matches on the number.

## What is not included

- **Live capture.** Reading packets from an interface needs a socket and
  a device, and every function in this package performs no input or
  output. The Rust crate `pcap` and the Python module `pcapy` do that
  job; this package reads and writes the files they produce.
- **Filtering with BPF.** Compiling and running a `tcpdump` filter
  expression is a compiler and a virtual machine, and neither is a file
  format. A caller filters the packets this package yields.
- **Compressed containers.** A `.pcap.gz` or a `.pcapng.zst` is a
  compressed file whose contents are a capture. The caller decompresses
  it and feeds the result here.
- **Packet parsing.** Reading an Ethernet, IPv4, TCP or UDP header out
  of a packet's octets is packet-nv, published separately. This package
  says what the octets are, with the link type, and hands them over
  without copying them.
- **An address type.** A Name Resolution Block's addresses stay the four
  or sixteen octets the block wrote. A caller that wants them parsed
  hands them to ipaddr-nv.
- **A clock.** A capture file records when packets were seen. Every
  timestamp written by this package arrives as an argument, so rewriting
  a capture carries the original instants through unchanged.
- **The Decryption Secrets Block and the Systemd Journal Export Block.**
  Both are pcapng blocks with contents of their own, and neither is a
  packet. They arrive as `PcapSkipped`, naming their type and their
  length, so a program rewriting a file knows what it dropped.
- **Repairing a damaged file.** The reader says where a file stopped
  making sense and stops. Finding the next plausible block after that is
  a tool's job and a heuristic.

## Related packages

- **packet-nv** parses the octets this package yields: Ethernet, ARP,
  IPv4, IPv6, ICMP, TCP and UDP headers, read as spans over the caller's
  own bytes. Take this package to read the file and that one to read the
  packets.
- [ipaddr-nv](https://novo-lang.org/packages/ipaddr-nv) parses and
  formats IP addresses, which is what a Name Resolution Block's
  addresses are.
- [libpcap-sys](https://novo-lang.org/packages/libpcap-sys) binds the C
  library, which does live capture and BPF filtering. Take it when
  packets must come from an interface rather than from a file.
- [bitstream-nv](https://novo-lang.org/packages/bitstream-nv) reads
  fields that are not a whole number of octets. Nothing in either
  capture format is one, which is why this package does not depend on
  it.

## Tests

```bash
novo test tests/pcaphdr_tests.nv     # the magic numbers, the header, the packet
novo test tests/pcapts_tests.nv      # the resolutions and the timestamps
novo test tests/pcapng_tests.nv      # the blocks, the section, the options
novo test tests/pcapread_tests.nv    # the reader
novo test tests/pcapwrite_tests.nv   # the writer
novo test tests/pcapcover_tests.nv   # every public function is reached
```

The normative sources are the pcapng specification for the blocks, the
options and the timestamp resolution; libpcap's `savefile.c` for the
four magic numbers and the 24-octet header; and tcpdump.org's registry
for the link types. The reference implementations are the Rust crate
`pcap-file` and the Python library `dpkt`.

The suite asserts that the four magic numbers give two independent
answers, that a prefix of a header is not an error, that a stream ending
inside a block is a truncation and one ending between blocks is not,
that option code 2 means four different things, that a one-octet option
occupies eight, that a Simple Packet Block carries no timestamp, that a
Simple Packet Block cannot record a truncation, and that a
LINKTYPE_NULL frame's address family is in the file's byte order.

The tests compile today and fail at run, each on the
`not implemented: pcapfile-nv.<module>.<function>` panic that is its
body. That is the expected state of an interface release. They turn
green one at a time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `PcapError`, `PcapByteOrder`, `PcapLinkType`, `PcapResolution`, `PcapTimestamp`, `PcapUnixTime` | the types are declared |
| `PcapFormat`, `PcapMagic`, `PcapSpan`, `PcapPacket`, `PcapPacketOrigin`, `PcapFileHeader` | the types are declared |
| `PcapngOption`, `PcapngOptionName`, `PcapngBlockType`, `PcapngBlock`, `PcapngSection`, `PcapngInterface` | the types are declared |
| `PcapngNameRecord`, `PcapngNameResolution`, `PcapngStatistics`, `PcapngCustom` | the types are declared |
| `PcapLimits`, `PcapEvent`, `PcapFed`, `PcapReader`, `PcapOutPacket` | the types are declared |
| `pcaperror.at`, `.code`, `.is_truncation`, `.is_caller_fault`, `PcapError.message` | no |
| `pcapbyte.order_name`, `.opposite` | no |
| `pcapbyte.read_u16`, `.read_u32`, `.read_i32`, `.read_u64`, `.read_i64` | no |
| `pcapbyte.write_u16`, `.write_u32`, `.write_i32`, `.write_i64` | no |
| `pcaplink.link_type`, `.link_code`, `.link_name`, `.is_named` | no |
| `pcaplink.fixed_header_length`, `.family_is_in_file_order` | no |
| `pcapts.micros`, `.nanos`, `.resolution_of_tsresol`, `.tsresol_octet` | no |
| `pcapts.ticks_per_second`, `.is_finer_than_nanosecond` | no |
| `pcapts.timestamp`, `.ticks`, `.from_halves`, `.high_half`, `.low_half` | no |
| `pcapts.unix_time`, `.to_unix`, `.from_unix`, `.is_exact` | no |
| `pcapts.from_classic`, `.classic_seconds`, `.classic_fraction` | no |
| `pcaphdr.format_of`, `.format_name`, `.magic_bytes`, `.magic_word`, `.magic_of` | no |
| `pcaphdr.magic_order`, `.magic_resolution`, `.magic_for` | no |
| `pcaphdr.file_header`, `.parse_file_header`, `.header_order`, `.header_resolution` | no |
| `pcaphdr.span`, `.span_end`, `.packet`, `.is_truncated`, `.has_timestamp`, `.length_is_exact` | no |
| `pcapngopt.option`, `.comment`, `.end_of_options`, `.parse_options` | no |
| `pcapngopt.option_name`, `.option_code`, `.option_label`, `.named`, `.all` | no |
| `pcapngopt.text_of`, `.octet_of`, `.u32_of`, `.u64_of` | no |
| `pcapngopt.padded_length`, `.padding_for` | no |
| `pcapngblock.block_code`, `.block_type`, `.block_name`, `.block` | no |
| `pcapngblock.carries_packet`, `.is_decoded` | no |
| `pcapngblock.section`, `.parse_section`, `.section_length` | no |
| `pcapngblock.interface`, `.parse_interface` | no |
| `pcapngblock.name_record`, `.name_resolution`, `.parse_name_resolution` | no |
| `pcapngblock.statistics`, `.parse_statistics`, `.packets_received`, `.packets_dropped` | no |
| `pcapngblock.custom` | no |
| `pcapread.default_limits`, `.embedded_limits`, `.reader` | no |
| `pcapread.feed`, `.finish` | no |
| `pcapread.format`, `.order`, `.offset`, `.pending_len`, `.needs`, `.body_open` | no |
| `pcapread.interfaces`, `.interface_of`, `.link_type`, `.resolution` | no |
| `pcapread.time_offset`, `.snap_length` | no |
| `pcapwrite.out_packet`, `.whole_packet`, `.padding`, `.options`, `.block` | no |
| `pcapwrite.classic_header`, `.classic_record`, `.classic_file` | no |
| `pcapwrite.section_header`, `.interface_description` | no |
| `pcapwrite.enhanced_packet`, `.simple_packet` | no |
| `pcapwrite.name_resolution`, `.interface_statistics`, `.custom_block`, `.ng_file` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
