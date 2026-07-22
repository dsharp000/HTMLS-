# Razor's Edge — Security Notes & Firebase Hardening Runbook

## The problem (severity: critical)

The Realtime Database `lunch-f7021` is shared by 13 tools and has **no authentication
anywhere**. Root reads are denied, but every whitelisted path is world-readable and
(because the apps write without auth) world-writable. With nothing but the `databaseURL`
— which is visible in any page's source — an anonymous person can:

- **Read** every path, including `/calls` (private phone-call transcripts and AI
  summaries), `/family-loan` (loan financials), and `/soccer` + `/screen-time`
  (children's names).
- **Overwrite or delete** any path, including the live `/worldcup` bracket.
- Run up RTDB bandwidth/storage cost with no rate limit.

The Firebase API key in the page source is **not** the problem — that is the normal
client-key model. The problem is the open rules.

## Why this isn't fixed yet

1. **RTDB rules can only be changed in the Firebase console** (or via an admin token /
   Firebase CLI). Neither is available from the build environment, so the rules in
   `database.rules.json` are prepared but **must be published by hand.**
2. Tightening the rules **breaks any client that isn't authenticating.** The clients need
   an anonymous-auth change deployed first, and two paths (`worldcup`, `alaska-trip`) are
   live right now with users whose cached pages have no auth — those migrate last.

## The fix — staged runbook

Do the stages in order. Nothing here touches the live `/worldcup` bracket until the
final, post-tournament stage.

### Stage 0 — Enable Anonymous auth (console, ~1 min, breaks nothing)
Firebase console → **Authentication → Sign-in method → Anonymous → Enable**.
This alone changes no behavior; it just makes `signInAnonymously()` succeed.

### Stage 1 — Add anonymous auth to the clients — ✅ DONE (deployed 2026-07)
**Already applied and live** for all 11 low-stakes tools: soccer-tracker, call-recorder,
family-dinner, lunch-picker, family-loan, summer-planner, parks-checklist-audubon-pa,
national-parks-checklist, screen-time, bathroom-designer, kitchen-designer. Each carries
a non-blocking `signInAnonymously().catch(...)`. `worldcup-pool.html` and
`alaska-packer.html` are intentionally left for Stage 3.

Nothing else is required here. The snippet used (for reference / for the Stage 3 tools):

**Module tools** (`type="module"`, most tools — they already
`import { getDatabase } from ".../firebase-database.js"`):

```js
import { getAuth, signInAnonymously } from
  "https://www.gstatic.com/firebasejs/10.12.0/firebase-auth.js";
// ...after initializeApp(firebaseConfig):
const auth = getAuth(fbApp);
await signInAnonymously(auth).catch(e => console.warn("anon auth failed:", e));
// attach your onValue() listeners AFTER this line
```

**Compat tools** (`call-recorder.html` — uses `firebase-*-compat.js`): add the
`firebase-auth-compat.js` script tag, then:

```js
await firebase.auth().signInAnonymously().catch(e => console.warn("anon auth:", e));
```

Deploy, and confirm each tool still reads/writes normally (it will — rules are still open).

### Stage 2 — Publish the hardened rules (console)
Firebase console → **Realtime Database → Rules** → paste `database.rules.json` → Publish.
The moment this lands, every gated path requires an anon token. Because Stage 1 shipped
that token, the tools keep working; anonymous `curl` scraping stops.
**Test:** `curl https://lunch-f7021-default-rtdb.firebaseio.com/calls.json` should now
return `Permission denied`.

### Stage 3 — Migrate the two live paths (last)
- **`alaska-trip`**: once the travelers are home, apply Stage 1 to `alaska-packer.html`,
  deploy, then change its rule to `"auth != null"`.
- **`worldcup`**: **after the tournament ends**, apply Stage 1 to `worldcup-pool.html`,
  deploy, then change its rule to `"auth != null"`. Do not touch it before then.

## Honest limits of this fix

Anonymous auth stops trivial scraping and casual tampering, but any visitor can still mint
an anonymous token, so it does **not** restrict data to your family. For genuinely private
data (`/calls`, `/family-loan`) the durable fix is to not keep it in a public shared DB —
either move those two tools to an authenticated project with real sign-in, or stop
persisting raw transcripts server-side. Flagged for a follow-up decision.

## Other findings (lower severity)

- **call-recorder.html** calls the Anthropic API browser-direct with the user's own key
  from `localStorage` (`anthropic-dangerous-direct-browser-access`). Acceptable for a
  single-user tool, but any XSS on that page leaks the key. Also its model id
  `claude-sonnet-4-6` looks stale and likely 404s — verify/update.
- **Firebase SDK versions** drift across tools (10.12.0 / 10.12.2 / 10.14.1). Standardize.
