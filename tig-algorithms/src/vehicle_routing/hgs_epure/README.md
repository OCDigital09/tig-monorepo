# TIG Code Submission

## Submission Details

* **Challenge Name:** vehicle_routing
* **Algorithm Name:** hgs_epure_ep3
* **Copyright:** Copyright in the base algorithm belongs to FP Labs; modifications © FiveStacks
* **Identity of Submitter:** FiveStacks
* **Identity of Creator of Algorithmic Method:** Thibaut Vidal (see References); base implementation by FP Labs
* **Unique Algorithm Identifier (UAI):** c002_a120

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
