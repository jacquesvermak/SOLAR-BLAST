# Solar Blast — 60 seconds, one solar cannon, blast everything

A kid grabs the tablet, taps to fire a giant solar cannon, and has 60
seconds to blast broken panels, energy monsters and electricity bills out
of the sky before the clock runs out and the whole screen detonates in one
final **Solar Blast**. Built as the fun, no-punishment counterpart to
[Don't Go Dark](https://github.com/jacquesvermak/Sunny-s-Power-Dash) — that
game is a serious survival sim about managing a battery under pressure;
this one is a pure arcade shooter, every hit scores points, there's no
wrong target.

Plain HTML + CSS (Canvas) + JS, no build step, no external dependencies.
One file.

## Run

```sh
python3 -m http.server 8000
```

Open <http://localhost:8000>. Or build and run the exact image the cluster
runs:

```sh
docker build -t solarblast:test .
docker run --rm -p 8080:8080 solarblast:test    # http://localhost:8080
```

## The loop

1. **Tap Play.** A solar cannon sits at the bottom of the screen; the clock
   starts at 60 seconds.
2. **Tap anywhere to fire.** The cannon aims at your tap and fires a bolt
   from its barrel. Anything the bolt touches is destroyed — there's no
   wrong target, every hit scores points.
3. **Things drift across the screen from every direction:** 🪟 broken
   panels (big, slow, easy), 👾 energy monsters (zigzag), 💸 electricity
   bills (fast, worth the most among targets), ☁️ clouds, and ⚡ power
   thieves (small, fast, hardest to hit). Let one drift off unhit and it
   costs 15 points and resets your streak — the one small stick in an
   otherwise all-carrot game, there purely to keep the pace up rather than
   letting a player go idle.
4. **☀️ Solar, 🔋 batteries and 💰 energy coins** drift in too — no penalty
   for missing these, only targets cost you. Solar and coins are bonus
   points; a battery grants a few seconds of rapid-fire.
5. **Chain hits without missing for a streak bonus** — the longer the
   streak, the more each hit is worth (capped so it stays readable, not a
   runaway number), with a big "NICE! / ON FIRE! / LEGENDARY!" callout
   every 5 hits in a row. Any target or collectible can also spawn
   **🌟 golden** (a small random chance) worth triple points, with its own
   jackpot particle burst — a pure-upside surprise, never a trap. Bigger
   hits and the finale kick in a bit of screen shake for impact.
6. **Three stages, all inside one 60-second clock:** easy and slow for the
   first 20 seconds, faster with more targets for the next 20, and
   **🚨 MEGA POWER MODE** for the final 20 — the screen fills with targets
   moving fast.
7. **☀️ Solar Blast finale.** The instant the clock hits zero, every target
   still on screen auto-detonates in a rapid escalating sequence —
   +500, +1,000, +2,000 — for one last dramatic burst before the results
   screen.
8. **Results, always shown.** Score, star rating, a breakdown (targets /
   collectibles / streak bonus / finale bonus), a leaderboard name field, a
   "New Best" badge and device percentile if warranted. Then Play Again,
   Share, Home, and — separately — "See My Solar Options," which reveals a
   lead form. Nothing about seeing your own result is gated behind giving
   up your details.

### Difficulty curve

Tied to time *remaining*, not elapsed, so it reads naturally as "it's
getting harder" while the clock counts down:

| Time left | Spawn gap | Speed | Max on screen |
| --- | --- | --- | --- |
| 60s → 40s ("easy") | ~0.85s | 0.75× | 4 |
| 40s → 20s ("fast") | ~0.55s | 1.1× | 6 |
| 20s → 0s ("crazy" / Mega Power Mode) | ~0.3s | 1.6× | 10 |

### A real shared leaderboard, but device-local leads

The top-10 board shown on the leaderboard screen and the results screen is
**real and shared across every device** — backed by a small API
([`leaderboard-api/`](https://github.com/jacquesvermak/Sunny-s-Power-Dash/tree/main/leaderboard-api)
in the Don't Go Dark repo, code-shared but *not* data-shared: this game
runs its own dedicated instance with its own SQLite database, entirely
separate from Don't Go Dark's — see that README for why) rather than this
device's own history. A name-and-score-only board carries none of the PII
risk that rules out a shared board for *leads* — that reasoning is unchanged, and
leads (name, email, phone) still accumulate only in this device's
`localStorage`, exactly the real sales-booth shape: one tablet, one rep,
walking a queue of kids (and the adults with them) over an afternoon. The
gear icon opens a **Sales Tools** panel that exports every captured lead
as a CSV.

The home screen's "High Score" and "Best Blasted" stats stay per-device on
purpose — a personal best to beat on *this* tablet, distinct from the
shared top 10. A submitted score is trusted at face value (see the API's
own README for why that's an accepted limitation, not an oversight), and
a network hiccup degrades to a friendly "couldn't load" message rather
than breaking the results screen — your score, rating and breakdown are
already computed locally by the time the leaderboard fetch happens.

## Architecture

```
                    index.html
                        │
        ┌───────────────┴───────────────┐
        ▼                               ▼
   inline <style>                  inline <script>
   (Stage Zero tokens, system      (spawn/collision/scoring loop,
   font stack, sheet chrome        finale sequencer, leaderboard/
   shared with Don't Go Dark)      lead state)
```

No React, no bundler, no build step — the whole game is one HTML file with
inline CSS and JS, same as `sz-zero-hour-game` and `sz-sunnydash-game`.
The sheet/HUD/leaderboard/lead-capture chrome is deliberately copied from
Don't Go Dark rather than reinvented — it's a proven, already-refined
pattern, and sharing it keeps the whole Stage Zero game portfolio looking
like one family instead of N unrelated skins.

## Deploying to QA

Push to `main` → GitHub Action builds the image → Harbor → the action rewrites
the image tag in `infrastructure/apps/sz-solar-blast-game/values.yaml` → ArgoCD
syncs. Lands at **https://solarblast.qa.stagezero.co.za**.

Same pattern as `sz-sunnydash-game`: the chart keeps its config in the default
`values.yaml` rather than an `env/qa/` overlay, so Helm — and therefore
ArgoCD — reads it with no `helm.valueFiles` setting on the Application. CI
rewrites the same file ArgoCD renders, which removes any way for the two to
disagree about the image tag.

### One-time setup

| What | Where |
| --- | --- |
| `HARBOR_URL` variable | GitHub repo → Settings → Variables (optional — defaults to `harbor.qa.stagezero.co.za`) |
| `HARBOR_CA_CERT`, `HARBOR_USERNAME`, `HARBOR_PASSWORD` | GitHub repo → Settings → Secrets |
| ArgoCD `Application` → `path: infrastructure/apps/sz-solar-blast-game` (no values file needed) | ArgoCD, `argocd` namespace |
| A per-repo git credential Secret (`argocd.argoproj.io/secret-type: repository`) for this exact repo URL | `argocd` namespace — this cluster has no wildcard/prefix credential, one Secret per repo |
| `qa-cert` TLS secret present in the `sz-solar-blast-game` namespace | cluster (auto-replicated on namespace creation) |
| Harbor pull credentials for a fresh namespace — set `imagePullSecrets` in `values.yaml` if needed | cluster |

## Files

- `index.html` — the whole game: tokens, canvas cannon/spawn/collision
  engine, scoring breakdown, HUD, home/game-over/leaderboard/info/sales
  sheets, lead-capture flow, all inline
- `Dockerfile` · `nginx.conf` — static image, non-root nginx on 8080, `/healthz`
- `infrastructure/apps/sz-solar-blast-game/` — Helm chart deployed by ArgoCD
- `.github/workflows/deploy.yml` — build → push to Harbor → rewrite manifest → ArgoCD sync

## Known limits of this build

- **Leads are per-device; the leaderboard isn't.** The top 10 is a real
  shared board now (see above); leads (name/email/phone) still only live
  in this device's `localStorage` — a deliberate trade-off for PII, not an
  oversight.
- **A submitted score isn't verified.** The leaderboard API trusts a
  client-reported score at face value beyond a sanity ceiling — see
  `leaderboard-api/README.md` for why that's an accepted limitation for a
  booth activation, not something to silently "fix" without deciding a
  server-authoritative game engine is worth the complexity.
- **"Share" copies text, it doesn't post anywhere.** `Web Share` is used
  where the browser supports it, otherwise it falls back to clipboard copy.
- **Lead export is a manual CSV download, not a live CRM push.**
- **Emoji sprites, no custom art yet.** Every target/collectible is a big
  emoji rather than a drawn character — fast to ship, easy to reskin later
  with real art the same way Don't Go Dark added real photos on top of its
  Canvas fallback, if that's ever worth doing here.
- **Single language, no i18n.** English only.
- **No analytics.** Nothing is instrumented yet.
