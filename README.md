# MyStudyApp

## 1. Projektbeschreibung

MyStudyApp ist ein Softwaretechnik-2-Projekt zur digitalen Organisation des Studiums.

Die Anwendung ist als verteilte Web-/Backend-Anwendung aufgebaut. Es gibt zwei Spring-Backend-Systeme, einen MQTT-Broker, eine MariaDB-Datenbank, ein vorbereitetes Web-/PWA-Frontend und eine UML-Dokumentation.

Die beiden Backend-Systeme kommunizieren primär asynchron über den MQTT-Broker.

---

## 2. Ziel des Projekts

Ziel ist eine einfache, saubere und prüfungsnahe technische Grundlage für eine Studienverwaltungs-App.

Wichtige Ziele:

- Studieninformationen verwalten.
- Kurse, Stundenpläne, Termine und Dokumente bereitstellen.
- Termin-Ereignisse erzeugen und über MQTT veröffentlichen.
- Termin-Ereignisse im zweiten Backend empfangen.
- Erinnerungen und Benachrichtigungen vorbereiten.
- REST-Schnittstellen für die Businesslogik bereitstellen.
- Die Backend-Struktur nach Vertical Slicing aufbauen.
- Die Architektur mit UML dokumentieren.

---

## 3. Projektstruktur

```text
my-study-app/
├── pom.xml
├── README.md
├── .gitignore
│
├── backend-studium/
│   ├── pom.xml
│   └── src/main/java/de/fhdo/swt2/backendstudium/
│       ├── BackendStudiumAnwendung.java
│       ├── dokument/
│       ├── ereignis/
│       ├── fassade/
│       ├── kurs/
│       ├── mqtt/
│       ├── stundenplan/
│       └── termin/
│
├── backend-benachrichtigung/
│   ├── pom.xml
│   └── src/main/java/de/fhdo/swt2/backendbenachrichtigung/
│       ├── BackendBenachrichtigungAnwendung.java
│       ├── benachrichtigung/
│       ├── beobachter/
│       ├── empfaenger/
│       ├── erinnerung/
│       ├── mqtt/
│       └── terminereignis/
│
├── docs/
│   ├── api.md
│   ├── praktikum-1-2-kurzuebersicht.md
│   ├── praktikum-3-abgabe.md
│   ├── praktikum-4-abgabe.md
│   └── uml/
│
├── infrastructure/
│   ├── docker-compose.yml
│   └── mosquitto/config/mosquitto.conf
│
└── web-frontend/
    └── README.md
```

---

## 4. Technologien

| Technologie | Zweck |
|---|---|
| Java 21 | Implementierung der Backend-Systeme |
| Spring Boot | REST-Backends |
| Spring Web | REST-Schnittstellen |
| Spring Integration MQTT | MQTT-Anbindung |
| Maven | Multi-Modul-Build |
| Docker Compose | Start der Infrastruktur |
| MariaDB | Datenbank |
| Eclipse Mosquitto | MQTT-Broker |
| PlantUML | UML-Dokumentation |
| Web/PWA | vorbereitetes Frontend |

---

## 5. Backend-Systeme

### 5.1 Backend Studium

Modul:

```text
backend-studium
```

Aufgabe:

- REST-Schnittstellen für das Frontend bereitstellen.
- Stundenpläne, Kurse, Termine und Dokumente liefern.
- Neue Termine anlegen.
- Beim Anlegen eines Termins ein Studium-Ereignis erzeugen.
- Das Ereignis über MQTT veröffentlichen.

Wichtige Fachmodule:

| Modul | Aufgabe |
|---|---|
| `stundenplan` | Stundenplan-Endpunkte und Stundenplan-Logik |
| `kurs` | Kurs-Endpunkte und Kurs-Logik |
| `termin` | Termin-Endpunkte und Termin-Logik |
| `dokument` | Dokument-Endpunkte und Dokument-Logik |
| `ereignis` | Ereigniserzeugung mit Fabrikmethode |
| `mqtt` | Senden von MQTT-Ereignissen |
| `fassade` | einfache Schnittstelle über mehrere Dienste |

---

### 5.2 Backend Benachrichtigung

Modul:

```text
backend-benachrichtigung
```

Aufgabe:

