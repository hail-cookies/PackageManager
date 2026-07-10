# StaticInstaller – workbench & DB connection flow

The diagram below shows, for the **default** setting
`COMPOSER.USE_NEW_WORKBENCH_FOR_EVERY_APP = TRUE`, when workbenches are
started/stopped and which database connections (SQL Server sessions) are
opened and closed during a Composer install/update run.

Legend for connections:

- **Conn A – metamodel connection**: created by `Workbench::start()` via
  `DataConnectionFactory::createModelLoaderConnection()` and given to the model
  loader. After the `SqlModelLoader` fix, the metamodel *data source* reuses
  this same connection, so **all metamodel reads and writes share one session**.
- **Conn X – app data-source connection**: a *separate* connection opened by an
  app's own SQL installer when it uses `setDataSourceSelector('0x…')` (e.g. an
  app with its own database). Apps that use `setDataConnection($modelConnection)`
  reuse Conn A instead.

```mermaid
sequenceDiagram
    autonumber
    participant Composer
    participant SI as StaticInstaller
    participant WB as Workbench
    participant ML as ModelLoader
    participant IA as InstallApp action
    participant DB as SQL Server (metamodel)
    participant AppDB as SQL Server (app data source)

    Composer->>SI: composerFinishInstall() / composerFinishUpdate()

    Note over SI: Loop: core first, then every app<br/>Each app => new self() => install()

    rect rgb(235, 245, 255)
    Note over SI,DB: ── Install app #1 ──
    SI->>SI: install(app1) → new self()
    SI->>IA: installApp(app1)
    IA->>SI: getWorkbench()
    SI->>WB: Workbench::startNewInstance()
    activate WB
    WB->>ML: create model loader +<br/>createModelLoaderConnection() = Conn A (not yet connected)
    Note over SI,WB: reads COMPOSER.USE_NEW_WORKBENCH_FOR_EVERY_APP<br/>= TRUE → workbench is per-app (not stored globally)

    IA->>DB: first metamodel query → OPEN Conn A
    activate DB
    Note over IA,DB: MetaModelInstaller:<br/>startTransaction() on Conn A → writes exf_object/…<br/>then commit() (single metamodel session)

    opt App has its own SQL installer (setDataSourceSelector)
        IA->>AppDB: run migrations → OPEN Conn X
        activate AppDB
        IA-->>AppDB: CLOSE Conn X (installer done)
        deactivate AppDB
    end

    IA-->>SI: install finished
    SI->>WB: stopWorkbenchIfNotGlobal() → Workbench::stop()
    WB->>WB: saveContexts()
    WB->>DB: data()->disconnectAll() (active_sources only)
    SI->>ML: disconnectModelLoader():<br/>commit open tx + disconnect Conn A
    SI->>DB: CLOSE Conn A
    deactivate DB
    WB-->>SI: workbench stopped
    deactivate WB
    end

    rect rgb(235, 255, 235)
    Note over SI,DB: ── Install app #2 (identical, fresh workbench + fresh Conn A) ──
    SI->>SI: install(app2) → new self()
    SI->>WB: Workbench::startNewInstance() → new Conn A
    activate WB
    activate DB
    Note over IA,DB: same pattern: OPEN Conn A → model import → …
    SI->>DB: CLOSE Conn A (stopWorkbenchIfNotGlobal)
    deactivate DB
    deactivate WB
    end

    SI-->>Composer: "Installed N apps"
```

## What changed to prevent the freeze

- **Before the fix:** the metamodel *writes* went through a **second**
  connection (Conn B) created by `SqlModelLoader::loadDataSource()`, separate
  from the model-loader connection (Conn A). Teardown only closed Conn A, so
  Conn B could be left **idle with an open transaction**, holding locks on the
  metamodel tables. The **next** app's install then **blocked** on those locks
  (the "sleeping" holder vs. "suspended" `INSERT` you observed) — which is why
  it hit the *second* app, not the first.
- **After the fix:** `SqlModelLoader` returns the model-loader connection for
  the metamodel connection UID, so metamodel reads **and** writes use a single
  session (Conn A). Teardown (`disconnectModelLoader()`) commits any open
  transaction and closes it, so nothing idle-with-open-transaction survives into
  the next app.

