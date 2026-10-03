---
title: "Why didn't I use YANG? A question I wasn't ready for"
date: 2026-10-03
description: "After my talk on 'JSON freedom or chaos', someone asked why I didn't use YANG. A look at the blind spot behind that question, and why Pydantic is still the right fit."
image: "/blogs/20261003_why-not-yang/images/pydantic_vs_yang.png"
tags:
  - pydantic
  - yang
  - validation
  - json
  - python
---

After my presentation on [JSON freedom or chaos: how to trust your data](https://github.com/bartdorlandt/json_freedom_or_chaos/blob/main/presentation/json_freedom_or_chaos_cphautomate.pdf), someone in the audience asked: "Why didn't you use YANG instead?"

It caught me off guard. I had never considered it. Isn't YANG about modeling network device configurations?

## A decision shaped by context

A 30-minute talk can't hold everything I know about a project. I couldn't have shared every detail that shaped my decision, and without that context, nobody else in the room could have made the same choice. Perhaps the framing of the situation wasn't clear enough for the audience to see that this not only involved network device specific data.

I found it to be an interesting question, one that I now dove into more.

## Could YANG be the alternative to pydantic?

To me, YANG has always been tied to network devices: NETCONF, RESTCONF, gNMI, models describing what a device accepts. Using it in a CI/CD pipeline to validate plain data, with no device involved, never crossed my mind.

So I looked into it. YANG is a data modeling language, and it can validate JSON offline. Tools like `yanglint` and `yangson` can run in a GitLab pipeline like any other check.

## What I found

**It can process JSON, but it is strict about it.** YANG may expect the JSON to be very strict (double quotes, instead of single quotes and perhaps more), see RFC 7951. Since the JSONs in my case have some history, it is fairly well possible it doesn't match on all the constraints given by the RFC. I haven't tested this myself.

**Pydantic gives clearer error messages.** When validation fails, I want to see what's wrong and where, right away. From what I've found, Pydantic does this better. YANG errors, especially from `must` statements written in XPath, are harder to read and debug.

**I know Pydantic far better than YANG.** This matters more than people admit. A tool I understand deeply lets me move faster, catch my own mistakes, and explain the result to others.

**It fits the team and the company.** Python, Pydantic, and pytest are already common tools here. YANG isn't used at all. Introducing it would mean a new language and new tooling for the team, and for the company, on top of the work itself. That's a real cost, and for this problem I couldn't see it paying off.

**JSON Schema comes along for free.** Pydantic can export a JSON Schema from the same models. I hadn't thought much about that before, but it's nice: the models in my code can double as a schema that other tools can consume.

## Would I ever use YANG?

Yes. YANG has real strengths: built-in constraints such as `must`, `when`, and `leafref`, a language-neutral schema, and a large library of existing models. If my data starts to cross language boundaries, or needs to reach network devices or controllers, it becomes a serious option, and I could add it as another job in the pipeline.

But today, none of that is needed. Python, Pydantic, and pytest are the right fit for this problem.

## What I took from it

The most useful part of giving a talk is often the question you didn't see coming. Sure, it can put you on the spot, but isn't that what knowledge sharing and growth is also about?

It made me think about it more and go beyond my initial assumptions. If you've validated data with YANG outside of device workflows, I'd love to hear how it went.
