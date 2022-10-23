+++
title = "Outage Stories: The Flapping BGP connection"
+++

When I started my career in wireless telecommunications, fresh out of school and wet behind the ears, I got assigned a project for a new technology introduction. 

This project was a disaster, but also helped me stumble into a number of issues impacting customers, one of which resulted in most of western canada having internet problems on their cell phones for several weeks.




Early in my career, I was on the operations team installing new equipment to overhaul the way the mobile network counted packets for cell phones. The new equipment was problematic from the start, and we always had trouble getting the byte counts to match the old system. 

The fun part is both systems worked differently and counted the packets at slightly different locations on the network. And by building the new system, we found counting issues in both the new and old systems.

One such case, was some customers could use data fast enough, that they would roll the byte counter around 0, and get a small bill. This was 10+ years ago, when cellular data was slow and expensive, and the entire cellular internet usage in Canada could fit on a single ethernet cable. I know some folks were in for a rude surprise of thousands of dollars when we fixed that bug.

The story I want to focus on today though is an investigation where I personally learned quite a few lessons.

# A little background

I'm going to try to keep this as simple as I can. Take a look at the map below.


At the time, cellular internet relied on a type of gateway to reach the internet. The gateway was the bridge, which provided a stable IP address and routing to the rest of the internet. At the time, the company I worked for only had these gateways in Toronto and Montreal. And as a mobile device moved around the network, it would attach at the closest location, and tunnel the traffic to the internet through one of these gateways.

Let's use a customer in Vancouver as an example. They would connect their phone to the internet, which would attach to a router in Calgary, that would tunnel to Toronto.

# The problem I was investigating...

So one of the big problems we had, that I spent weeks and months on, is looking at all the reasons that the old and new equipment for counting packets didn't match up. And trying to find any angle or correlation I could, until I stumbled into an interesting correlation. Customers in western Canada had more differences between both systems than elsewhere in the country.

So I started investigating what might influence our counting in Western Canada. But I was somewhat thwarted by, my company didn't own that part of the network. Canada is difficult to cover with cellular coverage, so at the time and still today, different carriers built out networks, but also made agreements with eachother to fill in coverage. And it was customers on this partner/competitor that were most impacted.

But I still investigated what I could, until I eventually came up with a theory I could test. The old and new systems were very different technologies, but should be generating the same packet/byte counts. It's not like a web page takes more bytes to transfer based on which equipment is doing the measuring.

There was one small difference though, for these western Canada customers, the old system was counting the packets and bytes in Calgary, and the new system was doing the counting in Toronto. But I did come up with an interesting theory. What if the old and new systems are both working. Is it possible that a packet from the internet, would go through Toronto and get counted, but not make it to Calgary. And not just one packet, but could many packets be getting lost.

So I started sending some pings... and to my surprise, something like 40% of packets were getting lost between Toronto and Calgary. 

# I stumbled into a giant issue

By losing packets, this was a much bigger problem than just installing some new equipment. Those packets weren't making it to the customers.

For anyone who doesn't have a background in networking, losing 1% of packets doesn't mean the website just loads 1% slower. Due to a series of network collapses in the early days of the internet, TCP stacks aggressively slow down their throughput when seeing almost any packet loss. This has changed over time, but at the time on an already slow wireless network, this packet loss might not just be the internet being slow, the web page might not load at all, or the email never received.

So I started pinging hop by hop on the network, until I isolated where the packet loss began. And upon logging into the router at the edge, between the networks, I discovered in the logs.

```txt
... bgp up ...
... bgp down ...
... bgp up ...
.... bgp down ...
```

Checking the hardware, the underling links appeared fine, with no apparent physical layer problems, errors, etc. But the BGP peering kept flapping, which exchanged routing information between networks. So no BGP, and no routes to send packets to the other network.

# Why was BGP unstable?

BGP itself operates by establishing a TCP connection between routers, that is then used to exchange routing information. This particular network link was incredibly funky for reasons I'm glossing over, but it did leave BGP at equal priority to all other network traffic. And same as I mentioned above, when the peering was established, there was tremendous packet loss, which would cause timeouts and errors on the BGP peering, withdrawing the routes.

