# Bürgerabstimmung

Digitale Bürgerbeteiligung mit dem Personalausweis. Die gesamte Anwendung ist
**eine einzige Datei**: `index.php`. Kein Framework, kein Composer, keine
externen Dienste, keine Aufrufe zu Dritten.

Konzept, Fachlogik und die ehrlichen Grenzen stehen im
[Whitepaper](whitepaper.md).

> **Testbetrieb:** Im Auslieferungszustand läuft die Anwendung im Testmodus und
> zeigt auf jeder Seite ein Banner „Testbetrieb – keine offizielle Seite der
> Bundesregierung oder einer Behörde.“ Der Testmodus ist keine Variable im
> Quelltext, sondern ein Zustand der Installation, und wird über die Oberfläche
> beendet.

---

## Installation

1. `index.php` in das Webverzeichnis legen — Hauptverzeichnis oder Unterordner,
   der Basispfad wird erkannt.
2. **PHP 8.0 oder neuer** mit `pdo_sqlite`, `mbstring` und `sodium`. Alle drei
   gehören zur Standardausstattung jedes Hosters.
3. Seite aufrufen. Beim ersten Aufruf legt die Datei selbst an:
   - `data/` — SQLite-Datenbank, Server-Geheimnis, Signaturschlüssel,
     Protokolle und eine eigene `.htaccess`, die den Ordner sperrt
   - `.htaccess` — saubere Adressen und Schutz interner Dateien
   - `robots.txt`

Mehr ist nicht nötig: **Apache und diese Datei.**

Was in `data/` liegt oder im Betrieb dort entsteht (einiges erst bei Bedarf):

| Datei | Inhalt |
|---|---|
| `buergerabstimmung.sqlite` | die Datenbank (Name aus der Entwicklung, ohne Bedeutung) |
| `secret.key` | Server-Geheimnis für die Stimmen-Marker — **niemals** herausgeben |
| `server_sign.key` | Signaturschlüssel für die profil.yaml |
| `authorized_keys.yaml` | freigegebene öffentliche Ausweis-Schlüssel |
| `config.yaml` | Einstellungen aus der Einrichtung (0600) |
| `setup.token` | Einrichtungsschlüssel, nach dem Umschalten gelöscht |
| `test_keys.list` | im Testbetrieb erzeugte Schlüssel, für das Aufräumen |
| `issued/` | private Schlüssel ausgegebener Demo-Ausweise |
| `app.log` | Sicherheitsprotokoll ohne Personenbezug |
| `php-error.log` | PHP-Fehlerprotokoll, sobald etwas anfällt |

Der ganze Ordner ist über eine eigene `.htaccess` gesperrt und die Anwendung
beantwortet Anfragen darauf nicht — unter nginx ist die Sperre selbst zu setzen
(siehe unten).

Meldet die Seite „Fast geschafft“, fehlen dem Verzeichnis Schreibrechte —
per FTP auf 755 oder 775 setzen und neu laden.

**nginx statt Apache:** Ohne `.htaccess` arbeitet die Anwendung über automatisch
erzeugte `/index.php/…`-Adressen weiter. Für saubere Adressen eine gleichwertige
Regel setzen und **`/data/` serverseitig sperren**:

```nginx
location /data/ { deny all; return 404; }
location / { try_files $uri /index.php$is_args$args; }
```

---

## Vom Testbetrieb in den Echtbetrieb

Beim ersten Start ist der **Testmodus** aktiv. Die Anmeldeseite zeigt dann genau
einen Knopf („Test-Anmeldung starten“), der eine zufällige, als gültig
behandelte Sitzung erzeugt; Ausweis-Aufforderungen entfallen. So lässt sich die
Plattform vorführen, ohne einen Ausweis zu benötigen.

Angemeldet steht oben rechts der Chip **„Testmodus“**. Er führt auf
`/setup` — **Echtbetrieb einrichten**:

