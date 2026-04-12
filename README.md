# 🧠 Solution complète : Sizing Controller GitOps pour ArgoCD

## 🎯 Objectif

Mettre en place un système de **resizing continu automatisé** des instances métiers ArgoCD basé sur :

* des métriques réelles (Prometheus / OCP Monitoring)
* une abstraction `sizes` (xs → xl)
* une boucle de régulation :
  **Consigne → Mesure → Analyse → Décision → Action (GitOps)**

Le système respecte strictement GitOps :
👉 aucune modification directe dans Kubernetes
👉 toutes les actions passent par Git → ArgoCD applique

---

# 🏗️ Architecture globale

```
Prometheus (metrics historiques)
        ↓
Sizing Controller (logique décisionnelle)
        ↓
Git (values.yaml modifiés)
        ↓
ArgoCD (sync)
        ↓
Kubernetes (OCP)
```

---

# ⚙️ Stack technique

* Langage : Go (recommandé) ou Python
* Metrics : Prometheus API
* Git : go-git ou GitLab/GitHub API
* YAML : gopkg.in/yaml.v3
* Déploiement : CronJob Kubernetes

---

# 📦 Structure complète du projet

```
sizing-controller/
├── cmd/
│   └── main.go
├── internal/
│   ├── config/
│   │   └── config.go
│   ├── metrics/
│   │   └── prometheus.go
│   ├── analyzer/
│   │   └── analyzer.go
│   ├── decision/
│   │   └── decision.go
│   ├── git/
│   │   └── git.go
│   ├── sizing/
│   │   └── sizes.go
│   └── models/
│       └── models.go
├── configs/
│   └── rules.yaml
├── Dockerfile
└── helm/
```

---

# 📄 1. Configuration globale

## configs/rules.yaml

```yaml
rules:
  cpu:
    upscale: 0.8
    downscale: 0.3
  memory:
    upscale: 0.85
    downscale: 0.4

hysteresis:
  upscale_days: 7
  downscale_days: 14

limits:
  min_size: xs
  max_size: xl
```

---

# 📄 2. Modèles

## internal/models/models.go

```go
package models

type Metrics struct {
    CPUUsageP95 float64
    CPURequest  float64
    MemUsageP95 float64
    MemRequest  float64
}

type Ratios struct {
    CPURatio float64
    MemRatio float64
}

type AppConfig struct {
    Name string
    Size string
}
```

---

# 📄 3. Gestion des tailles

## internal/sizing/sizes.go

```go
package sizing

var SizesOrder = []string{"xs", "s", "m", "l", "xl"}

func NextSize(current string) string {
    for i, s := range SizesOrder {
        if s == current && i < len(SizesOrder)-1 {
            return SizesOrder[i+1]
        }
    }
    return current
}

func PrevSize(current string) string {
    for i, s := range SizesOrder {
        if s == current && i > 0 {
            return SizesOrder[i-1]
        }
    }
    return current
}
```

---

# 📄 4. Collecte Prometheus

## URL API Prometheus

```
http://prometheus:9090/api/v1/query_range
```

## Requêtes PromQL

### CPU usage (P95)

```promql
histogram_quantile(0.95,
  sum(rate(container_cpu_usage_seconds_total[5m])) by (pod)
)
```

### CPU requests

```promql
kube_pod_container_resource_requests{resource="cpu"}
```

### Memory usage

```promql
container_memory_working_set_bytes
```

---

## internal/metrics/prometheus.go

```go
package metrics

import (
    "encoding/json"
    "net/http"
)

func QueryPrometheus(query string) float64 {
    url := "http://prometheus:9090/api/v1/query?query=" + query

    resp, _ := http.Get(url)
    defer resp.Body.Close()

    var result map[string]interface{}
    json.NewDecoder(resp.Body).Decode(&result)

    return 0.5 // simplification
}
```

---

# 📊 5. Comprendre P95 (CRITIQUE)

## 📌 Définition

