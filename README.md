# hAI.DeepTutor

DeepTutor für das HomeLAN: eigener Quellcode-Build auf GitHub Actions, eigenes GHCR-Image und Portainer-Stack. Kein bloßes Umtaggen des Upstream-Images.

## Architektur

Der Workflow checkt HKUDS/DeepTutor am in upstream.env festgelegten Release-Tag aus und baut dessen Dockerfile mit target production. Zunächst nur linux/amd64; ARM64 ist nicht zugesichert. Der App-Container enthält kein lokal betriebenes Sprachmodell. Anbieter für LLM und Embeddings werden nach dem Start in den Web-Einstellungen eingerichtet.

Upstream: https://github.com/HKUDS/DeepTutor
Docker-Dokumentation: https://docs.deeptutor.info/get-started/docker/
Die Lizenz und Hinweise des Upstream-Projekts gelten für dessen Code. Dieses Repository behauptet keine eigene Urheberschaft am DeepTutor-Code.

## 1. Image bauen

Ein Commit an upstream.env oder den Build-Workflow startet den Build. Alternativ: Actions > Build and publish DeepTutor > Run workflow. Das Image steht erst nach erfolgreichem Workflow bereit:

    ghcr.io/jbkunama1/hai.deeptutor:latest

Zusätzlich wird ein Tag aus Upstream-Version und vollständiger Deployment-Commit-SHA veröffentlicht. Dieser kann für ein festes Deployment verwendet werden. Upstream-Release-Tags und Basisimages sind nicht digest-/commit-fixiert; dies ist kein vollständig reproduzierbarer Build.

Authentifizierung für den Push erfolgt über GITHUB_TOKEN mit packages:write, ohne zusätzlichen PAT im Repository. Ein neu erstelltes GHCR-Package kann privat sein, obwohl dieses Repository öffentlich ist. Für anonymes Pulling in den Package-Einstellungen die Sichtbarkeit auf Public setzen; andernfalls in Portainer eine GHCR-Registry mit einem passenden read:packages-Token hinterlegen. Keine Tokens ins Repository schreiben.

## 2. Portainer

Nach erfolgreichem Build: Stacks > Add stack > Repository. Repository-URL https://github.com/jbkunama1/hAI.DeepTutor verwenden, Referenz refs/heads/main und Compose-Pfad compose.yaml.

Alternativ compose.yaml im Web editor einfügen. Kein lokaler Build auf dem HomeLAN-Host nötig.

Optionale Stack-Variablen:

| Variable | Standard | Zweck |
| --- | --- | --- |
| IMAGE_TAG | latest | Alternativ festen Build-Tag verwenden |
| WEB_PORT | 3782 | Freier Port auf dem Host |
| BIND_IP | 0.0.0.0 | Besser gezielt LAN-IP; 127.0.0.1 bei lokalem Reverse Proxy |

Aufruf: http://SERVER-IP:3782. Backend-Port 8001 wird absichtlich nicht veröffentlicht. Das feste Volume hai-deeptutor-data enthält Einstellungen, API-Schlüssel, Dokumente und Indizes. Nur eine Instanz gleichzeitig mit diesem Volume betreiben.

## 3. Modelle anbinden

In den Web-Einstellungen Sprachmodell und Embedding-Anbieter getrennt konfigurieren. Auf demselben Docker-Host laufendes Ollama: LLM-Base-URL http://host.docker.internal:11434/v1; Ollama-Embeddings laut Upstream http://host.docker.internal:11434/api/embed. Für andere LAN-Server deren erreichbare IP verwenden. localhost bezeichnet im Container den Container selbst. Der Modellserver muss auf einer aus dem Docker-Netz erreichbaren Adresse lauschen; nicht ungeschützt ins Internet öffnen.

## Sicherheit

Nur im vertrauenswürdigen LAN starten, keine Router-Portfreigabe einrichten. BIND_IP=0.0.0.0 bindet an alle Host-Interfaces. Vor weiteren Benutzern oder externer Erreichbarkeit Authentifizierung gemäß Upstream einrichten. Auth-Einstellungen liegen unter /app/data/user/settings/auth.json; Aktivierung erfordert einen Neustart. Keine erfundenen AUTH_* Umgebungsvariablen verwenden: aktuelle Upstream-Container laden Einstellungen aus JSON.

Keine Docker-Socket-Mounts, kein privileged-Modus und keine Hostverzeichnisse mit persönlichen Daten einbinden. Der einfache Stack enthält keinen separaten gehärteten Code-Runner. Agenten-/Codeausführung nicht als sichere Sandbox für fremde Benutzer betrachten. Bei externen Modell-/Embedding-APIs können Dokumentinhalte das LAN verlassen.

## Updates und Rollback

Vor Updates Volume sichern. UPSTREAM_REF in upstream.env gezielt ändern und committen; nach erfolgreichem Build in Portainer Image neu pullen und Stack erneut bereitstellen. Der Workflow führt keine automatischen Upstream-Versionssprünge durch. Ein Rollback nutzt den vorherigen festen Image-Tag; bei inkompatiblen Datenänderungen zusätzlich die passende Datensicherung wiederherstellen.

## Backup

Stack vor konsistenter Sicherung stoppen. Auf dem Docker-Host ein geschütztes Backup-Verzeichnis verwenden:

    mkdir -p backups
    chmod 700 backups
    docker run --rm -v hai-deeptutor-data:/data:ro -v "$PWD/backups:/backup" alpine:3.20 tar czf /backup/deeptutor-data.tar.gz -C /data .

Archiv enthält möglicherweise API-Schlüssel und personenbezogene Daten: geschützt lagern und niemals committen. Danach Stack starten. Wiederherstellung nur bei gestoppter App in ein leeres Zielvolume und mit getesteter Sicherung durchführen.

## Diagnose

    docker ps --filter ancestor=ghcr.io/jbkunama1/hai.deeptutor:latest
    docker stats

Logs und Health-Status sind in Portainer sichtbar. Der Healthcheck wird aus dem Upstream-Produktionsimage übernommen. Bei Portkonflikt WEB_PORT ändern. Ein Healthcheck startet einen ungesunden Container nicht automatisch neu; restart: unless-stopped greift bei Prozessende. RAM-/CPU-Verbrauch mit wenigen Dokumenten messen, bevor größere Sammlungen importiert werden.

## Status

Konfiguration vorbereitet; erfolgreicher CI-Build und Laufzeittest müssen separat geprüft werden. Dieses Repository allein bedeutet nicht, dass bereits ein Image veröffentlicht oder ein HomeLAN-Stack ausgerollt wurde.
