---
title: "NTS extensions for enabling pools"
abbrev: "NTS pools"
category: info

docname: draft-nts-pool-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
# area: AREA
# workgroup: WG Working Group
keyword:
 - NTP
 - NTS
venue:
#  group: WG
#  type: Working Group
#  mail: WG@example.com
#  arch: https://example.com/WG
  github: "pendulum-project/nts-pool-draft"
  latest: "https://pendulum-project.github.io/nts-pool-draft/draft-nts-pool.html"

author:
 -
    fullname: "David Venhoek"
    organization: Tweede Golf B.V.
    email: "david@tweedegolf.com"

normative:
  RFC8915:

informative:


--- abstract

The aim of this document is to describe a proof of concept system for NTS pools that are able to be used by clients without any knowledge beyond plain NTS. The work here focusses purely on creating an intermediate nts-ke server that can be configured with the addresses of multiple downstream servers and distribute load between them. The parts of pool operation dealing with managing the list of servers are left out of scope for this work.

--- middle

# Introduction


# Conventions and Definitions

Throughout the text, the terms client and server will refer to those roles in an NTS Key Exchange session as specified in {{RFC8915}}. Please note that this means that the pool itself operates in both roles: As a server towards users of the pool, and as a client towards the downstream time sources.

{::boilerplate bcp14-tagged}

# General pool architecture

We propose a pool model where the pool is providing an NTS Key Exchange service to the outside world. This allows the pool to terminate the TLS connection and avoids having to distribute certificates to all downstream time servers. However, that also implies that the pool needs to extract the keys and somehow get valid cookies for the selected downstream time server.

To solve this, we ask downstream servers to provide an extension of the NTS Key Exchange protocol that allows the pool to directly communicate the keys to the downstream server, instead of having the downstream server extract this from the TLS Session. The explicit communication of keys allows the pool to do the extraction from the TLS server whilst remaining oblivious to the cookie format of the downstream server.

# Client facilities for pools

One challenge with a pool through NTS Key Exchange is that clients that allow for explicit pool configuration do want to end up with multiple independent time sources. To ensure they won't have to do multiple NTS Key Exchange sessions just to discard the result because they already have the time server they
result in, we also introduce a record clients can use to indicate which downstream time servers they don't want.

# New NTS record types

## Keep Alive {#keepalive}
Record Type Number: To be assigned by IANA (draft implementations: 0x4000)
Critical bit: 0

Indicates a desire to keep the TLS connection active for more than one message exchange. This can be used by a Pool to reuse connections to downstream NTS-KE servers multiple times, reducing load on both the pool and downstream servers.

Client MUST send this record with a 0 sized body. Client MUST NOT use Keep Alive unless the request contains a record type allowing the use of Keep Alive. Within this specification, that is limited to the Supported Protocol List and Fixed Key Request records. A server SHOULD ignore any body for the Keep Alive record.

When supported by server and allowed for the request in question, the server MUST include a Keep Alive record with 0 sized body in the response and keep the TLS connection active after the response to handle further requests from the client. A client SHOULD ignore any body for the Keep Alive record.

When included in the request or response, the client respectively server MAY, contrary to the requirements in {{RFC8915}}, send another request or response. Any TLS "close_notify" SHALL be sent only after the last reqeust or response respectively to use the connection.

Once a Keep Alive record has been sent by a client, or honored by a server, the TLS connection over which it was sent MUST NOT be used for key extraction. Doing so anyway can result in reuse of keys and the may result in loss of confidentiality or authenticity of the resulting NTP exchanges.

## Supported Next Protocol List {#supportedprotocol}
Record Type Number: To be assigned by IANA (draft implementations: 0x4004)
Critical bit: 1

This record can be used by a pool to query downstream servers about which next protocols they support.

Client MUST send with no body. Clients MAY use Keep Alive in combination with this record. A request with this record SHOULD NOT inclue a "Next Protocol Negotiation", "AEAD Algorithm Negotiation" or "Fixed Key Request" record.

