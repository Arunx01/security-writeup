# Heap Dump to Full Admin Takeover: A Black-Box Chain Walkthrough

*Three "low severity" findings. One full identity-platform compromise. Here's how we chained them.*

---

Black-box engagement, zero credentials to start. This is the walkthrough of how a handful of overlooked misconfigurations turned into complete control over the target's admin console.

**Status:** Reported and fixed before publishing. Domains, tokens, and credentials below are all placeholders.

## Step 1: The exposed endpoint

Standard enumeration turned up a Spring Boot actuator endpoint that should never be public:

```
https://target.example.com/security-api/actuator/heapdump
```

That endpoint hands you a full snapshot of the app's live memory. On a checklist, this gets logged as "information disclosure, low-medium" and most testers move on. We didn't.

## Step 2: Let AI do the grepping

A heap dump is hundreds of megabytes of raw memory, mostly noise. Manually searching it for anything useful is slow and you miss things.

We fed it to Claude (needed a paid plan for the context length; most free tools choke on a file this size) and asked it to flag regions likely to hold structured secrets instead of blind-searching the whole thing.

It found a **stale authentication token** sitting in memory, left behind from a session the garbage collector hadn't cleared yet.

We verified it manually before trusting it. AI narrowed the haystack; it didn't hand us the needle for free.

> **No paid AI access? You can still do this analysis.** Heap dumps aren't a new problem, and there are solid free tools built specifically for this:
> - **Eclipse Memory Analyzer (MAT)** – free, purpose-built for Java heap dump analysis, has a "leak suspects" and string-search feature that's genuinely good at surfacing credential-shaped strings.
> - **VisualVM** – free, bundled with most JDK distributions, lighter-weight than MAT but fine for smaller dumps.
> - A local open-source LLM with a long context window, run through a tool like LM Studio or Ollama, can approximate what we did with Claude, just expect slower and less precise triage than a hosted frontier model.
> The AI step sped things up. It wasn't the only way in.

## Step 3: Chain it into an admin token

Separately, earlier recon had turned up some internal config values from other exposed files, individually boring findings. Combined with the token from Step 2, that was enough to request a **valid admin-scoped token** straight from the platform's token endpoint.

```bash
curl -s -X POST "https://target.example.com/auth/realms/<realm>/protocol/openid-connect/token" -d grant_type=client_credentials -d client_id=<REDACTED_CLIENT_ID> -d client_secret=<REDACTED_CLIENT_SECRET> | jq -r .access_token
```

Catch: the token's lifetime was very short. No time for back-and-forth, so everything from here on had to be scripted and fast.

## Step 4: Find the admin, reset the password

With a valid admin token, the user list is just an API call away:

```bash
curl -s -H "Authorization: Bearer <REDACTED_TOKEN>" "https://target.example.com/auth/admin/realms/<realm>/users"
```

Grab the ID of a privileged account, then reset its password directly, no email confirmation, no MFA, nothing standing in the way:

```bash
curl -s -X PUT -H "Authorization: Bearer <REDACTED_TOKEN>" -H "Content-Type: application/json" -d '{"type":"password","value":"<REDACTED_NEW_PASSWORD>","temporary":true}' "https://target.example.com/auth/admin/realms/<realm>/users/<REDACTED_USER_ID>/reset-password"
```

## Step 5: Walk in the front door

Logged in with the new password, got the expected forced password-change prompt, set a real one, and landed straight in the **IAM admin console**. Full control over users, realms, roles, and access policies. Started with nothing. Ended with the keys to the whole identity layer.

## Why nobody caught this

Every link in this chain is "fine" on its own:

| Finding alone | Usual rating |
|---|---|
| Exposed actuator endpoint | Low-Medium |
| Config disclosure | Low |
| Stale token in memory | Easy to overlook |

Scanners score findings in isolation and move on. They don't ask "what if I combine this with that other thing I found an hour ago?" That's still a human job. Where AI actually helped wasn't the thinking, it was clearing the mechanical bottleneck (parsing a huge dump) so we had time to do the thinking. And as shown above, free tools can get you most of the way there too.

## Fixes

- Disable or firewall `/actuator/*` endpoints in production, full stop
- Shorter token lifetimes, proper session invalidation, no secrets lingering in memory
- Strip client secrets and config out of anything web-reachable
- Require a second factor for admin password resets, even with a "valid" admin token
- Scope and log admin-token usage tightly

## Takeaway

The most damaging black-box findings rarely come from one critical bug. They come from patiently connecting a handful of "nobody cares" issues that nobody else bothered to chain.
