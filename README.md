# rcg-sites

Reusable RCG site pipelines and shared GitHub Actions (Docker, nginx, Nomad, `.github`).

## Branches

| Branch | Meaning |
| --- | --- |
| `main` | Library entry (this README). |
| [`refactor-from-cs`](https://github.com/VWJF/rcg-sites/tree/refactor-from-cs) | Filtered from [`coastal-science/coastal-science.github.io`](https://github.com/coastal-science/coastal-science.github.io) `main`: `.github/`, `nomad/`, `nginx/`, Dockerfiles, docker-compose, `entrypoint.sh`. Original commit **authors and dates** preserved; Hugo site content omitted. |

## Scope

Not included: Hugo content, themes, `static/`, blog posts, etc.

## Related

Packaging expectations informed by [`VWJF/mirroring`](https://github.com/VWJF/mirroring) (`dockerize`).
