# MulticastReceiver

An API that allows web applications to join a multicast (S,G) and receive
authenticated, congestion-safe multicast IP traffic from services that
offer it.

It provides:

 * per-packet authentication
 * an origin-based security model
 * protection against over-subscription

This lets browsers join in subscribing to popular live media events or
file downloads (software or pre-recorded media) that make use of multicast
IP to enable the efficient use of network and server resources.

See the [explainer](explainer.md) for more info.
