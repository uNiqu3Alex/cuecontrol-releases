# TableFlow Release

Repository di distribuzione per **CueManager** - sistema gestionale per sale biliardo.

Questo repository non contiene codice sorgente. Funge esclusivamente da **server di distribuzione** tramite GitHub Releases per il sistema di aggiornamento automatico di CueManager.

---

## Cos'e CueManager

CueManager e un sistema all-in-one per la gestione di sale biliardo che integra:

- **Controllo hardware** delle luci dei tavoli tramite rele Modbus TCP (moduli Waveshare 16CH)
- **Backend locale** NestJS con database SQLite - nessun cloud richiesto
- **Interfaccia desktop** Electron per il PC cassa / touchscreen in sala
- **App mobile** per accesso remoto via Internet
- **Sistema di licenze offline** con firma RSA

L'intera applicazione viene distribuita come un singolo installer Windows (NSIS) che include il backend NestJS embedded, un runtime Node.js standalone e l'interfaccia Electron. Il backend si avvia automaticamente come processo figlio, gestito da un watchdog interno che ne garantisce il funzionamento continuo.

---

## Architettura di distribuzione

```
                    GitHub Releases
                    (questo repo)
                         |
            +------------+------------+
            |                         |
    Release Completa             Hotfix Backend
    (tag: vX.Y.Z)             (tag: backend-vX.Y.Z)
            |                         |
    +-----------------+          +-----------+
    | Setup EXE       |          | ZIP dist/ |
    | latest.yml      |          +-----------+
    | .blockmap       |               |
    +-----------------+               |
            |                         |
     electron-updater          OTA backend-only
     (aggiornamento            (aggiornamento
      completo)                 a caldo)
```

### Due percorsi di aggiornamento

CueManager supporta due meccanismi di aggiornamento indipendenti, entrambi serviti da questo repository:

#### 1. Release completa (FE + BE)

Aggiornamento dell'intera applicazione tramite `electron-updater`.

| Elemento | Descrizione |
|---|---|
| **Tag** | `vX.Y.Z` (es. `v1.2.0`) |
| **Asset** | `CueManager-Setup-X.Y.Z.exe` - Installer NSIS completo |
| **Asset** | `latest.yml` - Manifest per electron-updater (versione, hash, dimensione) |
| **Asset** | `CueManager-Setup-X.Y.Z.exe.blockmap` - Delta update per download incrementali |
| **Meccanismo** | L'app Electron controlla `latest.yml` ogni ora. Se trova una versione superiore, scarica l'installer in background e lo applica automaticamente |

**Flusso di aggiornamento completo:**

1. `electron-updater` legge `latest.yml` da GitHub Releases
2. Confronta la versione con quella installata
3. Scarica il nuovo installer (usa il blockmap per download differenziale se disponibile)
4. Ferma il watchdog backend
5. Ferma il processo backend (rilascia il lock SQLite)
6. Attende 2 secondi per la chiusura pulita
7. Esegue `quitAndInstall` - chiude Electron e lancia il nuovo installer NSIS
8. L'installer sovrascrive i file in Program Files e riavvia l'applicazione
9. Al riavvio, il backend esegue automaticamente le migration pendenti sul database

#### 2. Hotfix backend-only (OTA)

Aggiornamento a caldo del solo backend, senza reinstallare l'applicazione.

| Elemento | Descrizione |
|---|---|
| **Tag** | `backend-vX.Y.Z` (es. `backend-v1.1.1`) |
| **Asset** | `cuemanager-backend-vX.Y.Z.zip` - Archivio contenente la directory `dist/` compilata |
| **Manifest** | `manifest-backend.json` nella root del repo sorgente - punta alla release corrente |
| **Meccanismo** | Il backend controlla il manifest ogni ora. Se trova una versione superiore, scarica lo ZIP, lo verifica e lo estrae nella directory di override in AppData |

**Flusso di aggiornamento OTA:**

1. Il backend scarica il manifest da `UPDATE_MANIFEST_URL`
2. Confronta la versione con quella corrente
3. Verifica che la versione corrente sia >= `minVersion` (altrimenti richiede aggiornamento manuale)
4. Scarica lo ZIP e ne verifica l'integrita tramite SHA256
5. Spawna `updater.js` come processo detached
6. L'updater estrae il contenuto in `%APPDATA%\CueManager\backend-override\`
7. Invia SIGTERM al processo backend tramite PID file
8. Il watchdog Electron rileva il backend offline e lo riavvia
9. `backend-manager.js` controlla la directory di override: se esiste, usa quei file al posto di quelli bundled in `resources/`
10. Al riavvio, `main.ts` esegue automaticamente le migration pendenti se necessario

**Directory di override:**

```
%APPDATA%\CueManager\
    backend-override\
        dist\
            src\
                main.js
                ...
```

I file nella directory di override hanno priorita su quelli installati in `Program Files\CueManager\resources\backend\`. Questo meccanismo evita problemi di permessi su Program Files e permette rollback semplicemente cancellando la directory di override.

---

## Struttura delle release

### Release completa

```
GitHub Release: "CueManager v1.2.0" (tag: v1.2.0)
  Assets:
    - CueManager-Setup-1.2.0.exe          (~80-120 MB)
    - CueManager-Setup-1.2.0.exe.blockmap (~200 KB)
    - latest.yml                           (~300 bytes)
