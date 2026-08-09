# claude-code-measure-efficiency

Measure how efficiently you use Claude Code. Three local scripts read the transcripts Claude Code already writes to disk and report where your tokens go: model tier mix, cache economics, cost per unit of work, and rework signals. Run them on their own to judge your current usage, or before and after a configuration change to verify that an optimization actually worked.

Most published Claude Code optimization guides assert a saving. Few provide a way to verify one. This repository contains the measurement scripts, the optimization setup that was evaluated with them (a seven-agent model-tiering configuration in `setup/`), and the reference results from that evaluation. The results below include both the effects that were confirmed and the effect that was not.

Everything runs locally. The scripts read `~/.claude/projects/**/*.jsonl`, transmit nothing, and print aggregate counts only. No conversation content is written to output.

## Contents

```
tools/usage_report.py     token mix by model tier, cache economics, subagent dispatch counts
tools/decompose.py        cost per unit of work, split into four components
tools/friction.py         rework proxies: tool errors, correction rate, edit churn
tools/session_growth.py   per-session context growth and cost concentration
setup/                    the seven-agent configuration and policy blocks that were measured
```

## Requirements

Python 3.8 or later. No dependencies. Claude Code must have written session transcripts to `~/.claude/projects/`.

## Usage

```bash
python3 tools/usage_report.py                    # last 14 days
python3 tools/usage_report.py --since 2026-08-03  # compare two periods
python3 tools/decompose.py 2026-08-03             # cost decomposition across a change date
python3 tools/friction.py 2026-08-03              # rework proxies across a change date
python3 tools/session_growth.py                   # per-session context growth, last 14 days
```

Pass the date a configuration change took effect. Each script splits the transcript history into a before period and an on/after period.

## Units

Cost-equivalent is computed by applying published API list rates to observed token counts. It is a comparison unit between periods. It is not a bill and does not represent subscription pricing. Rates are defined at the top of each script and should be updated when list prices change.

Cache reads are priced at 0.1x the input rate. Cache writes are priced at 1.25x. Both are configurable constants.

## Reference results

One user, one machine. Baseline period of 36 days and 512 sessions. Measurement period of 6 days and 115 sessions. The change under evaluation was the introduction of the agent configuration in `setup/`, which routes mechanical work to lower-cost model tiers.

### Adoption

The configuration was adopted in practice rather than remaining unused.

| Metric | Baseline | After |
|---|---|---|
| Fleet subagent dispatches | 1 of 103 | 43 of 92 |
| Built-in Explore dispatches | 46 | 8 |
| Sonnet and Haiku share of tokens | 1.9% | 6.5% |
| Highest-cost tier share of tokens | 29.9% | 6.5% |
| Cache hit rate | 96.2% | 97.4% |

### Cost decomposition

Cost is normalized by output tokens so that periods of different volume are comparable. Splitting the total into components separates the effect of model tiering from other effects.

| Component | Baseline | After | Change |
|---|---|---|---|
| Output tokens | 32.6 | 26.7 | −18% |
| Cache writes | 57.0 | 48.6 | −15% |
| Fresh input | 0.4 | 0.0 | −88% |
| Cache reads | 120.3 | 142.9 | +19% |
| **Total** | **210.3** | **218.3** | **+3.8%** |

Every component that model tiering acts on decreased between 15% and 18%. Cache reads, which model tiering does not act on, increased 19%. Cache reads are the largest single component, so the net result was approximately flat.

The cause of the cache read increase is visible in the workload shape. Sessions grew from 132 to 212 turns on average, an increase of 60%. Cache read per turn grew from 0.20M to 0.27M, an increase of 35%. Longer sessions re-read a larger transcript on every turn.

Cache reads accounted for 57.2% of cost in the baseline period and 65.4% after. This is the most transferable finding in the dataset: for this workload, session length governs cost more than model selection does.

Holding cache read per unit at its baseline value produces a total of 195.6, or −7.0%. That figure is a model rather than a measurement. It assumes session length would have been unchanged, which is an assumption, not an observation.

`tools/session_growth.py` reports this cost per session, so the sessions that grew past their task, where a fresh session would have paid, are visible individually.

### Rework proxies

Reported per turn, because session length differed between periods.

| Signal | Baseline | After | Change |
|---|---|---|---|
| Tool error rate | 3.4% | 2.9% | −15% |
| Tool errors per turn | 0.0175 | 0.0158 | −10% |
| Correction rate per turn | 0.00121 | 0.00094 | −22% |
| Edit churn | 87.6% | 80.9% | −6.7 points |

Correction rate is the fraction of human turns matching a regex for correction language, for example "that is wrong", "does not work", "revert", "you missed". It is a proxy for the operator judging output to be incorrect.

### Summary

Confirmed: the configuration was adopted, shifted work to lower-cost tiers, reduced the cost components it acts on by 15% to 18%, and coincided with lower measured rework.

Not confirmed: a net reduction in total cost. Net cost per unit of work rose 3.8%, attributable to a concurrent increase in session length that the configuration does not address.

## Limitations

1. Sample size is one user, one workload, six days of measurement against a 36 day baseline. These results indicate direction, not magnitude, and are not generalizable without replication.
2. The periods are not a controlled experiment. Workload composition differed. Some work during the measurement period was performed in a different tool and is absent from these transcripts, which means the remaining Claude Code workload may not be comparable in difficulty. This confound is unquantified and could account for part of the +3.8%.
3. Cost-equivalent uses list API rates. A subscription user pays a fixed amount regardless. The figure compares periods; it does not report spend.
4. Output tokens are used as the proxy for work produced. This is crude. A refactor and a long explanation are not equivalent work at equal token counts.
5. The scripts parse an undocumented internal transcript format. Format changes will break them. They are best-effort and version-pinned to nothing.
6. Friction metrics measure friction, not correctness. Regex correction detection misses indirect phrasing and cannot detect defects the operator never noticed.
7. The counterfactual in the decomposition is a model, not a measurement.

## The setup that was measured

See [`setup/`](setup/). It contains seven subagent definitions pinned to specific model tiers, and three policy blocks for `CLAUDE.md`. It is a small, deliberately curated configuration rather than a large library. Adoption data above suggests the curation matters: a smaller set of clearly differentiated agents was selected by the routing model in 47% of dispatches, where the prior state was 1%.

## License

MIT. See [LICENSE](LICENSE).
