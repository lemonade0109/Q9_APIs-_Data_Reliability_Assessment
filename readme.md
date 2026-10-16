# BCM Assessment Q9

**Files:**
subscriber.html — the subscribe function and browser-based test runner
q9_test.png — test output

**How to run:** open q9/subscribe.html in any browser. Tests run automatically on load. Node is not installed locally, so this runs in the browser instead of on the command line.

## The problem

The newsletter service accepted the subscriber, then the request timed out before the response came back. If I retry, the service might save the person twice. The fix is to make the retry safe, not to avoid retrying.

The way to make it safe is to give each signup a unique ID before the first request goes out. Every retry uses the same ID. If the service has already saved that ID, it recognizes the duplicate and returns success instead of creating a second record.

I've done this with Stripe webhooks before. Stripe sends the same event multiple times, and you have to handle each one without double-processing. Same shape of problem.

## How each failure is handled

**Timeout:** The request runs with a 5-second abort timeout. If the service accepted the signup but the response was lost, the retry carries the same idempotency key, and the service returns "already subscribed" instead of creating a duplicate. Retries use exponential backoff: 500ms, then 1000ms, then 2000ms. Max three attempts.

**Rate limit:** A 429 response includes a Retry-After header. The function reads the header and waits that long, then retries. If the header is missing, it falls back to 2 seconds. This is better than guessing a fixed delay, because the service tells us how long to wait.

**Invalid email:** A 400 response is not retried. It is logged with the email masked, and returned as a rejection. Retrying a client error that will never succeed wastes attempts and can get you rate-limited.

## What I store and what I log

**Stored:**

- The idempotency key, tied to the email, before the first attempt
- The final outcome: subscribed, rejected, or queued for retry
- Timestamp of each attempt
- HTTP status of each attempt

**Logged:**

- Event name (rejected, failed, subscribed)
- Email in masked form, never plain
- Reason for rejection from the service response
- Attempt number and error message on failure

**Not logged:**

- Full email in plain text. That's personal data.
- Any authentication headers or API keys
- Full response bodies from the service, since they can contain subscriber data

## How I test a retry after the service already accepted the request

The browser test simulates this directly. The mock service accepts the first request but throws a timeout instead of returning a response. The subscribe function retries with the same idempotency key. The mock recognizes the key on the second attempt and returns "already_subscribed". The test then counts how many entries exist for that email and confirms there is exactly one.

## The five tests cover

1. Happy path: signup succeeds on first attempt

2. Timeout then retry: confirms no duplicate entry (store count stays at 1)

3. Rate limit then success: confirms Retry-After is respected

4. Invalid email: confirms no retry and a clean rejection

5. Same idempotency key twice: confirms the service returns "already_subscribed" and no second record is created

## The two lines in the output that prove the fix works are

- Store entries for timeout.user@example.com: 1
- Store entries for dup@example.com: 1

Both confirm the retries did not create duplicates.

## Why this matters

Retries have to be safe. If a network hiccup, a deploy, or a rate limit can cause a duplicate signup, the system is broken even though every individual request succeeded. Idempotency keys are how you make retries safe. It's the same pattern I use for payment integrations.
