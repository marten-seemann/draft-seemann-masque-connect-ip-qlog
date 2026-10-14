---
title: "qlog Definitions for CONNECT-IP"
abbrev: "CONNECT-IP qlog"
category: std

docname: draft-seemann-masque-connect-ip-qlog-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Web and Internet Transport"
wg: MASQUE
keyword:
 - masque
 - connect-ip
 - qlog
venue:
  group: "Multiplexed Application Substrate over QUIC Encryption"
  type: "Working Group"
  mail: "masque@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/masque/"
  github: "marten-seemann/draft-seemann-masque-connect-ip-qlog"
  latest: "https://marten-seemann.github.io/draft-seemann-masque-connect-ip-qlog/draft-seemann-masque-connect-ip-qlog.html"

author:
 -
    fullname: Marten Seemann
    email: martenseemann@gmail.com

normative:
  QLOG-MAIN: I-D.ietf-quic-qlog-main-schema
  QLOG-HTTP: I-D.ietf-quic-qlog-h3-events
  CONNECT-IP: RFC9484
  CONNECT-IP-DNS: I-D.ietf-masque-connect-ip-dns
  CDDL: RFC8610
  RFC5952:

informative:

...

--- abstract

This document defines qlog representations for the capsules used by CONNECT-IP,
including the DNS and PREF64 configuration extensions. These representations
extend the HTTP capsule events defined in the HTTP/3 qlog event schema.


--- middle

# Introduction {#intro}

CONNECT-IP ({{CONNECT-IP}}) uses capsules to exchange address assignments,
address requests, and routing information. {{CONNECT-IP-DNS}} defines
additional capsules for DNS and PREF64 configuration.

This document defines how these capsules are represented in qlog
({{QLOG-MAIN}}), using the HTTP capsule events defined in {{QLOG-HTTP}}.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

The data types are defined using CDDL ({{CDDL}}). The `uint8`, `uint16`,
`uint64`, `hexstring`, and `RawInfo` types are defined in {{QLOG-MAIN}}.


# Event Schemas {#schema}

The schemas defined in {{connect-ip-schema}} and {{dns-schema}} extend the
`http` namespace defined in {{QLOG-HTTP}}. Implementations SHOULD include the
URIs of the schemas they use, together with the HTTP event schema URI, in the
trace's `event_schemas` field, as described in {{Section 8 of QLOG-MAIN}}.

## Draft Event Schema Identification
{:removeinrfc="true"}

Until this document is published as an RFC, implementations MUST NOT use the
unversioned event schema URIs. Instead, they MUST append a hyphen and the
two-digit draft number to each extension identifier. For draft 01, the URIs are
`urn:ietf:params:qlog:events:http#connect-ip-01` and
`urn:ietf:params:qlog:events:http#connect-ip-dns-01`.


## Common Capsule Encoding {#capsules}

The capsule types defined in the following sections extend the `$HTTPCapsule`
type socket defined in {{Section 7.1 of QLOG-HTTP}}.

These representations are used in the `capsule` field of
`http:capsule_created` and `http:capsule_parsed` events. The event's `stream_id`
identifies the CONNECT-IP request stream, allowing multiple sessions and
application protocols to share a trace.

Sequence fields preserve the order of entries in the capsule. Empty sequences
are represented by empty arrays. Integer fields contain their numeric values
without the variable-length integer encoding.

IP addresses use dotted-decimal notation for IPv4 and the text representation
defined in {{RFC5952}} for IPv6. An `ip_prefix` combines an address, a slash,
and a decimal prefix length, for example `192.0.2.0/24` or `2001:db8::/64`.

The optional `raw` field uses `RawInfo`. Its `length` includes the capsule's
Type, Length, and Value fields; `payload_length` is the length of the Value
field. Both exclude HTTP DATA frame headers.

# CONNECT-IP Capsule Schema {#connect-ip-schema}

This schema defines representations for the capsules in {{CONNECT-IP}}. Its URI
is `urn:ietf:params:qlog:events:http#connect-ip`.

~~~ cddl
ConnectIPCapsule = ConnectIPAddressAssignCapsule /
                   ConnectIPAddressRequestCapsule /
                   ConnectIPRouteAdvertisementCapsule

$HTTPCapsule /= ConnectIPCapsule
~~~

## ADDRESS_ASSIGN {#address-assign}

`ConnectIPAddressAssignCapsule` represents the ADDRESS_ASSIGN capsule defined
in {{Section 4.7.1 of CONNECT-IP}}. Each assigned address records its Request
ID and IP prefix.

~~~ cddl
ConnectIPAddressAssignCapsule = {
    capsule_type: "address_assign"
    assigned_addresses: [* ConnectIPAddress]
    ? raw: RawInfo
}

