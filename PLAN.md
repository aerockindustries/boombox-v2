# Boombox v2 — build plan

**What this is.** A clean rewrite of [`aerockindustries/project-boombox`](https://github.com/aerockindustries/project-boombox)
with a real foundation under it. v1 proved the loop works — QR → join → request → host approves →
picker plays → "who picked this?" → reveal. It is deployed and it demos. It is also ~1,300 lines
built in one day, and four of its load-bearing assumptions are wrong in ways you cannot patch
incrementally. Rewriting 1,300 lines is cheap. Living with the wrong identity model is not.

**Carry over unchanged:** `CLAUDE.md` (product judgment), `CONTEXT.md` (vocabulary),
`convex/picker.ts` + its tests (the one genuinely good piece of v1), the visual direction, and
the Playwright room-loop e2e as the acceptance test for parity.

**Keep v1 alive.** Do not delete or stop deploying `project-boombox`. It stays the thing you can
demo to a venue while v2 is under construction. v2 goes live only when it passes the parity gate
in Phase 2. No data migration — v1 rooms are throwaway.

---

## Why rewrite: the four things that are actually wrong

These are the reasons. If a change doesn't serve one of them, it isn't part of the rewrite.

### 1. Identity is a client-supplied string

Every v1 mutation takes `guestId: v.string()` as an argument and trusts it. `nowPlaying.guess`,
`requests.submit`, `requests.react`, `guests.profile` — all of them. Anyone with the browser
console can submit songs as another guest, guess as them, react as them, or read their profile.
The host side is a shared secret (`hostSecret`) passed as a mutation arg and parked in
`localStorage` from a `?secret=` URL; whoever sees that URL over someone's shoulder owns the
venue's music.

This is the single largest defect class in the codebase and it is unfixable by patching, because
every function signature encodes it. v2 establishes identity from `ctx.auth`, and `guestId` never
appears in an args validator again.

### 2. A room is a venue is a session

v1's `rooms` table is all three at once. Consequences: a venue cannot close and reopen, there is
no history across nights, there is no such thing as "this café" independent of "this café's
Tuesday", a host cannot have two spaces, and no staff member other than the secret-holder exists.
Every host-side feature you will want next — activity over time, repeat participation, recurring
QR codes that don't expire, an employee who isn't the owner — is blocked on splitting these apart.

### 3. There is no play log, so history is a lie

`nowPlaying` holds exactly one mutable row per room. `playNext` deletes the old one; `skip`
deletes it. Nothing records what actually played. The picker's fairness rules read "recently
played" from `requests` ordered by **creation time**, not play time — so on any room where
requests arrive out of order relative to plays, the artist cooldown and guest-fairness windows
operate on the wrong list. The fairness logic is correct; the data it's fed is not.

v2 makes plays an append-only log. "Now playing" is derived (the play with no `endedAt`), not
stored. Fairness, history, and every metric in `CLAUDE.md`'s core-metrics list fall out of it for
free.

### 4. Nothing knows whether the music is actually playing

`playNext` fires a Spotify call and forgets. Reveal is `scheduler.runAfter(30s)` regardless of
whether the track started, and nothing detects the track ending — so **a human has to press "Play
next" for every song, forever.** For a climbing gym that is a person standing at a tablet all
evening. It is the difference between a demo and something a venue will run.

v2 has a playback runtime: a poller that reads real provider state, advances the log when a track
ends, auto-picks the next one, and reconciles when the host's device dies.

Everything below serves these four. Nothing else is a reason to rewrite.

---

## Architecture

### Stack

Same shape as v1, deliberately — it was the right call and the team knows it.

| Layer | Choice | Change from v1 |
|---|---|---|
| Frontend | Next.js App Router, TypeScript, Tailwind, shadcn/ui | Drop `@base-ui/react`; pick one component system and stay in it |
| Backend | Convex | Same |
| Auth | **Convex Auth** — anonymous for guests, email/passkey for hosts | New. v1 had none |
| Catalog | Provider interface; iTunes impl (no key) + Spotify impl | Formalized behind an interface |
| Playback | Provider interface; manual + Spotify impls, licensed provider later | Real state sync, not fire-and-forget |
| Counters | `@convex-dev/aggregate` or denormalized counts | New. v1 counts by loading 500 rows |
| Rate limiting | `@convex-dev/rate-limiter` | New. v1 has none |
| Hosting | Vercel + Convex Cloud | Same |

### Data model

Written together, then frozen — the one v1 rule that worked, keep it.

```
venues          the business. persistent. name, slug, settings, timezone, createdBy
venueMembers    staff ↔ venue, role: owner | staff.  real accounts, not a shared secret
sessions        one live period at a venue. code, openedAt, closedAt|null, settings snapshot
guests          a person in a session. subject (from ctx.auth), nickname, avatar, taste
requests        sessionId, subject, trackRef, status, createdAt
plays           APPEND-ONLY. sessionId, requestId|null, trackRef, startedAt, endedAt|null,
                source: pick|host|venue, revealedAt|null, endedReason: finished|skipped|lost
guesses         playId, guesser subject, guessed subject, correct
reactions       playId, subject          (unique index on [playId, subject])
blocks          venueId (persistent) OR sessionId (temporary), kind, value
tracks          normalized catalog cache: provider, providerId, title, artist, artwork,
                previewUrl, durationMs, EXPLICIT
```

Four things to notice:

- **No `guestId` string as an identity.** `subject` comes from `ctx.auth.getUserIdentity()`.
- **`nowPlaying` is gone.** It's `plays` where `endedAt === null`.
- **`tracks.explicit` exists.** v1 never stored it, which means v1 cannot honor the single most
  common thing a café or a family-hours climbing gym will ask for. Venue setting: block explicit.
- **Blocks live at the venue level** and persist across sessions. A host who blocked an artist on
  Tuesday should not have to block them again on Wednesday.

### Authorization rules (non-negotiable)

1. No function takes a caller identity as an argument. Ever. Identity is `ctx.auth`.
2. Every host mutation resolves `venueMembers` for the caller and the target venue.
3. Every guest mutation resolves the guest's own session membership; writes are scoped to it.
4. Reads that expose another person's data (profiles, requester identity) check reveal state
   server-side. Nothing pre-reveal ships to a client that shouldn't see it. v1 got this one
   right in `nowPlaying.get` — keep the pattern, apply it everywhere.
5. Run `/convex-authz` before every phase gate.

### Module boundaries

`CLAUDE.md` names the boundaries that must stay replaceable. In v2 they are directories:

```
providers/catalog/     MusicCatalog:    search(q), getTrack(ref)  → normalized Track w/ explicit
providers/playback/    PlaybackController: play, pause, state(), position, deviceHealth
convex/selection/      the picker. pure. no ctx, no db. (carried over from v1, fed correct data)
convex/runtime/        the playback loop: poll state → close play → pick → start next
app/(guest)/  app/(host)/
```

The picker stays a pure function with unit tests. That was v1's best decision.

---

## Phases

Each phase has an exit criterion. Do not start the next one until it's met.

### Phase 0 — Foundations

Repo scaffold, Convex dev + prod, Convex Auth wired (anonymous + host accounts), schema written
and frozen, CI (`tsc`, `vitest`, `convex codegen`, license check), Vercel deploying `main`,
`convex-test` harness running. Port `CLAUDE.md`, `CONTEXT.md`, `picker.ts` + tests.

Do **not** commit the `.claude/` and `.agents/` skill mirrors that bloat v1 to 174 files.

**Exit:** a signed-in host and an anonymous guest can each call a trivial authed function in prod.

### Phase 1 — Venue and session

Venue creation, staff membership, session open/close, permanent venue QR that resolves to
whichever session is currently open (and to a friendly "nobody's playing right now" page when
none is). Host claim flow replaces `?secret=`.

**Exit:** one venue, two sessions on different days, both visible in history, one QR code that
worked for both.

### Phase 2 — Parity slice

Everything v1 does, on the new foundation: join, search, submit, pool, approve/reject/block,
play next, guess, reveal, reactions, print QR. Explicit-content policy enforced at submit time.
Rate limits on join and submit.

**Exit gate — this is the one that matters.** v1's `e2e/room.spec.ts` (host + two guests, full
loop) passes against v2 in prod, *and* `/convex-authz` finds nothing, *and* an authz test suite
proves guest A cannot act as guest B. When this passes, v2 becomes the demo and v1 is archived.

