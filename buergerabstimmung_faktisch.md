# Bürgerabstimmung

Digitale Bürgerbeteiligung mit dem Personalausweis. Die Anwendung besteht aus einer Datei: `index.php`. Sie nutzt kein Framework, keinen Composer, keine externen Dienste und keine Aufrufe zu Dritten.

Konzept, Fachlogik und Grenzen stehen im [Whitepaper](whitepaper.md).

> **Testbetrieb:** Im Auslieferungszustand läuft die Anwendung im Testmodus und zeigt auf jeder Seite das Banner „Testbetrieb – keine offizielle Seite der Bundesregierung oder einer Behörde.“ Der Testmodus ist ein Installationszustand und wird über die Oberfläche beendet.

---

## Installation

1. `index.php` in das Webverzeichnis legen. Hauptverzeichnis und Unterordner werden unterstützt, der Basispfad wird erkannt.
2. PHP 8.0 oder neuer mit `pdo_sqlite`, `mbstring` und `sodium` bereitstellen.
3. Seite aufrufen. Beim ersten Aufruf legt die Anwendung folgende Dateien und Verzeichnisse an:
   - `data/` für SQLite-Datenbank, Server-Geheimnis, Signaturschlüssel, Protokolle und eine eigene `.htaccess`, die den Ordner sperrt
   - `.htaccess` für saubere Adressen und Schutz interner Dateien
   - `robots.txt`

Benötigt werden Apache, PHP und `index.php`.

In `data/` liegen oder entstehen im Betrieb folgende Dateien und Verzeichnisse:

| Datei | Inhalt |
|---|---|
| `buergerabstimmung.sqlite` | Datenbank |
| `secret.key` | Server-Geheimnis für Stimmen-Marker. Nicht herausgeben. |
| `server_sign.key` | Signaturschlüssel für `profil.yaml` |
| `authorized_keys.yaml` | freigegebene öffentliche Ausweis-Schlüssel |
| `config.yaml` | Einstellungen aus der Einrichtung mit Rechten `0600` |
| `setup.token` | Einrichtungsschlüssel, nach dem Umschalten gelöscht |
| `test_keys.list` | im Testbetrieb erzeugte Schlüssel für das Aufräumen |
| `issued/` | private Schlüssel ausgegebener Demo-Ausweise |
| `app.log` | Sicherheitsprotokoll ohne Personenbezug |
| `php-error.log` | PHP-Fehlerprotokoll, sobald Einträge anfallen |

Der Ordner `data/` ist über eine eigene `.htaccess` gesperrt. Die Anwendung beantwortet Anfragen auf `/data/...` mit 404. Unter nginx muss die Sperre serverseitig gesetzt werden.

Meldet die Seite „Fast geschafft“, fehlen Schreibrechte für das Verzeichnis. Das Verzeichnis per FTP auf 755 oder 775 setzen und neu laden.

**nginx statt Apache:** Ohne `.htaccess` arbeitet die Anwendung über automatisch erzeugte `/index.php/...`-Adressen. Für saubere Adressen eine gleichwertige Regel setzen und `/data/` serverseitig sperren:

```nginx
location /data/ { deny all; return 404; }
location / { try_files $uri /index.php$is_args$args; }
```

---

## Vom Testbetrieb in den Echtbetrieb

Beim ersten Start ist der Testmodus aktiv. Die Anmeldeseite zeigt einen Knopf „Test-Anmeldung starten“. Dieser erzeugt eine zufällige, als gültig behandelte Sitzung. Ausweis-Aufforderungen entfallen.

Angemeldet steht oben rechts der Chip „Testmodus“. Er führt auf `/setup` zur Einrichtung des Echtbetriebs:

1. **Voraussetzungen prüfen**
   - Datenverzeichnis beschreibbar
   - HTTPS aktiv
   - `sodium` vorhanden
   - Zugriffsschutz gesetzt
   - Anzahl freigegebener Ausweis-Schlüssel vorhanden
2. **eID-Server konfigurieren**
   - SOAP-Adresse
   - Client-Zertifikat
   - Client-Schlüssel
   - Aktivierungsadresse der Ausweis-App
   - Abgleich-Adresse der Freigabeliste
   - Startadresse von Nect
   - Verbindungsprüfung vor Änderung der Konfiguration
3. **Umschalten**
   - Einstellungen werden bei gültigen Eingaben und passendem Einrichtungsschlüssel nach `data/config.yaml` geschrieben.
   - Die Datei erhält Rechte `0600`.
   - Testdaten werden gelöscht.
   - Der Echtbetrieb wird dauerhaft aktiviert.

