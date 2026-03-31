# ocp-workloads

Workloads-Repo für OpenShift. Verwaltet vom **Platform Team**.  
Entwickler-Teams erhalten Zugriff auf ihre jeweiligen App-Repos — nicht auf dieses Repo.

---

## Konzept

```
ocp-workloads/
├── apps/
│   ├── groups/                      ← Team-Definitionen (eine Datei pro Team)
│   │   ├── team-a.yaml              (alice)
│   │   ├── team-b.yaml              (bob)
│   │   └── team-c.yaml              (charlie)
│   ├── project-a/
│   │   ├── appproject-app.yaml      ← Application → project-config Chart
│   │   ├── appproject-values.yaml   ← project, sourceRepos, ArgoCD-Rollen
│   │   ├── my-app/
│   │   │   ├── app.yaml             ← Application → app-config Chart
│   │   │   └── values.yaml          ← app, project, appRepo, security, quota, rbac
│   │   └── your-app/
│   │       ├── app.yaml
│   │       └── values.yaml
│   └── project-b/
│       ├── appproject-app.yaml
│       ├── appproject-values.yaml
│       ├── my-app/
│       │   ├── app.yaml
│       │   └── values.yaml
│       └── your-app/
│           ├── app.yaml
│           └── values.yaml
└── charts/
    ├── app-config/                  ← Helm Chart: Namespace, Quota, NetPol, RBAC
    │   ├── Chart.yaml               │  + Pod Security Labels + ArgoCD Application
    │   ├── values.yaml
    │   └── templates/
    │       ├── namespace.yaml
    │       ├── resourcequota.yaml
    │       ├── networkpolicy.yaml
    │       ├── rbac.yaml
    │       └── application.yaml
    └── project-config/              ← Helm Chart: ArgoCD AppProject mit Rollen
        ├── Chart.yaml
        ├── values.yaml
        └── templates/
            └── appproject.yaml
```

---

## Sync-Flow

```
workloads-app (aus ocp-platform, recurse: true auf apps/)
├── groups/
│   ├── team-a.yaml                  Wave -1
│   ├── team-b.yaml                  Wave -1
│   └── team-c.yaml                  Wave -1
├── project-a/
│   ├── appproject-app.yaml          Wave -1  → AppProject via project-config Chart
│   ├── my-app/app.yaml              Wave  0  → Namespace, Quota, NetPol, RBAC,
│   │                                           Pod Security + App via app-config Chart
│   └── your-app/app.yaml            Wave  0
└── project-b/
    ├── appproject-app.yaml          Wave -1
    ├── my-app/app.yaml              Wave  0
    └── your-app/app.yaml            Wave  0
```

> `charts/` wird **nicht** von `workloads-app` deployt — die Charts werden als
> Helm-Source direkt in den `app.yaml` / `appproject-app.yaml` referenziert.

---

## Berechtigungskonzept

Einheitliche Terminologie auf beiden Ebenen: `admin`, `edit`, `view` — sowohl für
OpenShift Namespace-Zugriff als auch für ArgoCD AppProject-Rollen.

| Ebene | admin | edit | view |
|---|---|---|---|
| **OpenShift** (Namespace) | RoleBinding `admin` | RoleBinding `edit` | RoleBinding `view` |
| **ArgoCD** (AppProject) | `*`, alle Rechte | `get`, `sync`, `override` | `get` (read-only) |

Konfiguriert über Gruppen (`*Groups`) oder einzelne User (`*Users`) — je Ebene unabhängig.

Teams und Rollen sind **entkoppelt** — team-a kann in App X admin und in App Y edit sein.

### Berechtigungsmatrix (Test-Setup)

| User | Team | OpenShift Namespace | ArgoCD AppProject |
|---|---|---|---|
| `admin` | cluster-admins | cluster-admin | admin |
| `alice` | team-a | admin | edit |
| `bob` | team-b | edit | edit |
| `charlie` | team-c | view | view |

---

## Verantwortlichkeiten

