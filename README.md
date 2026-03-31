# ocp-workloads

Workloads-Repo für OpenShift. Verwaltet vom **Platform Team**.  
Entwickler-Teams erhalten Zugriff auf ihre jeweiligen App-Repos — nicht auf dieses Repo.

---

## Konzept

```
ocp-workloads/
├── apps/
│   ├── groups/                      ← Team-Definitionen (eine Datei pro Team)
│   │   ├── team-a.yaml              (developer — admin in project-a/b)
│   │   ├── team-b.yaml              (editor — edit in project-a/b)
│   │   └── team-c.yaml              (readonly — view in project-a/b)
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

Zwei unabhängige Berechtigungsebenen:

| Ebene | Was | Konfiguriert in |
|---|---|---|
| **OpenShift** | Namespace-Zugriff (kubectl/oc) | `app/values.yaml` → `rbac.*` |
| **ArgoCD** | UI-Zugriff auf Applications | `appproject-values.yaml` → `developerTeams`, `viewerTeams` |

### Berechtigungsmatrix (Test-Setup)

| User | Team | OpenShift Namespace | ArgoCD |
|---|---|---|---|
| `admin` | cluster-admins | cluster-admin | admin (alle Projekte) |
| `developer` | team-a | admin in project-a/b Namespaces | developer in project-a/b |
| `editor` | team-b | edit in project-a/b Namespaces | developer in project-a/b |
| `readonly` | team-c | view in project-a/b Namespaces | viewer in project-a/b |

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

Teams werden **global** in `apps/groups/` definiert — eine Datei pro Team.

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
  - developer
  - neuer-user
```

```powershell
git add . && git commit -m "feat(groups): add neuer-user to team-a"
git push
```

**2. Passwort direkt im Secret anlegen** (kein Tempfile):

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
#   developerTeams: [team-a]
#   viewerTeams: [team-c]
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
#     podSecurityEnforce: restricted   # oder baseline für Legacy-Apps
#   appRepo:
#     url: https://github.com/chriwo42-lang/new-app.git
#   rbac:
#     adminGroups: [team-a]
#     editGroups:  [team-b]
#     viewGroups:  [team-c]
```

### 3. Commit & Push

```powershell
git add . && git commit -m "feat: add project-c with new-app"
git push
```

---

## Helm Charts

### `charts/app-config`

Deployt direkt in den Ziel-Namespace:
- `Namespace` mit Labels (inkl. Pod Security Admission) und Annotations
- `ResourceQuota` + `LimitRange`
- `NetworkPolicy` (Deny-All Basis, konfigurierbare Ausnahmen)
- `RoleBindings` für Teams (admin/edit/view im OpenShift Namespace)
- ArgoCD `Application` für das App-Repo der Entwickler

| Parameter | Beschreibung | Default |
|---|---|---|
| `app` | App-Name (**Pflicht**) | — |
| `project` | Projektname (**Pflicht**) | — |
| `appRepo.url` | Git-URL (**Pflicht**) | — |
| `security.podSecurityEnforce` | Pod Security Level | `restricted` |
| `security.podSecurityAudit` | Pod Security Audit Level | `restricted` |
| `security.podSecurityWarn` | Pod Security Warn Level | `restricted` |
| `quota.*` | ResourceQuota | siehe values.yaml |
| `limitRange.*` | LimitRange | siehe values.yaml |
| `networkPolicy.*` | NetworkPolicy | alle allow-Flags true |
| `rbac.*` | Team-Zuweisungen | leer |

### `charts/project-config`

Generiert ein ArgoCD `AppProject` mit drei Rollen:

| Rolle | ArgoCD-Rechte | Konfiguriert via |
|---|---|---|
| `*-admin` | Vollzugriff | `adminGroups` |
| `*-developer` | get, sync, override | `developerTeams` |
| `*-viewer` | get (read-only) | `viewerTeams` |

---

## Teams

| Team | User | Zweck |
|---|---|---|
| team-a | developer | Admin in project-a/b Namespaces |
| team-b | editor | Editor in project-a/b Namespaces |
| team-c | readonly | Viewer in project-a/b Namespaces |

## Projekte

| Projekt | Apps | team-a | team-b | team-c |
|---|---|---|---|---|
| project-a | my-app, your-app | admin / ArgoCD developer | edit / ArgoCD developer | view / ArgoCD viewer |
| project-b | my-app, your-app | admin / ArgoCD developer | edit / ArgoCD developer | view / ArgoCD viewer |
