Supply chain under 20% message drop: one bad seed, and what it actually reveals
Part A — Scenario explored
`supply_chain`: a 4-agent linear pipeline (`supplier -> manufacturer -> distributor -> retailer`), 5 rounds of 3 items each (15 items total).
```
nest run supply_chain
nest inspect ./traces/supply_chain.jsonl
```
Baseline (seed 42, `failures.message_drop: 0.0`): 128 events, 15/15 items
delivered. Both validators pass:
```
PASS supply_chain_pipeline - all hops present
PASS supply_chain_no_lost - 15/15 materials delivered
```
Part B — The experiment
Setting changed: `failures.message_drop`, `0.0 -> 0.2`. Everything else
(seed, agent count, `task.config`) is untouched — see
`supply_chain_drop20.yaml`.
Why this setting: `supply_chain` is a strict linear chain with no
branching, and nothing in the docs or config suggests an
acknowledgment/retry layer. That's structurally different from
`marketplace`, which has many independent buyer-seller pairs that can each
fail without touching the others. My hypothesis was that message drop would
hurt this topology disproportionately, because a single lost handoff
blocks everything downstream of it with no alternate route.
What I expected: treating each of the ~4 hops an item must clear as an
independent 80%-survival event, naive compounding gives roughly
0.8⁴ ≈ 41% end-to-end delivery, i.e. ~6 of 15 items delivered.
What happened (seed 42):
```
nest run supply_chain_drop20.yaml
```
```
PASS supply_chain_pipeline - all hops present
FAIL supply_chain_no_lost - 12 of 15 materials not delivered
  (round 1: 2 of 3; round 2: 1 of 3; round 3: 3 of 3; round 4: 3 of 3; round 5: 3 of 3)
```
Only 3 of 15 items got through — about half of what the naive 41% estimate
predicted, and every item in rounds 3-5 was lost entirely.
Investigation
I dumped the raw JSONL trace event-by-event (`agent`, `kind`,
`send`/`receive`/`dropped`) to see where the loss was concentrated, rather
than just trusting the aggregate number.
Of the 15 messages supplier -> manufacturer, only 6 got through. That's a
40% survival rate at the first hop alone, well below the nominal 80% you'd
expect from `message_drop: 0.2` — because the drops weren't spread evenly,
they included one run of 5 consecutive drops in a row.
The 6 that survived stage 1 mostly survived stage 2 (5/6) and stage 3
(3/5), landing on 3 final deliveries.
Crucially, there is no retry: a dropped message is gone for good, and the
next stage never asks for it again. So the four hops act like four
independent coin flips whose outcome for a given item is locked in the
moment it's dropped — losses compound multiplicatively with no recovery
mechanism, and any run that happens to cluster its drops early gets
punished hardest, since the whole rest of the chain for that item is
moot.
To check whether seed 42 was simply an unlucky draw or the drop rate is
systematically worse than advertised, I reran the identical config across
five more seeds (1, 2, 3, 7, 99), changing nothing but `seed`:
seed	items lost (of 15)	items delivered
42	12	3
1	7	8
2	7	8
3	7	8
7	9	6
99	8	7
Average across the 5 additional seeds: ~7.6 lost / ~49% delivered — much
closer to the naive 41% compounding estimate than seed 42's 20%. So seed 42
specifically drew an unusually bad clustering of drops; it isn't evidence of
a bug in the drop-injection logic.
The actual finding isn't the seed-42 number — it's the spread. A linear,
no-retry, no-redundancy pipeline shows much higher run-to-run variance in
outcome than the "expected value" math suggests (delivered ranged from 3 to
8 out of 15 across just 6 seeds, at a single fixed 20% drop rate). For a
real-world 4-tier supply chain — or any protocol design modeled on this
topology — that variance, not just the mean, is the risk that matters: the
same "20% drop rate" input can mean "half your goods get through" or
"a fifth of your goods get through," depending on where in the sequence the
drops happen to land.
How I ran this
```
pip install "nest-core[plugins]"
nest doctor                       # 7/7 checks passed
nest run supply_chain             # baseline
nest scenarios cp supply_chain ./supply_chain_drop20.yaml
# edited: name, description, output.trace, failures.message_drop -> 0.2
nest run supply_chain_drop20.yaml
python -c "from pathlib import Path; from nest_core.validators import validate_trace; \
  [print(r.passed, r.name, r.detail) for r in validate_trace(Path('traces/supply_chain_drop20.jsonl'), 'supply_chain')]"
# repeated with seed in {1,2,3,7,99} to check variance
```
Use of AI
I used Claude to install and run `nest-core` in a sandboxed environment,
execute the baseline and modified scenarios, parse the raw JSONL trace to
identify where drops clustered, and draft this README from the actual
command output above. The experimental design (which setting to change, the
hypothesis, and the decision to re-run across multiple seeds once the
seed-42 result looked more extreme than the math predicted) was
directed by me; all numbers above are from real `nest run` / `nest inspect`
/ `validate_trace` output, not invented.
