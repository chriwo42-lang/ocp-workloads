# ocp-workloads

Workloads-Repo für OpenShift. Verwaltet vom **Platform Team**.  
Entwickler-Teams erhalten Zugriff auf ihre jeweiligen App-Repos — nicht auf dieses Repo.

---

## Konzept

```
ocp-workloads/
├── groups/                  ← Globale Gruppen-Definitionen (eine Datei pro Gruppe)
│   ├── project-a-admins.yaml
│   ├── project-a-developers.yaml
│   ├── project-a-viewers.yaml
│   ├── project-b-admins.yaml
│   ├── project-b-developers.yaml
│   └── project-b-viewers.yaml
├── charts/
│   └── namespace-config/    ← Helm Chart für Namespace-Konfiguration
└── apps/
    ├── project-a/
    │   ├── appproject.yaml
    │   ├── my-app/
    │   │   ├── namespace-config-app.yaml
    │   │   ├── values.yaml
    │   │   └── my-app-app.yaml
    │   └── your-app/
    │       ├── namespace-config-app.yaml
    │       ├── values.yaml
    │       └── your-app-app.yaml
    └── project-b/
        ├── appproject.yaml
        ├── my-app/
        │   ├── namespace-config-app.yaml
        │   ├── values.yaml
        │   └── my-app-app.yaml
        └── your-app/
            ├── namespace-config-app.yaml
            ├── values.yaml
            └── your-app-app.yaml
```

---

## Sync-Flow

```
workloads-groups-app (aus ocp-platform)     workloads-app (aus ocp-platform)
└── groups/                                 ├── apps/project-a/
    ├── project-a-admins.yaml  Wave -1      │   ├── appproject.yaml      Wave -1
    ├── project-a-developers.yaml           │   ├── my-app/
    ├── project-a-viewers.yaml              │   │   ├── namespace-config  Wave  0
    ├── project-b-admins.yaml               │   │   └── my-app-app        Wave  1
    ├── project-b-developers.yaml           │   └── your-app/
    └── project-b-viewers.yaml             │       ├── namespace-config  Wave  0
                                            │       └── your-app-app      Wave  1
                                            └── apps/project-b/
                                                ├── appproject.yaml      Wave -1
                                                ├── my-app/
                                                │   ├── namespace-config  Wave  0
                                                │   └── my-app-app        Wave  1
                                                └── your-app/
                                                    ├── namespace-config  Wave  0
                                                    └── your-app-app      Wave  1
```

---

## Verantwortlichkeiten

| Wer | Was |
|---|---|
| Platform Team | `groups/` — Gruppen global definieren und Mitglieder pflegen |
| Platform Team | `charts/namespace-config/` — Helm Chart pflegen |
| Platform Team | `apps/<project>/appproject.yaml` — AppProject anlegen |
| Platform Team | `apps/<project>/<app>/namespace-config-app.yaml` — Namespace-Config Application |
| Platform Team | `apps/<project>/<app>/values.yaml` — Gruppennamen zuweisen |
| Platform Team | `apps/<project>/<app>/<app>-app.yaml` — Application auf App-Repo zeigen |
| Entwickler | Eigenes App-Repo (Helm Chart oder Manifeste) |

---

## Gruppen-Management

Gruppen werden **global** in `groups/` definiert — eine Datei pro Gruppe.  
Die **Zuweisung** zu Namespaces erfolgt in `apps/<project>/<app>/values.yaml` unter `rbac`.

### Neue Gruppe anlegen

```powershell
# Neue Datei in groups/ anlegen (Vorlage: groups/project-a-admins.yaml)
# Gruppenname in values.yaml der jeweiligen App unter rbac.adminGroups eintragen
git add . && git commit -m "feat(groups): add new-group"
git push
```

### Mitglied zu Gruppe hinzufügen

```powershell
# groups/<gruppenname>.yaml editieren:
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

### 1. Gruppen anlegen (je eine Datei pro Rolle)

```powershell
# groups/project-c-admins.yaml    (Vorlage: groups/project-a-admins.yaml)
# groups/project-c-developers.yaml
# groups/project-c-viewers.yaml
```

### 2. Projektverzeichnis und AppProject anlegen

```powershell
mkdir apps\project-c
# appproject.yaml anlegen (Vorlage: apps/project-a/appproject.yaml)
# Rollen: project-c-admin, project-c-developer, project-c-viewer
```

### 3. Apps anlegen

```powershell
mkdir apps\project-c\my-first-app
# namespace-config-app.yaml, values.yaml, my-first-app-app.yaml anlegen
# In values.yaml: rbac.adminGroups, editGroups, viewGroups setzen
```

### 4. Commit & Push

```powershell
git add . && git commit -m "feat: add project-c"
git push
```

---

## Helm Chart: namespace-config

Siehe [charts/namespace-config/values.yaml](charts/namespace-config/values.yaml) für alle Werte.

Gruppen werden in `values.yaml` nur **referenziert** — sie müssen bereits in `groups/` definiert sein:

```yaml
rbac:
  adminGroups:
    - project-a-admins      # muss in groups/project-a-admins.yaml existieren
  editGroups:
    - project-a-developers  # muss in groups/project-a-developers.yaml existieren
  viewGroups:
    - project-a-viewers     # muss in groups/project-a-viewers.yaml existieren
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
