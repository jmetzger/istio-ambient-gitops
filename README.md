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

## Ausrollen

Voraussetzung: ein Cluster mit CNI (getestet: kubeadm, Kubernetes v1.37.2, Calico) und für das Gateway
eine LoadBalancer-Adresse (getestet: MetalLB). Istio ist **nicht** installiert.

```bash
# 1. ArgoCD installieren
#    --server-side ist nötig: das CRD applicationsets.argoproj.io ist für ein normales "kubectl apply" zu groß
kubectl create namespace argocd
kubectl apply --server-side --force-conflicts -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/v3.6.1/manifests/install.yaml
kubectl -n argocd wait --for=condition=available deployment --all --timeout=240s

# 2. Health-Prüfung für Applications (damit die Sync Waves zwischen den Applications warten)
curl -sLO https://raw.githubusercontent.com/jmetzger/istio-ambient-gitops/main/bootstrap/argocd-cm-app-health.yaml
kubectl -n argocd patch configmap argocd-cm --type merge --patch-file argocd-cm-app-health.yaml

# 3. Root-Application, ab hier macht ArgoCD alles
kubectl apply -f https://raw.githubusercontent.com/jmetzger/istio-ambient-gitops/main/bootstrap/root-app.yaml

# 4. Zuschauen (rund 3 Minuten)
kubectl -n argocd get applications -w
```

Erwartung: sieben Applications `Synced` und `Healthy` (`root`, `gateway-api-crds`, `istio-base`, `istiod`,
`istio-cni`, `ztunnel`, `bookinfo`).

## Prüfen

```bash
kubectl -n istio-system get pods          # istiod, je Node ein istio-cni-node und ein ztunnel
kubectl -n bookinfo get pods,gateway      # Pods 1/1 (kein Sidecar), beide Gateways PROGRAMMED True
istioctl ztunnel-config workloads | grep bookinfo   # Spalten WAYPOINT=waypoint, PROTOCOL=HBONE

# im Cluster
kubectl -n bookinfo exec deploy/ratings-v1 -- curl -s productpage:9080/productpage | grep -o "<title>.*</title>"

# von außen
IP=$(kubectl -n bookinfo get gateway bookinfo-gateway -o jsonpath='{.status.addresses[0].value}')
curl -s -o /dev/null -w '%{http_code}\n' http://$IP/productpage
```

## Abbauen

```bash
kubectl -n argocd delete application root
```

Die Root-Application löscht ihre Kinder und diese ihre Objekte (Finalizer). **Übrig bleiben** der Namespace
`istio-system` (von `CreateNamespace=true` angelegt), die Istio-CRDs (das Chart markiert sie mit
`helm.sh/resource-policy: keep`) und die Gateway-API-CRDs. Wer wirklich alles weghaben will:

```bash
kubectl get crd -o name | grep -E 'istio.io|gateway.networking' | xargs kubectl delete
kubectl delete namespace istio-system
```

## Was beim Testen aufgefallen ist

| Beobachtung | Ursache | Lösung im Repo |
|---|---|---|
| Alle Applications entstehen fast gleichzeitig, die Sync Waves wirken nicht. | ArgoCD bewertet den Zustand einer `Application` seit Version 1.8 nicht mehr von sich aus. | `bootstrap/argocd-cm-app-health.yaml` |
| `istiod` steht auf `Degraded`. | Der HPA von istiod bekommt ohne metrics-server keinen Messwert. | `autoscaleEnabled: false` in `apps/istiod.yaml` |
| `istio-base` und `istiod` bleiben `OutOfSync` (ValidatingWebhookConfiguration). | istiod stellt `failurePolicy` selbst auf `Fail` und trägt das CA-Bundle ein. | `ignoreDifferences` und `RespectIgnoreDifferences=true` |
| `gateway-api-crds` und `istio-base` bleiben `OutOfSync` (CRDs). | Die CRDs werden per Server-Side Apply geschrieben, aber clientseitig verglichen. | Annotation `argocd.argoproj.io/compare-options: ServerSideDiff=true` |
| ArgoCD meldet `configmap "argocd-cm" not found`, die Root-Application bleibt `Unknown`. | Eine Teil-ConfigMap per `kubectl apply --server-side` entfernt alle anderen Felder samt Labels (gleicher Feld-Manager wie bei der Installation). | `kubectl patch` statt `apply`; Reparatur: Installationsmanifest erneut anwenden |

Getestet am 11.11.2026: ArgoCD v3.6.1, Istio 1.31.4, Gateway API v1.5.1, Kubernetes v1.37.2. Zwei vollständige
Durchläufe von null (Aufbau 2:40 Minuten, Abbau 42 Sekunden).
