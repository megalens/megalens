# The SSRF Fix That Passed Its Tests and Was Still Unsafe to Ship
*What the council caught after the first-pass patches already looked done*


## The Setup

FixBounce is an email verification SaaS. On April 10, an internal security audit surfaced 3 blocker findings and 4 high-severity findings. One of those highs was an ops rotation item outside the code patch, so the engineer spent the session on the production code fixes: suspension enforcement, SSRF hardening, bulk-credit race protection, fail-fast secret handling, and upload-size enforcement.

This was one product, one engineer, one session, four touched files, and one coordinated production deploy. Not a universal benchmark. A real remediation sprint on a live system. Here, "council" means Gemini and Codex independently reviewing the remediation diff after the engineer's first pass. The numbers in this write-up come from the actual review bundle and post-fix verification notes, not model-to-model agreement alone.

The first pass looked good.

The patches compiled. The attack paths named in the audit were addressed. The SSRF fix even had tests behind it. If you stopped there, a tired human reviewer probably signs off.

That would have been a mistake.

## Round 0: The Fixes That Looked Finished

The engineer worked through the findings list in the usual way. Suspended users were blocked at the database layer. The bulk verification flow moved from check-then-deduct to pre-debit plus refund. Hardcoded secret fallbacks were removed and replaced with fail-fast env checks. Upload handlers stopped trusting client-reported file size and switched to bounded reads.

The SSRF fix also looked clean on paper. FixBounce verifies email domains by resolving mail servers and opening SMTP connections to them. The first-pass patch added a public-IP filter before any outbound connect. It rejected private, loopback, link-local, multicast, reserved, and unspecified ranges using Python's `ipaddress` module.

It passed its tests.

That still wasn't enough to ship.

## Round 1: Gemini Catches The Bugs Behind The Fixes

Gemini reviewed the full diff with context from the original findings and the surrounding code paths.

It found two problems that mattered because they were hiding behind fixes that looked correct.

The first was in the account suspension patch.

The initial change tightened the user lookup so disabled accounts were filtered out early. That closes the obvious hole. But it also changed behavior somewhere else in the product: an existing branch that was supposed to return a specific "your account is suspended" response became permanently unreachable.

Now a suspended user with a still-valid token would hit a generic unauthorized path instead of the intended suspended-account path.

That is exactly the kind of bug that disappears in code review because the patched query looks obviously safer. Gemini caught it because it read the caller, not just the patch. The final design became asymmetric on purpose: strict where fresh authentication happens, lenient enough where session validation needs to preserve the suspension-specific response.

The second catch was worse.

The SSRF filter was bypassable through IPv6 tunnel forms.

The first patch checked whether a resolved address looked globally routable. That works for ordinary IPv4 and IPv6 literals. It does not work if the address is an IPv6 wrapper carrying an embedded IPv4 target inside it. In those cases, Python evaluates the outer IPv6 object unless you explicitly unwrap the embedded address first.

So the fix blocked the obvious private targets, but a cloud metadata endpoint wrapped inside an IPv6-mapped or transition address could still pass the "public" test and slip through.

That bug had already survived implementation and the tests behind it.

Gemini flagged it immediately. The patch was rewritten to unwrap IPv4-mapped, 6to4, and Teredo-style tunnel forms before evaluating whether the destination was public. After that rewrite, fourteen test cases passed, including the exact tunneled cloud-metadata vector that the first version missed.

This would have shipped without a second brain.

## The Important Part: The Council Wasn't A Yes-Man

The value here wasn't just "more review."

It was adversarial review.

Gemini also pushed on two points the engineer did not accept.

One was the DNS rebinding concern on the MX lookup path. Gemini pushed harder on that residual risk. The engineer kept it as a documented Phase 2 issue rather than a release blocker for this patch set, because fixing it properly means pinning resolved IPs through the later connection path instead of pretending one more filter closes the window.

The other was the boundary around fail-fast-at-import-time checks for shared-module environment variables. Gemini pressed on the startup behavior. The engineer held the line on the chosen boundary for this deploy, with production environment injection handled deliberately before restart.

This matters because "multi-model" only works if disagreement is allowed to stay alive long enough to be tested.

Codex then reviewed the disputed points independently and landed on the engineer's side for both. Round 2 ended with dual SHIP verdict.

That is the part people miss when they hear "council." The goal isn't to pile on agreement. The goal is to create a structure where one reviewer can catch what the fixer missed, another reviewer can challenge the reviewer, and a third perspective can arbitrate without being anchored to either side.

## Round 2: Ship

By the end of the session, FixBounce had shipped the blocker and high-severity code fixes in one coordinated deploy.

The concrete shape of the work was small and surgical: four files touched, seven fixes shipped across the blocker/high set, and the SSRF filter backed by fourteen tunnel-aware test cases covering IPv4-mapped, 6to4, and Teredo forms that the first version missed.

Production deploy happened the same day. No incident.

That outcome matters because this wasn't a lab exercise. These were live security patches on a public SaaS with billing logic, authentication, and outbound network behavior in the blast radius.

## What This Actually Proves

Single-model review has a blind spot that is easy to underestimate.

A model that writes a patch is usually anchored to the vulnerability it is trying to close. It checks, "Did I block the thing?" It is much worse at asking, "What did this change quietly break one layer up?" or "What weird representation still passes my validation logic even though the ordinary form does not?"

That is why the two best catches in this session were both second-order problems.

Not missed bugs in old code.

Bugs introduced or preserved by correct-looking remediation work.

This is the MegaLens thesis in miniature. One brain can produce a plausible answer. A second brain catches the non-obvious mistake. A third brain resolves whether the second brain is actually right, or just confidently overreaching.

Also, this is not a replacement for static analysis, dependency scanning, or human security review. Those tools catch different classes of failure. The lesson here is narrower and more useful: independent council review is unusually good at catching fix-on-fix failures, dead-code security controls, and edge-case bypasses that survive both implementation and conventional review.

In this case, that difference was the gap between "patched" and "safe to ship."


Weighted dissent is only useful if it changes the code before production. Here, it did.
