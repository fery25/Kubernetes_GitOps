# Kubernetes GitOps — ArgoCD App of Apps + Helm

Tento repozitář ukazuje způsob nasazení podobných HTTP aplikací (generic-app) do Kubernetes pomocí ArgoCD. Řešení pokrývá tři prostředí (dev/stage/prod) a je navrženo tak, aby přidání další aplikace znamenalo pouze jeden nový záznam v konfiguračním souboru. Architektura stojí na dvou pilířích: jednom sdíleném Helm chartu pro všechny aplikace a jednom řídicím Helm chartu implementujícím vzor **"App of Apps"**.

## Struktura repozitáře

```
charts/generic-app/      # Sdílený chart pro všechny aplikace (Deployment + Service + Ingress)
argocd-apps/             # "App of Apps" - generuje N × ArgoCD Application objektů
envs/{dev,stage,prod}/   # Konfigurace pro jednotlivá prostředí (aplikace, repliky, verze)
bootstrap/               # Kořenová aplikace pro každé prostředí (jediné, co se aplikuje ručně)
```

**Princip fungování:** Ručně aplikuji `bootstrap/root-app-<env>.yaml` → ArgoCD zpracuje řídicí chart (`argocd-apps`) s konfiguračním souborem `envs/<env>/values.yaml` → tím se vygeneruje *N* podřízených `Application` objektů → každý z těchto objektů následně nasadí sdílený chart (`generic-app`) se svými specifickými parametry do cílového jmenného prostoru (namespace).

## Jak řešení spustit

**Lokální ověření (bez nutnosti clusteru):**

```bash
# Zkontroluje syntaxi obou Helm chartů
helm lint charts/generic-app/ argocd-apps/

# Vygeneruje finální manifesty a ukáže, co by ArgoCD vytvořilo pro DEV prostředí
helm template root argocd-apps/ -f envs/dev/values.yaml
```

**Nasazení do Kubernetes clusteru** (předpokladem je nainstalované ArgoCD verze 2.6 nebo vyšší ve jmenném prostoru `argocd`):

```bash
# V souboru bootstrap/root-app-dev.yaml upravte 'repoURL' na adresu svého repozitáře
kubectl apply -f bootstrap/root-app-dev.yaml

# Zkontrolujte stav synchronizace
argocd app list
```

**Proč se kořenová aplikace aplikuje ručně?** Jde o klasický problém typu „co bylo dřív, vejce nebo slepice". ArgoCD synchronizuje pouze ty objekty, na které ukazují `Application` CRD již existující v clusteru. Aby ArgoCD vůbec věděl o existenci tohoto repozitáře, musí v něm být alespoň jedna `Application`, a tu tam musí zvenku „naseednout" někdo jiný. Vše ostatní si pak ArgoCD vytváří sám. V produkčním provozu by tento krok prováděla bootstrap pipeline po vytvoření clusteru — „ručně" tedy znamená jednou při zakládání clusteru, nikoliv při každém nasazení aplikace.

## Jak přidat další aplikaci

Celý proces je navržen pro maximální jednoduchost. Stačí přidat jeden záznam do souboru `envs/<env>/values.yaml`:

```yaml
- name: loans-api
  replicas: 2
  image:
    repository: ghcr.io/example-org/loans-api
    tag: "0.1.0"
  ingress:
    enabled: true
```

Po odeslání změn (commit → push) provede ArgoCD automaticky synchronizaci a nová `Application` pro `loans-api` je vytvořena.

## Proč byl zvolen tento přístup

**Proč Helm?** Protože zadání mluví o 10 podobných aplikacích, které sdílí stejnou strukturu a liší se jen parametry, jsou `values` soubory ideální. Podmíněné vytváření zdrojů (např. Ingressu) je v Helmu otázka jedné `{{ if }}` podmínky, zatímco v Kustomize je řešení těžkopádnější.

**Proč "App of Apps" a ne ApplicationSet?** ApplicationSet je elegantní pro jednoduché scénáře (jedna šablona × více clusterů). Pro potřebu specifických odchylek u každé aplikace (jiné resources, proměnné prostředí) je ale flexibilnější náš přístup. Navíc `helm template` lokálně zobrazí naprosto přesný stav, který ArgoCD vytvoří, což zjednodušuje audit i ladění (debugging). ApplicationSet generuje manifesty až ve svém kontroleru v clusteru.

