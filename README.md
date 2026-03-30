# ocp-workloads

Workloads-Repo für OpenShift. Verwaltet vom **Platform Team**.  
Entwickler-Teams erhalten Zugriff auf ihre jeweiligen App-Repos — nicht auf dieses Repo.

---

## Konzept

```
ocp-workloads/
├── groups/                  ← Globale Gruppen-Definitionen (eine Datei pro Gruppe)
│   ├── project-a-admins.yaml
│   └── project-a-developers.yaml
├── charts/
│   └── namespace-config/    ← Helm Chart für Namespace-Konfiguration
└── apps/
    └── project-a/           ← je Projekt ein Verzeichnis
        ├── appproject.yaml  ← ArgoCD AppProject (Wave -1)
        ├── my-app/          ← je App ein Unterverzeichnis
        │   ├── namespace-config-app.yaml  ← Namespace, Quota, NetPol, RBAC (Wave 0)
        │   ├── values.yaml                ← referenziert Gruppennamen
        │   └── my-app-app.yaml            ← Application → App-Repo (Wave 1)
        └── your-app/
            ├── namespace-config-app.yaml
            ├── values.yaml
            └── your-app-app.yaml
```

---

## Sync-Flow

```
workloads-groups-app (aus ocp-platform)     workloads-app (aus ocp-platform)
└── groups/                                 └── apps/project-a/
    ├── project-a-admins.yaml  Wave -1          ├── appproject.yaml    Wave -1
    └── project-a-developers.yaml               ├── my-app/
                                                │   ├── namespace-config-app  Wave 0
                                                │   └── my-app-app            Wave 1
                                                └── your-app/
                                                    ├── namespace-config-app  Wave 0
                                                    └── your-app-app          Wave 1
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

### 1. Gruppen anlegen

```powershell
# groups/project-b-admins.yaml anlegen (Vorlage: groups/project-a-admins.yaml)
# groups/project-b-developers.yaml anlegen
```

### 2. Projektverzeichnis und AppProject anlegen

```powershell
mkdir apps\project-b
# appproject.yaml anlegen (Vorlage: apps/project-a/appproject.yaml)
```

### 3. Erste App anlegen

```powershell
mkdir apps\project-b\my-first-app
# namespace-config-app.yaml, values.yaml, my-first-app-app.yaml anlegen
# In values.yaml: rbac.adminGroups: [project-b-admins]
```

### 4. Commit & Push

```powershell
git add . && git commit -m "feat: add project-b"
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
```

---

## Gruppen

| Gruppe | Mitglieder | Zugewiesen in |
|---|---|---|
| project-a-admins | — | project-a-my-app, project-a-your-app |
| project-a-developers | — | project-a-my-app, project-a-your-app |

## Projekte

| Projekt | Gruppen | Apps |
|---|---|---|
| project-a | project-a-admins, project-a-developers | my-app, your-app |
