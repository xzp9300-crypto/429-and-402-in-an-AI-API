# 429-and-402-in-an-AI-API
429 and 402 in an AI API Gateway: What "Provider returned error" Actually Means
Editorial metadata — remove before publishing
•Slug:/errors/429-402-handling
•Role in cluster: A1 — hub. A2 and A3 link up to this page; this page links down to both.
•Primary keywords:openrouter 429 provider returned error, provider returned error, 429 too many requests api, 402 insufficient credits, retry-after header api retry
•Intent layers:
￮L1 diagnostic ("why am I getting this") — queries where the developer has an error in hand and needs a meaning. Target: rank for the exact error string, since the string is the query.
￮L2 procedural ("how do I retry it correctly") — queries about backoff, Retry-After, retry libraries, retry budgets.
￮L3 architectural ("how should my client be built") — queries about failover, multi-provider routing, production error policy.
•Primary CTA:/sign-up?lang=en
•Internal links:/pricing?lang=en, /ai-api-gateway/, /ai-model-integration/, /ai-api-gateway-faq/
•Word count target: 1,500+ (prose)

Short answer:Provider returned error is a wrapper message, not a diagnosis. It is the generic string an aggregating gateway emits when the failure happened somewhere upstream of it and it has nothing specific to say. The status code and the headers next to that string are what tell you what to do. A 429 with rate-limit headers on your own key means back off your request rate. A 429 carrying a provider code means the model's backend is saturated and you should retry with jitter or fall back to another model. A 402 means money. A 502 or 529 carrying that same sentence means the upstream fell over mid-request and your only good move is retry-with-model-fallback.
If you remember one rule from this page, make it this: branch on status + headers + the typed error field, never on error.message. The message is prose written for humans reading logs. It is not stable, it is deliberately vague on 5xx, and the same sentence is used for at least two different failure classes.
The rest of this page is the decision table, the header parsing you probably got wrong, and code you can paste.

1. Why the message string is useless (and provably so)
Look at how the largest gateway in this space documents its own error schema. In OpenRouter's published OpenAPI spec, the example body for 502 Bad Gateway is:
json
{ "error": { "code": 502, "message": "Provider returned error" } }
And the example body for 529 Provider Overloaded is:
json
{ "error": { "code": 529, "message": "Provider returned error" } }
Two different failure classes — a broken upstream response versus a saturated upstream — rendered as byte-identical text. That is not a bug in their docs; it is what a normalizing gateway does. It has no reliable way to describe the provider's internals, so it says the one true thing it knows: the provider failed.
There is a second reason the message goes vague: masking. OpenRouter's error documentation states that on 500-class failures the upstream message is replaced with a generic string and the raw provider code is dropped, specifically so upstream internals do not leak. If you build a client that greps for "Rate limit exceeded" in the message, you are building on text the gateway is entitled to rewrite.
So the string is a signal that something upstream broke, and nothing more. Everything actionable lives in three other places: the HTTP status, the response headers, and — if the platform exposes one — a typed code field (error_type, provider_code, error.code).
One more thing worth pinning down before we go further, because it explains a lot of confusing logs: which layer rejected you. In a gateway chain there are at least four layers that can say no — your client, the gateway's auth, the gateway's rate limiter, and the upstream provider. Each has a different fix. The status codes overlap, which is why people misdiagnose.

