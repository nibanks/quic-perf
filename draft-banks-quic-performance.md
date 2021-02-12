---
title: QUIC Performance
abbrev: QUIC-PERF
docname: draft-banks-quic-performance-01
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

--- abstract

The QUIC performance protocol provides a simple, general-purpose protocol for
testing the performance characteristics of a QUIC implementation.  With this
protocol a generic server can support any number of client-driven performance
tests and configurations.  Standardizing the performance protocol allows for
easy comparisons across different QUIC implementations.

--- middle

# Introduction

The various QUIC implementations are still quite young and not exhaustively
tested for many different performance heavy scenarios.  Some have done their own
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

TODO - Use a different ALPN ("perfcfg"?) to allow for configuring the server?
Perhaps allowing for a client to "lock" the server for a period of time to
synchronize independent client testing?

## Streams

The performance protocol is primarily centered around sending and receiving
data.  Streams are the primary vehicle for this.  All performance tests are
client-driven:

 - The client opens a stream.
 - The client encodes the size of the requested server response.
 - The client sends any data it wishes to.
 - The client cleanly closes the stream with a FIN.

When a server receives a stream, it does the following:

 - The server accepts the new stream.
 - The server processes the encoded response size.
 - The server drains the rest of the client data, including the FIN.
 - The server then sends any response payload that was requested.

**Note** - While it is possible for a server to "cheat" and start sending its
data immediately once it knowns the response size. There is no built-in way to
prevent this behavior. This protocol practically operates on the honor system.

### Encoding Response Size

Every stream opened by the client uses the first 8 bytes of the stream data to
encode a 64-bit unsigned integer in network byte order to indicate the length of
data the client wishes the server to respond with.  An encoded value of zero is
perfectly legal, and a value of MAX_UINT64 (0xFFFFFFFFFFFFFFFF) is practically
used to indicate an unlimited server response.  The client may then cancel the
transfer at its convenience with a STOP_SENDING frame.

On the server side, any stream that is closed before all 8 bytes are received
should just be ignored, and gracefully closed on its end (if applicable).

### Bidirectional vs Unidirectional Streams

When a client uses a bidirectional stream to request a response payload from the
server, the server sends the requested data on the same stream.  If no data is
requested by the client, the server merely closes its side of the stream.

When a client uses a unidirectional stream to request a response payload from
the server, the server opens a new unidirectional stream to send the requested
data.  If no data is requested by the client, the server need take no action.

## Unreliable Datagrams

For endpoints that support the QUIC Unreliable Datagram extension, datagrams may
also be used.  The datagrams are used similarly, but with a few minor differences
to account for their unreliable nature:

- The client encodes a flow ID and the response size in every datagram sent.

When a server receives a datagram, it does the following:

- The server decodes the flow ID and the response size.
- If the server has not received a datagram with a new flow ID, it then responds
  with its own datagrams with matching flow ID.  It sends datagrams up to the
  requested response size.

**Note** - This design requires the server to maintain connection-wide state.
This could become a DoS vector.  To avoid this, the client SHOULD use
monotomically increasing flow ID values and the server may trivially keep track
of the highest flow ID received to decide on whether it should respond.  This
solution is not perfect as it can result in reordered datagrams from being
ignored.

### Encoding Flow ID and Response Size

Every datagram sent by the client uses the first 8 bytes of the payload to
encode a 64-bit unsigned integer in network byte order to indicate the flow ID,
and the following 8 bytes to encode a 64-bit unsigned integer in network byte
order to indicate the total length of data the client wishes the server to
respond with.  This can be larger that can fit into a single datagram.  In that
case, the server should continue to send datagrams in response until the total
length is reached.

To account for possible loss of datagrams, the client encodes the total response
length for that flow ID in every datagram it sends.  Then, the server needs to
only respond to the first (if any) datagram it receives.

