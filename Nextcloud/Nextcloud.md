# M300 – Nextcloud als containerisierte Cloud-Lösung

---

# 1. Dienstleistungskonzept

## Ziel der Lösung

In diesem Projekt wurde eine private Cloud-Lösung mit **Nextcloud** containerisiert umgesetzt.

Ziel ist es, eine selbst gehostete Alternative zu Google Drive oder OneDrive bereitzustellen, bei der:

- Daten vollständig unter eigener Kontrolle bleiben
- Benutzer zentral verwaltet werden können
- Datei-Synchronisation möglich ist
- WebDAV-Unterstützung integriert ist
- Erweiterungen (Apps) flexibel installiert werden können

Die gesamte Infrastruktur läuft in Docker-Containern und ist dadurch:

- reproduzierbar
- portierbar
- isoliert
- einfach wartbar

---

# 2. Architektur & Aufbau

## Verwendete Komponenten

| Komponente | Aufgabe |
|------------|----------|
| Nextcloud Container | Webserver + Applikation |
| MariaDB Container | Datenbank |
| Docker Netzwerk | Interne Kommunikation |
| Docker Volume | Persistente Speicherung |
| cAdvisor | Monitoring |

---

## Architekturübersicht

Host-System  
→ Docker Engine  
→ Docker Netzwerk `nextcloud-net`  
→ MariaDB Container  
→ Nextcloud Container  
→ Zugriff über `http://localhost:8082`  
→ Monitoring über `http://localhost:8085`

---

# 3. Installation & Aufsetzung

## Docker Netzwerk erstellen

```bash
docker network create nextcloud-net
```

---

## MariaDB Container starten

```bash
docker run -d \
--name nextcloud_db \
--network nextcloud-net \
-e MYSQL_ROOT_PASSWORD=admin \
-e MYSQL_DATABASE=nextcloud \
-e MYSQL_USER=nextcloud \
-e MYSQL_PASSWORD=secret \
--restart=always \
mariadb:latest
```

Status prüfen:

```bash
docker logs nextcloud_db
```

Wichtige Meldung:

```
ready for connections
```

Die Datenbank ist **nur intern im Docker-Netzwerk erreichbar**.

---

## Nextcloud Container starten

```bash
docker run -d \
--name nextcloud \
--network nextcloud-net \
-p 8082:80 \
-e MYSQL_HOST=nextcloud_db \
-e MYSQL_DATABASE=nextcloud \
-e MYSQL_USER=nextcloud \
-e MYSQL_PASSWORD=secret \
-v nextcloud_data:/var/www/html \
--restart=always \
nextcloud:latest
```

Zugriff im Browser:

```
http://localhost:8082
```

### Weboberfläche

Nach erfolgreichem Start erscheint die Nextcloud-Weboberfläche:

![Nextcloud Login](Images/nextcloud.png)

---

# 4. Persistenz (Volumes)

Das Volume:

```
nextcloud_data
```

speichert:

- Benutzerdaten
- Konfiguration
- Installierte Apps
- Uploads

Auch nach einem Neustart der Container bleiben alle Daten erhalten.

---

# 5. Monitoring mit cAdvisor

```bash
docker run -d \
--name cadvisor \
--restart=always \
-p 8085:8080 \
-v /:/rootfs:ro \
-v /var/run:/var/run:rw \
-v /sys:/sys:ro \
-v /var/lib/docker/:/var/lib/docker:ro \
gcr.io/cadvisor/cadvisor:latest
```

Zugriff:

```
http://localhost:8085
```

### Monitoring Übersicht

cAdvisor zeigt:

- CPU-Auslastung
- RAM-Verbrauch
- Netzwerktraffic
- Container-Status

![cAdvisor Monitoring](Images/cadvisor.png)

---

# 6. Sicherheitskonzept

## Netzwerk-Isolation

- Eigenes Docker-Netzwerk `nextcloud-net`
- Datenbank nicht extern freigegeben
- Zugriff nur über Nextcloud-Webserver

---

## Zugriffsschutz

- Admin-Account bei Erstinstallation
- Restart-Policy aktiviert (`--restart=always`)
- Container isoliert vom Host-System

---

## Erweiterte Sicherheitsmassnahmen (empfohlen)

Für Produktivbetrieb:

- HTTPS via Reverse Proxy (z.B. Nginx)
- Firewall-Regeln
- Regelmässige Updates
- Starke Passwörter
- Backup-Rotation

---

# 7. Backup-Strategie

## Datenbank Backup

```bash
docker exec nextcloud_db mysqldump -u root -p nextcloud > backup.sql
```

---

## Volume Backup

```bash
docker run --rm \
-v nextcloud_data:/data \
-v ${PWD}:/backup \
ubuntu \
tar czf /backup/nextcloud_backup.tar.gz /data
```

Damit können sowohl Datenbank als auch Dateien wiederhergestellt werden.

---

# 8. WebDAV Integration

Windows Dienst prüfen:

```powershell
Get-Service WebClient
```

Status:

```
Running
```

Netzlaufwerk verbinden:

```
http://localhost:8082/remote.php/dav/files/<username>
```

### Netzlaufwerk Einbindung

Nextcloud kann als Netzlaufwerk eingebunden werden:

![WebDAV Verbindung](Images/netzlaufwerk.png)

---

# 9. Fehleranalyse

### Problem

Beim ersten Start konnte keine Verbindung zur Datenbank hergestellt werden.

Browsermeldung:

```
Error while trying to create admin account
```

---

### Ursache

- Falscher MYSQL_HOST
- Container nicht im gleichen Netzwerk

---

### Lösung

- Beide Container im Netzwerk `nextcloud-net`
- MYSQL_HOST korrekt auf `nextcloud_db` gesetzt

Nach Neustart funktionierte die Verbindung korrekt.

---

# 10. Ressourcen-Management

Standardmässig dürfen Docker-Container unbegrenzt CPU und RAM verwenden.

Empfohlene Begrenzung:

```bash
--memory=512m
--cpus=1
```

Damit wird verhindert, dass ein Container das Hostsystem überlastet.

---

# 11. Reflexion

Durch dieses Projekt habe ich gelernt:

- Container sauber zu vernetzen
- Persistente Volumes korrekt einzusetzen
- Monitoring mit cAdvisor umzusetzen
- Sicherheitsaspekte zu berücksichtigen
- Fehler systematisch mittels Logs zu analysieren

Die Lösung ist:

- Stabil
- Reproduzierbar
- Isoliert
- Erweiterbar

Nextcloud eignet sich hervorragend als selbst gehostete Cloud-Lösung für kleine und mittlere Unternehmen.