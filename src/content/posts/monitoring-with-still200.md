---
title: Monitoring a FastAPI App and Its Background Workers with Still200
published: 2026-10-23T18:00:00.122Z
description: 'How I set up uptime and background worker monitoring for a FastAPI application using Still200'
updated: ''
tags:
  - Uptime monitoring
  - Still200
draft: false
pin: 0
toc: true
lang: ''
abbrlink: ''
---

I recently built a small side project, [Kindly Note](https://kindlynote.app), a fun
little app that uses AI to turn everyday messages into beautifully worded notices, which you can then send anonymously via email.

After getting it up and running, I figured would make a good real-world example of how to set up monitoring with [Still200](https://still200.com), another product I've been building.

The idea is simple: I want to know when something breaks, preferably before someone using the app tells me.

## Quick look at the stack

Kindly Note is intentionally simple:
- **Python/FastAPI** for the backend
- **Jinja templates + HTMX** for the frontend
- **Postgres** for the database and notification delivery job queue
- **OpenAI API** for drafting the notices
- A background worker written in Python for processing notifications

Even with a relatively simple setup like this, quite a few things can go wrong.

The API might be responding perfectly fine, but the database could be unavailable. The OpenAI API could be failing, preventing users from generating notices. Or the background worker could stop processing emails, leaving notices sitting in the queue indefinitely.

In all these cases, a simple check that the homepage returns HTTP 200 wouldn't tell me the full story.

## What I want to monitor

For this setup, I want to answer the following questions:
1. **Is the application up?** Can users reach the app and get a successful HTTP response?
2. **Is the database healthy?** Can the application connect to and query Postgres?
3. **Is notice generation working?** Can the application communicate with the OpenAI API?
4. **Is the email worker running?** Is it actively processing jobs, or has it silently stopped?

I'll walk through how I set up these checks using Still200, including the small changes I made to Kindly Note to expose the information needed for monitoring.

## Setting up HTTP monitoring

Let's start with the basics, that is, making sure the application is up and responding
to requests.

Still200 supports HTTP monitoring, which periodically sends requests to a URL and checks whether the application responds successfully. If a check fails, Still200 records an incident and sends a notification.

For KindlyNote, I've exposed a simple health endpoint in the FastAPI application:

```python
@app.get("/health")
async def health():
    return {"service_name": "Kindly Note"}
```

This gives us a lightweight endpoint at https://kindlynote.app/health that returns an HTTP 200 OK response when the application is running.

## Creating the monitor in Still200

With the endpoint ready, I can head over to the [Still200 dashboard](https://app.still200.com) and create a new HTTP monitor.
The setup is straightforward: add the URL, validate it, set the check interval. That's it.

<video autoplay muted loop playsinline
       style="width: 100%; border-radius: 10px;">
  <source src="/videos/still200-demo.mp4" type="video/mp4">
</video>

Still200 will now send a request to the health endpoint every minute and track whether the application is responding successfully.

If the endpoint becomes unreachable, times out, or returns an unsuccessful HTTP status code, Still200 can detect the failure and notify me.

Once the monitor is running, I can also see its uptime history, response times, and any incidents that have occurred.

This is a good starting point. However, a `200` does not tell the whole story.
What if Postgres is unavailable? Or the email delivery worker has stopped running?
The application could still return HTTP 200 while some of its most important functionality is broken.

That's where dependancy monitoring comes in.

In the next section, we'll extend this health endpoint to report the status of the application's dependencies, giving us a much better picture of what's actually working.

## Monitoring application dependencies

Now that we know Kindly Note is reachable, let's take things a little further.

An application being up doesn't necessarily mean everything is working. Kindly Note, for example, relies on Postgres to store notices and queue email deliveries, and the OpenAI API to generate notices.

If either of these dependencies fails, parts of the application stop working, even if the FastAPI server continues responding to requests.

Fortunately, Still200 supports monitoring individual components through the same HTTP monitor we just created.

## Extending the health endpoint

Instead of simply returning `{"service_name": "Kindly Note"}`, we can extend the health
endpoint to include the health of the application's dependencies.

For Postgres, we'll execute a lightweight `SELECT 1` query to verify that our application
can communicate with the database. Using FastAPI and SQLAlchemy, the check might look something like this:

```python
import asyncio
import logging
import time

from sqlalchemy import text
from sqlalchemy.ext.asyncio import AsyncSession

logger = logging.getLogger(__name__)


async def check_postgres(session: AsyncSession) -> dict:
    start = time.perf_counter()
    error = None

    try:
        async with asyncio.timeout(3):
            await session.execute(text("SELECT 1"))

    except TimeoutError:
        error = "Database check timed out"
        logger.warning("Postgres health check timed out")

    except Exception:
        error = "Database connection failed"
        logger.exception("Postgres health check failed")

    latency_ms = round(
        (time.perf_counter() - start) * 1000, 2
    )

    result = {"latency_ms": latency_ms}

    if error:
        result["error"] = error

    return {"postgres": result}
```

**What this gives us:**

A successful check:

```json
{
  "postgres": {
    "latency_ms": 22.14
  }
}
```

A timeout:

```json
{
  "postgres": {
    "latency_ms": 3001.27,
    "error": "Database check timed out"
  }
}
```

A couple of things worth pointing out here.

First, we're using asyncio.timeout() to limit the database check to three seconds. If Postgres becomes unresponsive, we don't want our health endpoint waiting indefinitely.

Second, we're measuring elapsed time even when the check fails. This gives us a useful latency measurement regardless of the outcome.

Finally, we deliberately avoid exposing raw database exceptions in the response. The actual error gets logged on the server, while the health endpoint returns a simple, safe error message.

With this in place, Still200 can determine whether the Postgres dependency is healthy and track how long the check takes.

## Monitoring notice generation

The next dependency is the OpenAI API, which Kindly Note uses to generate its notices.