1. **Voraussetzungen prüfen** — Datenverzeichnis beschreibbar, HTTPS aktiv,
   `sodium` vorhanden, Zugriffsschutz gesetzt, Anzahl freigegebener
   Ausweis-Schlüssel.
2. **Zugang festlegen** — entweder **eigener eID-Server** (BSI TR-03130) oder
   **eigene Trust-Liste**. Erfasst werden SOAP-Adresse, Client-Zertifikat und
   -Schlüssel, die Aktivierungsadresse der Ausweis-App, die Abgleich-Adresse der
   Freigabeliste und die Startadresse von Nect. Ein Knopf **prüft die
   Verbindung**, bevor irgendetwas verändert wird.
3. **Umschalten** — erst bei gültigen Eingaben und passendem
   **Einrichtungsschlüssel** werden die Einstellungen nach `data/config.yaml`
   (Rechte 0600) geschrieben, alle Testdaten gelöscht und der Echtbetrieb
   dauerhaft aktiviert. Die Verbindungsprüfung ist dabei nur auf dem
   eID-Server-Weg zwingend; auf dem Weg über die eigene Trust-Liste wird kein
   Server kontaktiert.

Der **Einrichtungsschlüssel** steht in `data/setup.token` und ist nur über
Dateizugriff oder `php index.php setup-token` lesbar. Damit kann nur umschalten,
wer den Server betreibt — im Testbetrieb erzeugt sich sonst jeder per Knopfdruck
eine Sitzung. Nach dem Umschalten wird die Datei gelöscht und `/setup` ist zu.

Beim Umschalten werden nur die **im Testmodus erzeugten** Schlüssel aus der
Freigabeliste entfernt; ausgegebene oder abgeglichene Ausweise bleiben erhalten.

Danach zeigt die Anmeldeseite die Ausweis-Apps. **Eine Einschränkung, die man
kennen muss:** Auf dem Weg über die eigene Trust-Liste (`eid_mode = demo`)
erscheint zusätzlich der Knopf „Ausweis auflegen“, sobald in der Sitzung bereits
ein freigegebener Schlüssel liegt — etwa nach dem Öffnen eines
`/claim/<handle>`-Links aus `php index.php issue-card`. Dieser Weg ist für die
Ausgabe von Demo- und Pilot-Ausweisen gedacht; wer ihn nicht will, setzt beim
Umschalten `eid_mode = eid`.

---

## Anmeldung mit dem Ausweis

- **AusweisApp direkt eingebunden (BSI TR-03124).** Der Knopf leitet den Browser
  auf die Aktivierungsadresse des eID-Clients
  (`http://127.0.0.1:24727/eID-Client?tcTokenURL=…`). Die AusweisApp fängt diese
  Adresse ab — am PC wie am Smartphone — und holt das **tcToken** unter
  `/eid/tctoken`. Dieser Teil braucht **keine Konfiguration**.
  Weil die App das Token als eigener HTTP-Client **ohne Browser-Cookie** holt,
  verbindet ein Einmal-Nonce in der tcTokenURL beide Seiten (10 Minuten gültig).
- **Ohne eingerichteten eID-Server** liefert das tcToken bewusst nur eine
  `CommunicationErrorAddress`: Die App bricht sauber ab, der Browser kommt
  zurück, **angemeldet wird niemand**.
- **Wichtig:** Der Rücksprung `/eid/callback` ist in dieser Fassung noch **nicht
  ausgeprägt** — er weist grundsätzlich ab. Über die Ausweis-Apps kommt daher
  aktuell **niemand hinein**, auch mit eingerichtetem eID-Server nicht. Was
  fehlt, ist die Prüfung der signierten Zusicherung des eID-Servers und die
  Entnahme der geprüften Kennung. **Nect Wallet** ist als zweiter Anbieter
  konfigurierbar und endet an derselben Stelle.
- **Nur freigegebene Ausweise.** Anmelden kann sich nur, wessen öffentlicher
  Schlüssel in `data/authorized_keys.yaml` steht **und** wer den passenden
  privaten Schlüssel besitzt (Signatur-Challenge). Die Identität ist der
  öffentliche Schlüssel selbst — kein abgeleitetes Pseudonym.
