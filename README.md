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
│   │   ├── appproject.yaml
│   │   ├── my-app/
│   │   │   ├── namespace-config-app.yaml
│   │   │   ├── values.yaml          ← Team-Zuweisung: adminGroups, editGroups, viewGroups
│   │   │   └── my-app-app.yaml
│   │   └── your-app/
│   │       ├── namespace-config-app.yaml
│   │       ├── values.yaml
│   │       └── your-app-app.yaml
│   └── project-b/
│       ├── appproject.yaml
│       ├── my-app/
│       │   ├── namespace-config-app.yaml
│       │   ├── values.yaml
│       │   └── my-app-app.yaml
│       └── your-app/
│           ├── namespace-config-app.yaml
│           ├── values.yaml
│           └── your-app-app.yaml
└── charts/
    └── namespace-config/            ← Helm Chart für Namespace-Konfiguration
        ├── Chart.yaml
        ├── values.yaml
        └── templates/
            ├── namespace.yaml
            ├── resourcequota.yaml
            ├── networkpolicy.yaml
            └── rbac.yaml
```

---

## Sync-Flow

```
workloads-app (aus ocp-platform, recurse: true auf apps/)
├── groups/
│   ├── team-a.yaml                  Wave -1
│   └── team-b.yaml                  Wave -1
├── project-a/
│   ├── appproject.yaml              Wave -1
│   ├── my-app/
│   │   ├── namespace-config-app     Wave  0
│   │   └── my-app-app               Wave  1
│   └── your-app/
│       ├── namespace-config-app     Wave  0
│       └── your-app-app             Wave  1
└── project-b/
    ├── appproject.yaml              Wave -1
    ├── my-app/
    │   ├── namespace-config-app     Wave  0
    │   └── my-app-app               Wave  1
    └── your-app/
        ├── namespace-config-app     Wave  0
        └── your-app-app             Wave  1
```

> `charts/` wird **nicht** von `workloads-app` deployt — es wird als Helm-Source
> direkt in den `namespace-config-app` Applications referenziert.

---

## Verantwortlichkeiten

| Wer | Was |
|---|---|
| Platform Team | `apps/groups/` — Teams und Mitglieder pflegen |
| Platform Team | `charts/namespace-config/` — Helm Chart pflegen |
| Platform Team | `apps/<project>/appproject.yaml` — AppProject anlegen |
| Platform Team | `apps/<project>/<app>/namespace-config-app.yaml` — Namespace-Config Application |
| Platform Team | `apps/<project>/<app>/values.yaml` — Team-Zuweisung (admin/edit/view) |
| Platform Team | `apps/<project>/<app>/<app>-app.yaml` — Application auf App-Repo zeigen |
| Entwickler | Eigenes App-Repo (Helm Chart oder Manifeste) |

---

## Team-Management

Teams werden **global** in `apps/groups/` definiert — eine Datei pro Team.  
Die **Zuweisung** als admin/editor/viewer erfolgt je App in `values.yaml` unter `rbac`.

### Konzept

```
apps/groups/team-a.yaml        ← Team-Definition (wer ist Mitglied)
apps/project-a/my-app/values.yaml:
  rbac:
    adminGroups: [team-a]      ← team-a ist Admin in diesem Namespace
    editGroups:  [team-b]      ← team-b ist Editor
    viewGroups:  []
```

### Neues Team anlegen

```powershell
# apps/groups/team-c.yaml anlegen (Vorlage: apps/groups/team-a.yaml)
git add . && git commit -m "feat(groups): add team-c"
git push
```

### Mitglied zu Team hinzufügen

```powershell
# apps/groups/team-a.yaml editieren:
# users:
#   - vorhandener-user
#   - neuer-user
git add . && git commit -m "feat(groups): add neuer-user to team-a"
git push
```

### Team einer App zuweisen

In `apps/<project>/<app>/values.yaml`:

```yaml
rbac:
  adminGroups:
    - team-a    # admin-RoleBinding im Namespace
  editGroups:
    - team-b    # edit-RoleBinding im Namespace
  viewGroups:
    - team-c    # view-RoleBinding im Namespace
```

### Passwort für neuen User anlegen (manuell, außerhalb Git)

```powershell
oc get secret htpasswd-secret -n openshift-config `
  -o jsonpath='{.data.htpasswd}' | `
  [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) | `
  Out-File -FilePath "$env:TEMP\htpasswd" -Encoding utf8NoBOM

# Hash generieren: https://bcrypt-generator.com (Rounds 10)
Add-Content "$env:TEMP\htpasswd" 'neuer-user:$2a$10$HASH_HIER'

oc create secret generic htpasswd-secret `
  --from-file=htpasswd="$env:TEMP\htpasswd" `
  -n openshift-config `
  --dry-run=client -o yaml | oc apply -f -

Remove-Item "$env:TEMP\htpasswd"
```

---

## Neues Projekt anlegen

### 1. AppProject und Apps anlegen

```powershell
mkdir apps\project-c\my-first-app
# apps/project-c/appproject.yaml       (Vorlage: apps/project-a/appproject.yaml)
# apps/project-c/my-first-app/namespace-config-app.yaml
# apps/project-c/my-first-app/values.yaml  ← Teams unter rbac zuweisen
# apps/project-c/my-first-app/my-first-app-app.yaml
```

### 2. Commit & Push

```powershell
git add . && git commit -m "feat: add project-c"
git push
```

> Neue Teams bei Bedarf vorher in `apps/groups/` anlegen.

---

## Helm Chart: namespace-config

Siehe [charts/namespace-config/values.yaml](charts/namespace-config/values.yaml) für alle Werte.

Teams werden in `values.yaml` nur **referenziert** — sie müssen bereits in `apps/groups/` definiert sein:

```yaml
rbac:
  adminGroups:
    - team-a    # definiert in apps/groups/team-a.yaml
  editGroups:
    - team-b    # definiert in apps/groups/team-b.yaml
```

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
