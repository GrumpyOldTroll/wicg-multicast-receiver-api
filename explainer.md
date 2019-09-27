# MulticastReceiver Explainer

## Problem and Motivation

The capacity of CDNs and others to deliver popular content at
scale is not keeping up with demand.

### Scaling Problem Examples

An example walkthrough with some real-world numbers was [presented](https://yana.techark.org/wp-content/uploads/2019/04/ietf104-vig-multicast-video.pdf) at
an IETF side meeting in 2019.  The arithmetic is simple, and reproduced here.

As an example for a sense of scale, we'll use the following value as a
high end for an achievable bit-rate for delivering popular traffic:

 * `72 tbps`: new record peak traffic delivery to end users, announced [December 2018](https://www.akamai.com/us/en/about/news/press/2018-press/akamai-hits-new-high-for-peak-web-traffic-delivered.jsp) by Akamai.

#### Linear Media Delivery

Using these values:

 * **40 mbps**: bit-rate for 4k video
 * **10 mbps**: bit-rate for 1080p video

Example reachable audience sizes:

 * **1.8m users** = 72 tbps / (40mbps / end user)
 * **7.2m users** = 72 tbps / (10mbps / end user)

For comparison, audience sizes of popular content:

Overall viewership:

 * 500m = Fifa World Cup Finals, 2018
 * 200m-300m = Cricket World Cup 2015 (when India is playing)
 * 100m = Super Bowl 2019

Online Viewership: 

 * 4m viewers = Twitch concurrent viewers [all time peak](https://twitchtracker.com/statistics)
 * 9m viewers = 2018 World Cup [peak concurrent viewers](https://www.conviva.com/peak-concurrent-plays-broke-records-world-cup-finals/)

What _is_ achievable?:

 * Most popular 2017-2018 Nielsen show with <1.8m viewers is ranked **#179**
 * Most popular 2017-2018 Nielsen show with <7.2m viewers is ranked **#56**

When one of the largest providers on the Internet can only handle the
179th-most popular TV show (doing nothing else) before setting a
new record for traffic, because of the Internet's unicast delivery model,
the Internet is really not living up to its potential.

#### Downloads

Consider a 1GB OS upgrade that has to be delivered to 1b devices.  What's
the average delivery time at a high-end peak delivery rate?

 * **1.3 days** = 1GB * 1b devices / 72 Tbps

Consider also that some OS or app updates are over **10GB**, and therefore
would take over **13 days** to deliver to 1b users, at a record-setting
aggregate delivery rate, doing nothing else.  (Sure hope there's no urgent
security updates!)

### Conclusion

We think these problems of scale are addressable with IP Multicast.

If web pages were able to subscribe to this kind of popular content and
have traffic replicated to many end users as part of delivery through the
network, it would have a transformative effect on the capability of the
Internet to handle these scaling issues, and would provide dramatic network
efficiency gains, even for much less popular content than the extreme
cases.

(The extreme cases simply can't be done satisfactorily on the
Internet by any means today, regardless of the real-world need to do
them sometimes.)

## Goals

 * Provide a way to subscribe to one-to-many multicast streams.

 * Provide an API that can be used by many use cases.  There are lots
   of protocols, some of them proprietary.  As long as they can live
   within the safety rules, they're all welcome.

 * Authenticate the traffic cryptographically.  An app can only receive
   data from a packet with cryptographic proof that it was sent by the
   correct sender, with a server-controlled origin policy.

 * Ensure network safety.  Using the API cannot blow out the network
   by signing up for too much traffic, because that will get noticed
   and the group will be pruned at the browser (and also probably in
   the network, if it succeeds at the browser).

## Non-goals

 * Sending.  This API does not do any outbound data traffic.

 * ASM (Any-Source Multicast).  This proposal only permits use of
   [SSM](https://tools.ietf.org/html/rfc4607) (Source Specific Multicast).
   (Note: A work-in-progress effort in the IETF is under way to
   [deprecate ASM for interdomain multicast](https://datatracker.ietf.org/doc/draft-ietf-mboned-deprecate-interdomain-asm/).)

 * Secrecy within your local network.

   Your IGMP or MLD membership report goes on the LAN, to a local
   network that can see your machine's IP and MAC address.

   The network is replicating the packets, and they know the global IP
   for source and group.  If they want to, they can probably find out
   what content it contains.

   This API is mainly for content that's so popular it's expected that
   lots of people will consume it, but if you can't let your local
   network know you're consuming it, you'll have to tunnel it in as
   secured unicast from somewhere else that can know.  It's NOT just you
   and the sender, and it cannot be, because the network is involved in
   replicating the packets.  You need to have a trust relationship with
   someone downstream of that packet replication, because they can find
   out what you're consuming.

   Upstream of your local network it gets more private--they'll only
   know that at least one machine inside your network is subscribed,
   without knowing who it was.  But you can assume your next-hop router
   will know that your machine has asked to receive a specific piece of
   content.

 * Encrypting the data.

   Although the sender MAY encrypt the data, it is not required,
   nor is it effective at providing privacy or secrecy in the intended
   use case of a scalable one-to-many distribution.  The contents of the
   packets are sent to many receivers, and some of them will not be
   trustworthy.  Any key that allows a receiver to decrypt it will also be
   available to the receiver's adversaries, perhaps because they are a
   legitimate customer to the sender and paid for it.

   That said, the data could for example be built out of encrypted
   segments, and the receiver could construct those segments and feed them
   into a player that provides its own guarantees about key control.

   Nothing prevents the data from being encrypted, it is entirely
   possible and likely for some use cases.  But nothing in this
   architecture provides or requires encryption for the transported data.

## Proposed Solution

### Overview

The API subscribes to a Source Specific Multicast (S,G) with a UDP port
number and a domain name for the authentication metadata.  After that,
it receives UDP payloads if the join is successful (and may of course
leave).

If the loss rate is high enough or if the total bandwidth would go above
a system-managed threshold, the join will fail with an error.

If the loss rate is above a safety threshold but below an emergency
disconnect threshold, there's a signal and grace period for the application
to gracefully transition to a state that maintains a lower loss level.

### Specifications

This API will build upon an ongoing set of efforts within the IETF.  The
basic idea is to provide a standardized and secure method for the browser
to discover and process metadata about the traffic streams, which it can
use to ensure safe operation.  (The network MAY also use the same
metadata to ensure network safety.)

 * DORMS (Discovery Of RESTCONF Metadata for SSM):

   [draft-jholland-mboned-dorms](https://datatracker.ietf.org/doc/draft-jholland-mboned-dorms/)

   This defines a mechanism for the metadata to get discovered and
   delivered to the browser and the network.  (It uses
   [RESTCONF](https://tools.ietf.org/html/rfc8040) and a
   [YANG](https://tools.ietf.org/html/rfc7950) model.)  The other 2
   (AMBI and CBACC, below), extend that data with the info they need to
   operate.

 * AMBI (Asymmetric Manifest-Based Integrity):

   [draft-jholland-mboned-ambi](https://datatracker.ietf.org/doc/draft-jholland-mboned-ambi/)

   This is how the browser verifies the integrity of the data received
   and gets the information to monitor packet loss, without needing to
   understand the payload of the protocol being delivered.  It sends
   packet hashes out of band in an authenticated stream, so the
   receiver knows whether it received what the sender sent.

 * CBACC (Circuit Breaker Assisted Congestion Control):

   [draft-jholland-mboned-cbacc](https://datatracker.ietf.org/doc/draft-jholland-mboned-cbacc/)

   This is how the browser (and the network) ensure that capacity
   limits are not exceeded.  This defines the behavior both for a
   policy-based limit, and for responding to loss by using the
   loss detection provided by AMBI.  It notices and cuts off
   multicast streams whenever someone has over-subscribed.

Note: other relevant extensions may be forthcoming. In particular, some
work is ongoing to define a transport for AMBI that can live inside the
same multicast channel as the data using
[ALTA](https://datatracker.ietf.org/doc/draft-krose-mboned-alta/) to
authenticate the packet manifests.  (We hope this will make for
some improvements in achievable latency targets.)
However, the components listed here are intended to be sufficient to
provide a safe core architecture, and to be adequate on their own for
solutions to scalability problems where network support for multicast is
enabled, and for use cases where the achievable latency is acceptable.

## Examples

### Basic Join, Sleep, Leave

A pretty dumb receiver.  It joins, waits 20s, and leaves.  Any packets
received are passed to the "processPackets" function.

```javascript
var mrc = new MulticastReceiverConfig();
mrc.source = '198.51.100.10';
mrc.group = '232.10.10.1';
mrc.port = 5001;
mrc.dorms = 'dorms.example.com';

var mr = new MulticastReceiver(mrc);
// process any packets received
mr.onmessage = function(evt) { processPackets(evt.data); }

// monitor other events, but do nothing with them.
mr.onjoin = function(evt) { console.log('multicast receiver joined'); }
mr.onleave = function(evt) { console.log('multicast receiver left'); }
mr.onerror = function(evt) { console.log('multicast error: ' + evt.data); }
mr.onloss = function(evt) { console.log('loss warning: ' + evt.data); }

mr.join();
setTimeout(function() { mr.leave(); }, 20000);
```

### Enhancement Layers

Join a base flow and an enhancement layer.  On a loss warning from
either, leave the enhancement layer and set a timer to rejoin in 10s.

```javascript
var mrc_base = new MulticastReceiverConfig();
mrc_base.source = '198.51.100.10';
mrc_base.group = '232.10.10.1';
mrc_base.port = 5001;
mrc_base.dorms = 'dorms.example.com';

var mrc_enh = new MulticastReceiverConfig();
mrc_enh.source = '198.51.100.10';
mrc_enh.group = '232.10.10.2';
mrc_enh.port = 5001;
mrc_enh.dorms = 'dorms.example.com';

var mr_base = new MulticastReceiver(mrc_base);
var mr_enh = new MulticastReceiver(mrc_enh);

// inserting my own variable
mr_enh.my_joined = 0;

mr_base.onmessage = function(evt) { processBase(evt.data); }
mr_enh.onmessage = function(evt) { processEnhancement(evt.data); }

mr_enh.onleave = function(evt) { mr_enh.my_joined = 0; }
mr_enh.onerror = function(evt) { mr_enh.my_joined = 0; }

function checked_join() {
  if (check_bw_estimate_ok() && !mr_enh.my_joined) {
    mr_enh.my_joined = 1;
    mr_enh.join();
  } else {
    setTimeout(10000, checked_join);
  }
}

function onloss(evt) {
  if (mr_enh.my_joined) {
    mr_enh.myjoined = 0;
    mr_enh.leave();
    setTimeout(10000, checked_join);
  }
}

mr_enh.onloss = onloss;
mr_base.onloss = onloss;

mr_base.join();
// leave time for bandwidth estimate, then join enhancement layer.
setTimeout(10000, checked_join);
```

TBD: flesh out the packet access API.  In particular, explain support for
[pathchirp](http://www.spin.rice.edu/Software/pathChirp/) and/or
[pathrate](https://www.cc.gatech.edu/~dovrolis/bw-est/pathrate.html) to
detect available path bandwidth, particularly for use in adding enhancement
layers/upshifting bitrates.  (We intend to support this by having network
rx timestamps given with the payloads as part of the API, so that apps can
use various packet dispersion techniques in conjunction with send patterns
known to the web app, in coordination with the sender.)

TBD: add examples of tunneling support/non-support.  This will be an
app-controlled bool in the MulticastConfigReceiver that permits or denies
establishing a unicast tunnel to a sender-known relay (via
[DRIAD](https://tools.ietf.org/html/draft-ietf-mboned-driad-amt-discovery-08),
or perhaps an equivalent via DORMS metadata) when multicast is not available
through the local network.  The 2 use cases are: 1. the app knows how to
fail over to unicast and retrieve the content with https when the local
network cannot provide native multicast; or 2. the app is consuming
something that is only available via multicast, and wants the payloads from
this stream even if it means tunneling the multicast data over unicast.

TBD: define the CBACC oversubscription threshold model for the browser.
The rough initial idea is to start with a threshold at ~1mbps, grow if it's
succeeding without loss, and shrink if loss goes persistently high.

TBD: detailed design.  This early draft is intended to solicit community
feedback about whether this general approach is doomed for reasons not
addressed in the IETF drafts, or whether attempting an implementation and
filling out all the details will actually be deployable, once it all works.

## Alternative designs considered

### Individual Payloads

An early prototype for the API used a separate onmessage call per UDP
packet, but at 10mbps this consumed 40% CPU in the renderer and 20% CPU
in the browser process on a Macbook Pro, so the new API will batch the
payloads, with a cap on the event rate to decide how many packets to batch.

### Specific Protocols

It's reasonable to question why we wouldn't solve the scaling problem by
defining an API for a specific transport protocol, such as
[FLUTE](https://tools.ietf.org/html/rfc6726) or
[NORM](https://tools.ietf.org/html/rfc5740),
or by adding support for [RTP](https://tools.ietf.org/html/rfc3550) with [RAMS](https://tools.ietf.org/html/rfc6285) or other
[multicast-enabling features](https://tools.ietf.org/html/rfc5760).

Part of the problem is that many existing applications (such as IPTV
systems currently deployed by some ISPs) don't use these protocols,
and we hope not to exclude their use cases, even though some of the
deployments use proprietary protocols with no publicly available
implementations.

Multicast IP also is an area of ongoing research with new protocols
still being developed, for example
[draft-pardue-quic-http-mcast](https://datatracker.ietf.org/doc/draft-pardue-quic-http-mcast/).  We believe enabling this kind of experimentation
with web apps will be extremely valuable.

Additionally, a number of gaming or communication protocols exist
that are built directly on top of unreliable UDP transport, and are
specific to the application itself.  These applications are also often
proprietary or least non-standard (but still sometimes deployed and
useful and could benefit from scalability).

The intent of the proposed architecture is to provide all the necessary
guard rails to ensure safety for the network and receiver so that
interdomain multicast can be delivered approximately as safely as other
kinds of traffic, while permitting experimentation that can realize the
untapped potential for a wide variety of uni-directional one-to-many
protocol designs.

### WebTransport

It has been suggested that this might fit as a unidirectional protocol
definition within [WebTransport](https://github.com/WICG/web-transport).

As it stands today, this proposal is not compatible with the WebTransport
goal of "Ensure the same security properties as WebSockets", because the
one-to-many nature of this proposal makes it impossible to provide the
same security guarantees as TLS.

After discussion with some of the WebTransport contributors, the advice
was to open this as an independent proposal.

If there's enough consensus that it's worthwhile, perhaps it's possible
to make the goals compatible in some way and to fold this proposal into
a subset of WebTransport.  But as of today the idea has been (very
tentatively) considered, and sits somewhere between 'rejected' and
'deferred'.