Der Einrichtungsschlüssel steht in `data/setup.token` und ist nur über Dateizugriff oder `php index.php setup-token` lesbar. Nach dem Umschalten wird die Datei gelöscht und `/setup` ist geschlossen.

Beim Umschalten werden nur die im Testmodus erzeugten Schlüssel aus der Freigabeliste entfernt. Ausgegebene oder abgeglichene Ausweise bleiben erhalten.

Nach dem Umschalten zeigt die Anmeldeseite die Ausweis-Apps.

---

## Anmeldung mit dem Ausweis

- **AusweisApp direkt eingebunden nach BSI TR-03124.** Der Knopf leitet den Browser auf die Aktivierungsadresse des eID-Clients: `http://127.0.0.1:24727/eID-Client?tcTokenURL=...`. Die AusweisApp ruft das `tcToken` unter `/eid/tctoken` ab. Ein Einmal-Nonce in der `tcTokenURL` verbindet Browser-Sitzung und Token-Abruf. Der Nonce ist 10 Minuten gültig.
- **eID-Server erforderlich.** Ohne eingerichteten eID-Server liefert `/eid/tctoken` nur eine `CommunicationErrorAddress`. Die App bricht ab, der Browser kehrt zurück, und es erfolgt keine Anmeldung.
- **Status des Rücksprungs.** `/eid/callback` ist in dieser Fassung nicht ausgeprägt und weist Anmeldungen grundsätzlich ab. Über die Ausweis-Apps kann sich daher aktuell niemand anmelden. Erforderlich sind die Prüfung der signierten Zusicherung des eID-Servers und die Entnahme der geprüften Kennung. Nect Wallet ist als zweiter Anbieter konfigurierbar und endet an derselben Rücksprungstelle.
- **Freigegebene Ausweise.** Anmelden kann sich nur, wessen öffentlicher Schlüssel in `data/authorized_keys.yaml` steht und wer den passenden privaten Schlüssel besitzt. Die Prüfung erfolgt über eine Signatur-Challenge. Die Identität ist der öffentliche Schlüssel selbst.
- **Zeitfenster.** Der Anmeldenachweis gilt kurzzeitig: 5-Minuten-Fenster, höchstens zwei Fenster. Danach ist eine erneute Ausweisprüfung nötig. Im Testbetrieb ist diese Prüfung abgeschaltet. Dort gelten die Sitzungs-Timeouts: 30 Minuten Leerlauf und 8 Stunden absolut.

In Deutschland gibt es keine staatliche Liste aller Ausweis-Schlüssel und keine API, die eine solche Liste bereitstellt. Die echte Prüfung läuft über die BSI-Zertifikatskette nach TR-03110 in einem eID-Server. Ein Ausweis-Chip ist nur mit AusweisApp und PIN lesbar. Die Freigabeliste ist ein lokaler Vertrauensanker, `sync-keys` ist der Anschlusspunkt für eine eigene Trust-Liste.

---

## Funktionen

- **Eine Hauptseite.** Thema einbringen und Suche öffnen jeweils ein eigenes Fenster. Darunter stehen Merkzettel-Chips, die Gruppe „kürzlich abgestimmt (noch änderbar)“ und die Themenliste. `/`, `/topics` und `/me` führen auf dieselbe Seite. In der Kopfzeile steht rechts „Abmelden“. Im Testbetrieb steht zusätzlich der Chip „Testmodus“. Bei anstehender Jury-Aufgabe erscheint ein Hinweis.
- **Ein Thema pro Tag** je Ausweis. Das nächste Thema ist ab Mitternacht möglich.
- **Geltungsbereich ohne Freitext.** Mit JavaScript erfolgt die Auswahl in drei Stufen: Ebene, Land, Kreis. Ohne JavaScript bleibt eine einfache Auswahlliste mit allen rund 400 Kreisen.
- **Ende der Abstimmung.** Eine Abstimmung kann an einem Datum, bei erreichter Stimmenzahl oder bei beidem enden. Bei beiden Bedingungen endet sie, sobald die erste Bedingung erreicht ist. Zielwerte können als Stimmenzahl oder Prozentwert angegeben werden.
- **Abstimmen dafür oder dagegen.** Neutral bedeutet nicht abstimmen. Die eigene Stimme ist 24 Stunden änderbar und danach fest. Der Zustand wird in den Knöpfen angezeigt: beide grau, gewählte Seite blau solange änderbar, danach blass-blau gegen abgedunkelt und nicht mehr bedienbar.
- **Live-Zahlen.** Zahlen aktualisieren sich über einen JSON-Endpunkt alle zwölf Sekunden und beim Zurückkehren auf den Tab. Ohne JavaScript bleibt der gerenderte Stand stehen.
- **Keine Löschung von Themen.** Themen werden nicht gelöscht. Ein Thema ohne Stimmen kann der Verfasser archivieren. Es verschwindet aus Listen und Suche, bleibt aber unter seiner Adresse erreichbar. Ab der ersten Stimme ist eine Archivierung nicht mehr möglich.
- **Gleiche Titel erlaubt.** Das Formular zeigt beim Tippen ähnliche Themen. Die Themenseite listet ähnliche Themen unter „Ähnliche Themen“.
- **Drei Symbole am Thema.** Oben rechts auf Höhe der Schilder für Kategorie und Gebiet stehen:
  - **Lesezeichen:** Thema, Kategorie und Gebiet merken
  - **Archivbox:** archivieren, nur für den Verfasser und nur ohne Stimmen
  - **Rote Flagge:** melden, nicht für den Verfasser und nicht bei beendeten Themen. Bei laufender Meldung ist die Flagge unterlegt und ohne Funktion.
