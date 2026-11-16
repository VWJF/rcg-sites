# rcg-sites

Reusable RCG site pipelines and shared GitHub Actions.

## Branches

| Branch | Meaning |
| --- | --- |
| `main` | Library / product entry (starts empty aside from this README). |
| [`from-coastal-science`](https://github.com/VWJF/rcg-sites/tree/from-coastal-science) | Full git history imported from [`coastal-science/coastal-science.github.io`](https://github.com/coastal-science/coastal-science.github.io) (`main` tip). |
| [`from-coastal-science-modular`](https://github.com/VWJF/rcg-sites/tree/from-coastal-science-modular) | Same lineage plus the modular GHA refactor (`refactor/ghaworkflow-modular`). |

History was preserved via a normal branch push (all ancestor commits retained). Active extraction of reusable workflows/actions will land on dedicated branches; consumers should pin by tag/SHA once released.

## Related prior art

Packaging expectations are informed by [`VWJF/mirroring`](https://github.com/VWJF/mirroring) (`dockerize`): clear public surface, README/FAQ design notes, versioned releases, and CI for the library itself.
