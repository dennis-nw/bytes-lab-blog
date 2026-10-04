---
title: 'I removed ~400ms from every API request by looking at my traces'
published: 2026-10-04T12:48:44.603Z
description: 'A hidden Supabase Auth network call was adding hundreds of milliseconds to every request. Here is how tracing helped me find and fix it.'
updated: ''
tags:
  - FastAPI
  - Python
  - Supabase
  - Observability
  - Pydantic Logfire
draft: false
pin: 0
toc: true
lang: ''
abbrlink: ''
---

A few days ago, while looking at API metrics for [Still200](https://still200.com), I noticed something that bothered me. I use the excellent [Pydantic Logfire](https://logfire-us.pydantic.dev/) for my observability needs.

Most API requests were taking around **600–700ms**.

Nothing was broken. There were no errors. Users could log in, monitors loaded, incidents loaded. Everything worked exactly as expected.

But 600ms for some pretty ordinary API endpoints felt wrong.

So I went looking.

## Finding the missing 400ms

I added more instrumentation around HTTP calls and database queries and went back to the traces.

The culprit became obvious pretty quickly. Every authenticated API request was making an external HTTP request to Supabase Auth.

My FastAPI authentication dependency was doing this:

```python
user = await supabase_client.auth.get_user(jwt=jwt)
```

`get_user()` validates the JWT by making a request to the Supabase Auth server.

That makes perfect sense if you need the latest user record from Supabase.
But I didn't.

For most requests, all my API needed to know was:
Is this token valid, and which user does it belong to?

I was paying for a network round trip on every authenticated request to answer a question that could be answered locally. And because authentication sits in front of almost every useful API endpoint, that latency tax was being paid everywhere.

## The fix

Supabase supports asymmetric JWT signing keys.
With asymmetric cryptography, Supabase signs a JWT using a private key, while your application can verify that signature using the corresponding public key.
Your API doesn't need the private key, and it doesn't need to ask the Auth server whether the signature is valid.
It can verify the token itself.
Supabase exposes the public signing keys through its JWKS endpoint and its client libraries can cache them, meaning verification can normally happen locally rather than putting the Auth server in the hot path of every request.

In Python, my change was almost comically small.

Before:

```python
user = await supabase_client.auth.get_user(jwt=jwt)
```

After:
```python
claims_response = await supabase_client.auth.get_claims(jwt=jwt)
```

I could then extract the user ID and other attributes I needed directly from the verified claims.
Supabase actually recommends `get_claims()` for this use case. You can read more about [JWT signing keys](https://supabase.com/blog/jwt-signing-keys) and, in particular, their [section](https://supabase.com/blog/jwt-signing-keys#getting-the-most-benefit) on getting the most benefit.

## The result

Here's a comparison of the average latency numbers before and after the change. 

| HTTP Route | Before (Remote Auth) | After (Local Auth) | Improvement |
| :--- | :---: | :---: | :---: |
| `GET /monitors/{monitor_id}/stats` | 728.5 ms | 294.9 ms | **-433.6 ms** (59% faster) |
| `GET /monitors` | 698.5 ms | 135.4 ms | **-563.1 ms** (81% faster) |
| `GET /incidents` | 654.8 ms | 140.5 ms | **-514.3 ms** (79% faster) |
| `GET /subscriptions/plan` | 629.9 ms | 196.6 ms | **-433.3 ms** (69% faster) |
| `GET /monitors/{monitor_id}` | 545.8 ms | 125.6 ms | **-420.2 ms** (77% faster) |


One authentication change removed roughly 400–500ms from many requests.  
`GET /monitors` went from nearly 700ms to 135ms.  
Nothing about the endpoint itself changed.
No database indexes. No Redis cache. No query optimization. No faster server.
I just stopped making an unnecessary network request.

## The more interesting lesson

The performance improvement is nice, but what stuck with me was how easy this would have been to miss.
The original code was completely functional.
Authentication worked. Tests passed. The API returned the correct responses. There was no exception or obvious bug telling me something was wrong.

If I hadn't been looking at production traces, I could have left it that way for months.
And I think this matters even more in the age of AI-assisted development.
AI has made it incredibly cheap to produce working software. You can describe an authentication flow, generate an implementation, run the tests and have something functional remarkably quickly.

But working code and well-behaved production software are not the same thing.

The code doesn't tell you that one innocent-looking function call crosses the network.
It doesn't tell you that the call sits on the critical path of every request.
And it doesn't tell you that your users are paying that latency tax hundreds or thousands of times a day.
Your production system does. You just have to be looking.

## Observability isn't optional

There's a tendency to think of observability as something you add when your system becomes "big enough."

I'd argue the opposite.

As it becomes easier to ship more software, faster, understanding what that software is actually doing becomes more important, not less. Logs tell you what happened.
Metrics tell you that something changed. Traces help tell you where the time went.  

In this case, nothing was down. Nothing was throwing errors. Nothing would have triggered an uptime alert.

The software was simply slower than it needed to be. A trace made the reason obvious.

That's also a big part of why I'm building [Still200](https://still200.com): a simple reliability layer for developers to monitor their APIs, websites, scheduled jobs and AI agents.
AI can help us ship code incredibly fast.

**We still have to understand what happens after we ship it.**