- **Zeitfenster:** Der Anmeldenachweis gilt nur kurz — 5-Minuten-Fenster,
  höchstens zwei; danach ist erneutes Auflegen nötig. **Im Testbetrieb ist diese
  Prüfung abgeschaltet**, dort greifen nur die Sitzungs-Timeouts (30 Minuten
  Leerlauf, 8 Stunden absolut).

> **Ehrlich zur eID:** In Deutschland gibt es **keine** staatliche Liste aller
> Ausweis-Schlüssel und keine API, die sie liefert. Echte Prüfung läuft über die
> BSI-Zertifikatskette (TR-03110) in einem eID-Server, und ein Ausweis-Chip ist
> nur mit der AusweisApp und PIN lesbar — kein Browser kann das. Die Freigabeliste
> ist der Platzhalter für genau diese Vertrauensprüfung, `sync-keys` der
> Anschlusspunkt für eine eigene Trust-Liste. Kein Griff in ein Behördenregister.

---

## Was die Seite kann

- **Eine Seite.** Thema einbringen und Suche öffnen je ein eigenes Fenster;
  darunter Merkzettel-Chips, die Gruppe „kürzlich abgestimmt (noch änderbar)“
  und die Themenliste. `/` , `/topics` und `/me` führen auf dieselbe Seite.
  In der Kopfzeile steht rechts „Abmelden“, im Testbetrieb zusätzlich der Chip
  „Testmodus“ und bei anstehender Jury-Aufgabe ein Hinweis darauf.
- **Ein Thema pro Tag** je Ausweis; das nächste ab Mitternacht.
- **Geltungsbereich** ohne Freitext. Mit JavaScript in drei Stufen
  (Ebene → Land → Kreis); ohne JavaScript bleibt die einfache Auswahlliste mit
  allen rund 400 Kreisen stehen.
- **Ende der Abstimmung — Datum, Zielwert oder beides.** Zwei Haken: „an einem
  Datum“ und „bei erreichter Stimmenzahl“ (Einheit *X Stimmen* oder
  *% Stimmen*). Beides zusammen ist erlaubt — es endet, was zuerst eintritt.
- **Abstimmen dafür/dagegen**, neutral heißt nicht abstimmen. Die eigene Stimme
  ist 24 Stunden änderbar, danach fest. Der Zustand steht in den Knöpfen:
  beide grau, die gewählte Seite blau solange änderbar, danach blass-blau
  gegen abgedunkelt und nicht mehr bedienbar.
- **Zahlen aktualisieren sich ohne Neuladen** — ein schlanker JSON-Endpunkt,
  alle zwölf Sekunden und beim Zurückkehren auf den Tab. Ohne JavaScript bleibt
  der gerenderte Stand stehen.
- **Themen werden nie gelöscht.** Ein Thema ohne Stimmen kann der Verfasser
  **archivieren**: es verschwindet aus Listen und Suche, bleibt aber unter
  seiner Adresse erhalten. Ab der ersten Stimme entfällt auch das.
- **Gleiche Titel sind erlaubt, aber sichtbar** — das Formular zeigt beim Tippen
  ähnliche Themen, die Themenseite listet sie unter „Ähnliche Themen“.
- **Drei Symbole oben rechts am Thema**, auf einer Höhe mit den Schildern für
  Kategorie und Gebiet:
  - **Lesezeichen** — merken: dieses Thema, seine Kategorie, sein Gebiet
  - **Archivbox** — archivieren; nur für den Verfasser und nur solange noch
    niemand abgestimmt hat
  - **rote Flagge** — melden; nicht für den Verfasser und nicht bei beendeten
    Themen. Läuft bereits eine Meldung, steht die Flagge unterlegt und ohne
    Funktion da.
