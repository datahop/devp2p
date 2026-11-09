# Node Discovery Protocol v5 - Rationale

**Protocol version v5.1**

Note that this specification is a work in progress and may change incompatibly without
prior notice.

This document explains the design requirements and security needs of Discovery v5. In
addition, the document tries to gather the various vulnerabilities and threats that
pertain to Kademlia-like p2p networks. Our aim is to make it plain which issues are
addressed and how they are mitigated, so that the design of the [wire protocol] may be
verified.

# Design Requirements

## Basic Goals

#### 1.1.1 Replace the Discovery v4 Endpoint Proof

The existing mutual endpoint verification process is unreliable because either side may
forget about a previously performed endpoint proof. If node A assumes that node B already
knows about a recent PING/PONG interaction and sends FINDNODE, the request may fail.
Implementations of Discovery v4 may guard against this flaw using retries, but retrying is
really slow and usually not done.

#### 1.1.2 Require knowledge of destination node ID for communication

Make it expensive to obtain the logical node ID from discovery communications. In
Discovery v4, any node can provoke responses knowing IP alone, and obtain information
about a node without knowing its ID. This encourages sloppy implementations to not perform
proper validation of FINDNODE results and increases the risk of DHT misuse for DDoS
purposes.

#### 1.1.3 Support more than one node ID cryptosystem

Ensure the DHT can accommodate ENR's with multiple identity systems. This will allow
identity cryptosystems other than *secp256k1/keccak256*.

#### 1.1.4 Replace node information tuples with ENRs

ENRs include discovery information and more. These signed, versioned records fulfill
multiple requirements, such as permitting capability advertisement and transport
negotiation.

#### 1.1.5 Guard against Kademlia implementation flaws

Discovery v4 trusts other nodes to return neighbors according to an agreed distance
metric. Mismatches in implementation can make it hard for nodes to join the network, or
lead to network fragmentation.

#### 1.1.6 Topic-based Service Discovery

The protocol must support discovery of nodes that participate in a higher-level service or
topic. A topic is identified by a fixed-length topic identifier in the same key space as
node IDs.

Topic-based service discovery should reuse the ordinary Node Discovery v5 network rather
than requiring each service to operate its own bootstrap network or discovery DHT.

Finding nodes that participate in a topic should be efficient even when the topic is
supported by only a small fraction of the discovery network. At the same time, topic
discovery should preserve the security benefits of sampling from a large global discovery
network and should not concentrate discovery for a topic at a small set of predictable
nodes.

#### 1.1.7 Change replay prevention

The use of timestamps as a replay prevention mechanism in Discovery v4 has led to many
complaints about connectivity when the host's clock was wrong. The protocol should be
independent of the clock.

#### 1.1.8 Message obfuscation

The protocol should obfuscate traffic to prevent accidental packet mangling or trivial
sniffing. It must also avoid inclusion of obvious markers to prevent naive blocking of
discovery traffic using hard-coded packet signatures. Defense against advanced traffic
analysis systems, e.g. using inter-packet timing is a secondary concern.

## Security Goals

Individual potential vulnerabilities are identified below. These each represent their own
risk mitigation goal.

#### 1.2.1 Replay of the handshake

The handshake, if successfully replayed from an older session, would allow a malicious
node to occupy a former IP location, or pollute the routing table with old information.

#### 1.2.2 Replay NODES

A NODES response, if successfully replayed, would pollute the routing table with stale
information.

#### 1.2.3 Replay PONG

A PONG, if successfully replayed, could convince a node that a node is live and
participating when it isn't.

#### 1.2.4 Kademlia redirection

A FindNode response contains false endpoint information intended at directing traffic at a
victim / polluting the routing table. A topic query results in fake endpoint information,
directing traffic at a victim.

#### 1.2.5 Kademlia redirection + self-propagation

As 1.2.3 but the responses attempt to replicate the malicious node throughout the routing
table, to amplify the source of pollution and traffic.

#### 1.2.6 Unsolicited replies

A malicious node is attempting to spam a node with fake responses to typical requests.
These messages may be replayed from previous communications, or may be new messages with
spoofed source endpoints. The aim is to disrupt weak implementations or have their
information be received as authentic, to pollute the recipient's routing table.

#### 1.2.7 Amplification

Malicious requests of small message size are sent from spoofed source IPs to direct larger
response messages at the victim.

#### 1.2.8 Kademlia direct validation

Direct validation of a newly discovered node can be an attack vector. A malicious node may
supply false node information with the IP of a victim. Validation traffic is then directed
at the victim.

#### 1.2.9 Kademlia ID count per address validations

There are various attacks facilitated by being able to associate multiple fake (or even
real) malicious node ids with a single IP endpoint. One mitigation method that is
sometimes considered is to globally limit the number of logical node IDs that can be
associated with an IP address. However, this is an attack vector. A malicious actor can
supply many logical node ids for a single IP address and thus prevent the correct node
from being able to join the network.

#### 1.2.10 Sybil/Eclipse attacks

These attacks rely on being able to create many real nodes, or spoof many logical node IDs
for a small number of physical endpoints, to form a large, isolated area of the network
under the control of the malicious actor. The victim's discovery findings are directed
into that part of the network, either to manipulate their traffic or to fully isolate them
from the network. Additional security goals specific to topic-based service discovery
([TopDisc]) are listed further below.

## Version Interoperability / Upgrade Paths

There are several considerations regarding the coexistence of v4 and v5 network members.

#### 1.3.1 Transition period during network formation

Discovery v4 clients should be able to serve as discovery v5 bootstrap nodes while the
number of new discovery v5 clients is still low.

#### 1.3.2 Circumvention of 1.1.2 with v4 PING

While a client supports both the old v4 and newer versions, it is possible for malicious
actors to pose as a v4 node and recover node IDs from arbitrary IP addresses. This should
somehow be avoided.

# Rationale

## Why UDP?

The wire protocol specification mandates the use of UDP. This may seem restrictive, but
use of UDP communication is an important part of the design. While there is no single
reason which ultimately dictates this choice, there are many reasons why the system as a
whole will function a lot better in the context of UDP.

For discovery to work, all nodes must be able to communicate with each other on equal
footing. The network won't form properly if some nodes can only communicate with certain
other nodes. Incooperative NAT in between the node and the Internet can cause
communication failure. UDP is fundamentally easier to work with when it comes to NAT
traversal. No explicit hole-punching is required if the NAT setup is capable of full-cone
translation, i.e. a single packet sent to any other node establishes a port mapping which
allows packets from others to reach the node behind NAT.

Unlike other DHT systems such as IPFS, the node discovery protocol mandates a single wire
protocol to be implemented by everyone. This avoids communication failures due to
incompatible transports and strengthens the DHT because all participants are guaranteed to
be reachable on the declared endpoint. It is also fundamentally simpler to reason about
and implement: the protocol either works in a certain context or it doesn't. If the
protocol cannot be used because the networking environment doesn't support UDP, another
discovery mechanism must be chosen.

Another reason for UDP is communication latency: participants in the discovery protocol
must be able to communicate with a large number of other nodes within a short time frame
to establish and maintain the neighbor set and must perform regular liveness checks on
their neighbors. For TopDisc, advertisers collect tickets from registrars and retry
registration after the indicated waiting time. Low-latency request/response interactions
help advertisers renew advertisements and maintain placements despite churn.

These protocol interactions are difficult to implement in a TCP setting where connections
require multiple round-trips before application data can be sent and the connection
lifecycle needs to be maintained. An implementation of the wire protocol on a TCP-based
transport would either need permanent connection to hundreds of nodes, in which case the
application would be short on file descriptors, or establish many short-lived TCP
connections per second to communicate with specific nodes.

