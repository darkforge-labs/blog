---
layout: post
title:  "A Case for Authentication at the Edge"
date:   2025-04-01 09:00:00 +0200
categories: Next.js middleware authentication
---

# Authentication at the Edge

When [CVE-2025-29927](https://zhero-web-sec.github.io/research-and-things/nextjs-and-the-corrupt-middleware) dropped, it lit up the infosec world. An authentication bypass in Next.js middleware raised urgent questions not just about the vulnerability itself but about architectural choices.

*Should we be enforcing authentication in the middleware?*

It’s a question that’s polarized developers and security engineers alike. [Our own X poll came back split almost exactly down the middle.](https://x.com/moopinger/status/1906281539815579790)

But after digging into the excellent write-ups from [neox](https://www.neoxs.me/blog/critical-nextjs-middleware-vulnerability-cve-2025-29927-authentication-bypass), [Assetnote](https://slcyber.io/assetnote-security-research-center/doing-the-due-diligence-analysing-the-next-js-middleware-bypass-cve-2025-29927), [JFrog](https://jfrog.com/blog/cve-2025-29927-next-js-authorization-bypass/), and [Rapid7](https://www.rapid7.com/blog/post/2025/03/25/etr-notable-vulnerabilities-in-next-js-cve-2025-29927/) — we realized something:

We’ve been asking the wrong question.

## The Real Question: Should *All* Your Authentication Run in *Next.js* Middleware?

That’s the actual crux. And the answer is a resounding <span class="underline">no</span>.

Middleware in Next.js, especially in the App Router, runs in an **edge runtime**. That means it operates in a different environment than your backend or API routes. It’s fast, global, and lightweight. But it also has real limitations:

- [The edge runtime runs on a separate more limited Node runtime](https://nextjs.org/docs/app/building-your-application/rendering/edge-and-nodejs-runtimes)
- [Limited access to full request bodies](https://clerk.com/blog/what-is-middleware-in-nextjs)

Compare this to `Express.js` middleware, which runs in a Node.js server with full control over the request lifecycle. Same word *middleware* but completely different realities.

That distinction matters. Because when you put critical auth logic in Next.js edge middleware, you're betting your security on a layer that wasn’t built for that burden.

Even **Vercel** says as much in their [postmortem](https://vercel.com/blog/postmortem-on-next-js-middleware-bypass):

> "We do not recommend Middleware to be the sole method of protecting routes in your application"

## So What Is Edge Based Middleware Good For?

Used correctly, Next.js middleware can still be a powerful tool — especially for **coarse-grained access control**. Think of it like the lobby guard, not the vault door.

* **Use it to block unauthenticated users** from public routes  
* **Redirect users to login** if they’re missing a session  
* **Apply basic tenant-aware routing**  

But the real enforcement should happen deeper:

* **Don't check roles or permissions here**  
* **Don’t make critical auth decisions** based only on this layer

Instead, **put your fine-grained logic** on the server side — RBAC, ABAC, tenant isolation — **inside your API routes or backend logic**.

## Lessons from CVE-2025-29927

The flaw that triggered this debate didn’t involve tricky encoding or exotic Unicode behavior to bypass route matches. It was much simpler: by crafting a request with a specific HTTP header (`x-middleware-subrequest`), attackers could trick the server into skipping middleware entirely — and bypass authentication logic that only lived there.

It’s a classic case of trusting the wrong layer.

> **If one bypass breaks your security, you didn’t have security. You had hope.**

Here’s how to build for reality:

- **Defend in depth** — don’t make any single layer your only layer.
- **Authorize where it counts** — at the point of data access, in your APIs.

## Final Thoughts

CVE-2025-29927 isn’t the first middleware bypass — and it likely won’t be the last. That doesn’t mean middleware auth in Next.js is useless. Far from it.

Edge-based middleware is still a great place for coarse-grained controls. It can improve UX. It can help performance. It can offload basic gating logic.

But <span class="underline">it can’t be your only line of defense.</span>

Use edge middleware wisely, as a first-pass filter, not your final gatekeeper. Build layers. Validate everywhere. And expect failure.

Because someone out there is already crafting the next bypass.