### Phase 3 — Playback runtime

The thing that makes it a product. Track-end detection, auto-advance, host "auto-DJ" toggle,
device-health surfacing ("your Spotify device went away — press play on your phone once"),
reveal timed off real playback position rather than a blind 30-second timer, graceful degradation
to manual when the provider is unavailable.

**Exit:** a two-hour session runs with the host touching the tablet only to reject something.

### Phase 4 — Host product

Venue dashboard: what played tonight, how many people joined, how many requested, how many
distinct people got a song played, repeat participation. Exactly the metrics `CLAUDE.md` lists —
no more. Staff roles. An onboarding flow a café manager completes without you on the phone.

**Exit:** someone who has never seen the product runs a session from the printed QR alone.

### Phase 5 — Pilot readiness

Licensing decision resolved (see Risks), abuse handling, observability on the core metrics,
load sanity at ~100 concurrent guests, an on-site runbook, and a rollback plan for a session
that goes wrong mid-evening.

**Exit:** a real venue runs a real night.

---

## Risks, honestly

**Licensing is the actual blocker, not code.** Playing music through a personal Spotify account
in a commercial venue violates Spotify's terms, and Spotify Development Mode caps you at 5
allow-listed testers — Extended Quota needs an organization and a review. Neither fact is fixable
by writing better software. The venue's existing licensed system (Soundtrack Your Brand, Rockbot,
their own subscription) is the realistic playback path, and `PlaybackController` exists precisely
so you can swap to it. **Decide this in Phase 0, not Phase 5** — it determines whether Phase 3
targets Spotify or a partner API, and building Phase 3 twice would be the most expensive mistake
available here.

**Rewrite drift.** The failure mode of a clean rewrite is never reaching parity because new ideas
keep landing. The Phase 2 gate is the defense: no feature that v1 doesn't already have ships
before that gate, with two exceptions — auth and explicit-content policy, which are foundations,
not features.

**Anonymous auth is still anonymous.** Convex Auth fixes *spoofing*; it does not stop someone
from clearing storage and rejoining as a new person to get around the one-request rule. That's
what rate limiting and per-session caps are for. Accept it as good enough for a venue where
everyone is physically in the room.

---

## Not building

Unchanged from `CLAUDE.md`, restated because a rewrite is exactly when scope creeps: no native
apps, no ML recommendations, no public feed, no venue map, no gamification, no billing, no
microservices. The picker stays deterministic rules until real session data says otherwise.
