# Home Assistant Blueprints

Sammlung eigener Automations-Blueprints für [Home Assistant](https://www.home-assistant.io/).

## Übersicht

| Blueprint | Domain | Kurzbeschreibung | Min. HA |
|---|---|---|---|
| [🗓️⏰ Zeitschaltuhr mit Schaltbestätigung](#zeitschaltuhr-mit-schaltbestätigung) | `automation` | Schaltet Entitäten nach frei wählbaren Zeitquellen und prüft nach, ob der Befehl angekommen ist | 2024.10 |
| [🔌📉 Standby-Abschaltung mit Betriebserkennung](#standby-abschaltung-mit-betriebserkennung) | `automation` | Schaltet einen Verbraucher ab, wenn seine Leistungsaufnahme lange genug im Standby-Bereich liegt | 2024.10 |

## Aufbau

```
blueprints/
├── automation/    # Automations-Blueprints
├── script/        # Script-Blueprints
└── template/      # Template-Blueprints
```

Das Layout spiegelt Home Assistants eigenes `/config/blueprints/`-Verzeichnis.

---

## Import in Home Assistant

> **Wichtig:** Der Import arbeitet **pro URL, nicht pro Repository**. Eine URL = eine
> Datei = ein Blueprint. Ein Repo mit mehreren Blueprints wird also mehrfach importiert,
> jeweils mit der URL der betreffenden Datei. Home Assistant sieht sich das Repo
> drumherum nicht an.

**Per Knopf:** auf das Import-Badge beim jeweiligen Blueprint klicken.

**Manuell:** Einstellungen → Automationen & Szenen → Tab *Blueprints* →
*Blueprint importieren* → Blob-URL der Datei einfügen. Die gewöhnliche
`github.com/.../blob/...`-Adresse genügt, Home Assistant rechnet sie selbst auf
`raw.githubusercontent.com` um.

### Wohin die Datei landet

Home Assistant bildet den Zielordner aus dem **GitHub-Benutzernamen**, nicht aus dem
Ordner im Repo:

```
/config/blueprints/<domain>/Toutzn/<dateiname>.yaml
```

Liegt dort bereits eine Datei gleichen Namens — etwa aus einer früheren manuellen
Installation — bricht der Import ab. Die alte Datei vorher löschen oder umbenennen.

### Updates

Änderung hier committen und pushen, dann in Home Assistant beim Blueprint
*Blueprint neu importieren*. Bestehende Automationen behalten ihre Konfiguration.
Das funktioniert, weil jeder Blueprint seine `source_url` mitführt.

---

## Zeitschaltuhr mit Schaltbestätigung

[![Blueprint importieren](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FToutzn%2Fha-blueprints%2Fblob%2Fmain%2Fblueprints%2Fautomation%2Fzeitschaltuhr_mit_bestaetigung.yaml)

Schaltet beliebige Entitäten nach einer oder mehreren **Zeitquellen** ein und aus – und
**prüft nach, ob es wirklich geklappt hat**. Verlorene Funkbefehle, kurz nicht erreichbare
Aktoren und verpasste Trigger nach einem Neustart fallen damit nicht mehr durchs Raster.

**Datei:** [`blueprints/automation/zeitschaltuhr_mit_bestaetigung.yaml`](blueprints/automation/zeitschaltuhr_mit_bestaetigung.yaml)
**Benötigt:** Home Assistant 2024.10 oder neuer

> **Icon für die Automation:** Der Blueprint-Name kann nur Text und Emoji enthalten – der
> `blueprint:`-Block hat kein Icon-Feld. Ein echtes mdi-Icon lässt sich aber auf der
> erzeugten Automation setzen: Einstellungen → Geräte & Dienste → Tab *Entitäten* → die
> `automation.*` anklicken → Zahnrad → *Symbol* → z. B. `mdi:calendar-clock`.

### Warum

Eine gewöhnliche Zeitschaltuhr-Automation feuert den Schaltbefehl ab und hofft das Beste.
Kommt der Befehl nicht an – Funkaussetzer, Aktor gerade `unavailable`, Home Assistant hat
im Schaltmoment neu gestartet – bleibt die Last unbemerkt an. Dieser Blueprint liest den
Ist-Zustand nach dem Schalten zurück, wiederholt gezielt für die Entitäten, die nicht
reagiert haben, und meldet erst dann Erfolg oder Fehlschlag.

### Zeitquellen: Zustand, nicht Ereignis

Die Zeitquelle ist **jede Entität mit `on`/`off`** – ein `schedule.*`-Helfer, ein
Template-Binärsensor, eine Kalender-Entität, eine Gruppe. Mehrere Quellen lassen sich
per ODER (Fenster addieren) oder UND (Bedingungen kombinieren) verknüpfen.

Warum ein Zustand und kein Trigger? Der Blueprint ist selbstheilend: er fragt zyklisch
und nach jedem Neustart *„was soll jetzt gerade sein?"*. Ein `sun`-Trigger feuert im
Moment des Sonnenuntergangs und ist dann vorbei – startet HA zwanzig Minuten später neu,
weiß niemand mehr, dass die Last an sein sollte. Ein Zustand lässt sich jederzeit
befragen, ein Ereignis nicht.

Deshalb passen diese Entitäten **nicht** direkt als Quelle, weil ihr Zustand nicht
`on`/`off` ist – ein Template-Binärsensor davor löst das (Rezepte unten):

| Entität | Zustand |
|---|---|
| `sun.sun` | `above_horizon` / `below_horizon` |
| `person.*`, `device_tracker.*` | `home` / `not_home` |

### Zwei Ebenen darüber

Über den Zeitquellen liegt eine Hierarchie mit **je eigenem Verhalten**, beide optional:

1. **Master-Schalter** – sinnbildlich der Stecker der Zeitschaltuhr. Hat Vorrang.
2. **Zeitschaltuhr aktiv** – die Betriebsart: regieren die Zeitquellen, oder Handbetrieb?

Damit lässt sich „Stecker gezogen = Last aus" und „Betriebsart aus = Hände weg" gleichzeitig
abbilden – zwei Anforderungen, die sich mit nur einem Schalter widersprechen würden. Beide
leer lassen = die Zeitquellen regieren immer, ohne Übersteuerung.

### Ablauf

```
Trigger (Zeitquelle / Schalter / Neustart / zyklisch)
        ▼
alle Entscheidungs-Eingaben mit brauchbarem Zustand ?
        │                  ──nein──▶  ENDE, keine Meldung
        ja                            (z. B. kurz nach dem HA-Start)
        ▼
alle Master-Schalter on ?  ──nein──▶  Verhalten bei Master aus
        │                             (Zwangs-Aus oder Handbetrieb)
        ja
        ▼
Zeitschaltuhr aktiv on ?   ──nein──▶  Verhalten bei Zeitschaltuhr aus
        │                             (Handbetrieb oder Zwangs-Aus)
        ja
        ▼
Soll = Zeitquellen, ODER/UND-verknüpft (on / off)
        │
        └──────────┬─────────────────  Handbetrieb ▶ ENDE, keine Meldung
                   ▼
           Ist == Soll ?  ──ja──▶  ENDE, keine Meldung
                   │
                  nein
                   ▼
   ┌─────────────────────────────────┐
   │ schalten (nur die Abweichenden) │
   │ warten                          │
   │ Ist-Zustand zurücklesen         │◀── bis alle im Soll
   └─────────────────────────────────┘    oder max. Versuche
                   ▼
        Erfolgs- oder Fehler-Aktion
```

### Eingaben

| Feld | Pflicht | Default | Beschreibung |
|---|---|---|---|
| **Zeitquellen** | ja | – | Eine oder mehrere Entitäten mit `on`/`off`, die den Soll-Zustand vorgeben |
| **Verknüpfung mehrerer Zeitquellen** | nein | ODER | *ODER* = eine `on` genügt · *UND* = alle müssen `on` sein |
| **Master-Schalter** | nein | – | Oberste Ebene, „der Stecker". Alle müssen `on` sein. Hat Vorrang vor der Betriebsart. |
| **Verhalten bei ausgeschaltetem Master** | nein | Zwangs-Aus | *Zwangs-Aus* = Ziele werden ausgeschaltet · *Handbetrieb* = Automatik hält sich raus |
| **Zeitschaltuhr aktiv** | nein | – | Betriebsart-Schalter. Leer = kein Handbetrieb, Zeitquellen regieren immer. |
| **Verhalten bei deaktivierter Zeitschaltuhr** | nein | Handbetrieb | *Handbetrieb* = aktueller Zustand bleibt, du schaltest frei · *Zwangs-Aus* = Ziele werden ausgeschaltet |
| **Zu schaltende Entitäten** | ja | – | `switch`, `light`, `input_boolean`, `fan`, `media_player` |
| **Wartezeit vor der Nachprüfung** | nein | 10 s | Bei Funk-Aktoren großzügiger wählen |
| **Maximale Schaltversuche** | nein | 5 | Wiederholt nur für die Entitäten, die noch nicht reagiert haben |
| **Zyklische Nachprüfung aktiv** | nein | ein | Korrigiert Abweichungen laufend |
| **Intervall der Nachprüfung** | nein | 5 min | `/1`, `/5`, `/15` oder `/30` |
| **Push bei Erfolg an diese Geräte** | nein | – | Geräte mit HA-App, die bei erfolgreichem Schalten einen Push erhalten. Leer = kein Push. |
| **Push bei Fehlschlag an diese Geräte** | nein | – | Dasselbe für den Fehlerfall. Leer = kein Push. |
| **Titel der Push-Benachrichtigung** | nein | `Zeitschaltuhr` | Überschrift der Meldung |
| **Fehlschlag als kritische Benachrichtigung** | nein | aus | Push kommt auch bei stummem Telefon durch |
| **Aktion bei Erfolg** | nein | Logbuch-Eintrag | Zusätzliche Aktion über den Push hinaus |
| **Aktion bei Fehlschlag** | nein | Persistente Benachrichtigung | Zusätzliche Aktion über den Push hinaus |

### Push aufs Handy

Dafür genügt es, unter *Push-Benachrichtigungen* die Geräte auszuwählen – getrennt für
Erfolg und Fehlschlag. Der Blueprint baut den Notify-Aufruf selbst:

```yaml
action: "notify.mobile_app_{{ device_attr(repeat.item, 'name') | slugify }}"
```

Auswählbar sind alle Geräte mit installierter Home-Assistant-App. Leere Liste = kein
Push für diesen Fall; beide Listen leer = der Blueprint arbeitet ganz ohne Push.

> **Der Dienstname hängt am Gerätenamen zur Registrierungszeit.** Die Companion-App legt
> `notify.mobile_app_<slug des Gerätenamens>` an. `device_attr(…, 'name')` liefert genau
> diesen ursprünglichen Namen, eine spätere Umbenennung in HA ändert daran nichts. Wird
> die App aber deinstalliert und neu registriert, kann sich der Dienstname ändern – dann
> das Gerät hier neu auswählen. Falls ein Push ausbleibt, den Dienstnamen unter
> Entwicklerwerkzeuge → Aktionen gegenprüfen.

### Zusätzliche Aktionen

Die beiden Felder *Aktion bei Erfolg* und *Aktion bei Fehlschlag* laufen **zusätzlich**
zum Push und nehmen beliebige Aktionen auf – Logbuch, TTS-Ansage, ein zweiter Schalter,
Telegram, was auch immer. Nutzbare Variablen:

| Variable | Inhalt |
|---|---|
| `soll_zustand` | `on` oder `off` |
| `grund` | `zeitquelle`, `master_aus`, `zeitschaltuhr_aus` oder `unbekannt` |
| `grund_text` | Dasselbe im Klartext, z. B. `Master-Schalter aus` |
| `ziel_liste` | Alle konfigurierten Ziel-Entitäten |
| `restliche` | Entitäten, die den Soll-Zustand **nicht** erreicht haben |
| `versuche` | Anzahl der Schaltversuche |

Die vorgegebenen Aktionen sind Platzhalter: „Aktivität protokollieren" (`logbook.log`)
schreibt nur ins HA-Logbuch, „Anhaltende Benachrichtigung erstellen"
(`persistent_notification.create`) nur in die Glocke der Oberfläche. Beide sind **keine**
Push-Benachrichtigungen – dafür ist die Geräteauswahl oben da. Wer sie nicht braucht,
kann sie löschen.

### Rezepte für Zeitquellen

Alles über Einstellungen → Geräte & Dienste → **Helfer** → *Template* →
*Template-Binärsensor*. Vorher im Template-Editor unter Entwicklerwerkzeuge live testen.

**Sonnenuntergang bis Sonnenaufgang, aber nur zu zivilen Zeiten** – der Klassiker für
Außenbeleuchtung und den Weihnachtsbaum:

```jinja
{% set dunkel = state_attr('sun.sun', 'elevation') < -2 %}
{{ dunkel and today_at('06:00') <= now() < today_at('23:00') }}
```

An von 06:00 bis Sonnenaufgang und von Sonnenuntergang bis 23:00; nach Mitternacht aus.
Die Zeitklammer trennt Morgen und Abend – ohne sie würde der Sensor die ganze Nacht
durchlaufen. `-2` ist der Feinregler: `0` ist der Sonnenuntergang selbst, `-6` das Ende
der bürgerlichen Dämmerung, also merklich später.

**Nur abends:**

```jinja
{% set dunkel = state_attr('sun.sun', 'elevation') < -2 %}
{{ dunkel and today_at('12:00') <= now() < today_at('23:00') }}
```

**Anwesenheit** (als zweite Quelle mit UND zu verknüpfen):

```jinja
{{ is_state('person.torsten', 'home') or is_state('person.partner', 'home') }}
```

Diese Sensoren aktualisieren sich von allein: `elevation` ändert sich laufend, und
Templates mit `now()` rechnet Home Assistant jede Minute neu.

**Ohne Template geht auch:** zwei `schedule.*`-Helfer mit ODER für „vormittags und
abends", oder ein `schedule.*` mit UND zu einem `binary_sensor` für „im Zeitfenster und
nur wenn jemand da ist".

### Beispielkonfiguration: Poolsteuerung

Zwei Helfer, klar getrennte Bedeutung:

| Helfer | Rolle | Blueprint-Feld |
|---|---|---|
| `input_boolean.pool_periode` | Poolsaison – „Zeitschaltuhr eingesteckt" | Master-Schalter, Verhalten **Zwangs-Aus** |
| `input_boolean.pool_zeitschaltplan` | Betriebsart – Zeitplan oder Handbetrieb | Zeitschaltuhr aktiv, Verhalten **Handbetrieb** |
| iPhone | Meldungen | Push bei Erfolg **und** bei Fehlschlag, Titel „Pool", Fehler kritisch |

Daraus ergibt sich:

| Periode | Zeitschaltplan | Verhalten |
|---|---|---|
| an | an | Zeitplan regiert, Schaltbestätigung und Korrektur aktiv |
| an | aus | Handbetrieb – Zustand bleibt, freies Schalten (z. B. Pumpe für die Heizung) |
| aus | egal | Saisonende: Pumpe wird ausgeschaltet, mit Meldung |

### Beispielkonfiguration: Weihnachtsbaum draußen

Kein Handbetrieb, kein Master – nur Zeitquelle und Ziel:

| Feld | Wert |
|---|---|
| Zeitquellen | `binary_sensor.dunkel_und_zivile_zeit` (Rezept oben) |
| Master-Schalter | *leer* |
| Zeitschaltuhr aktiv | *leer* |
| Zu schaltende Entitäten | `switch.tannenbaum_aussen` |
| Push bei Fehlschlag | eigenes Handy |
| Push bei Erfolg | *leer* – zweimal täglich eine Meldung will man nicht |

Die Nachprüfung sorgt dafür, dass die Lichterkette auch nach einem WLAN-Aussetzer
angeht, und meldet, wenn die Steckdose im Garten gar nicht mehr reagiert.

### Trigger

| ID | Trigger | Zweck |
|---|---|---|
| `zeitquelle` | `state` auf die Zeitquellen, `to: "on"` / `to: "off"` | Regulärer Schaltzeitpunkt |
| `schalter` | `state` auf Master-Schalter und Betriebsart, `to: ["on", "off"]` | Sofortige Neubewertung |
| `neustart` | `homeassistant` / `start` | Nach einem Neustart verpasste Schaltzeitpunkte nachholen |
| `sync` | `time_pattern` | Verlorene Funkbefehle einfangen |

`mode: queued`, `max: 10`, `max_exceeded: silent`

### Designentscheidungen

- **Explizites `to:` an allen State-Triggern.** Ohne `to:` feuert ein State-Trigger auch auf
  reine Attributänderungen – ein `schedule`-Entity aktualisiert beim Umschalten auch
  `next_event`. Zusammen mit `mode: single` konnte so der eigentliche `off`-Lauf als
  „already running" verworfen werden.
- **`mode: queued` statt `single`.** Kein Lauf wird mehr stillschweigend verworfen.
- **Entity-Selektor statt `target:` / `device_id`.** Nur mit konkreten Entity-IDs lässt sich
  der Ist-Zustand zurücklesen – mit einer Geräte-ID wäre die Erfolgsprüfung unmöglich.
- **`homeassistant.turn_on` / `turn_off`** statt domänenspezifischer Dienste, damit gemischte
  Ziel-Typen in einer Automation funktionieren.
- **`continue_on_error: true`** am Schaltbefehl, damit ein `unavailable`-Aktor den Lauf nicht
  abbricht und die Fehlermeldung noch zugestellt wird.
- **Abbruch bei `Ist == Soll`** vor dem Schalten – sonst würde die zyklische Nachprüfung
  Erfolgsmeldungen im Minutentakt produzieren.
- **Master vor Betriebsart.** Ist beides aus, gilt das Master-Verhalten. „Stecker gezogen"
  schlägt „Betriebsart".
- **Zeitquelle als Zustand, nicht als Trigger.** Nur so kann der Blueprint nach einem
  Neustart nachholen, was er verpasst hat. Ein eingebauter Sonnen-Modus hätte die
  Zustandsmatrix vervielfacht; delegiert an einen Template-Helfer bleibt der Blueprint
  klein und ist gleichzeitig mächtiger.
- **Unbekannte Zustände blockieren.** Ist eine Zeitquelle oder ein Schalter `unavailable`
  oder `unknown`, tut die Automation nichts (`grund: unbekannt`). Ohne diese Sperre würde
  „weiß ich nicht" als „aus" gelesen – und der `homeassistant.start`-Trigger würde die
  Last kurz nach jedem Neustart abschalten, weil Entitäten dann noch nicht geladen sind.
  Geprüft werden nur die Entscheidungs-Eingaben; ein `unavailable` **Ziel** bleibt ein
  Fehlerfall und wird gemeldet.

### Bekannte Einschränkungen

- **Manuelles Schalten bei aktiver Zeitschaltuhr wird korrigiert.** Wer die Last außerhalb
  des Zeitfensters von Hand einschaltet, wird von der zyklischen Nachprüfung innerhalb des
  eingestellten Intervalls wieder überstimmt. Richtige Reihenfolge also: **erst die
  Betriebsart ausschalten, dann von Hand schalten.**
- Stille kann drei Dinge bedeuten: Automatik nicht zuständig, Zustand stimmte schon, oder
  die Automation lief nicht. Unterscheiden lässt sich das nur über die Traces.
- `unavailable`-Ziele gelten als Abweichung, werden bis zum Versuchslimit angestoßen und
  danach als Fehler gemeldet. Das ist beabsichtigt.
- Nach einem HA-Neustart kann es bis zum nächsten Nachprüf-Intervall dauern, bis
  korrigiert wird – der `homeassistant.start`-Trigger läuft ins Leere, solange die
  Zeitquellen noch `unavailable` sind. Mit dem Default von 5 Minuten unkritisch.
- Die Master-Schalter werden mit Default `[]` als `entity_id` in einem State-Trigger
  verwendet. Eine leere Liste registriert einfach keinen Listener. Sollte eine HA-Version das
  beim Speichern beanstanden, den betreffenden Trigger entfernen – die zyklische Nachprüfung
  fängt Änderungen dann verzögert ab.

---

## Standby-Abschaltung mit Betriebserkennung

[![Blueprint importieren](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FToutzn%2Fha-blueprints%2Fblob%2Fmain%2Fblueprints%2Fautomation%2Fstandby_abschaltung.yaml)

Schaltet einen Verbraucher stromlos, sobald seine Leistungsaufnahme lange genug im
**Standby-Bereich** liegt: Waschmaschine fertig, Trockner fertig, Fernseher aus. Danach
wird **nachgeprüft**, ob der Aktor wirklich aus ist.

**Datei:** [`blueprints/automation/standby_abschaltung.yaml`](blueprints/automation/standby_abschaltung.yaml)
**Benötigt:** Home Assistant 2024.10 oder neuer

> **Ein Blueprint pro Verbraucher.** Schwellen und Nachlaufverhalten unterscheiden sich je
> Gerät, und ein Leistungssensor gehört immer zu genau einer Last.

### Warum

Der naive Ansatz „Leistung unter 10 W, also fertig" schaltet mitten im Waschgang ab. Eine
Waschmaschine zieht in der Heizphase über 2 kW, danach nur noch stoßweise, wenn die Trommel
dreht – zwischen zwei Stößen können Minuten liegen. Das Kriterium ist deshalb nie ein
einzelner Messwert, sondern ein **durchgehaltenes Zeitfenster**:

> Die Leistung liegt seit *20 Minuten* ununterbrochen unter *10 W*.

Steigt sie im Fenster wieder an, beginnt es von vorn. Der Blueprint hält dieses Fenster
selbst, statt sich auf ein `for:` am Trigger zu verlassen – nur so lassen sich einzelne
Spitzen gezielt tolerieren.

### Geräte mit Nachlauf: die Spitzen-Toleranz

Ein Trockner mit Knitterschutz lässt die Trommel nach Programmende alle paar Minuten kurz
anlaufen. Strikt gerechnet wird das Fenster dadurch nie voll und der Trockner nie
abgeschaltet. Zwei Wege führen daran vorbei:

| Strategie | Einstellung | Wann sinnvoll |
|---|---|---|
| **Spitzen-Toleranz** | Schwelle niedrig lassen, Toleranz auf 90–120 s | Nachlaufspitzen sind so hoch wie echter Betrieb, unterscheiden sich aber in der Dauer |
| **Höhere Schwelle** | Schwelle über die Nachlaufspitze legen (z. B. 300 W), Toleranz 0 | Nachlauf zieht deutlich weniger als der Betrieb – Trommelmotor 150 W gegen Programm über 1 kW |

Die Toleranz begrenzt die Dauer **einer einzelnen** Überschreitung. Ein erneuter
Programmstart dauert länger und bricht das Fenster korrekt ab. Länger als die kürzeste
echte Betriebsphase darf der Wert nicht sein, sonst wird ein Start als Nachlauf verkannt.
Ein toleriertes Ereignis **verlängert das Fenster nicht**.

### Betriebserkennung

Ohne Startschwelle (`0`) gilt: Verbraucher eingeschaltet + Leistung niedrig + Fenster voll
→ abschalten. Eine Waschmaschine, die eingeschaltet auf ihre Startzeitvorwahl wartet, wird
damit nach 20 Minuten stromlos gemacht, bevor sie überhaupt losgelegt hat.

Mit einer Startschwelle größer 0 wird erst abgeschaltet, **nachdem** die Leistung
mindestens einmal darüber lag. Faustwert: deutlich über der Standby-Schwelle, etwa die
Hälfte der Heizphasen-Leistung – 50 W passen fast immer.

Die Erkennung lebt im laufenden Automations-Durchlauf, nicht in einem Helfer pro Gerät. Das
kostet nichts an Einrichtung, hat aber eine Konsequenz: siehe *Bekannte Einschränkungen*.

### Eingaben

| Feld | Pflicht | Default | Beschreibung |
|---|---|---|---|
| **Leistungssensor** | ja | – | Momentanleistung in W (`device_class: power`) – **nicht** der kWh-Zähler |
| **Standby-Schwelle** | nein | 10 W | Darunter gilt der Verbraucher als nicht arbeitend |
| **Beobachtungsdauer** | nein | 20 min | Muss länger sein als die längste Pause im laufenden Betrieb |
| **Spitzen-Toleranz** | nein | 0 s | Maximale Dauer einer einzelnen Überschreitung, die als Nachlauf gilt |
| **Startschwelle** | nein | 0 W | `0` = Betriebserkennung aus · größer 0 = erst abschalten, wenn das Gerät gelaufen ist |
| **Ziel-Entitäten** | ja | – | `switch`, `light`, `input_boolean`, `fan`, `media_player` – Entitäten, keine Geräte |
| **Bezeichnung** | nein | `Der Verbraucher` | Für Meldungstexte, als Variable `bezeichnung` nutzbar |
| **Sperr-Entitäten** | nein | – | Solange eine `on` ist, wird nicht abgeschaltet |
| **Mindest-Einschaltdauer** | nein | 0 min | Erst abschalten, wenn der Aktor mindestens so lange an ist |
| **Wartezeit vor der Prüfung** | nein | 10 s | Bei Funk-Aktoren großzügiger wählen |
| **Maximale Schaltversuche** | nein | 5 | Danach gilt der Vorgang als fehlgeschlagen |
| **Zyklische Nachprüfung** | nein | ein | Fängt vergessene Verbraucher nach Neustart oder Reload ein |
| **Intervall der Nachprüfung** | nein | 5 min | `/1`, `/5`, `/15` oder `/30` |
| **Push bei Erfolg** | nein | – | Geräte mit HA-App. Leer = kein Push. |
| **Push bei Fehlschlag** | nein | – | Dasselbe für den Fehlerfall |
| **Titel der Push-Meldung** | nein | `Hinweis` | Überschrift |
| **Text bei Erfolg / Fehlschlag** | nein | „*{{ bezeichnung }} ist fertig und wurde abgeschaltet.*" / Fehlermeldung mit Versuchszahl und Restliste | Freier Text, Templates erlaubt |
| **Fehler-Push als kritischer Alarm** | nein | aus | Nur iOS, kommt auch bei stummem Telefon durch |
| **Aktion bei Erfolg / Fehlschlag** | nein | Logbuch / Glocken-Meldung | Zusätzliche Aktion neben dem Push |

Variablen für eigene Texte und Aktionen: `bezeichnung`, `leistung` (Messwert beim
Abschalten), `versuche`, `ziel_liste`, `restliche`.

### Ablauf

1. **Grundprüfung** – ist überhaupt etwas eingeschaltet, ist keine Sperre aktiv, liefert
   der Sensor eine Zahl? (`unavailable` darf nie als „0 W, also Standby" durchgehen.)
2. **Betriebsbeginn abwarten** – nur mit Startschwelle. Wird stattdessen ausgeschaltet,
   endet der Durchlauf.
3. **Betriebsende abwarten** – warten, bis die Leistung unter die Standby-Schwelle fällt.
4. **Fenster halten** – auf Spitzen warten, jede gegen die Toleranz prüfen, bei
   Überschreitung abbrechen. Der nächste Messwert unter der Schwelle setzt neu auf.
5. **Prüfungen unmittelbar vor dem Schaltbefehl** – noch etwas an, keine Sperre in der
   Zwischenzeit dazugekommen, Mindest-Einschaltdauer erreicht?
6. **Abschalten mit Wiederholung** – nur die Entitäten anstoßen, die nicht `off` sind,
   warten, nachlesen, bis zum Versuchslimit.
7. **Melden** – Push an die passende Geräteliste, dann die frei definierte Zusatzaktion.

### Trigger

| ID | Trigger | Zweck |
|---|---|---|
| `standby` | `numeric_state` unter der Schwelle | Der eigentliche Anlass |
| `ein` | `state` auf die Ziele, `to: on` | Durchlauf beginnt und wartet auf den Betrieb – nötig für die Betriebserkennung |
| `neustart` | `homeassistant` `start` | Nach Neustart oder Reload ist kein Durchlauf mehr aktiv |
| `sync` | `time_pattern` | Fängt ein, was zwischen den Triggern verloren ging |

`mode: single`, `max_exceeded: silent`: Ein Durchlauf begleitet einen kompletten
Gerätezyklus. Läuft schon einer, ist er zuständig, und jeder weitere Trigger wäre nur ein
Duplikat.

### Beispielkonfigurationen

| Gerät | Schwelle | Dauer | Toleranz | Startschwelle | Begründung |
|---|---|---|---|---|---|
| Waschmaschine | 10 W | 20 min | 0 s | 50 W | Trommeldrehen *während* des Programms muss als „läuft noch" gelten |
| Trockner mit Knitterschutz | 15 W | 20 min | 120 s | 50 W | Nachlauf alle paar Minuten, 20–60 s lang – oder Schwelle 300 W und Toleranz 0 |
| Geschirrspüler | 10 W | 30 min | 0 s | 50 W | Lange Trocknungsphase am Ende |
| Fernseher | 20 W | 20 min | 0 s | 0 W | Standby zieht 10–15 W; „läuft nie" ist meist eine zu niedrig gesetzte Schwelle |
| Kaffeevollautomat | 10 W | 15 min | 0 s | 0 W | Heizt in kurzen Intervallen nach |

Die Schwelle findet man am schnellsten im Verlaufsdiagramm des Leistungssensors: einen
kompletten Durchlauf ansehen und einen Wert wählen, der über dem Ruhestrom und unter der
niedrigsten Dauerlast im Betrieb liegt.

### Designentscheidungen

- **Fenster im Skript statt `for:` am Trigger.** Ein `numeric_state`-Trigger mit `for:`
  könnte das Fenster auch halten, aber nicht einzelne Spitzen tolerieren – jede
  Überschreitung würde es zurücksetzen.
- **Kein Helfer pro Gerät.** Die Vorgänger-Automation brauchte einen `input_number` als
  Schleifenzähler. Hier trägt der laufende Durchlauf den Zustand: `wait_for_trigger` und
  `wait_template` statt Polling im Sekundenraster, also auch keine 240 Schleifendurchläufe
  pro Minute.
- **Entity-Selektor statt `device_id`** für die Ziele. Nur mit einer Entity-ID lässt sich
  der Ist-Zustand zurücklesen – mit einer Geräte-ID wäre die Erfolgsprüfung unmöglich.
- **`homeassistant.turn_off`** statt domänenspezifischer Dienste, damit gemischte Ziele
  möglich sind.
- **Sensorwerte immer frisch gelesen.** Skript-Variablen werden einmal bei ihrer
  Definition gerendert; nach einem Wait wäre ein dort gemerkter Messwert veraltet. Der
  Blueprint liest die Leistung deshalb an jeder Entscheidung neu über `states()`.
- **Timeout `1 s` als Untergrenze** für das Fensterende: `timeout: 0` liest Home Assistant
  als „kein Timeout" – der Durchlauf würde ewig auf eine Spitze warten.
- **24-Stunden-Notbremse** an beiden Warteschritten. Wird Sensor oder Aktor dauerhaft
  `unavailable`, endet der Durchlauf von selbst, statt die Automation bis zum nächsten
  HA-Neustart zu blockieren.
- **Zwei getrennte Push-Zweige** statt einer per Template gewählten Empfängerliste: nur so
  lässt sich der Meldungstext als Blueprint-Eingabe mit Templates füllen. Ein `!input`,
  das selbst Template-Text enthält, wird nur gerendert, wenn es direkt im Zielfeld steht.
- **`continue_on_error: true`** am Schaltbefehl und am Notify-Aufruf, damit ein
  `unavailable`-Aktor oder ein totes Handy den Durchlauf nicht abbricht.
- **Sperren zweimal geprüft** – beim Start des Fensters und direkt vor dem Schaltbefehl.
  Eine Sperre, die während der Beobachtung gesetzt wird, greift also noch.

### Traces lesen

Zwei Dinge im Trace sehen nach Fehler aus und sind keiner.

**Der wartende Durchlauf.** Solange der Verbraucher läuft, steht der Durchlauf im Schritt
*Auf einen von 2 Auslösern warten* und zeigt:

```
result: false
state: 44
wanted_state_below: 20
```

`result: false` ist nicht das Ergebnis des Schritts, sondern die Zustandsauswertung beim
Registrieren des Listeners: „44 W ist nicht unter 20 W, hier gibt es nichts zu
überspringen." Der Schritt ist **aktiv** und wartet auf eins von drei Ereignissen –
Leistung unter die Schwelle, Ziel-Entität auf `off`, oder die 24-Stunden-Notbremse.

**Die verworfenen Läufe.** Jeder `sync`-Trigger, der auf einen laufenden Durchlauf
trifft, endet mit:

```
Gestoppt, da nur eine einzige Ausführung zulässig ist (Laufzeit: 0.00 Sekunden)
```

Das ist `mode: single` bei der Arbeit: der wartende Durchlauf hat genau das Ereignis
abonniert, auf das es ankommt, ein zweiter wäre nur ein Duplikat. `max_exceeded: silent`
unterdrückt dabei nur die `WARNING`-Zeile im Log, nicht den Trace-Eintrag.

**Praktischer Nebeneffekt:** Home Assistant speichert standardmäßig nur **5 Traces pro
Automation**. Läuft ein Verbraucher stundenlang, verdrängen die verworfenen Läufe genau
den Trace, der interessant ist. Zwei Wege:

- **Intervall der Nachprüfung auf `/15` stellen.** Bei 20 Minuten Beobachtungsdauer bringt
  eine Nachprüfung jede Minute nichts – sie startet nur häufiger einen Durchlauf, der
  ohnehin wartet.
- **Trace-Puffer aufdrehen.** In der Automation *In YAML bearbeiten* und ergänzen:

  ```yaml
  trace:
    stored_traces: 25
  ```

  Das ist eine Eigenschaft der Automation, nicht des Blueprints, und übersteht auch ein
  *Blueprint neu importieren*.

### Bekannte Einschränkungen

- **Fällt Home Assistant genau in der Nachlaufphase aus** – Programm gelaufen, Leistung
  schon im Standby, Abschaltung noch nicht erfolgt – bleibt der Verbraucher an. Der neue
  Durchlauf nach dem Neustart wartet mit aktiver Betriebserkennung auf einen Betrieb, den
  es nicht mehr gibt. Ohne Startschwelle tritt das nicht auf.
- **Die Beobachtungsdauer ist die Wartezeit nach dem Programmende.** Wer schneller
  abschalten will, braucht ein kürzeres Fenster – und riskiert damit die Abschaltung in
  einer langen Betriebspause. 20 Minuten sind der Kompromiss.
- **Ein Verbraucher pro Automation.** Mehrere Ziel-Entitäten sind erlaubt, teilen sich dann
  aber einen Leistungssensor und ein Fenster (Steckdose plus Steckdosenleiste am selben
  Aktor).
- **`unavailable`-Ziele** gelten als „nicht aus", werden bis zum Versuchslimit angestoßen
  und danach als Fehler gemeldet. Beabsichtigt.
- **Der Notify-Dienstname hängt am Gerätenamen zur Registrierungszeit.** App neu
  registrieren kann ihn ändern – dann das Gerät im Blueprint neu auswählen.
- **Der Trace-Puffer läuft mit verworfenen `sync`-Läufen voll**, solange ein Verbraucher
  arbeitet. Kein Funktionsproblem, aber lästig beim Debuggen – siehe *Traces lesen*.

### Migration von einer eigenen Automation

Die typische Vorgänger-Automation pollt jede Minute per `time_pattern`, zählt in einem
`input_number` Schleifendurchläufe und wartet in 5-Sekunden-Schritten. Umstellung:

| Alt | Neu |
|---|---|
| `time_pattern` jede Minute + `repeat: count: 240` mit `delay: 5s` | Beobachtungsdauer in Minuten |
| `input_number.*_loop_state` als Merker | entfällt – der Durchlauf trägt den Zustand |
| `condition: device` mit `type: is_on` | Ziel-Entität (Entity-ID, nicht Geräte-ID) |
| `type: is_power` mit `below: 10` | Standby-Schwelle |
| `stop: Verbraucher wieder online` | Spitzen-Toleranz (`0` = identisches Verhalten) |
| `notify.mobile_app_...` je Empfänger | Geräteauswahl „Push bei Erfolg" |
| kein Gegencheck nach dem Ausschalten | Erfolgsprüfung mit Wiederholung |

Der `input_number`-Helfer kann nach der Umstellung gelöscht werden. Die alte Automation
vorher deaktivieren, nicht löschen – zum Vergleich der Traces in den ersten Tagen.

---

## Neuen Blueprint hinzufügen

1. Datei nach `blueprints/<domain>/<name>.yaml` legen (`domain` = `automation`, `script`
   oder `template`).
2. Im `blueprint:`-Block eintragen:
   ```yaml
   blueprint:
     name: Sprechender Name
     author: Torsten Wilms
     source_url: https://github.com/Toutzn/ha-blueprints/blob/main/blueprints/<domain>/<name>.yaml
     homeassistant:
       min_version: "2024.10.0"
     domain: <domain>
   ```
   Die `source_url` muss exakt dem Pfad im Repo entsprechen – sie ist der Update-Pfad.
3. Zeile in die Übersichtstabelle oben aufnehmen und einen eigenen Abschnitt anlegen.
4. Import-Badge einfügen:
   ```markdown
   [![Blueprint importieren](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=<URL-encodierte Blob-URL>)
   ```

## Lizenz

MIT – siehe [LICENSE](LICENSE).