Server MUST ignore any client body sent, and MUST send in response a Supported Next Protocol List record with as data a list of 16 bit integers, giving the protocol ID's the server supports.

When included, the server MUST NOT negotiate a next protocol, aead algorithm or keys for this request.

## Supported Algorithm List {#supportedalgorithm}
Record Type Number: To be assigned by IANA (draft implementations: 0x4001)
Critical bit: 1

This record can be used by a pool to query downstream servers about which AEAD algorithms they support.

Client MUST send with no body. Clients MAY use Keep Alive in combination with this record. A request with this record SHOULD NOT include a "Next Protocol Negotiation", "AEAD Algorithm Negotiation" or "Fixed Key Request" record.

Server MUST ignore any client body sent, and MUST send in response a Supported Algorithm List record with as data a list of tuples of two 16 bit integers, the first giving a algorithm ID for the AEAD and the second giving the length of the key for that algorithm ID.

When included, the server MUST NOT negotiate a next protocol, aead algorithm or keys for this request.

## Fixed Key Request {#fixedkey}
Record Type Number: To be assigned by IANA (draft implementations: 0x4002)
Critical Bit: 1

When client is properly authenticated, the server SHOULD not perform Key Extraction for but rather use the keys provided by the client in the extension field. This allows a pool to do key negotiation on behalve of its users with the downstream NTS-KE servers, even though it terminates the TLS connection.

When used, the client MUST provide an AEAD Algorithm Negotiation record with precisely one algorithm, and a Next Protocol Negotiation record with precisely one next protocol. The data in the Fixed Key Request record must have length twice the key length N of the AEAD algorithm in the AEAD Algorithm Negotiation record. The first N bytes MUST be the C2S Key and the second set of N bytes MUST be the S2C key. Clients MAY use Keep Alive in combination with this record.

MUST not be sent by a server. Server SHOULD treat extension field as unknown when sent by any client not authorized to make fixed key requests.

## NTP Server Deny {#serverdeny}
Record Type Number: To be assigned by IANA (draft implementations: 0x4003)
Critical Bit: 0

When provided by a client, indicates a desire to connect to a server other than the server specified in the record. This can be used to ensure a client receives independent NTP servers from one NTS Key Exchange server without having to potentially try multiple times to get a new server.

A client MAY send multiple of these records if desired. The data in the record SHOULD match that given through an NTPv4 Server Negotiation received in an earlier request from the same NTS Key Exchange server.

MUST not be sent by a server. Server MAY at its discretion ignore the request from the client and still provide the given server in an NTPv4 Server Negotiation record.

# Security Considerations

In the pool design presented above, the pool effectively acts as a man in the middle between the user and the ultimate time source during the NTS Key Exchange portion of the session. This means that the pool has access to the key material of all these sessions. Although this is a small additional risk, we consider this acceptable as the pool could already always assign sessions for a user to time servers it controls anyway.

The fact that the pool also gets access to key material makes it less advisable to have a pool as a downstream time source for another pool, as this increases the number of actors with access to the key material even further.

The design above does avoid sharing key material between all downstream time sources. As a consequence, a downstream time source in the pool will not be able to break confidentiality or authenticity of traffic with other downstream time sources of the pool. Furthermore, any traffic directly with the downstream time source has no key material involved that is known to the pool.


# IANA Considerations

IANA is requested to allocate the following entries in the Network Time Security Key Establishment Record Types registry {{RFC8915}}:

| Record Type Number | Description | Reference |
| --- | --- | --- |
| [[TBD]] | Keep Alive | [[this memo]] {{keepalive}} |
| [[TBD]] | Supported Next Protocol List | [[this memo]] {{keepalive}} |
| [[TBD]] | Supported Algorithm List | [[this memo]] {{supportedalgorithm}} |
| [[TBD]] | Fixed Key Request | [[this memo]] {{fixedkey}} |
| [[TBD]] | NTP Server Deny | [[this memo]] {{serverdeny}} | 


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
