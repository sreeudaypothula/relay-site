# Indrashikha Media portal: review and mobile app proposal

Reviewed 25 Nov 2026 on media.indrashikha.com, Techwave Academy project, signed in with the account provided. I only looked: I opened every screen, read the front-end code the browser downloads, and watched the API calls. I did not create, edit, publish or delete anything. Server-side points are inferred from what the browser can see, so they are marked "to verify".

## 1. What the product is

An agency portal that runs a client's social media end to end. Each client or brand is a project, and one project holds the whole loop:

| Stage | Screens | What it does |
|---|---|---|
| Assess | Strategy › Assessment | Scores social maturity on 8 weighted dimensions (Techwave is 22.5, "Emerging"), plus questionnaire, website audit and findings |
| Plan | Strategy › Plan, Brand kit | A 15-section versioned plan with approval, AI redraft per section, brand colours, voice, claims, targets, competitors |
| Produce | Content › Posts, Calendar, composer | Posts with per-platform captions, media, tracked links, automatic publish checks, AI caption drafts |
| Approve | Content › Approvals | Internal review, and a client review link that needs no account |
| Publish | Channels, Post by hand, Claude | API publishing where a platform allows it; otherwise a "copy and open" manual queue, or a Claude job |
| Advertise | Ads › Campaigns, Budget, Creators | Campaigns logged from Ads Manager, kill and target cost rules, budget forecast, creator pipeline |
| Measure | Results › Analytics, Live, Links, Weekly report | Followers per channel, tracked short links with UTM tags, sign-up attribution, a one-page client report |
| Improve | Results › Recommendations, Home › Inbox, Tasks | Rule-based recommendations, one inbox of everything that needs a person, tasks with automatic checks |

The distinctive part is the **Claude hand-off**: any task, recommendation or failed post can be sent to Claude Code running on an office Mac. It connects through an MCP server, claims the job, reports progress, asks for approval before anything public or paid, and returns proof (links, numbers). Three freedom levels: "Just do it", "Ask me first", "Only check".

## 2. How it is built (observed)

- **Front end:** hand-written JavaScript, no framework and no build step. 10 ES modules (about 245 KB unminified), a hash router (`#/w/<project>/<view>`), views rendered with `innerHTML`, one 43 KB stylesheet.
- **Hosting:** Cloudflare (HTTP/3, Hyderabad edge). Text generation is Workers AI, `llama-3.3-70b`. The API is very likely a Cloudflare Worker with a D1 database (to verify).
- **Auth:** session cookie that JavaScript cannot read, a custom header on API calls, eight roles (owner, admin, strategist, creator, ads, analyst, client, viewer), an audit log.
- **API:** about 60 REST endpoints under `/api`, most scoped as `/api/ws/<project>/...`.

The full endpoint map is in the private copy of this review.

## 3. What is done well

- Clear information architecture: six sections, each with a few tabs, and one inbox for everything that needs a person.
- Every piece of work says who does it (You, Claude, Claude with your OK) and finished work carries proof.
- A person approves before anything public, paid or irreversible. This is the right design for agents.
- Careful details: Cmd+K search, undo on destructive actions, "leave without saving?" guard, India time everywhere, focus-trapped dialogs, a phone layout with a bottom bar, print-ready report, WhatsApp summary copy.
- Good security basics: no framing, no MIME sniffing, cookie hidden from scripts, not indexed by search engines, revocable runner keys shown once.

## 4. Problems found

### Bugs
1. **The Claude page does not open.** The sidebar item "Claude" (badge 8), the top pill "Claude: 1 needs your OK" and "All jobs" all show Home. Cause: `app.js` never imports or registers the `claude` view from `claude.js`, so the router falls back to Home.
2. **Duplicates.** Two identical "LinkedIn failed" cards on Home; the same Telegram post published three times at 21:39 on 20 Nov; the same stuck YouTube job listed twice. Publishing and job creation need idempotency keys.
3. **Numbers disagree between screens.** Home says 0 sign-ups from social and "tracking not verified". Analytics says 212 sign-ups and "tracking verified 29 Oct". The weekly report says 36 sign-ups. Home shows followers up 69 on last week while the channel table adds up to about +7.
4. **Placeholder posts reach publishing.** "linkedin post (to write)" was scheduled, failed and now sits in the manual queue twice.
5. **Stale manual queue.** "10 items due in the next 48 hours" lists items from 31 Oct onward. They are weeks overdue and need an overdue state or expiry.
6. **Budget screen text.** The conversion name is pluralised into "sign-up (OTP-verified, utm_campaign=tw-launch-nov26)s" in headers and cards, and the page scrolls sideways on a phone.
7. **Composer.** Counter reads "FB 35 / 63,206" on an empty caption, preview shows the placeholder link `tw-pN-ig`, and there is a stray dot after the status.
8. **Home layout.** A long Instagram URL overflows the Claude card, and messages are cut mid-word with no ellipsis.

