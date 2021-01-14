# MulticastReceiver Explainer

## Problem and Motivation

The capacity of CDNs and others to deliver popular content at
scale is not keeping up with demand.

### Scaling Problem Examples

A few different presentations have addressed the scaling problem that can be solved by multiast, but we'll refer here to some real-world numbers presented at NANOG 79:

 * [video](https://www.youtube.com/watch?v=2aihLUb1elg&t=4m56s)
 * [slides](https://storage.googleapis.com/site-media-prod/meetings/NANOG79/2209/20200530_Holland_Ip_Multicast_Next_v1.pdf#page=5)

NB: The presentation used [mistaken numbers](https://github.com/GrumpyOldTroll/wicg-multicast-receiver-api/issues/2) for video bitrate estimates, corrected below, but the overall point still holds.  The arithmetic is simple.

 * `167 tbps`: new record peak traffic delivery to end users, announced [April 2020](https://news.mit.edu/2020/3-questions-tom-leighton-managing-covid-19-internet-traffic-surge-0427) by Akamai.

#### Linear Media Delivery

Using these values:

 * **20 mbps**: bit-rate for 4k video
 * **5 mbps**: bit-rate for 1080p video

Example reachable audience sizes:

 * **8.35m users** = 167 tbps / (20mbps / end user)
 * **33.5m users** = 167 tbps / (5mbps / end user)

For comparison, audience sizes of popular events:

Overall viewership:

 * 500m = Fifa World Cup Finals, 2018
 * 200m-300m = Cricket World Cup 2015 (whenever India is playing)
 * 100m = Super Bowl 2019

Online Viewership: 

 * 6m viewers = Twitch concurrent viewers [all time peak](https://twitchtracker.com/statistics) in June 2020 (as of December 2020)
 * 9m viewers = 2018 World Cup [peak concurrent online viewers](https://www.conviva.com/peak-concurrent-plays-broke-records-world-cup-finals/)

What *is* achievable in 4k as a maximum upper bound?:

 * Most popular 2017-2018 Nielsen show with \<8.3m viewers is ranked **#43** ("Scorpion")

When one of the largest providers on the Internet can only handle the
43rd-most popular TV show with no extra capacity before setting a
new record for traffic because of the Internet's unicast delivery model,
the Internet is really not living up to its potential.

#### Downloads

Consider a 2GB OS upgrade that has to be delivered to 1b devices.  What's
the average delivery time at a high-end peak delivery rate?

 * **1.1 days** = 2GB * 1b devices / 167 Tbps

Consider also that some OS or app updates are over **10GB**, and therefore
would take over **11 days** to deliver to 1b users, at a record-setting
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

 * Sending.  This API does not do any outbound data traffic.  The functions in this API invoke a few outbound packets with IGMP or MLD, plus some other signaling protocols defined in the IETF's RFC series, but provides no capability to create app-controllable outbound traffic, outside of the specific narrow signaling to join multicast groups and process their traffic.

 * ASM (Any-Source Multicast).  This proposal only permits use of
   [SSM](https://tools.ietf.org/html/rfc4607) (Source Specific Multicast).
   (NB: [RFC 8815](https://tools.ietf.org/html/rfc8815) deprecated interdomain multicast using ASM)

 * Privacy and Secrecy within your local network.

   As a receiver, your IGMP or MLD membership report goes on the LAN, to a local
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
   into a player that provides its own guarantees about key control, as many existing DRM systems do.

   Nothing prevents the data from being encrypted, it is entirely possible and likely for some use cases.
   But nothing in this architecture provides or requires encryption for the transported data.
   This spec provides authenticated UDP payloads to the web app, and a different layer would have to understand the content of those payloads (including whether they hold encrypted data).

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

   [draft-ietf-mboned-dorms](https://datatracker.ietf.org/doc/draft-ietf-mboned-dorms/)

   This defines a mechanism for the metadata to get discovered and
   delivered to the browser and the network.  (It uses
   [RESTCONF](https://tools.ietf.org/html/rfc8040) and a
   [YANG](https://tools.ietf.org/html/rfc7950) model.)  The other 2
   (AMBI and CBACC, below), extend that data with the info they need to
   operate.

 * AMBI (Asymmetric Manifest-Based Integrity):

   [draft-ietf-mboned-ambi](https://datatracker.ietf.org/doc/draft-ietf-mboned-ambi/)

   This is how the browser verifies the integrity of the data received
   and gets the information to monitor packet loss, without needing to
   understand the payload of the protocol being delivered.  It sends
   packet hashes out of band in an authenticated stream, so the
   receiver knows whether it received what the sender sent.

 * CBACC (Circuit Breaker Assisted Congestion Control):

   [draft-ietf-mboned-cbacc](https://datatracker.ietf.org/doc/draft-ietf-mboned-cbacc/)

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

### JavaScript API

Here is an example that joins a multicast flow for 20 seconds.

```javascript
// Multicast flow to join:

let multicastFlow = {
  source: '198.51.100.10',
  group: '232.10.10.1',
  port: 5001,
  dorms: 'dorms.example.com'
};
  
// Construct MulticastReceiver and subscribe to the multicast flow on the 
// network:

let multicastReceiver=new MulticastReceiver(multicastFlow);

// Read multicast UDP packets:

let multicastReader=multicastReceiver.readable.getReader();

async function readMulticastData() {
  for(;;) {
    let { done, value } = await multicastReader.read();
    if(done) {
      return;
    } else {
      // value is an UInt8Array with the payload of one UDP packet.
      console.log("Got multicast packet with size "+value.length);
    }
  }
}

readMulticastData().then( () => {
  console.log("Cancel was called.");
}).catch( error => {
  console.log(error,"Error. Closecode is "+multicastReader.closecode);
});

// Cancel after 20 seconds:

setTimeout( () => {
  console.log("Canceling multicast");
  multicastReader.cancel();
},20000);
```

The browser will try to join the multicast flow on the network as soon as MulticastReceiver is constructed. The browser will leave the multicast flow on the network whenever there is an event that will trigger either a) a promise returned by `read()` to be rejected or b) the `done` value to become true. After such an event, no more multicast data can be read from that MulticastReceiver. So if JavaScript wants to receive the same or another multicast flow later, it will need to construct a new MulticastReceiver.

In case the promise returned by `read()` is rejected, it will be rejected with an informative text which may be useful for debugging and which is logged to the console in the example above. In addition to this, the closecode will be set to one of several well-known values such as:

* `MulticastReceiver.FLOW_PROBLEM`: There was a problem with the multicast flow itself. For example:
  * DORMS could not identify a multicast flow on the given source and group.
  * The multicast flow had a higher bandwidth than announced by DORMS.
  * Multicast packets were received but could not be authenticated.
* `MulticastReceiver.LOCAL_PROBLEM`: There was a problem on the local device that prevented reception of the multicast flow. For example:
  * A socket to receive multicast on could not be created or was closed.
* `MulticastReceiver.OVERSUBSCRIBED`: A CBACC oversubscription threshold (see below) was reached so the left the multicast flow or did not join it at all.

### Getting DORMS metadata

Consider the subscription to the multicast flow from the example in the previous section:

```javascript
{
  source: '198.51.100.10',
  group: '232.10.10.1',
  port: 5001,
  dorms: 'dorms.example.com'
};
```

Here `dorms.example.com` is a domain name and an optional port (default is 443) where the browser will try to get metadata for the multicast flow with HTTPS as described in the DORMS specification. Among other things, this metadata must include the trust anchor used by AMBI to perform integrity checks on the multicast flow. When downloading the metadata, the browser will verify that the server presents a valid certificate for `dorms.example.com`. This ensures that a chain of trust is established from the JavaScript that subscribes to a multicast flow and to the integrity checks the browser performs using AMBI.

We also want to ensure that the browser only subscribes to a multicast flow which wants to be subscribed to by a browser. This is to prevent for example:

* A malicious website to subscribe to a multicast flow private to a network.
* A pirate website to subscribe to a multicast flow from a normal website.

There are two mechanisms in place to prevent this:

* When downloading DORMS metadata with HTTPS, the browser will perform [CORS](https://fetch.spec.whatwg.org/#http-cors-protocol) checks.
* The DORMS server provided by JavaScript needs to be the same identified by the DORMS reverse-DNS scheme described in the DORMS specification (see next paragraph).

The second bullet ensures that JavaScript can only subscribe to a multicast flow from a given source IP if the reverse DNS of that source IP has been setup to allow it. More precisely, when receiving the multicast subscription in our example, the browser will make a reverse-DNS lookup for a `SRV` record for the domain `_dorms._tcp.10.100.51.198.in-addr.arpa`. In order for the subscription to succeed, this must give a `SRV` record with the hostname `dorms.example.com` and port 443. A different port can be used if provided in the JavaScript API as well. Obviously, DNS is usually not considered secure, so in some cases it may be possible to hack DNS to circumvent this check. This problem may need further considerations.

### TBDs

TBD: In the current API, one packet is delivered at a time to JavaScript. This is similar to how datagrams work in [WebTransport](https://github.com/WICG/web-transport). We may want to deliver multiple packets at a time for performance reasons. We are currently working on profiling this in Chromium.

TBD: It would be nice if JavaScript had access to precise timestamps for when packets are received by the OS or at least by the browser from the OS. This could allow JavaScript to implement protocols for detecting the amount of available bandwidth using mechanisms such as [pathchirp](http://www.spin.rice.edu/Software/pathChirp/) and/or [pathrate](https://www.cc.gatech.edu/~dovrolis/bw-est/pathrate.html). Do we want to support this? And if we do, what should the API look like? It seems like an extra field in the object returned by `read()` would be an obvious place. But this may not be compatible with the ReadableStream system. So we may need to wrap the UInt8Array with the packet content in an extra object?

TBD: Define the CBACC oversubscription threshold model for the browser.
The rough initial idea is to start with a threshold at ~1mbps, grow if it's
succeeding without loss, and shrink if loss goes persistently high.

TBD: add api extensions for loss rate reporting ahead of cbacc+ambi-based cutoff.  need some kind of early warning from stats.

TBD: We may want to extend the API such that the browser can inform JavaScript that a CBACC oversubscription threshold is about to be reached. This may allow JavaScript to decide which multicast flows to unsubscribe from rather than relying on the browser to select one or more flows and close them with an `MulticastReceiver.OVERSUBSCRIBED` closecode.

TBD: If JavaScript is not reading any packets, should the browser close the multicast flow with a specific closecode? Related to this, should there be an API where JavaScript can set the maximum number of packets that can be queued up internally in the ReadableStream?

TBD: Should the browser unsubscribe automatically from a multicast flow on which no data is received from the network? Most likely the answer is *no* as subscription to a multicast flow without data can be used to signal to the network that it may consider creating the flow.

TBD: AMBI may give information about dropped packets. Do we want that information to be available in the API? Perhaps by the reading of a special packet that indicates one or more dropped packets.

TBD: Detailed design.  This early draft is intended to solicit community
feedback about whether this general approach is doomed for reasons not
addressed in the IETF drafts, or whether attempting an implementation and
filling out all the details will actually be deployable, once it all works.

## Alternative designs considered

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

