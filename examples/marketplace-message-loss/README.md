# Marketplace under message loss

Author: Samuel Haas, MIT AI Studio (MAS.665), Fall 2026
Scenario: `marketplace` (50 buyers, 50 sellers, state-machine agents, seed 42)
Setting changed: `failures.message_drop`

## What I did

I took the built-in marketplace scenario and changed one thing: the fraction of
messages the network randomly drops. I ran it at 0%, 10%, 30%, and 50% loss, kept the
seed and everything else fixed, and compared the results. My question was simple. If you
delete a share of the messages, do you lose roughly that same share of deals, or does it
get worse than that?

## The scenario

The marketplace runs 100 agents, half buyers and half sellers. A buyer sends a signed
offer to a seller ("buy this product for $50"), the seller replies, and the deal either
closes or gets rejected. Every offer is a short back and forth, not a single message. In
a clean run every message arrives and about 53% of offers turn into deals. The rest are
rejections, which is normal haggling.

Baseline run:

```bash
nest run marketplace
nest inspect ./traces/marketplace.jsonl
nest report ./traces/marketplace.jsonl -o report.html
```

## The setting I changed and why

I changed `failures.message_drop`, the probability that any single message silently
disappears. I picked it because it stresses the coordination between agents rather than
any one agent's logic. That is the thing NANDA Town is built to test: does the protocol
still work when the network is unreliable.

My hypothesis, written before I ran it: deals would not fall one to one with the drop
rate. They would fall faster. A deal needs a chain of messages to all get through, so
losing any one of them kills the whole conversation, not just that message. (I worked
the prediction out with a collaborator before looking at the numbers. See "Tools and
help.")

Method: I swept `message_drop` over 0.0, 0.1, 0.3, 0.5 with the seed fixed at 42, so the
drop rate is the only variable.

```bash
for d in 0.0 0.1 0.3 0.5; do
  tag=$(echo $d | tr -d '.')
  sed "s/^  message_drop: .*/  message_drop: $d/" marketplace_message_drop.yaml > _run.yaml
  nest run _run.yaml -o ./traces/mkt_drop$tag.jsonl
  nest report ./traces/mkt_drop$tag.jsonl -o ./reports/report_drop$tag.html
done
```

## What happened

| message_drop | messages sent | delivery_rate | deal_rate | rejection_rate | dropped |
|---:|---:|---:|---:|---:|---:|
| 0.0 | 1000 | 1.000 | 0.532 | 0.468 | 0 |
| 0.1 | 431  | 0.905 | 0.563 | 0.319 | 41 |
| 0.3 | 174  | 0.718 | 0.439 | 0.187 | 49 |
| 0.5 | 89   | 0.438 | 0.328 | 0.062 | 50 |

The raw traces for each run are in `traces/`, so anyone can rerun `nest inspect` or
`nest report` and get the same numbers.

The main result is the "messages sent" column. At 30% loss the agents ended up sending
174 messages instead of 1000. That is an 83% drop in activity from a 30% drop in
messages. My hypothesis held, and the effect was bigger than I expected. In the traces a
lost message shows up as a `dropped` event with nothing after it on that conversation
id. The exchange just stops. Because a deal needs every message in its chain to land,
and each message is a separate chance to fail, the losses stack up and take out whole
negotiations instead of single messages.

## What surprised me and how I checked it

Two things.

First, the `dropped_count` metric looked like it capped near 50 even at 50% loss, which
made no sense if half of 2000 messages were vanishing. I pulled the raw event types out
of the trace (`grep -o '"kind":"[a-z_]*"'`) and found the event is called `dropped`, not
`drop`, and that sent equals received plus dropped in every run (at 50% loss, 89 = 39 +
50). So the count is small only because the total number of messages sent collapses, not
because few are lost. The real signal was in the send count the whole time, not the drop
count.

Second, the deal rate went up at 10% loss, from 0.532 to 0.563. Losing messages made
deals slightly more likely, which is backwards. My read is that rejections come from
longer haggling, so those longer exchanges are more exposed to loss and get cut off
first. What survives is a higher share of quick, clean yes deals. The supporting number
is that deal_rate plus rejection_rate, meaning the share of offers that resolved at all,
falls steadily: 1.00, 0.88, 0.63, 0.39. That matches a round trip getting less likely as
more hops have to survive. I would not call this proven off one seed. To confirm it I
would rerun across several seeds and measure messages per resolved offer for deals versus
rejections. For now I treat it as a well supported guess.

## Tools and help

I used Claude Code (Claude Opus 4.8) as the coding agent to set up the environment (uv
virtual env, `nest-core[plugins]`), run the sweep, and pull apart the traces. Under the
hood it is the `nest-core` 0.1.4 CLI (`run`, `inspect`, `report`, `scenarios cp`) plus
uv, grep, sed, and awk. I chose the scenario and the setting, wrote the hypothesis before
seeing results, and checked each finding against the raw traces instead of trusting the
summary metrics. The `dropped_count` confusion is why that mattered.

## What I would build next

A retry and acknowledgement option for the comms layer, so a dropped message gets resent
instead of killing the conversation, plus a validator that reports goodput, meaning deals
closed per message sent, across a message_drop sweep. This experiment shows that
delivery_rate on its own hides the real damage. A goodput curve with and without retries
would let you compare how well two protocol stacks hold up under loss, which is the
question NANDA Town exists to answer.