### Operations
9. **Nothing publishes by itself.** Channels shows "6 live, 6 tracked, 0 post by themselves", although Telegram, Facebook and Instagram have APIs. 0 posts went out in the last 7 days against a plan of about 18 a week.
10. **One root cause, many alerts.** Every YouTube job is stuck because the office Mac is signed in to the wrong Google account. The portal raises it once per post. It should raise one channel-level blocker and pause the dependent jobs.
11. **The runner is a single point of failure.** One Mac, browser logins, a 15-minute pick-up. If it is off or signed out, publishing stops.

### Performance
12. **Inbox takes 2.0 s on every call** and Home waits for it, so Home shows a bare "Loading…" for over two seconds.
13. **Every API call costs 250 to 600 ms**, and a 404 costs about 120 to 210 ms, so the time is in the database round trips.
14. **"Sync all" took 10.9 s** and is triggered by opening the Live screen.
15. **No pagination.** `/posts` returns all 67 posts with full captions (65 KB) and ignores `?limit=5`. `/claude/jobs` returns 143 KB of full briefs and is fetched on every Home visit.
16. **No HTTP caching.** Every response is `no-store` with no ETag. The 10 script modules are not versioned and are revalidated on every load, which also risks mixing old and new files after a deploy.

### Security
I noted a small number of security observations from the browser. They are shared privately with the Indrashikha team and are not included in this public copy.

### Accessibility
21. Lighthouse accessibility 93. Failures: search text contrast 4.47:1, heading order, search button name mismatch. 143 form fields have no id or name and 8 have no label. On the Tasks screen at phone width, 19 of 40 tap targets are under 40 px.

### Experience
22. **Home repeats itself.** The same items appear in the top cards, "Also on the list" and the Claude panel. "32 things to do" has no priority and no grouping by cause.
23. **Jargon.** "L1 recommend", "partial", "kill line", "Maturity 22.5", and platform codes such as "in", "@", "GA" need plain labels.
24. **Internals leak into messages.** Users see "~/.chrome-youtube is signed in to...". Show "YouTube is signed in to the wrong Google account. Reconnect" and keep the detail behind "Technical details".
25. **Recommendations are noisy.** Five near-identical "N more posts needed" cards, one per channel, should be one card: "18 posts short this week, create all draft slots".
26. **Empty calendar.** The week is empty while the plan expects 18 posts. Show the plan's rhythm as ghost slots to fill.
27. **"Posts" and "All posts"** sit side by side with no clear difference. Rename the second to "Published results" or merge it.

28. **Ads run under three different limits.** Campaigns stop above ₹40 a sign-up with a ₹20 target. The plan's targets say stop above ₹60 with a ₹30 target. The budget forecast expects ₹142 to ₹938 a sign-up. As set, every ad would be stopped. Agree one limit before the 26 November launch.
29. **Posts in the last 7 days read 0 on Home and 24 on All posts.** Home counts portal posts; All posts also counts the Telegram script's posts. Same label, different sources.
30. **Launch readiness is not visible.** The two launch campaigns are due to go live on 26 November while the ad account and budget approval task is 5 days overdue and the ads are not yet built. Nothing on the Ads screen says so.

## 5. What I would change, in order

**Week 1: trust**
- Register the Claude view. Add a Playwright smoke test that opens every route, which would have caught it.
- One definition for sign-ups, followers and "tracking verified", computed in one place and used by Home, Analytics and the report.
- Idempotency keys on publish and job creation; block publishing of placeholder or empty posts.
- Collapse alerts by root cause into a channel health status.

