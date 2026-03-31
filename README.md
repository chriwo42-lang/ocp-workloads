# ocp-workloads

Workloads-Repo für OpenShift. Verwaltet vom **Platform Team**.  
Entwickler-Teams erhalten Zugriff auf ihre jeweiligen App-Repos — nicht auf dieses Repo.

---

## Konzept

```
ocp-workloads/
├── apps/
│   ├── groups/                      ← Team-Definitionen (eine Datei pro Team)
│   │   ├── team-a.yaml
│   │   └── team-b.yaml
│   ├── project-a/
│   │   ├── appproject-app.yaml      ← Application → project-config Chart
│   │   ├── appproject-values.yaml   ← project-a: name, sourceRepos
│   │   ├── my-app/
│   │   │   ├── app.yaml             ← Application → app-config Chart
│   │   │   └── values.yaml          ← app, project, appRepo, quota, rbac, ...
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
    │   ├── Chart.yaml               │  + ArgoCD Application für das App-Repo
    │   ├── values.yaml
    │   └── templates/
    │       ├── namespace.yaml
    │       ├── resourcequota.yaml
    │       ├── networkpolicy.yaml
    │       ├── rbac.yaml
    │       └── application.yaml
    └── project-config/              ← Helm Chart: ArgoCD AppProject
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
│   └── team-b.yaml                  Wave -1
├── project-a/
│   ├── appproject-app.yaml          Wave -1  → deployt AppProject via project-config Chart
│   ├── my-app/
│   │   └── app.yaml                 Wave  0  → deployt via app-config Chart:
│   │                                           Namespace, Quota, NetPol, RBAC
│   │                                           + Application für App-Repo
│   └── your-app/
│       └── app.yaml                 Wave  0
└── project-b/
    ├── appproject-app.yaml          Wave -1
    ├── my-app/
    │   └── app.yaml                 Wave  0
    └── your-app/
        └── app.yaml                 Wave  0
```

> `charts/` wird **nicht** von `workloads-app` deployt — die Charts werden als
> Helm-Source direkt in den `app.yaml` / `appproject-app.yaml` referenziert.

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
Die **Zuweisung** als admin/editor/viewer erfolgt je App in `values.yaml` unter `rbac`.

### Neues Team anlegen

```powershell
# apps/groups/team-c.yaml anlegen (Vorlage: apps/groups/team-a.yaml)
git add . && git commit -m "feat(groups): add team-c"
git push
```

### Mitglied zu Team hinzufügen

```powershell
# apps/groups/team-a.yaml editieren
git add . && git commit -m "feat(groups): add neuer-user to team-a"
git push
```

### Passwort für neuen User anlegen (manuell, außerhalb Git)

```powershell
oc get secret htpasswd-secret -n openshift-config `
  -o jsonpath='{.data.htpasswd}' | `
  [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) | `
  Out-File -FilePath "$env:TEMP\htpasswd" -Encoding utf8NoBOM

Add-Content "$env:TEMP\htpasswd" 'neuer-user:$2a$10$HASH_HIER'

oc create secret generic htpasswd-secret `
  --from-file=htpasswd="$env:TEMP\htpasswd" `
  -n openshift-config `
  --dry-run=client -o yaml | oc apply -f -

Remove-Item "$env:TEMP\htpasswd"
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
```

### 2. App anlegen

```powershell
mkdir apps\project-c\new-app

# app.yaml     (Vorlage: apps/project-a/my-app/app.yaml)
#   → destination.namespace: project-c-new-app anpassen
# values.yaml:
#   app: new-app
#   project: project-c
#   appRepo:
#     url: https://github.com/chriwo42-lang/new-app.git
#   rbac:
#     adminGroups: [team-a]
#     editGroups:  [team-b]
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
- `Namespace` mit Labels und Annotations
- `ResourceQuota` + `LimitRange`
- `NetworkPolicy` (Deny-All Basis, konfigurierbare Ausnahmen)
- `RoleBindings` für Teams (admin/edit/view)
- ArgoCD `Application` für das App-Repo der Entwickler

Konfigurierbar via `values.yaml`:

| Parameter | Beschreibung | Pflicht |
|---|---|---|
| `app` | App-Name | ✅ |
| `project` | Projektname | ✅ |
| `appRepo.url` | Git-URL des App-Repos | ✅ |
| `appRepo.targetRevision` | Branch/Tag (Default: main) | — |
| `appRepo.path` | Pfad zum Helm Chart (Default: helm) | — |
| `displayName` | Anzeigename für den Namespace | — |
| `quota.*` | ResourceQuota Werte | — |
| `limitRange.*` | LimitRange Werte | — |
| `networkPolicy.*` | NetworkPolicy Konfiguration | — |
| `rbac.*` | Team-Zuweisungen | — |

### `charts/project-config`

Generiert ein ArgoCD `AppProject`. Konfigurierbar via `appproject-values.yaml`:
`project`, `description`, `sourceRepos`, `adminGroups`.

---

## Teams

| Team | Mitglieder |
|---|---|
| team-a | developer |
| team-b | — |

## Projekte

| Projekt | Apps | team-a | team-b |
|---|---|---|---|
| project-a | my-app, your-app | admin | edit |
| project-b | my-app, your-app | admin | edit |
