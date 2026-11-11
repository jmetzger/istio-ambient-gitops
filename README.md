# istio-ambient-gitops

Istio im Ambient Mode und die Beispielanwendung Bookinfo, vollständig von ArgoCD aus diesem Repo ausgerollt
(App of Apps mit Sync Waves).

| Ordner | Inhalt |
|---|---|
| `bootstrap/root-app.yaml` | Root-Application. Der einzige Handgriff am Cluster. |
| `apps/` | Je Komponente eine ArgoCD-Application (Reihenfolge über Sync Waves). |
| `charts/istio-1.31.4/` | Helm-Charts `base`, `istiod`, `cni`, `ztunnel` aus dem Istio-Release 1.31.4 (Apache 2.0, unverändert). |
| `bookinfo/` | Namespace (Ambient-Label, Waypoint-Label), Bookinfo, Waypoint, Gateway und HTTPRoute. |

## Reihenfolge

| Wave | Application | Quelle |
|---|---|---|
| -3 | `gateway-api-crds` | `kubernetes-sigs/gateway-api`, Tag `v1.5.1`, `config/crd/standard` |
| -2 | `istio-base` | `charts/istio-1.31.4/base` (CRDs) |
| -1 | `istiod` | `charts/istio-1.31.4/istiod`, `profile: ambient` |
| 0 | `istio-cni` | `charts/istio-1.31.4/cni`, `profile: ambient` |
| 1 | `ztunnel` | `charts/istio-1.31.4/ztunnel` |
| 2 | `bookinfo` | `bookinfo/` |

Warum liegen die Charts im Repo? Für Istio 1.31.4 gibt es (Stand 11.11.2026) keine Charts im Helm-Repo
`istio-release.storage.googleapis.com/charts`. Die Charts stammen aus dem Release-Archiv
(`istio-1.31.4/manifests/charts`).