Yet another useful property of UDP is that packets aren't required to reach their
destination --- intermediaries may drop arbitrary packets. This strengthens the protocol
because it must be designed to function even under bad connectivity. Implementations may
exploit the possibility of packet loss to their advantage. A participant can never tell
whether a certain request wasn't answered in time because the recipient chose to ignore it
or because their own connection isn't working. An implementation that tries to minimize
traffic or CPU overhead could simply drop a certain amount of packets at application level
to stay within self-imposed limits.

## Why Kademlia?

Kademlia is a simple distributed hash table design proposed in 2002. It is commonly used
for file-sharing systems where content is stored by hash and distributed among
participants based on their 'proximity' according to the XOR distance metric.

Node discovery is a Kademlia-inspired system but doesn't store any files, only node
information is relayed. We chose Kademlia primarily because the algorithm is simple and
understandable while providing a distributed database that scales with the number of
participants. Our system also relies on the routing table to allow enumeration and random
traversal of the whole network, i.e. all participants can be found. Most importantly,
having a structured network with routing enables thinking about DHT 'address space' and
'regions of address space'. These concepts are used by TopDisc to construct service tables
centred on topic identifiers.

Kademlia is often criticized as a naive design with obvious weaknesses. We believe that
most issues with simple Kademlia can be overcome by careful programming and the benefits
of a simple design outweigh the cost and risks of maintaining a more complex system.

## Sybil and Eclipse Attacks

The well-known 'sybil attack' is based on the observation that creating node identities is
essentially free. In any system using a measure of proximity among node identities, an
adversary may place nodes close to a chosen node by generating suitable identities. For
basic node discovery through network enumeration, the 'sybil attack' poses no significant
challenge. Sybils are a serious issue for topic-based service discovery, especially for
topics provided by few participants. An adversary may try to generate node IDs close to a
chosen topic identifier, dominate registrars selected for that topic, or flood registrar
ad caches with advertisements under its control.

An 'eclipse attack' is usually based on generating sybil nodes with the goal of polluting
the victim node's routing table. Once the table is overtaken, the victim has no way to
find any other nodes but those controlled by the adversary. Even if creating sybil nodes
were somehow impossible, 'eclipsing' a node might still be achieved through other means
such as directing large amounts of traffic to the node. When the victim node is unable to
keep up regular communication with the rest of the network it may lose connection and be
forced into re-bootstrapping its routing table --- a situation in which it is most
vulnerable.

Both the 'sybil attack' and the 'eclipse attack' must be considered for any structured
overlay network, and there is no single optimal solution to fully protect against these
attacks. However, certain implementation decisions can make them more expensive or render
them ineffective.

As a general measure, implementations can place IP-based limits on the content of their
routing table. For example, limiting Kademlia table buckets to two nodes from every /24 IP
subnetwork and the whole table to 10 nodes per /24 IP subnetwork significantly increases
the number of hosts an attacker must control to overtake the routing table. Such limits
are effective because IPv4 addresses are a scarce resource. Subnetwork-based limits remain
effective even as IPv6 adoption progresses.

To counter being eclipsed via repeated contact by an adversary, implementations of the
Kademlia table should avoid taking on new members on incoming contact unless the table is
well-stocked from outbound queries. Readers of the original Kademlia paper may easily
assume that liveness checks on bucket members should be performed just when a new node
tries to enter the bucket, but doing so increases the risk of emptying the table through
DoS. We therefore recommend to perform liveness checks on a separate schedule which is
independent of incoming requests. Checks may also be paused or delayed when the node is
under high load. The number of past liveness checks performed on a bucket member is an
important indicator of its age: Implementations should favor long-lived nodes and may
relax liveness checks according to node age.

A well-researched countermeasure to sybil attacks is to make creation of identities
computationally expensive. While effective in theory, there are significant downsides to
this approach. Nodes on resource-constrained devices such as mobile phones may not be able
to solve the computational puzzle in time to join the network. Continuous advances in
hashing technology which speed up cryptocurrency proof-of-work algorithms show that this
way of securing the network requires constant adjustments to thresholds and can never beat
determined attackers.

Support for mixed ENR identity schemes, described later in this document, allows for an
escape hatch to introduce arbitrary optional constraints (including proof-of-work) on node
identities. Thus, while the issue is not directly addressed at wire protocol level, there
is no inherent blocker for solving it as the need arises.

## Node Records and Their Properties

In Discovery v5, all node information is exchanged using [node records]. Records are
self-signed by the node they describe and contain arbitrary key-value pairs. They also
contain a sequence number to determine which copy of the record is newer when multiple
copies are available. When a node record is changed by its owner, the sequence number
increases. The new record 'syncs' to neighboring nodes because they will request it during
liveness revalidation. The record is also 'pushed' on to newly seen nodes as part of the
handshake.

Signing records prevents any intermediary node from changing the content of a record. Any
node's information is either available in the exact form it was published or not at all.
To make the system secure, proper validation of records is important. Implementations must
verify the signature of all received records. Implementations should also avoid sharing
records containing no usable IP addresses or ports and check that Internet hosts do not
attempt to share records containing LAN IP addresses.

## On Encryption

An early draft of Discovery v5 integrated weak obfuscation based on XORing packet content
as an optional facility. As development of the protocol progressed, we understood that
traffic amplification, replay and packet authentication could all be solved by introducing
a real encryption scheme. The way the handshake and encryption works is primarily aimed at
these issues and is not supposed to ensure complete anonymity of DHT users. While it does
protect against passive observers, the handshake is not forward-secure and active protocol
participants can access node information by simply asking for it.

Node identities can use different kinds of keys depending on the identity scheme used in
the node record. This has implications on the handshake because it deals with the public
key used to derive the identity. Implementations of Discovery v5 must agree on the set of
supported identity schemes to keep the network interoperable and custom code to verify the
handshake is required for every new scheme. We believe this is an acceptable tradeoff
because introducing a new kind of node identity is a rare event.

Since the handshake performs complex cryptographic operations (ECDH, signature
verification) performance of the handshake is a big concern. Benchmarking the experimental
Go implementation shows that the handshake computation takes 500µs on a 2014-era laptop
using the default secp256k1/keccak256 identity scheme. That's a lot, but note the cost
amortizes because nodes commonly exchange multiple packets. Subsequent packets in the same
conversation can be decrypted and authenticated in just 2µs. The most common protocol
interaction is a FINDNODE or TOPICQUERY request on an unknown node with 4 NODES responses.

To put things into perspective: encryption and authentication in Discovery v5 is still a
significant improvement over the authentication scheme used in Discovery v4, which
performs secp256k1 signature 'recovery' (benchmark: ~170µs) on every packet. A FINDNODE
interaction with an unknown v4 node takes 7 packets (2x PING/PONG, FINDNODE, 2x NEIGHBORS)
and costs 1.2ms on each side for the crypto alone. In addition, the v5 handshake reduces
the risk of computational DoS because it costs as much to create as it costs to verify and
cannot be replayed.

## On Amplification and Replay

Any openly accessible packet-based system must consider misuse of the protocol for traffic
amplification purposes. There are two possible avenues of attack: In the first, an
adversary who wishes to attack a third-party host may send packets with 'spoofed' source
IP address to a node, attempting to make the node send a larger response to the victim
endpoint. In the second, the adversary attempts to install a node record containing the
victim's endpoint in the DHT, causing other nodes to direct packets to the victim.

The handshake handles the first kind of attack by responding with a small WHOAREYOU packet
whenever any request is received from an unknown endpoint. This is safe because the
adversary's packet is always larger than the WHOAREYOU response, removing the incentive
for the attack. To make the countermeasure work, implementations must keep session secrets
not just per node ID, but also per node IP.

