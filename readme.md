# Postgres‑Provisionierung für Dev, Test und Prod

Diese Dokumentation beschreibt den internen GitHub‑Actions‑Workflow zur Provisionierung von Applikationsdatenbanken auf drei PostgreSQL‑Umgebungen (dev, test, prod). Ziel ist eine wiederholbare, sichere und idempotente Bereitstellung von:
- einem Applikationsbenutzer (Login‑Rolle),
- einer dedizierten Datenbank,
- grundlegenden Privilegien (CONNECT, TEMP, CREATE),
- einer konsolidierten Übersicht über die bereitgestellten Ressourcen.

Während der Ausführung wird die notwendige Firewall‑Freigabe für den CI‑Runner temporär geöffnet und anschließend automatisch entfernt.

---

## Kurzüberblick

- Auslösung: Manuell über GitHub Actions (workflow_dispatch)
- Umgebungen: dev, test, prod (Matrix‑Strategie)
- Namenskonventionen:
    - Benutzer: {app_name}_user
    - Datenbank: {app_name}_db
    - Schema: public
- Sicherheit:
    - SSL erzwungen (sslmode=require)
    - Passwortmaskierung in Logs
    - Temporäre, IP‑gebundene Firewall‑Regel
- Robustheit:
    - Wiederholte Verbindungsversuche bei DB‑Operationen

---

## Funktionsumfang

- Anlage oder Aktualisierung des Applikations‑Logins pro Umgebung
- Erstellung einer dedizierten UTF‑8‑Datenbank (falls nicht vorhanden)
- Vergabe der Privilegien CONNECT, TEMP, CREATE auf die Datenbank an den Applikationsnutzer
- Temporäres Öffnen von TCP/5432 für die Runner‑IP in der zuständigen Firewall
- Zusammenführung einer environmentspezifischen Ergebniszeile zu einer Gesamtübersicht (Actions Summary)

---

## Voraussetzungen

- Je Umgebung (dev, test, prod) ein erreichbarer PostgreSQL‑Server mit SSL‑Unterstützung
- Admin‑Konto mit Rechten für:
    - CREATE ROLE / ALTER ROLE (Passwörter setzen/ändern)
    - CREATE DATABASE
    - GRANT auf Datenbanken
- Zugriff auf die jeweilige Netzwerk‑/Firewall‑Konfiguration
- GitHub Actions aktiviert, mit Berechtigung zum Setzen von Secrets und Variablen

---

## Konfiguration

Secrets (Actions > Secrets and variables > Actions > Secrets):
- CIVO_API_KEY: API‑Token für den Cloud‑Firewall‑Zugriff
- DEV_SERVER_ADMIN_PASSWORD: Admin‑Passwort für dev
- TEST_SERVER_ADMIN_PASSWORD: Admin‑Passwort für test
- PROD_SERVER_ADMIN_PASSWORD: Admin‑Passwort für prod

Variablen (Actions > Secrets and variables > Actions > Variables):
- DEV_SERVER_HOST: Host/IP des PostgreSQL‑Servers in dev
- TEST_SERVER_HOST: Host/IP in test
- PROD_SERVER_HOST: Host/IP in prod
- ADMIN_USER: Admin‑Benutzername (z. B. postgres)
- DEV_FIREWALL_NAME: Firewall‑Bezeichner für dev
- TEST_FIREWALL_NAME: Firewall‑Bezeichner für test
- PROD_FIREWALL_NAME: Firewall‑Bezeichner für prod

Hinweise:
- Standardport ist 5432. Bei Abweichungen muss die Workflow‑Konfiguration und die Firewall‑Freigabe angepasst werden.
- Secrets werden ausschließlich zur Laufzeit verwendet und in Logs maskiert.

---

## Verwendung (Schritt für Schritt)

1) Vorbereiten
- Prüfen, ob alle oben genannten Secrets und Variablen vorhanden und korrekt sind.
- Erreichbarkeit der DB‑Server (inkl. SSL) sicherstellen.
- Firewall‑Bezeichner je Umgebung verifizieren.

2) Workflow starten
- In GitHub auf Actions gehen, den Provision‑Workflow öffnen und „Run workflow“ auswählen.

3) Eingaben beim Start
- app_name (Pflicht): Basisname der Anwendung, z. B. shop
    - Daraus werden abgeleitet: Benutzer shop_user, Datenbank shop_db
- dev_password (Pflicht): Passwort für den Applikationsnutzer in dev
- test_password (Pflicht): Passwort für den Applikationsnutzer in test
- prod_password (Pflicht): Passwort für den Applikationsnutzer in prod

4) Ablauf während der Ausführung
- Öffentliche IPv4 des Runners wird ermittelt.
- Temporäre Firewall‑Regel für TCP/5432 wird für diese IP gesetzt.
- psql‑Client wird installiert.
- Pro Umgebung:
    - Anlegen bzw. Aktualisieren der Rolle {app_name}_user (Passwort wird idempotent gesetzt).
    - Erstellen der Datenbank {app_name}_db mit UTF‑8 (falls noch nicht vorhanden).
    - GRANT CONNECT, TEMP, CREATE auf die DB an {app_name}_user.
