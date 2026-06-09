# MyStudyApp

## 1. Projektziel

MyStudyApp ist eine verteilte Web-/Backend-Anwendung für die Organisation des Studiums. Die Anwendung besitzt ein modernes responsives Web-/PWA-Frontend, zwei Spring-Boot-Backends, eine MariaDB-Datenbank und einen MQTT-Broker.

Studierende können folgende Bereiche nutzen:

- Stundenplan
- Kurse
- Termine
- Dokumente
- Erinnerungen
- Benachrichtigungen

## 2. Vollständige Architektur

```text
Browser mit Web-/PWA-Frontend
        |
        | REST / JSON
        v
Backend Studium :8080 ---- speichert ----> MariaDB :3306
        |
        | MQTT-Ereignis über mystudyapp/termine
        v
Eclipse Mosquitto :1883
        |
        v
Backend Benachrichtigung :8081 ---- speichert ----> MariaDB :3306
```

## 3. Wichtige technische Entscheidungen

| Thema | Umsetzung |
|---|---|
| Backend | Java 21 und Spring Boot |
| Build | Maven-Multi-Modul-Projekt |
| Frontend | HTML5, CSS3 und JavaScript |
| PWA | Manifest, Service Worker und App-Symbole |
| Kommunikation | REST und asynchrones MQTT |
| Datenhaltung | MariaDB mit SQL-Initialisierung |
| Datenstruktur | selbst programmierte `EigeneListe` mit `ListenElement`, ohne Generics und ohne gemischte Listen |
| Infrastruktur | Docker Compose |
| UML | PlantUML-Dateien und Erzeugungsskript |

## 4. EigeneListe statt ArrayList

In beiden Backends gibt es eine selbst programmierte Liste:

```text
gemeinsam/liste/
├── EigeneListe.java
└── ListenElement.java
```

Die Backends verwenden intern keine `ArrayList`. Die `EigeneListe` enthält nur die Funktionen, die im Projekt wirklich gebraucht werden:

- Werte am Ende hinzufügen,
- Werte über eine Stelle lesen,
- die Anzahl liefern.

Im Backend Benachrichtigung kann zusätzlich ein MQTT-Beobachter entfernt werden. Dadurch wird der Beobachter beim Start sauber angemeldet und beim Beenden sauber abgemeldet. Unbenutzte Listenmethoden wurden entfernt.

Die Liste verwendet keine Generics und keine Java-Listenklasse. Die zentrale `EigeneListe` ist nur intern sichtbar. Fachlich getrennte Klassen wie `TerminListe`, `KursListe` und `DokumentListe` verhindern gemischte Inhalte. Für REST werden die Werte erst an der Ausgabegrenze in fachliche Felder wie `Termin[]` umgewandelt.

## 5. MariaDB-Anbindung

MariaDB ist nicht mehr nur vorbereitet. Beide Backends verwenden die Datenbank aktiv.

Tabellen des Backend Studium:

```text
stundenplan
kurs
termin
dokument
```

Tabellen des Backend Benachrichtigung:

```text
erinnerung
benachrichtigung
```

Die Dateien `schema.sql` erstellen fehlende Tabellen. Die Datei `data.sql` des Backend Studium fügt übersichtliche Startdaten ein, ohne vorhandene Daten zu überschreiben.

## 6. Projektstruktur

```text
my-study-app/
├── pom.xml
├── README.md
├── starten.sh
├── stoppen.sh
├── projekt_pruefen.sh
│
├── backend-studium/
│   ├── pom.xml
│   └── src/main/
│       ├── java/de/fhdo/swt2/backendstudium/
│       │   ├── gemeinsam/liste/
│       │   ├── dokument/
│       │   ├── ereignis/
│       │   ├── fassade/
│       │   ├── konfiguration/
│       │   ├── kurs/
│       │   ├── mqtt/
│       │   ├── stundenplan/
│       │   └── termin/
│       └── resources/
│           ├── application.properties
│           ├── schema.sql
│           └── data.sql
│
├── backend-benachrichtigung/
│   ├── pom.xml
│   └── src/main/
│       ├── java/de/fhdo/swt2/backendbenachrichtigung/
│       │   ├── gemeinsam/liste/
│       │   ├── benachrichtigung/
│       │   ├── beobachter/
│       │   ├── empfaenger/
│       │   ├── erinnerung/
│       │   ├── konfiguration/
│       │   ├── mqtt/
│       │   └── terminereignis/
│       └── resources/
│           ├── application.properties
│           ├── schema.sql
│           └── data.sql
│
├── infrastructure/
│   ├── docker-compose.yml
│   └── mosquitto/config/mosquitto.conf
│
├── web-frontend/
│   ├── index.html
│   ├── style.css
│   ├── app.js
│   ├── manifest.json
│   ├── service-worker.js
│   ├── design/ui-konzept.png
│   └── icons/
│
└── docs/
    ├── api.md
    ├── datenbank.md
    ├── eigene-liste.md
    ├── frontend.md
    ├── startanleitung.md
    ├── pruefbericht.md
    └── uml/
```

## 7. Einfachster Start

Voraussetzungen:

- Docker Desktop ist geöffnet.
- Java 21 ist installiert.
- Maven ist installiert.
- Python 3 ist installiert.

Im Hauptordner ausführen:

```bash
chmod +x starten.sh stoppen.sh projekt_pruefen.sh
./starten.sh
```

Danach öffnen:

```text
http://localhost:5500
```

Das Skript startet automatisch in der richtigen Reihenfolge:

1. MariaDB und Eclipse Mosquitto,
2. Maven-Build,
3. Backend Benachrichtigung,
4. Backend Studium,
5. Frontend.

Beenden:

```bash
./stoppen.sh
```

## 8. Manuelle REST-Tests

```bash
curl http://localhost:8080/api/kurse
curl http://localhost:8080/api/stundenplan
curl http://localhost:8080/api/termine
curl http://localhost:8080/api/dokumente
curl http://localhost:8080/api/studium/uebersicht
curl http://localhost:8081/api/erinnerungen
curl http://localhost:8081/api/benachrichtigungen
```

Neuen Termin erstellen:

```bash
curl -X POST http://localhost:8080/api/termine \
  -H "Content-Type: application/json" \
  -d '{"titel":"SWT2 Praktikum","datum":"2026-06-11","uhrzeit":"10:00"}'
```

## 9. Entwurfsmuster

| Kategorie | Muster | Umsetzung |
|---|---|---|
| Strukturmuster | Fassade | `StudiumFassade` |
| Erzeugungsmuster | Fabrikmethode | `StudiumEreignisFabrik`, `TerminEreignisFabrik` |
| Verhaltensmuster | Beobachter | `MqttTerminEreignisQuelle`, `TerminEreignisBeobachter`, `ErinnerungDienst` |

Das Singleton-Muster wird nicht verwendet.