- **Themen lassen sich nicht bearbeiten.** Was eingebracht ist, bleibt wie es
  ist — eine stille Umdeutung nach bereits abgegebenen Stimmen ist damit
  ausgeschlossen. Wer sich vertan hat, archiviert (solange noch keine Stimme da
  ist) und bringt das Thema neu ein.
- **Ohne JavaScript** funktionieren Anmeldung, Themen, Abstimmen, Melden und
  Jury vollständig. JavaScript ergänzt Countdown, NFC-Auslösung, die stufige
  Bereichsauswahl, die Live-Zahlen und den Duplikat-Hinweis — und ist für die
  Selbstauskunft `profil.yaml` derzeit **erforderlich**.
- **Melden nur bei Gesetzesverstoß.** Ein Suchfeld führt zum eingebauten
  Gesetzesregister, der gewählte Paragraph wird **1:1 zitiert** und der Jury
  wortgleich vorgelegt.
- **Bürger-Jury:** 1 % der Nutzerschaft wird per Los gezogen (mindestens 5),
  Quorum 0,5 % (mindestens 3), 24 Stunden Frist, 3 Tage Karenz. Melder, Verfasser
  und laufende Jurorinnen sind ausgeschlossen.
- **Deutsch und Englisch**, gewählt beim Sitzungsbeginn, nirgends gespeichert.
- **Hell und dunkel** folgen der Systemeinstellung.

---

## Adressen

| Pfad | Zweck |
|---|---|
| `/start` | Sprachwahl beim Sitzungsbeginn (erster Aufruf landet hier) |
| `/auth` | Anmeldung mit dem Ausweis |
| `/` , `/topics` , `/me` | die Hauptseite |
| `/topic/{id}` | ein Thema |
| `/topic/{id}/archive` | archivieren (nur Verfasser, nur ohne Stimmen) |
| `/vote` , `/favorite` | Stimme, Merkzettel (POST) |
| `/report/{id}` , `/report` | melden |
| `/jury` , `/jury/vote` | Jury-Aufgabe |
| `/setup` , `/setup/check` , `/setup/finish` | Echtbetrieb einrichten (nur im Testbetrieb) |
| `/eid/start` , `/eid/tctoken` , `/eid/callback` | Ausweis-App |
| `/claim/{handle}` | Demo-Ausweis in die Sitzung laden |
| `/tap` , `/logout` , `/lang` | anmelden, abmelden, Sprache (POST) |
| `/profil.yaml` | verschlüsselte Selbstauskunft |
| `/server.pub` | öffentlicher Prüfschlüssel für die Selbstauskunft |
| `/api/topics` , `/api/similar` | JSON für Live-Zahlen und Duplikat-Hinweis |
| `/imprint` , `/privacy` | Impressum, Datenschutz |
| `/a/app.css` , `/a/app.js` , `/a/icon.svg` | Oberfläche |
| `/favicon.ico` , `/favicon.png` , `/apple-touch-icon.png` | Icons |
| `/robots.txt` | verbietet die Indexierung |
| `/data/…` | wird immer mit 404 beantwortet |

---

## Kommandozeile

```bash
php index.php selftest        # 122 automatisierte Prüfungen der Fachregeln
php index.php cron            # Wartungslauf (sonst beiläufig bei Seitenaufrufen)
php index.php seed 400        # Demo-Stimmen, anonym wie im Echtbetrieb
php index.php jurysim         # Demo-Jury stimmt in laufenden Prüfungen ab
php index.php issue-card 3    # autorisierte Demo-Ausweise samt Abhol-Verweis
php index.php sync-keys       # Freigabeliste aus der konfigurierten Trust-Liste
php index.php setup-token     # Einrichtungsschlüssel für die Umschaltung
php index.php config          # wirksame Einstellungen aus data/config.yaml

php -S 127.0.0.1:8080 index.php   # lokale Vorführung ohne Webserver
```

Für den Wartungslauf genügt eine Cron-Zeile; ohne sie läuft er beiläufig mit:

