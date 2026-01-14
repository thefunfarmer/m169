# M169 Abschlussprojekt (Docker Compose Infrastruktur)

## Ziel und Aufgabenstellung
Für ein Informatik-KMU werden mehrere Dienste als Container-Infrastruktur bereitgestellt:

- **MediaWiki** als firmeninternes Wiki, erreichbar auf **Port 8085**
- **Nextcloud** als Open Source Filesharing und Collaboration, erreichbar auf **Port 8080**
- **CVS / Codeverwaltung** als interne Git-Plattform via **Gitea**, erreichbar auf **Port 3000**
- **Portainer** zur Überwachung und Verwaltung der Container, erreichbar auf **Port 9443** (HTTPS) und optional **9000** (HTTP)
- **Persistenz**: Alle Daten müssen persistent gespeichert werden
- **Versionsverwaltung**: Notwendige Dateien sowie Systemdokumentation werden im privaten GitHub Repository versioniert
- **Security**: Keine Passwörter im Git (Secrets nur lokal)

Dieses Projekt setzt die Infrastruktur als **Infrastructure as Code (IaC)** mit Docker Compose um.

---

## Ordnerstruktur
- `compose/`  
  Docker Compose Datei und Umgebungsvariablen (`.env.example`, lokal `.env`)
- `config/`  
  Platzhalter für Konfigurationsnotizen oder spätere Erweiterungen
- `docs/`  
  Systemdokumentation, Screenshots, Testprotokolle, Arbeitsjournal

---

## Architekturüberblick und Zusammenspiel
Die Infrastruktur ist als Docker Compose Stack aufgebaut und nutzt zwei Netzwerke:

- **frontend**: Dienste, die per Browser erreichbar sein sollen (Web UIs)
- **backend (internal)**: interne Kommunikation, z.B. Datenbanken und Redis, nicht direkt von aussen erreichbar

**Datenfluss (vereinfacht):**
- Browser → `nextcloud` → `nextcloud-db` (MariaDB) und `nextcloud-redis` (Redis)
- Browser → `gitea` → `gitea-db` (Postgres)
- Browser → `mediawiki` → `mediawiki-db` (MariaDB)
- Browser → `portainer` → Docker API (via `/var/run/docker.sock`)

---

## Container und Ports

### Portainer (Monitoring)
- Zweck: Überwachung und Verwaltung der Container und Stacks
- Ports:
  - `9443` (HTTPS)
  - `9000` (HTTP, optional)
- Persistenz:
  - Named Volume: `portainer_data`

Wichtig: Portainer nutzt den Docker Socket (`/var/run/docker.sock`). Das ist praktisch, aber sicherheitsrelevant, da es sehr weitreichende Rechte bedeutet. In diesem Projekt ist es bewusst so umgesetzt und wird im Sicherheitskonzept beschrieben.

### Gitea (CVS / interne Git-Plattform)
- Zweck: internes Code-Hosting, Web UI und Repositories
- Ports:
  - `3000` (Web UI)
  - `2222` (SSH, optional für Git over SSH)
- Datenbank:
  - `gitea-db` (Postgres)
- Persistenz:
  - `gitea_data` (Gitea Daten)
  - `gitea_db` (Postgres Daten)

Hinweis: Repositories werden im Normalfall im Web UI erstellt. Ein Push zum automatischen Erstellen eines neuen Repos ist in Gitea standardmässig nicht aktiv.

### Nextcloud (Filesharing und Collaboration)
- Zweck: Filesharing, Web UI, Kollaboration
- Port:
  - `8080` (Web UI)
- Nebencontainer:
  - `nextcloud-db` (MariaDB)
  - `nextcloud-redis` (Redis)
  - `nextcloud-cron` (Cron für Hintergrundjobs)
- Persistenz:
  - `nextcloud_data`, `nextcloud_db`, `nextcloud_redis`

### MediaWiki (firmeninternes Wiki)
- Zweck: Wissensdatenbank für interne Dokumentation (ähnlich Wikipedia)
- Port:
  - `8085` (Web UI)
- Datenbank:
  - `mediawiki-db` (MariaDB)
- Persistenz:
  - `mediawiki_data` (Wiki Dateien inkl. `LocalSettings.php`)
  - `mediawiki_db` (MariaDB Daten)

---

## Installation und Inbetriebnahme

### 1) Voraussetzungen
- Docker und Docker Compose installiert
- Projektordner: `Abschlussprojekt/compose/` enthält die Compose Datei

### 2) Secrets lokal setzen (wichtig: nicht committen)
Im Ordner `compose/`:

- `compose/.env.example` ist eine Vorlage
- `compose/.env` wird lokal erstellt und bleibt ausserhalb von Git

```bash
cd Abschlussprojekt/compose
cp .env.example .env
```

Danach in `compose/.env` Passwörter setzen, z.B.:
- `GITEA_DB_PASSWORD`
- `NEXTCLOUD_DB_PASSWORD`
- `NEXTCLOUD_DB_ROOT_PASSWORD`
- `MEDIAWIKI_DB_PASSWORD`

### 3) Stack starten
```bash
cd Abschlussprojekt/compose
docker compose up -d
docker compose ps
```

---

## Wichtige Hinweise bei den Setup Wizards

### MediaWiki Datenbank Host (kein localhost)
Im MediaWiki Setup Wizard muss als Datenbank Host **der Containername** verwendet werden:

- Datenbank Host: `mediawiki-db`
- Datenbank Name: `mediawiki`
- Benutzer: `mediawiki`
- Passwort: aus `compose/.env`

`localhost` oder `127.0.0.1` sind hier falsch, weil MediaWiki im Container läuft und damit `localhost` auf den MediaWiki Container selbst zeigt, nicht auf die Datenbank.

Nach dem Wizard wird eine Datei `LocalSettings.php` zum Download angeboten. Diese Datei muss ins MediaWiki Volume kopiert werden, damit die Konfiguration persistent bleibt.

Beispiel:
```bash
docker cp ~/Downloads/LocalSettings.php m169-mediawiki:/var/www/html/LocalSettings.php
docker exec m169-mediawiki chown www-data:www-data /var/www/html/LocalSettings.php
```

### Nextcloud Datenbank Host
Im Nextcloud Setup Wizard:
- Datenbank Host: `nextcloud-db`
- Datenbank Name: `nextcloud`
- Benutzer: `nextcloud`
- Passwort: aus `compose/.env`

---

## Persistenz (Nachweis)
Die Persistenz wird über Docker Named Volumes umgesetzt. Ein einfacher Nachweis:

1) In Nextcloud oder MediaWiki eine Änderung machen (z.B. Seite oder Datei)
2) Container neu starten:
```bash
cd Abschlussprojekt/compose
docker compose restart
```
3) Die Änderung ist weiterhin vorhanden

---

## Ressourcenbeschränkungen
Das Docker Compose File enthält pro Service Ressourcenbeschränkungen (CPU und RAM), um ein kontrolliertes Laufzeitverhalten sicherzustellen und die Bewertungsanforderungen zu erfüllen. Die Limits können mit folgendem Befehl geprüft werden:

```bash
docker stats --no-stream
```

---

## Bedienung: Wichtige URLs
- Portainer: https://localhost:9443
- Nextcloud: http://localhost:8080
- MediaWiki: http://localhost:8085
- Gitea: http://localhost:3000

---

## Stopp und Cleanup
Stoppen (Container bleiben vorhanden):
```bash
cd Abschlussprojekt/compose
docker compose stop
```

Komplett runterfahren:
```bash
docker compose down
```

Achtung: Volumes bleiben bei `down` erhalten (das ist gewollt für Persistenz).