## TRUE mode — original code, without our changes (the freeze)

This shows the **unmodified** behaviour (before the `SqlModelLoader` fix and
before `stopWorkbenchIfNotGlobal()` / `disconnectModelLoader()`). With
`COMPOSER.USE_NEW_WORKBENCH_FOR_EVERY_APP = TRUE`, **every app gets its own new
workbench**, so every app opens a **new** pair of metamodel sessions: the
model-loader bootstrap connection (**Conn A**, reads) and a **separate**
connection from `SqlModelLoader::loadDataSource()` (**Conn B**, writes). There is
no teardown between apps, so app #1's Conn B is left **idle** ("sleeping") while
still holding locks on the metamodel tables. When app #2 starts a **fresh**
workbench and its **new** Conn B tries to write, it **blocks** on those leftover
locks — the install freezes. This is why it always hits the *second* app, not
the first.

```mermaid
sequenceDiagram
    autonumber
    participant Composer
    participant SI as StaticInstaller
    participant WB as Workbench
    participant ML as ModelLoader
    participant IA as InstallApp action
    participant DB as SQL Server (metamodel)
    participant AppDB as SQL Server (app data source)

    Composer->>SI: composerFinishInstall() / composerFinishUpdate()
    Note over SI: Loop: core first, then every app → new self() → install()

    rect rgb(255, 240, 240)
    Note over SI,DB: ── Install app #1 (new workbench) ──
    SI->>IA: installApp(app1) → getWorkbench()
    SI->>WB: Workbench::startNewInstance()
    activate WB
    WB->>ML: createModelLoaderConnection() = Conn A1
    IA->>DB: metamodel reads → OPEN Conn A1 (session 1)
    activate DB
    IA->>DB: metamodel writes via loadDataSource() → OPEN Conn B1 (session 2, separate)
    Note over IA,DB: startTransaction() on Conn B1 → write exf_object/…
    opt App has its own SQL installer (setDataSourceSelector)
        IA->>AppDB: OPEN Conn X → migrations → CLOSE Conn X
    end
    IA-->>SI: install(app1) returns
    Note over SI,WB: NO teardown (original code):<br/>workbench abandoned to GC, Conn A1 & Conn B1 left OPEN.<br/>Conn B1 stays idle ("sleeping") still holding locks on exf_* tables
    end

    rect rgb(255, 220, 220)
    Note over SI,DB: ── Install app #2 (another new workbench) ──
    SI->>WB: Workbench::startNewInstance() → Conn A2
    activate WB
    IA->>DB: metamodel reads → OPEN Conn A2 (session 3)
    IA->>DB: metamodel writes → OPEN Conn B2 (session 4)
    IA->>DB: INSERT exf_object/… on Conn B2
    DB--xIA: BLOCKED — waits on locks held by idle Conn B1 (app #1)
    Note over DB: session 4 "suspended" ⟂ session 2 "sleeping" → install FREEZES
    end
```

## FALSE mode (single shared workbench) — original code, without our changes


This shows the **unmodified** behaviour (before the `SqlModelLoader` and
`disconnectModelLoader()` changes). With
`COMPOSER.USE_NEW_WORKBENCH_FOR_EVERY_APP = FALSE`, the first `getWorkbench()`
stores the workbench in `StaticInstaller::$globalWorkbench` and **all apps reuse
it**. The metamodel still uses **two** sessions — the model-loader bootstrap
connection (**Conn A**, reads) and a **separate** connection opened by
`SqlModelLoader::loadDataSource()` (**Conn B**, writes) — but because the
workbench is shared, Conn A and Conn B are opened **once** and reused for every
app. All metamodel writes therefore run on the **one** Conn B session, which
never blocks on its own (already-committed) locks, so there is no freeze.
`stopWorkbenchIfNotGlobal()` is a no-op; the connections are closed only when
the process ends.

