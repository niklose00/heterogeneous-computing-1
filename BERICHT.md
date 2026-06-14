# Ergebnisbericht – Heterogene Smart-Home-Infrastruktur

**Heterogeneous Computing · Übungsblatt 1 (Aufgabe 1b) · Sommersemester 2026**

## 1. Einleitung

Smart-Home-Systeme sind ein typisches heterogenes, verteiltes System: Geräte
unterschiedlicher Hersteller, Protokolle und Leistungsklassen müssen lokal wie
über die Cloud zusammenspielen. Diese Arbeit recherchiert bestehende
Architekturen und Technologien (Kap. 2), leitet daraus Anforderungen ab (Kap. 3)
und setzt diese in einem prototypischen Demonstrator um (Kap. 4–5). Physische
Sensoren/Aktoren sind nicht erforderlich und werden als Software-Mockups
simuliert; der Fokus liegt auf den architektonischen Prinzipien.

## 2. Recherche

**Architekturtypen.** Cloud-zentrisch (Alexa, Google Home), lokal/Edge-first
(Home Assistant, openHAB) und hybride Mischformen. Der Trend 2026 geht klar zu
**hybriden, lokal-first Ansätzen** (Datenschutz, Latenz).

**Plattformen.**

| Plattform           | Typ            | Charakteristik                          | Relevanz 2026 |
|---------------------|----------------|-----------------------------------------|---------------|
| Home Assistant      | Lokal          | Große Community, starke Matter-Stützung | Hoch          |
| openHAB             | Lokal          | Hohe Flexibilität                       | Mittel        |
| Apple Home          | Hybrid         | Gute Matter-/Thread-Unterstützung       | Hoch          |
| Google Home / Alexa | Cloud-dominant | Einfache Einrichtung, große Auswahl     | Mittel        |

**Kommunikationsmodelle.** Es dominieren **Publish-Subscribe** (Entkopplung,
Skalierbarkeit; v. a. MQTT) und **Request-Response** (HTTP/REST, CoAP). Im
Smart-Home hat sich Pub/Sub als besonders geeignet erwiesen, da es gut mit
heterogenen, ressourcenbeschränkten Geräten umgeht – die Grundlage für die
Architekturentscheidung in Kap. 4.

**Protokolle.**

| Protokoll  | Typ                  | Stärken / Einsatz 2026                                       |
|------------|----------------------|-------------------------------------------------------------|
| **Matter** | Application Layer    | Herstellerübergreifender Standard – stärkster Trend         |
| **Thread** | Network Layer (Mesh) | Energieeffizient, selbstheilend; bevorzugt für Batteriegeräte |
| **MQTT**   | Application Layer    | Leichtgewichtig, skalierbar; sehr weit verbreitet           |
| **Zigbee** | Mesh                 | Große installierte Basis, wird durch Matter/Thread ergänzt  |
| **Wi-Fi**  | Network              | Hohe Bandbreite; für stromintensive Geräte                  |

Matter ist kein Funkprotokoll, sondern ein Application-Layer-Standard über
Thread, Wi-Fi oder Ethernet, der vor allem die Interoperabilität verbessert.

**Cloud vs. Edge.**

| Aspekt            | Cloud-zentrisch | Edge-first | Hybrid       |
|-------------------|-----------------|------------|--------------|
| Latenz            | Mittel–hoch     | Sehr niedrig | Niedrig    |
| Datenschutz       | Schwach         | Sehr gut   | Gut          |
| Offline-Fähigkeit | Schlecht        | Sehr gut   | Gut          |
| Skalierbarkeit    | Sehr hoch       | Begrenzt   | Hoch         |
| Trend 2026        | Rückläufig      | Steigend   | **Dominant** |

**Interoperabilität.** Lange ein Problem durch proprietäre Protokolle; Lösungen
heute: Matter als zentraler Standard, Brücken/Gateways (z. B. Zigbee2MQTT),
Multi-Protokoll-Plattformen (Home Assistant), Thread Border Router. Reale Systeme
2026 sind dennoch meist hybrid.

**Trends 2025–2026.** Matter + Thread als De-facto-Interoperabilitätsbasis;
Zunahme lokaler Systeme (Datenschutz); Edge-Computing und lokale KI; mit Thread
1.4 standardisiertes Teilen von Netzwerk-Credentials (plus Energieeffizienz als
Nebeneffekt); Multi-Admin-Fähigkeit; Standardisierung statt proprietärer Lösungen.

