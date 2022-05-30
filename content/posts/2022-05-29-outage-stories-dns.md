+++
title = "Outage Stories: Cell Phone Roaming DNS Outage"
+++

When I used to work in Telecom networks, on technologies like LTE or UMTS, I was often tasked with investigating and resolving hard problems. This post covers an interesting outage I investigated which caused users roaming into Canada from their home networks fail to get an internet connection. Some of these users would be redirected to a competitor, but we would've lost the roaming revenue. 

The outage was caused by the uniqueness of the way wireless carriers use DNS, a configuration discrepancy, and a subtle behavior of the way the DNS protocol works.

I've been out of the industry for 5 years now and am reconstructing the story, so there may be some inaccuracy.


## Background
At the time, network operators such as the one I worked for generally operated as integrators of proprietary solutions. Think of a company like Cisco, that makes large scale routers for carrier grade networks, the same applies to Wireless. This generally creates a few challenges, troubleshooting requires builtin tools, logs, metrics or to look at the equipment externally, such as by taking traffic captures. We rarely had the source code or internal knowledge of the products.

When our company originally deployed its 3G wireless network, someone just built a half dozen linux servers and installed ISC BIND to act as DNS servers. These DNS servers were effectively used for service discovery within the network, and to facilitate roaming between networks from different carriers. Because these servers were just thrown in as a network build out in a company culture that doesn't really manage servers, they immediately went unmaintained. 

These unmaintained DNS servers were eventually replaced with a new set of DNS servers that were provided by a vendor as an appliance and had a team assigned to maintain the new system with proper vendor support for hardware and software. 

Even though all of this service discovery was built on the DNS protocol, LTE and UMTS used an architecture completely isolated from the internet, with its own set of root servers and a nice set of standards violations which caused a nice set of problems. In our case, the root servers were provided by a roaming exchange, which can be thought of as a company that exists to connect wireless carriers and facilitate how your cell phone can move from one carrier to another.

## The Outage
The roaming exchange we were partnered with normally had 3 DNS servers in operation. These servers are sort of a combination of DNS roots and Zone servers in one. In internet terms, it’s as if the root and .com servers were running on the same machines. But these are not the internet root servers, they're something custom just for the wireless telecoms. 

The roaming exchange had taken one of their servers down for extended maintenance and then experienced an outage on a second server. And our connectivity to international networks went down, even though there should have been one working server. From our perspective, it was as if the internet root servers all disappeared. Our internal network was fine, just the rest of the world blinked out of existence. 

The outage was eventually recovered by the roaming exchange fixing one of the down DNS servers.

## The Investigation
For 2 of 3 root servers, the root cause was pretty straightforward and was up to the partner to solve. The problem I was brought in to investigate, is why we weren’t working with the third server which according to the partner was online and working throughout the outage. No other complaints according to them.

The summary of the initial findings was something along the lines of:
- There was no network cause for the failure. We could reach the server. 
- Testing and packet captures showed bi-directional communications:
  - Normal communications for several seconds after DNS service was restarted on our new DNS resolvers.
  - Manual queries from the host were working, despite the server itself not trying to communicate with the particular root.
- Our old DNS servers were still running and did not experience an outage to the working server.

Note: The new and old servers were both based on ISC Bind, but different versions.

Inspecting the DNS caches on the new servers, showed records related to the working root server were missing, while records towards other roots were present.

At the time I likely had an average understanding of the DNS protocols and services, similar to anyone who has had to configure records to set up a website. We engaged our vendor support and began working with someone whose expertise was specifically in the DNS protocol and specifications, and even escalated to basically the guy who wrote the book on DNS. The outcome of that discussion was that the vendor would write us a custom patch, just for us, the exact details of which I don’t remember.

## The Sniff Test
The difficulty was that this solution didn’t seem to pass the sniff test. There were too many unexplained components to the problem, that writing a patch may or may not resolve. But if nothing else, it was clear that we hadn’t established at this point what the problem was, and suggesting patches or features was premature.

So I set up several VMs that I could use to simulate every host in the production network. I queried the real servers, and then configured my simulation with matching records. I had effectively recreated the production DNS servers of our company, the roaming exchange, and a roaming partner. I was able to reproduce the problem and began experimenting.

I built 10-20 releases of Bind, covering the versions between our old and new servers until I had isolated a series of patches where the problem began. I went through the release notes and nothing stood out but found that the patch that broke our DNS was more than a year old. It seemed unlikely that we were the only ones to have this problem across a year+ of patches. I ended up setting aside this line of investigation before investigating the source code for each patch.

