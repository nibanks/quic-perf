---
title: QUIC Performance
abbrev: QUIC-PERF
docname: draft-banks-quic-performance-00
category: exp
date: 2020

stand_alone: yes

ipr: trust200902
area: Transport
kw: Internet-Draft

coding: us-ascii
pi: [toc, sortrefs, symrefs, comments]

author:
  -
    ins: N. Banks
    name: Nick Banks
    org: Microsoft Corporation
    email: nibanks@microsoft.com
    country: U.S.A

--- abstract

The QUIC performace protocol provides a simple, general-purpose protocol for
testing the performance characteristics of a QUIC implementation.  With this
protocol a genric server can support any number of client-driven performance
tests and configurations.  Standardizing the performance protocol allows for
easy comparisons across different QUIC implementations.

--- middle

# Introduction

The various QUIC implementations are still quite young and not exhaustively
tested for many different peformance heavy scenarios.  Some have done their own
testing, but many are just starting this process.  Additionally, most only test
the performance between their own client and server.  The QUIC performance
protocol aims to standardize the performance testing mechanisms.  This will
hopefully achieve the following:

 - Remove the need to redesign a performance test for each QUIC implementation.
 - Provide standard test cases that can produce performance metrics that can be
   easily compared across different configurations and implementations.
 - Allow for easy cross-implementation performance testing.

## Terms and Definitions

The keywords "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

# Specification

The sections below describe the mechanisms used by a client to connect to a
QUIC perf server and execute various performance scenarios.

## Protocol Negotiation

The ALPN used by the QUIC performance protocol is "perf".  It can be used on any
UDP port, but UDP port 443 is used by default, if no other port is specified. No
SNI is required to connect, but may be optionally provided if the client wishes.

## Configuration

TODO - Possible options: use the first stream to exchange configurations data OR
use a custom transport parameter.

## Streams

The performance protocol is primarily centered around sending and receiving
data.  Streams are the primary vehicle for this.  All performance tests are
client-driven:

 - The client opens a stream.
 - The client encodes the size of the requested server response.
 - The client sends any data it wishes to.
 - The client cleanly closes the stream with a FIN.

When a server receives a stream does the following:

 - The server accepts the new stream.
 - The server processes the encoded response size.
 - The server drains the rest of the client data.
 - The server then sends any response payload that was requested.

**Note** - Should the server wait for FIN before replying?

### Encoding Server Response Size

Every stream opened by the client uses the first 8 bytes of the stream data to
encode a 64-bit unsigned integer in network byte order to indicate the length of
data the client wishes the server to respond with.  An encoded value of zero is
perfectly legal, and a value of MAX_UINT64 (0xFFFFFFFFFFFFFFFF) is practically
used to indicate an unlimited server response.  The client may then cancel the
transfer at its convienence with a STOP_SENDING frame.

On the server side, any stream that is closed before all 8 bytes are received
should just be ignored, and gracefully closed on its end (if applicable).

### Bidirectional vs Unidirectional Streams

When a client uses a bidirectional stream to request a response payload from the
server, the server sends the requested data on the same stream.  If no data is
requested by the client, the server merely closes its side of the stream.

When a client uses a unidirectional stream to request a response payload from
the server, the server opens a new unidirectional stream to send the requested
data.  If no data is requested by the client, the server need take no action.

# Example Performance Scenarios

All stream payload based tests below can be achieved either with bidirectional
or unidirectional streams.

## Single Connection Bulk Throughput

Bulk data throughput on a single QUIC connection is probably the most common
metric when first discussing the performance of a QUIC implementation.  It uses
only a single QUIC connection.  It may be either an upload or download.  It can
be of any desired length.

For an upload test, the client need only open a single stream, encodes a zero
server response size, sends the upload payload and then closes (FIN) the stream.

For a download test, the client again opens a single stream, encodes the
server's response size (N bytes) and then closes the stream.

The total throughput rate is measured by the client, and is calculated by
dividing the total bytes sent or received by difference in time from when the
client created its initial stream to the time the client received the server's
FIN.

## Requests Per Second

Another very common performance metric is calculating the maximum requests per
second that a QUIC server can handle.  Unlike the bulk throughput test above,
this test generally requires many parallel connections (possibly from multiple
client machines) in order to saturate the server properly.  There are several
variables that tend to directly affect the results of this test:

 - The number of parallel connections.
 - The size of the client's request.
 - The size of the server's response.

All of the above variables may be changed to measure the maximum RPS in the
given scenario.

The test starts with the client connecting all parallel connections and waiting
for them to be connected.  It's recommended to wait an additional couple of
seconds for things to settle down.

The client then starts sending "requests" on each connection. Specifically, the
client should keep at least one request pending (preferrably at least two) on
each connection at all times.  When a request completes (receive server's FIN)
the client should immediately queue another request.

The client continues to do this for a configured period of time.  From my
testing, ten seconds seems to be a good amount of time to reach the steady
state.

Finally, the client measures the maximum requests per second rate as the total
number of requests completed divided by the total execution time of the
requests phase of the connection (not including the handshake and wait period).

## Handshakes Per Second

TODO

# Things to Note

There are a few important things to note when doing performance testing.

## What Data Should be Sent?

## Ramp up Congestion Control or Not?

TODO

## Disabling Encryption

TODO - Point to quic-disable-encryption draft

# Security Considerations

Since the performance protocol allows for a client to trivially request the
server to do a significant amount of work, it's generally advisable not to
deploy a server running this protocol on the open internet.

One possible mitigation for unauthenticated clients generating an unacceptable
amount of work on the server would be to use client certificates to authenticate
the client first.

# IANA Considerations

None

--- back