```cron
*/5 * * * * php /var/www/html/index.php cron >/dev/null 2>&1
```

---

## Sicherheit

- Ausschließlich Prepared Statements, durchgängiges Output-Escaping
- CSP `default-src 'none'` ohne `unsafe-inline`; CSS, JavaScript und Icons
  liefert die Datei selbst als eigene Routen aus. Keine Inhalte von Dritten,
  keine Tracker, keine CDN
- Einmal-Token für **jede** POST-Anfrage — deckt CSRF ab und verhindert Replay
- Jede Änderung läuft über einen **versiegelten, zeitgebundenen Umschlag**: die
  Karte versiegelt die Aktion, der Server öffnet mit dem öffentlichen Schlüssel
- **Stimmen ohne Ausweis-Bezug gespeichert** — nur ein HMAC aus Thema und
  öffentlichem Schlüssel mit Server-Geheimnis. Ohne dieses Geheimnis lässt sich
  nicht rückschließen, welcher Ausweis wie gestimmt hat; Doppelstimmen bleiben
  ausgeschlossen
- **profil.yaml** wird bei jedem Seitenaufruf frisch angefordert, an den
  öffentlichen Ausweis-Schlüssel verschlüsselt und servergegengezeichnet
  (Prüfschlüssel unter `/server.pub`); „Abmelden“ löscht sie
- Sitzungen: HttpOnly, SameSite, ID-Rotation, Idle- und Absolut-Timeout
- Kernregeln zusätzlich als DB-Constraints: 1 Thema/Tag, 1 Stimme/Thema,
  1 offene Meldung/Thema, 1 Jury-Sitz/Meldung
- Jury-Losverfahren mit CSPRNG, Ratenbegrenzung ohne Klar-IP-Speicherung
- Keine Klaridentitäten, keine Passwörter, keine externen Abhängigkeiten

---

## Vor einem echten Betrieb

- **Die Datenschutzerklärung sagt derzeit drei Dinge, die nicht stimmen.**
  `/privacy` behauptet (a) es werde „ausschließlich ein Pseudonym (Hash mit
  serverseitigem Geheimnis)“ gespeichert — tatsächlich ist die Kennung der
  öffentliche Ausweis-Schlüssel selbst; (b) es werde „nichts im Browser
  gespeichert“ — die `profil.yaml` liegt im `sessionStorage`; (c) das Konto
  lasse sich „jederzeit in ‚Meine Übersicht‘ löschen“ — diese Seite und dieser
  Knopf existieren nicht. **Das ist vor jedem öffentlichen Betrieb zu
  korrigieren**, es sind Falschangaben gegenüber Nutzern.
- **Impressum füllen.** `/imprint` enthält nur einen Platzhalter (`*`) und ist
  mit den Pflichtangaben des Betreibers zu ersetzen.
- **Selbstauskunft ohne JavaScript ermöglichen** und einen Weg zur
  **Konto-Löschung in der Oberfläche** ergänzen — die Routine `account_delete`
  existiert, hat aber keine Schaltfläche. Ohne beides ist die DSGVO nicht
  erfüllt.
- **Den Rücksprung `/eid/callback` ausprägen** — ohne ihn führt kein
  Ausweis-App-Weg zu einer Anmeldung.
- **`data/` außerhalb des Webroots legen.** Datenbank und Geheimnisse liegen
  derzeit im Webroot und sind nur durch `.htaccess` und eine 404-Regel der
  Anwendung geschützt.
- **Gesetzestexte** wortgleich gegen gesetze-im-internet.de abgleichen.
- **Gebietsverzeichnis** (16 Länder, rund 400 Kreise) gegen das amtliche
  Verzeichnis (ARS/Destatis) abgleichen.
- **Externes Sicherheits-Audit und Pentest**, Datenschutz-Folgenabschätzung,
  Barrierefreiheit nach BITV.
- Die Seite ist **kein Wahlsystem** und darf nicht als solches bezeichnet
  werden. Sie hat keine Rechtsverbindlichkeit.
