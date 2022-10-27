+++
title = "Outage Stories: The Flapping Link"
+++

Early in my career I was assigned a project to introduce a new way of counting the bytes used by cell phones for the carrier I worked for. This project itself was a complete disaster, which is a story for a different time, but this project also helped me uncovered and understand a number of other network problems.

The story I want to focus on today, covers degraded service, where for weeks, if not longer, cellular internet service in Western Canada was unstable. And explore what circumstances allowed the outage to occur.

# A little background

I'm going to gloss over a lot of the more nuanced details in this post and a lot of technical specifics of cellular networks. These specifics aren't that important for the contents of this post. 

What is important to know, is this story revolves around the connections between two cellular carriers in Canada. Canada is vast, and building complete nationwide coverage is expensive. So the carriers using common technologies made agreements to roam on each other to extend coverage. This wasn't an exact divide. High customer density areas would still be built by each carrier, but as you moved out of coverage you'd switch to the other carrier. This is different from today, where carriers build a common radio network and subdivide who builds what part of the country.

The link that failed was between Toronto (our network) and Calgary (partner network). So a user in Western Canada, would route internally in the partner network to Calgary, cross the link between companies from Calgary to Toronto, and go through the home network in Toronto to reach the internet.

From an internet routing perspective, the cell phone appeared to be in Toronto, and then routed through the cellular data network to reach the device wherever it was in Canada. 

If this sounds a bit silly or high latency, it was. But at the time every cell phone combined on the network fit on a single 100 Mbps ethernet connection.

# The two systems don't match

The project I was working on was introducing new equipment to count the bytes used by customers for billing purposes. Normally, this was just done directly by the cellular equipment as it processed the packets, but my company wanted to introduce a bunch of new capabilities and flexibility the existing equipment didn't support. 

So naturally, when introducing new traffic counting, we compared the counts per customer against the old equipment. And the problem was, they were different, not even close to producing the same results.

So a big part of my job became figuring out why these two systems were producing wildly different usage counts.

And I found lots of problems, in both the new and old equipment. The new equipment, for example could miss a message and get out of sync with which customer has what IP address. And start counting traffic for one customer against the last customer who had that IP address. So we had to develop a reconciliation that could throw out those dirty records. 

On the old system, we had problems with rolling the byte counter. So a user who used more than 4GB (32bits) within a 24 hour record, would wrap back to 0. So some users figured out if they used enough data fast enough and disconnected, they got a small bill instead of a giant one. And some customers were in for a surprise when we fixed that bug.

One enterprise customer I heard about, was doing some sort of video streaming. When they did the trial, they were using just a bit over 4GB, so were getting tiny bills. So they signed a contract. Then we fixed the bug a month or two later, and their bill skyrocketed. They thought we did a bait and switch, even though it was a far more innocuous, we just discovered and fixed a bug.

But with all these problems and fixes, the systems still didn't match. I built a series of programs and scripts that did comparisons of records to identify which records were different and why. And one day when trying to break the data down in different ways, I separated the data by geographic region within Canada. And to my surprise, Western Canada was way worse than the rest of the country for records that didn't match between systems. 

# Why Western Canada?

