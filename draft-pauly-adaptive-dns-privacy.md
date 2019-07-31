---
title: "Adaptive DNS: Improving Privacy of Name Resolution"
abbrev: ADNS Privacy
docname: draft-pauly-adaptive-dns-privacy-latest
date:
category: std

ipr: trust200902
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
  -
    ins: T. Pauly
    name: Tommy Pauly
    org: Apple Inc.
    street: One Apple Park Way
    city: Cupertino, California 95014
    country: United States of America
    email: tpauly@apple.com
  -
    ins: E. Kinnear
    name: Eric Kinnear
    org: Apple Inc.
    street: One Apple Park Way
    city: Cupertino, California 95014
    country: United States of America
    email: ekinnear@apple.com
  -
    ins: C. Wood
    name: Chris Wood
    org: Apple Inc.
    street: One Apple Park Way
    city: Cupertino, California 95014
    country: United States of America
    email: cawood@apple.com
    
informative:
    RRTYPE:
      title: Associated Trusted Resolver Records
      authors:
        -
          T. Pauly
    OBFUSCATION:
      title: Obfuscated DNS Over HTTPS
      authors:
        -
          T. Pauly

--- abstract

This document defines an architecture that allows client hosts to dynamically
discover authoritative resolvers that offer encrypted DNS services, and use them
in an adaptive way that improves privacy while co-existing with locally
provisioned resolvers. These resolvers can be used directly when
looking up names for which they are authoritative. These resolvers also provide the ability
to proxy encrypted queries, thus obfuscating the identity of the client requesting resolution.

--- middle

# Introduction

When client hosts need to resolve names into addresses in order to establish networking connections,
they traditionally use by default the DNS resolver that is provisioned by the local router, or by
a tunneling server such as a VPN.

However, privacy-sensitive client hosts often would prefer to use a encrypted DNS service other
than the one locally provisioned in order to prevent interception or modification by untrusted parties along
the network path and centralized profiling by a single local resolver. Protocols that can improve the privacy
stance of a client when using DNS or creating TLS connections include DNS-over-TLS {{!RFC7858}},
DNS-over-HTTPS {{!RFC8484}}, and encrypted Server Name Indication (ENSI) {{!I-D.ietf-tls-esni}}.

There are several concerns around a client host using such privacy-enhancing mechanisms
for generic system traffic. A remote service that provides encrypted DNS may not provide
correct answers for locally available resources, or private resources (such as domains only
accessible over a private network). A remote service may also itself be untrusted from a
privacy perspective: while encryption will prevent on-path observers from seeing hostnames,
the client host needs to trust the encrypted DNS service to not store or misuse queries made
to it.

Client systems are left with choosing between one of the following stances:

1. Send all application DNS queries to a particular encrypted DNS service, which requires establishing
user trust of the service. This can lead to resolution failures for local or private enterprise domains.

2. Allow the user or another entity to configure local policy for which domains to send to local,
private, or encrypted resolvers. This provides more granularity, but increases user burden.

3. Only use locally-provisioned resolvers, and opportunistically use encrypted DNS to these resolvers
when possible. (Clients may learn of encrypted transport support by actively probing such
resolvers.) This provides marginal benefit over not using encrypted DNS at all, especially if clients
have no means of authenticating or trusting local resolvers.

This document defines an architecture that allows clients to improve the privacy of their
DNS queries without requiring user intervention, and allowing coexistence with local, private,
and enterprise resolver.

This architecture is composed of several mechanisms:

- A DNS RRTYPE that indicates an authoritative DoH server associated with a name ({{RRTYPE}})

- An extension to DoH that allows queries to be obfuscated ({{OBFUSCATION}})

- A DoH server that responds to queries directly and supports proxying ({{server}})

- Client behavior rules for how to resolve names using a combination of authoritative DoH resolvers, obfuscated queries, and local resollvers ({{client}})

## Specification of Requirements

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in BCP 14
{{?RFC2119}} {{?RFC8174}} when, and only when,
they appear in all capitals, as shown here.

# Terminology

This document defines the following terms:

Adaptive DNS:
: Adaptive DNS is a technique to provide an encrypted transport for DNS queries that can
be sent directly to an Authoritative DoH Server, use Obfuscated DoH to hide the client
IP address, or use Direct Resolvers when required or appropriate.

Authoritative DoH Server:
: A DNS resolver that provides connectivity over HTTPS (DoH) that is known to be authoritative
for a given domain.

Direct Resolver:
: A DNS resolver using any transport that is provisioned directly by a local router or a VPN.

Exclusive Direct Resolver:
: A Direct Resolver that requires the client to use it exclusively for a given set of domains, such
as private domains managed by a VPN. This status is governed by local system policy.

