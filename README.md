# rcg-sites

Reusable RCG site pipelines and shared GitHub Actions (Docker, nginx, Nomad, `.github`).

## Branches

| Branch | Meaning |
| --- | --- |
| `main` | Library entry (this README). |
| [`from-coastal-science`](https://github.com/VWJF/rcg-sites/tree/from-coastal-science) | Filtered import from coastal-science `main`: `.github/`, `nomad/`, `nginx/`, Dockerfiles, docker-compose, `entrypoint.sh`. Commit **authors and dates preserved**; site content history omitted. |
| [`from-coastal-science-modular`](https://github.com/VWJF/rcg-sites/tree/from-coastal-science-modular) | Same filter from `refactor/ghaworkflow-modular`. |

## Scope

Not included: Hugo content, themes, `static/`, blog posts, etc.

## Related

Packaging expectations informed by [`VWJF/mirroring`](https://github.com/VWJF/mirroring) (`dockerize`).
