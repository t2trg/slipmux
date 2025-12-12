---
title: >
  Slipmux: Using an UART interface for diagnostics, configuration, and packet transfer
abbrev: UART slipmux
docname: draft-bormann-t2trg-slipmux-latest
# date: 2019-11-05

v: 3
keyword: Internet-Draft
cat: info
stream: IRTF

pi: [toc, sortrefs, symrefs, compact, comments]

author:
  -
    name: Carsten Bormann
    org: Universität Bremen TZI
    street: Postfach 330440
    city: Bremen
    code: D-28359
    country: Germany
    phone: +49-421-218-63921
    email: cabo@tzi.org
  -
    name: Tobias Kaupat
    org: Lobaro UG
    street: Tempowerkring 21d
    city: Hamburg
    code: D-21079
    country: Germany
    phone: +49-40-22816531-0
    email: tobias.kaupat@lobaro.de

normative:
  RFC7252: coap
  RFC1055: slip
  RFC1662: ppphdlc

informative:
  RFC1661: ppp
  RFC8613: oscoap
  STD94: cbor
#  BCP205:

venue:
  group: "Thing-to-Thing"
  type: "Research Group"
  mail: "t2trg@irtf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/t2trg"
  github: "t2trg/slipmux"
  latest: "https://t2trg.github.io/slipmux/draft-bormann-t2trg-slipmux.html"

...

--- abstract

Many research and maker platforms for Internet of Things
experimentation offer a serial interface.
This is often used for programming, diagnostic output, as well as
a crude command interface ("AT interface").
Alternatively, it is often used with SLIP (RFC1055) to transfer IP
packets only.

The present report describes how to use a single serial interface for
diagnostics, configuration commands and state readback, as well as
packet transfer.

[^status]

[^status]: This revision –04 is a resubmission of 2019's –03 with some
    editorial updates for 2025.

--- middle