- Entfernen der temporären Firewall‑Regel.
- Erzeugung einer kompakten Übersicht im GitHub Actions Summary.

---

## Ergebnisse und Nachweise

- GitHub Actions Summary enthält eine Tabelle mit:
    - Umgebung, Host:Port, Benutzer, Datenbank, Schema (public)
- Pro Umgebung wird eine Kurzdatei generiert, die in die Gesamtsummary einfließt.
- Die Logs dokumentieren sämtliche Schritte, inkl. Maskierung sensibler Informationen.

---

## Idempotenz und Wiederholbarkeit

- Rollenpasswörter werden bei jedem Lauf explizit gesetzt (bewusst idempotent).
- DB‑Erstellung: Ist die DB bereits vorhanden, wird der Schritt übersprungen/endet ohne kritischen Einfluss; Rechtevergabe erfolgt dennoch.
- GRANT‑Anweisungen sind wiederholbar.

Empfehlungen:
- Für Passwortrotation den Workflow erneut starten und neue Passwörter angeben.
- Änderungen an Hosts/Ports oder Firewall‑Bezeichnern konsequent in Variablen pflegen.

---

## Betrieb und Änderungen

- Passwortrotation:
    - Workflow erneut starten, neue Passwörter übergeben.
- Portwechsel:
    - Port in der Workflow‑Konfiguration und in der Firewall‑Freigabe anpassen.
- Erweiterung um weitere Umgebungen (z. B. staging):
    - Eine weitere Matrix‑Definition hinzufügen (Host, Admin‑Secret, Firewall‑Bezeichner, env_name).
    - Zugehörige Secrets und Variablen ergänzen.
- Regelmäßige Pflege:
    - API‑Keys und Admin‑Zugangsdaten periodisch erneuern.
    - DB‑Versionen, SSL‑Vorgaben und Unternehmensrichtlinien im Blick behalten.

---

## Sicherheitsaspekte

- SSL‑Verbindungen erzwungen (sslmode=require).
- Passwörter werden unmittelbar nach Einlesen in Logs maskiert (inkl. getrimmter Varianten).
- Firewall‑Freigaben sind strikt auf die Runner‑IP (/32) begrenzt und werden zuverlässig wieder entfernt (auch bei Fehlerpfaden).
- Admin‑Zugänge ausschließlich über Secrets bereitstellen; keine Klartext‑Credentials in Variablen oder Dateien.

---

## Troubleshooting

Verbindung schlägt fehl:
- Hosts/Ports/Admin‑User in den Variablen prüfen.
- Erreichbarkeit (DNS, Routing, Security Groups/Firewall) verifizieren.
- SSL‑Anforderungen sicherstellen; Server muss verschlüsselte Verbindungen akzeptieren.

Firewall‑Probleme:
- Korrekte Firewall‑Bezeichner je Umgebung gesetzt?
- Wurde die temporäre Regel erstellt (Logs prüfen)?
- Stimmt die regionale Zuordnung der Firewall mit der Zielumgebung überein?

Berechtigungsfehler:
- Verfügt der Admin‑Benutzer über CREATE ROLE, ALTER ROLE, CREATE DATABASE, GRANT?
- Wenn CREATE DATABASE nicht erlaubt ist: DB vorab manuell anlegen und den Workflow die Rechte vergeben lassen.

Instabilität/Timeouts:
- Es existiert ein Retry‑Mechanismus (mehrere Versuche mit Wartezeit). Netzwerkstabilität, Last und Latenz prüfen.

Leere Zusammenfassung:
- Prüfen, ob pro Umgebung die Ergebnisdatei erzeugt wurde und der Summary‑Schritt durchlief.

---

## FAQ

- Welche Rechte erhält der Applikationsnutzer?
    - CONNECT, TEMP und CREATE auf die dedizierte Datenbank (weitere Objektrechte – z. B. auf Tabellen/Schemata – werden separat verwaltet).
- Welche Kodierung nutzt die Datenbank?
    - UTF‑8. Anpassungen sind möglich, sollten aber zentral vereinbart werden.
- Was passiert, wenn eine Umgebung nicht erreichbar ist?
    - Nach mehreren Verbindungsversuchen schlägt der entsprechende Teil fehl. Nach Stabilisierung erneut starten.

---

## Governance und Änderungsmanagement

- Änderungen an Secrets/Variablen dokumentieren (Wer/Was/Wann/Warum).
- Review von Workflow‑Änderungen vor dem Merge durchführen.
- Bei größeren Änderungen (z. B. Rechte‑Modell, zusätzliche Schemas) vorab Abstimmung mit DB‑ und Security‑Verantwortlichen.

---

## Glossar

- Applikationsnutzer: Login‑Rolle, über die die Anwendung mit der DB verbindet.
- Dedizierte Datenbank: Pro Anwendung und Umgebung eine eigene DB zur Isolation.
- Matrix‑Job: Mehrfachausführung des Workflows für definierte Umgebungen.
- Summary: Automatisch erzeugte Ergebnisübersicht im GitHub Actions‑Lauf.