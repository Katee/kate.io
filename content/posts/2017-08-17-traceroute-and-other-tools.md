---
title: "Traceroute and Other Tools"
uuid: "337bbdb9-7394-4fa2-8d40-0a0de5a22a0e"
date: 2017-08-17
slug: "traceroute-and-other-tools"
tags: ["traceroute", "star"]
categories: ["star"]
---
When you connect to another computer on a network your data usually travels through many devices (also called "hops", "gateways", or "routers"). I can use traceroute to list the devices my packets travel through to arrive at `steampowered.com` from the [Recurse Center](https://www.recurse.com/).

{{< highlight sh >}}
$ time traceroute steampowered.com
traceroute to steampowered.com (104.88.12.183), 64 hops max, 52 byte packets
 1  gateway.net.recurse.com (10.0.0.1)  6.991 ms  1.959 ms  1.961 ms
 2  207.251.103.45 (207.251.103.45)  3.264 ms  2.433 ms  3.217 ms
 3  te0-7-0-18.ccr21.jfk04.atlas.cogentco.com (38.104.73.241)  6.573 ms  4.164 ms  4.728 ms
 4  be2325.ccr42.jfk02.atlas.cogentco.com (154.54.47.29)  3.179 ms  2.550 ms  14.625 ms
 5  be2057.ccr21.jfk10.atlas.cogentco.com (154.54.80.178)  4.080 ms
    be2056.ccr21.jfk10.atlas.cogentco.com (154.54.44.218)  5.944 ms
    be2057.ccr21.jfk10.atlas.cogentco.com (154.54.80.178)  5.057 ms
 6  ae-13.r08.nycmny01.us.bb.gin.ntt.net (129.250.8.145)  3.689 ms  3.687 ms  3.398 ms
 7  ae-3.r07.nycmny01.us.bb.gin.ntt.net (129.250.6.176)  25.137 ms  16.652 ms  12.502 ms
 8  a104-88-12-183.deploy.static.akamaitechnologies.com (104.88.12.183)  4.710 ms  7.913 ms  4.445 ms
traceroute steampowered.com  0.00s user 0.01s system 3% cpu 0.214 total
{{< /highlight >}}

Here's a diagram representing the route:
{{< figure src="/images/traceroute-example.png" alt="Diagram of above network showing multiple devices at step #5" >}}

I found a couple of the steps interesting:

- \#1 is over Wifi to a router in the same room as me. It can take a long time to reach this!
- \#2 is ISP Stealth Communications which provides fiber optic internet in NYC. However no "symbolic name" was found for it, stealthy.
- \#5 has multiple lines because not all replies came from the same IP address. I think this might be the result of load balancing.
- \#8 is a response from the IP for `steampowered.com` and ends the route. Now I know it is served by Akamai.

## How do you trace a route?

Traceroute sends out [UDP](https://en.wikipedia.org/wiki/User_Datagram_Protocol) packets with "time-to-live" (TTL) values starting at 1 and increasing by 1. The TTL field is special because every device that processes a packet decrements it by 1. When a device decrements the TTL on a packet to 0 the packet is "dropped" and not sent to any other devices. Instead a [Internet Control Message Protocol](https://en.wikipedia.org/wiki/Internet_Control_Message_Protocol) (ICMP) "Time-to-live exceeded" packet is dispatched back to the original sender. When traceroute sends a packet with a high enough TTL to actually reach the final destination you usually get back "Destination unreachable (Port unreachable)" because the UDP packets sent to a destination port that is unlikely to be open.

I found the name "time-to-live" initially confusing because in other contexts "time-to-live" refers to actual *time* not "number of devices travelled through".

## Is it fast?

In the example the entire process took 214 milliseconds with 153 milliseconds of that spent waiting for packets to return. On the same network `ping steampowered.com` gets results in about 8 milliseconds (with some results coming significantly faster). However ping and traceroute aren't directly comparable because ping only needs to make one round trip while traceroute makes at least one round trip per device between you and the destination.

{{< figure src="/images/ping-traceroute-comparison.png" alt="Diagram of network showing ping taking one step and traceroute taking multiple steps" caption="Example of ping vs. traceroute on a fictitious route">}}

Another thing that makes traceroute slower is that by default three packets (called probes in the manual) are sent at each step. Also getting the symbolic names (e.g. `gateway.net.recurse.com`) isn't free! Unless the name has already been cached a <abbr title="Domain Name System">DNS</abbr> query is made.

The closest I can get to making traceroute behave like the simplified diagram above is to send only one packet per TTL (`-q` option) and disable looking up symbolic names (`-n` flag):

{{< highlight sh >}}
$ time traceroute -q 1 -n steampowered.com
traceroute to steampowered.com (23.33.112.147), 64 hops max, 52 byte packets
 1  10.0.0.1  4.793 ms
 2  207.251.103.45  7.811 ms
 3  38.104.73.241  5.402 ms
 4  154.54.47.17  5.815 ms
 5  154.54.80.2  9.054 ms
 6  38.104.74.6  6.528 ms
 7  216.151.177.249  7.988 ms
 8  23.33.112.147  9.635 ms
traceroute -n -q 1 steampowered.com  0.00s user 0.00s system 4% cpu 0.100 total
{{< /highlight >}}

The total time here 100 is milliseconds with 57 milliseconds being accounted for by the time spent waiting for the responses. That is fast enough for human consumption.

## Can it be faster?

It might be possible to speed up traceroute by sending out many probes with different TTLs simultaneously. The ICMP response includes part of the original UDP packet making it possible to identify responses even if they return out of order if you include something unique in the original packet (like a unique destination port). The manual mentions that "Some systems such as Solaris and routers such as Ciscos rate limit ICMP messages.". I sent 30 packets with TTLs between 1-30 simultaneously as a test and it mostly worked. However I only received a few responses for messages with a high enough TTL to reach the final destination which could be from rate-limiting.

Another big slowdown I experienced was the timeout when a hop returns no response. In that case traceroute waits `number_of_probes × timeout` before trying the next TTL. With the default settings the wait is 15 seconds (3 probes &times; 5 seconds).

## Other tools

There is a useful tool [mtr](https://github.com/traviscross/mtr) which is like a combination of traceroute and ping. It first gets ICMP responses like traceroute. Then it continually pings each device to display ongoing statistics. Unlike traceroute it will not find multiple IP addresses responding to the same TTL. I was happy to see that mtr is much faster that traceroute when a hop does not return anything.

I did experience something odd in mtr for hops where traceroute showed multiple IP addresses (e.g. #5 in the first example). For those hops mtr sometimes reports high rates of packet loss even though packet loss for the final destination is very low. This isn't a bug with mtr but instead says something about the underlying route!

## Resources

- [Code of Apple's traceroute](https://opensource.apple.com/source/network_cmds/network_cmds-77/traceroute.tproj/traceroute.c.auto.html)
- [mtr](https://github.com/traviscross/mtr)
