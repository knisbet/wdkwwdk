+++
title = "ShellGuard: Week 1"
+++

This week, I decided to start a new project called ShellGuard. For the last year or so, I've had an itch. It all started with a customer at [Teleport](https://goteleport.com/) who had concerns about system administrators ability to access customer data on the systems they administer.

My knee-jerk thought is this is an impossible problem. The security controls don't create a clear way to allow a user to change the system, but not touch data. Even if you can write some clever selinux policy or BPF rules, it's generally trivial to bypass that protection.

Even if we narrow our view to just audit, common syscall / BPF audit mechanisms are easy to bypass. They're more for theatre than actually catching a sophisticated breach of security.

But it bugged me... badly... that we can't restrict access. 

And that's where the idea of ShellGuard came in. We already have technologies for running untrusted workloads on a Linux host. Things like (gVisor)[https://gvisor.dev/] and (Firecracker)[https://firecracker-microvm.github.io/] provide lightweight ways to virtualize or isolate workloads. What strikes me about a solution like gVisor, is basically the ability to redirect syscall handling to another application. Why can't we do the same thing with a shell session. If we wrap the user session within an application Kernel like gvisor, and insert ourselves between the user and kernel, we can implement a new policy engine.

This idea fascinated me for a few reasons. 

A user can really be setup with a read only view of the host. Policy could be designed in a way where a user really has multiple use cases. I need internet access to GitHub and database access. A user running untrusted software could be exploited to upload the database to GitHub. But what if you could write a policy that says, you can get internet access and database access, but not at the same time. There isn't a path left open customer data to reach the internet, even if the user needs both as part of their job.

So week 1 has mostly been about getting started. I've started to put together a tech demo, so that I can begin to show the concept and how it would work. I really thought isolating user access impossible... but the more I think about it, the more I believe in a world where sensitive access can be segmented away. 
