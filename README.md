# ocp-workloads

Workloads-Repo für OpenShift. Verwaltet vom **Platform Team**.  
Entwickler-Teams erhalten Zugriff auf ihre jeweiligen App-Repos — nicht auf dieses Repo.

---

## Konzept

```
ocp-workloads/
├── charts/
│   └── namespace-config/        ← Helm Library Chart (Platform Team)
│                                   Templates für Namespace, Quota, NetPol, RBAC
└── apps/
    └── project-a/               ← je Projekt ein Verzeichnis
        ├── appproject.yaml      ← ArgoCD AppProject (Wave -1)
        ├── groups.yaml          ← Projekt-Gruppen (Wave  0)
        └── my-app/              ← je App ein Unterverzeichnis
            ├── namespace-config-app.yaml  ← Namespace-Konfiguration via Helm (Wave 0)
            ├── values.yaml                ← App-spezifische Werte
            └── my-app-app.yaml            ← Application → App-Repo Entwickler (Wave 1)
```

---

## Sync-Flow

```
workloads-app (aus ocp-platform)
└── apps/project-a/
    ├── appproject.yaml                    Wave -1  AppProject anlegen
    ├── groups.yaml                        Wave  0  Gruppen anlegen
    └── my-app/
        ├── namespace-config-app.yaml      Wave  0  Namespace, Quota, NetPol, RBAC
        └── my-app-app.yaml                Wave  1  eigentliche App deployen
```

---

## Verantwortlichkeiten

| Wer | Was |
|---|---|
| Platform Team | `charts/namespace-config/` — Library Chart pflegen |
| Platform Team | `apps/<project>/appproject.yaml` — AppProject anlegen |
| Platform Team | `apps/<project>/groups.yaml` — Projekt-Gruppen und Mitglieder |
| Platform Team | `apps/<project>/<app>/namespace-config-app.yaml` — Namespace-Config Application |
| Platform Team | `apps/<project>/<app>/values.yaml` — Werte für Namespace-Konfiguration |
| Platform Team | `apps/<project>/<app>/<app>-app.yaml` — Application auf App-Repo zeigen |
| Entwickler | Eigenes App-Repo (Helm Chart oder Manifeste) |

---

## User- und Gruppen-Management

Gruppen und ihre Mitglieder werden in Git verwaltet.  
Passwörter liegen **nicht in Git** — sie werden manuell im HTPasswd-Secret gepflegt.

### Mitglied zu Projekt-Gruppe hinzufügen

```bash
# apps/project-a/groups.yaml editieren:
# z.B. Entwickler zu project-a-developers hinzufügen
users:
  - vorhandener-user
  - neuer-entwickler    # neu

git add . && git commit -m "feat(project-a): add neuer-entwickler" && git push
```

ArgoCD übernimmt die Änderung automatisch.

### Passwort für neuen User anlegen (manuell, außerhalb Git)

```bash
oc get secret htpasswd-secret -n openshift-config \
  -o jsonpath='{.data.htpasswd}' | base64 -d > /tmp/htpasswd

htpasswd /tmp/htpasswd neuer-entwickler

oc create secret generic htpasswd-secret \
  --from-file=htpasswd=/tmp/htpasswd \
  -n openshift-config \
  --dry-run=client -o yaml | oc apply -f -

rm /tmp/htpasswd
```

---

## Neues Projekt anlegen

### 1. Verzeichnis und Pflichtdateien anlegen

```bash
mkdir -p apps/project-b

# Folgende Dateien anlegen (Vorlage: apps/project-a/):
# - appproject.yaml   AppProject "project-b"
# - groups.yaml       project-b-admins, project-b-developers
```

### 2. Erste App im Projekt anlegen

```bash
mkdir -p apps/project-b/my-first-app

# Folgende Dateien anlegen (Vorlage: apps/project-a/my-app/):
# - namespace-config-app.yaml
# - values.yaml
# - my-first-app-app.yaml
```

### 3. App-Repo in AppProject eintragen

In `apps/project-b/appproject.yaml` unter `sourceRepos`:

```yaml
sourceRepos:
  - https://github.com/chriwo42-lang/ocp-workloads.git
  - https://github.com/chriwo42-lang/my-first-app.git
```

### 4. Commit & Push

```bash
git add . && git commit -m "feat: add project-b with my-first-app" && git push
```

ArgoCD deployt automatisch.

---

## Neue App zu bestehendem Projekt hinzufügen

```bash
mkdir -p apps/project-a/second-app

# Dateien anlegen:
# - namespace-config-app.yaml  (Vorlage: apps/project-a/my-app/namespace-config-app.yaml)
# - values.yaml                (Vorlage: apps/project-a/my-app/values.yaml)
# - second-app-app.yaml        (Vorlage: apps/project-a/my-app/my-app-app.yaml)

# App-Repo in AppProject eintragen:
# apps/project-a/appproject.yaml → sourceRepos ergänzen

git add . && git commit -m "feat(project-a): add second-app" && git push
```

---

## Helm Library Chart: namespace-config

Siehe [charts/namespace-config/values.yaml](charts/namespace-config/values.yaml) für alle verfügbaren Werte und Defaults.

Konfigurierbar je App über `values.yaml`:

| Bereich | Konfigurierbar |
|---|---|
| ResourceQuota | Pods, CPU, Memory, Services, Secrets, ConfigMaps |
| LimitRange | Default-Limits und Requests für Container |
| NetworkPolicy | Deny-All Basis, Ingress vom Router, zusätzliche Namespaces |
| RBAC | Admin/Edit/View-RoleBindings für Gruppen und einzelne User |

---

## Projekte

| Projekt | Gruppen | Apps |
|---|---|---|
| project-a | project-a-admins, project-a-developers | my-app |
