# TIG Code Submission

## Submission Details

* **Challenge Name:** vehicle_routing
* **Algorithm Name:** hgs_epure_ep3
* **Copyright:** 2026 Copyright FP Labs modifications FiveStacks
* **Identity of Submitter:** FiveStacks
* **Identity of Creator of Algorithmic Method:** Thibaut Vidal base implementation by FP Labs
* **Unique Algorithm Identifier (UAI):** null

## Hyperparameters

All `Params` fields are set via the TIG runtime hyperparameters map (serde-merged over the
defaults; `exploration_level` is the only key the solver intercepts itself). The solver's
two Iteration-1/2 levers:

| key | type | default | behaviour |
|---|---|---|---|
| `regret_k` | usize | `1` | Repair look-ahead in the SREX crossover. `1` = stock greedy repair (bit-identical), `>1` = regret-k insertion (insert the unplanned customer with the largest best-minus-second-best gap first). |
| `credit_scale_permille` | usize | `1000` | LS deterioration-budget multiplier (permille). `1000` = exact stock `move_credit` cap, `>1000` lets the local search accept more deteriorating moves to escape local minima, `<1000` tightens it into a purer descent. |

**Recommended scored setting:** `credit_scale_permille = 2500`. Measured same-seed/reproducible:
quality +2.2% at n=600 and +1.9% at n=1000 vs the stock setting (see `genetic.rs`/`local_search.rs`
commit history for the runs). `regret_k > 1` does **not** combine with the credit widening — the
two cancel (measured 171290 vs 172263 at n=600) — so either keep `regret_k = 1` with the credit
boost, or run regret-k alone with `credit_scale_permille = 1000`.

**Default-build guarantee:** both default to stock-equivalent values (`regret_k = 1`,
`credit_scale_permille = 1000`), so an unmodified submission is bit-identical to the base
`hgs_epure`. The `bench_standalone_fuel` Cargo feature (the `__fuel_remaining` shim for local
benchmarks) is **off** in the default build and must never be enabled on a scored build.

## Fuel profile (measured, judged build 0.0.8)

Fuel is TIG's build-instrumented counter (LLVM pass), **not** wall time — the numbers below were
measured by running the real instrumented `.so` through the judge's `tig-runtime`/`tig-verifier`
(dev/runtime `0.0.8` images). Max per-instance budget is 500 billion; the `--fuel 2_000_000_000`
`tig-runtime` default is a test flag, not the on-chain budget (benchmarker-set per track through
`track_settings`, capped by `challenge_config["max_fuel_budget"]`).

| track | fuel per nonce (converging) | share of 500e9 cap | notes |
|---|---|---|---|
| n=600 | ~90.5e9 | 18% | small-instance preset iterates far longer |
| n=3000 | ~990e9 (corrected target) | ~20% | see note below — old "~4.1e9 cheap" profile was a give-up artifact |
| n=2000 | ~1.8e9 **or** up to ~468e9 | 0.4%–94% | **bimodal** — see below |

**Important correction (measured on the real fuel counter, judged 0.0.8):** earlier
profiles listed n=3000 at ~4.1e9 and n=2000's converging nonces at ~1.8e9 as "cheap."
That was an **artifact of a target bug, not real convergence**: the early-stop TARGETS
table lacked n=2000/3000 entries, so the `.min_by_key` fallback snapped both to the
n=1000 line (186200), which sits ABOVE both live qualifying lines — making the internal
`giveup_quality` unreachable and causing the solver to bail in phase 1 with ~80k quality
on only ~2-4e9 fuel. It could never qualify those tracks. With the corrected targets
(n=2000 -> 148123, n=3000 -> 136092 from live round-25 lines), the solver genuinely
searches: n=3000 nonces now run to ~990e9 (~20% of cap) to chase the line, and n=2000's
hard nonces reach ~150-980e9 (the previous ~1.8e9 "converging" figure was the same
give-up artifact). Benchmarkers must set budgets against the CORRECTED profile, not the
stale "cheap" one.

**n=2000 is bimodal.** Roughly 3 in 5 nonces converge in ~1.8e9, but a hard subset (e.g. nonces 4
and 7 of the sampled set) needs up to **~468e9 (94% of the cap)** to converge. Fuel is **not**
predictable per nonce on this track. Achieved quality on those hard nonces rises monotonically
with the benchmarker's fuel budget (10e9→122718, 50e9→146628, 100e9→159610, 500e9→173411 on one
sampled nonce), so a benchmarker running a tight budget gets visibly lower quality than one
running near the cap. The solver does **not** self-terminate early — it spends whatever budget it
is given improving and is hard-killed at the cap, always having saved its best incumbent
(feasible + verifier-passing) at the last phase boundary. Budgets well below ~50e9 on n=2000 will
under-achieve on the hard nonces by construction.

## References and Acknowledgments

This submission builds on hgs_prometheus: a hybrid genetic search for vehicle routing with
capacity and time window constraints, combining a population-based search with local search and
problem decomposition. It has no thread, no clock and no filesystem access; the search is
deterministic and anytime, and feasible solutions are reported through the TIG solution callback.

### Academic Papers

[1] Vidal, T., Crainic, T. G., Gendreau, M., and Prins, C. (2013). *A hybrid genetic algorithm with adaptive diversity management for a large class of vehicle routing problems with time windows*. Computers & Operations Research, 40(1), 475-489. https://doi.org/10.1016/j.cor.2012.07.018

[2] Vidal, T. (2022). *Hybrid genetic search for the CVRP: Open-source implementation and SWAP\* neighborhood*. Computers & Operations Research, 140, 105643. https://doi.org/10.1016/j.cor.2021.105643

### Code References

- Base algorithm: `hgs_epure`, authored by **FP Labs** (vehicle_routing, c002_a110). Copyright remains with FP Labs. This submission is a derivative fork of that base.
- `hgs_prometheus`: the hybrid genetic search this lineage builds on (see References above).

### Modifications by FiveStacks (this submission)

The base `hgs_epure` engine (population, local search, SREX crossover, decomposition, reverse mode,
constructive, individual, problem) is FP Labs' work. FiveStacks adds:

- `regret_k` hyperparameter: look-ahead repair in the SREX unplanned-customer reinsertion.
- `credit_scale_permille` hyperparameter: gates the local-search deterioration budget.
- Incremental population bookkeeping and bench scaffolding (feature-gated `__fuel_remaining` shim,
  off in default build).
- Early-stop layer (`early_stop`, `target_quality`): between-decomposition-phase stopping to halt a
  run once it clearly reaches or can no longer reach the qualifying quality. The technique is
  informed by the approach in `vehicle_routing/murelan`; re-implemented inside this crate.
- Track profiles for the 2000/3000 `n_nodes` tracks (10 decomposition phases, scaled granularity).

## License

The files in this folder are under the following licenses:
* TIG Benchmarker Outbound License
* TIG Commercial License
* TIG Inbound Game License
* TIG Innovator Outbound Game License
* TIG Open Data License
* TIG THV Game License

Copies of the licenses can be obtained at:
https://github.com/tig-foundation/tig-monorepo/tree/main/docs/licenses
