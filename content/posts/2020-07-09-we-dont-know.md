+++
title = "We don't know what we don't know"
+++

Recently, I've begun to spend more effort considering and emphasizing on what I don't know. Like many engineers, I rely heavily on mental models when tackling technical problems and often miss approaches missing from my mental model. I also have the aspect of my personality where I don't like to be wrong.

Let's spend some time considering what it means not to know what we don't know and how we can remind ourselves of some of our own biases.


## What is WDKWWDK?
It's an acronym to remind ourselves that we may be missing some piece of the puzzle. I'm not the first to use this acronym, but I also don't believe it's widely spread.

I've slowly begun to use this acronym as a reminder in chat or conversation that we don't know or remember every detail. It's a reminder that we should step back and ask ourselves questions about what may be missing from our approach, conversation, engineering design, etc.

It doesn't mean we have to answer every question, only that we're cognizant that we'll always be missing some details, and that there are risks with doing so.


## My first experience with WDKWWDK
For me, this realization first came during a meeting.

We were trying to tackle a specific security challenge that required selecting a cryptographic protocol with particular properties. I was the security lead on the team and was able to recommend Noise based on some industry experts' suggestions.

The problem we had is we weren't able to locate an implementation of the protocol for our embedded CPU. While I was evolving into a security lead for the team with a better than average understanding of security principles, I was utterly ill-prepared to write a crypto implementation that would pass any scrutiny.

However, with the most knowledge, the team asked me to develop the implementation.

I objected, indicating that writing a cryptographic implementation is detailed work, and it's challenging to be sure that the implementation is correct.

The team countered, saying that it must be correct if the implementation can communicate with another library.

The problem is it's possible to write a compatible protocol implementation, that has significant security problems. Re-using nonces or timing attacks could ruin our day, and we'd easily miss out on the security properties we desired.

The team countered again, indicating that if I knew about problems like nonce re-use or timing attacks, that I should avoid those problems. 

Any problem I can describe is a problem I can avoid.

Eventually, I had to figure out to approach this engineering conversation as an admission, I don't know what I don't know, so I'm ill-prepared to solve problems I don't even know to avoid. 

Not only did I need to admit what I didn't know, but as a team, we had to admit combined that we didn't know what we didn't know. In many scenarios, we may be able to get away with this, but here, a failure meant we failed to deliver on our security promises.



## Is this the Dunning-Kruger effect?
For those who aren't familiar, a simplified version of the Dunning-Kruger effect states that a cognitive bias exists where individuals with limited knowledge or competence in a given intellectual domain greatly overestimate their expertise or skill in that domain. And that those with a great deal of expertise tend to underestimate their results. 

While the Dunning-Kruger effect may apply here, we often forget that this cognitive bias only means that people scoring in the 12th percentile had ranked themselves in the 62nd percentile. But those in the 90th percentile had self-assessed to be within the ~75th percentile. 

![Image from Dunning-Kruger Research](https://www.researchgate.net/profile/David_Dunning2/publication/12688660/figure/fig1/AS:394431292297232@1471051156893/Perceived-ability-to-recognize-humor-as-a-function-of-actual-test-performance-Study-1.png)
[Dunning-Kruger effect on research gate for more information](https://www.researchgate.net/figure/Perceived-ability-to-recognize-humor-as-a-function-of-actual-test-performance-Study-1_fig1_12688660)
<br /><br /><br />
So while this cognitive bias is fascinating, in software engineering disciplines, it's unreasonable to believe in senior teams we're dealing with anyone in the 1st or 2nd quartile who are vastly overestimating their abilities. 

The important lesson here is we need to be aware of and careful of our biases, whether it be the Dunning-Kruger effect or something else.


## Fear
As engineers, I think we get used to analyzing complex problems and coming back with answers. Especially those of us who are known for digging in and finding the solutions.

When given a task, there may also be a particular fear associated with not having the answer. If your leadership or team members expect you are an expert in a domain and are assigned a problem, it feels like a failure to return without the full solution. Or without carefully considering every possibility.

It's also happened to me, where others have tried to shame me for not knowing some detail off hand. As an expert in X, you should immediately know the answer to a question. 

A large part of this is realizing it's not a failure to admit that we have missing knowledge. We may not remember some details or have more research to do. Maybe there is no action required. The failure is to proceed without identifying the risk of the missing information. Whether an expert or not, there are always going to be missing details. When working in security, the missed detail could be the difference between high and abysmal security. In other domains, the risks may be much different.



## How do we apply this?
I think it's as simple as stating, "We don't know what we don't know" in a conversation, or typing WDKWWDK into a document or chat session. And ensuring our teams and leaders don't allow a stigma to exist around not knowing some details.

This statement of fact, acts as a reminder, that we should step back and consider how our biases may impact a particular decision and find what information may be missing.

We can then consider:
- What we know
- What we know we don't know
- What we don't know that we don't know

And decide on the risks of proceeding with missing information, cognizant that there will always be missing information.


## Summary
- When decision making or proposing engineering solutions, we need to remind ourselves that there are details that we don't know
- We need to do our best to avoid stigmas about gaps in our knowledge. Being an expert in security does not dictate knowledge of every algorithm, every protocol, or every approach.
- WDKWWDK can often be about risk management. Missing details in security me be the difference between a solution being secure, or offering no security properties. In other contexts, it may just be a bug.











