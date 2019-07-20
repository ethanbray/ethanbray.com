---
layout: post
title:  "Working Remotely: 2 Months In"
date:   2019-07-20 07:20:00 +0000
categories: []
---

I've been working at [MessageCloud](https://www.messagecloud.com) for around 
3 and a half years now. After taking a break to travel at the beginning of this
year, I returned to work in a 75% remote position. I spend the majority of my 
time working from home and spend a week in the office each month.

We've had several other remote employees in the past, but they've always started
as on-site. I've read plenty of articles about the pros and cons of
working remotely, tackling issues such as loneliness and lack of communication,
so I felt prepared.

This post will cover some of the teething issues we've encountered as well as
the solutions we've put in place to help combat them. It's likely that this post
will be the first in a series detailing my experience working remotely.

### Using video standups instead of Slack messaging

Previously I would speak with each of the team members individually and discuss
our plans for the day. It was rare that I needed to ask for more information as 
being on-site made it a lot easier to passively soak up information regarding 
their progress and workload.
 
Once I began remote working, I tried to continue the daily chats using Slack. 
However, Slack isn't particularly conducive to long conversations. Especially 
several at once in the same room. People's replies would often be delayed as
they were sidetracked by other people or simply did not see that I had replied 
to them amidst other conversations.

Our solution for this was to have video standups every morning. This way each 
member of the team has their chance to speak and have everyone's attention. 
These standups help both myself and the team stay up to date with everyone's
work and any blockers they might be facing.

### Having all members of a meeting using a webcam

This issue became apparent when we introduced the standups as well as the regular 
catchup meetings I have with other colleagues. Without a webcam it can be difficult
to know when one person is done speaking or who is speaking to who in the case of
meetings with more than 2 people.

It also just _feels_ better. Being able to see the person you're conversing with
is an underrated requirement of working remotely, in my opinion.

At first I imagine some members of the team felt uncomfortable with being required
to use a webcam, but since it has become a regular affair in our standups everyone
has consistently used a webcam in other meetings.

### Detailed issue tracking

At MessageCloud, we use GitHub issues to record our work. However, it's fair to say
that at times we were less than verbose with the details we left in our issues. We
unknowingly relied on the passive information we received throughout the day to inform
ourselves on the issues we were tackling.

This occurred when team members would start conversing with me about an issue
and I would struggle to grasp what they were talking about immediately. I would often
have to ask my colleague to start from the beginning and explain the issue to me in
greater depth. This can become slightly frustrating as you spend more time asking for
clarification than actually solving the issue.

I spoke with the team and we worked together to record our issues with greater detail,
as well as keeping them up to date with any progress, blockers and extra information that
came to light. We're not yet doing this constantly but we're working on making it a 
consistent habit among the whole team.

### Missing Break/Lunch

Although I work remotely, I'm required to work the same hours as if I was in the 
office as my role mainly consists of team leadership. A surprising symptom of working 
remotely was that without the break time hubbub of office to remind me, I frequently 
missed my scheduled breaks and lunch time. When focusing in on a particularly tricky 
piece of work, I tend to turn on do not disturb and full-screen my editor, hiding the 
time and exacerbating the issue. 

At MessageCloud, we have a Go based Slack bot called [Ava](https://non-aliencreatures.fandom.com/wiki/Ava_(Ex_Machina)).
Ava is used to kick off Jenkins deployments as well as having several fun little
features. 

I added a small piece of code to Ava that sends me a message on Slack when it's break
time.

![An example of Ava reminding me it's breaktime](/../static/img/2019-07-20-ava-example.png)

As a side note, I'll cover how we built Ava in a future blog post.

### Using my on-site time wisely

As much as everyone likes to sing the praises of remote working, there are definite
issues. Especially so in a company that does not have many remote workers and a 
'remote-first' attitude. Certain tasks such as event storming and helping junior 
developers are infinitely easier in person.

In the week that I spend on-site, I focus on performing the tasks that may otherwise 
be difficult remotely. Meeting with domain experts as well as having in-person 
catchups with other members of the organisation.

It's also good to spend this time bonding with the team and ensuring that you're not 
just a face on a computer screen to them!
