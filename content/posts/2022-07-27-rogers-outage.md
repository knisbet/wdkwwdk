+++
title = "Taking a look at the Rogers Outage CRTC Letter"
+++

Last week, the CRTC made public a redacted version of the response Rogers filed with answers to the regulators questions. I started my career in wireless telecommunications for a competitor to Rogers here in Canada, and wanted to dig through the outage. I've been out of the industry for a few years now, but my outdated perspective may be of some use here.

Unfortunately, the redacted version removes almost all the useful information for other carriers or other industries to learn from this outage. But I still dug through the report to try and find what useful information I could.

# Resources

## Regulator Filings
- CRTC Letter to Rogers: [https://crtc.gc.ca/eng/archive/2022/lt220712.htm](https://crtc.gc.ca/eng/archive/2022/lt220712.htm)
- Rogers response to CRTC (docx): [https://crtc.gc.ca/public/otf/2022/c12_202203868/4215445.docx](https://crtc.gc.ca/public/otf/2022/c12_202203868/4215445.docx)
- Google docs version of the response: [https://docs.google.com/document/d/1e8fmZGzy_VaYuLtXgDz62LTh4DfvkPkj/edit?usp=sharing&ouid=116779044200847408744&rtpof=true&sd=true](https://docs.google.com/document/d/1e8fmZGzy_VaYuLtXgDz62LTh4DfvkPkj/edit?usp=sharing&ouid=116779044200847408744&rtpof=true&sd=true)


## Media Reports

If you're instead interested in a media level analysis, here are some of the media reports i've found.
- [How a coding error caused Rogers outage that left millions without service - Globe and Mail](https://www.theglobeandmail.com/business/article-how-a-coding-error-caused-rogers-outage-that-left-millions-without/)
- [Rogers shares explanation after 'unprecedented' outage - National Post](https://nationalpost.com/news/rogers-shares-explanation-after-unprecedented-outage/wcm/31db4432-713a-4d70-98ef-56f41957c536/amp/)
- [Rogers unable to switch customers to Bell, Telus, despite competing carrier offers - Toronto Star](https://www.thestar.com/business/2022/07/23/rogers-unable-to-switch-customers-to-bell-telus-despite-competing-carrier-offers.html)


# CRTC Letter
The letter itself basically outlines that the Canadian Radio-television and Telecommunications Commission (CRTC) wants to inquire into the outage that began on July 8th, 2022. And adds the basis for this outreach, including disruptions to business, emergency services, and more.

You can read the full letter above, but effectively it's an outline of questions that Rogers has been asked to respond to. I'll go through what I see as the important questions and perspective as we go through the response.

# Rogers Response
At a high level, Rogers starts by confirming that they had identified the cause of the outage. 

```txt
We have identified the cause of the outage to a network system failure following an update in our core IP network during the early morning of Friday July 8th. This caused our IP routing network to malfunction. To mitigate this, we re-established management connectivity with the routing network, disconnected the routers that were the source of the outage, resolved the errors caused by the update and redirected traffic, which allowed our network and services to progressively come back online later that day. While the network issue that caused the full-service outage had largely been resolved by the end of Friday, some minor instability issues persisted over the weekend.
```

What I find interesting about this is the reference to minor instability issues that persisted over the weekend. 

When I worked on a similar outage, where the underlying core network went down for 20 or 30 minutes, my team spent the next 10 or so hours finding and fixing the glitches in the cellular network. This was mainly in broken states within different equipment. Nothing exposes those bugs as good as taking out the underlying network at all layers at the same time.

For example think of an HTTP proxy. A client comes to the proxy, makes a request, the proxy holds some request state, and sends the request to a server. You probably know exactly what happens and it's well tested if the upstream server goes down. Just return an error to the client. If the client loses connection, well understood as well, complete the upstream request, and discard it. But what happens if both the upstream and client networks fail at the same time. You start exercising a code path that combines failures that usually aren't as well tested. Combine this with some missing internal state, and you start having failures.

Especially in cellular networks, there are all sorts of state that's distributed around the network and needs to be kept in sync for different call procedures. This can result in things like being able to send but not receive an SMS message. 

```txt
Our engineers and technical experts have been and are continuing to work alongside our global equipment vendors to fully explore the root cause and its effects. 
```

This is another interesting point that might be missed on many that don't know the industry. Telecommunication companies primarily operate as integrators of vendors equipment. While this isn't always the case, most equipment to build and create the network will be purchased from vendors. So if you want a cellular radio for LTE, you may buy the particular piece of equipment from Nokia, Ericcson, Huawei, or others. 

In this model of buying equipment from vendors, it often creates an incentive to blame the vendor for outages. But even harder at times, can be to convince another company to work on a feature or design change to make configuration errors less likely. Or like any complex situation, there is probably plenty of blame to share between both the operator and the vendor.


```txt
Additionally, Rogers will work with governmental agencies and our industry peers to further strengthen the resiliency of our network and improve communication and co-operation during events like this. Most importantly, we will explore additional measures to maintain or transfer to other networks 9-1-1 and other essential services during events like these.
```

I'll dig into this deeper later on. But the failure of 911 services, especially for cell phones seems like an incredibly bad design oversight. Cell phones when not attached to a network, can already do an emergency call on any network the phone can find. So by continuing to advertise the Rogers network throughout the outage, caused phone to stick to the broken network. Or probably caused an unattached phone to scan for an available network, and not find a working 911 network.


# Questions and Answers

## About the outage

### Provide a complete and detailed report on the service outage that began on 8 July 2022
Unfortunately it looks like Rogers preferred to keep the full details confidential, and the full timeline was attached as confidential. But included are some high level details.

```txt
The network outage experienced by Rogers on July 8th was the result of a network update that was implemented in the early morning. The business requirements and design for this network change started many months ago. Rogers went through a comprehensive planning process including scoping, budget approval, project approval, kickoff, design document, method of procedure, risk assessment, and testing, finally culminating in the engineering and implementation phases. Updates to Rogers’ core network are made very carefully.
```

For those outside of telecom, method of procedure may be unfamiliar. While different companies may operate differently, the method of procedure is basically a document that outlines how a change in the network should be executed. It can be as detailed as every command someone in operations should run to make the change, step by step. It may also contain pre-checks that should be executed.

The philosophy is a sort of separation of inputs, where one engineer will write the procedure for the change to be executed, and then someone tasked with operations will be responsible for execution. I don't know if this is how Rogers is using MOPs however.

```txt
Maintenance and update windows always take place in the very early morning hours when network traffic is at its quietest.
```

I can't comment as to Rogers, however I know at other big telecoms in Canada that this isn't true for every change. There are plenty of changes that are deemed non-risky, and will be executed when convenient. I've also seen different managers and agenda come into play that try to push against change windows to be more efficient, and often a back and forth on the balance of stability vs getting things done. 


```txt
The configuration change deleted a routing filter and allowed for all possible routes to the Internet to pass through the routers. As a result, the routers immediately began propagating abnormally high volumes of routes throughout the core network. Certain network routing equipment became flooded, exceeded their capacity levels and were then unable to route traffic, causing the common core network to stop processing traffic. 
```

This twitter thread likely covers this side of the impact better than I can: https://twitter.com/atoonk/status/1550896347691134977



```txt
The Rogers outage on July 8, 2022, was unprecedented. As discussed in the previous response, it resulted during a routing configuration change to three Distribution Routers in our common core network. Unfortunately, the configuration change deleted a routing filter and allowed for all possible routes to the Internet to be distributed; the routers then propagated abnormally high volumes of routes throughout the core network. Certain network routing equipment became flooded, exceeded their memory and processing capacity and were then unable to route and process traffic, causing the common core network to shut down. As a result, the Rogers network lost connectivity internally and to the Internet for all incoming and outgoing traffic for both the wireless and wireline networks for our consumer and business customers.
```

I'm going to nitpick calling this an unprecedented outage. Based on what I can get out of the report and not being able to see the full internal analysis, this seems fairly predictable. The outage explanation is basically some automation removed a piece of configuration that was essential for the routers to function. Without the configuration, routers within the network went into CPU overload processing routing updates. 

Someone knew about this and placed that configuration on the router. 

This is a known failure mode for most routing equipment. Most of these big routers seem like big powerful machines, but they tend to be more of a fast path for pushing many millions of packets. And a separate compute for sending and receiving signalling about where those packets should go. The control plane side that does the signalling there are often many ways to overload, which will cause peers to think the router has died. 

### how did Rogers prioritize reinstating services and what repairs were required
```txt
The prioritization of service restoration was always dependent on which service was most relied upon by Canadians for emergency services. As wireless devices have become the dominant form of communicating for a vast majority of Canadians, the wireless network was the first focus of our recovery efforts. Subsequently, we focused on landline service, which remains another important method to access emergency care. We then the worked to restore data services, particularly for critical care services and infrastructure.
```

I suspect reality is a bit more nuanced that multiple teams were probably looking at the equipment they were responsible for in parallel. If there was a glitch for some subscribers in the LTE network for example, people who focus on the LTE network were likely addressing that. But from a total outage perspective, I'm sure teams were focussing more or less in this order.

Having working big outages like this, once the underlying IP network is restored and the upper layer network comes back up, there will be a plethora of impacts, alarms, metrics that need to be sorted through. 

When I worked a similar outage for a competitor, the team would find something like call failure rates are 5x normal, but also trending back towards normal. And then it's a tough thing to figure out, will it return to normal on its own? Maybe customers are rebooting their phones, so it looks like it's returning to normal but only because customers are taking their own actions. What options do we have to clean up the call states that are leading to those failures? Maybe the only tool available doesn't know who is and isn't working, so the only option is to reset every device. Is resetting every device worth it to move from a 2% failure rate to a normal 0.4% failure rate? 




### what measures or steps were put in place in the aftermath of the earlier-mentioned April 2021 outage, and why they failed in preventing this new outage

Everything substantive in the response to this question was redacted. It's unfortunate, because as an industry, including non-telecom like SRE can learn substantially from how Telecoms operate networks as reliable as they are. 

Basically Rogers did lots of vague things to prevent similar failures.

### how did the outage impact Rogers’ own staff and their ability to determine the cause of the outage and restore services;

```txt
At the early stage of the outage, many Rogers’ network employees were impacted and could not connect to our IT and network systems.  This impeded initial triage and restoration efforts as teams needed to travel to centralized locations where management network access was established. To complicate matters further, the loss of access to our VPN system to our core network nodes affected our timely ability to begin identifying the trouble and, hence, delayed the restoral efforts.  
```

This is the inherent difficulty in managing network systems. While in my experience the network architectures tend to be layered, when you're also the ISP I can imagine the difficulty in this separation. And problems are rare enough that it gets difficult to predict exactly what outages will knock out employee access.

Alot of the carriers also lease infrastructure from eachother, so one big telco can cause impacts in another.

I almost wonder if there is room for something like Starlink here, as the satellite infrastructure would almost be guaranteed to provide diverse network access to a location, without running additional wires in the ground. 

Having experienced a similar failure, when I noticed my home internet and cell phone were all offline, I started driving to the office to get a stable connection to production. That was a long day.


### extent to which Rogers sought or received assistance from other TSPs in addressing the outage or situation arising from the service interruption;

```txt
In order to allow our customers to use Bell or TELUS’ networks, we would have needed access to our own Home Location Register (“HLR”), Home Subscriber Server (“HSS”) and Centralized User Database (“CUDB”). This was not possible during the incident. 
```

Basically what Rogers is describing here are the databases used to track and authenticate mobile devices were also unavailable. In cellular mobility, the network can be thought of as two networks for two different purposes. There is a visited network, more or less the radio towers you connect to. And the home network, which is what actually gives you internet, cellular, SMS, etc services. When you're on your own carrier, you're using your own carriers visited and home networks. 

But when you go somewhere else, say the United States, you change your visited network and keep your home network. What Rogers is basically saying, is there home network was broken, so the other carriers weren't able to help, because they wouldn't be able to authenticate devices, or connect them back to this home network.

There is some tech that allows for internet connections to only use the visited network, but when I was in the industry this was not commonly deployed.

```txt
Furthermore, given the national nature of this event, no competitor’s network would have been able to handle the extra and sudden volume of wireless customers (over 10.2M) and the related voice/data traffic surge. If not done carefully, such an attempt could have impeded the operations of the other carriers’ networks. 
```

This is a key point. I don't know that any of the other major carriers would be able to predict how the sudden influx would affect their networks. While telecom is trying to move in a model alot more like SaaS providers that can autoscale equipment, I don't know how successful this would be.

Let's pick something simple. The number of IP addresses available to be assigned to a phone. The other operators likely only provision enough to cover their own needs. Taking over another carrier in Canada isn't an expected failover pattern, so there isn't provisioning to handle this need. And IP addresses are likely only 1 of 10,000 different capacity constraints that go into a mobile network.

My 2 cents is the most likely path to even considering allowing something like this would be more akin to multi-sim support. Where you would have credentials on your Sim card for both Rogers and a competitor network and the device could switch, if warranted. And then the possibility of this failover goes into the capacity planning for the networks and is only for customers that need it.

## Impact on Emergency Services
The impact to emergency services is really why I wanted to dig into this report, and try to understand specifically why Rogers cell phones were unable to make emergency calls. Having done some small work on 911 services for cell phones, normally any device is able to attach to any network, unauthenticated, to make emergency calls. Which means while the Rogers core network was down, something was causing the Rogers radio network to continue to advertise that it was able to take emergency calls. Otherwise the cell phones should have scanned for any available wireless network, and done an emergency attach to get emergency services.

You don't need a SIM card in your phone, a valid account, etc to make a 911 call. 



### Provide a complete and detailed report on the impact on emergency services of the outage that began on 8 July 2022, including but not limited to:

```txt
With respect to wireless public alerting service (WPAS”), the Rogers Broadcast Message Center (“BMC”) platform was operable to receive alerts from Pelmorex, the WPAS administrator.  However, broadcast-immediate (“BI”) public alerts could not be delivered to any wireless devices across Rogers’ coverage areas due to the outage. Based on a review of the alerts received into the WPAS BMC platform, the only impact occurred in the Province of Saskatchewan. There were four alerts, and associated updates, received but not delivered to wireless devices in Rogers’ coverage area. There were no other alerts issued, as seen on our WPAS BMC platform.
```

I interpret this and other areas of the report to basically indicate that the IP core outage caused the cell sites to be unreachable. So to send an alert out to all devices in an area, you need to reach the cell site to send the message to phones.

```txt
With respect to broadcasting (cable TV/Radio) alerts, our alert hardware is connected to our IP network. Since we had no connection to the Internet on July 8th, we were unable to send out any alerts on that day in the regions that we were serving.
```

And also alerts on services that weren't cell sites all use the IP network.


### whether the outage specifically impacted the 9-1-1 networks or only the originating networks, and if the former, how was this possible in light of resiliency and redundancy obligations imposed by the Commission;

```txt
The outage solely impacted Rogers’ originating network. The 9-1-1 networks that receive calls from originating networks are not operated by Rogers. Rather, they are operated by the three large Canadian Incumbent Local Exchange Carriers (“ILECs”). They were unaffected by the outage.
```

This is basically saying that Rogers is not operating the 911 networks themselves, as those are operated by Telus, Bell, and Sasktell across Canada. So the impact was isolated to Rogers network being able to deliver emergency calls to the 911 networks.

### number of public alerts sent that did not reach Rogers’ customers, broken down by province;
```txt
Only four (4) alerts were received on Rogers WPAS BMC platform on July 8th. All alerts were in the Province of Saskatchewan. No other alert was issued in Canada on that day:

1. WPAS ID 957:  7:40AM CST: Saskatchewan RCMP – Civil Emergency (Dangerous Person)
2. WPAS ID 960:  4:05PM CST: Environment Canada – Tornado (Warning)
3. WPAS ID 964:  4:19PM CST: Environment Canada – Tornado (Warning)
4. WPAS ID 982:  5:31PM CST: Environment Canada – Tornado (Warning)
```

I'm just including this as it adds perspective on the number of alerts Rogers wasn't able to deliver across the country that day.

### how were 9-1-1 calls processed during the outage and whether they were able to be processed by other wireless networks within the same coverage area;

```txt
As seen in Rogers(CRTC)11July2022-2.i above, Rogers was able to route thousands of 9-1-1 calls on July 8th.  Rogers’ wireless network worked intermittently during that day as we were trying to restore our IP core network, varying region by region.

...

The connection state of the UE to Rogers wireless network, and the stability of our network, determined the ability of Rogers wireless customers to have their 9-1-1 calls processed by other wireless networks within the same coverage area.

Bell and TELUS confirmed to us that some of our customers were able to connect to their wireless networks in order to place 9-1-1 calls.
```

The UE is the User Equipment, basically the cell phone.

I think what Rogers is trying to say, is if the cell phone was in a state where it didn't see the Rogers network, like "No Service" on the display, an emergency call would do a network scan and attach to another network with service. 

### whether other measures could have been taken to re-establish 9-1-1 services sooner;

```txt
No other measures would have helped restore 9-1-1 service on July 8th. One possible option that was explored by Rogers was to shut down our RAN. Normally, if a customer’s device cannot connect to their own carrier’s RAN, they will automatically connect to the strongest signal available, even from another carrier, for the purpose of making a 9-1-1 call. However, since Rogers’ RAN remained in service on July 8th, many Rogers customers phones did not attempt to connect to another network.
```

If this is the case, what the report doesn't talk about at all is that non-Rogers customers could be impacted by this outage as well. If a competitor had very weak coverage, it might scan and find the Rogers network and try to use it for the emergency call, instead of finding a network to use. Or a device that is roaming and in airplane mode and then needs to make an emergency call, will try to attach to the first network it finds.

So if this was the case, the Rogers network behaving like this could have prevented emergency calls from reaching a working network. This certainly would be the minority of devices, but I find this to be infuriating that the network sat there advertising 911 services, for hours, that it could not be delivered.

### what alternatives are available to Rogers’ customers to access 9-1-1 services during such outages;

```txt
The GSM standard for the routing of 9-1-1 calls implies that a wireless customer always has the option to remove the SIM card from their device and then to place the 9-1-1 call. The handset will register to another wireless network (the one with the strongest signal, even if there are not roaming arrangements).
```

I think the problem is most people wouldn't know even what their sim card is, let alone that it can be removed to make a 911 call. But as with above, the problem here appears to be Rogers continued to advertise a 911 network, so the device might still connect.

I'm also not sure that the strongest signal is correct. With so many frequencies now in use for cellular networks, it can take quite some time to do a full network scan. So I'm not sure devices in this state will scan all possible networks and then choose the strongest signal. I think but am not sure that the devices would connect to the first network found.

```txt
Further, some newer smart devices have the capability to reconnect automatically to other wireless network for 9-1-1 calls when the home network is down. 
```

If this is the case that's great. It would probably be a great idea to put this into the standard.

# Summary

I really wanted to dig into this report, as I've worked these sorts of outages. And they are really difficult, as there are infinite ways to make the problems worse, lots of contradictory information, theories of the root cause, and missing information. It is really difficult to operate these networks.

But what I found most frustrating, was the 911 services. The core network is down, but the radio network is continuing to indicate 911 services. While I only suspect this is the case, there isn't a good reason that other networks couldn't be used, other than a desire to do so. I really hope this doesn't get lost on all the major carriers after this outage.


