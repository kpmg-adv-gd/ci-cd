Ecco il tuo `README.md` formattato in modo coerente, con una numerazione logica dei paragrafi, utilizzo omogeneo delle emoji e sezioni ben organizzate:

---

# 📦 CI/CD Pipeline per Applicazioni su SAP BTP (Cloud Foundry)

## ✨ Obiettivo

Questa documentazione descrive come automatizzare il processo di **build e deploy** per applicazioni in esecuzione su **SAP BTP con Cloud Foundry**, utilizzando **GitHub Actions**.
Sono previsti due ambienti:

* **Ambiente di test** (`development`)
* **Ambiente di produzione** (`main`)

---

## 1️⃣ Struttura dei Branch

* `main`: contiene la versione stabile in produzione.
* `development`: contiene la versione attiva per i test.
* `development-*`: branch temporanei per lo sviluppo di funzionalità (feature branch). Nascono da `development`.

---

## 2️⃣ Prerequisiti

### 2.1 🔐 Configurare gli environment e le variabili nel repository GitHub

Per abilitare il deploy automatico su **SAP BTP Cloud Foundry**, è necessario configurare due **environment** nel repository GitHub e associare variabili e secret richiesti per ciascun ambiente.

#### ✨ Creazione degli environment

Vai in `Settings > Secrets and variables > Actions` e crea:

* `CI-CD-DEV` – per il deploy nello **space di test**
* `CI-CD-PRD` – per il deploy nello **space di produzione**

#### 🧾 Variabili richieste

| Nome            | Descrizione                                                              | Tipo                  | Environment      |
| --------------- | ------------------------------------------------------------------------ | --------------------- | ---------------- |
| `CF_API`        | URL dell’API Cloud Foundry (es: `https://api.cf.eu10.hana.ondemand.com`) | Variabile di ambiente | Entrambi         |
| `CF_ORG`        | Nome dell'organizzazione (es: `my-org`)                                  | Variabile di ambiente | Entrambi         |
| `CF_SPACE_DEV`  | Nome dello space BTP per l’ambiente di test                              | Variabile di ambiente | Solo `CI-CD-DEV` |
| `CF_SPACE_PROD` | Nome dello space BTP per l’ambiente di produzione                        | Variabile di ambiente | Solo `CI-CD-PRD` |
| `CF_USERNAME`   | Utente Cloud Foundry (service key o account tecnico)                     | Variabile di ambiente | Entrambi         |
| `CF_PASSWORD`   | Password dell’utente Cloud Foundry                                       | **Secret**            | Entrambi         |

> ⚠️ `CF_PASSWORD` deve essere **configurato come secret**, perché contiene credenziali sensibili.
> Le altre voci possono essere definite come **variabili di ambiente**.

---

## 3️⃣ Struttura del Repository

La seguente struttura supporta una pipeline CI/CD per SAP BTP Cloud Foundry, con ambienti separati, test automatici e parametrizzazione centralizzata.

```
.
├── .github/
│   └── workflows/
│       ├── deploy-dev.yml     # Pipeline CI/CD per ambiente di sviluppo (CI-CD-DEV)
│       └── deploy-prd.yml     # Pipeline CI/CD per ambiente di produzione (CI-CD-PRD)
│
├── .staging/
│   ├── manifest-dev.yml       # Configurazione Cloud Foundry per ambiente di sviluppo
│   └── manifest-prd.yml       # Configurazione Cloud Foundry per ambiente di produzione
│
├── unit-test/
│   └── *.test.js              # Test unitari (es. server.test.js), eseguiti automaticamente prima del deploy
│
├── manifest.yml               # File temporaneo sovrascritto dallo script CI/CD
├── server.js                  # App Node.js (Express)
├── package.json               # Script di build/test e dipendenze
└── README.md                  # Documentazione
```

### 📌 Convenzioni

* **Pattern di naming dei test:** `*.test.js`
* **Cartella test:** `unit-test/`
* **Test automatici:** eseguiti in ogni pipeline CI/CD (`npm test`)
* **Manifest per ambiente:** mantenuti in `.staging/` e selezionati dinamicamente dalla pipeline

---

## 4️⃣ Configurazione dei manifest Cloud Foundry

Per ambienti separati, NON modificare direttamente `manifest.yml`.
Invece, crea due file nella cartella `.staging/`:

```
.staging/
├── manifest-dev.yml
└── manifest-prd.yml
```

### 🧾 Esempio `manifest-dev.yml`

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

### 🧾 Esempio `manifest-prd.yml`

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

> ⚙️ Durante il deploy, il workflow selezionerà automaticamente il manifest corretto e lo copierà come `manifest.yml`.

---

## 5️⃣ Esempi di Workflow

### 5.1 🚀 Deploy in ambiente di sviluppo (`deploy-dev.yml`)

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

      - name: 🧪 Installa dipendenze e lancia i test
        run: |
          echo "📦 Installo dipendenze..."
          npm install
          echo "🧪 Eseguo test unitari..."
          npm test

      - name: Usa manifest per ambiente Dev
        run: cp .staging/manifest-dev.yml manifest.yml


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

### 5.2 🚀 Deploy in ambiente di produzione (`deploy-prd.yml`)

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

      - name: 🧪 Installa dipendenze e lancia i test
        run: |
          echo "📦 Installo dipendenze..."
          npm install
          echo "🧪 Eseguo test unitari..."
          npm test

      - name: 📄 Usa manifest per ambiente Prd
        run: cp .staging/manifest-prd.yml manifest.yml
        

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

## 6️⃣ ✅ Riepilogo Pipeline

| Branch          | Ambiente   | Azione                          |
| --------------- | ---------- | ------------------------------- |
| `development-*` | Nessuno    | Test e lint (CI)                |
| `development`   | Test (dev) | Deploy automatico su dev        |
| `main`          | Produzione | Deploy automatico su produzione |

---

Fammi sapere se vuoi allegare anche uno schema visivo (mermaid, PlantUML o SVG) della pipeline o un badge GitHub Actions per il build status.