The second kind of attack--- installing the victim as a node ---is handled by requiring
that implementations mustn't answer queries with nodes whose liveness hasn't been
verified. When a node is added to the Kademlia table, it must pass at least one check on
the IP declared in the node record before it can be returned in a NODES response.

An adversary may also try to replay previously sent/seen packets to impersonate a node or
disturb the operation of the protocol. Session keys per node-ID/IP generally prevent
replay across sessions. The `request-id`, mirrored in response packets, prevents replay of
responses within a session.

# Topic-based Service Discovery

This section explains the rationale for [TopDisc], the topic-based service discovery
extension to Node Discovery v5. TopDisc is based on the design described in [DISC-NG]
paper.

TopDisc addresses the problem of discovering peers that participate in a particular
decentralised service or application. The participants of such a service form a
service-specific overlay, but the discovery mechanism should not require each service to
operate a separate discovery network. A *topic* is the protocol-level identifier for a
service. The terms *topic-based discovery* and *service discovery* refer to the same
mechanism: discovering peers for the service identified by a topic.

## Service Discovery Performance Goals

The following performance goals are specific to TopDisc service discovery.

### Progressive Lookup Toward the Topic Identifier

Lookup should be able to find advertisements without immediately concentrating requests at
the nodes closest to the topic identifier.

The [TopDisc lookup procedure] begins in search buckets far from the service identifier and
adapts its preferred distance according to observed advertisement availability. Sparse responses
move lookup closer, where advertiser placements and discoverer queries are more likely to
overlap. Consistently full responses permit lookup to move farther away.

The goal is to spread lookup load across the service-centred key space while retaining
the ability to converge toward the service identifier for less widely advertised services.

### Efficient Discovery for Small Topics

Topic discovery should remain efficient even when the target topic is advertised by only a
small fraction of nodes in the global discovery network.

Random sampling over the ordinary node discovery network is robust, but it may require
many probes before finding enough peers for a rare topic. TopDisc reduces this cost through [advertisement placement] at registrars and by allowing
discoverers to query registrars selected from [service tables].

### Bounded Registrar Storage

A registrar should be able to bound the amount of memory it dedicates to topic discovery.
TopDisc addresses this through a bounded [ad cache]. Advertisements are soft state and
expire, so storage is reclaimed automatically unless advertisers renew their advertisements.

### Stateless Registration Operations

A registrar should not be required to store any state for advertisers waiting to be
admitted. TopDisc uses [tickets] to carry pending-registration state back to the advertiser.
This allows the registrar to validate a later retry without keeping per-request state
for every pending registration attempt.

### Compact Responses

TopDisc responses should remain small enough for UDP-based discovery. The [lookup response]
and [auxiliary ENR selection] rules bound the advertisements and auxiliary routing
information returned to a requester. This keeps responses compact while still allowing
requesters to improve their service tables.

### Incremental Service-Table Improvement

A node should be able to start TopDisc operations before it has a complete service table
for a topic. As described in [service tables], a service table is soft state. It is initially derived
from the ordinary node table and can subsequently be refined using [auxiliary ENRs]. This allows
lookup and advertisement placement to begin with partial knowledge and improve over time.

## Service Discovery Security Goals

The following security goals are specific to TopDisc service discovery. They complement
the Node Discovery v5 security goals listed above.

### Advertisement Flooding

A malicious advertiser may attempt to fill registrar ad caches with its own
advertisements, or with advertisements for services it controls, so that honest
advertisements are delayed or excluded.

TopDisc mitigates this attack using the bounded [ad cache] and [admission control]. Advertisements that increase cache occupancy or reduce cache diversity receive
longer waiting times.

### Registrar Resource Exhaustion

A malicious node may send many registration attempts, retries, or malformed requests in an
attempt to exhaust registrar memory, CPU, or bandwidth.

TopDisc mitigates memory exhaustion by avoiding unbounded per-request registrar state for
pending registrations. Pending registration state is carried by advertisers using [tickets]. Registrars may also apply local rate limits, request
validation, and temporary exclusion policies for nodes that repeatedly fail TopDisc
operations.

### Service Censorship

A malicious actor may attempt to prevent honest advertisements for a target topic from
being discovered.

This may be attempted by flooding registrars with competing advertisements, by returning
incomplete or misleading lookup responses, or by trying to control nodes in parts of the
key space relevant to a topic.

TopDisc mitigates this risk through distributed [advertisement placement] across the
topic-centred key space and through the [TopDisc lookup procedure], which can query
registrars across multiple buckets rather than relying on a single topic-specific location.

### Service Eclipse

A malicious actor may attempt to make discoverers of a target topic receive only
advertisements for malicious nodes.