```mermaid
sequenceDiagram
    autonumber
    participant SI as StaticInstaller
    participant WB as Workbench (global)
    participant ML as ModelLoader
    participant IA as InstallApp action
    participant DB as SQL Server (metamodel)
    participant AppDB as SQL Server (app data source)

    SI->>WB: getWorkbench() (app1) → Workbench::startNewInstance()<br/>[StaticInstaller.php:504 → Workbench.php:179→130]
    activate WB
    Note over SI,WB: config FALSE → stored in $globalWorkbench,<br/>reused for ALL apps [StaticInstaller.php:510]
    WB->>ML: createModelLoaderConnection() = Conn A (not yet connected)<br/>[Workbench.php:157 → DataConnectionFactory.php:172]<br/>setDataConnection() [Workbench.php:158]

    Note over SI,DB: ── Install app #1 ── [StaticInstaller.php:449 $action->installApp()]
    IA->>DB: first metamodel read → OPEN Conn A (session 1)<br/>lazy connect [AbstractDataConnector.php:286 → MsSqlConnector.php:186]
    activate DB
    IA->>DB: metamodel writes via loadDataSource() → OPEN Conn B (session 2, separate)<br/>[SqlModelLoader.php:725; new conn :843-844]
    Note over IA,DB: MetaModelInstaller: startTransaction() on Conn B →<br/>write exf_object/… → commit() (locks released)<br/>[MetaModelInstaller.php:178 … :200]
    opt App has its own SQL installer (setDataSourceSelector)
        IA->>AppDB: OPEN Conn X → migrations → CLOSE Conn X<br/>[AbstractSqlDatabaseInstaller.php:124, :134/:230/:286]
    end

    Note over SI,DB: ── Install app #2 … appN ──
    Note over IA,DB: SAME global workbench → reuse Conn A + Conn B.<br/>Every app's metamodel writes run on the one Conn B session,<br/>so it never blocks on its own (already-committed) locks → no freeze.

    Note over SI,WB: original code: no per-app teardown (install() just returns)
    SI-->>WB: process ends → CLOSE Conn A + Conn B (+ pooled Conn X)<br/>[AbstractDataConnector.php:312 → MsSqlConnector.php:227]
    deactivate DB
    deactivate WB
```

### Code references for the FALSE (old-code) diagram

| Operation in diagram | Source location |
| --- | --- |
| `getWorkbench()` → `Workbench::startNewInstance()` | `StaticInstaller.php:504` → `Workbench.php:179` → `start()` `Workbench.php:130` |
| Store shared workbench (`$globalWorkbench`) when config is FALSE | `StaticInstaller.php:510` |
| Create **Conn A** (metamodel/model-loader connection) | `Workbench.php:157` → `DataConnectionFactory::createModelLoaderConnection()` `DataConnectionFactory.php:172`; bound at `Workbench.php:158` |
| Kick off the app install | `StaticInstaller.php:449` (`$action->installApp($app_selector)`) |
| Physically **OPEN Conn A** (lazy, on first query) | `AbstractDataConnector::connect()` `AbstractDataConnector.php:286` → `MsSqlConnector::performConnect()` `MsSqlConnector.php:186` |
| Create/**OPEN Conn B** (separate metamodel data-source connection) | `SqlModelLoader::loadDataSource()` `SqlModelLoader.php:725`; new connector at `:843` (`createDataConnectionFromDbRow`) and `:844` (`setConnection`) |
| `startTransaction()` … `commit()` on Conn B | `MetaModelInstaller.php:178` … `MetaModelInstaller.php:200` |
| **Conn X** (optional app SQL installer) | `AbstractSqlDatabaseInstaller::install()` `AbstractSqlDatabaseInstaller.php:124`, using `getDataConnection()` at `:134` / `:230` / `:286` |
| **CLOSE** connections at process shutdown | `AbstractDataConnector::disconnect()` `AbstractDataConnector.php:312` → `MsSqlConnector::performDisconnect()` `MsSqlConnector.php:227` |

> Note: line numbers reflect the current source. In the *old code* the metamodel
> data source really did open a second connection at `SqlModelLoader.php:843-844`
> (our fix later added a short-circuit just above that returns the model-loader
> connection instead), and `StaticInstaller::install()` simply returned after the
> install with no per-app teardown.






