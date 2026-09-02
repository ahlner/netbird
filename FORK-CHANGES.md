# Fork-Änderungen ggü. netbirdio/netbird

Zentrale Übersicht aller Eigenänderungen des Forks `github.com/ahlner/netbird`.
Kein Upstream-PR — die Änderungen werden per Merge/Rebase auf Upstream-Stand
weitergepflegt (siehe „Wartung"). Detailpläne pro Feature: `PLAN-*.md`.

| # | Änderung | Status | Kern-Dateien |
| --- | --- | --- | --- |
| 1 | Traffic-Flow-Logging (Management-injizierte Flow-Konfig) | gemergt auf `origin/main` (`b10e53168`, PR #1) | `management/fork/integrations/`, `go.mod` |
| 2 | Management-Docker-Image (GitHub → ghcr.io) | implementiert, uncommittet | `.github/workflows/management-image.yml`, `management/Dockerfile.fork` |

## 1. Traffic-Flow-Logging

Vollständiger Plan und Env-Referenz: **`PLAN-traffic-flow.md`** (Teil A umgesetzt,
Teil B „Flow-Receiver" noch nicht gebaut).

- Eigenes Replace-Modul `management/fork/integrations/` (Modulpfad
  `github.com/netbirdio/management-integrations/integrations`), aktiviert durch
  genau eine `replace`-Zeile in der Root-`go.mod`. Keine anderen OSS-Dateien
  angefasst. Lizenz: GPL-3.0 wie das gespiegelte Upstream-Modul
  (`LICENSE` im Modulverzeichnis); liegt unter `management/`, außerhalb des
  Top-Level-Scans des internen AGPL-Dependencies-Checks.
- Aktivierungsregel: Flow nur aktiv, wenn `NB_FLOW_GROUPS` **und**
  `NB_FLOW_RECEIVER_URL` **und** `NB_FLOW_SIGNING_KEY` gesetzt sind, sonst
  deaktiviert + Warnlog. Peers melden, wenn ihre Gruppen `NB_FLOW_GROUPS`
  schneiden; Toggle zur Laufzeit über Gruppenmitgliedschaft (nächster Sync).
- Env-Variablen: `NB_FLOW_GROUPS`, `NB_FLOW_RECEIVER_URL`, `NB_FLOW_SIGNING_KEY`
  (Pflicht), `NB_FLOW_INTERVAL` (Default `5m`), `NB_FLOW_DNS_COLLECTION`,
  `NB_FLOW_EXITNODE_COLLECTION` (je Default `false`).
- `UpdateExtraSettings` bleibt No-op → REST-PUT auf
  `settings.extra.network_traffic_logs_*` wirkt nicht (bewusst, POC).

## 2. Management-Docker-Image (GitHub → ghcr.io)

- Ziel: Image mit den Fork-Features, automatisiert von GitHub gebaut, ohne
  Secrets (nur das eingebaute `GITHUB_TOKEN`).
- Upstreams `management/Dockerfile.multistage` funktioniert am Fork nicht:
  Stage 1 kopiert nur `go.mod`/`go.sum` und scheitert dann am
  `go mod download`, weil das lokale Replace `./management/fork/integrations`
  zu diesem Zeitpunkt nicht im Build-Kontext liegt. `management/Dockerfile.fork`
  kopiert das Modulverzeichnis vor dem Download mit (einziger Unterschied).
- Workflow `.github/workflows/management-image.yml`: push auf `main` (nur bei
  Änderungen unter `management/**`, `go.mod`, `go.sum`) + `workflow_dispatch`;
  baut `linux/amd64` und pusht nach
  `ghcr.io/ahlner/netbird-management:{latest,sha-<short>}`; Buildx-Cache via
  GHA. Aktionen wie im Repo üblich auf SHAs gepinnt. Kein Docker-Hub, kein
  GPG, kein Tag nötig — bewusst ohne die Upstream-Release-Pipeline, die am
  Fork an fehlenden Secrets scheitert.
- Paket-Sichtbarkeit: ghcr-Pakete sind initial privat; beim ersten Lauf
  erzeugt, Sichtbarkeit ggf. unter Package-Settings auf public stellen oder
  Client mit PAT bei `ghcr.io` einloggen.
- Arm64 wäre möglich (`platforms` erweitern), ist wegen CGO unter QEMU aber
  langsam — Server-typisch amd64.

## Wartung (Upstream-Rebase)

1. `git fetch upstream && git merge upstream/main` (Fork-main ist Merge-basiert,
  PR #1-Verlauf bleibt erhalten).
2. Nach dem Merge prüfen: `replace … => ./management/fork/integrations` noch in
   der Root-`go.mod`; `management/fork/integrations/` kompiliert gegen geänderte
   Upstream-API der Integrations-Schnittstelle (brechen laut statt still,
   Dateilayout ist gespiegelt).
3. Docker-Image: `Dockerfile.fork`/`management-image.yml` sind reine Fork-Dateien
   ohne Upstream-Kollision; nach Upstream-Änderungen an
   `management/Dockerfile.multistage` (Go-Version, Build-Flags, Base-Image)
   parallel pflegen.
4. Verifikation pro Änderung wie im jeweiligen Abschnitt; danach
   `git push origin main`.