| Wer | Was |
|---|---|
| Platform Team | `apps/groups/` — Teams und Mitglieder pflegen |
| Platform Team | `charts/` — Helm Charts pflegen |
| Platform Team | `apps/<project>/appproject-app.yaml` + `appproject-values.yaml` |
| Platform Team | `apps/<project>/<app>/app.yaml` + `values.yaml` |
| Entwickler | Eigenes App-Repo (Helm Chart oder Manifeste) |

---

## Team-Management

### Neues Team anlegen

```powershell
# apps/groups/team-d.yaml anlegen (Vorlage: apps/groups/team-a.yaml)
git add . && git commit -m "feat(groups): add team-d"
git push
```

### Mitglied zu Team hinzufügen

**1. Gruppe in Git pflegen:**

```yaml
# apps/groups/team-a.yaml
users:
  - alice
  - neuer-user
```

```powershell
git add . && git commit -m "feat(groups): add neuer-user to team-a"
git push
```

**2. Passwort direkt im Secret anlegen:**

```powershell
$existing = oc get secret htpasswd-secret -n openshift-config `
  -o jsonpath='{.data.htpasswd}' | `
  [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_))

# Hash generieren: https://bcrypt-generator.com (Rounds 10)
$combined = "$existing`nneuer-user:`$2a`$10`$HASH_HIER"

$encoded = [System.Convert]::ToBase64String(
  [System.Text.Encoding]::UTF8.GetBytes($combined)
)

oc patch secret htpasswd-secret -n openshift-config `
  --type merge `
  -p "{`"data`":{`"htpasswd`":`"$encoded`"}}"
```

---

## Neues Projekt anlegen

### 1. AppProject anlegen

```powershell
mkdir apps\project-c

# appproject-app.yaml  (Vorlage: apps/project-a/appproject-app.yaml)
# appproject-values.yaml:
#   project: project-c
#   sourceRepos:
#     - https://github.com/chriwo42-lang/ocp-workloads.git
#     - https://github.com/chriwo42-lang/new-app.git
#   editGroups:  [team-a]
#   viewGroups:  [team-c]
#   editUsers:   [alice]       # optional: einzelne User direkt
```

### 2. App anlegen

```powershell
mkdir apps\project-c\new-app

# app.yaml  (Vorlage: apps/project-a/my-app/app.yaml)
#   → destination.namespace: project-c-new-app anpassen
# values.yaml:
#   app: new-app
#   project: project-c
#   security:
#     podSecurityEnforce: restricted
#   appRepo:
#     url: https://github.com/chriwo42-lang/new-app.git
#   rbac:
#     adminGroups: [team-a]
#     editGroups:  [team-b]
#     viewGroups:  [team-c]
#     adminUsers:  []           # optional: einzelne User direkt
```

### 3. Commit & Push

```powershell
git add . && git commit -m "feat: add project-c with new-app"
git push
```

---

## Helm Charts

### `charts/app-config` — Namespace-Konfiguration + ArgoCD Application

Deployt direkt in den Ziel-Namespace. Einheitliche RBAC-Felder:

| Parameter | OpenShift-Rolle | Pflicht |
|---|---|---|
| `app`, `project`, `appRepo.url` | — | ✅ |
| `rbac.adminGroups` / `rbac.adminUsers` | admin | — |
| `rbac.editGroups` / `rbac.editUsers` | edit | — |
| `rbac.viewGroups` / `rbac.viewUsers` | view | — |
| `security.podSecurityEnforce` | — | Default: `restricted` |

### `charts/project-config` — ArgoCD AppProject

Generiert ArgoCD-Rollen mit einheitlicher Terminologie:

| Parameter | ArgoCD-Rechte |
|---|---|
| `adminGroups` / `adminUsers` | `*` (Vollzugriff) |
| `editGroups` / `editUsers` | `get`, `sync`, `override` |
| `viewGroups` / `viewUsers` | `get` (read-only) |

---

## Teams

| Team | User |
|---|---|
| team-a | alice |
| team-b | bob |
| team-c | charlie |

## Projekte

| Projekt | Apps | team-a OpenShift / ArgoCD | team-b OpenShift / ArgoCD | team-c OpenShift / ArgoCD |
|---|---|---|---|---|
| project-a | my-app, your-app | admin / edit | edit / edit | view / view |
| project-b | my-app, your-app | admin / edit | edit / edit | view / view |
