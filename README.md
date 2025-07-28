# 📦 CI/CD Pipeline per Applicazioni su SAP BTP (Cloud Foundry)

## ✨ Obiettivo

Questa documentazione descrive come automatizzare il processo di **build, test e deploy** per applicazioni in esecuzione su **SAP BTP con Cloud Foundry**, utilizzando **GitHub Actions**. Sono previsti due ambienti:

- **Ambiente di test** (branch: `development`)
- **Ambiente di produzione** (branch: `main`)

---

## 🏗️ Struttura dei Branch

- `main`: contiene la versione stabile in produzione.
- `development`: contiene la versione attiva per i test.
- `development-*`: branch temporanei per lo sviluppo di funzionalità. Nascono da `development`.

---

## 🔧 Prerequisiti

Per abilitare il deploy automatico su SAP BTP Cloud Foundry, è necessario:

### 1. Configurare le variabili nel repository GitHub

Vai in `Settings > Secrets and variables > Actions` e aggiungi:

| Nome              | Descrizione                                                   |
|-------------------|---------------------------------------------------------------|
| `CF_API`          | URL dell’API Cloud Foundry (es: `https://api.cf.eu10.hana.ondemand.com`) |
| `CF_ORG`          | Nome dell'organizzazione (es: `my-org`)                       |
| `CF_SPACE_DEV`    | Nome dello space BTP per l’ambiente di test                   |
| `CF_SPACE_PROD`   | Nome dello space BTP per l’ambiente di produzione             |
| `CF_USERNAME`     | Utente Cloud Foundry (service key o account tecnico)          |
| `CF_PASSWORD`     | Password dell’utente                                           |

---

## ⚙️ File della Pipeline

### 1. `ci.yml` - Build, Test e Lint

```yaml
name: CI Build & Test

on:
  push:
    branches: ['**']

jobs:
  build-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup ambiente
        uses: actions/setup-node@v3
        with:
          node-version: '18'

      - name: Installazione dipendenze
        run: npm ci

      - name: Esecuzione test
        run: npm test

      - name: Linting
        run: npm run lint
```

---

### 2. `deploy-dev.yml` - Deploy in ambiente di test

```yaml
name: Deploy to Dev (Cloud Foundry)

on:
  push:
    branches:
      - development

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Installazione CLI Cloud Foundry
        run: |
          curl -L "https://packages.cloudfoundry.org/stable?release=linux64" -o cf-cli.tgz
          tar -xzf cf-cli.tgz
          sudo mv cf /usr/local/bin

      - name: Login su Cloud Foundry
        run: |
          cf api ${{ secrets.CF_API }}
          cf auth ${{ secrets.CF_USERNAME }} ${{ secrets.CF_PASSWORD }}
          cf target -o ${{ secrets.CF_ORG }} -s ${{ secrets.CF_SPACE_DEV }}

      - name: Deploy su ambiente Dev
        run: cf push --var ENV=dev
```

---

### 3. `deploy-prod.yml` - Deploy in produzione

```yaml
name: Deploy to Production (Cloud Foundry)

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://<tuo-url-produzione>  # opzionale

    steps:
      - uses: actions/checkout@v4

      - name: Installazione CLI Cloud Foundry
        run: |
          curl -L "https://packages.cloudfoundry.org/stable?release=linux64" -o cf-cli.tgz
          tar -xzf cf-cli.tgz
          sudo mv cf /usr/local/bin

      - name: Login su Cloud Foundry
        run: |
          cf api ${{ secrets.CF_API }}
          cf auth ${{ secrets.CF_USERNAME }} ${{ secrets.CF_PASSWORD }}
          cf target -o ${{ secrets.CF_ORG }} -s ${{ secrets.CF_SPACE_PROD }}

      - name: Deploy in Produzione
        run: cf push --var ENV=prod
```

---

## 📁 File `manifest.yml` (esempio)

```yaml
applications:
  - name: nome-app
    memory: 256M
    instances: 1
    path: .
    buildpacks:
      - nodejs_buildpack
    env:
      ENV: ((ENV))
```

---

## ✅ Riepilogo Pipeline

| Branch        | Ambiente      | Azione                         |
|---------------|---------------|--------------------------------|
| `feature/*`   | Nessuno       | Test e Lint (CI)               |
| `development` | Test (dev)    | Deploy automatico su dev       |
| `main`        | Produzione    | Deploy automatico su produzione|

---

## 🔁 Rollback del Deploy in Produzione

### 🎯 Obiettivo

Il rollback consente di **ripristinare una versione precedente** dell'applicazione nel caso in cui un deploy in produzione causi errori o malfunzionamenti.

### ⚙️ Approcci al rollback in Cloud Foundry

Cloud Foundry **non ha un comando "rollback" diretto**. Tuttavia, ci sono tre strategie per gestirlo:

### ✅ 1. Versionamento dei deploy

```bash
git checkout tags/<tag_versione_stabile>
cf push --var ENV=prod
```

GitHub Action per tagging:

```yaml
- name: Tag produzione
  run: |
    git config --global user.email "ci@bot.com"
    git config --global user.name "CI Bot"
    git tag prod-$(date +%Y%m%d-%H%M%S)
    git push origin --tags
```

### 🔁 2. Blue-green deploy

```bash
cf push my-app-v2 -f manifest.yml --no-start
cf map-route my-app-v2 my.domain.com --hostname my-app
cf stop my-app
cf delete my-app -f
cf rename my-app-v2 my-app
```

### 🛑 3. Snapshot manuale

Salva `manifest.yml`, env vars, artefatti per versioni stabili.

---

## ✉️ Contatti

Per domande o modifiche alla pipeline, contattare il team DevOps interno o l’amministratore SAP BTP dell’organizzazione.