**Proč Multi-source a `valuesObject`?** Kořenová aplikace čte `values.yaml` z druhého zdroje pomocí odkazu `$values`, čímž se vyhneme relativním cestám (`../`), které ArgoCD z bezpečnostních důvodů blokuje. Funkce `valuesObject` navíc předává hodnoty jako strukturovaný YAML, ne jako vnořený textový řetězec, což eliminuje chyby v odsazování.

## Co bylo pro účely úkolu zjednodušeno

- **Žádné reálné aplikace ani obrazy kontejnerů** — použil jsem zástupné názvy (placeholdery jako `ghcr.io/example-org/...`).
- **Žádná hesla uložená v Gitu** — citlivé údaje do verzovacího systému nepatří, v praxi by se řešily odděleně (např. přes External Secrets a Key Vault).
- **Automatická oprava (`selfHeal: true`) je zapnutá i v produkci** — v reálném prostředí by nasazení do produkce podléhalo manuálnímu schválení.
- **Jeden cílový Kubernetes cluster** — pro ukázku se vše nasazuje do jednoho clusteru, v realitě by každé prostředí cílilo na svůj vlastní oddělený cluster.
- **Bezpečnostní politiky na úrovni clusteru chybí** — v aplikaci je sice nastaveno základní zabezpečení podů (`securityContext`), ale globální politiky (např. Kyverno nebo NetworkPolicies) by měly patřit do odděleného repozitáře, který cluster zakládá.
- **Chybí automatizované testování (CI)** — repozitář neobsahuje automatickou validaci Helm chartů ani kontrolu YAML souborů při vytvoření Pull Requestu.

## Co by se v produkci muselo doplnit

- **Správa infrastruktury** — oddělený repozitář v Terraformu pro založení samotného clusteru, databází, DNS a sítí. Infrastruktura má jiný životní cyklus, jiná přístupová práva a nese jiné riziko než nasazování samotných aplikací.
- **Správa citlivých údajů (Secrets)** — využití nástroje External Secrets Operator napojeného na Azure Key Vault/HashiCorp Vault, ověřování přes Workload Identity. V Gitu by pak ležely pouze bezpečné odkazy, nikoliv samotná hesla.
- **CI pipeline pro samotné aplikace** — před nasazením musí proběhnout kontrola kvality kódu (SonarQube), bezpečnostní skeny (SAST, Trivy) a podepsání kontejneru. Na konci CI by se automaticky vytvořil Pull Request sem, který by zvedl verzi v DEV prostředí. Povýšení do Stage/Prod by podléhalo manuálnímu schválení.
- **Automatická validace tohoto repozitáře** — každý zásah do konfigurace by měl být otestován nástroji jako `helm lint` a `kubeconform` ještě před sloučením kódu (Merge).
- **Vynucování bezpečnostních pravidel** — nasazení nástrojů jako OPA Gatekeeper nebo Kyverno, které zakážou spuštění kontejnerů s právy roota nebo stahování obrazů z neschválených registrů.
- **Dohled (Observability)** — doplnění metrik a logů (Prometheus, Grafana, Loki), v univerzálním chartu by přibyla šablona pro `ServiceMonitor`.

## Architektonické kompromisy (Trade-offs)

Každé architektonické rozhodnutí přináší výhody i nevýhody. Zde jsou kompromisy, které jsem v tomto návrhu přijal:

| Volba | Získaná výhoda | Daň za řešení (Nevýhoda) |
|---|---|---|
| **App of Apps místo ApplicationSet** | Vysoká flexibilita pro každou aplikaci zvlášť. Možnost lokálně si přes `helm template` ověřit naprosto přesný stav, který vznikne v clusteru. | Bylo nutné vytvořit více "obslužného" Helm kódu a složitější strukturu. |
| **Funkce `valuesObject` a více zdrojů** | YAML konfigurace je krásně strukturovaná, vyhnul jsem se vnořeným textovým řetězcům. | Řešení striktně vyžaduje novější verze ArgoCD (2.6 a vyšší). |
| **Zapnutý `selfHeal` pro všechna prostředí** | Konzistentní smyčka automatizace. Pokud někdo provede manuální zásah v clusteru, ArgoCD to okamžitě opraví. | V produkčním prostředí by automatické opravy měly ideálně podléhat schvalovacímu procesu (Change gate). |
| **Přísné zabezpečení v základu (`runAsNonRoot`)** | Aplikace automaticky splňují bezpečnostní standardy bez nutnosti zásahu vývojářů. | Zastaralé aplikace (legacy), které vyžadují práva roota, bez explicitní výjimky v konfiguraci nespadnou a nespustí se. |