## 3. Abgeleitete Anforderungen

Aus der Recherche ergeben sich folgende zentrale Anforderungen an eine moderne
Smart-Home-Infrastruktur:

| Anforderung | Kurzbeschreibung | Grundlage (Recherche) |
|-------------|------------------|-----------------------|
| Lose Kopplung | Komponenten unabhängig entwickel- und ersetzbar; Kommunikation über definierte Schnittstellen ohne direkte Sender-Empfänger-Bindung | Dominanz von Pub/Sub (2.3) |
| Heterogenität | Geräte, Hersteller und Protokolle integrierbar; keine Bindung an ein Ökosystem | Interoperabilitätsproblematik (2.6) |
| Skalierbarkeit | wachsende Geräte-, Nutzer- und Nachrichtenzahl ohne Performanceeinbruch | Pub/Sub + Cloud-Skalierung (2.3, 2.5) |
| Verteilung | dezentrale, lokal autonome Verarbeitung – auch bei Ausfall von Cloud/Hub | Edge-first, Offline-Fähigkeit (2.5) |
| Fehlertoleranz / Robustheit | Teilausfälle führen nicht zum Gesamtausfall; resilientes, konsistentes Zustandsmanagement | Bedeutung der Offline-Fähigkeit (2.5) |
| Erweiterbarkeit | neue Geräte, Protokolle und Dienste ohne Änderung am Kern; modularer Aufbau | Trend zur Standardisierung (2.7) |

Ergänzend: **Datenschutz** durch lokale Verarbeitung (2.5),
**Interoperabilität** über einheitliche Standards (2.6) sowie **Wartbarkeit/
Beobachtbarkeit** (Logging, klare Schnittstellen).

## 4. Architektur

### 4.1 Entwurfsprinzip

Kern ist ein **nachrichtenbasierter Bus (MQTT)**. Alle Teilnehmer – simulierte
Geräte wie Backend-Dienste – kommunizieren ausschließlich über Topics; sie
kennen einander nicht direkt. Damit ist lose Kopplung konstruktiv verankert. Ein
zentral definiertes Nachrichtenformat (Topic-Schema + JSON-Envelope, siehe
`common/topics.py`) bildet den Vertrag zwischen allen Komponenten.

### 4.2 Komponentenübersicht

```
 simulierte Geräte                 Bus                 Backend-Dienste
 ┌─────────────┐                                      ┌────────────────┐
 │ temp-living │──telemetry──┐                   ┌───▶│ automation     │
 │ motion-hall │──telemetry──┤   ┌───────────┐   │    │ (Regeln, Edge) │
 │ light-living│◀──command───┼──▶│   MQTT    │───┼───▶│ cloud-sync     │
 └─────────────┘             │   │  Broker   │   │    │ (SQLite)       │
 ┌─────────────┐   HTTP      │   └───────────┘   └───▶│ dashboard      │
 │ plug-kitchen│◀──▶ adapter─┘                        └────────────────┘
 │ (REST only) │   (Protokoll-Übersetzung = Interoperabilität)
 └─────────────┘
```

### 4.3 Entwurfsentscheidungen

**Pub/Sub statt Punkt-zu-Punkt:** MQTT als geeignetstes Smart-Home-Modell (2.3),
entkoppelt Teilnehmer und erlaubt das Hinzufügen weiterer ohne Anpassung
bestehender Komponenten.

**Edge/Cloud-Trennung:** lokale, latenzkritische Logik (Regel-Engine) läuft am
Edge und bleibt ohne Cloud funktionsfähig; die zentrale Aggregation übernimmt ein
separater Dienst (hier SQLite als Cloud-Stellvertreter).

**Verzicht auf natives Matter/Thread:** Da physische Geräte nicht im Scope sind
und eine vollständige Matter-Integration den Rahmen eines Prototyps sprengt,
dient MQTT als repräsentatives Pub/Sub-Rückgrat. Die Kernidee von Matter –
herstellerübergreifende Interoperabilität – wird stattdessen durch das
**Adapter-Muster** abgebildet: Ein protokollfremdes Gerät wird auf das
einheitliche Nachrichtenformat übersetzt und ist danach für alle Komponenten
ununterscheidbar von nativen Geräten.

