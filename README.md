# Kubernetes GitOps — ArgoCD App of Apps + sdílený Helm chart

> **TL;DR** — Tento repozitář ukazuje, jak deployovat ~10 podobných HTTP aplikací
> do Kubernetes přes ArgoCD tak, aby přidání 11. aplikace znamenalo **jeden
> nový řádek** ve values souboru a nic víc. Stojí na dvou pilířích:
> jeden univerzální Helm chart (`charts/generic-app`) a jeden řídicí Helm chart
> (`argocd-apps`) implementující pattern **App of Apps**.

---

## Obsah

1. [Architektura ve zkratce](#architektura-ve-zkratce)
2. [Struktura repozitáře](#struktura-repozitáře)
3. [Jak to spustit](#jak-to-spustit)
4. [Jak přidat 11. aplikaci](#jak-přidat-11-aplikaci)
5. [Proč tento přístup (rozhodnutí a odůvodnění)](#proč-tento-přístup-rozhodnutí-a-odůvodnění)
6. [Omezení tohoto repozitáře](#omezení-tohoto-repozitáře)
7. [Co chybí pro produkci (Infrastructure & Security)](#co-chybí-pro-produkci-infrastructure--security)

---

## Architektura ve zkratce

```
                       ┌───────────────────────────────┐
   kubectl apply  ───▶ │  root-<env>  (Application)    │   (1 ručně per env)
                       └──────────────┬────────────────┘
                                      │ Helm renders
                                      ▼
                       ┌───────────────────────────────┐
                       │  argocd-apps  (Helm chart)    │
                       │  templates/applications.yaml  │
                       │  range .Values.applications   │
                       └──────────────┬────────────────┘
                                      │ generuje N×
                                      ▼
              ┌────────────┐  ┌────────────┐  ┌────────────┐
              │ Application│  │ Application│  │ Application│   (N child apps)
              │ payments-* │  │ accounts-* │  │ cards-*    │
              └─────┬──────┘  └─────┬──────┘  └─────┬──────┘
                    │               │               │
                    ▼               ▼               ▼
              charts/generic-app  (jeden sdílený chart pro všechny)
                    │
                    ▼
              Deployment + Service + Ingress  v cílovém namespace
```

**Flow v jedné větě:** root Application načte env-specifické `values.yaml`,
předá je App of Apps chartu, ten vygeneruje samostatný ArgoCD `Application`
objekt pro každou aplikaci, a každý child Application renderuje sdílený
`generic-app` chart s parametry konkrétní aplikace.

---

## Struktura repozitáře

```
.
├── README.md
├── bootstrap/                      # Root Applications (jediné, co se aplikuje ručně)
│   ├── root-app-dev.yaml
│   ├── root-app-stage.yaml
│   └── root-app-prod.yaml
│
├── charts/
│   └── generic-app/                # Sdílený chart pro VŠECHNY aplikace
│       ├── Chart.yaml
│       ├── values.yaml             # Výchozí (rozumné) hodnoty
│       └── templates/
│           ├── _helpers.tpl
│           ├── deployment.yaml
│           ├── service.yaml
│           └── ingress.yaml
│
├── argocd-apps/                    # App of Apps chart
│   ├── Chart.yaml
│   ├── values.yaml                 # Default schema (pole `applications: []`)
│   └── templates/
│       └── applications.yaml       # {{ range }} generující N × Application
│
└── envs/                           # Per-environment konfigurace
    ├── dev/values.yaml             # 2 aplikace, 1 replika
    ├── stage/values.yaml           # 2 aplikace, 2 repliky, env anotace
    └── prod/values.yaml            # 3 aplikace, 3 repliky, compliance anotace
```

---

## Jak to spustit

### Prerekvizity

- Kubernetes cluster (kind / k3d / EKS / AKS / GKE) s `kubectl` přístupem.
- Nainstalovaný **ArgoCD** v namespace `argocd` (verze ≥ 2.6 kvůli
  `valuesObject` a multi-source Applications).
- `helm` ≥ 3.10 lokálně (pro `helm template` ověření, viz níže).

### Lokální ověření (bez clusteru)

Vyrenderuj manifesty a podívej se, co by ArgoCD vytvořilo:

```bash
# Co App of Apps vygeneruje pro DEV
helm template root argocd-apps/ -f envs/dev/values.yaml

# Co generic-app vyrenderuje pro jednu aplikaci (s prod hodnotami)
helm template payments-api charts/generic-app/ \
  --set nameOverride=payments-api \
  --set replicas=3 \
  --set image.repository=ghcr.io/example-org/payments-api \
  --set image.tag=1.4.2 \
  --set ingress.enabled=true \
  --set ingress.host=payments-api.prod.example.cluster.local
```

### Nasazení do clusteru

1. **Uprav `repoURL`** v `bootstrap/root-app-*.yaml` na svůj fork repozitáře.
2. **Aplikuj root Application** pro dané prostředí:

   ```bash
   kubectl apply -f bootstrap/root-app-dev.yaml
   ```

3. **Hotovo.** ArgoCD si zbytek vytvoří sám:
   - root Application → vytvoří N child Application objektů,
   - každý child Application → nasadí Deployment + Service + Ingress
     do `apps-dev` namespace.

4. Stav synchronizace zkontroluješ v UI (`argocd-server`) nebo přes CLI:

   ```bash
   argocd app list
   argocd app get root-dev
   ```

---

## Jak přidat 11. aplikaci

To je důkaz, že princip funguje. Stačí jeden záznam:

```yaml
# envs/prod/values.yaml
applications:
  # ... existující ...

  - name: loans-api               # nová 11. aplikace
    replicas: 3
    image:
      repository: ghcr.io/example-org/loans-api
      tag: "0.1.0"
    ingress:
      enabled: true
```

Commit → push → ArgoCD reconciluje → nová `Application` `loans-api-prod`
existuje a tahá manifesty. Žádné kopírování YAML, žádný nový chart,
žádný nový pipeline step.

---

## Proč tento přístup (rozhodnutí a odůvodnění)

### 1. Proč Helm a ne raw YAML / Kustomize

| Kritérium | raw YAML | Kustomize | **Helm** |
|---|---|---|---|
| 10× kopírovat Deployment | ❌ duplikace | částečně | ✅ jeden template |
| Per-env diff (3 prostředí) | ❌ × 30 souborů | overlays | ✅ values.yaml |
| Conditional resources (Ingress) | ❌ | omezené | ✅ `{{ if }}` |
| Familiar v ArgoCD | ✅ | ✅ | ✅ |

Pro 10 **podobných** aplikací je Helm jasně nejlepší — všechny sdílejí
stejnou strukturu (Deployment + Service + Ingress) a liší se jen
parametricky (image, replicas, host). Přesně na to jsou values souboty.

### 2. Proč App of Apps a **ne** ApplicationSet

Toto byla nejtěžší volba a zaslouží si vysvětlení.

**ApplicationSet** je modernější a má elegantní generátory (Git, List, Cluster).
Pro jednoduché *cookie-cutter* scénáře (jedna aplikace × N clusterů) je lepší
volba. **App of Apps** jsem zvolil úmyslně z těchto důvodů:

| Důvod | Detail |
|---|---|
| **Per-aplikační odchylky** | App of Apps generujeme přes Helm `range`, takže jednotlivé aplikace mohou mít libovolně různé hodnoty (resources, env, sidecary). ApplicationSet generátory jsou méně flexibilní u "ten samý template, ale tahle jedna aplikace potřebuje navíc X". |
| **Auditovatelnost** | `helm template` lokálně vyrenderuje **přesně** to, co ArgoCD vytvoří. ApplicationSet controller renderuje až v clusteru a debugging je opruz. |
| **Žádný extra controller** | ApplicationSet je separátní controller, který musí být nainstalovaný a updatovaný. App of Apps používá jen `Application` CRD, které je core ArgoCD. Menší attack surface, méně závislostí. |
| **Bank-friendly determinismus** | V bankovním prostředí preferuju "co commitnu, to dostanu" před implicitní generací. App of Apps je v tomhle čitelnější pro auditora. |

ApplicationSet je technicky validní volba — v rozhovoru ji rád obhájím
jako "co bych zvolil, kdyby šlo o jednu aplikaci nasazovanou do 50 clusterů".

### 3. DRY — co se NEpíše víckrát

- **Manifesty Deployment/Service/Ingress** — existují **jednou** v `generic-app`.
- **Definice ArgoCD Application** — existuje **jednou** v `argocd-apps/templates`.
- **Konfigurace per prostředí** — existuje **jednou** v `envs/<env>/values.yaml`.

Přidání aplikace = +1 záznam. Přidání prostředí = +1 soubor + +1 root Application.
Přidání nového typu resource (např. ServiceMonitor) = +1 template v `generic-app`.

### 4. Proč `valuesObject` a multi-source root Application

- `helm.valuesObject` (ArgoCD ≥ 2.6) předává values jako **strukturované YAML**,
  ne string. Žádné záludnosti s indentací uvnitř `values: |`.
- Multi-source root Application čte `argocd-apps/` z jednoho `source` a
  `envs/<env>/values.yaml` z druhého (oba ze stejného repa). Tím se vyhneme
  `../envs/...` cestám, které ArgoCD z bezpečnostních důvodů blokuje
  (`--allow-oob-symlinks`).

---

## Omezení tohoto repozitáře

Tento repozitář je **technická ukázka pro pohovor**, ne produkční platforma.
Vědomě obsahuje následující zjednodušení:

- **Žádné reálné credentials** — všechny image registry, hosts, tokeny apod.
  jsou placeholdery (`ghcr.io/example-org/...`, `example.cluster.local`).
- **Žádné reálné aplikace** — `payments-api`, `accounts-api`, `cards-api`
  jsou jen jména. Image se nikam reálně netáhne.
- **Žádné sekrety v Gitu** — vůbec. Sekrety nepatří do Gitu, viz sekce níže.
- **`automated.selfHeal: true` i v produkci** — v reálu by produkční sync
  byl manuální nebo přes change-management gate.
- **Žádné NetworkPolicies, PodSecurityStandards, ResourceQuotas** — patří
  na úroveň namespace/clusteru, ne do `generic-app` chartu.
- **Žádné testy chartu** (`helm unittest`, `conftest`, `kubeconform`) — měly
  by být v CI, viz sekce níže.
- **Single cluster destination** — `https://kubernetes.default.svc` všude.
  V reálu by každé prostředí typicky cílilo na jiný cluster (jiný `server:`).

---

## Co chybí pro produkci (Infrastructure & Security)

Cílem tohoto repozitáře je **deployment aplikací**. Aby šlo o produkční
platformu, je potřeba kolem něj ještě následující — záměrně **mimo** tento
repozitář, protože jde o jiný change-management cyklus a jiné role.

### 1. Infrastruktura — samostatný Terraform repozitář

Tento Git repo dělá **runtime** (co běží v K8s). Pod ním musí ležet:

- **Kubernetes cluster** — AKS / EKS / GKE provisioning, node pools,
  VNet/VPC peering, identity (Workload Identity / IRSA).
- **Object storage** — S3 / Azure Blob buckety pro logy, backupy,
  Helm cache, ArgoCD repo cache.
- **Container registry** — ACR / ECR / GAR s RBAC pro CI a pull-secrets
  pro K8s service accounts.
- **Key Vault / Secrets Manager** — Azure Key Vault / AWS Secrets Manager
  jako jediný zdroj pravdy pro tajemství.
- **DNS + TLS** — Route53 / Azure DNS zóny, cert-manager + Let's Encrypt
  nebo enterprise CA pro mTLS.
- **Identity Provider integrace** — Entra ID / Okta pro ArgoCD SSO a
  kubectl `--exec` auth.
- **Sítě a firewall** — privátní cluster API, egress NAT, WAF před Ingress.

Tohle by ležel v **samostatném Terraform repu** (např. `infra-terraform/`)
s vlastním state backendem (Terraform Cloud / Azure Storage backend),
review gate a deploy přes Atlantis / TF Cloud / GitHub Actions.

Rozdělení důvodů:
- **Jiný cyklus změn** — infra se mění zřídka, runtime denně.
- **Jiné role** — infra mění platform team, runtime tým aplikací.
- **Jiné blast radius** — `terraform apply` může smazat cluster;
  GitOps merge max rozbije jednu aplikaci.

### 2. Sekrety — External Secrets Operator + Azure Key Vault

V Gitu **nesmí** být žádný sekret (heslo k DB, JWT secret, API klíč
do třetí strany). Pattern:

```
Azure Key Vault                External Secrets Operator         K8s Secret
─────────────────              ─────────────────────────         ─────────────
  payments-db-password   ───▶  ExternalSecret CR (v Gitu)  ───▶  payments-db-creds
  payments-jwt-secret               namespace=apps-prod          (in cluster only)
```

- **External Secrets Operator** (ESO) běží v K8s a syncuje sekrety z
  Key Vaultu do K8s `Secret` objektů na základě `ExternalSecret` CRD.
- `ExternalSecret` manifesty leží v Gitu (žádná tajná data, jen
  *odkazy* na cesty v Key Vaultu) → ideální pro GitOps.
- Autentizace ESO → Key Vault přes **Workload Identity** (Azure) nebo
  **IRSA** (AWS). Žádné `clientSecret` v Gitu.
- Rotace sekretů → ESO automaticky resyncne, ArgoCD nemusí dělat nic.

V `generic-app` chartu by přibyla šablona `externalsecret.yaml` (zapínaná
přes `values: externalSecrets: [...]`), která by vygenerovala
`ExternalSecret` per aplikace. K Deploymentu by se sekret namountoval
přes `envFrom: secretRef` nebo `volumeMounts`.

### 3. CI pipeline před tímto GitOps repem

Tento repozitář je **target** GitOps loopu. Před ním musí běžet CI:

```
   dev push          PR (image build)            merge to main
       │                    │                          │
       ▼                    ▼                          ▼
  ┌─────────┐         ┌──────────────┐        ┌─────────────────┐
  │ App     │ ──────▶ │  CI pipeline │ ─────▶ │  Image registry │
  │ source  │         │  (GH Actions │        │  (ACR/ECR)      │
  │ repo    │         │   / GitLab)  │        └─────────────────┘
  └─────────┘         └──────────────┘                 │
                            │                          │ tag bump
                            ▼                          ▼
                  ┌────────────────────┐      ┌─────────────────┐
                  │ Quality gates:     │      │  GitOps repo    │
                  │ - SonarQube scan   │ ───▶ │  (tento repo)   │
                  │ - Unit tests       │      │  PR: bump tag   │
                  │ - SAST (Semgrep)   │      │  v envs/dev     │
                  │ - SCA (Snyk/Trivy) │      └─────────────────┘
                  │ - Container scan   │
                  │ - Helm lint        │
                  │ - kubeconform      │
                  └────────────────────┘
```

**Konkrétně by CI dělalo:**

- **SonarQube** — statická analýza source kódu aplikace, quality gate
  (coverage, code smells, security hotspots). Bez green gate se PR nezmerguje.
- **SAST** (Semgrep / SonarQube) — hledání bezpečnostních patternů ve zdrojáku.
- **SCA** (Snyk / Dependabot / Trivy `fs`) — sken závislostí na CVE.
- **Container scan** (Trivy / Grype) — sken finálního image před pushem
  do registry. Blokující na High/Critical CVE.
- **Image signing** (Cosign) — podpis image + SBOM (Syft), v K8s pak
  enforcement přes Kyverno / Sigstore policy controller.
- **Helm lint + `kubeconform`** — validace tohoto GitOps repa v jeho CI
  na každý PR (zachytí překlepy ve values, neplatné K8s schéma).
- **`helm unittest`** — testy `generic-app` chartu (renderuje se
  Deployment se správným počtem replik při dané values? Vytvoří se
  Ingress jen když `ingress.enabled: true`? atd.).

**Promotion mezi prostředími** — typicky:

- CI merge na main aplikační repo → automatický PR do tohoto repa
  bumpne tag v `envs/dev/values.yaml`.
- Tag v `envs/stage/values.yaml` se bumpne na manuální schválení
  (např. po passing UAT).
- Tag v `envs/prod/values.yaml` se mění jen přes formální change request
  + CAB approval (v bance povinné).

### 4. Další produkční must-have (nad rámec úkolu)

| Oblast | Co přidat |
|---|---|
| **Policy** | Kyverno / OPA Gatekeeper — vynucování `runAsNonRoot`, `readOnlyRootFilesystem`, povolených registry, label conventions. |
| **RBAC** | ArgoCD AppProject per tým, K8s RBAC vázané na Entra ID skupiny. Nikdo nemá `cluster-admin` denně. |
| **Observabilita** | Prometheus + Grafana + Loki + Tempo (LGTM stack), alerty do PagerDuty. ServiceMonitor jako další template v `generic-app`. |
| **Backup / DR** | Velero pro K8s objekty, snapshoty PV. Dokumentovaný RPO/RTO. |
| **Network policy** | Default-deny v každém namespace + explicitní allow per aplikace. |
| **Cost** | OpenCost / Kubecost, allocation podle label `company.bank/owner`. |
| **DR drill** | Pravidelný test obnovy clusteru z Terraform + reaplikace root Applications. |

---

## Decision log (rychlý souhrn)

| Rozhodnutí | Volba | Hlavní důvod |
|---|---|---|
| Templating | **Helm** | Conditional resources, ArgoCD native support, ekosystém |
| Multi-app pattern | **App of Apps** | Per-aplikační flexibilita, auditovatelnost, žádný extra controller |
| Hodnoty v Application | **`valuesObject`** | Strukturované YAML, ne string |
| Root → child Apps | **multi-source `$values`** | Vyhne se zablokovaným `../` cestám |
| Sync policy v prod | `automated + selfHeal` (pro účely demo) | V realitě by se zapínalo manuálně / přes CAB |
| Sekrety | **mimo Git, ESO + Key Vault** | Nikdy plaintext v Gitu, rotace bez deploymentu |
| Infrastruktura | **separátní TF repo** | Jiný cyklus změn, jiné role, jiný blast radius |
| Quality gate | **CI před GitOps, SonarQube + Trivy + Cosign** | Bezpečnost je shift-left, ne afterthought |
