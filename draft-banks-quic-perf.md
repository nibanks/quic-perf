---
title: QUIC Performance
abbrev: QUIC-PERF
docname: draft-banks-quic-perf-00
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

TODO - First 8 bytes of a stream from a client indicates the requested response
size.

# Example Performance Scenarios

Generally, all stream payload based tests can be achieved either with
bidirectional or unidirectional streams.  The only difference is that the
server's response is on the same stream for bidirectional streams but on a new
stream when using unidirectional streams.

## Single Connection Bulk Throughput

Bulk data throughput on a single QUIC connection is probably the most common
metric when first discussing the performance of a QUIC implementation.  It may
be either an upload or download.  It can be of any desired length.

For an upload test, the client need only open a single stream, encodes a zero
server response size, sends the upload payload and then closes (FIN) the stream.

For a download test, the client again opens a single stream, encodes the
server's response size (N bytes) and then closes the stream.

## Requests Per Second

Another very common performance metric is calculating the maximum requests per
second that a QUIC server can handle.

TODO

## Handshakes Per Second

TODO

# Things to Note

There are a few important things to note when doing performance testing.

## Disabling Encryption

TODO - Point to quic-disable-encryption draft

## Ramp up Congestion Control or Not?

TODO

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
