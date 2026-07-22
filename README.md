# MyStudyApp

MyStudyApp ist eine verteilte Webanwendung zur Organisation des Studiums. Die Anwendung
bÃ¼ndelt Stundenplan, FÃ¤cher, Gruppen, Termine, Erinnerungen und Benachrichtigungen in
einer gemeinsamen OberflÃ¤che fÃ¼r Desktop- und Laptop-Browser.

Das Projekt besteht aus einem Webfrontend, zwei Spring-Boot-Backends, einem MQTT-Broker
und zwei persistenten H2-Datenbanken. Das Backend Studium Ã¼bernimmt zusÃ¤tzlich die Rolle
eines Backends fÃ¼r das Frontend und liefert Ã¼ber eine Fassade eine gemeinsame
StudienÃ¼bersicht.

## Inhalt

- [Funktionen](#funktionen)
- [Systemarchitektur](#systemarchitektur)
- [Technologien](#technologien)
- [Entwurfsmuster](#entwurfsmuster)
- [Eigene Datenstruktur](#eigene-datenstruktur)
- [Projektstruktur](#projektstruktur)
- [Schnellstart mit Docker Compose](#schnellstart-mit-docker-compose)
- [Adressen und Ports](#adressen-und-ports)
- [REST-Schnittstellen](#rest-schnittstellen)
- [Datenhaltung](#datenhaltung)
- [Projekt Ã¼bertragen](#projekt-auf-einen-anderen-rechner-Ã¼bertragen)
- [Lokaler Maven-Build](#lokaler-maven-build)
- [Fehlerbehebung](#fehlerbehebung)

## Funktionen

### Dashboard

- Anzahl der Stundenplan-EintrÃ¤ge anzeigen
- Anzahl der gespeicherten FÃ¤cher anzeigen
- Anzahl der fÃ¤lligen Erinnerungen anzeigen
- TagesÃ¼bersicht der nÃ¤chsten Stundenplan-EintrÃ¤ge anzeigen
- nÃ¤chsten gespeicherten Termin anzeigen
- direkt zu Stundenplan, Studienplanung und Benachrichtigungen wechseln

### Stundenplan

- Wochenansicht von Montag bis Freitag anzeigen
- zwischen vorheriger, aktueller und nÃ¤chster Woche wechseln
- Stundenplan-EintrÃ¤ge nach Datum und Uhrzeit einordnen
- Termine zusÃ¤tzlich in der Wochenansicht anzeigen

### Studienplanung

- Stundenplan-EintrÃ¤ge anlegen, bearbeiten und lÃ¶schen
- FÃ¤cher mit Fachname und Lehrperson verwalten
- Gruppen mit Gruppenname und Personenanzahl verwalten
- vorhandene FÃ¤cher und Gruppen auswÃ¤hlen
- beim Planen direkt ein neues Fach oder eine neue Gruppe anlegen
- gespeicherte FÃ¤cher und Gruppen durchsuchen

### Termine

- Termine Ã¼ber die REST-Schnittstelle anlegen, lesen, bearbeiten und lÃ¶schen
- Termine im Dashboard und in der Wochenansicht darstellen
- beim Anlegen eines neuen Termins automatisch ein MQTT-Ereignis verÃ¶ffentlichen

### Erinnerungen und Benachrichtigungen

- gespeicherte Erinnerungen und Benachrichtigungen gemeinsam anzeigen
- Hinweise nach â€žAlleâ€œ, â€žBenachrichtigungenâ€œ oder â€žErinnerungenâ€œ filtern
- Hinweise manuell neu laden
- aus einem neuen Termin automatisch eine Erinnerung erzeugen
- aus derselben Erinnerung automatisch eine sichtbare Benachrichtigung erzeugen

## Systemarchitektur

```mermaid
flowchart TB
    U[Desktop- oder Laptop-Browser]
    F[Webfrontend<br/>Port 5500]
    S[Backend Studium<br/>BFF und REST<br/>Port 8080]
    HS[(H2-Datei<br/>backend-studium.mv.db)]
    M[Eclipse Mosquitto<br/>MQTT-Broker<br/>Port 1883]
    B[Backend Benachrichtigung<br/>REST und MQTT<br/>Port 8081]
    HB[(H2-Datei<br/>backend-benachrichtigung.mv.db)]

    U --> F
    F -->|REST und JSON| S
    F -->|REST und JSON| B
    S --> HS
    S -->|Publish: mystudyapp/termine| M
    M -->|Subscribe: mystudyapp/termine| B
    B --> HB
```

### Ablauf beim Anlegen eines Termins

1. Ein neuer Termin wird an `POST /api/termine` gesendet.
2. Das Backend Studium prÃ¼ft und speichert den Termin.
3. `TerminEreignisFabrik` erzeugt ein `StudiumEreignis`.
4. `MqttEreignisSender` verÃ¶ffentlicht die Nachricht auf `mystudyapp/termine`.
5. Eclipse Mosquitto leitet die Nachricht an das Backend Benachrichtigung weiter.
6. `MqttEreignisEmpfaenger` wandelt die MQTT-Nachricht in ein `TerminEreignis` um.
7. `MqttTerminEreignisQuelle` informiert den angemeldeten `ErinnerungDienst`.
8. Der `ErinnerungDienst` speichert eine Erinnerung.
9. Der `BenachrichtigungDienst` speichert zusÃ¤tzlich eine sichtbare Benachrichtigung.
10. Das Frontend lÃ¤dt beide Hinweise Ã¼ber REST.

## Technologien

| Bereich                           | Verwendete Technik                        |
| --------------------------------- | ----------------------------------------- |
| Programmiersprache                | Java 21                                   |
| Backend                           | Spring Boot 3.3.5                         |
| Webschnittstellen                 | Spring Web, REST und JSON                 |
| Datenzugriff                      | Spring JDBC mit `JdbcTemplate`            |
| Standard-Datenbank                | H2 im Dateimodus                          |
| Optionale Datenbank-Infrastruktur | MariaDB 11.4                              |
| Asynchrone Kommunikation          | MQTT mit Spring Integration               |
| MQTT-Broker                       | Eclipse Mosquitto 2.0                     |
| Frontend                          | HTML5, CSS3 und JavaScript ohne Framework |
| Frontend-Auslieferung             | Nginx 1.27 Alpine                         |
| Frontend-Grundlage                | Manifest, App-Symbole und Service Worker  |
| Build                             | Maven-Multi-Modul-Projekt                 |
| Container                         | Docker und Docker Compose                 |
| Automatischer Build               | GitHub Actions mit Temurin Java 21        |
| UML                               | PlantUML                                  |

Die WeboberflÃ¤che ist fÃ¼r Desktop und Laptop vorgesehen. `manifest.json` und
`service-worker.js` stellen zusÃ¤tzlich die technische Grundlage fÃ¼r eine installierbare
Webanwendung und den Cache der statischen Frontend-Dateien bereit.

## Entwurfsmuster

| Muster        | Kategorie        | Umsetzung im Projekt                                                                                              |
| ------------- | ---------------- | ----------------------------------------------------------------------------------------------------------------- |
| Fassade       | Strukturmuster   | `StudiumFassade` bÃ¼ndelt Stundenplan-, Kurs-, Gruppen- und Termin-Dienst fÃ¼r `/api/studium/uebersicht`.           |
| Fabrikmethode | Erzeugungsmuster | `StudiumEreignisFabrik` und `TerminEreignisFabrik` erzeugen das Ereignis fÃ¼r einen neuen Termin.                  |
| Adapter       | Strukturmuster   | `MqttEreignisSender` und `MqttEreignisEmpfaenger` trennen die Fachlogik von der MQTT-Technik.                     |
| Beobachter    | Verhaltensmuster | `MqttTerminEreignisQuelle` informiert `TerminEreignisBeobachter`; `ErinnerungDienst` ist der konkrete Beobachter. |

Das Singleton-Muster wird nicht verwendet. Die Lebenszyklen der Spring-Komponenten
werden durch den Spring-Container verwaltet.

## Eigene Datenstruktur

Die Backends verwenden intern keine `ArrayList`. Stattdessen besitzt jedes Backend eine
selbst programmierte, einfach verkettete `EigeneListe` mit `ListenElement`.

```mermaid
flowchart LR
    A[Fachlicher Wert] --> B[Passende Fachliste]
    B --> C[EigeneListe]
    C --> D[ListenElement 1]
    D --> E[ListenElement 2]
    E --> F[weiteres Element oder null]
```

Die zentrale Liste arbeitet ohne Generics und ohne Java-Listenklasse. Sie speichert
intern `Object`-Werte, wÃ¤hrend fachlich getrennte Listen nur den jeweils passenden Typ
nach auÃŸen anbieten:

- `StundenplanListe`
- `KursListe`
- `GruppeListe`
- `TerminListe`
- `BenachrichtigungListe`
- `TerminEreignisBeobachterListe`

Die grundlegenden Listenoperationen sind:

- Wert am Ende hinzufÃ¼gen
- Wert an einer Position lesen
- Anzahl der Werte liefern
- Beobachter im Backend Benachrichtigung entfernen

An der REST-Ausgabegrenze werden die Inhalte in fachliche Felder wie `Kurs[]`,
`Termin[]` oder `Benachrichtigung[]` umgewandelt. Dadurch bleiben die internen
Listenklassen einfach und fachlich getrennt.

## Projektstruktur

```text
my-study-app/
â”œâ”€â”€ .github/
â”‚   â””â”€â”€ workflows/
â”‚       â””â”€â”€ maven-build.yml
â”œâ”€â”€ backend-studium/
â”‚   â”œâ”€â”€ Dockerfile
â”‚   â”œâ”€â”€ pom.xml
â”‚   â””â”€â”€ src/main/
â”‚       â”œâ”€â”€ java/de/fhdo/swt2/backendstudium/
â”‚       â””â”€â”€ resources/
â”‚           â”œâ”€â”€ application.properties
â”‚           â”œâ”€â”€ application-mariadb.properties
â”‚           â”œâ”€â”€ schema.sql
â”‚           â””â”€â”€ data.sql
â”œâ”€â”€ backend-benachrichtigung/
â”‚   â”œâ”€â”€ Dockerfile
â”‚   â”œâ”€â”€ pom.xml
â”‚   â””â”€â”€ src/main/
â”‚       â”œâ”€â”€ java/de/fhdo/swt2/backendbenachrichtigung/
â”‚       â””â”€â”€ resources/
â”‚           â”œâ”€â”€ application.properties
â”‚           â”œâ”€â”€ application-mariadb.properties
â”‚           â””â”€â”€ schema.sql
â”œâ”€â”€ datenbank/
â”‚   â”œâ”€â”€ backend-studium.mv.db
â”‚   â””â”€â”€ backend-benachrichtigung.mv.db
â”œâ”€â”€ docker-backup/
â”œâ”€â”€ docs/
â”‚   â””â”€â”€ uml/
â”œâ”€â”€ infrastructure/
â”‚   â””â”€â”€ mosquitto/config/mosquitto.conf
â”œâ”€â”€ web-frontend/
â”‚   â”œâ”€â”€ design/
â”‚   â”œâ”€â”€ icons/
â”‚   â”œâ”€â”€ Dockerfile
â”‚   â”œâ”€â”€ index.html
â”‚   â”œâ”€â”€ style.css
â”‚   â”œâ”€â”€ app.js
â”‚   â”œâ”€â”€ manifest.json
â”‚   â””â”€â”€ service-worker.js
â”œâ”€â”€ .dockerignore
â”œâ”€â”€ .gitignore
â”œâ”€â”€ DATENBANK-INFO.txt
â”œâ”€â”€ docker-compose.yml
â”œâ”€â”€ pom.xml
â””â”€â”€ README.md
```

Die fachlichen Backend-Module folgen jeweils derselben klaren Richtung:

```text
REST-Controller â†’ Dienst â†’ Ablage â†’ Datenbank
```

Die Projektarbeit wird ausschlieÃŸlich Ã¼ber GitHub Issues und GitHub Milestones
organisiert.

## Schnellstart mit Docker Compose

### Voraussetzung

BenÃ¶tigt wird nur eine laufende Docker-Umgebung mit Docker Compose, zum Beispiel Docker
Desktop. Java und Maven mÃ¼ssen fÃ¼r diesen Startweg nicht lokal installiert sein, weil
die Backend-Dockerfiles den Maven-Build und Java 21 bereits enthalten.

### Projekt starten

Im Hauptordner `my-study-app` ausfÃ¼hren:

```bash
docker compose up --build
```

Der erste Build kann lÃ¤nger dauern, weil Docker-Images und Maven-AbhÃ¤ngigkeiten geladen
werden.

Danach im Browser Ã¶ffnen:

```text
http://localhost:5500
```

### Im Hintergrund starten

```bash
docker compose up --build -d
```

Status aller Container anzeigen:

```bash
docker compose ps
```

### Projekt beenden

```bash
docker compose down
```

Dieser Befehl entfernt die Container und das Docker-Netzwerk, aber nicht die H2-Dateien
im Ordner `datenbank/`.

### Protokolle anzeigen

```bash
docker compose logs -f
```

Nur die beiden Backends und den MQTT-Broker beobachten:

```bash
docker compose logs -f backend-studium backend-benachrichtigung mqtt-broker
```

## Adressen und Ports

| Komponente               | Adresse oder Port       | Aufgabe                                                                   |
| ------------------------ | ----------------------- | ------------------------------------------------------------------------- |
| Webfrontend              | `http://localhost:5500` | sichtbare MyStudyApp-WeboberflÃ¤che                                        |
| Backend Studium          | `http://localhost:8080` | Studienfunktionen und BFF-/Fassaden-API                                   |
| Backend Benachrichtigung | `http://localhost:8081` | Erinnerungs- und Benachrichtigungs-API                                    |
| MQTT-Broker              | `localhost:1883`        | asynchrone Termin-Ereignisse                                              |
| MariaDB-Container        | `localhost:3306`        | vorbereitete, im Standardbetrieb nicht verwendete Datenbank-Infrastruktur |

Wichtig: `http://localhost:8080/` und `http://localhost:8081/` besitzen keine
HTML-Startseite. Eine Spring-Whitelabel-Seite oder `404 Not Found` an diesen beiden
Stammadressen ist deshalb normal. Die sichtbare Anwendung lÃ¤uft auf Port `5500`; die
Backends liefern Daten ausschlieÃŸlich unter `/api/...`.

## REST-Schnittstellen

### Backend Studium â€“ Port 8080

| Methode  | Pfad                      | Aufgabe                                                  |
| -------- | ------------------------- | -------------------------------------------------------- |
| `GET`    | `/api/studium/uebersicht` | Stundenplan, FÃ¤cher, Gruppen und Termine gemeinsam laden |
| `GET`    | `/api/stundenplan`        | alle Stundenplan-EintrÃ¤ge laden                          |
| `POST`   | `/api/stundenplan`        | Stundenplan-Eintrag anlegen                              |
| `PUT`    | `/api/stundenplan/{id}`   | Stundenplan-Eintrag bearbeiten                           |
| `DELETE` | `/api/stundenplan/{id}`   | Stundenplan-Eintrag lÃ¶schen                              |
| `GET`    | `/api/kurse`              | alle FÃ¤cher laden                                        |
| `POST`   | `/api/kurse`              | Fach anlegen                                             |
| `PUT`    | `/api/kurse/{id}`         | Fach bearbeiten                                          |
| `DELETE` | `/api/kurse/{id}`         | Fach lÃ¶schen                                             |
| `GET`    | `/api/gruppen`            | alle Gruppen laden                                       |
| `POST`   | `/api/gruppen`            | Gruppe anlegen                                           |
| `PUT`    | `/api/gruppen/{id}`       | Gruppe bearbeiten                                        |
| `DELETE` | `/api/gruppen/{id}`       | Gruppe lÃ¶schen                                           |
| `GET`    | `/api/termine`            | alle Termine laden                                       |
| `POST`   | `/api/termine`            | Termin anlegen und MQTT-Ereignis auslÃ¶sen                |
| `PUT`    | `/api/termine/{id}`       | Termin bearbeiten                                        |
| `DELETE` | `/api/termine/{id}`       | Termin lÃ¶schen                                           |

Das Frontend verwendet zusÃ¤tzlich die vorhandenen Routen `einfach-speichern`,
`einfach-bearbeiten` und `einfach-loeschen` mit URL-Parametern. Die REST-Controller
unterstÃ¼tzen auÃŸerdem die alternativen `POST`-Routen `bearbeiten/{id}` und
`loeschen/{id}`.

### Backend Benachrichtigung â€“ Port 8081

| Methode | Pfad                      | Aufgabe                                       |
| ------- | ------------------------- | --------------------------------------------- |
| `GET`   | `/api/benachrichtigungen` | alle Benachrichtigungen laden                 |
| `POST`  | `/api/benachrichtigungen` | Benachrichtigung anlegen                      |
| `GET`   | `/api/erinnerungen`       | alle automatisch erzeugten Erinnerungen laden |

### Wichtige API-Tests

```bash
curl http://localhost:8080/api/studium/uebersicht
curl http://localhost:8080/api/stundenplan
curl http://localhost:8080/api/kurse
curl http://localhost:8080/api/gruppen
curl http://localhost:8080/api/termine
curl http://localhost:8081/api/benachrichtigungen
curl http://localhost:8081/api/erinnerungen
```

Wenn der Browser bei diesen Adressen JSON-Text anzeigt, arbeiten die REST-Schnittstellen
korrekt.

### Neuen Termin anlegen

```bash
curl -X POST http://localhost:8080/api/termine \
  -H "Content-Type: application/json" \
  -d '{"titel":"PrÃ¼fungstermin","datum":"2026-08-20","uhrzeit":"10:00"}'
```

Nach erfolgreicher Verarbeitung enthÃ¤lt das Backend Studium den Termin. Bei aktivem MQTT
entstehen im Backend Benachrichtigung zusÃ¤tzlich eine Erinnerung und eine
Benachrichtigung.

### Neues Fach anlegen

```bash
curl -X POST http://localhost:8080/api/kurse \
  -H "Content-Type: application/json" \
  -d '{"name":"Softwaretechnik 2","dozent":"Prof. Beispiel"}'
```

### Neue Gruppe anlegen

```bash
curl -X POST http://localhost:8080/api/gruppen \
  -H "Content-Type: application/json" \
  -d '{"name":"Lerngruppe A","beschreibung":"Gemeinsames Lernen","personenAnzahl":4}'
```

## Datenhaltung

### Aktiver Standardbetrieb

Beide Backends verwenden standardmÃ¤ÃŸig eine eigene persistente H2-Datenbank im
Dateimodus:

```text
datenbank/backend-studium.mv.db
datenbank/backend-benachrichtigung.mv.db
```

Docker bindet den lokalen Ordner `datenbank/` als `/app/datenbank` in beide
Backend-Container ein. Dadurch bleiben die Daten nach `docker compose down`, einem
Neustart oder einem erneuten Build erhalten.

### Tabellen

| Backend                  | Tabelle            | Inhalt                                     |
| ------------------------ | ------------------ | ------------------------------------------ |
| Backend Studium          | `stundenplan`      | EintrÃ¤ge der Wochenplanung                 |
| Backend Studium          | `kurs`             | FÃ¤cher und Lehrpersonen                    |
| Backend Studium          | `gruppe`           | Gruppen und Personenanzahl                 |
| Backend Studium          | `termin`           | einzelne Termine                           |
| Backend Benachrichtigung | `erinnerung`       | automatisch erzeugte Erinnerungen          |
| Backend Benachrichtigung | `benachrichtigung` | sichtbare Benachrichtigungen und EmpfÃ¤nger |

Die beiden `schema.sql`-Dateien legen fehlende Tabellen beim Start automatisch an.
`backend-studium/src/main/resources/data.sql` ergÃ¤nzt sechs FÃ¤cher und sechs Gruppen als
Startdaten, sofern die jeweiligen IDs noch nicht vorhanden sind. Bereits vorhandene
EintrÃ¤ge werden nicht Ã¼berschrieben.

### MariaDB

`docker-compose.yml` startet zusÃ¤tzlich einen MariaDB-Container mit einem eigenen
Docker-Volume. Die beiden Backends sind im aktuellen Standardbetrieb jedoch ausdrÃ¼cklich
mit den H2-Dateien verbunden. Die Dateien `application-mariadb.properties` und der
MariaDB-Treiber bereiten einen spÃ¤teren Wechsel vor; ohne eine bewusste Profil- und
Verbindungsumstellung wird MariaDB von den Backends nicht verwendet.

## Projekt auf einen anderen Rechner Ã¼bertragen

1. Alle Container sauber beenden:

   ```bash
   docker compose down
   ```

2. Den vollstÃ¤ndigen Ordner `my-study-app` kopieren oder als ZIP-Datei Ã¼bertragen.
3. Darauf achten, dass der Ordner `datenbank/` mit beiden `.mv.db`-Dateien enthalten
   ist.
4. Auf dem anderen Rechner den Projektordner Ã¶ffnen.
5. Im Hauptordner erneut starten:

   ```bash
   docker compose up --build
   ```

FÃ¼r den normalen Projektbetrieb ist kein separater Datenbankexport nÃ¶tig, weil die
aktiven H2-Daten direkt im Projektordner liegen. Der Ordner `docker-backup/` enthÃ¤lt
zusÃ¤tzlich vorhandene Sicherungsarchive der Docker-Infrastruktur.

## Lokaler Maven-Build

FÃ¼r die Entwicklung ohne Docker werden Java 21 und Maven benÃ¶tigt.

Im Hauptordner ausfÃ¼hren:

```bash
mvn clean install
```

Der Maven-Reaktor baut beide Module:

1. `backend-studium`
2. `backend-benachrichtigung`

Der gleiche Befehl wird bei Pushes und Pull Requests auf `main` durch
`.github/workflows/maven-build.yml` ausgefÃ¼hrt.

FÃ¼r den vollstÃ¤ndigen Systembetrieb einschlieÃŸlich Frontend, MQTT und korrekter
Container-Konfiguration wird der Docker-Compose-Start empfohlen.

## Fehlerbehebung

### Port 8080 oder 8081 zeigt eine Whitelabel-Seite

Das ist an der Stammadresse normal. Verwende:

```text
WeboberflÃ¤che:  http://localhost:5500
Studien-API:    http://localhost:8080/api/studium/uebersicht
Hinweise-API:   http://localhost:8081/api/benachrichtigungen
```

### Im Browser werden nur JSON-Daten angezeigt

Das ist bei einer `/api/...`-Adresse korrekt. JSON ist die Datenausgabe des Backends.
Die gestaltete WeboberflÃ¤che befindet sich unter `http://localhost:5500`.

### Das Frontend meldet, dass ein Backend nicht erreichbar ist

Containerstatus prÃ¼fen:

```bash
docker compose ps
```

Backend-Protokolle prÃ¼fen:

```bash
docker compose logs backend-studium backend-benachrichtigung
```

AnschlieÃŸend die Seite Ã¼ber die SchaltflÃ¤che â€žNeu ladenâ€œ oder mit einer vollstÃ¤ndigen
Browser-Aktualisierung erneut laden.

### Die API wird beim direkten Ã–ffnen der HTML-Datei blockiert

`index.html` nicht Ã¼ber `file://` Ã¶ffnen. Die CORS-Konfiguration erlaubt das Frontend
unter diesen beiden UrsprÃ¼ngen:

```text
http://localhost:5500
http://127.0.0.1:5500
```

### Ein Port ist bereits belegt

MyStudyApp benÃ¶tigt die Ports `5500`, `8080`, `8081`, `1883` und `3306`.

Unter macOS oder Linux prÃ¼fen:

```bash
lsof -nP -iTCP:5500 -sTCP:LISTEN
lsof -nP -iTCP:8080 -sTCP:LISTEN
lsof -nP -iTCP:8081 -sTCP:LISTEN
lsof -nP -iTCP:1883 -sTCP:LISTEN
lsof -nP -iTCP:3306 -sTCP:LISTEN
```

Unter Windows PowerShell prÃ¼fen:

```powershell
netstat -ano | findstr :5500
netstat -ano | findstr :8080
netstat -ano | findstr :8081
netstat -ano | findstr :1883
netstat -ano | findstr :3306
```

Danach den belegenden Prozess oder die andere Docker-Umgebung beenden und MyStudyApp neu
starten:

```bash
docker compose down --remove-orphans
docker compose up --build
```

### Alte Frontend-Dateien werden angezeigt

Der Service Worker speichert statische Frontend-Dateien im Browsercache. Eine
vollstÃ¤ndige Aktualisierung ausfÃ¼hren oder die Website-Daten fÃ¼r `localhost:5500` im
Browser lÃ¶schen.

### Die H2-Datenbank ist gesperrt

Nicht gleichzeitig dieselben H2-Dateien mit Docker und einer zweiten lokal gestarteten
Backend-Instanz Ã¶ffnen. Eine der beiden Umgebungen vollstÃ¤ndig beenden und danach nur
eine Umgebung starten.

### Docker verwendet noch einen alten Backend-Build

```bash
docker compose down
docker compose build --no-cache
docker compose up
```
