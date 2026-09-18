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

## Das eingebaute Zeitfenster

Für den häufigsten Fall – Sonnenuntergang, Sonnenaufgang, feste Uhrzeiten – braucht es
**keinen Helfer und kein Template**. Beginn und Ende werden direkt in der Automation
eingestellt:

| Feld | Auswahl |
|---|---|
| **Einschalten ab** | Kein Fenster · Sonnenuntergang · Feste Uhrzeit |
| **Ausschalten ab** | Kein Fenster · Sonnenaufgang · Feste Uhrzeit |

Alle Kombinationen sind erlaubt, auch über Mitternacht hinweg:

| Gewünscht | Einstellung |
|---|---|
| Außenlicht die ganze Nacht | Sonnenuntergang → Sonnenaufgang |
| Weihnachtsbeleuchtung bis 01:00 | Sonnenuntergang → Feste Uhrzeit `01:00` |
| Spätes Licht bis zum Morgen | Feste Uhrzeit `23:00` → Sonnenaufgang |
| Nur nachts bedienbar (mit *nur ausschalten*) | Sonnenuntergang → Sonnenaufgang |
| Reines Uhrzeitfenster | Feste Uhrzeit → Feste Uhrzeit |

**Beide Felder müssen gesetzt sein.** Steht eines auf „Kein Fenster", gilt das Fenster als
nicht konfiguriert und es zählen nur die Zeitquellen aus Abschnitt 2. Das ist auch die Vorgabe –
bestehende Automationen ändern sich durch das Fenster also nicht.

### Der Versatz in Minuten

Zu jeder Sonnenkante gehört ein Versatz – Minuten vor oder nach dem Ereignis, mit dem
Minuszeichen für „vorher". Das ist dieselbe Schreibweise wie beim Sonnen-Trigger im
Automations-Editor.

| Wunsch | Versatz |
|---|---|
| Schon in der Dämmerung einschalten | Sonnenuntergang **−30** |
| Erst wenn es wirklich dunkel ist | Sonnenuntergang **+20** |
| Genau zum Sonnenuntergang | Sonnenuntergang **0** (Vorgabe) |
| Morgens erst ausschalten, wenn es sicher hell ist | Sonnenaufgang **+30** |

Die Sonnenzeiten selbst kommen von Home Assistant und wandern über das Jahr mit; für
Aachen liegt der Sonnenuntergang am 21.12. gegen 16:33 und am 21.06. gegen 21:53.

Zu bedenken ist nur: 30 Minuten nach Sonnenuntergang sind im Juni eine andere Helligkeit
als im Dezember, weil die Dämmerung im Sommer länger dauert. Wer über das Jahr gleiche
Helligkeit will statt gleicher Minuten, nimmt einen Template-Binärsensor auf
`sun.sun`-`elevation` als zusätzliche Zeitquelle (Rezepte unten).

### Zusammenspiel mit den Zeitquellen

Das Fenster zählt wie **eine weitere Zeitquelle**: bei *ODER* genügt es allein, bei *UND*
muss es zusammen mit allen anderen Quellen `on` sein. So lässt sich etwa „nur bei
Dunkelheit **und** wenn jemand zu Hause ist" ohne ein einziges Template bauen – Fenster
plus Anwesenheits-Binärsensor als Quelle, Verknüpfung UND.

Mehr als ein eingebautes Fenster pro Automation gibt es nicht. Wer zwei Fenster braucht
(morgens und abends), nimmt dafür einen `schedule.*`-Helfer als zusätzliche Quelle oder
legt eine zweite Automation an.

## Zeitquellen: Zustand, nicht Ereignis

Zeitquellen sind jetzt **optional** – für alles, was das eingebaute Fenster nicht abdeckt:
Wochentage, mehrere Blöcke pro Tag, Anwesenheit, Kalender, eigene Bedingungen.

Die Zeitquelle ist **jede Entität mit `on`/`off`** – ein `schedule.*`-Helfer, ein
Template-Binärsensor, eine Kalender-Entität, eine Gruppe. Mehrere Quellen lassen sich
per ODER (Fenster addieren) oder UND (Bedingungen kombinieren) verknüpfen.

