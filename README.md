# ocp-workloads

Workloads-Repo für OpenShift. Verwaltet vom **Platform Team**.  
Entwickler-Teams erhalten Zugriff auf ihre jeweiligen App-Repos — nicht auf dieses Repo.

---

## Konzept

```
ocp-workloads/
├── apps/
│   ├── groups/                      ← Globale Gruppen (je Projekt ein Unterverzeichnis)
│   │   ├── project-a/
│   │   │   ├── admins.yaml
│   │   │   ├── developers.yaml
│   │   │   └── viewers.yaml
│   │   └── project-b/
│   │       ├── admins.yaml
│   │       ├── developers.yaml
│   │       └── viewers.yaml
│   ├── project-a/
│   │   ├── appproject.yaml
│   │   ├── my-app/
│   │   │   ├── namespace-config-app.yaml
│   │   │   ├── values.yaml
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
```

---

## Sync-Flow

```
workloads-app (aus ocp-platform, recurse: true)
├── groups/
│   ├── project-a/admins.yaml        Wave -1
│   ├── project-a/developers.yaml    Wave -1
│   ├── project-a/viewers.yaml       Wave -1
│   ├── project-b/admins.yaml        Wave -1
│   ├── project-b/developers.yaml    Wave -1
│   └── project-b/viewers.yaml       Wave -1
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

---

## Verantwortlichkeiten

| Wer | Was |
|---|---|
| Platform Team | `apps/groups/` — Gruppen global definieren und Mitglieder pflegen |
| Platform Team | `charts/namespace-config/` — Helm Chart pflegen |
| Platform Team | `apps/<project>/appproject.yaml` — AppProject anlegen |
| Platform Team | `apps/<project>/<app>/namespace-config-app.yaml` — Namespace-Config Application |
| Platform Team | `apps/<project>/<app>/values.yaml` — Gruppennamen zuweisen |
| Platform Team | `apps/<project>/<app>/<app>-app.yaml` — Application auf App-Repo zeigen |
| Entwickler | Eigenes App-Repo (Helm Chart oder Manifeste) |

---

## Gruppen-Management

Gruppen werden **global** in `apps/groups/<project>/` definiert — eine Datei pro Rolle.  
Die **Zuweisung** zu Namespaces erfolgt in `apps/<project>/<app>/values.yaml` unter `rbac`.

### Mitglied zu Gruppe hinzufügen

```powershell
# apps/groups/<project>/<rolle>.yaml editieren:
# users:
#   - vorhandener-user
#   - neuer-user
git add . && git commit -m "feat(groups): add neuer-user to project-a-developers"
git push
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

### 1. Gruppen anlegen

```powershell
mkdir apps\groups\project-c
# apps/groups/project-c/admins.yaml    (Vorlage: apps/groups/project-a/admins.yaml)
# apps/groups/project-c/developers.yaml
# apps/groups/project-c/viewers.yaml
```

### 2. AppProject und Apps anlegen

```powershell
mkdir apps\project-c\my-first-app
# apps/project-c/appproject.yaml
# apps/project-c/my-first-app/namespace-config-app.yaml
# apps/project-c/my-first-app/values.yaml
# apps/project-c/my-first-app/my-first-app-app.yaml
```

### 3. Commit & Push

```powershell
git add . && git commit -m "feat: add project-c"
git push
```

---

## Helm Chart: namespace-config

Siehe [charts/namespace-config/values.yaml](charts/namespace-config/values.yaml) für alle Werte.

Gruppen werden in `values.yaml` nur **referenziert** — sie müssen bereits in `apps/groups/<project>/` definiert sein:

```yaml
rbac:
  adminGroups:
    - project-a-admins      # definiert in apps/groups/project-a/admins.yaml
  editGroups:
    - project-a-developers  # definiert in apps/groups/project-a/developers.yaml
  viewGroups:
    - project-a-viewers     # definiert in apps/groups/project-a/viewers.yaml
```

---

## Gruppen

| Gruppe | ArgoCD-Rolle | OpenShift-Rolle | Mitglieder | Namespaces |
|---|---|---|---|---|
| project-a-admins | project-a-admin | admin | — | project-a-* |
| project-a-developers | project-a-developer | edit | developer | project-a-* |
| project-a-viewers | project-a-viewer | view | — | project-a-* |
| project-b-admins | project-b-admin | admin | — | project-b-* |
| project-b-developers | project-b-developer | edit | developer | project-b-* |
| project-b-viewers | project-b-viewer | view | — | project-b-* |

## Projekte

| Projekt | Gruppen | Apps |
|---|---|---|
| project-a | project-a-admins, project-a-developers, project-a-viewers | my-app, your-app |
| project-b | project-b-admins, project-b-developers, project-b-viewers | my-app, your-app |