An encoded value of zero for the response size is perfectly legal, and a value
of MAX_UINT64 (0xFFFFFFFFFFFFFFFF) is practically used to indicate an unlimited
server response, but there is no way for the client to cancel receipt of this
data besides closing the connection.

On the server side, any datagram that received but does not contain enough
payload for the flow ID and response size should be ignored.

## Error Codes

Error codes on connections are unused in this protocol and are ignored on
receipt.

There are several scenarios where error codes may be exchanged on streams.
These error codes are generally used to debug interopability issues between the
endpoints:

0: Success or No Error
1: Not Implemented (e.g. receiving a unidirectional stream if not supported)
2: Rejected (e.g. server is too loaded and can't accept a stream)
3: Invalid Data (e.g. the client didn't encode the full response size)

# Example Performance Scenarios

All stream payload based tests below can be achieved either with bidirectional
or unidirectional streams.  Generally, the goal of all these performance tests
is to measure the maximum load that can be achieved with the given QUIC
implementation and hardware configuration.  To that end, the network is not
expected to be the bottleneck in any of these tests.  To achieve that, the
appropriate network hardware must be used so as to not limit throughput.

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

Another metric that may reveal the connection setup efficiency is handshakes
per second. It lets multiple clients (possibly from multiple machines) setup
QUIC connections (then close them by CONNECTION_CLOSE) with a single server.
Variables that may potentially affect the results are:

 - The number of client machines.
 - The number of connections a client can initialize in a second.
 - The size of ClientHello (long list of supported ciphers, versions, etc.).

All the variables may be changed to measure the maximum handshakes per second
in a given scenario.

The test starts with the multiple clients initializing connections and waiting
for them to be connected with the single server on the other machine. It's
recommended to wait an additional couple of seconds for connections to settle
down.

The clients will initialize as many connections as possible to saturate the
server. Once the client receive the handshake from the server, it terminates
the connection by sending a CONNECTION_CLOSE to the server. The total
handshakes per second are calculated by dividing the time period by the total
number of connections that have successfully established during that time.

## Throughput Fairness Index

Connection fairness is able to help us reveal how the throughput is allocated
among each connection. A way of doing it is to establish multiple hundreds or
thousands of concurrent connections and request the same data block from a
single server. Variables that have potential impact on the results are:

 - the size of the data being requested.
 - the number of the concurrent connections.

The test starts with establishing several hundreds or thousands of concurrent
connections and downloading the same data block from the server simultaneously.

The index of fairness is calculated using the complete time of each connection
and the size of the data block in [Jain's manner]
(https://www.cse.wustl.edu/~jain/atmf/ftp/af_fair.pdf).

Be noted that the relationship between fairness and whether the link is
saturated is uncertain before any test. Thus it is recommended that both cases
are covered in the test.

TODO: is it necessary that we also provide tests on latency fairness in the
multi-connection case?

## Maximum Number of Idle Connections

TODO

## Voice & Video

Unreliable datagrams may be used to to produce voice or video-like traffic.
Then loss, delay and jitter can be measured.

TODO

# Things to Note

There are a few important things to note when doing performance testing.

## What Data Should be Sent?

Since the goal here is to measure the efficiency of the QUIC implementation and
not any application protocol, the performance application layer should be as
light-weight as possible.  To this end, the client and server application layer
may use a single preallocated and initialized buffer that it queues to send when
any payload needs to be sent out.

## Ramp up Congestion Control or Not?

When running CPU limited, and not network limited, performance tests ideally we
don't care too much about the congestion control state.  That being said,
assuming the tests run for enough time, generally congestion control should ramp
up very quickly and not be a measureable factor in the measurements that result.

## Disabling Encryption

A common topic when talking about QUIC performance is the effect that its
encryption has.  The draft-banks-quic-disable-encryption draft specifies a way
for encryption to be mutually negotiated to be disabled so that an A:B test can
be made to measure the "cost of encryption" in QUIC.

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
