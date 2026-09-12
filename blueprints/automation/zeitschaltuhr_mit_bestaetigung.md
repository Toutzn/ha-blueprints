# 🗓️⏰ Zeitschaltuhr mit Schaltbestätigung

[![Blueprint importieren](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FToutzn%2Fha-blueprints%2Fblob%2Fmain%2Fblueprints%2Fautomation%2Fzeitschaltuhr_mit_bestaetigung.yaml)

Schaltet beliebige Entitäten nach einer oder mehreren **Zeitquellen** ein und aus – und
**prüft nach, ob es wirklich geklappt hat**. Verlorene Funkbefehle, kurz nicht erreichbare
Aktoren und verpasste Trigger nach einem Neustart fallen damit nicht mehr durchs Raster.

**Datei:** [`blueprints/automation/zeitschaltuhr_mit_bestaetigung.yaml`](zeitschaltuhr_mit_bestaetigung.yaml)
**Benötigt:** Home Assistant 2024.10 oder neuer

> **Icon für die Automation:** Der Blueprint-Name kann nur Text und Emoji enthalten – der
> `blueprint:`-Block hat kein Icon-Feld. Ein echtes mdi-Icon lässt sich aber auf der
> erzeugten Automation setzen: Einstellungen → Geräte & Dienste → Tab *Entitäten* → die
> `automation.*` anklicken → Zahnrad → *Symbol* → z. B. `mdi:calendar-clock`.

## Warum

Eine gewöhnliche Zeitschaltuhr-Automation feuert den Schaltbefehl ab und hofft das Beste.
Kommt der Befehl nicht an – Funkaussetzer, Aktor gerade `unavailable`, Home Assistant hat
im Schaltmoment neu gestartet – bleibt die Last unbemerkt an. Dieser Blueprint liest den
Ist-Zustand nach dem Schalten zurück, wiederholt gezielt für die Entitäten, die nicht
reagiert haben, und meldet erst dann Erfolg oder Fehlschlag.

## Zeitquellen: Zustand, nicht Ereignis

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

## Zwei Ebenen darüber

Über den Zeitquellen liegt eine Hierarchie mit **je eigenem Verhalten**, beide optional:

1. **Master-Schalter** – sinnbildlich der Stecker der Zeitschaltuhr. Hat Vorrang.
2. **Zeitschaltuhr aktiv** – die Betriebsart: regieren die Zeitquellen, oder Handbetrieb?

Damit lässt sich „Stecker gezogen = Last aus" und „Betriebsart aus = Hände weg" gleichzeitig
abbilden – zwei Anforderungen, die sich mit nur einem Schalter widersprechen würden. Beide
leer lassen = die Zeitquellen regieren immer, ohne Übersteuerung.

## Ablauf

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

## Eingaben

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

## Push aufs Handy

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

## Zusätzliche Aktionen

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
| `versuche` | Anzahl der Schaltversuche innerhalb **dieses** Laufs (1 = sofort geklappt) |

Die vorgegebenen Aktionen sind Platzhalter: „Aktivität protokollieren" (`logbook.log`)
schreibt nur ins HA-Logbuch, „Anhaltende Benachrichtigung erstellen"
(`persistent_notification.create`) nur in die Glocke der Oberfläche. Beide sind **keine**
Push-Benachrichtigungen – dafür ist die Geräteauswahl oben da. Wer sie nicht braucht,
kann sie löschen.

## Rezepte für Zeitquellen

Alles über Einstellungen → Geräte & Dienste → **Helfer** → *+ Helfer anlegen* →
*Template* → *Template für einen Binärsensor*. Das Template vorher unter
Entwicklerwerkzeuge → *Vorlagen* einfügen und live prüfen – dort steht sofort
`True` oder `False`.

### Die Regel: jede Sonnen-Bedingung braucht eine Tageshälften-Klammer

`elevation < x` heißt „es ist dunkel" – und das gilt **abends wie morgens**. Ohne
Einschränkung auf eine Tageshälfte wird das Fenster stillschweigend größer als gedacht,
und zwar an dem Ende, an dem man gerade nicht hinschaut.

| Kante | Klammer |
|---|---|
| Abend | `today_at('12:00') <= now()` |
| Morgen | `now() < today_at('12:00')` |

Weglassen darf man sie nur, wenn man wirklich die **ganze** Dunkelphase will.

### Die fünf Muster

| An | Aus | Zustandstemplate |
|---|---|---|
| Uhrzeit | Uhrzeit | kein Template – `schedule.*`-Helfer nehmen |
| Uhrzeit abends | Sonnenaufgang | `{{ now() >= today_at('23:00') or (now() < today_at('12:00') and state_attr('sun.sun','elevation') < 0) }}` |
| Sonnenuntergang | Uhrzeit abends | `{{ today_at('12:00') <= now() < today_at('23:00') and state_attr('sun.sun','elevation') < -2 }}` |
| Sonnenuntergang | Sonnenaufgang | `{{ state_attr('sun.sun','elevation') < -2 }}` |
| Uhrzeit morgens | Sonnenaufgang | `{{ today_at('06:00') <= now() < today_at('12:00') and state_attr('sun.sun','elevation') < 0 }}` |

Nur das zweite Muster ist ein **ODER** – es läuft über Mitternacht, und die beiden
Tageshälften müssen getrennt beschrieben werden. Alle anderen sind UND-Verknüpfungen
innerhalb eines Tages.

Beim letzten Muster ist zu bedenken, dass es im Sommer **gar nicht** schaltet: geht die
Sonne vor 06:00 auf, ist das Fenster leer. Das ist richtig so, überrascht aber, wenn man
es nicht erwartet.

### Der Fall über Mitternacht, ausgeschrieben

„An um 23:00, aus bei Sonnenaufgang" – das gängige Muster für Außenbeleuchtung, die
nicht die halbe Nacht brennen soll:

```jinja
{{ now() >= today_at('23:00')
   or (now() < today_at('12:00') and state_attr('sun.sun', 'elevation') < 0) }}
```

Nachgerechnet für Aachen (50,78° N):

| Tag | An | Aus |
|---|---|---|
| 09.09. | 23:00 | 07:06 |
| 21.12. | 23:00 | 08:42 |
| 21.06. | 23:00 | 05:29 |

> **Falle:** Ohne die Morgen-Klammer sieht die Formel richtig aus, ist es aber nicht:
>
> ```jinja
> {{ now() >= today_at('23:00') or state_attr('sun.sun','elevation') < 0 }}
> ```
>
> Der Sonnen-Zweig ist schon ab Sonnenuntergang wahr, also **schaltet der Sensor um
> 20:00 statt um 23:00** ein (21.12.: 16:26, 21.06.: 21:46) – und die eingestellte
> Uhrzeit ist wirkungslos, solange sie nach dem Sonnenuntergang liegt. Im Sommer fällt
> das nicht auf, im Winter sofort.

### Die Schwelle als Feinregler

`elevation` ist ein Winkel: `0` ist der geometrische Sonnenauf- bzw. -untergang, negative
Werte liegen in der Dämmerung. Wirkung am **Morgen-Ende** (aus bei Sonnenaufgang):

| Schwelle | Aus am 21.12. | Aus am 21.06. | |
|---:|---|---|---|
| `-6` | 07:56 | 04:36 | Ende der bürgerlichen Dämmerung, noch dunkel |
| `-2` | 08:26 | 05:12 | kurz vor Sonnenaufgang |
| **`0`** | **08:42** | **05:29** | **exakt Sonnenaufgang** |
| `2` | 08:59 | 05:45 | kurz danach |
| `5` | 09:26 | 06:08 | deutlich danach |

Die Schwelle wirkt an beiden Tagesenden **gegenläufig**: ein tieferer Wert heißt abends
später an, morgens aber früher aus. Für „an bei Sonnenuntergang" sind `-2` bis `-6`
sinnvoll, für „aus bei Sonnenaufgang" eher `0`.

### Weitere Quellen

**Morgens und abends in einem Sensor** – der Klassiker für den Weihnachtsbaum:

```jinja
{% set e = state_attr('sun.sun', 'elevation') %}
{{ (today_at('06:00') <= now() < today_at('12:00') and e < 0)
   or (today_at('12:00') <= now() < today_at('23:00') and e < -2) }}
```

Zwei Klammern, zwei Schwellen: morgens aus bei Sonnenaufgang, abends an in der Dämmerung.

**Anwesenheit** (als zweite Quelle mit UND zu verknüpfen, weil `person.*` selbst
`home`/`not_home` ist und nie `on`):

```jinja
{{ is_state('person.torsten', 'home') or is_state('person.partner', 'home') }}
```

**Ohne Template geht auch:** zwei `schedule.*`-Helfer mit ODER für „vormittags und
abends", oder ein `schedule.*` mit UND zu einem `binary_sensor` für „im Zeitfenster und
nur wenn jemand da ist".

### Aktualisierung

Diese Sensoren rechnen sich von allein neu: `elevation` ändert sich laufend, und
Templates, die `now()` enthalten, wertet Home Assistant jede Minute neu aus. Zusätzlich
prüft der Blueprint im eingestellten Intervall nach – ein verpasster Moment wird also
spätestens dort aufgefangen.

## Beispielkonfiguration: Poolsteuerung

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

## Beispielkonfiguration: Weihnachtsbaum draußen

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

## Trigger

| ID | Trigger | Zweck |
|---|---|---|
| `zeitquelle` | `state` auf die Zeitquellen, `to: "on"` / `to: "off"` | Regulärer Schaltzeitpunkt |
| `schalter` | `state` auf Master-Schalter und Betriebsart, `to: ["on", "off"]` | Sofortige Neubewertung |
| `neustart` | `homeassistant` / `start` | Nach einem Neustart verpasste Schaltzeitpunkte nachholen |
| `sync` | `time_pattern` | Verlorene Funkbefehle einfangen |

`mode: queued`, `max: 10`, `max_exceeded: silent`

## Designentscheidungen

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
- **Die Entscheidung fällt beim Ausführen, nicht beim Auslösen.** Variablen auf
  Automations-Ebene rendert Home Assistant im Moment des Triggers – *bevor* ein Lauf bei
  `mode: queued` in die Warteschlange geht. Ein zweiter Lauf würde also mit dem Zustand
  von vor seiner Wartezeit rechnen. Deshalb stehen dort nur die statischen `!input`-Werte;
  Soll-Zustand, Grund und der Soll-Ist-Vergleich werden als **Aktionsschritt** ermittelt.

  Ohne das passiert Folgendes: Fällt ein Schaltzeitpunkt mit einem `sync`-Tick zusammen –
  bei einem 5-Minuten-Intervall also zu jeder vollen Viertelstunde, halben Stunde und
  Stunde – feuern zwei Trigger in derselben Sekunde. Beide erheben ihr Lagebild, solange
  die Last noch im alten Zustand ist, beide halten sich für zuständig. Der erste schaltet,
  der zweite läuft mit leerer Zielliste durch und meldet trotzdem Erfolg. Ergebnis: **ein
  Schaltvorgang, zwei Erfolgsmeldungen**, beide mit „1 Schaltversuch".
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

## Bekannte Einschränkungen

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

Teil der Blueprint-Sammlung [`ha-blueprints`](../../README.md).