Obfuscated DoH:
: A technique that uses multiple DoH servers to proxy queries in a way that obfuscates
the client's IP address.

Obfuscation Proxy:
: A resolution server that proxies encrypted client DNS queries to another resolution server that
will be able to decrypt the query (the Obfuscation Target).

Obfuscation Target:
: A resolution server that receives encrypted client DNS queries via an Obfuscation Proxy.

Privacy-Sensitive Connections:
: Connections made by clients that are explicitly Privacy-Sensitive are treated differently
from connections made for generic system behavior, such as non-user-initiated maintenance
connections. This distinction is only relevant on the client host, and does not get communicated
to other network entities. Certain applications, such as browsers, can choose to treat
all connections as privacy-sensitive.

Web PvD:
: A Web Provisioning Domain, or Web PvD, represents the configuration of resolvers, proxies,
and other information that a server deployment makes available to clients. See {{configuration}}.

# Client Behavior {#client}

Adaptive DNS allows client systems and applications to improve the privacy
of their DNS queries and connections both by requiring confidentiality via encryption
and by limiting the ability to correlate client IP addresses with query contents.
Specifically, the goal for client queries is to achieve the following properties:

- Eavesdroppers on the local network or elsewhere on the path will not be able to
read the names being queried by the client or the answers being returned
by the resolver.
- Only an authoritative DNS resolver that is associated with the deployment that is also
hosting content will be able to read both the client IP address and queried names for
Privacy-Sensitive Connections.
- Clients will be able to comply with policies required by VPNs and local networks that
are authoritative for private domains.

The algorithm for determining how to resolve a given name in a manner that satisfies
these properties is described in {{resolution-algorithm}}.

## Discovering Authoritative DoH Servers {#authoritative-discovery}

All direct (non-obfuscated) queries for names in privacy-sensitive connections MUST be sent to a
server that both provides encryption and is known to be authoritative for the domain.

Clients dynamically build and maintain a set of known Authoritative DoH Servers. The information
that is required to be associated with each server is:

- The URI Template of the DoH server {{!RFC8484}}
- The public key of the DoH server used for proxied obfuscated queries
- A list of domains for which the DoH server is authoritative

This information can be retrieved from several different sources. The primary source
for discovering Authoritative DoH Server configurations is the NS2 DNS Record
{{RRTYPE}}. This record provides the URI Template of the server and the public
obfuscation key for a specific domain.

When a client resolves a name (based on the order in {{resolution-algorithm}}) is SHOULD
issue a query for the NS2 record for any name that does not fall within known Authoritative
DoH Server's configuration. The client MAY also issue queries for the NS2 record for
more specific names to discover further Authoritative DoH Servers.

Any NS2 record MUST be validated using DNSSEC before a client uses the information
about the authoritative DoH Servers.

In order to bootstrap discovery of Authoritative DoH Servers, client systems SHOULD
have some saved list of at least two names that they use consistently to perform
NS2 record queries on the Direct Resolvers configured by the local network. Since
these queries are likely not private, they SHOULD NOT be associated with user
action or contain user-identifying content. Rather, the expection is that all client
systems of the same version and configuration would issue the same bootstrapping
queries when joining a network for the first time when the list of Authoritative
DoH Servers is empty.

### Whitelisting Authoritative DoH Servers  {#whitelisting}

Prior to using an Authoritative DoH Server for direct name queries on privacy-sensitive
connections, clients MUST whitelist the server.

The requirements for whitelisting are:

- Support for acting as an Obfuscation Proxy. Each Authoritative DoH Server is
expected to support acting as a proxy for Obfuscation. A client MUST issue at
least one query that is proxied through the server before sending direct queries
to the server.
- Support for acting as an Obfuscation Target. Each Authoritative DoH Server is
expected to support acting as a target for Obfuscation. A client MUST issue at
least one query that is targetd at the server through a proxy before sending direct queries
to the server.
- Signature/secondary cert by a trusted auditors.

Clients MAY further choose to restrict the whitelist by other local policy. For example,
a client system can have a list of trusted resolver configurations, and it can limit
the whitelist of Authoritative DoH Servers to configurations that match this list.

### Accessing Extended Information

When an Authoritative DoH Server is discovered, clients SHOULD also check to see
if this server provides an extended configuration in the form of a Web PvD {{configuration}}.
To do this, the client performs a lookup of https://\<DoH Server\>/.well-known/pvd, requesting
a media type of “application/pvd+json”.

If the retrieved JSON contains a "dnsZones" array, the client SHOULD perform an NS2 lookup
of each of the listed zones on the DoH server and validate that the DoH server is authoritative
for the domain; and if it is, add the domain to the local configuration.