## 5. Demonstrator

### 5.1 Komponenten

| Komponente        | Datei                      | Rolle                           |
|-------------------|----------------------------|---------------------------------|
| Temperatursensor  | `devices/temp_sensor.py`   | publiziert Telemetrie           |
| Bewegungssensor   | `devices/motion_sensor.py` | publiziert Ereignisse           |
| Smart-Lampe       | `devices/smart_light.py`   | Aktor, abonniert Kommandos      |
| REST-Fremdgerät   | `rest_device/service.py`   | HTTP-only Gerät (Heterogenität) |
| Adapter           | `rest_device/adapter.py`   | Brücke REST ↔ MQTT              |
| Automation-Engine | `services/automation.py`   | Regeln am Edge                  |
| Cloud-Sync        | `services/cloud_sync.py`   | Aggregation/Historie (SQLite)   |
| Dashboard         | `services/dashboard.py`    | Live-Status über HTTP           |

Jede Komponente ist ein eigener Prozess/Container (Docker Compose) – Verteilung
auch betrieblich umgesetzt.

### 5.2 Schnittstellen

Einheitlicher JSON-Envelope über vier Topic-Klassen, z. B.
`home/telemetry/{id}/{metric}` (Gerät → Bus) und `home/command/{id}`
(Bus → Aktor); Zustand und Verfügbarkeit (Last-Will) werden *retained*
veröffentlicht. Vollständige Definition: `common/topics.py`.

### 5.3 Simulation der Geräte

Die Sensoren erzeugen ihre Werte rein rechnerisch über ein deterministisches
Modell: Die Temperatur folgt einer Tageskurve (Tiefpunkt nachts, Hochpunkt am
Nachmittag) mit leichter Trägheit, sodass aufeinanderfolgende Werte glatt
verlaufen; Bewegung folgt einem tageszeitabhängigen Belegungsmuster mit kurzen
Aktivitätsphasen; die Luftfeuchte schwankt als Random Walk um einen Mittelwert.
Dadurch verhalten sich die Mockups realistisch, ohne externen Dienst oder KI zur
Laufzeit. (Generative KI kam lediglich bei der Erstellung des Codes zum Einsatz.)
Ausführungsanleitung siehe README; der vollständige Ablauf wurde end-to-end
verifiziert.

### 5.4 Anforderungen ↔ Umsetzung

| Anforderung                     | Umsetzung                                                            |
|---------------------------------|---------------------------------------------------------------------|
| Lose Kopplung                   | Pub/Sub über Topics; gemeinsamer Vertrag in `common/topics.py`      |
| Heterogenität/Interoperabilität | REST-Fremdgerät + Adapter (Protokoll-Übersetzung)                   |
| Skalierbarkeit                  | weiteres Gerät = neuer Dienst + Topic, kein Eingriff in den Kern    |
| Verteilung                      | jede Komponente eigener Prozess/Container                           |
| Fehlertoleranz/Robustheit       | MQTT Last-Will (Offline-Erkennung), Reconnect-Schleife, Verwerfen ungültiger Nachrichten |
| Erweiterbarkeit                 | Sensor-Basisklasse; neue Protokolle via zusätzlichem Adapter        |

## 6. Bewertung, Grenzen und Ausblick

Der Demonstrator setzt alle abgeleiteten Anforderungen an einem lauffähigen,
mehrteiligen Beispiel sichtbar um. Bewusste Grenzen: keine
Authentifizierung/Verschlüsselung am Broker (TLS/ACLs als nächster Schritt);
SQLite nur als Cloud-Platzhalter; nur eine Beispielregel in der Automation.
Naheliegende Erweiterungen sind eine native Matter-/Thread-Anbindung, ein zweiter
Geräte-Adapter, horizontale Skalierung über Broker-Bridging sowie ergänzendes
Monitoring.

## 7. Quellen

- Thread Group: *Thread 1.4 Features White Paper* (2024) – <https://www.threadgroup.org/ThreadSpec>
- Connectivity Standards Alliance: *Matter* – <https://csa-iot.org/all-solutions/matter/>
- OASIS: *MQTT Specification* – <https://mqtt.org/>
- Eclipse Foundation: *Mosquitto* – <https://mosquitto.org/>
- Home Assistant – <https://www.home-assistant.io/> · openHAB – <https://www.openhab.org/>