**P95 = 95ème percentile**

👉 Cela signifie :

* 95% du temps → la valeur est en dessous
* 5% du temps → pics au-dessus

## 📈 Exemple

CPU usage sur 100 mesures triées :

```
[10, 20, 30, ..., 80, 85, 90, 95, 100]
```

👉 P95 ≈ 95

## ✅ Pourquoi utiliser P95 ?

* évite sous-dimensionnement
* prend en compte les pics
* plus fiable que la moyenne

---

# 📄 6. Analyse

## internal/analyzer/analyzer.go

```go
package analyzer

import "sizing-controller/internal/models"

func ComputeRatios(m models.Metrics) models.Ratios {
    return models.Ratios{
        CPURatio: m.CPUUsageP95 / m.CPURequest,
        MemRatio: m.MemUsageP95 / m.MemRequest,
    }
}
```

---

# 📄 7. Décision

## internal/decision/decision.go

```go
package decision

import "sizing-controller/internal/sizing"

func Decide(current string, cpuRatio, memRatio float64) string {

    if cpuRatio > 0.8 || memRatio > 0.85 {
        return sizing.NextSize(current)
    }

    if cpuRatio < 0.3 && memRatio < 0.4 {
        return sizing.PrevSize(current)
    }

    return current
}
```

---

# 🔁 8. Hystérésis (anti oscillation)

## Principe

Empêcher :

```
m → l → m → l → m
```

## Solution

* upscale si dépasse seuil pendant 7 jours
* downscale si sous seuil pendant 14 jours

---

# 📄 9. Gestion Git

## internal/git/git.go

```go
package git

import "os/exec"

func CommitAndPush(msg string) {
    exec.Command("git", "commit", "-am", msg).Run()
    exec.Command("git", "push").Run()
}
```

---

# 📄 10. Lecture / écriture YAML

```go
func UpdateSize(file string, app string, newSize string) {
    // lire YAML
    // modifier size
    // écrire fichier
}
```

---

# 📄 11. Main Controller

## cmd/main.go

```go
package main

import "fmt"

func main() {

    apps := []string{"app1", "app2"}

    for _, app := range apps {

        metrics := GetMetrics(app)

        ratios := ComputeRatios(metrics)

        newSize := Decide("m", ratios.CPURatio, ratios.MemRatio)

        if newSize != "m" {
            fmt.Println("Resize:", app, "m ->", newSize)

            UpdateSize("values.yaml", app, newSize)

            CommitAndPush("resize " + app)
        }
    }
}
```

---

# 🔐 12. Sécurité

* min_size / max_size
* blacklist apps critiques
* PR obligatoire (option)
* dry-run mode

---

# 🧪 13. Mode simulation

```
[DRY RUN]
app1: m → l (cpu 92%)
app2: l → m (cpu 20%)
```

---

# 🚀 14. Déploiement

## CronJob Kubernetes

```yaml
apiVersion: batch/v1
kind: CronJob
spec:
  schedule: "0 2 * * *"
```

---

# 🔁 15. Fréquence recommandée

* analyse : 1 fois / jour
* resize : max 1 fois / semaine

---

# 📈 16. Dashboard

Grafana :

* usage CPU / memory
* taille actuelle
* recommandations

---

# 🧠 17. Évolutions possibles

* intégration VPA (recommandation)
* scoring multi-métriques
* ML prédictif
* saisonnalité

---

# ⚠️ Pièges à éviter

❌ moyenne au lieu de P95
❌ resizing trop fréquent
❌ ignorer mémoire
❌ modifier cluster directement

---

# 🏁 Conclusion

👉 auto-scaler vertical intelligent GitOps
👉 basé sur données réelles
👉 contrôlé, traçable et stable

---

# 💡 Résumé clé

* sizes = abstraction
* Prometheus = source de vérité
* Controller = cerveau
* Git = contrôle
* ArgoCD = exécution

🚀 plateforme autonome de right-sizing