## Clues
Using the simulation I eventually narrowed in on a set of clues:
- In the vendor UI, the root servers were simply called something like root server configuration. However, in Bind we configure a “Root Hints”, an initial set of records to locate other DNS servers. 
- When starting Bind and inspecting the caches, records within the root hints would be present.
- Going packet by packet through the startup sequence, there was a discrepancy in the responses to AAAA queries to the root servers for the names of the root servers. The working servers would return a no data result. The non-working root would return a name error response to the AAAA query.
  - This was an IPv4 only network, so it didn’t seem like focussing on AAAA (IPv6 records) was an important detail at the time. Turns out it was quite important.
- After the startup queries were returned, the records for the root server were missing from the cache.

I reread the DNS standards a couple of times, top to bottom until the following section of [RFC 1034](https://datatracker.ietf.org/doc/html/rfc1034) stood out.

```txt
When the resolver performs the indicated function, it usually has one of
the following results to pass back to the client:

   - One or more RRs giving the requested data.

     In this case the resolver returns the answer in the
     appropriate format.

   - A name error (NE).

     This happens when the referenced name does not exist.  For
     example, a user may have mistyped a host name.

   - A data not found error.

     This happens when the referenced name exists, but data of the
     appropriate type does not.  For example, a host address
     function applied to a mailbox name would return this error
     since the name exists, but no address RR is present.

It is important to note that the functions for translating between host
names and addresses may combine the "name error" and "data not found"
error conditions into a single type of error return, but the general
function should not.  One reason for this is that applications may ask
first for one type of information about a name followed by a second
request to the same name for some other type of information; if the two
errors are combined, then useless queries may slow the application.
```

It’s subtle...

Analyzing DNS as a protocol, it’s easy to think of a request and response for a particular record type as an independent operation. But this particular portion of the standard allows a caching resolver to draw a relationship between different record types based on the error result, specifically whether the name exists for other record types.

So BIND was drawing the inference it was supposed to, the particular name received a name error (NXDomain) response, which meant the name doesn’t exist for any record type. So when performing an IPv6 system query, it should delete all records from its cache that contain the name that doesn’t exist, even those entries that we statically configured in our hints file for the IPv4 address of our root server.

So our IPv4 records for who is root, were getting deleted by system queries that were checking to see if any IPv6 addresses were available for those root servers. 


## Root Cause
With this breakthrough, I was able to document what was happening:

1. During startup/reset, the DNS server would load the root hints and prepopulate the cache to use those roots.
1. During startup, it would also see it only has a bunch of IPv4 roots, so would start querying the roots for the equivalent IPv6 records.
1. For one of the upstream servers, the response for IPv6 records would return NXDomain, which we now know that name doesn’t exist anywhere and would clear the matching IPv4 records from the cache. The missing records from the cache wouldn’t allow the particular server to be used as a root.
1. There is one detail I’m not sure I remember accurately, which is how the servers were responding to the root system query and the interaction with these statically configured records. 
1. Our new servers had never worked with this one root server, it was missed during commissioning that the root server was not being used and had never worked since the new servers were added.
1. The underlying BIND implementation didn’t always implement this behaviour, buried in a patch somewhere is an update to follow the standard more closely, as in it’s plausible a bug was just noticed and fixed.
1. We had the wrong server name on both the old and new servers, only the new servers had the relevant patches. If we had kept the old servers routinely patched, they would have had the same problem.
1. One of the server names we had for the roots was wrong, this is why we were getting an NXDomain response on the AAAA query instead of a no records response.

If the 3 roaming exchange servers were called:
```txt
dns1.grx.gsma.
dns2.grx-partner.gsma.
dns3.grx-partner.gsma.
```

Our configuration was:
```txt
dns1.grx.gsma.
dns2.grx.gsma.
dns3.grx-partner.gsma.
```

As I recall, the way this problem happened was that the roaming partner was rebuilding their DNS servers one by one. And the rebuilt servers were using a new naming scheme, but they had gotten ahead of themselves and the datasheet used to exchange information between companies was out of date or miss-matched, so we still had an old name for a new server with the same IP address. 

This allowed us to draw one more conclusion, we were the only carrier partnered with this roaming exchange using BIND that had updated our installation in the previous year, because we were the only ones who reported going down when the other two root servers were down. 

