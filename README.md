# Equilibrist: GLEE Competition agent

A rule-based agent, with no LLM in its decision loop, for the
[GLEE Competition](https://glee-competition.com) (NeurIPS 2026 IAB workshop). It
entered on day 25 of 29 and finished **17th of 152 ranked agent accounts** over
70,449 games. The competition closed on August 29, 2026, and the agent no longer runs.

The accompanying paper and poster are in `paper/` and `poster/`.

## Run

The agent needs a `GLEE_API_KEY` in `.env` (never committed).

```bash
./run.sh          # starts agent.py in the background, logs to agent.log
tail -f agent.log
```

Run a single copy per API key. Several copies share the key's 60 requests/minute and
their duplicate submissions count against a game's attempt budget, which the platform
records as invalid-move forfeits. Stop it by PID, or with `pkill -fi 'python agent.py'`
(the process name is capitalized on macOS, so a case-sensitive match finds nothing);
confirm with `pgrep -fil agent.py`. The SDK leaves the matchmaking queue on exit.

During the competition the agent ran as an always-on background worker on a hosting
platform, using `install.sh` (installs the SDK at image build) and `launch.sh` (starts
the worker); `agent.py` touches `$XHOST_READY_FILE` once the key validates so a
no-HTTP health check passes. That deployment was removed after the close.

The run loop (`concurrency=12`, `poll_interval=3.0`) clears the competition's activity
rules (100 games per family per 48 hours above a raw rating of 1800, and 10 games per
day per family in the top 100). Polling costs one request per cycle regardless of
games in flight, so the request budget is the throughput cap: poll rarely, play many
games at once.

## Strategy

Each game is scored as the **payoff percentile within its exact configuration and
role**, against a pool seeded by the public GLEE dataset and adjusted for opponent
strength. The agent therefore targets the pool's payoff distribution, not the
equilibrium payoff. `pool_quantiles.json` holds per-configuration payoff quantiles
computed from the 77,467 published games; `pool.py` matches a live game to it.

- **Bargaining**: open at the share worth the 90th-percentile payoff and concede one
  quantile per rejection. Accept when the discounted offer clears a declining ladder
  (q75 in rounds 1-2 down to q10 after round 14), capped by 0.8 times an equilibrium
  continuation value and floored by q25 (rounds up to 6) or q10 (rounds up to 14). The
  last round accepts any positive offer.
- **Negotiation**: aspiration levels use quantiles conditioned on a positive outcome.
  When the opponent's valuation range is stated, the price is blended with the
  take-it-or-leave-it optimum `(hi + V)/2` against a uniform buyer. Never accept a loss.
- **Persuasion**: the seller always recommends when the prior alone justifies buying
  or the buyer has shown it ignores messages, and otherwise signals honestly with a
  defection window (`rounds_left * p < 1.5`). The buyer treats quality as iid with a
  known `p`, so outcomes never update the prior; only a seller who sometimes says no
  is judged on the track record of its "yes".
- **Safety net**: moves are filtered against `valid_actions.fields`, splits sum
  exactly, a per-decision retry memory degrades to a guaranteed-valid move, and a
  catch-all fallback covers the rest.

## Test

```bash
.venv/bin/python test_agent.py   # 34 offline branch tests, no network
```

## Iterating

Up to 5 agents per account; only the best one ranks the account. To A/B a variant,
create a second agent in the dashboard and run a copy with its key. Post-game
percentiles (shown in the dashboard history) are the tuning signal: look for
configurations where the percentile is consistently low and adjust that branch.

`analyze.py` pulls every completed game's transcript through the API (cached in
`games/`, throttled to respect the rate limit the live agent shares) and prints
per-family/role deal rates, surplus shares, persuasion round-by-round detail, and
all no-deal games:

```bash
set -a; source .env; set +a; .venv/bin/python analyze.py
```

## Paper and poster

`paper/` holds the IAB (NeurIPS 2026) competition paper, *Targeting the Percentile, Not
the Equilibrium*, as LaTeX plus the compiled PDF and figures. Build with
`cd paper && tectonic -X compile paper.tex` (the NeurIPS 2026 style file is included).

`poster/` holds the workshop poster (portrait, 61 cm by 85 cm) as a single LaTeX file
plus the compiled PDF. Build with `cd poster && tectonic -X compile poster.tex`. Its
chart is drawn from the numbers in `analysis/results/matched_summary.md`.

## Reproducing the analysis

The competition scores each game as a payoff percentile within its configuration and
role, seeded by the public GLEE dataset. `analysis/` rebuilds that reference pool and
scores this agent's own games against it, per code era:

```bash
git clone --filter=blob:none --depth 1 https://github.com/eilamshapira/GLEE.git /tmp/GLEE
python analysis/build_pool.py /tmp/GLEE analysis/pool.pkl        # 77,467 dataset games
python analysis/history_eras.py analysis/history.jsonl           # official rating deltas by era
python analysis/matched_comparison.py                             # configuration-matched percentiles
```

`analysis/history.jsonl` is the agent's full game history exported from the platform
dashboard (one JSON object per game); transcripts for scoring are fetched into `games/`
with `analysis/fetch_era_transcripts.py`. Computed outputs used in the paper are checked
in under `analysis/results/`.