Warum ein Zustand und kein Trigger? Der Blueprint ist selbstheilend: er fragt zyklisch
und nach jedem Neustart *„was soll jetzt gerade sein?"*. Ein `sun`-Trigger feuert im
Moment des Sonnenuntergangs und ist dann vorbei – startet HA zwanzig Minuten später neu,
weiß niemand mehr, dass die Last an sein sollte. Ein Zustand lässt sich jederzeit
befragen, ein Ereignis nicht.

Deshalb passen diese Entitäten **nicht** direkt als Quelle, weil ihr Zustand nicht
`on`/`off` ist – ein Template-Binärsensor davor löst das (Rezepte unten). Für `sun.sun`
ist das seit dem eingebauten Fenster nicht mehr nötig:

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

## Schaltrichtung: Schaltuhr oder Sperre

Normalerweise geben die Zeitquellen beide Richtungen vor: Fenster auf → ein, Fenster zu →
aus. Mit der **Schaltrichtung** lässt sich eine Richtung abschalten, und aus derselben
Mechanik wird etwas anderes:

| Richtung | Quelle `on` | Quelle `off` | Wofür |
|---|---|---|---|
| **Beides** (Default) | schaltet ein | schaltet aus | Die klassische Zeitschaltuhr |
| **Nur ausschalten** | tut nichts | schaltet aus | Sperre: im Fenster frei, außerhalb konsequent aus |
| **Nur einschalten** | schaltet ein | tut nichts | Automatik schaltet an, du schaltest aus |

Das typische Beispiel für *nur ausschalten* ist Außenlicht: tagsüber hat es nichts zu
suchen, nachts soll es aber ganz normal von Hand bedienbar sein. Die Zeitquelle ist dann
schlicht ein Binärsensor „es ist dunkel"; wer tagsüber einschaltet, bekommt das Licht
umgehend wieder ausgeschaltet, und was morgens noch brennt, geht mit dem Sonnenaufgang aus.

Die Richtung betrifft **nur die Zeitquellen**. Master-Schalter und Betriebsart behalten ihr
eigenes Verhalten – ein dort eingestelltes Zwangs-Aus greift unabhängig davon weiter. Sonst
ließe sich die „Stecker gezogen"-Regel unbemerkt aushebeln.

## Sofort nachregeln

Ohne Zutun merkt die Automation erst beim nächsten Nachprüf-Tick, dass jemand von Hand
geschaltet hat – im Standardintervall also bis zu fünf Minuten später. Die Option **Sofort
nachregeln, wenn fremd geschaltet wird** horcht zusätzlich auf die Ziel-Entitäten selbst
und bewertet die Lage augenblicklich neu. Erfasst wird jedes Schalten: Wandschalter,
App, Szene, Sprachassistent.

Für eine Sperre ist das der eigentliche Unterschied zwischen „das Licht geht sofort wieder
aus" und „das Licht brennt noch ein paar Minuten". Für eine gewöhnliche Zeitschaltuhr ist
die Option meist überflüssig – deshalb ist sie aus.

## Ablauf