In theory, there shouldn't really be anything special about Western Canada. Although another carrier provided the cell sites and access network, they used basically all the same equipment we did. It shouldn't be one of those crazy [500-mile email problems](https://web.mit.edu/jemorris/humor/500-miles), as we didn't see problems in Eastern Canada. 

As with any troubleshooting, I started to develop, test, and reject theories and possible explanations. Until I noticed an interesting and subtle correlation. We recorded separate counters for upload and download, and it was only the download counters that didn't match. This didn't match any of the many problems with this new equipment I had been investigating for weeks.

And when iterating on theories and possible explanations, I hit on one that turned out to be unexpected. What if the equipment was actually working. What if the two systems were counting different byte counters, because they physically saw different numbers of bytes. 

One of the details of this new byte counting, was it always counted in Toronto. But the old system, collected its packets from the access network. For Western Canada packet counting took place in Calgary in the partner network. 

So I started sending some pings... and to my surprise, I discovered a huge amount of packet loss between Toronto and Calgary. 

# Turns out I stumbled into a giant issue

By losing packets, this was a much bigger problem than just my project of introducing new features. If those packets are getting lost before reaching the partner network, they're also not making it to customers. 

For anyone who doesn't have a background in networking, losing 1% of packets doesn't mean the website just loads 1% slower. Due to a series of network collapses in the early days of the internet, TCP stacks a decade ago aggressively slow down their throughput when seeing almost any packet loss. This has changed over time, but at the time on an already slow wireless network, this packet loss might not just be the internet being slow, the web page might not load at all, or the email never received. And I'm pretty sure I saw way higher than 1% packet loss. 

So I started pinging hop by hop on the network, until I isolated where the packet loss began, and isolated the link directly between companies. And upon logging into the edge router, between the networks, I discovered in the logs.

```txt
... bgp up ...
... bgp down ...
... bgp up ...
.... bgp down ...
```

Checking the hardware, the underling links appeared fine, with no apparent physical layer problems, errors, etc. So the link that carried packets initially appeared healthy, but the routing protocol running on top this apparently stable link was flapping up and down. 

And if the routing connection is flapping, routes that use that link will be added and withdrawn. While down, traffic would be rerouted via another set of links in Montreal. 


# Why was BGP unstable?

This particular network link was incredibly funky for reasons I'm glossing over. BGP itself operates by establishing a TCP connection between routers, that is then used to exchange routing information. While the BGP connection is alive, it will send keep alives to test the peer is alive. If those keepalives don't make it, the peer must be having a problem, so we should withdraw the routes, and alternative paths may be used if possible.

So why weren't the keep-alives making it?

Well besides the flapping BGP connectivity there was one other sign of a problem. Packet drops due to network interface overload. We were trying to send more packets than the link could handle, and it was not only causing packet loss, but the BGP link to flap. 

So the routing table kept flapping, rerouting traffic through Montreal. Then while traffic is re-routed, the link would be idle. So BGP would come back up, see a better path, and re-create the routes through the link. The link would overload again, BGP would time out. Rinse and repeat.

# Mistakes were made
As much as we all hate our local telecoms, at least in North America, there is an incredible culture around stability. While the current big tech companies were developing SRE and production cultures, Telecoms were running incredibly reliable and redundant systems.

So a question some of you might be asking by now, isn't the job of a network operator to basically deliver voice, text messages, and internet? And these are billion dollar companies, don't they have teams dedicated to just managing that enough capacity is available. Shouldn't someone notice and be planning months ahead of time for capacity needs of the system. 

Well just like any complicated failure, there were multiple small issues that combined in the exact right way to allow this problem to impact customers.

One of the most key mistakes, was how the capacity team monitored capacity and timed upgrades and increases in link sizes. There charts were based on interface counters, that were aggregated into 5 minute intervals.

So the capacity engineers were looking at something like this:

![link-utilization-1s](link-utilization-5m-light.svg#gh-light-mode-only)
![link-utilization-1s](link-utilization-5m-light.svg#gh-dark-mode-only)

As far as anyone looking at that report would be concerned, is the network links are a hotter than they'd like, so they're planning for link upgrades, but there isn't a rush. So they can take their time with ordering new links and getting them setup. The new links were only a couple of weeks away when I discovered this problem.

But what the capacity team didn't know was the underlying link was flapping, so if they had metrics that say recorded every second, they would see something like this:

![link-utilization-1s](link-utilization-1s-light.svg#gh-light-mode-only)
![link-utilization-1s](link-utilization-1s-light.svg#gh-dark-mode-only)

The capacity team just couldn't see it. 

What about other teams, such as the IP operations and routing team? Surely they would notice. It turns out they did notice, but somehow they developed an understanding that they weren't to worry about this link. 

In software development we use Code Reviews and Pull requests for making changes. Have more than one engineer check over work and changes. In Telecom at the time, this check and balance was by having different teams. Operations are separate from Engineering as a discipline. So engineering would submit work orders to operations for changes. And the mobile wireless teams in a similar structure would drive changes with the IP routing teams. 

This left the IP operations team several layers removed from what any of their links were actually for. So out of the 10,000 links they were responsible for all over the country, there is some link on an edge router in Toronto connected to another cellular carrier in Calgary. So missing the context on what this link did, the team just ignored the problem.

There were at least two other teams aware of problems, but didn't have what they needed to figure out the problem. 

There was a VOIP product that ran on the data network instead of the circuit network, which had complaints of garbled voice. But working off customer complaints and being a service team removed from the underlying mobile network, they didn't have what they needed to identify packet loss as a culprit. 

And the custom service escalation teams did notice, and had lots of complaints escalated from customers. But this is a cellular network with radio connections, there are lots of reasons a web page doesn't load on a phone on those early data connections. And queries to the network teams got responses that the network is healthy. So there investigation steered towards explanations that would not be caused by the underlying IP network.

# Culture

What I find fascination about this particular incident, is this wasn't a hard problem to solve. It's barely a technical issue, the link was too small. It got replaced with a bigger link.

Why I find this fascinating, especially with perspective years later, is this was one of my first experiences with how organizations fail. These were all relatively small mistakes any of us could make. But teams with different missions were losing context on how their piece of the network, how their work directly impacted customers. 

But there was also a larger cultural problem. For a culture that's supposed to be about availability and stability, far beyond what almost any SRE team strives for today, there was no outage report. I think I tried to open one, and before it was filled out, a manager went in and closed it indicating no customer impact. So there was no organization effort to understand what happened, what improvements can be made, how to prevent reoccurrence. Any sense of an investigation was discarded once the link capacity increased.

I believe the problem was unaligned incentives. One of my favorite articles on the topic, is [Sins of Commission by Joel Spolsky](https://www.inc.com/magazine/20081001/how-hard-could-it-be-sins-of-commissions.html?partner=fogcreek), about how incentive plans backfire. The TLDR is that directly connecting employee compensation to some sort of company metrics, is almost guaranteed to backfire. In our case, if the outage impact had been calculated, it might've blown the availability numbers for the year. 

And the problem is, everyone's bonus was tied to the availability numbers. And the bonus was a big component of take home pay at that company. 

So there was a massive incentive to not understand the issue, and bury the problem. It doesn't even have to be nefarious, the context of the impacts of this problem was all spread out. So customer service teams knew customers were complaining, but the manager who closed the outage investigation, knew a network link was bouncing. That managers understanding was that the service was somewhat working and there was no context this impacted customers at all. The incentive structure nudged the team towards not being curious, and not wanting to find out or learn how degraded service impacted customers.

As an aside, customers were openly complaining on internet forums about how their cell phones didn't work. So customers definitely noticed this problem. 

The only reason I have the context to tell the story years later, is that I had relationships with each team that was impacted. And kept a silent eye on the forums for what customers were discussing amongst themselves. 


# What can we learn from this

I try to always remind myself that we don't know what we don't know. I could've easily and readily made any of the mistakes that lead to this outage. To hit on a few key points:

- Troubleshooting tools don't necessarily tell you what you think they do. It's important to know how the tools actually work. For example, metrics are a proxy for what's happening. But are also aggregated or removing precision to make the data accessible and useful. 
- Be conscious on the incentive structure and culture. Misaligned incentives can be dangerous, and lead to cultures that act contrary to what leadership is trying to achieve. Push back on cultures and incentives where appropriate that cause these sorts of problems.
- Not knowing how your work impacts your customers, or the customers of your customers, or why someone pays you for your service, leaves blind spots. Combined with a lack of curiosity, and problems can be allowed to just sit their and fester. So alarms can be ringing and there's no reaction... it's just noise for someone who doesn't know what it's for. This is doubled when the team hearing the alarm isn't the one to fix it, such as capacity increases done by engineering and capacity teams. 
- Be curious. I think intellectual curiosity is more important than many people realized, especially when responsible for production. There still needs to be a balance on not getting stuck in rabbit holes and getting things done. But that problem you see, might be making a customer have a miserable day. And they may not have a mechanism to yell loud enough to draw your attention to the problem right under your nose.

Disclaimer: This story is from more than a decade ago, and doesn't accurately reflect the present state of any company or network. Any opinions expressed are my own.