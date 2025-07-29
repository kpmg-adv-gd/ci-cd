# 📦 CI/CD Pipeline per Applicazioni su SAP BTP (Cloud Foundry)

## ✨ Obiettivo

Questa documentazione descrive come automatizzare il processo di **build e deploy** per applicazioni in esecuzione su **SAP BTP con Cloud Foundry**, utilizzando **GitHub Actions**. Sono previsti due ambienti:

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

## 1. Configurare gli environment e le variabili nel repository GitHub

Per abilitare il deploy automatico su **SAP BTP Cloud Foundry**, è necessario configurare due environment nel repository GitHub e associare le variabili e i secret richiesti per ciascun ambiente.

### 🔧 Creazione degli environment

Vai in `Settings > Secrets and variables > Actions` e crea **due environment**:

* `CI-CD-DEV` – per il deploy nello **space di test**
* `CI-CD-PROD` – per il deploy nello **space di produzione**

All'interno di ciascun environment, configura le seguenti variabili e secret:

| Nome            | Descrizione                                                              | Tipo                  | Environment       |
| --------------- | ------------------------------------------------------------------------ | --------------------- | ----------------- |
| `CF_API`        | URL dell’API Cloud Foundry (es: `https://api.cf.eu10.hana.ondemand.com`) | Variabile di ambiente | Entrambi          |
| `CF_ORG`        | Nome dell'organizzazione (es: `my-org`)                                  | Variabile di ambiente | Entrambi          |
| `CF_SPACE_DEV`  | Nome dello space BTP per l’ambiente di test                              | Variabile di ambiente | Solo `CI-CD-DEV`  |
| `CF_SPACE_PROD` | Nome dello space BTP per l’ambiente di produzione                        | Variabile di ambiente | Solo `CI-CD-PROD` |
| `CF_USERNAME`   | Utente Cloud Foundry (service key o account tecnico)                     | Variabile di ambiente | Entrambi          |
| `CF_PASSWORD`   | Password dell’utente                                                     | **Secret**            | Entrambi          |

> ⚠️ **Nota**:
>
> * `CF_PASSWORD` deve essere sempre configurato come **secret**, in quanto contiene credenziali sensibili.
> * Le altre voci possono essere configurate come **variabili di ambiente**.

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
    environment: CI-CD-DEV

    steps:
      - name: 🔄 Checkout codice
        uses: actions/checkout@v4

      - name: Usa manifest per ambiente Dev
        run: cp manifest-dev.yml manifest.yml


      - name: 🔧 Installa cf CLI via dpkg
        run: |
          echo "📦 Scarico pacchetto Debian cf CLI..."
          curl -L "https://packages.cloudfoundry.org/stable?release=debian64" -o cf.deb
          sudo dpkg -i cf.deb
          cf version

      - name: 🔐 Login su Cloud Foundry
        run: |
          set -e
          echo "🌐 API endpoint: ${{ vars.CF_API }}"
          echo "👤 Login con utente: ${{ vars.CF_USERNAME }}"
          cf login -a '${{ vars.CF_API }}' -u '${{ vars.CF_USERNAME }}' -p '${{ secrets.CF_PASSWORD }}' -o '${{ vars.CF_ORG }}' -s '${{ vars.CF_SPACE_DEV }}'

      - name: 🚀 Deploy su ambiente Dev
        run: |
          set -e
          echo "📂 Lista file in workspace:"
          ls -la
          echo "📦 Avvio cf push..."
          cf push --var ENV=dev
```

---

### 3. `deploy-prod.yml` - Deploy in produzione

```yaml
name: Deploy to Prd (Cloud Foundry)

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: CI-CD-PRD

    steps:
      - name: 🔄 Checkout codice
        uses: actions/checkout@v4

      - name: Usa manifest per ambiente Prd
        run: cp manifest-prd.yml manifest.yml


      - name: 🔧 Installa cf CLI via dpkg
        run: |
          echo "📦 Scarico pacchetto Debian cf CLI..."
          curl -L "https://packages.cloudfoundry.org/stable?release=debian64" -o cf.deb
          sudo dpkg -i cf.deb
          cf version

      - name: 🔐 Login su Cloud Foundry
        run: |
          set -e
          echo "🌐 API endpoint: ${{ vars.CF_API }}"
          echo "👤 Login con utente: ${{ vars.CF_USERNAME }}"
          cf login -a '${{ vars.CF_API }}' -u '${{ vars.CF_USERNAME }}' -p '${{ secrets.CF_PASSWORD }}' -o '${{ vars.CF_ORG }}' -s '${{ vars.CF_SPACE_PROD }}'

      - name: 🚀 Deploy su ambiente Prd
        run: |
          set -e
          echo "📂 Lista file in workspace:"
          ls -la
          echo "📦 Avvio cf push..."
          cf push --var ENV=prd
```

---

Ecco la sezione **modificata e ampliata** per includere l’uso dei file `manifest-dev.yml` e `manifest-prd.yml` nella cartella `staging`, come richiesto:

---

## 📁 File `manifest.yml` (configurazione per ambienti)

Per gestire in modo efficace il deploy su diversi ambienti (dev e prod), è consigliato **non modificare direttamente** il file `manifest.yml` nella root del progetto.
Invece, crea due file distinti all'interno di una cartella chiamata `staging`:

```
staging/
├── manifest-dev.yml
└── manifest-prd.yml
```

Questi file conterranno le configurazioni specifiche per ciascun ambiente (ad esempio lo space, le variabili, ecc.).

### Esempio di `staging/manifest-dev.yml`

```yaml
applications:
  - name: nome-app-dev
    memory: 256M
    instances: 1
    path: .
    buildpacks:
      - nodejs_buildpack
    env:
      ENV: dev
```

### Esempio di `staging/manifest-prd.yml`

```yaml
applications:
  - name: nome-app
    memory: 512M
    instances: 2
    path: .
    buildpacks:
      - nodejs_buildpack
    env:
      ENV: prod
```

> ⚙️ **Deploy automatico**
> Durante l'esecuzione del workflow CI/CD, lo script di deploy selezionerà automaticamente il file corretto (`manifest-dev.yml` o `manifest-prd.yml`) dalla cartella `staging`, **lo copierà e sovrascriverà** come `manifest.yml` nella root del progetto, che è il file atteso dal comando `cf push`.

---

Fammi sapere se vuoi aggiungere anche uno snippet dello script che gestisce questa logica.


---

## ✅ Riepilogo Pipeline

| Branch        | Ambiente      | Azione                         |
|---------------|---------------|--------------------------------|
| `development-*`   | Nessuno       | Test e Lint (CI)               |
| `development` | Test (dev)    | Deploy automatico su dev       |
| `main`        | Produzione    | Deploy automatico su produzione|

---
