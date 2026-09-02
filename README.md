# Solar Blast — 30 seconds, one solar cannon, blast everything

A kid grabs the tablet, taps to fire a giant solar cannon, and has 30
seconds to blast power blocks, energy monsters and electricity bills out
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
   starts at 30 seconds.
2. **Tap anywhere to fire.** The cannon aims at your tap and fires a bolt
   from its barrel. Anything the bolt touches is destroyed — there's no
   wrong target, every hit scores points.
3. **Things drift across the screen from every direction:** 🧱 power
   blocks (big, slow, easy), 👾 energy monsters (zigzag), 💸 electricity
   bills (fast, worth the most among targets), ☁️ clouds, and ⚡ power
   thieves (small, fast, hardest to hit). Miss one and it just drifts off
   — no penalty, this is a pure feel-good shooter.
4. **☀️ Solar, 🔋 batteries and 💰 energy coins** drift in too. Solar and
   coins are bonus points; a battery grants a few seconds of rapid-fire.
5. **Chain hits without missing for a streak bonus** — the longer the
   streak, the more each hit is worth (capped so it stays readable, not a
   runaway number).
6. **Three stages, all inside one 30-second clock:** easy and slow for the
   first 10 seconds, faster with more targets for the next 10, and
   **🚨 MEGA POWER MODE** for the final 10 — the screen fills with targets
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
| 30s → 20s ("easy") | ~0.85s | 0.75× | 4 |
| 20s → 10s ("fast") | ~0.55s | 1.1× | 6 |
| 10s → 0s ("crazy" / Mega Power Mode) | ~0.3s | 1.6× | 10 |

### Why device-local leaderboard and leads, not a shared public one

Same reasoning as Don't Go Dark: a shared/public leaderboard would put
every player's name, email and phone number somewhere anyone with the
link — including the players themselves — could read. Scores and leads
accumulate in this device's `localStorage` instead, which is exactly the
real sales-booth shape: one tablet, one rep, walking a queue of kids (and
the adults with them) over an afternoon. The gear icon opens a **Sales
Tools** panel that exports every captured lead as a CSV.

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

- **Leaderboard and leads are per-device.** `localStorage` doesn't sync
  across browsers or people — the deliberate trade-off, not an oversight
  (see "Why device-local" above).
- **"Share" copies text, it doesn't post anywhere.** `Web Share` is used
  where the browser supports it, otherwise it falls back to clipboard copy.
- **Lead export is a manual CSV download, not a live CRM push.**
- **Emoji sprites, no custom art yet.** Every target/collectible is a big
  emoji rather than a drawn character — fast to ship, easy to reskin later
  with real art the same way Don't Go Dark added real photos on top of its
  Canvas fallback, if that's ever worth doing here.
- **Single language, no i18n.** English only.
- **No analytics.** Nothing is instrumented yet.