The link was overloaded. When the peering was fully established, the network utilization was 100%. It was like trying to stuff 10 Mbps onto a 6Mbps link, some traffic has to be dropped.

# Mistakes were made

So a question some of you might be asking by now, isn't the job of a network operator to basically deliver voice, text messages, and internet? Shouldn't some notice that large swaths of western Canada can't load a web page most of the day? Or wouldn't there be a team that is monitoring usage, and would see these links are overloaded?

Well just like when a plane crashes, there are multiple things that had to go wrong together to allow this outage to occur.

The capacity engineering team uses capacity reports to figure out when it's time to upgrade network links. But there metrics are taken every 5 minutes, so they were looking at a chart that looked something like this:

<chart>

As far as anyone looking at that report would be concerned, is the network links are a hotter than they'd like, so they're planning for link upgrades, but there isn't an immediate concern. There is still some head way in there. 

But what they didn't know was the underlying link was flapping, so if they had metrics that say recorded every second, they would see something like this:

<chart>

But when the points are summed up for 5 minutes, you just don't see that the link is flapping and spending a minute out of every 5 minutes at 0% utilization.


What about the IP operations / routing team? Surely they would notice? Well they did, but somehow they developed an understanding that they weren't to worry about this link. IP engineering would give the team work orders, but the team had no context about what any of the links did. They just saw thousands of IP connections that went all over the country and to different equipment, and had no idea what held customer traffic and what didn't. This allowed a culture to occur, that these types of errors or problems weren't important, and there was nothing to react to. 

And the customer service and service oriented teams did notice, and had lots of complaints escalated from customers. But the company itself didn't have a big presence in the west coast, so it was difficult and time consuming to get any sort of independent tests done. So there were complaints, but no team knew how to move the investigation forward. And when asking around, the network teams because of the culture above, said everything is fine. The link reports are clean, and the operations team has no major issues that they're tracking. 

# It gets a bit worse

As I recall, and this was a long time ago, there was never an outage report done on this incident, no lessons learned, no investigation or brainstorming on how to avoid a similar incident. The report I started in the outage tracking system was closed by a manager indicating no customer impact.

The problem I think was unaligned incentives. One of my favorite Joel Spolsky articles, is Sins of Commission, about how incentive plans backfire. This was my direct experience with a similar incentive structure. That outage, if fully calculated, might've shown 5% of the customer base had significant service impacts for weeks. When you're chasing 99.999% or more availability, that outage would've torpedoed the numbers.

And everyone in the organizations bonuses were tied to availability numbers. So it was sort of the same as telling everyone to take a 20% pay cut, because the organization philosophy was that large chunks of compensation should be tied to performance. And it was a multiplier in the calculation. Bonus was something like (Base Pay * 20% * Personal Performance (0.2-1.2) * Availability Factor (0-1) * Overall Corporate Performance (0-1)). If you get a low number or a 0 in any of the columns, you got a shitty bonus when they multiplied together, even if your individual performance was stellar. 

So there was a massive incentive to not understand the issue, and bury the problem. 

# What did I learn from this

1. Understand what the tools are telling you and how they work. Something like a link utilization report, is just a tool, a calculation on some data. Because it says the link is 80% utilized, only tell's you what the report thinks it was. There can be missing data, corruption, that impact the report, or the sampling rate can be too slow to show you what's going on.
1. Having teams that don't know how their work impacts customers, leaves visibility gaps in their domain. For larger companies, where teams service teams who service teams who serve customers. The teams that are farthest away from the customer may have no idea how their work actually impacts a customer. So alarms can be ringing all around them, with no reaction. It's just noise.
1. Bad incentive structures in production teams hinders progress and can cripple investigations. Wherever there is wiggle room, a bias will be made to under report issues. The manager who closed the outage report, a year later, brought up the issue with me. And when I explained the impact, the response was, he had no idea it was so bad. The incentive structure can drive a lack of curiosity, dilligence, and actually wanting to deliver the best experience for customers.
1. When investigating problematic equipment, it's not safe to assume that it's always the culprit. That new packet counting equipment was a disaster, but not every problem was its fault. Slicing the data and having good mental models, allows you to develop and test theories. Even if the theory is absurd, sometimes it's still worth exploring. 