Using advertisements from multiple registrars and encouraging diversity in registrar ad
caches can reduce the effectiveness of this attack. Distributed placements and registrar
selection provide opportunities for source diversity; lookup does not enforce a minimum
number of distinct responding registrars. The [lookup response] limits how much one registrar can return, while
[advertisement placement] distributes advertisements across multiple registrars. The
lookup result target counts distinct advertisers and does not itself guarantee diversity
of registrar sources. The lookup [mixes results](#why-filter-and-mix-lookup-results): it
interleaves queued results from available registrars where possible. Mixing is not a
minimum-source guarantee: one registrar may supply all the usable results, and every queued
result is released when the pass ends.

### Service-Table Poisoning

A malicious registrar may return misleading auxiliary ENRs in TopDisc responses in order
to pollute the requester's service table.

Implementations mitigate this by validating returned ENRs, requiring a supported TopDisc
capability before inserting nodes into service tables, and temporarily excluding nodes that fail TopDisc operations (see [service-table update rules]).
Auxiliary ENRs are routing information, not lookup results, and should not be treated as
proof that a node participates in the requested topic.

### Advertisement Redirection

A malicious node may attempt to advertise an ENR that directs discoverers to a victim
endpoint or to a node that does not actually participate in the advertised service.

TopDisc relies on ENR validation and on the self-signed nature of node records to prevent
intermediaries from modifying advertised node information. Applications using discovered
advertisements should still perform their normal service-level checks before relying on
the discovered peer.

## Why Not Use Separate Discovery Networks?

A simple way to discover service-specific peers would be to run a separate discovery
network for each service. Nodes interested in a service would join that service's
discovery network directly.

This approach makes each service responsible for its own bootstrapping and security. New
or small services would have few participants and would therefore be easier to isolate or
eclipse. Running many small discovery networks would also duplicate infrastructure and
fragment the global peer-discovery ecosystem.

TopDisc instead makes all nodes join the same network. Services benefit from the existing
global discovery network, and nodes can discover peers for many services without joining a
separate discovery DHT for each one.

## Why Not Use Only Random Sampling?

[FINDNODE] can be used to sample nodes from the discovery network, i.e. performing a
random walk. A service-specific protocol could use a random walk and then check whether
each sampled node supports the desired topic or service.

This approach has a useful security property: the search is spread across the global
discovery network rather than being concentrated at a small set of predictable
service-specific nodes. However, it is inefficient when the target topic is supported by
only a small fraction of nodes. In that case, many unrelated nodes must be contacted
before enough useful service peers are found.

TopDisc keeps the benefit of using the global discovery network, but adds explicit topic
advertisements so that discoverers do not need to test many unrelated nodes.

## Why Not Store Advertisements Only Near the Topic Identifier?

A simple DHT-style design would store all advertisements for a topic at the nodes whose
node IDs are closest to the topic identifier. Advertisers and discoverers would then know
where to place and retrieve advertisements.

This design is efficient, but it concentrates load and trust near the topic identifier.
Popular topics would create hotspots around their identifiers. More importantly, an
adversary could generate node IDs close to a chosen topic identifier and attempt to
control advertisement storage or lookup results for that topic.

TopDisc therefore does not rely only on the closest nodes to a topic identifier.
[Advertisement placement] distributes advertisements across the service-centred key space,
while the [TopDisc lookup procedure] can adapt the distance from which registrars are
queried. The rationale for that adaptation is discussed in
[adaptive search](#why-use-adaptive-search).

## Why Use Adaptive Search?

Always searching close to the service identifier offers high advertisement availability,
but concentrates lookup traffic and trust on a relatively small registrar population.
Remaining too far away distributes load, but can make sparse services difficult to
discover. A fixed number of queries at every distance cannot adapt to differing service
popularity and the current distribution of advertisements.

The [TopDisc lookup procedure] records the advertisement yield of each registrar query, defined as
the number of valid, de-duplicated advertisement records in the registrar's response.
Each yield sample is associated with the search bucket containing
the queried registrar.

The sample counts the records as the registrar sent them, before the discoverer's own
filtering. It measures how many advertisements registrars at that distance hold. Counting
after the result filter would make a registrar look sparse once the discoverer had seen its
advertisements, and would drive the search inward although the registrar is dense. A partial response counts
only if it reaches `SearchYieldFloor`, because its count is a lower bound: it can show that
a bucket is dense enough, but not that it is sparse.

As specified by the [TopDisc lookup procedure], once enough yield samples have been
collected for a bucket, they are used to choose the next search bucket. Let `lower` and `upper` denote the lower and
upper middle samples after sorting the observed yields.

If `lower >= F_return`, registrars in the bucket are consistently returning full
responses, so the search moves one bucket farther from the service identifier.
Otherwise, if `upper >= SearchYieldFloor`, the search continues in the same bucket.

When `upper < SearchYieldFloor`, the search moves towards buckets closer to the service
identifier. For a non-zero yield, the number of buckets moved is:

    ceil(log2(SearchYieldFloor / upper))

with the movement limited to four buckets. If `upper = 0`, the search moves up to four
buckets closer to the service identifier.

The adaptive rule therefore moves closer when the current bucket provides too few
advertisements, remains at the current distance when its yield is sufficient, and moves
slightly farther away when registrars are consistently returning full responses.

The jump size follows from [advertisement placement]. Advertisers aim for the same number of
registrations, `K_register`, in every bucket, while each bucket closer to the service
identifier covers half as much of the key space and so holds about half as many registrars.
Advertisements per registrar therefore roughly double with each bucket inward. A bucket
yielding `upper` advertisements per query is about `log2(SearchYieldFloor / upper)` buckets
farther out than the first bucket expected to reach the floor, so the search jumps there
directly.

A floor limits the load on registrars, and its value sets the speed of the search. In a
10,000-node testbed run, a floor of 4 cut the maximum number of TOPICQUERY requests received by
one registrar from 640,319 with no floor to 16,710. In a 30,000-node simulation of one large
service, floors of 2 and 4 settled where a response carries about four advertisements and took
1,330 s and 1,297 s to find 90 % of the service's providers. Floors of 8 and 12 took 377 s and
352 s, against 401 s with no floor, with half of its queries. With a floor of 8 the busiest
registrar received about 2,000 queries, against about 150,000 with no floor. `SearchYieldFloor`
is 8, the lowest of these values that keeps the speed of a search with no floor.

A yield sample from a fallback bucket more than `SearchAuxRadius` from the active bucket moves
it only in the direction that
advertisement density does not already predict. Low yield farther out than the active bucket,
or sufficient yield closer in, is expected and says nothing about the active bucket. Sufficient
yield farther out shows that the search can move out, and low yield closer in shows that it
must move further in.

## Why Search in Passes?

A lookup has no natural end: the caller reads results for as long as it needs peers. The
[TopDisc lookup procedure] therefore runs as a series of bounded passes, each starting from
the bucket the previous one settled on. The cap of `SearchPassQueries` queries per pass
bounds a pass even when auxiliary ENRs keep refilling the active bucket.

Registrar ad caches change on the time scale of the advertisement lifetime `E`. Asking the
same registrar again within that time mostly returns advertisements the discoverer already
has, so the asked set keeps answered registrars out of later passes for `E`. One spare
re-ask per pass, choosing the registrar asked longest ago, keeps a lookup going when every
known registrar has been asked. The failed set keeps a dead node from the routing table from
being picked again in every pass. A bucket whose queries all failed gets one retry pass, so
that a single dead registrar cannot hold the search at one distance.

When a lookup that started at the closest bucket sees no usable advertisement, the service may have
had no advertisers when the lookup reached that bucket. Starting again at the farthest bucket
lets the lookup find advertisers that appear later.

Passes that find nothing new are spaced out, doubling from `SearchPassGap` up to
`SearchPassBackoff`. In a 30,000-node simulation with late-joining advertisers, a 60 s cap cut
the TOPICQUERY requests received by the busiest registrar from 2,806 without back-off to 565. A
late joiner in a topic with fewer than 10 providers was found 0.6 s later at the median and
10 s later at the 90th percentile. For topics with 10 to 99 providers, t90, the time until a
searcher has found 90 % of its topic's providers, went from 321 s to 350 s; a 300 s cap raised
it to 510 s.

The discoverer stops sending queries while `SearchUnreadLimit` results wait unread. A caller
that reads slowly then does not cause queries whose results it would not use.

## Why Filter and Mix Lookup Results?

Successive passes query different registrars, which often hold the same advertisers. The
result filter keeps the caller from receiving the same advertiser again for an advertisement
lifetime. An advertiser that changes its ENR is delivered once more, so the caller learns the
new record, but only once, so that an advertiser cannot reach the caller repeatedly by
updating its record.

Without mixing, a pass would deliver the first registrar's whole response before any other
registrar's results. A caller that stops after its first few results would then depend on a
single registrar, which a malicious registrar could fill with sybil advertisements. Mixing
holds results until several registrars have answered, a short wait has passed, or the pass ends, and
then delivers them one registrar at a time. In simulation, mixing
cut the share of searches whose first results were all sybils from 21 % to 0.15 %. Mixing
does not remove the eclipse risk of a large `F_return`; see
[Eclipse Risk and `F_return`](#eclipse-risk-and-f_return).

`SearchMixRegistrars` is 10. In one simulation run of 50,000 nodes, with 20 % sybil nodes on
100 /24 networks and all IP limits applied, the share of searches that received only sybils in
their first 16 results fell from 0.27 % with 5 registrars to 0.02 % with 10. Recall and the
sybil share of all results remained approximately unchanged. The mean sybil share of the first
16 results rose from 38.5 % to 41.3 %. The median first result on this run's single large topic
arrives later, at 0.60 s rather than 0.30 s in the simulation's latency model. A search that
finds fewer registrars with results waits for `SearchMixWait` or the end of the pass.

## Why Topic-Centered Service Tables?

The ordinary node table is centred on the local node ID. It is useful for maintaining the
global discovery network and for finding nodes close to arbitrary node IDs. For topic
discovery, however, advertisers and discoverers need a shared reference point: the topic
identifier.

A [service table] is centred on the topic identifier rather than on the local node ID. This
gives advertisers and discoverers for the same topic a compatible view of the key space.
An advertiser uses the service table to choose registrars for advertisement placement. A
discoverer uses the service table to choose registrars for lookup.

Service tables do not replace the ordinary node table. They are derived from ordinary node
discovery state and refined using auxiliary ENRs returned by TopDisc responses.

## Why Use Registrars?

The TopDisc [node roles] define registrars as the nodes that store admitted advertisements
and return them to discoverers.

Registrars decouple advertisers from discoverers. Advertisers do not need to be online at
the exact moment a discoverer performs a lookup, as long as their advertisements remain
stored at registrars. Discoverers do not need to contact arbitrary nodes and test their
service membership one by one; they can query registrars for advertisements that have
already been placed.

Because any TopDisc-capable node can act as a registrar, the design does not depend on a
central registry or trusted topic-specific bootstrap node.

## Why Use an Ad Cache?

Registrars store admitted advertisements in a bounded [ad cache], where advertisements
are maintained as expiring soft state. The fixed advertisement
lifetime gives advertisers predictable soft state. Once admitted, an advertisement remains
available. Advertisers can therefore schedule renewal rather than continuously
re-registering at every moment.

The cache bounds registrar storage and makes advertisements soft state. Advertisers must
renew or replace advertisements over time, which allows the system to adapt to churn,
failures, and changes in service participation.

The cache is a hard limit. A full cache does not evict an admitted advertisement, so that a
new advertiser cannot push out an honest one by asking. It quotes `maxWaitTime` minus the
credited waiting time, and returns a ticket for the time of its next expiry only once the
credited waiting time is within `W_min` of `maxWaitTime`.

A registrar also drops the advertisements of an advertiser that its ordinary node table has
dropped for failing liveness checks. The registrar has then seen the advertiser fail, and
returning its advertisements would send discoverers to a node that is likely gone for the
rest of the advertisement lifetime.

The [ad cache] and [service table] serve different purposes: the former stores
advertisements at a registrar, while the latter provides the service-centred view used
to select registrars.

## Why Use Admission Control?

Registrars have finite storage. If every registration request were admitted immediately,
popular topics or malicious advertisers could dominate the ad cache and crowd out other
advertisements.

TopDisc [admission control] uses a [waiting-time function] to regulate when an
advertisement may enter the cache. Advertisements that would increase cache
occupancy or reduce diversity receive longer waiting times.

This makes flooding more expensive, promotes diversity across topics and IP prefixes, and
helps ensure that less popular topics can still obtain representation in registrar caches.

## Why Should Advertisers Wait?

The [waiting-time function] is the registrar's main admission-control mechanism.

The waiting time makes it costly to flood registrars with advertisements, limits the rate
at which the ad cache fills, and gives the registrar a way to prefer advertisements that
improve cache diversity. An advertiser that receives a waiting time must come back later
with a valid ticket before the advertisement can be admitted.

Waiting also avoids a simple replacement-policy problem. If registrars used only policies
such as least-recently-used replacement, an attacker could repeatedly send new
advertisements to evict honest ones. With waiting-time admission, the attacker must pay
the cost of waiting before advertisements are admitted.

## Why Use Tickets?

The [ticket mechanism] allows a registrar to enforce waiting without storing unbounded
per-request state.

A ticket records the advertisement binding and registrar-generated timing information
needed to show that the advertiser has waited. The ticket is carried by the advertiser and
returned to the registrar on retry. The registrar verifies the ticket and recomputes the
waiting time against the current cache state.

This design prevents pending registrations from consuming unbounded registrar memory. If
an advertiser never returns, the registrar does not need to clean up per-request state for
that advertiser.

Tickets are bound to the topic and the advertiser's node ID. This prevents a ticket issued for
one topic or node from being reused for another topic or by another node. A ticket is not bound
to the advertised ENR; see [Ticket Transfer](#ticket-transfer).

## Why Answer Every Registration Request?

An advertiser cannot tell a silent registrar from a dead one. If a registrar drops a request
it does not like, the advertiser sees a timeout: it holds the registrar in its failed set for
an advertisement lifetime and counts a liveness failure against it in its node table. The
registrar is penalised as if it had failed, and the advertiser loses a working registrar.

In a 25,000-node run with churn and unreachable ("ghost") nodes, with registrars that answer
nothing to a ticket outside its window, TopDisc searchers found a median of 35.7 % of their
topic's providers, against 72.3 % with plain discv5. The REGTOPIC traces of that run show
that 96 % of the ticket retries arrived after the 5 s window and
were dropped without an answer; these made up 76 % of all REGTOPIC timeouts. The [tickets]
rules therefore answer every request: a rejection with an explicit empty response, an early
ticket with the remaining wait, a late ticket within the grace `ε` with the wait it was
scheduled to have, and an expired or invalid ticket as a new attempt.

The retries that did arrive inside the window were late, relative to its start, by 50 ms at the
median and 3.8 s at the 99th percentile. The late grace `ε` credits a retry that misses the
window by a few seconds with the wait it was scheduled to have, instead of restarting its
wait. `ε` is short against `E`, so a ticket cannot be redeemed long after it was earned, and
the ticket issued in reply is re-based so that the lateness is not credited later either.

A ticket is credited once while the registrar remembers it. Without that rule, an advertiser whose advertisement was removed,
for example after the registrar's node table dropped it, could present the ticket that
admitted it again and obtain a new advertisement with a full lifetime without waiting. An early
ticket is returned unchanged rather than re-issued: a re-issued ticket would be a second valid
ticket for the same window, which the memory of credited tickets would not recognise. It also
keeps an early retry from learning the current waiting time.

A registrar admits any advertisement whose remaining wait is at most `W_min`, so it has no
reason to quote less. An honest registrar can still send a shorter wait: a full cache quotes the
time until its next expiry, and an early ticket comes back with whatever remains of its wait.
Dropping a registrar for such a quote would penalise correct behaviour, so the advertiser
waits `W_min` instead. A hostile registrar that answers every retry with a zero wait gains
little from this. The advertiser sends it at most one request per `W_min`, per-bucket
scheduling keeps it from delaying the other buckets, and the clamped waits add up to
`RegAttemptBudget`, after which the attempt is abandoned.

## Why Schedule Registrations Per Bucket?

A ticket retry is useful only inside its window. With a single sender per advertised topic, a
due ticket retry waits behind every first contact ahead of it, each of which can take a 700 ms
timeout when the registrar is dead, and misses its window. Several concurrent senders keep one
unresponsive registrar from holding back the rest.

Serving buckets in turn keeps a bucket with many new candidates from delaying the retries of
another bucket. Within a bucket, a due ticket retry goes before a first contact, because its
window is about to pass and a first contact can wait. The priority is per bucket, not global:
registrars that answer every request with a ticket cannot take all the senders.

The attempt budget `RegAttemptBudget` bounds how long an advertiser keeps waiting on one
registrar. After `E`, every advertisement in the registrar's cache has expired and been
replaced, so a registrar that still has not admitted the advertisement is overloaded or
malfunctioning, and the advertiser does better with another one. The budget is not a failure:
the registrar answered, so it goes to a separate overloaded set rather than the failed set,
and it may be tried again once it leaves that set after `E`.

The failed set keeps a registrar that timed out, rejected the request or sent a malformed
response out of the advertise table for an advertisement lifetime, from every source, so that auxiliary
ENRs and the node table do not offer it again at once.

Renewal takes the next standby candidate rather than the same registrar. This spreads an
advertiser's advertisements over more registrars across successive lifetimes, and a registrar
does not extend an advertisement it already holds, so registering again with it before expiry
would not help.

## Why Recompute Waiting Times?

The state of the ad cache may change while an advertiser is waiting. Advertisements may
expire, new advertisements may be admitted, and the diversity of the cache may change.

For this reason, the waiting time in a ticket is not binding. When the advertiser retries,
the registrar recomputes the waiting time using the current cache state. The advertiser is
admitted only if its accumulated waiting time is sufficient according to the recomputed
value.

This prevents stale tickets from forcing admission under cache conditions that no longer
justify it.

## Why Include Service Similarity?

The [waiting-time function] includes a service-similarity component that accounts for how
well represented the advertised topic already is in the registrar's ad cache.

This prevents popular topics from crowding out less represented topics. It also helps less
popular topics obtain cache entries even when the total registration demand is high.

## Why Include IP Similarity?

A malicious actor may create many node identities from a small number of physical hosts or
IP prefixes. Counting only node IDs would not distinguish this behaviour from a diverse
set of independent advertisers.

The [IP similarity tree] provides the IP-similarity component used by the
[waiting-time function] to enforce diversity among advertiser addresses. This discourages a small number of IP prefixes
from dominating the ad cache.

This approach is more flexible than a fixed rule such as "only one node per prefix". Fixed
prefix limits can harm honest users behind NATs, shared hosting providers, or other common
infrastructure. A similarity score instead increases cost gradually as concentration
increases.

## Why Change the Waiting-Time Function?

The [waiting-time function] differs from the form in [DISC-NG], which multiplies
`E / (1 - c/C)^Pocc` by `c(ad.service)/c + score(ad.IP) + G` for a small safety constant `G = 10^-7`.

The safety constant `G` is kept, as an additive term multiplied by the occupancy factor. It
adds less than `W_min` while the cache is less than about 69 % full, so at light load the
admission slack absorbs it. Near full occupancy it is the only cost charged to an advertisement
for a new service from a diverse address: it reaches `E` at about 84 % occupancy and
`RegAttemptBudget` at about 85 %. Above that, a registrar stops admitting such advertisements,
which keeps its last places free and is the flow control [DISC-NG] intends; see
[Admission Near Full Occupancy](#admission-near-full-occupancy). The safety term is not
lower-bounded: it falls with occupancy, while the service and IP components keep their bounds.

Using `max(c(ad.service)/c + score(ad.IP), 10^-6)` instead of an additive `G` would charge a new service from
a diverse address almost nothing at any occupancy.

The service term divides by `c + 1` instead of `c`, which defines it for an empty cache
without a special case. The factor `Wbase = 0.1` scales the occupancy term, so that an empty
cache charges at most `E / 10` for a fully similar advertisement.

The occupancy factor makes the waiting time grow very fast as the cache fills: about
`3 * 10^10` seconds at 86 % occupancy. Converted to a 64-bit nanosecond duration, such a value
is out of range, and an overflowing conversion can turn a long wait into an immediate
admission: a cache about 86 % full or more would then admit every request. The waiting time
is therefore capped at `maxWaitTime` before the
conversion. The cap is far above `RegAttemptBudget`, so a capped quote still makes the
advertiser move to another registrar, on every platform. For the same reason, advertisers
saturate a `wait-time` that does not fit their duration type.

## Why Use a Waiting-Time Lower Bound?

If waiting times could decrease freely as the ad cache changes, advertisers would be
incentivised to repeatedly request new tickets in the hope of obtaining a shorter wait.
This would create unnecessary traffic and processing load.

The [waiting-time lower bound] limits how fast the service and IP components of a new waiting
time can fall: while a bound is retained, a component charged later is at least the recorded
value minus the elapsed time. Bounds are capped at `maxWaitTime`, a service bound is dropped with
the service's last advertisement, IP bounds disappear with their branch of the IP similarity
tree, and the safety term is not bounded, so the limit does not hold beyond them. It reduces the
incentive to repeatedly request new tickets while keeping registrar state bounded.

The lower-bound state is maintained only for bounded structures, such as topics that have
advertisements in the ad cache and prefixes represented in the IP similarity tree.

## Why Return Auxiliary ENRs?

A service table is initially derived from the ordinary node table. However, the ordinary
node table is centred on the local node ID, not on the topic identifier. It may therefore
contain few nodes in buckets that are important for a particular topic. TopDisc responses
can include auxiliary ENRs selected from the registrar's ordinary node table by distance to
the topic. Advertisers and discoverers can use these ENRs to refine their local service
tables. Returned ENRs are auxiliary routing information, not lookup results.
Implementations must validate them and check TopDisc capability in the ENR before
inserting them into service tables.

When a request includes topic-distances for which the requester needs additional
candidates, the registrar returns at most one auxiliary ENR for each requested distance.
This keeps responses compact and limits the amount of routing information contributed
for any one requested distance.

During lookup, auxiliary candidates are requested for topic-distances
corresponding to the active search bucket and nearby buckets with available capacity.
With `SearchAuxRadius = 1`, this normally covers the active bucket and its immediate
neighbours. This helps replenish registrar candidates near the part of the
service-centred key space currently being searched. In a 30,000-node simulation with the
service sizes of a discv5 crawl, this radius cut the total number of TOPICQUERY requests from
35.4 million, with auxiliary ENRs requested for every bucket with free space, to 2.7 million.
Every service with fewer than 10 providers was still found in full.

## Why Support Mixed Deployments?

TopDisc is an extension to Node Discovery v5. During deployment, some nodes may support
only ordinary Node Discovery v5 while others support both ordinary Node Discovery v5 and
TopDisc.

Nodes that do not support TopDisc remain useful for maintaining the ordinary discovery
network. They are not selected for TopDisc registration or lookup operations, but they can
still participate in ordinary node discovery.

TopDisc-capable nodes advertise support in their ENR. This allows implementations to build
service tables from ordinary node discovery state while selecting only nodes that can
handle TopDisc messages.

## Why These Parameter Values?

The [parameters](./discv5-theory.md#parameters) marked Local policy are defaults, and those
marked MUST are fixed by the protocol. Where a measurement or a bound supports a value, this
section gives it. Where none does, the section says so, and the value is a working default
rather than an optimum.

### Advertisement Lifetime

`E` is 15 minutes. In a 24-hour crawl of the discv5 network, nodes left at 1.7 to 3.4 % per
hour, depending on how long a silence counts as leaving. With continuous renewal, that leaves
0.2 to 0.4 % of advertisement time pointing at a departed node at 15 minutes, against 0.8 to
1.2 % at 60 minutes. A 5-minute lifetime would save under 0.3 percentage points and about triple
renewal traffic. A 25,000-node run replaying this churn at 16 times the rate returned 45 of 87
million results from nodes down longer than `E`.

The value is not an optimum: 30 minutes also stays under 1 % across all nodes, at about half the
renewal traffic.
An [earlier evaluation](https://github.com/datahop/p2p-service-discovery/blob/b7f2324/doc/report_m4/reports/ad_lifetime.md)
preferred a short lifetime because the network then stays easier for new nodes to register in,
spreads discovered nodes better, and resists sybil and denial-of-service attacks better.

### Ad Cache and Waiting Times

`C` is 1,000. A 25,000-node simulation with the service sizes of a discv5 crawl and no attackers
compared 500, 1,000, 2,500 and 5,000, averaged over three runs. From 5,000 to 1,000, the
advertisements held by the busiest registrar fell from 1,011 to 233 and its REGTOPIC requests from
68,800 to 50,500. Recall stayed at 100 %, and t90 for services with 1,000 or more advertisers rose
from 109 s to 116 s. Advertisers abandoned a registrar more often, 131 times per advertiser against
23. At 500, abandonment about doubled again for a further fall to 138 advertisements. Occupancy of
the busiest registrar stopped at 20 to 28 % at every size.

`Pocc` is 10 and `Wbase` is 0.1. `Wbase` makes an empty cache charge at most `E / 10` for a fully
similar advertisement. With these values the safety term reaches `E` at about 84 % occupancy and
`RegAttemptBudget` at about 85 %. No measurement supports these two values.

`G` is the value from the [DISC-NG] paper, and no measurement in this document compares values.

`W_min` is 1 s: a remaining wait below it is too short to be worth another round trip. No
measurement supports this value.

`δ` is 5 s and `ε` is 10 s. `ε` credits a retry that misses the window by a few seconds and is
short against `E`, so a ticket cannot be redeemed long after it was earned. No measurement
compares other values of either.

### Advertise Table

The size of the advertise table is a tradeoff between registration performance and the load on
peak nodes, the registrars nearest the service identifier. Each bucket closer to the identifier
covers half as much of the key space, so it holds about half as many registrars to take the same
`K_register` advertisements. A larger bucket places more advertisements, but the traffic it adds
falls on those registrars. An
[earlier 2,000-node evaluation](https://github.com/datahop/p2p-service-discovery/blob/b7f2324/doc/report_m4/reports/tickettable.md)
found that registrations and traffic grew together with bucket size, without a matching gain in
lookups.

`K_register` is 10. The table gives a 50,000-node simulation with the service sizes of a discv5
crawl and no attackers, averaged over two runs. Registrars per advertiser is the median number
of registrars holding one advertiser's advertisements. Discovery time is t90 for services with
1,000 or more advertisers.

| `K_register` | Registrars per advertiser | REGTOPIC received per registrar, median / 99th percentile | Median admission wait | Discovery time |
|---:|---:|---:|---:|---:|
| 3  | 31  | 143 / 4,500   | 0.06 s | 369 s |
| 5  | 48  | 248 / 7,500   | 0.18 s | 305 s |
| 10 | 87  | 525 / 16,300  | 1.2 s  | 230 s |
| 16 | 128 | 858 / 24,400  | 4.1 s  | 240 s |

Discovery improves up to 10. At 16 it does not improve further, while the load grows by about
60 %.

`RegTableDepth` is 18. The closest bucket covers 2^-17 of the key space and absorbs anything
closer, so it reaches `K_register` registrars only at about 1.3 million nodes, and the depth does
not limit placement at the current network size. A smaller depth merges the registrars nearest
the identifier into one bucket, which lowers the peak load but leaves rare services harder to
find. In one 50,000-node simulation with 1,000 services of Zipf-distributed size, a depth of 10
cut the REGTOPIC requests received by the busiest registrar from 53,777 to 25,894. It raised t90
for services with fewer than 10 advertisers from 0.96 s to 15.9 s, and for 10 to 99 advertisers
from 0.96 s to 2.5 s. Recall did not change.

`RegStandbyLimit` is 20, enough candidates to replace a full bucket twice. It keeps the state of
one bucket to 30 registrars. No measurement supports the exact value.

`RegSenders` is 4, so that a ticket retry rarely waits for a timeout to a dead registrar to
finish. `RegLoopMinPeriod` keeps the registration loop from spinning when the local node table is
nearly empty. `OverloadedTTL` is `E` because the registrar's cache has then turned over.
`RegAttemptBudget` is `1.5 * E`: after `E` every advertisement in the cache has been replaced, and
the extra half lifetime keeps a quote close to `E` from ending the attempt. No measurement
supports these four values.

### Search Table and Pacing

`SearchTableDepth` is 18, the same as the advertise table, so that a bucket covers the same part
of the key space in both. `SearchBucketSize` is 8, which keeps at most 144 candidates held at
once. Results are not sensitive to the shape of this table: in a 30,000-node simulation with
a yield floor of 4, 32 nodes per bucket instead of 16 and 14 buckets instead of 10 changed
discovery time, total queries and the load on the busiest registrar by about 2 % or less. Eight
has not been compared with larger values at the current parameters.

`SearchYieldWindow` is 4 and `SearchMinSamples` is 2. A single response, which may come from a
hostile registrar, cannot move the search, and four samples keep it reacting within a few queries.
A search sends one query at a time, so that each sample can steer the next query. No measurement
supports these two values.

`SearchMaxJump` is 4. With `SearchYieldFloor` at 8, a non-zero yield moves the search by at most
`ceil(log2(8)) = 3` buckets, so the limit applies only to an empty yield. `AuxENRLimit` is 8: in
the 30,000-node table-shape simulation with a yield floor of 4, 16 auxiliary ENRs per response
made no measurable difference. `RegAuxAccept` equals `AuxENRLimit`. `QueryAuxAccept` is 16. No
measurement supports this value.

`SearchPassGap` is 2 s, the shortest spacing of passes, and only passes that find something new
stay at that spacing. `SearchPassQueries` bounds one pass. `SearchMixWait` bounds how long a
result can wait when too few registrars have answered. A search that reaches `SearchFilterSize`
forgets its oldest entries and may receive those advertisers again. No measurement supports these
four values. `SearchUnreadLimit` equals `F_return`, one full response.

### Search IP Limits

`SearchBucketPrefix` allows 2 registrars per /24 in a search bucket, and `SearchSubnetLimit`
10 distinct node IDs per /24 in one search, each counted for `E` from its first query. Both values
are those of the ordinary node table, which allows 2 per /24 in a bucket and 10 in the table.

A per-bucket limit alone has no memory: the search table is rebuilt every pass, so an attacker
with a few /24 networks can present new node IDs on each pass. In a 30,000-node simulation with
honest addresses from a discv5 crawl and an attacker on two /24 networks answering with useless
advertisements, the per-bucket limit alone left 92.1 % of queries going to sybils. With the limit
of 10 per search, 0.63 % went to sybils, and all 918 searchers reached 90 % of the service's
providers, with a mean t90 of 132 s. Limits of 1 and 2 per search also stop this attack, with
0.07 % and 0.13 %, but slow honest searches, to a mean t90 of 273 s and 175 s, against 124 s with
10. The result comes from one run. The attacker's allowance grows by 10 with each further /24.

The advertise table allows only one registrar per /24 in a bucket because a place in it holds an
advertisement for a whole lifetime, while a place in a search bucket costs one query. The
[eclipse results](#why-filter-and-mix-lookup-results) for `SearchMixRegistrars` were measured with
both limits in force.

### Protocol Limits

`ENRsPerPacket` is 3 because it is the most that always fits. A packet holds at most 1,280 bytes
and an ENR up to 300. Four ENRs take 1,200 bytes, which leaves 80 for a header and
authentication tag of at least 87 bytes.

`RegRespLimit` is 8 and `QueryRespLimit` is 16. `total` includes the REGCONFIRMATION, so
`RegRespLimit` leaves seven NODES packets, and in a TOPICQUERY the NODES and TOPICNODES packets
share the 16. The caps keep a registrar from holding a request open by announcing a large
`total` or by continuing to send packets. Since the timer restarts on each packet and every packet
counts towards the cap, they bound a request to about 5.6 s and 11.2 s. `MaxTopicDistances` is 256 because a request has one entry per log-distance.
`MaxTopicNodes` is 16, the FINDNODE result limit and `F_return`.

`RespTimeout` is 700 ms, the timeout of other discv5 requests. A timeout counts
against the node, so a shorter value would penalise slow but working registrars. No measurement
supports this value for TopDisc. `MinUDPPort` is 1024: discovery nodes do not use privileged ports,
and dropping such records keeps a response from directing a node's packets at system services.

### Ticket Bounds

Only the issuing registrar verifies a ticket, so its authentication is local. HMAC-SHA256 is one
example, and interoperability does not depend on it.

`TicketKeyUses` is 100,000. It limits how many outstanding tickets a single exposed key can
affect; it is not a limit of the MAC. No measurement supports this value. `TicketKeyLifetime` is
6 h from the key's last use, 16 times
`RegAttemptBudget`, so a retired key is still present for any ticket an advertiser keeps waiting
on. A ticket with a quote longer than the key lifetime can lose its credit. This is deliberate: an
advertiser abandons a registrar at `RegAttemptBudget`, so a quote of hours serves no honest
advertiser.

`UsedTicketLimit` is 16,384. A ticket can be replayed only within `UsedTicketTTL`, 15 s after it
is credited, so the limit covers about 1,090 credited tickets per second, against about 1.1
admissions per second for a full cache of 1,000 at `E = 15 min`. The MACs and expiry times take about 0.7 MB. Only
a flood of credited tickets reaches the limit, and the oldest entries, those of the flood, are
dropped first.


## TopDisc Security Considerations

The following security considerations apply to TopDisc.

### Ticket Transfer

The [ticket mechanism] binds a ticket to the topic and the advertiser's node ID. A ticket handed
to another node is not credited, so tickets cannot
be moved between identities, and a ticket issued for one topic cannot be used for another. A
ticket is not bound to the advertised ENR: a node that changes its record during an attempt,
under the same node ID, keeps its ticket and the waiting time it has credited. The required
waiting time is recomputed at admission from the presented record, so a record with a worse
address must still wait as long as that address requires; only the elapsed waiting carries
over. Single use limits each ticket to one admission while the registrar remembers it (see
[tickets]).

### Admission Near Full Occupancy

Without `G`, an advertisement for a service with no cached advertisements, from an address with
a score of `0`, would wait for nothing at any occupancy. Advertisements for many distinct
services from diverse addresses could then fill a registrar's cache up to `C`. With `G`, such
advertisements are quoted more than `RegAttemptBudget` above about 85 % occupancy, so an
attacker fills at most about 85 % of the cache this way, and only with addresses diverse enough
to keep their IP score at `0`: the IP similarity tree counts one entry per advertisement, so
many advertisements from one prefix raise its score. An advertiser quoted over its budget moves
to another registrar from its standby list.

### IPv6 Prefixes

Neither address rule aggregates IPv6 addresses at an operator's allocation, such as a `/48` or
a `/64`. The advertise and search tables apply the IPv4 `/24` prefix length to IPv6 addresses,
which groups addresses far more coarsely than any single allocation. The IP similarity tree
considers only prefix lengths `k` with `2^k <= N`, which is at most `9` bits with `C = 1,000`.
Two IPv6 addresses are therefore compared only on their first `9` bits.

### LAN Addresses

LAN addresses are exempt from the `/24` limits and are not scored by the IP similarity tree. A
registrar admits an advertisement only if its ENR address equals the packet source, so a LAN
address is accepted only from a node on the registrar's own network. In a deployment on one
network, such as a test network, these rules then give no address diversity at all.

### Unverified Auxiliary ENRs

A registrar returns auxiliary ENRs from its ordinary node table without checking that the
nodes are alive, unlike [FINDNODE] responses. The requester inserts them into its service
table before contacting them. A dead or malicious registrar candidate costs the requester a
timeout and then enters its failed set. The one-candidate-per-registrar-per-bucket rules bound
how many entries a single registrar can place in one bucket.

### Eclipse Risk and `F_return`

A large `F_return` raises the eclipse risk when one registrar can fill a caller's first results.
In a 30,000-node simulation with 10 % sybil nodes and no result mixing, an `F_return` of 16
eclipsed 10.3 % of searchers, against 0.2 % with an `F_return` of 6.

`F_return` is 16, the same limit discv5 applies to a FINDNODE result, so a TOPICNODES answer is
no larger than an ordinary NODES answer. With [mixing](#why-filter-and-mix-lookup-results) the
value is safe, and it balances eclipse resistance against search speed. With results mixed across
`SearchMixRegistrars` registrars and all IP limits applied, 0.02 % of searches received only sybils
in their first 16 results. In a 30,000-node simulation of one large service, an `F_return` of 6
raised t90 from 348 s to 1,006 s and lowered recall from 100 % to 97.9 %.

# References

- Petar Maymounkov and David Mazières.
  *Kademlia: A Peer-to-peer Information System Based on the XOR Metric.* 2002.\
  <https://www.scs.stanford.edu/~dm/home/papers/kpos.pdf>

- Atul Singh, Tsuen-Wan “Johnny” Ngan, Peter Druschel, Dan S. Wallach.
  *Eclipse Attacks on Overlay Networks: Threats and Defenses*. 2006.\
  <https://www.cs.rice.edu/~dwallach/pub/eclipse-infocom06.pdf>

- Ingmar Baumgart and Sebastian Mies.
  *S/Kademlia: A Practicable Approach Towards Secure Key-Based Routing.* 2007.\
  <https://telematics.tm.kit.edu/publications/Files/267/SKademlia_2007.pdf>

- Xin Sun, Ruben Torres and Sanjay Rao. *Feasiblity of DDoS Attacks with P2P Systems and
  Prevention through Robust Membership Management.* 2007.\
  <https://docs.lib.purdue.edu/ecetr/361>

- Erik Hjelmvik, Wolfgang John. *Breaking and Improving Protocol Obfuscation.* 2010.\
  <https://research.chalmers.se/en/publication/123751>

- Adam Langley, Wan-Teh Chang. *QUIC Crypto*. 2016.\
  <https://docs.google.com/document/d/1g5nIXAIkN_Y-7XJW5K45IblHd_L2f5LTaDUDwvZ5L6g>

- W3C Credentials Community Group. *Decentralized Identifiers (DIDs) Spec.* 2017.\
  <https://w3c-ccg.github.io/did-spec>

- Seoung Kyun Kim, Zane Ma, Siddharth Murali, Joshua Mason, Andrew Miller, Michael Bailey.
  *Measuring Ethereum Network Peers*. 2018.\
  <https://faculty.cc.gatech.edu/~mbailey/publications/imc18_ethereum.pdf>

- Yuval Marcus, Ethan Heilman, Sharon Goldberg.
  *Low-Resource Eclipse Attacks on Ethereum’s Peer-to-Peer Network.* 2018.\
  <https://eprint.iacr.org/2018/236.pdf>

- Michał Król, Onur Ascigil, Sergi Rene, Alberto Sonnino, Matthieu Pigaglio, Ramin Sadre, Felix Lange, and Etienne Rivière.
  *DISC-NG: Robust Service Discovery in the Ethereum Global Network.* 2024 IEEE 9th European Symposium on Security and Privacy (EuroS&P), pp. 193–215. IEEE, 2024.\
  <https://ieeexplore.ieee.org/document/10629017>

[wire protocol]: ./discv5-wire.md
[node records]: ../enr.md
[TopDisc]: ./discv5-theory.md#topic-based-service-discovery

[TopDisc lookup procedure]: ./discv5-theory.md#lookup-procedure

[advertisement placement]: ./discv5-theory.md#advertisement-placement

[service tables]: ./discv5-theory.md#service-tables

[ad cache]: ./discv5-theory.md#ad-cache

[tickets]: ./discv5-theory.md#tickets

[lookup response]: ./discv5-theory.md#lookup-responses

[auxiliary ENR selection]: ./discv5-theory.md#auxiliary-enr-selection

[auxiliary ENRs]: ./discv5-theory.md#auxiliary-enr-selection

[admission control]: ./discv5-theory.md#admission-control

[service-table update rules]: ./discv5-theory.md#updating-the-search-table-during-lookup

[service table]: ./discv5-theory.md#service-tables

[node roles]: ./discv5-theory.md#node-roles

[waiting-time function]: ./discv5-theory.md#waiting-time-function

[ticket mechanism]: ./discv5-theory.md#tickets

[IP similarity tree]: ./discv5-theory.md#ip-similarity-tree

[waiting-time lower bound]: ./discv5-theory.md#waiting-time-lower-bound

[FINDNODE]: ./discv5-wire.md#findnode-request-0x03

[DISC-NG]: https://doi.org/10.1109/EuroSP60621.2024.00019
