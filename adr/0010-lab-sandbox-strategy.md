# ADR 0010 — Lab Sandbox Stratejisi

- **Status:** Accepted

## Karar
**Aşamalı sandbox** — 3 fazda büyür:

### Faz 1 (Sprint 4) — Tarayıcı içi (in-browser)
- HTML/CSS/JS → iframe srcdoc (sandboxed)
- Python → **Pyodide** (Python WASM)
- SQL → **sql.js** (SQLite WASM)

**Avantaj:** sıfır server maliyeti, sıfır güvenlik riski (browser sandbox).
**Dezavantaj:** Bash/Docker/Linux CLI imkansız.

### Faz 2 (Sprint ~10) — Container terminal
- Bash, Docker, Linux CLI için ephemeral Docker container
- xterm.js + WebSocket bridge
- Per-session container, 15 dk timeout
- gVisor runtime (extra security)

**Avantaj:** TryHackMe seviyesinde lab.
**Dezavantaj:** server CPU/RAM, container orchestration karmaşıklığı.

### Faz 3 (Sprint ~15) — Cloud lab
- AWS/Azure/GCP gerçek API'leri için **LocalStack** (AWS emulator)
- Per-user sahte AWS hesabı, gerçek CLI
- Premium tier feature

## Faz 1 Detayı (şu an planlananın)

### HTML/CSS Lab
```html
<iframe sandbox="allow-scripts" srcdoc="<user-html>">
```

### JS Lab
```js
// Sandboxed eval, console capture
const logs = [];
const sandbox = new Function('console', userCode);
sandbox({ log: (...args) => logs.push(args.join(' ')) });
```

### Python Lab (Pyodide)
```js
const pyodide = await loadPyodide();
const result = await pyodide.runPythonAsync(userCode);
```

### SQL Lab (sql.js)
```js
const SQL = await initSqlJs();
const db = new SQL.Database();
// Pre-load schema + data
db.run(seedSql);
const result = db.exec(userQuery);
```

## Güvenlik

### Faz 1 (in-browser)
- Sandbox attribute (iframe)
- Time limit (Web Worker timeout)
- Memory limit (Pyodide heap)

### Faz 2 (container)
- gVisor / Firecracker runtime
- No network egress
- Read-only filesystem (writable /tmp)
- ulimit (CPU, memory, processes)
- Container destroy after timeout

## Test Case Execution

- **Python:** stdin → user code → stdout match
- **SQL:** expected JSON ↔ actual rows diff
- **HTML/CSS:** visual snapshot (Playwright, faz 2)

## Cost Model

- Faz 1: zero ops cost (browser)
- Faz 2: ~$50/ay başlangıç (Hetzner CX21 + Docker)
- Faz 3: AWS API gerçek lab — premium tier, kullanıcıdan ödeme