**Weeks 2 to 4: speed and automation**
- Make the inbox a precomputed summary; run its queries in parallel; show skeletons, not "Loading…".
- Cursor pagination and slim list responses (no captions or briefs in lists); one `/home` call replacing three.
- Turn on real API publishing for Telegram, Facebook and Instagram through a queue with retries, backoff and a dead-letter list. Keep the Claude runner for platforms with no API.
- Run "sync all" on a schedule in the background, never on page open.
- Live job status over server-sent events in place of polling.

**Later: foundation**
- TypeScript and a Vite build with hashed file names; a small component layer with safe templating in place of raw `innerHTML`.
- An OpenAPI spec with a generated typed client, shared by web and mobile.
- Error tracking and API timing dashboards.
- An evaluation set for AI captions (brand voice, approved claims, banned phrases) and a duplicate check for AI news posts.
- Move the runner from one office Mac to a managed worker with a per-client credential vault.

## 6. Mobile app proposal

**Why an app:** the portal is desktop-shaped, but its most frequent jobs are phone jobs. Approving a post, posting by hand to WhatsApp or YouTube, answering "Claude needs your OK", and checking today's numbers each take seconds and are driven by a notification.

**Who:** the operator (daily), the owner (approvals and Claude), the client (approve and read the report).

### Screens

| # | Screen | Main actions |
|---|---|---|
| 1 | Sign in | Email and password, then Face ID or fingerprint unlock |
| 2 | Projects | Switch client; counts of items waiting |
| 3 | Today | Up to three "needs you" cards, four numbers, what goes out next |
| 4 | Inbox | Filter chips (Problems, Claude, Approvals, Post by hand, Tasks); swipe to act |
| 5 | Approvals | Post preview per platform; approve, or reject with a note; client mode |
| 6 | Post by hand | Copy caption and open the target app through the share sheet; paste the live URL; mark posted |
| 7 | Calendar | Week agenda; tap for a quick look; drag to reschedule |
| 8 | Composer | Platforms, caption with AI draft, camera or gallery media, publish checks, schedule |
| 9 | Claude | Jobs needing OK, running, stuck; see what it will do; approve, retry, cancel |
| 10 | Results | Followers by channel, best posts, weekly report with "share to WhatsApp" |
| 11 | Ads | Campaign cards, log today's results, pause |
| 12 | Tasks | Mine, due, done; mark done with proof |
| 13 | Account | Notification choices, sign out |

Kept on the web: plan editor, brand kit, assessment, budget forecast, channel setup, members.

### Design
- Bottom tabs: Today, Content, a centre "+" for a new post, Results, More.
- One primary action per card, thumb-reachable; swipe right to approve, left to reject.
- Keep the existing identity: navy, cream and gold, serif headings.
- Skeleton loading, pull to refresh, and an offline banner with queued actions.
- Push notifications that deep-link to the exact item: "Approve: Excel reel for Instagram".

### Build
- **React Native with Expo and TypeScript**, one codebase for Android and iOS. Android first, since the team and clients are in India.
- TanStack Query for server data with a persisted cache; an offline queue for approve, mark posted and task done.
- Expo Notifications (FCM and APNs), SecureStore for tokens, the share sheet and deep links for posting by hand.
- EAS for builds and over-the-air updates; Sentry for crashes; Maestro for end-to-end tests.

### API work the app needs
- Token sign-in with a short-lived access token and a refresh token (cookies suit browsers, not apps).
- A versioned API (`/api/v1`) described in OpenAPI.
- Pagination and slim lists; one `/home` summary call.
- Idempotency keys on every write, so a retried offline action cannot publish twice.
- `POST /devices` to register push tokens, and push events for: needs approval, Claude needs OK, publish failed, job stuck.
- Direct media upload to object storage with signed URLs, resumable for video.

### Delivery
| Phase | Weeks | Scope |
|---|---|---|
| 0 | 1 | API groundwork: tokens, pagination, idempotency, push registration |
| 1 | 2 to 4 | Sign in, Projects, Today, Inbox, Approvals, Post by hand, push |
| 2 | 5 to 7 | Claude jobs, Calendar, Composer with media and AI draft |
| 3 | 8 to 9 | Results, Ads, Tasks, client mode, weekly report sharing |
| 4 | 10 | Offline queue hardening, accessibility pass, store release |

**Success measures:** median time from "submitted" to "approved"; share of manual-queue items posted on time; crash-free sessions above 99.5%; Today screen usable in under 1.5 s on a mid-range Android phone.
