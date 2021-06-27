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

Visit the [demo page](https://grumpyoldtroll.github.io/wicg-multicast-receiver-api/demo-multicast-receive-api.html) with a [browser that implements the API](https://github.com/GrumpyOldTroll/chromium_fork) to experiment with trying to receive traffic.  (You may need some extra steps to receive external traffic, see the [getting started](https://github.com/w3c/multicast-cg/blob/main/primers/01-getting-started.md) primer for help, and please open issues if you encounter trouble.)