```
Trigger (Fensterkante / Zeitquelle / Schalter / Ziel / Neustart / zyklisch)
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
Soll = Fenster + Zeitquellen, ODER/UND-verknüpft (on / off)
        │                             durch die Schaltrichtung ggf. auf
        │                             „egal" gesetzt (Sperre)
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

Die Oberfläche ist in sieben Abschnitte gegliedert, in der Reihenfolge, in der man beim
Einrichten denkt: **wann** geschaltet wird, unter welchen **Bedingungen**, **was**
geschaltet wird, wer von Hand **übersteuern** darf, wie das Schalten **bestätigt** wird,
und wie **gemeldet** wird.

### 1 · Wann geschaltet werden soll

| Feld | Default | Beschreibung |
|---|---|---|
| **Einschalten ab** | Kein Fenster | *Kein Fenster* · *Sonnenuntergang* · *Feste Uhrzeit* |
| **Versatz zum Sonnenuntergang** | 0 | Minuten vor (−) oder nach (+) dem Sonnenuntergang |
| **Uhrzeit für den Beginn** | 23:00 | Nur bei „Feste Uhrzeit" |
| **Ausschalten ab** | Kein Fenster | *Kein Fenster* · *Sonnenaufgang* · *Feste Uhrzeit* |
| **Versatz zum Sonnenaufgang** | 0 | Minuten vor (−) oder nach (+) dem Sonnenaufgang |
| **Uhrzeit für das Ende** | 01:00 | Nur bei „Feste Uhrzeit". Liegt sie vor dem Beginn, läuft das Fenster über Mitternacht |
| **Schaltrichtung** | Beides | *Beides* · *Nur ausschalten* (Sperre) · *Nur einschalten* |

### 2 · Zusätzliche Bedingungen

| Feld | Default | Beschreibung |
|---|---|---|
| **Weitere Zeitquellen** | – | Entitäten mit `on`/`off` für alles, was das Fenster nicht abdeckt. Leer lassen, wenn das Fenster genügt |
| **Verknüpfung** | ODER | *ODER* = eine genügt · *UND* = alle müssen zutreffen. Das Fenster zählt als eine davon |

### 3 · Was geschaltet wird

| Feld | Pflicht | Default | Beschreibung |
|---|---|---|---|
| **Zu schaltende Entitäten** | ja | – | `switch`, `light`, `input_boolean`, `fan`, `media_player` – Entitäten, keine Geräte |
| **Sofort nachregeln, wenn fremd geschaltet wird** | nein | aus | Horcht auf die Ziele selbst, statt bis zum nächsten Nachprüf-Durchlauf zu warten |

### 4 · Von Hand übersteuern

| Feld | Default | Beschreibung |
|---|---|---|
| **Master-Schalter** | – | Oberste Ebene, „der Stecker". Alle müssen `on` sein. Hat Vorrang vor allem anderen |
| **… und wenn der Master aus ist?** | Zwangs-Aus | *Zwangs-Aus* = Ziele werden ausgeschaltet · *Handbetrieb* = Automatik hält sich raus |
| **Zeitschaltuhr aktiv** | – | Betriebsart-Schalter. Leer = kein Handbetrieb |
| **… und wenn die Betriebsart aus ist?** | Handbetrieb | *Handbetrieb* = Zustand bleibt, du schaltest frei · *Zwangs-Aus* = Ziele werden ausgeschaltet |

### 5 · Schaltbestätigung und Selbstheilung

| Feld | Default | Beschreibung |
|---|---|---|
| **Wartezeit vor der Nachprüfung** | 10 s | Bei Funk-Aktoren großzügiger wählen |
| **Maximale Schaltversuche** | 5 | Wiederholt nur für die Entitäten, die noch nicht reagiert haben |
| **Zyklische Nachprüfung** | ein | Korrigiert Abweichungen laufend |
| **Intervall der Nachprüfung** | 5 min | `/1`, `/5`, `/15` oder `/30` |

### 6 · Benachrichtigungen aufs Handy

| Feld | Default | Beschreibung |
|---|---|---|
| **Push bei Erfolg** | – | Geräte mit HA-App. Leer = kein Push bei Erfolg |
| **Push bei Fehlschlag** | – | Dasselbe für den Fehlerfall |
| **Titel der Benachrichtigung** | `Zeitschaltuhr` | Überschrift der Meldung |
| **Fehlermeldungen aus der Nachprüfung** | unterdrücken | Verhindert, dass ein totes Gerät alle paar Minuten meldet |
| **Fehlschlag als kritische Benachrichtigung** | aus | Kommt auch bei stummem Telefon durch |

### 7 · Eigene Aktionen

| Feld | Default | Beschreibung |
|---|---|---|
| **Aktion bei Erfolg** | Logbuch-Eintrag | Zusätzlich zum Push – **kein** Benachrichtigungsfeld |
| **Aktion bei Fehlschlag** | Persistente Benachrichtigung | Dasselbe für den Fehlerfall |

## Push aufs Handy

Dafür genügt es, in Abschnitt *6 · Benachrichtigungen aufs Handy* die Geräte auszuwählen – getrennt für
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
| `grund` | `zeitquelle`, `nachgeregelt`, `master_aus`, `zeitschaltuhr_aus` oder `unbekannt` |
| `grund_text` | Dasselbe im Klartext, z. B. `Master-Schalter aus` |
| `ziel_liste` | Alle konfigurierten Ziel-Entitäten |
| `restliche` | Erreichbare Entitäten, die den Soll-Zustand **nicht** erreicht haben |
| `nicht_erreichbar` | Entitäten ohne `on`/`off`-Zustand – offline, abgesteckt, Integration weg |
| `versuche` | Anzahl der Schaltversuche innerhalb **dieses** Laufs (1 = sofort geklappt) |

Die vorgegebenen Aktionen sind Platzhalter: „Aktivität protokollieren" (`logbook.log`)
schreibt nur ins HA-Logbuch, „Anhaltende Benachrichtigung erstellen"
(`persistent_notification.create`) nur in die Glocke der Oberfläche. Beide sind **keine**
Push-Benachrichtigungen – dafür ist die Geräteauswahl oben da. Wer sie nicht braucht,
kann sie löschen.

## Rezepte für eigene Zeitquellen

> **Für Sonne und Uhrzeit ist das nicht mehr nötig** – dafür gibt es das eingebaute
> Zeitfenster weiter oben. Dieses Kapitel bleibt für alles, was darüber hinausgeht: zwei
> Fenster an einem Tag, Bedingungen mit Anwesenheit, Kalender oder Wetter – und für den
> Fall, dass über das Jahr gleiche *Helligkeit* statt gleicher Minuten gewünscht ist.
> Dafür ist die Sonnenhöhe das richtige Maß, und die folgenden Muster zeigen, worauf man
> dabei achten muss.

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

### Die sechs Muster

| An | Aus | Zustandstemplate |
|---|---|---|
| Uhrzeit | Uhrzeit | kein Template – `schedule.*`-Helfer nehmen |
| Uhrzeit abends | Sonnenaufgang | `{{ now() >= today_at('23:00') or (now() < today_at('12:00') and state_attr('sun.sun','elevation') < 0) }}` |
| Sonnenuntergang | Uhrzeit abends | `{{ today_at('12:00') <= now() < today_at('23:00') and state_attr('sun.sun','elevation') < -2 }}` |
| Sonnenuntergang | Uhrzeit nach Mitternacht | `{{ (today_at('12:00') <= now() and state_attr('sun.sun','elevation') < -2) or now() < today_at('01:00') }}` |
| Sonnenuntergang | Sonnenaufgang | `{{ state_attr('sun.sun','elevation') < -2 }}` |
| Uhrzeit morgens | Sonnenaufgang | `{{ today_at('06:00') <= now() < today_at('12:00') and state_attr('sun.sun','elevation') < 0 }}` |

**ODER** sind genau die beiden Muster, die über Mitternacht laufen – dort müssen die beiden
Tageshälften getrennt beschrieben werden. Alle übrigen sind UND-Verknüpfungen innerhalb
eines Tages.

Beim vierten Muster (Sonnenuntergang → Uhrzeit nach Mitternacht) fällt auf, dass die
Morgenhälfte **ohne** Sonnenbedingung auskommt: bis 01:00 ist es in unseren Breiten
ohnehin dunkel, und der Sonnenstand würde die Grenze nur unscharf machen. Die
Tageshälften-Klammer steckt hier in `today_at('12:00') <= now()` auf der Abendseite.

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

## Beispielkonfiguration: Gartenlicht nur bei Dunkelheit

Keine Schaltuhr, sondern eine Sperre: nachts bedienst du das Licht ganz normal von Hand,
tagsüber lässt die Automation es nicht zu, und was morgens noch brennt, geht mit dem
Sonnenaufgang aus.

Kein Helfer, kein Template – alles steht in der Automation:

| Feld | Wert |
|---|---|
| Zeitquellen | *leer* |
| Einschalten ab | **Sonnenuntergang** |
| Ausschalten ab | **Sonnenaufgang** |
| Versatz zum Sonnenuntergang | 0 (oder +20, wenn erst bei richtiger Dunkelheit freigegeben werden soll) |
| Versatz zum Sonnenaufgang | 0 (oder +30, damit erst bei sicherem Tageslicht abgeschaltet wird) |
| Schaltrichtung | **Nur ausschalten** |
| Master-Schalter | *leer* |
| Zeitschaltuhr aktiv | *leer* |
| Zu schaltende Entitäten | die Gartenlichter |
| Sofort nachregeln | **ein** – sonst brennt das Licht bis zum nächsten Nachprüf-Tick |
| Zyklische Nachprüfung | ein, Intervall `/5` |
| Push bei Erfolg | *leer* – sonst meldet sich jeder Fehlgriff aufs Handy |
| Push bei Fehlschlag | eigenes Handy |

Warum das ohne Master und ohne Betriebsart auskommt: „nur ausschalten" heißt, dass die
Automatik im Fenster gar keinen Soll-Zustand hat. Handbetrieb ist damit der Normalfall und
braucht keinen eigenen Schalter.

Soll das Licht zeitweise auch nachts gesperrt sein (Urlaub, Nachtruhe), kommt ein
`input_boolean` als **Master-Schalter** mit Verhalten *Zwangs-Aus* dazu – der greift
unabhängig von der Schaltrichtung.

## Beispielkonfiguration: Weihnachtsbeleuchtung draußen

Hier soll wirklich geschaltet werden: an bei Sonnenuntergang, aus um 01:00. Kein
Handbetrieb, kein Master – nur Zeitquelle und Ziel:

| Feld | Wert |
|---|---|
| Zeitquellen | *leer* |
| Einschalten ab | **Sonnenuntergang** |
| Ausschalten ab | **Feste Uhrzeit**, `01:00` |
| Versatz zum Sonnenuntergang | 0 |
| Schaltrichtung | Beides |
| Master-Schalter | *leer* (oder ein `input_boolean.weihnachtszeit` mit Zwangs-Aus, dann endet die Saison mit einem Klick) |
| Zeitschaltuhr aktiv | *leer* |
| Zu schaltende Entitäten | `switch.tannenbaum_aussen` |
| Sofort nachregeln | aus – hier soll die Uhr regieren, nicht gegen den Nutzer arbeiten |
| Push bei Fehlschlag | eigenes Handy |
| Push bei Erfolg | *leer* – zweimal täglich eine Meldung will man nicht |
| Fehlermeldungen bei der Nachprüfung | unterdrücken – die Steckdose im Garten fällt im Winter öfter aus |

Die Nachprüfung sorgt dafür, dass die Lichterkette auch nach einem WLAN-Aussetzer
angeht, und meldet, wenn die Steckdose im Garten gar nicht mehr reagiert.

## Trigger

| ID | Trigger | Zweck |
|---|---|---|
| `zeitquelle` | `state` auf die Zeitquellen, `to: "on"` / `to: "off"` | Regulärer Schaltzeitpunkt |
| `fenster` | `sun` auf Sonnenuntergang und Sonnenaufgang mit dem eingestellten Versatz sowie `time` auf die beiden eingestellten Uhrzeiten | Kanten des eingebauten Fensters |
| `schalter` | `state` auf Master-Schalter und Betriebsart, `to: ["on", "off"]` | Sofortige Neubewertung |
| `ziel` | `state` auf die Ziel-Entitäten, `to: ["on", "off"]` | Fremdes Schalten sofort nachregeln – nur wirksam, wenn die Option eingeschaltet ist |
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
- **„Reagiert nicht" und „ist nicht da" sind zwei Fehler.** Ein Ziel mit Zustand
  `unavailable` oder `unknown` lässt sich nicht schalten – es wiederholt anzufunken kostet
  nur Zeit und sagt nichts Neues. Solche Ziele landen in `nicht_erreichbar`, werden von der
  Retry-Schleife übersprungen und getrennt gemeldet. Das führt auch zu einer anderen
  Handlung: „reagiert nicht" heißt Aktor prüfen, „nicht erreichbar" heißt Funkstrecke oder
  Integration prüfen.
- **`while` statt `until` in der Schleife.** `until` prüft erst *nach* dem Durchgang, würde
  also selbst dann einen Leerbefehl absetzen und die Wartezeit verbrauchen, wenn es nichts
  zu schalten gibt. Bei einem toten Gerät waren das 50 Sekunden Leerlauf pro Durchlauf.
  `while` prüft vorher – die Schleife läuft dann null Mal.
- **Fehlermeldungen kennen den Auslöser.** Eine Störung, die bestehen bleibt, würde bei
  jedem Nachprüf-Tick erneut melden. Standardmäßig melden deshalb nur Läufe, die von einem
  echten Ereignis kommen – Schaltzeitpunkt, Schalter, Neustart. **Korrigiert** wird
  weiterhin bei jedem Durchlauf, und **Erfolgsmeldungen** gehen immer raus: die kommt genau
  einmal, wenn es endlich geklappt hat.
- **Ein Notification-Tag pro Automation.** Neue Meldungen ersetzen die vorige, statt das
  Mitteilungszentrum zu füllen.
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
- **Das eingebaute Fenster zählt wie eine Zeitquelle, nicht daneben.** Es fließt in
  dieselbe ODER/UND-Verknüpfung ein, statt eine eigene Vorrangebene zu bekommen. Damit
  bleibt die Entscheidungsmatrix unverändert – es kommt eine Quelle hinzu, keine neue
  Ebene. „Nur bei Dunkelheit und nur wenn jemand da ist" ist dann Fenster + Quelle mit UND.
- **Sonnenkanten als `sun`-Trigger mit Versatz in Minuten.** Das ist die Schreibweise,
  die Home Assistant im Automations-Editor selbst verwendet – wer den Blueprint verlässt
  und von Hand weiterbaut, findet dieselben Begriffe wieder. Der Versatz steckt als
  Duration-Eingabe im Trigger *und* in der Auswertung, damit beide zum selben Zeitpunkt
  umschalten.
- **Das Fenster rechnet mit Zeitpunkten, nicht mit Tageshälften.** Beide Kanten werden auf
  einen konkreten Zeitpunkt des heutigen Tages gebracht – die Sonnenkanten aus
  `next_setting` / `next_rising`, die Uhrzeit-Kanten aus `today_at()`. Danach ist es ein
  Intervallvergleich, und liegt das Ende vor dem Beginn, läuft das Fenster eben über
  Mitternacht. Die Tageshälften-Klammer der handgeschriebenen Rezepte entfällt damit
  ersatzlos – sie war nur nötig, weil „Sonnenhöhe unter x" keine Richtung kennt.
- **Ein halbes Fenster gilt als kein Fenster.** Steht nur eine der beiden Kanten, gibt es
  keinen definierten Fensterzustand. Statt zu raten (Rest des Tages? bis Mitternacht?)
  wird das Fenster ignoriert – sichtbar daran, dass ohne weitere Zeitquelle gar nichts
  passiert.
- **Die Tageshälften-Klammer steckt fest im Blueprint.** `elevation < x` heißt abends wie
  morgens „dunkel". Welche Hälfte gemeint ist, leitet der Blueprint aus der Kombination ab:
  eine Endzeit vor 12:00 bedeutet „nach Mitternacht", eine Startzeit ab 12:00 bedeutet
  „abends". Das ist genau die Regel, an der die handgeschriebenen Rezepte am häufigsten
  scheitern (siehe Rezept-Kapitel).
- **Die Schaltrichtung wirkt nur auf die Zeitquellen-Ebene.** Master-Schalter und
  Betriebsart haben ihr eigenes, ausdrücklich eingestelltes Verhalten; würde die Richtung
  auch darauf wirken, ließe sich ein Zwangs-Aus über eine ganz anderslautende Einstellung
  aushebeln. Ein „egal" aus der Richtung heißt deshalb nur: *die Zeitquellen geben für
  diese Richtung nichts vor*.
- **Der Ziel-Trigger ist immer vorhanden, seine Verarbeitung ist die Option.** Ein
  Trigger-Block lässt sich in einem Blueprint nicht per Eingabe weglassen. Die Prüfung
  steht deshalb als erster Aktionsschritt neben der Nachprüf-Prüfung – zulässig, weil
  `mode: queued` einen laufenden Durchlauf nicht abbricht, sondern nur einreiht. Bei
  `mode: restart` müsste sie in `conditions:` (siehe das Bewegungslicht).
- **Kein Kontextfilter am Ziel-Trigger.** Man könnte über
  `trigger.to_state.context.parent_id is none` auf rein manuelles Schalten filtern. Genau
  das wäre hier falsch: eine Szene oder ein Sprachassistent, der das Licht am Tag
  einschaltet, soll ebenso korrigiert werden. Preis ist ein Folgelauf nach jedem eigenen
  Schaltvorgang, der sofort an `Ist == Soll` abbricht.
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
- **Die Sonnenzeit nach dem Ereignis ist auf wenige Minuten genau.** Für den heutigen
  Sonnenuntergang rechnet der Blueprint `next_setting` minus einen Tag zurück, sobald das
  Ereignis vorbei ist. Rund um die Tagundnachtgleiche wandert der Sonnenuntergang täglich
  um zwei bis vier Minuten, der zurückgerechnete Wert weicht also entsprechend ab. Der
  `sun`-Trigger feuert trotzdem exakt; im ungünstigen Fall schaltet die Automation erst
  mit dem nächsten Nachprüf-Durchlauf. Für Beleuchtung ohne Belang – wer es genauer
  braucht, stellt die Nachprüfung auf „jede Minute".
- **„Feste Uhrzeit morgens → Sonnenaufgang" wird über Mitternacht gelesen.** Liegt der
  Sonnenaufgang im Sommer vor der eingestellten Uhrzeit (etwa 06:00 gegen Aufgang 05:23),
  ergibt das kein leeres Fenster, sondern ein fast tagesfüllendes: ab 06:00 bis zum
  *nächsten* Sonnenaufgang am Morgen darauf. Das ist die konsequente Lesart eines Fensters,
  dessen Ende vor seinem Beginn liegt – gewollt ist meist etwas anderes. Für „morgens von
  06:00 bis es hell wird" gehört die Sonnenbedingung in eine eigene Zeitquelle.
- **Ein Fenster pro Automation.** Zwei getrennte Fenster an einem Tag (morgens und abends)
  gehen nur über einen `schedule.*`-Helfer als zusätzliche Zeitquelle oder über eine zweite
  Automation.
- **Die Fenster-Trigger feuern auch, wenn das Fenster nicht benutzt wird.** Die beiden
  Zeit-Trigger hängen an den eingestellten Uhrzeiten (Vorgabe 23:00 und 01:00), die
  Sonnen-Trigger an der Dämmerungsschwelle – ein Trigger-Block lässt sich in einem
  Blueprint nicht per Eingabe weglassen. Solche Läufe enden beim Soll-Ist-Vergleich, ohne
  zu schalten und ohne zu melden; sie belegen nur einen Trace-Platz.
- **Sofortiges Nachregeln füllt den Trace-Puffer.** Jeder eigene Schaltvorgang löst den
  Ziel-Trigger mit aus. Der Folgelauf findet Soll = Ist und endet sofort, belegt aber einen
  der standardmäßig **5** gespeicherten Traces – der interessante Lauf ist dann schneller
  verdrängt. Wer viel in Traces liest, setzt in der Automation
  `trace: { stored_traces: 25 }`.
- Stille kann drei Dinge bedeuten: Automatik nicht zuständig, Zustand stimmte schon, oder
  die Automation lief nicht. Unterscheiden lässt sich das nur über die Traces.
- Ein dauerhaft nicht erreichbares Ziel wird **einmal** gemeldet – beim nächsten echten
  Ereignis. Bleibt es weg, bleibt es still. Wer stattdessen eine Dauererinnerung möchte,
  stellt *Fehlermeldungen bei der zyklischen Nachprüfung* auf „Auch bei jedem Durchlauf".
- Nach einem HA-Neustart kann es bis zum nächsten Nachprüf-Intervall dauern, bis
  korrigiert wird – der `homeassistant.start`-Trigger läuft ins Leere, solange die
  Zeitquellen noch `unavailable` sind. Mit dem Default von 5 Minuten unkritisch.
- Die Master-Schalter werden mit Default `[]` als `entity_id` in einem State-Trigger
  verwendet. Eine leere Liste registriert einfach keinen Listener. Sollte eine HA-Version das
  beim Speichern beanstanden, den betreffenden Trigger entfernen – die zyklische Nachprüfung
  fängt Änderungen dann verzögert ab.

---

Teil der Blueprint-Sammlung [`ha-blueprints`](../../README.md).