## Discovering Local Resolvers {#local-discovery}

If the local network provides configuration with an Explicit Provisioning Domain (PvD), as
defined by {{!I-D.ietf-intarea-provisioning-domains}}, clients can learn about domains
for which the local network's resolver is authoritative.

If an RA provided by the router on the network defines an Explicit PvD that has additional
information, and this additional information JSON dictionary contains the key "dohTemplate" {{iana}},
then the client SHOULD add this DoH server to its list of known DoH configurations. The
domains that the DoH server claims authority for are listed in the "dnsZones" key. Clients
MUST peform an NS2 record query to the locally-provisioned DoH server and validate
the answer with DNSSEC before creating a mapping from the domain to the server.
Once this has been validated, clients can use this server for resolution as described in
step 2 of {{resolution-algorithm}}.

See {{local-deployment}} for local deployment considerations.

## Hostname Resolution Algorithm {#resolution-algorithm}

When establishing a secure connection to a certain hostname, clients need
to first determine which resolver configuration ought to be used for DNS resolution.
Given a specific hostname, and assuming that no other PvD or interface selection
requirement has been specified, the order of preference for which resolver to use
SHOULD be:

1. An Exclusive Direct Resolver, such as a resolver provisioned by a VPN,
domain rules that include the hostname being resolved. If the resolution
fails, the connection will fail. See {{local-discovery}} and {{local-deployment}}.

2. A Direct Resolver, such as a local router, with domain rules that is known to be
authoritative for the domain containing the hostname. If the resolution fails,
the connection will try the next resolver configuration based on this list.

3. The most specific Authoritative DoH Server that has been whitelisted ({{whitelisting}}) for the domain
containing the hostname, i.e., the DoH server which is authoritative for the longest
matching prefix of the hostname. For example, given two Authoritative DoH Servers, one for
foo.example.com and another example.com, clients connecting to bar.foo.example.com
should use the former. If the resolution fails, the connection will try an obfuscated
query.

4. Obfuscated queries using multiple DoH Servers ({{obfuscation}}). If this resolution fails,
Privacy-Sensitive Connections will fail. All other connections will use the last resort,
the default Direct Resolvers.

5. The default Direct Resolver, generally the resolver provisioned by the local router,
is used as the last resort for any connection that is not explicitly Privacy-Sensitive.

## Obfuscated Resolution {#obfuscation}

For all privacy-sensitive connection queries for names that do not correspond
to an Authoritative DoH Server, the client SHOULD use obfuscation to help
conceal its IP address from local eavesdroppers and untrusted resolvers.

DNS obfuscation is achieved by using Obfuscated DoH ({{OBFUSCATION}}). This
extension to DoH allows a client to encrypt a query with a target DoH server's public
key, and proxy the query through another server. The query is packaged with a unique
client-defined symmetric key that is used to sign the DNS answer, which is sent
back to the client via the proxy.

All DoH Servers that are used as Authoritative DoH Servers by the client
MUST support being both an Obfuscation Proxy and an Obfuscation Target,
as described in the server requirements ({{server}}).

Since each Authoritative DoH Server can act as one of two roles in an
obfuscated exchange, there are (N) * (N - 1) / 2 possible pairs of servers, where
N is the number of whitelisted servers. While clients SHOULD use a variety of
server pairs in rotation to decrease the ability for any given server to track
client queries, it is not expected that all possible combinations will be used.
Some combinations will be able to handle more load than other, and will have
better latency properties than others. To optimize performance, clients SHOULD
maintain statistics to track the performance characteristics and success rates of
particular pairs.

Clients that are performing obfuscated resolution SHOULD fall back to another
pair of servers if a first query times out, with a locally-determined limit for the
number of fallback attempts that will be performed.

# Server Requirements {#server}

Any server deployment that provides a set of services within one or more domains,
such as a CDN, can run a server node that allows clients to run Adaptive DNS.
A new server node can be added at any time, and can be used once it is
advertised to clients and can be validated and whitelisted. The system overall
is intended to scale and provide improved performance as more nodes become
available.

The basic requirements to participate as a server node in this architecture are
described below.

## Provide a DoH Server



### Obfuscated DoH Proxy

### Obfuscated DoH Target

### Keying Material

## Advertise the DoH Server

- Add DoH URI template to NS2 records
- Add Obfuscation public key to NS2 records
- Sign them with DNSSEC

## Provide Extended Configuration as a Web PvD {#configuration}

Beyond providing basic DoH server functionality, server nodes SHOULD
offer a set of extended configuration to help clients discover the default
set of domains for which the server is authoritative, as well as other
capabilities offered by the server deployment.

