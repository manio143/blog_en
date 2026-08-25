---
title: 1. Learning vs Productivity with AI
images: []
---

# Learning vs Productivity

We're in the middle of an AI revolution. No doubt about it.
I'm a software engineer and I got to where I am through a deep exploration of problems - having an issue, trying to solve it and needing to build understanding to do this.
But today problems can be solved by AI agents without me struggling.
The models have seen enough to help them go in the right direction most of time time.
What I get to see is then the explanation of the root cause of the problem and how it can be fixed.
Great for productivity, terrible for building expertise. 

## How we learn
I stumbled on this wonderful article why Khan's Academy failed to improve their students' learning through introduction of a chatbot tutor: [Why Sal Khan’t: On Learning by Making but Teaching by Telling](https://punyamishra.com/2026/04/16/why-sal-khant-on-learning-by-making-but-teaching-by-telling/).
Come back here after reading it.

The point is that we don't learn very well by just consuming content (even if there's an "app like tiktok but with smart topics").
We learn through exploration and ideally when that exploration aims to solve an actual problem.
Thinking through the same issue from multiple perspectives and the act of coming up with those different angles is something that can't be substituted easily by being fed the end solution.

However, there's still a place for content consumption in learning.
At the University I was attending lectures which are a way for information to be passed by the professor onto the students, not too differently from how we take information from written or video content.
But the passive consumption is not enough, which is why most subjects are composed of both lecture and exercise, which means to put the information you heard into practice and force you to think how to solve a problem with it.
 
A side note: [Diataxis](https://diataxis.fr/start-here/) is about how different communication methods in documentation have different purposes, some of which is learning.

## AI as a tool
I recently started a new job. 
Our team is built from scratch with all new hires and over the course of 2 months we had to take over responsibilities of the previous team which is now engaged in some other area. 
We had to become productive fast. 
And for that AI is amazing, because even if I'm not an expert in the domain I can get somewhere and use my general engineering skills to assess the quality of the output.

But we are self aware as a team and we recognize that we're lacking the depth to really know if AI is right or wrong in every aspect of what we're working on.
The productivity is there until it isn't.

Part of the job is responding to tickets from our support engineers who need more information to help the customers.
They gather diagnostics, report on the problem and expect directions on how to move forward.
It's easy to hand this to AI with read access to the source code and ask it to investigate.
It can and will find bugs in the code.
It will suggest mitigation steps for the customer.
But it can also miss the point.

One such ticket described a problem and indicated a bug in our product.
A colleague took that and their AI found a bug, but not quite what the customer was seeing.
They asked me to validate their findings.
I kicked off AI to do a first pass and it kept spinning for a day trying to build better understanding of the code and customer logs to validate the bug or find another one.
It likely wouldn't have found anything if not for the fact that we have a 3 day SLA on providing an answer and I was forced to give it more attention.
I had to split the problem between the customer experience and the suspected bug and try to clarify the timeline of events coming from their logs so that we could better understand if any behavior of our code was unexpected.
Turns out there may be no bug on our side, but their data has some subtle problem when loaded into the database.

Point being, even as AI gets smarter, we need even smarter humans to drive it to achieve the goal.
At least some of the time.

Our manager asked the juniors on the team not to use AI for 80% of their work so they can grind a little and learn more.
Initially I was sceptical but now I agree.
Ultimately, we should try to use AI as a tool and not as a substitute for a human.
Power users achieve more through their tools.
Sometimes it's ok to delegate more to AI, sometimes it's better to drive and use AI as a fast data fetcher or query producer.

**Know your tools, their pros and cons and limitations!**