Introduction        {#intro}
============

Many research and maker hardware modules for Internet of Things
experimentation ("platforms") offer a serial ("UART") interface.
This is often used for programming, diagnostic output, as well as
a crude command interface ("AT interface").
Alternatively, it is used with SLIP (RFC1055) to transfer IP packets
only; this may require the use of another serial interface for
diagnostics and configuration.

The present report describes how to use a single serial interface for
diagnostics, configuration commands and state readback, as well as
packet transfer.

## Terminology         {#terms}

The term "byte" is used in its now customary sense as a synonym for
"octet".
Where bit arithmetic is explained, this document uses the notation
familiar from the programming language C (including C++14's 0bnnn
binary literals), except that the operator "\*\*" stands for
exponentiation.

{::boilerplate bcp14-tagged-bcp14}


Using a UART interface
======================

The serial interfaces provided by today's platforms often do not
actually use EIA-232 ("RS-232") levels, but some form of logic levels
(TTL or more likely 3.3 V CMOS).  The present report does not discuss
physical interfacing, but assumes that a TXD (transmit data) pin, a
RXD (receive data) pin, and a GND (common ground) pin are all that is
available.  To interface laptops and similar devices to these serial
interfaces, inexpensive ($2) USB to UART adapters based on chips such
as PL2303, CP2102 or CH340 are easily obtainable.  (The usual care
needs to be taken when mixing 3.3 V and 5 V platforms; this is well
understood but beyond the scope of the present report.)

The general assumption is that a serial port configuration of 8N1 (8
bits per character, no parity, 1 stop bit) and a bit rate of
115200 bit/s is used.  As with the logic levels, alternative
arrangements are possible, but a 3.3 V CMOS, 115200 bit/s interface is
most likely to provide the best interoperability.

While it would be possible to run relatively complex and versatile
protocols such as PPP {{-ppp}} on such serial interfaces, this goes
against a need for simplicity and ease of setup.  In today's systems,
either weird ad-hoc protocols based on "AT commands" are used that are
not interoperable at all, or a simple encapsulation such as SLIP
{{-slip}} is used for packet transfer only.

For the purposes of the present report, on top of the serial UART
protocol, the frame format defined by {{-slip}} is indeed employed.
The detailed descriptions below generally describe the frame data
before applying SLIP escaping in the transmitter or after removing it
in the receiver.

The approach described here is informally referred to as "slipmux".

Packet Transfer
===============

Packet transfer uses the definitions of {{-slip}}.  However, contrary
to the statement in section DEFICIENCIES of {{-slip}}, multiplexing is
very well possible.  A frame used for packet transfer is detected by
an initial byte of one of the two forms:

* 0x45 to 0x4f: IPv4 packet
* 0x60 to 0x6f: IPv6 packet

This initial byte forms part of the packet; it is not removed from the
payload as with the other formats defined below.

There are no changes to the formats defined by {{-slip}}, so there
should be immediate interoperability with tools such as tunslip.

Diagnostics Transfer
====================

While not transferring a frame bearing a packet, the platform can
alternatively transmit a diagnostic frame.  These are encoded (and
escaped) using SLIP framing exactly like packet frames, but start with
the byte 0x0a (ASCII newline) and contain UTF-8 encoded characters after
that byte.  There is no semantics attached to the diagnostics message,
except that it is intended as a human-readable debug or diagnostic
message from the platform code.  It is generally preferable to end the
payload of a diagnostics message in another newline (0x0a, which on the wire
is then followed by 0xc0 due to the SLIP framing).  Note that, as long
as only ASCII characters are used, there is no need to actually
perform escaping on the diagnostic message.

Since diagnostic messages are intended for humans, they are only
defined for the direction from platform to host (e.g., laptop); for
robustness when connecting two platform modules, they should be
ignored by platform modules.

Configuration
=============

Configuration is performed by sending CoAP messages {{-coap}} in SLIP
framing.  The encapsulation of a CoAP message starts with an
additional byte 0xA9, with the bytes of the CoAP message following
(which, as for all data in frames, are escaped as necessary as per
{{-slip}}).

In contrast to the packet and diagnostics frames defined above, CoAP
frames benefit from a frame check mechanism.  After the CoAP message,
the last two bytes of a CoAP frame therefore contain a 16-bit CRC FCS
computed over the byte 0xA9 followed by the (unescaped) bytes of the
CoAP message, computed as specified in {{-ppphdlc}}.  (Note that the
two bytes of the CRC are escaped, as necessary, by the SLIP framing,
as are all other bytes of the CoAP message.)

CoAP messages with incorrect CRCs are silently discarded.

Where a local URI needs to be formed for the configuration messages,
the URI scheme "coap+uart" is used; the authority part of that URI
might be used to refer to local interface names as needed, as in:

~~~
coap+uart://ttyUSB0/APlist
~~~
{: artwork-align="center"}

The client could also be using a local mapping table to provide some
indirection in translating the authority part to a local identifier of
the serial port(e.g., COM0 to ttyUSB0).

Using an empty URI authority allows the client to use a default port, as in:

~~~
coap+uart:///APlist
~~~
{: artwork-align="center"}

A later version of this report might define some common CoAP resources
that research or maker platforms might want to provide, e.g. to cover
the configuration and status checking often done by "AT commands" today.

Framing considerations {#framing}
======================

To make SLIP framing robust, it is important to send SLIP frame
delimiters (0xc0) before and after each SLIP frame (maybe unless
frames follow each other back to back).  This means that empty frames
need to be silently ignored by a receiver.

If a platform starts to send a packet or message, but then decides it
should not complete the message before having sent the rest of the
frame, it can send the SLIP ESC (0xdb) followed by SLIP END (0xc0) to
abort the frame.  Note that this goes beyond the error handling
suggested by the section "SLIP DRIVERS" in {{-slip}} and might
therefore be of limited interoperability at first.

Messages in slipmux are strictly sequential; there is no {{?RFC2687}}
style suspension.  In particular, this means that diagnostic messages
that are generated while another message is in progress may have to be
buffered (unless they are important enough to abort the frame as
described above).

Discussion
==========

## Why no shell?

The present report is somewhat radical in that it does not provide a
common staple of interactive computer access: A command line interface
(CLI), or "shell".

This would be easy to add, but distracts from the use of the platform
as a "thing" -- it should not have to carry an (even primitive) user
interface; instead it should provide what would have been "shell
commands" as CoAP resources.

As a transition aid, existing shell commands can first be converted to
just accept their parameters via CoAP but continue to provide their
output as ASCII text over the diagnostic channel.

However, in order to aid script-driven use of the platform, the next
step should then be to also provide the response to the command in a
CoAP response, possibly structured for better use by the script.
Often code that was designed to format the data for human consumption
can be simplified to just ship the raw data, e.g. in a CBOR data item {{-cbor}}.

## Frame aborts

Implementing frame aborts as described in {{framing}} requires a
receiver to receive the entire frame before acting on it.
For diagnostic information, this is somewhat moot -- the information
is there independent of whether its frame was aborted or not.
For packets, it is usually necessary to check a UDP or TCP header
checksum before acting on it, anyway.
For CoAP requests, similarly, the CRC needs to be checked.
So implementing frame aborts should not be an undue burden.

## Unknown initial bytes

Frames with unknown initial bytes should be silently ignored.

The same is true for frames with initial bytes that are unimplemented.
However, there is an expectation that true slipmux implementations do
implement CoAP framing.
If this is unexpectedly not the case, as a courtesy to a peer CoAP
client, a slipmux implementation could at least send CoAP Reset messages:
a CoAP frame (initial byte 0xA9) with a message that starts with 0x40
to 0x5f could be replied
to with a CoAP frame with a CoAP RST message, containing just these four
bytes (as always, escaped as needed, and framed with an initial 0xA9
and a CRC):

* 0x70
* 0x00
* The third (unescaped) byte of the message being replied to
* The fourth (unescaped) byte of the message being replied to

(Generating proper CoAP framing in response does, require implementing the
PPP CRC.)
In conjunction with the CoAP ping response of a normal CoAP
implementation, this also can be used for liveness testing.

(The check for the first byte of the CoAP message is needed to avoid endless back and
forth of reset messages in certain error situations.)

IANA Considerations
============

The present report does not foresee adding additional frame types, but
as a matter of precaution, this section might define a registry for
initial bytes in a frame.  At this point, this would contain:

* 0x0a: Diagnostics
* 0x45 to 0x4f: IPv4 packet
* 0x60 to 0x6f: IPv6 packet
* 0xA9: CoAP message with 16-bit FCS

If such a registry is desired, the following values for initial bytes
should probably be reserved (while all these values could be used if
required, implementation is easier if they are not):

* 0x00
* 0xc0: {{-slip}} END
* 0xdb: {{-slip}} ESC

There might also be a need to formally register the URI scheme "coap+uart".

Security Considerations
============

The usual security considerations apply to the IP packets transferred
in packet frames.

When displaying information from diagnostic frames, care should be
taken that features of a terminal triggered e.g. by escape sequences
cannot be used for nefarious purposes.

The CoAP configuration interface does not itself provide any
security.  This may be appropriate for the local configuration needs
of an experimentation platform that is not expected to be physically connected to
any system that is not allowed full control over it (e.g., by using
the same physical interface for reflashing new firmware).  Where the
platform might connect to other systems over serial, object security
for CoAP {{-oscoap}} might be employed, or the configuration interface
might be restricted to a read-only mode only providing information
that does not need confidentiality protection.  (It would be possible
to provide a DTLS encapsulation, but this might go beyond the
objective of extreme simplicity.)

--- back

Implementation Status
=====================
{: removeinrfc}

[^ietfonly]

[^ietfonly]: (The following boilerplate is excerpted from {{Section 2.1
    of RFC7942}} (BCP205).
    That RFC does not consider implementation status sections in IRTF
    Internet-drafts, so the reader is requested to perform the necessary
    cognitive transfer.)

{::boilerplate rfc7942}

## Lobaro SLIP

A work in progress implementation of slipmux is available as part of
Lobaro's SLIP implementation (MIT License, Go language):

~~~
https://github.com/Lobaro/slip
https://github.com/Lobaro/util-slip
~~~
{: artwork-align="center"}

## aiocoap

A work in progress implementation of slipmux is a vailable as part of
the aiocoap library (MIT License, Python language, for capable systems):

<https://codeberg.org/aiocoap/aiocoap/pulls/52>

Implementation was generally straightforward,
thanks to reusing the precise {{RFC7252}} format.
Minor problem points were obtaining test vectors for FCS calculation
(interoperating with the earlier Rust implementation by Bennet Hattesen helped),
tolerating the lowercasing normalization of the device name that comes from {{Section 6.2.2.1 of ?RFC3986}},
and overriding the library's defaults on when to send the Uri-Host option.
Troubles around binding servers to serial ports stem from the general library's shortcomings and not from this specification.

Acknowledgements
================
{: unnumbered}

TBD

<!--  LocalWords:  CBOR extensibility IANA uint sint IEEE endian
 -->
<!--  LocalWords:  signedness endianness
 -->