This set of extended configuration information is referred to as a
Web Provisioning Domain, or a Web PvD. Provisioning Domains are
sets of consistent information that clients can use to access networks,
including rules for resolution and proxying. Generally, these PvDs are
provisioned directly, such as by a local router or a VPN.
{{!I-D.ietf-intarea-provisioning-domains}} defines an extensible configuration
dictionary that can be used to add information to local PvD configurations.
Web PvDs share the same JSON configuration format, and share the
registry of keys defined as "Additional Information PvD Keys".

If present, the PvD JSON configuration MUST be available at the URI
with the format https://\<DoH Server\>/.well-known/pvd, using the well-known URI
format defined in {{!I-D.ietf-intarea-provisioning-domains}}. HTTP requests and responses
for the extended configuration information use the “application/pvd+json” media type.
Clients SHOULD include this media type as an Accept header in their GET requests,
and servers MUST mark this media type as their Content-Type header in responses.

The "identifer" key SHOULD be the hostname of the DoH Server itself.

For Web PvDs, the "prefixes" key within the JSON configuration SHOULD contain
an empty array.

The key "dnsZones", which contains an array of domains as strings, indicates the
zones that belong to the PvD. Any zone that is listed in this array for a Web PvD
MUST have a corresponding NS2 record that defines the DoH server as authoritative
for the zone. Servers SHOULD include in this array any names that are considered
default or well-known for the deployment, but is not required or expected to list
all zones or domains for which it is authoritative. The trade-off here is that zones
that are listed can be fetched and validated automatically by clients, thus removing
a bootstrapping step in discovering mappings from domains to Authoritative
DoH Servers.

Client that retrieve the Web PvD JSON dictionary SHOULD perform an NS2 record
query for each of the entries in the "dnsZones" array in order to populate the
mappings of domains. These MAY be performed in an obfuscated fashion, but
MAY also be queried directly on the DoH server (since the information is not user-specific,
but in response to generic server-driven content). Once clients retrieve the PvD JSON
information, servers MAY pre-populate the client cache by sending an HTTP Server
Push for the NS2 records for the entries in the "dnsZones" array.

This document also registers one new key in the Additional Information PvD Keys registry,
to identify the URI Template for the DoH server {{iana}}. When included in Web PvDs, this URI
MUST match the template in the NS2 DNS Record.

Beyond providing resolution configuration, the Web PvD configuration can be extended
to offer information about proxies and other services offered by the server deployment.
Such keys are not defined in this document.

# Local Resolver Deployment Considerations {#local-deployment}

A key goal of Adaptive DNS is that clients will be able to use Authoritative DoH Servers
to improve the privacy of queries, without entirely bypassing local network authority and
policy. For example, if a client host is attached to an enterprise Wi-Fi network that provides
access and resolution for private names not generally accessible on the Internet, such
names will only be usable when a local resolver is used.

In order to achieve this, a local network can advertise itself as authoritative for a domain,
allowing it to be used prior to external servers in the client resolution algorithm {{resolution-algorithm}}.

## Local Authoritative DoH Servers

If a local network wants to have clients send queries for a set of private domains to its own resolver,
it needs to define an explicit provisioning domain, as defined in {{!I-D.ietf-intarea-provisioning-domains}}.
The PvD RA option SHOULD set the H-flag to indicate that Additional Information is available.
This Additional Information JSON object SHOULD include both the "dohTemplate" and "dnsZones"
keys to define the local DoH server and which domains it claims authority over.

# Security Considerations

In order to avoid interception and modification of the information retrieved by clients
using Adaptive DNS, all exchanges between clients and servers are performed over
TLS connections.

Clients must also be careful in determining which DoH servers they send queries to
directly, without obfuscation. In order to avoid the possibility of a spoofed NS2
record defining a malicious DoH server as authoritiative, clients MUST ensure that
such records validate using DNSSEC. Even servers that are officially registered
as authoritative can risk leaking or logging information about client lookups.
Such risk can be mitigated by validating that the DoH servers can present proof
of logging audits, or by a local whitelist of servers maintained by a client.

# IANA Considerations {#iana}

This document adds a key to the "Additional Information PvD Keys" registry, defined
by {{!I-D.ietf-intarea-provisioning-domains}}.

| JSON key | Description         | Type      | Example      |
|:------------|:-----------------------|:---------------------|:------------|
| dohTemplate     | DoH URI Template {{!RFC8484}} | String | "https://dnsserver.example.net/dns-query{?dns}" |

# Acknowledgments

Thanks to Erik Nygren, Lorenzo Colitti, and Patrick McManus for their input on this approach.