- MQTT-Ereignisse empfangen.
- Termin-Ereignisse an Beobachter weitergeben.
- Erinnerungen erstellen.
- Benachrichtigungen erstellen und speichern.
- REST-Schnittstellen für Erinnerungen und Benachrichtigungen bereitstellen.

Wichtige Fachmodule:

| Modul | Aufgabe |
|---|---|
| `mqtt` | MQTT-Empfang und Ereignisquelle |
| `beobachter` | Beobachter-Interface |
| `erinnerung` | Erinnerungslogik und Erinnerungs-API |
| `benachrichtigung` | Benachrichtigungslogik, Ablage und API |
| `terminereignis` | Datenklasse für Termin-Ereignisse |
| `empfaenger` | Datenklasse für Empfänger |

---

## 6. REST-Schnittstellen

### Backend Studium

| Methode | Pfad | Bedeutung |
|---|---|---|
| GET | `/api/stundenplan` | Stundenplan anzeigen |
| GET | `/api/kurse` | Kurse anzeigen |
| GET | `/api/termine` | Termine anzeigen |
| POST | `/api/termine` | neuen Termin anlegen und MQTT-Ereignis senden |
| GET | `/api/dokumente` | Dokumente anzeigen |
| GET | `/api/studium/uebersicht` | Übersicht über mehrere Studiendaten |

### Backend Benachrichtigung

| Methode | Pfad | Bedeutung |
|---|---|---|
| GET | `/api/erinnerungen` | Erinnerungen anzeigen |
| GET | `/api/benachrichtigungen` | Benachrichtigungen anzeigen |

---

## 7. MQTT-Kommunikation

Die Backends kommunizieren nicht direkt miteinander.

Ablauf:

```text
1. Das Backend Studium legt einen Termin an.
2. TerminDienst erzeugt ein StudiumEreignis.
3. MqttEreignisSender veröffentlicht das Ereignis.
4. Eclipse Mosquitto verteilt die Nachricht.
5. Backend Benachrichtigung empfängt die Nachricht.
6. MqttTerminEreignisQuelle informiert die Beobachter.
7. ErinnerungDienst erstellt eine Erinnerung.
8. BenachrichtigungDienst erstellt eine Benachrichtigung.
```

Standard-Topic:

```text
mystudyapp/termine
```

---

## 8. Entwurfsmuster aus Praktikum 3

| Kategorie | Muster | Umsetzung |
|---|---|---|
| Strukturmuster | Fassade | `StudiumFassade` |
| Erzeugungsmuster | Fabrikmethode | `StudiumEreignisFabrik`, `TerminEreignisFabrik`, `StudiumEreignis` |
| Verhaltensmuster | Beobachter | `MqttTerminEreignisQuelle`, `TerminEreignisBeobachter`, `ErinnerungDienst` |

Das Singleton-Muster wird nicht verwendet.

---

## 9. Build

Im Hauptordner ausführen:

```bash
mvn clean install
```

Erwartetes Ergebnis:

```text
BUILD SUCCESS
```

---

## 10. Infrastruktur starten

Im Hauptordner ausführen:

```bash
docker compose -f infrastructure/docker-compose.yml up -d
```

Prüfen:

```bash
docker compose -f infrastructure/docker-compose.yml ps
```

Erwartet:

```text
mystudyapp-mariadb   Up
mystudyapp-mqtt      Up
```

Stoppen:

```bash
docker compose -f infrastructure/docker-compose.yml down
```

---

## 11. Backends starten

Backend Studium:

```bash
cd backend-studium
mvn spring-boot:run
```

Backend Benachrichtigung:

```bash
cd backend-benachrichtigung
mvn spring-boot:run
```

Ports:

| Backend | Port |
|---|---|
| Backend Studium | 8080 |
| Backend Benachrichtigung | 8081 |

---

## 12. Schnelle REST-Tests

```bash
curl http://localhost:8080/api/kurse
curl http://localhost:8080/api/stundenplan
curl http://localhost:8080/api/termine
curl http://localhost:8080/api/dokumente
curl http://localhost:8080/api/studium/uebersicht
curl http://localhost:8081/api/erinnerungen
curl http://localhost:8081/api/benachrichtigungen
```

Neuen Termin anlegen:

```bash
curl -X POST http://localhost:8080/api/termine \
  -H "Content-Type: application/json" \
  -d '{"titel":"SWT2 Praktikum","datum":"2026-05-20","uhrzeit":"10:00"}'
```