2. The four codes people conflate
These four are routinely treated as "the API is busy". They are not the same thing, and the correct response differs:
Status	Typical meaning in a gateway chain	Retryable?	What actually fixes it
402	Credits. Somewhere in the chain an account cannot pay for this request.	Usually no	Top up, raise the key limit, or shrink the request
403	Authorized but refused — quota exhausted on the gateway account, guardrail block, or policy refusal	no	Change account state or the request; retrying is pure waste
429	Rate or concurrency limit hit. Origin matters enormously.	yes, with the right wait	Slow down, back off, or route elsewhere
503	No capacity that satisfies your routing requirement	yes, short delay	Wait, or relax routing
502 and 529 sit alongside these as upstream-failure codes and are worth their own row:
Status	Typical meaning	Retryable?	Note
502	Upstream returned an invalid or empty response	yes, with fallback	Often transient provider bug
529	Upstream overloaded	yes, short delay	Capacity, not your rate
504 / 524	Timeout at the provider, or at the edge	yes, but read §6 first	These are the ones that break streaming clients
Now the important nuance: a 403 from your gateway's own billing is not a 402. They both mean "you cannot pay", they both come with a message about quota, and they are different codes depending on which software you are talking to. If you write if status == 402: top_up() and your gateway reports quota exhaustion as 403, your alerting silently misses every out-of-credit event. Handle both, and distinguish by the body's typed code, not by the number.
Concretely, on a New API–based gateway (the software family Copent's gateway reports itself as; see §8), the credit-exhaustion path returns 403 with an error code of insufficient_user_quota or pre_consume_token_quota_failed. We could not trigger this live without a funded account, so treat it as code-level evidence rather than a measured response: the source is public, the branch is unambiguous, but the deployed configuration is per-operator. Check your own account once and record what you actually get — that is a five-minute test and it beats any table, including this one.

3. 402: three different causes wearing one code
402 Payment Required is the code developers are most likely to over-simplify into "add money". In current gateway practice there are at least three distinct causes, and only one of them is retryable.
OpenRouter's limits documentation splits them explicitly:
1.Account balance. Negative or insufficient balance. This affects even free models on that platform. Fix: add credits. Not retryable.
2.Per-key credit limit. An optional spending cap on one API key. The key's limit_remaining is what you check. Fix: raise the cap or wait for limit_reset. Not retryable until the window rolls.
3.In-flight spending budget. A hold placed against your balance for estimated cost of requests currently running or recently completed. A request whose estimate does not fit alongside the in-flight holds is rejected with 402before it reaches a provider, even though your balance is positive.
Only case 3 is transient. Their doc is precise about how to tell: the response carries Retry-After, and error.metadata.limit_source is openrouter_in_flight_budget. A 402 without Retry-After is not a wait-and-retry case.
The practical code shape, then, is not "402 → sleep". It is:
python
if status == 402:
    if headers.get("retry-after"):
        # concurrency hold, not insolvency. Wait it out.
        sleep(parse_retry_after(headers["retry-after"]))
        retry()
    else:
        # real credit problem. Paging someone at 3am will not help.
        raise InsufficientCredit(detail=body)
And the operational point that follows: if you run high concurrency against a prepaid balance, a 402 can appear while your balance is positive. Teams have spent days on this. If your dashboard says "$40 left" and production says 402, check the in-flight budget before you check the invoice.

4. 429: the origin is the whole story
A 429 means "too many". The question that decides your response is too many of what, according to whom.
Origin A — the gateway's own limiter. Something in front of the model counted your requests (or your tokens, or your concurrent streams) and rejected this one before it did any work. Signals:
•x-ratelimit-limit-*, x-ratelimit-remaining-*, x-ratelimit-reset-* headers describing the limit that was hit.
•A Retry-After value that is short and specific.
•A body that does not name a provider.
Fix: reduce rate, honor the header, and if you are running many workers, make sure your limiter is shared (Redis or equivalent) rather than per-process. A per-process limiter multiplied by 16 workers is the single most common cause of "we only send 40 rps, why are we throttled".
Origin B — the provider's limiter. The request was accepted, routed to a model backend, and the backend said no. Signals:
•The status arrives with a typed provider code (provider_code, metadata.provider_code, or a provider-shaped code in the body).
•No gateway rate-limit headers, because you did not hit a gateway limit.
•The message may or may not be the generic Provider returned error.
Fix: this is not your rate. Retrying immediately with the same model will usually fail again. Use jittered backoff, and treat model fallback as the primary remedy rather than the last resort.
A wrinkle worth internalizing. Microsoft's documentation for Azure-hosted OpenAI models describes three separate causes of 429 on that service: request-rate bursts, backend system capacity, and a temporary reduction of your effective rate limit while the shared pool is under pressure. It also notes two things that make naive reasoning wrong: RPM enforcement is evaluated in small windows — their doc says 1–10 seconds — so a burst inside a ten-second window triggers throttling even when your per-minute total is under the limit; and rejected requests can still count toward rate limiting while not appearing in billed token metrics.
That is Microsoft's documentation for Azure, not a statement about OpenAI's own API, and the numbers there are theirs. But the mechanism generalizes: 429 is frequently not "you exceeded the number you configured". It can be a queue effect, a capacity effect, or a sub-minute burst effect. Which is exactly why the next section matters.

5. Measure your own limit instead of trusting a table
Published per-provider thresholds are a moving target, they differ by account tier, and they are revised without notice. Rather than hardcode a number you read somewhere, measure the limit on the account you actually run production traffic on, and record what the rejection looks like.
Two practical warnings before the probe:
An invalid key cannot produce a 429. Auth is evaluated before rate limiting in this gateway family, so a bad key short-circuits at 401 and never reaches the limiter. We verified this on Copent's gateway on 2026-11-02: 25 concurrent requests with an invalid bearer token returned 25 identical 401 responses and zero 429s. If you were planning to "test rate limits with a dummy key", that plan does not work.
Probe on the cheapest model you can use, with max_tokens set low. You are buying a header, not an answer.
Here is a zero-dependency probe that ramps concurrency until it observes a 429, then dumps the headers that describe the limit:
python
#!/usr/bin/env python3
"""
probe_429.py - find your real rate limit and capture the rejection headers.

Reads config from env:
  API_BASE   e.g. https://one.copentai.com
  API_KEY    a valid key on the account you care about
  MODEL      a cheap model id

Run:  API_BASE=... API_KEY=... MODEL=... python3 probe_429.py
"""
import json
import os
import time
import urllib.error
import urllib.request
from concurrent.futures import ThreadPoolExecutor

BASE = os.environ["API_BASE"].rstrip("/")
KEY = os.environ["API_KEY"]
MODEL = os.environ["MODEL"]

PAYLOAD = json.dumps({
    "model": MODEL,
    "max_tokens": 1,
    "messages": [{"role": "user", "content": "hi"}],
}).encode()

HEADERS_OF_INTEREST = (
    "retry-after",
    "retry-after-ms",
    "x-ratelimit-limit-requests",
    "x-ratelimit-remaining-requests",
    "x-ratelimit-reset-requests",
    "x-ratelimit-limit-tokens",
    "x-ratelimit-remaining-tokens",
    "x-ratelimit-reset-tokens",
    "x-request-id",
    "x-oneapi-request-id",
)

def one_call(_):
    req = urllib.request.Request(
        f"{BASE}/v1/chat/completions",
        data=PAYLOAD,
        headers={
            "Authorization": f"Bearer {KEY}",
            "Content-Type": "application/json",
        },
        method="POST",
    )
    started = time.time()
    try:
        with urllib.request.urlopen(req, timeout=30) as r:
            return r.status, dict(r.headers), r.read(600), time.time() - started
    except urllib.error.HTTPError as e:
        return e.code, dict(e.headers), e.read(600), time.time() - started
    except Exception as e:  # network-level failure
        return None, {}, str(e).encode(), time.time() - started

def interesting(headers):
    lowered = {k.lower(): v for k, v in headers.items()}
    return {k: lowered[k] for k in HEADERS_OF_INTEREST if k in lowered}

def main():
    for burst in (2, 4, 8, 16, 32):
        t0 = time.time()
        with ThreadPoolExecutor(max_workers=burst) as pool:
            results = list(pool.map(one_call, range(burst)))
        codes = {}
        for status, headers, body, elapsed in results:
            codes[status] = codes.get(status, 0) + 1
            if status == 429:
                print(f"\n--- 429 at burst={burst} ---")
                print("headers:", json.dumps(interesting(headers), indent=2))
                print("body:", body.decode("utf-8", "replace")[:400])
                print(f"window elapsed: {time.time() - t0:.2f}s")
                return
        print(f"burst={burst:>3}  codes={codes}  window={time.time() - t0:.2f}s")
        time.sleep(2)

    print("\nNo 429 observed up to burst=32. The limit is above this probe, "
          "or it is token-based rather than request-based. Raise burst, or "
          "send longer prompts to push tokens.")

if __name__ == "__main__":
    main()
What to keep from the run: the status mix, the retry-after value, and any x-ratelimit-* headers. That triple is your account's real contract, and it will survive the next time a docs page changes.

6. Retry-After is two formats, and most clients parse one
Retry-After is defined in RFC 9110 §10.2.3 and accepts either a number of seconds or an HTTP-date:
Retry-After: 120
Retry-After: Mon, 02 Nov 2026 07:14:00 GMT
Number('Mon, 02 Nov...') is NaN. parseInt('Mon, 02 Nov...') is NaN. A very large fraction of retry code in the wild does one of those two things and then falls through to a default because the parse produced garbage — which means the client waits 1 second when the server said "wait two minutes", gets rejected again, and burns its retry budget in under ten seconds.
Also note that Retry-After is not a general-purpose header on every error. It appears on 429 and 503, and on specific 402 cases that represent a hold rather than insolvency. Do not assume a 502 carries one.
Parse it properly:
python
import email.utils
import time
from datetime import datetime, timezone

def parse_retry_after(value, now=None):
    """Return seconds to wait, or None if the header is absent/unparseable."""
    if not value:
        return None
    value = value.strip()

    # Form 1: delay-seconds
    try:
        secs = float(value)
        return max(0.0, secs)
    except ValueError:
        pass

    # Form 2: HTTP-date (RFC 9110 s10.2.3)
    try:
        dt = email.utils.parsedate_to_datetime(value)
    except (TypeError, ValueError):
        return None
    if dt is None:
        return None
    if dt.tzinfo is None:
        dt = dt.replace(tzinfo=timezone.utc)
    now = now or datetime.now(timezone.utc)
    return max(0.0, (dt - now).total_seconds())
Two guards on top of the parse. First, cap it. A server that says "wait 3600" should not park your worker for an hour; clamp to something like 60s and let the caller decide whether to fail the job. Second, add jitter even when the header is present. If forty workers all read Retry-After: 30 and all sleep exactly 30.0 seconds, you have built a synchronized thundering herd out of a header that was supposed to prevent one.

7. A retry policy you can actually ship
Most retry bugs are not in the backoff formula. They are in three places: retrying something that will never succeed, retrying twice through two layers, and retrying without a budget.
Don't retry what cannot succeed.400, 401, 403, 404, 413, 422 are deterministic for the same request. Retrying them multiplies your error rate, adds latency, and in the 401 case can trip provider-side key-disable heuristics. A 400 that says your context is too long will say the same thing on attempt five.
Don't retry twice. If you use an SDK that retries internally and a retry library wrapped around it, a single logical request can become nine HTTP requests with three independent backoff schedules. Pick exactly one layer: either disable SDK retries and own them, or let the SDK own them and never wrap it. The failure mode is ugly — you see one failure, the provider sees a burst.
Do budget. A retry loop without a deadline is a latency bomb.
typescript
// retry.ts - a retry policy with a deadline, a budget, and correct 402/429 handling.
// Node 18+. No dependencies.

type Class = "retry" | "retry_with_fallback" | "fail_fast" | "billing";

interface Decision {
  cls: Class;
  reason: string;
}

interface RetryConfig {
  maxAttempts: number;
  baseDelayMs: number;
  maxDelayMs: number;
  deadlineMs: number;
}

const DEFAULT: RetryConfig = {
  maxAttempts: 4,
  baseDelayMs: 500,
  maxDelayMs: 15_000,
  deadlineMs: 90_000,
};

export function classify(status: number, headers: Headers, bodyText: string): Decision {
  const retryAfter = headers.get("retry-after");

  if (status === 402) {
    // A 402 that hands you a wait is a concurrency hold, not insolvency.
    return retryAfter
      ? { cls: "retry", reason: "in_flight_hold" }
      : { cls: "billing", reason: "insufficient_credit" };
  }

  if (status === 403) {
    // Gateway quota exhaustion lands here on New API-family gateways,
    // and policy refusals land here on most platforms.
    return bodyText.includes("insufficient_user_quota") ||
      bodyText.includes("pre_consume_token_quota_failed")
      ? { cls: "billing", reason: "account_quota_exhausted" }
      : { cls: "fail_fast", reason: "refused" };
  }

  if (status === 429) {
    // Same status, two very different fixes.
    const hasGatewayLimit =
      headers.get("x-ratelimit-limit-requests") !== null ||
      headers.get("x-ratelimit-remaining-requests") !== null;
    return {
      cls: hasGatewayLimit ? "retry" : "retry_with_fallback",
      reason: hasGatewayLimit ? "own_rate_limit" : "provider_capacity",
    };
  }

  if (status === 503) return { cls: "retry_with_fallback", reason: "no_capacity" };
  if (status === 502 || status === 529) return { cls: "retry_with_fallback", reason: "upstream_failure" };
  if (status === 500 || status === 504 || status === 524) return { cls: "retry", reason: "transient_5xx" };

  if (status >= 400 && status < 500) return { cls: "fail_fast", reason: "deterministic_4xx" };

  return { cls: "retry", reason: "unclassified" };
}

function parseRetryAfter(value: string | null): number | null {
  if (!value) return null;
  const trimmed = value.trim();

  const asNumber = Number(trimmed);
  if (Number.isFinite(asNumber) && asNumber >= 0) return asNumber * 1000;

  const asDate = Date.parse(trimmed); // handles HTTP-date
  if (!Number.isNaN(asDate)) return Math.max(0, asDate - Date.now());

  return null;
}

function fullJitter(attempt: number, cfg: RetryConfig, retryAfterMs: number | null): number {
  if (retryAfterMs !== null) {
    // Honor the server, but add jitter so N workers do not wake together.
    return Math.min(cfg.maxDelayMs, retryAfterMs + Math.random() * 0.2 * retryAfterMs);
  }
  const ceiling = Math.min(cfg.maxDelayMs, cfg.baseDelayMs * 2 ** attempt);
  return Math.random() * ceiling; // "full jitter" — AWS-style
}

export async function withRetry<T>(
  fn: (signal: AbortSignal) => Promise<T>,
  onFallback?: () => Promise<T>,
  cfg: Partial<RetryConfig> = {},
): Promise<T> {
  const c = { ...DEFAULT, ...cfg };
  const started = Date.now();
  let lastError: unknown;

  for (let attempt = 0; attempt < c.maxAttempts; attempt++) {
    if (Date.now() - started > c.deadlineMs) break;

    const controller = new AbortController();
    const remaining = c.deadlineMs - (Date.now() - started);
    const timer = setTimeout(() => controller.abort(), remaining);

    try {
      return await fn(controller.signal);
    } catch (err: any) {
      lastError = err;
      const status: number | undefined = err?.status;
      if (status === undefined) throw err; // network error: let the caller decide

      const decision = classify(status, err.headers ?? new Headers(), String(err.body ?? ""));

      if (decision.cls === "fail_fast" || decision.cls === "billing") throw err;

      const wait = fullJitter(attempt, c, parseRetryAfter(err.headers?.get?.("retry-after") ?? null));
      if (Date.now() - started + wait > c.deadlineMs) break;
      await new Promise((r) => setTimeout(r, wait));

      if (decision.cls === "retry_with_fallback" && onFallback) {
        return await onFallback();
      }
    } finally {
      clearTimeout(timer);
    }
  }

  throw lastError;
}
The part that carries the most value is classify, not the backoff. A 429 carrying gateway rate-limit headers means you are the problem and slowing down is correct. A 429 without them means the model's backend is the problem and switching models is correct. Those are opposite actions behind an identical status code, and a client that cannot tell them apart will either hammer a saturated provider or needlessly downgrade its own quality.

8. What Copent-specific error output looks like
These are measured against the live gateway on 2026-11-02, using an invalid credential, so they describe the envelope rather than success paths:
HTTP/1.1 401 Unauthorized
X-New-Api-Version: v0.0.0
X-Oneapi-Request-Id: 202611020612506741553508268d9d6NUOCxBVk

{"error":{"code":"","message":"Invalid token (request id: ...)","type":"new_api_error"}}
Three things to build against:
•type is a stable field. The gateway uses a typed vocabulary (new_api_error, openai_error, claude_error, upstream_error, and others). code is sometimes an empty string while type is populated — so never key your logic on code being present.
•Every rejection carries a request id, in the X-Oneapi-Request-Id response header and again inside the message text. Put it in your logs. When you open a support thread, that id is the difference between "it failed" and a five-minute answer.
•A 401 beats a malformed body. We sent broken JSON with an invalid key and got 401, not 400 — auth is evaluated before the body is parsed, so a syntactically invalid request with a bad credential reports as an auth failure. If you are debugging a 400-vs-401 confusion, check the credential first.
The Anthropic-shaped route (POST /v1/messages with x-api-key) returns the same envelope, so you do not need a second parser for the Claude skin.
Routes that exist on the gateway, confirmed by credential probe: /v1/chat/completions, /v1/completions, /v1/responses, /v1/messages, /v1/embeddings, /v1/rerank, /v1/images/generations, /v1/audio/speech, and the Gemini-native /v1beta/models/{model}:generateContent. Unknown paths return 404 with type: "invalid_request_error" and an Invalid URL (POST /path) message — which is, incidentally, the cheapest way to tell "wrong base path" (404) from "wrong key" (401).
What is not verified yet: your account's specific rate limits, concurrent-request caps, and whether retry-on-upstream-failure is enabled for your key. Those are per-account configurations. Check them in the console, and note that Copent's own documentation asks you to confirm limits and billing units in the signed-in console before sending production traffic rather than relying on published examples.
Two standing caveats from Copent's own published pages, worth quoting because they set expectations correctly: the Terms of Service state that the platform "strive[s] for 99.99% high availability with automatic failover" while services are provided "as-is" and "as-available" — an aspiration plus a disclaimer, which is exactly why your client needs its own retry policy and not just the gateway's. And the integration guide warns against relying on "copied examples that have not been verified against your account."

9. What to log so the next incident takes ten minutes
For each failed call, log: status, type, code, provider_code if present, retry-after raw string, the gateway request id, the model id, whether the request was streaming, the attempt number, and your final classification. That is eight fields and it collapses almost every argument about "whose fault was it".
Then build two counters from it: failures classified billing (should page someone) and failures classified retry_with_fallback (should not). Mixing them into one "error rate" metric is how a saturated provider becomes a false alarm at 2am and a real insolvency gets ignored.

10. FAQ, in the order people search
I see 429 with provider returned error from OpenRouter. Am I rate limited?
Not necessarily by OpenRouter. That combination usually means a provider in the chain was at capacity. Check whether the body carries a provider code and whether rate-limit headers are present. If they are absent, slowing your own request rate will not help; changing model or relaxing provider routing will.
Is provider returned error retryable?
Generally yes, but not by repeating the identical request immediately. It indicates upstream failure or upstream saturation. Retry with jittered backoff and, for repeated occurrences on the same model, fall back to another model.
Why do I get 402 when my balance is positive?
Most likely an in-flight spending hold: concurrency-based estimation of cost, not actual insolvency. If the response carries Retry-After, wait and retry; if it does not, it is a real credit problem.
Should I retry a 504?
Carefully. A 504 from an upstream that never began responding is safe to retry. A 504-equivalent that arrives after your stream started is not a status code at all — it is a truncated body, and it is handled in our timeout and streaming guide. Retrying a partially-delivered answer duplicates output and, in most gateways, duplicates billing.
Why does my retry code retry four times in two seconds?
Because it is parsing Retry-After with parseInt, getting NaN, and falling back to a fixed 500ms. See §6.
Do I need different code for the Claude or Gemini routes?
The rejection envelope is the same shape on both the Anthropic route and the Gemini-native route as measured here — an invalid credential returned the same new_api_error body on all three. What an upstream Gemini or Claude error looks like after it is forwarded is a separate question we have not verified on this gateway, so keep one internal error type and normalize at the boundary rather than assuming the upstream wire format survives intact.

Where to go next
For failures where the interesting question is which component failed — your code, the gateway, the provider, or the model's own limits — the provider attribution guide has a classifier and a five-experiment probe you can run in one sitting. For streams that die after tokens have already arrived, the timeout and streaming interruption guide is the one to read, because the failure there is invisible in the status code.
If you want to run these probes against a real account instead of reading about them, create one — the probe scripts here need a valid key and nothing else. Current per-model rates are on the pricing page, and per-account limits are only accurate in the console.

Appendix: source notes
•OpenRouter error semantics, response shapes, Retry-After scoping, mid-stream error structure, typed error_type tables and message masking: https://openrouter.ai/docs/api_reference/errors-and-debugging
•OpenRouter credit/budget model, three 402 causes, in-flight budget mechanics, 403 budget errors: https://openrouter.ai/docs/api_reference/limits
•502 and 529 both documented with the example message Provider returned error: OpenRouter's published OpenAPI document, components.schemas.BadGatewayResponse and components.schemas.ProviderOverloadedResponse, https://openrouter.ai/openapi.json
•Retry-After grammar (delay-seconds or HTTP-date): RFC 9110 §10.2.3, https://www.rfc-editor.org/rfc/rfc9110#name-retry-after
•Three distinct 429 causes, 1–10 second burst windows, rejected-request accounting, x-ratelimit-* and retry-after-ms header semantics on Azure-hosted OpenAI models: https://learn.microsoft.com/en-us/azure/foundry/openai/how-to/quota
•Gateway retry classification, skip-retry status codes, streaming error writer, Retry-After emission, and 403 for account quota exhaustion: New API source, https://github.com/QuantumNous/new-api (setting/operation_setting/status_code_ranges.go, service/relay_error.go, controller/relay.go, middleware/rate-limit.go, service/billing_session.go)
•Copent live envelopes, headers, route inventory and 401-before-400 behaviour: direct probes against https://one.copentai.com on 2026-11-02
•Copent availability language: https://one.copentai.com/terms (last updated October 2026)
•Copent guidance on verifying limits and examples per account: https://one.copentai.com/ai-model-integration/

Data risk and confidence note
Scene/Argument	Facts	Data Risks and Limitations	Confidence	Verification method
The provider returned an error that corresponds to both 502 and 529 failure types.	Facts verified	This conclusion comes from the example definition in the OpenRouter public OpenAPI documentation and does not mean that other gateways must use the same string.	high	The original document is components.schemas.BadGatewayResponse / ProviderOverloadedResponse
A 402 error can be caused by three factors: account balance, single key limit, or in-flight budget. Only the third factor allows for retrying.	Facts verified	This describes the rules for the OpenRouter platform; the 402 semantics may differ for other gateways.	high	openrouter.ai/docs/api_reference/limits
Azure-hosted OpenAI has three causes for 429 errors, is evaluated within a 1–10 second window, and rejected requests are still counted in rate limiting.	Facts verified	This is from Microsoft Azure documentation and is not equivalent to the thresholds in the official OpenAI API. This article does not cite any official threshold figures.	high	Azure quota documentation at learn.microsoft.com
The Copent gateway 401 response header contains X-Oneapi-Request-Id and X-New-Api-Version, with type new_api_error.	Facts verified	Testing with only invalid keys revealed a rejected path rather than a successful path.	high	On November 2, 2026, a direct request was sent to one.copentai.com.
Invalid key cannot trigger 429 (authentication takes precedence over rate limiting)	Facts verified	A single experiment involved 25 concurrent 401 attempts, not covering all rate-limited branches.	high	Same as above, 25 concurrent probes
The gateway returns a 403 (insufficient_user_quota) error when its own quota is exhausted.	Facts to be verified	The source is open-source implementation code; Copent deployment and configuration have not been tested (lack of usable balance accounts).	middle	The source code for the new API is available at github.com/QuantumNous/new-api, and testing with an account is pending.
Actual rate limits for each account, concurrency limits, and whether upstream failure retries are enabled.	Facts to be verified	It is configured on an account-by-account basis and is not disclosed on the public page.	Low	You need to log in to the console to confirm; the text has been marked "not verified yet".
The code examples in this article (probe_429.py, parse_retry_after, retry.ts)	Facts verified	The code is a generic implementation and has not been fully tested and successfully deployed on a Copent production account.	middle	Local syntax and logic self-check; it is recommended to test each line individually before scaling up.
