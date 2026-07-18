---
title: 'Timing Attacks: Stop Comparing Secrets with =='
published: 2026-07-17T20:00:00.198Z
description: 'How a RevenueCat webhook integration taught me about timing attacks'
updated: ''
tags:
  - Security
  - Python
draft: false
pin: 0
toc: true
lang: ''
abbrlink: ''
---

# Introduction

This week, I worked on integrating
[RevenueCat](https://www.revenuecat.com) into my monitoring tool,
[Still200](https://still200.com), to manage subscriptions on the app.
Part of the work included listening to their webhooks in my API.
A webhook is simply a way for a system to receive updates from
another system. Instead of your system repeatedly polling another
server for updates, you specify a URL where the system that needs
to provide an update can push data to once a predefined event occurs.
In my case building out subscriptions management, I need my system to
be notified about new purchases, renewals and subscription expiry events.

To protect your webhook URL and ensure that you only process events
from the trusted source, RevenueCat allows you to define a secret token.
They will pass that token in the `Authorization` header of every request
to your webhook URL.
Your API verifies it against your stored secret before doing any heavy lifting.
That comparison step turned out to be the most interesting part of the build. It forced me to look closely at a classic vulnerability: **Timing Attacks**.

# Generating the secret

The best way to generate cryptographically strong strings for security-sensitive
operations is by using... you guessed it right... the `secrets` module 😃

Here's how to do it:

```python
import secrets

# URL-safe, base64-based token — good default for headers/env vars
token = secrets.token_urlsafe(32)
print(token)
# 'kR3f9z_Yq2vXW1pLd8Nc0Th5mBs7eUo4iJgAxPqZrKw'
```

I used `token_urlsafe` to avoid characters that could cause issues in HTTP headers.

# Verifying it in my API

Now, in my API, I needed to get this header value and compare it
to what I generated to verify that the request is indeed coming from
RevenueCat. Here's how I did it in FastAPI with the verification method
as a dependency, so that the request fails if the header value is invalid:

```python
import hmac

from fastapi import APIRouter, Depends, HTTPException, Request, status

from app.subscriptions.schemas import RevenueCatWebhookSchema # defined Pydantic schema

router = APIRouter()

async def verify_rc_headers(request: Request):
    auth_header = request.headers.get("Authorization")

    # DON'T DO THIS
    # if auth_header != settings.REVENUE_CAT_WEBHOOK_SECRET:
    #   ...
    
    if not auth_header or not hmac.compare_digest(
        settings.REVENUE_CAT_WEBHOOK_SECRET, auth_header
    ):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED, detail="Invalid auth header"
        )


@router.post("/webhooks/rc", status_code=status.HTTP_200_OK)
async def rc_webhook(
    rc_data: RevenueCatWebhookSchema,
    _=Depends(verify_rc_headers),
):
    ...  # Process the webhook event
```

# The timing attack: why `==` leaks

When Python evaluates `string1 == string2`, it evaluates characters from left to right
and short-circuits the moment it hits a mismatch. If the very first character is wrong,
it bails instantly. If the first five characters match before a failure, the operation
takes slightly longer.

A wrong guess that matches more of the correct string's prefix
takes slightly longer to reject than one that matches less. In theory,
an attacker who can measure response times very precisely could exploit
this by guessing one character at a time, watching which guess takes
marginally longer, and gradually reconstructing your secret token
character by character.

`hmac.compare_digest` is a constant-time comparison. It compares the
full length of both inputs every time, regardless of where mismatches occur,
so the execution time doesn't leak information about how much of the string matched.

# Conclusion

In practice, this might be overkill for my webhook endpoint.
Exploiting a timing attack over the internet against a webhook
endpoint is very hard to pull off reliably.
But it costs me nothing to use it, and it's the kind of small habit that's
good to bake in everywhere you're comparing secrets, so it's worth doing
as a default rather than reasoning case-by-case about whether a given
endpoint is "sensitive enough" to warrant it.
