# hAI.DeepTutor

DeepTutor im HomeLAN mit eigenem GitHub-Actions-Quellcode-Build, GHCR-Image und Portainer-Stack. Standarddatei: docker-compose.yaml.

## Portainer

Nach erfolgreichem Build: Stacks > Add stack > Repository.

| Feld | Wert |
| --- | --- |
| Repository URL | https://github.com/jbkunama1/hAI.DeepTutor |
| Repository reference | refs/heads/main |
| Compose path | docker-compose.yaml |
| Host-Port | 8042 |
| Container-Port | 3782 |
| Externes Docker-Netzwerk | highfishNetwork |
| Persistentes Volume | hai-deeptutor-data |

Aufruf: http://SERVER-IP:8042. Das externe Netzwerk muss auf dem Zielhost bereits existieren. Falls es fehlt, in Portainer als Bridge-Netzwerk highfishNetwork anlegen. Kein Host-Build notwendig. Backend-Port 8001 wird nicht veröffentlicht. Die alte compose.yaml nicht mehr für Deployments verwenden.

## Welche ENVs?

Keine Pflicht-ENVs in Portainer. Alle Werte haben Vorgaben:

| Variable | Standard | Zweck |
| --- | --- | --- |
| IMAGE_TAG | latest | Optional festen Build-Tag verwenden |
| WEB_PORT | 8042 | Port auf dem Docker-Host |
| BIND_IP | 0.0.0.0 | Alle Host-Interfaces; alternativ gezielt LAN-IP |

TZ=Europe/Berlin ist im Container gesetzt. Das Netzwerk ist fest highfishNetwork. Keine LLM_API_KEY-, OPENAI_API_KEY- oder AUTH_*-Variablen für diesen Stack erforderlich. Modellanbieter, Sprachmodell, Embeddings und API-Schlüssel werden nach dem Start über die DeepTutor-Web-Einstellungen konfiguriert und im Datenvolume gespeichert. Aktuelle Upstream-Container laden ihre Laufzeitkonfiguration aus JSON. Eine optionale .env.example liegt bei; Portainer-Nutzer können die optionalen Werte direkt unter Environment variables setzen.

## Build und GHCR

upstream.env legt den Upstream-Tag fest, zunächst v1.7.1. Änderungen daran oder am Build-Workflow lösen einen Build aus; alternativ Actions > Build and publish DeepTutor > Run workflow. Der Workflow checkt HKUDS/DeepTutor aus und baut dessen Dockerfile mit target production für linux/amd64. Kein bloßes Umtaggen, kein lokales Sprachmodell im App-Image. ARM64 ist nicht zugesichert.

Image nach erfolgreichem Build:

    ghcr.io/jbkunama1/hai.deeptutor:latest

Zusätzlich gibt es einen Tag aus Upstream-Version und Deployment-Commit-SHA. Für reproduzierbarere Deployments diesen statt latest einsetzen; Release-Tags und Basisimages sind jedoch nicht vollständig commit-/digest-fixiert. Push über GITHUB_TOKEN mit packages:write, kein zusätzlicher PAT im Repo nötig. Neu erzeugte GHCR-Packages können trotz öffentlichem Repo privat sein: Package für anonymes Pulling auf Public setzen oder Portainer mit passendem read:packages-Token an GHCR anmelden.

## Lokale Modelle

LLM und Embeddings separat einrichten. Bei Ollama auf demselben Docker-Host kann die OpenAI-kompatible LLM-URL http://host.docker.internal:11434/v1 verwendet werden; Embedding-Endpunkt passend zum gewählten Anbieter konfigurieren. Für andere LAN-Server deren erreichbare Adresse verwenden. localhost im Container bezeichnet nicht den Docker-Host. Modellserver nur im vertrauenswürdigen Netz erreichbar machen.

## Sicherheit

8042 ist der veröffentlichte Host-Port, keine automatische Internetfreigabe. Keine Router-Portfreigabe einrichten. BIND_IP=0.0.0.0 bindet an alle Host-Interfaces. Vor weiteren Benutzern oder externer Erreichbarkeit Authentifizierung gemäß Upstream einrichten und nötigenfalls neu starten. Konfiguration unter /app/data/user/settings/, unter anderem auth.json und model_catalog.json.

Keine Docker-Socket-Mounts, kein privileged-Modus, keine persönlichen Hostverzeichnisse einbinden. Dieser einfache Stack enthält keinen separaten gehärteten Code-Runner: Agenten-/Codeausführung ist keine zugesicherte sichere Sandbox für fremde Benutzer. Bei externen Modell- und Embedding-APIs können Dokumentinhalte das LAN verlassen. Das gemeinsam verwendete highfishNetwork ersetzt keine Sicherheitsisolation zwischen Containern.

## Updates, Backup und Rollback

Vor Updates Stack stoppen und Datenvolume konsistent sichern. Auf dem Docker-Host:

    mkdir -p backups
    chmod 700 backups
    docker run --rm -v hai-deeptutor-data:/data:ro -v "$PWD/backups:/backup" alpine:3.20 tar czf /backup/deeptutor-data.tar.gz -C /data .

Danach Stack starten. Archiv enthält möglicherweise API-Schlüssel und personenbezogene Daten: geschützt lagern, nie committen. Wiederherstellung nur bei gestoppter App in ein leeres Zielvolume und mit getesteter Sicherung. Nur eine Instanz gleichzeitig mit hai-deeptutor-data betreiben.

Für Updates UPSTREAM_REF in upstream.env gezielt ändern, committen und erfolgreichen Build abwarten. Dann in Portainer Image neu pullen und Stack erneut bereitstellen. Keine automatischen Upstream-Versionssprünge. Rollback über vorherigen festen Image-Tag, bei inkompatiblen Datenänderungen zusätzlich passende Sicherung.

## Diagnose und Status

Logs und Health-Status in Portainer prüfen. Healthcheck wird vom Upstream-Image übernommen; restart: unless-stopped startet einen bloß ungesunden Container nicht automatisch neu. Mit docker stats Ressourcenverbrauch messen. Bei Portkonflikt WEB_PORT ändern.

Ein erfolgreicher Commit beweist weder einen erfolgreichen CI-Build noch ein veröffentlichtes Image oder einen funktionierenden HomeLAN-Stack; diese Schritte separat prüfen.

Upstream: https://github.com/HKUDS/DeepTutor

Dokumentation: https://docs.deeptutor.info/get-started/docker/

Lizenz und Hinweise des Upstream-Projekts gelten für dessen Code; dieses Repository behauptet keine eigene Urheberschaft daran.