- **Keine Bearbeitung von Themen.** Eingebrachte Themen bleiben unverändert. Wer sich vertan hat, kann ein Thema ohne Stimmen archivieren und neu einbringen.
- **Ohne JavaScript nutzbar.** Anmeldung, Themen, Abstimmen, Melden und Jury funktionieren ohne JavaScript. JavaScript ergänzt Countdown, NFC-Auslösung, stufige Bereichsauswahl, Live-Zahlen und Duplikat-Hinweis. Für die Selbstauskunft `profil.yaml` ist JavaScript derzeit erforderlich.
- **Melden nur bei Gesetzesverstoß.** Ein Suchfeld führt zum eingebauten Gesetzesregister. Der gewählte Paragraph wird 1:1 zitiert und der Jury wortgleich vorgelegt.
- **Bürger-Jury.** 1 % der Nutzerschaft wird per Los gezogen, mindestens 5 Personen. Das Quorum beträgt 0,5 %, mindestens 3 Personen. Die Frist beträgt 24 Stunden, die Karenz 3 Tage. Melder, Verfasser und laufende Jurorinnen sind ausgeschlossen.
- **Deutsch und Englisch.** Die Sprache wird beim Sitzungsbeginn gewählt und nicht gespeichert.
- **Hell und dunkel.** Die Oberfläche folgt der Systemeinstellung.

---

## Adressen

| Pfad | Zweck |
|---|---|
| `/start` | Sprachwahl beim Sitzungsbeginn, erster Aufruf landet hier |
| `/auth` | Anmeldung mit dem Ausweis |
| `/`, `/topics`, `/me` | Hauptseite |
| `/topic/{id}` | ein Thema |
| `/topic/{id}/archive` | archivieren, nur Verfasser und nur ohne Stimmen |
| `/vote`, `/favorite` | Stimme, Merkzettel per POST |
| `/report/{id}`, `/report` | melden |
| `/jury`, `/jury/vote` | Jury-Aufgabe |
| `/setup`, `/setup/check`, `/setup/finish` | Echtbetrieb einrichten, nur im Testbetrieb |
| `/eid/start`, `/eid/tctoken`, `/eid/callback` | Ausweis-App |
| `/claim/{handle}` | Demo-Ausweis in die Sitzung laden |
| `/tap`, `/logout`, `/lang` | anmelden, abmelden, Sprache per POST |
| `/profil.yaml` | verschlüsselte Selbstauskunft |
| `/server.pub` | öffentlicher Prüfschlüssel für die Selbstauskunft |
| `/api/topics`, `/api/similar` | JSON für Live-Zahlen und Duplikat-Hinweis |
| `/imprint`, `/privacy` | Impressum, Datenschutz |
| `/a/app.css`, `/a/app.js`, `/a/icon.svg` | Oberfläche |
| `/favicon.ico`, `/favicon.png`, `/apple-touch-icon.png` | Icons |
| `/robots.txt` | verbietet die Indexierung |
| `/data/...` | wird immer mit 404 beantwortet |

---

## Kommandozeile