ConnectIPAddress = {
    request_id: uint64
    ip_prefix: text
}
~~~

Unsolicited assignments retain Request ID 0. Refused assignments retain the
prefix `0.0.0.0/32` or `::/128`, as specified in
{{Section 4.7.2 of CONNECT-IP}}.

## ADDRESS_REQUEST {#address-request}

`ConnectIPAddressRequestCapsule` represents the ADDRESS_REQUEST capsule defined
in {{Section 4.7.2 of CONNECT-IP}}. Each requested address records its Request
ID and requested IP prefix, including an unspecified address when the sender
has no preference for a particular address.

~~~ cddl
ConnectIPAddressRequestCapsule = {
    capsule_type: "address_request"
    requested_addresses: [+ ConnectIPAddress]
    ? raw: RawInfo
}
~~~

## ROUTE_ADVERTISEMENT {#route-advertisement}

`ConnectIPRouteAdvertisementCapsule` represents the ROUTE_ADVERTISEMENT capsule
defined in {{Section 4.7.3 of CONNECT-IP}}. Each range records its inclusive
start and end addresses and its IP Protocol value. A protocol value of 0
retains its meaning of allowing all protocols.

~~~ cddl
ConnectIPRouteAdvertisementCapsule = {
    capsule_type: "route_advertisement"
    ip_address_ranges: [* ConnectIPAddressRange]
    ? raw: RawInfo
}

ConnectIPAddressRange = {
    start_ip: text
    end_ip: text
    ip_protocol: uint8
}
~~~

# DNS and PREF64 Capsule Schema {#dns-schema}

This schema defines representations for the capsules in {{CONNECT-IP-DNS}}.
Its URI is `urn:ietf:params:qlog:events:http#connect-ip-dns`.

~~~ cddl
ConnectIPDNSCapsule = ConnectIPDNSAssignCapsule / ConnectIPPREF64Capsule

$HTTPCapsule /= ConnectIPDNSCapsule
~~~

## DNS_ASSIGN {#dns-assign}

`ConnectIPDNSAssignCapsule` represents the DNS_ASSIGN capsule defined in
{{Section 3.4 of CONNECT-IP-DNS}}. Each configuration records its nameservers,
internal domains, and search domains.

~~~ cddl
ConnectIPDNSAssignCapsule = {
    capsule_type: "dns_assign"
    dns_configurations: [* ConnectIPDNSConfiguration]
    ? raw: RawInfo
}

ConnectIPDNSConfiguration = {
    nameservers: [* ConnectIPDNSNameserver]
    internal_domains: [* text]
    search_domains: [* text]
}

ConnectIPDNSNameserver = {
    service_priority: uint16
    ipv4_addresses: [* text]
    ipv6_addresses: [* text]
    authentication_domain_name: text
    service_parameters: [* ConnectIPDNSServiceParameter]
}

ConnectIPDNSServiceParameter = {
    key: uint16
    value: hexstring
}
~~~

Domain names retain the presentation format used in the capsule, including
empty strings where permitted by {{CONNECT-IP-DNS}}. Each service parameter
records its numeric SvcParamKey and its SvcParamValue wire bytes as a
hexadecimal string. Parameters are listed in ascending key order.

## PREF64 {#pref64}

`ConnectIPPREF64Capsule` represents the PREF64 capsule defined in
{{Section 4.1 of CONNECT-IP-DNS}}. Each prefix uses the same address and prefix
length notation as `ip_prefix`. The 96 address bits carried in the capsule
are followed by 32 zero bits to form the IPv6 address.

~~~ cddl
ConnectIPPREF64Capsule = {
    capsule_type: "pref64"
    prefixes: [* text]
    ? raw: RawInfo
}
~~~


# Security Considerations {#security}

TODO Security


# IANA Considerations {#iana}

This document requests two entries in the "qlog event schema URIs" registry
defined in {{Section 15 of QLOG-MAIN}}.

## CONNECT-IP Capsule Schema

Event schema URI:
: urn:ietf:params:qlog:events:http#connect-ip

Namespace:
: http

Event Types:
: None. This schema extends the capsule data types of existing events.

Description:
: Capsule representations for CONNECT-IP.

Reference:
: {{connect-ip-schema}} of this document.

## DNS and PREF64 Capsule Schema

Event schema URI:
: urn:ietf:params:qlog:events:http#connect-ip-dns

Namespace:
: http

Event Types:
: None. This schema extends the capsule data types of existing events.

Description:
: Capsule representations for the CONNECT-IP DNS and PREF64 extensions.

Reference:
: {{dns-schema}} of this document.


--- back
