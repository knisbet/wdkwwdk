+++
title = "Outage Stories: The copy and paste outage"
+++

When working in Telecommunications, one of the things you learn is to be very careful in production. The companies are trying to chase something like 99.999% and higher availability numbers. So mistakes in production are to be avoided, but like anything else, team members get overconfident and make simple mistakes.

One of these outages, was an accidental paste into a terminal. The result of which took out LTE access for a good chunk of Canada. The good news in this case, at the time the phones probably fell back to the slower and higher latency UMTS network. 

# Context

In these high availability networks, redundancy tends to be a key point. And there is redundancy at multiple layers. So if you have a signalling proxy, you deploy one in Toronto, and one in Montreal. If the Toronto data center has a catastrophic failure, the Montreal data center takes over.

But also within a data center, you don't deploy one router, you deploy 2, and connect servers to both. The servers have 2 power supplies, using DC power supplies fed off a battery room. So a power supply burns out, the server is still running. SFP module burns out, you've got another network connection. Router crashes or is rebooted for an upgrade, and you fail over to another network path.

Some equipment will be deployed as multiple blades or a cluster within a data center. One blade fails, and another takes over. 

So for almost any hardware or physical failure you can think of, there is some other path ready to take over. Whether that's networking, power, fire, flood, meteors, or more. At least, that's the theory… it gets a lot harder when it comes to software and state exchange.

# The Outage

So we've established that the services should be highly redundant, to achieve high availability. So an admin logged into a single host, shouldn't be able to do much damage. Even an evil `rm -rf /` should only take out a single blade, and there would be more blades or even more regions to take over.

So what happens if you're logged in as root, and accidentally paste a big block of text, that just happens to contain something along the lines of:
```shell
...
to change the hostname you run
hostname derp
and the system will update the hostname
...
```

Well the shell will try to interpret those commands. Probably throw tons of errors, but, the `hostname derp` line is a valid command. And when you're root, you're allowed to do things like change the hostname.

Check out the output, you might not even notice the hostname changed because your PS1 is cached. Especially if buried in dozens of lines of errors from the copy and paste.

```shell
root@test:/home/knisbet# to change the hostname you run
hostname derp
and the system will update the hostname
bash: to: command not found
bash: and: command not found
root@test:/home/knisbet#
```

So the hostname got changed, big deal.

Well this particular system was using [Linux-HA](https://en.wikipedia.org/wiki/Linux-HA) to do clustering, and the software operated as a cold standby. What that means is for a particular process, it would only be running on a single host within the cluster. And the cluster control plane if it saw a failure, would start a new instance. This is just like if a pod were to fail in Kubernetes, Kubernetes would start a new pod, but years before Kubernetes.

And the way this system detected a process failure was to search the process list for a matching process. If we run the command `super-ha-proxy...`, and then it's in the process list, the process is working. If it's not in the process list, it must have crashed, exited, etc. and we need to launch a new instance.

And if you pass the hostname as a parameter to your software instance, you're running process would look something like `super-ha-proxy --instances=1 --hostname=server01...`.

But the script that checks the process list, uses the new hostname when searching the process list. The process is still there and operating normally, but our supervisor can't find it anymore, decides the process has crashed, and starts launching another one.

Except, this software runs in cold standby, so it conflicts with itself when multiple instances are running. Not in a way that causes it to clearly crash, but instead gets very confused. The original process still has connections to all peers, and is trying it's best to operate normally. But the new process, is also establishing connections to peers with the same identifier, and messing with internal state about where to route messages. So now messages are ending up in unexpected processes, that don't know what to do. 

If the equipment had just failed, all the peers would've seen that, and started rerouting through Montreal. But the software was still running, in a corrupted state, telling the world that Toronto was still alive and well. But when using those proxies, messages would fail, timing out or getting errors. 

So the team when logged in and investigates and notices pretty quickly the hostname is wrong. But fixing the hostname doesn't solve the problem. The supervisor script just starts detecting the old process, and has no idea that the additional instance has been launched. So it's just as happy to keep moving forward in it's corrupted state.

But some reboots later and Toronto is back online.

And a million cell phones lose their LTE internet access...