```bash
php index.php selftest        # 122 automatisierte Prüfungen der Fachregeln
php index.php cron            # Wartungslauf, sonst beiläufig bei Seitenaufrufen
php index.php seed 400        # Demo-Stimmen, anonym wie im Echtbetrieb
php index.php jurysim         # Demo-Jury stimmt in laufenden Prüfungen ab
php index.php issue-card 3    # autorisierte Demo-Ausweise samt Abhol-Verweis
php index.php sync-keys       # Freigabeliste aus der konfigurierten Trust-Liste
php index.php setup-token     # Einrichtungsschlüssel für die Umschaltung
php index.php config          # wirksame Einstellungen aus data/config.yaml

php -S 127.0.0.1:8080 index.php   # lokale Vorführung ohne Webserver
```

Für den Wartungslauf genügt eine Cron-Zeile. Ohne Cron läuft er beiläufig mit:

```cron
*/5 * * * * php /var/www/html/index.php cron >/dev/null 2>&1
```

---

## Sicherheit

- Prepared Statements und durchgängiges Output-Escaping
- CSP `default-src 'none'` ohne `unsafe-inline`; CSS, JavaScript und Icons werden über eigene Routen ausgeliefert. Keine Inhalte von Dritten, keine Tracker, keine CDN.
- Einmal-Token für jede POST-Anfrage gegen CSRF und Replay
- Jede Änderung läuft über einen versiegelten, zeitgebundenen Umschlag: Die Karte versiegelt die Aktion, der Server öffnet mit dem öffentlichen Schlüssel.
- Stimmen werden ohne Ausweis-Bezug gespeichert. Gespeichert wird ein HMAC aus Thema und öffentlichem Schlüssel mit Server-Geheimnis. Ohne dieses Geheimnis lässt sich nicht zurückschließen, welcher Ausweis wie gestimmt hat. Doppelstimmen bleiben ausgeschlossen.
- `profil.yaml` wird bei jedem Seitenaufruf frisch angefordert, an den öffentlichen Ausweis-Schlüssel verschlüsselt und servergegengezeichnet. Der Prüfschlüssel steht unter `/server.pub`. „Abmelden“ löscht `profil.yaml`.
- Sitzungen nutzen HttpOnly, SameSite, ID-Rotation, Idle-Timeout und Absolut-Timeout.
- Kernregeln sind zusätzlich als DB-Constraints umgesetzt: 1 Thema pro Tag, 1 Stimme pro Thema, 1 offene Meldung pro Thema, 1 Jury-Sitz pro Meldung.
- Jury-Losverfahren mit CSPRNG
- Ratenbegrenzung ohne Klar-IP-Speicherung
- Keine Klaridentitäten, keine Passwörter, keine externen Abhängigkeiten

---

## Vor einem echten Betrieb

- **Datenschutzerklärung korrigieren.** `/privacy` enthält derzeit falsche Angaben:
  - Es wird nicht ausschließlich ein Pseudonym gespeichert. Die Kennung ist der öffentliche Ausweis-Schlüssel selbst.
  - Es wird etwas im Browser gespeichert. `profil.yaml` liegt im `sessionStorage`.
  - Eine Kontolöschung über „Meine Übersicht“ ist derzeit nicht verfügbar. Die Seite und der Knopf existieren nicht.
- **Impressum füllen.** `/imprint` enthält nur einen Platzhalter (`*`) und muss mit den Pflichtangaben des Betreibers ersetzt werden.
- **Selbstauskunft ohne JavaScript ermöglichen.**
- **Konto-Löschung in der Oberfläche ergänzen.** Die Routine `account_delete` existiert, hat aber keine Schaltfläche.
- **Rücksprung `/eid/callback` ausprägen.** Ohne diese Umsetzung führt kein Ausweis-App-Weg zu einer Anmeldung.
- **`data/` außerhalb des Webroots legen.** Datenbank und Geheimnisse liegen derzeit im Webroot und sind nur durch `.htaccess` sowie eine 404-Regel der Anwendung geschützt.
- **Gesetzestexte abgleichen.** Die Texte sind wortgleich gegen gesetze-im-internet.de zu prüfen.
- **Gebietsverzeichnis abgleichen.** Die Angaben zu Ländern und Kreisen sind gegen das amtliche Verzeichnis, ARS/Destatis, zu prüfen.
- **Externes Sicherheits-Audit und Pentest durchführen.**
- **Datenschutz-Folgenabschätzung durchführen.**
- **Barrierefreiheit nach BITV prüfen.**
- **Nicht als Wahlsystem bezeichnen.** Die Seite hat keine Rechtsverbindlichkeit.