```

Contenuto di `latest.yml`:

```yaml
version: 1.2.0
files:
  - url: CueManager-Setup-1.2.0.exe
    sha512: <base64-encoded-sha512>
    size: <bytes>
path: CueManager-Setup-1.2.0.exe
sha512: <base64-encoded-sha512>
releaseDate: '2026-04-13T12:00:00.000Z'
```

### Hotfix backend

```
GitHub Release: "Backend Hotfix v1.1.1" (tag: backend-v1.1.1)
  Assets:
    - cuemanager-backend-v1.1.1.zip       (~5-15 MB)
```

Struttura interna dello ZIP:

```
dist/
    src/
        main.js
        app.module.js
        app.controller.js
        common/
            prisma.service.js
            jwt-auth.guard.js
            event-log.service.js
        modules/
            auth/
            users/
            tables/
            sessions/
            relay/
            players/
            roles/
            day/
            reports/
            license/
            system/
            update/
```

---

## Manifest backend

Il file `manifest-backend.json` (nel repository sorgente, non in questo repo) ha la seguente struttura:

```json
{
  "version": "1.1.1",
  "releaseDate": "2026-04-13T14:30:00.000Z",
  "downloadUrl": "https://github.com/uNiqu3Alex/tableflow-release/releases/download/backend-v1.1.1/cuemanager-backend-v1.1.1.zip",
  "sha256": "a1b2c3d4e5f6...",
  "minVersion": "1.0.0",
  "changelog": "Fix critico: sessioni non si chiudono correttamente",
  "requiresMigration": false
}
```

| Campo | Tipo | Descrizione |
|---|---|---|
| `version` | string | Versione semver dell'aggiornamento |
| `releaseDate` | string | Data ISO 8601 della pubblicazione |
| `downloadUrl` | string | URL diretto allo ZIP su GitHub Releases |
| `sha256` | string | Hash SHA256 dello ZIP per verifica integrita |
| `minVersion` | string | Versione minima richiesta per applicare l'OTA. Se il client ha una versione inferiore, l'aggiornamento viene bloccato e si richiede reinstallazione manuale |
| `changelog` | string | Note di rilascio leggibili dall'utente |
| `requiresMigration` | boolean | Se `true`, indica che l'aggiornamento include modifiche allo schema del database |

---

## Sicurezza e integrita

- **SHA256**: Ogni release include un hash SHA256 calcolato e stampato durante il processo di pubblicazione. Per le release complete, `latest.yml` include anche un hash SHA512 generato da electron-builder
- **Verifica pre-installazione**: Il backend verifica l'hash SHA256 del pacchetto scaricato prima di estrarlo. Se non corrisponde, il file viene eliminato e l'aggiornamento viene annullato
- **Dimensione minima**: Gli script di release verificano che l'EXE sia > 50 MB per rilevare build corrotte
- **Coerenza versione**: La versione in `package.json`, nel nome del file EXE e in `latest.yml` deve corrispondere esattamente
- **Versione incrementale**: Una nuova release deve avere una versione strettamente maggiore dell'ultima pubblicata

---

## Convenzioni di versionamento

Il progetto segue [Semantic Versioning](https://semver.org/):

- **MAJOR** (`X.0.0`): Cambiamenti incompatibili, ristrutturazione architetturale, reset database
- **MINOR** (`0.X.0`): Nuove funzionalita retrocompatibili, nuovi moduli, nuovi endpoint API
- **PATCH** (`0.0.X`): Bug fix, hotfix backend, correzioni UI

### Convenzione tag

| Tipo | Pattern tag | Esempio |
|---|---|---|
| Release completa | `vX.Y.Z` | `v1.2.0` |
| Hotfix backend | `backend-vX.Y.Z` | `backend-v1.2.1` |

---

## Pubblicazione

Le release vengono create tramite script PowerShell automatizzati nel repository sorgente:

```powershell
# Release completa - dry run (solo controlli)
.\scripts\release-full.ps1 -DryRun

# Release completa - pubblicazione
.\scripts\release-full.ps1

# Hotfix backend - pubblicazione
.\scripts\release-hotfix-be.ps1 -ReleaseNotes "Descrizione fix"

# Hotfix con migration database
.\scripts\release-hotfix-be.ps1 -RequiresMigration -ReleaseNotes "Aggiunta tabella"
```

Gli script eseguono automaticamente tutti i controlli di validazione (versione, artefatti, hash, coerenza) prima di creare la release su GitHub.

---

## Stack tecnologico dell'applicazione

| Componente | Tecnologia |
|---|---|
| Backend | NestJS 11, TypeScript 5, Prisma 7, SQLite |
| Frontend | Electron 39, Vanilla JavaScript |
| Hardware | Modbus TCP (moduli Waveshare 16CH) |
| Autenticazione | JWT (passport-jwt) |
| Real-time | Socket.IO 4 |
| Installer | electron-builder, NSIS |
| Auto-update | electron-updater + OTA custom |

---

## Licenza

Questo repository e il software CueManager sono proprietari.
Tutti i diritti riservati. Copyright (c) 2026 CueManager.
