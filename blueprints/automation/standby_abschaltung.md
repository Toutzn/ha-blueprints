# 🔌📉 Standby-Abschaltung mit Betriebserkennung

[![Blueprint importieren](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FToutzn%2Fha-blueprints%2Fblob%2Fmain%2Fblueprints%2Fautomation%2Fstandby_abschaltung.yaml)

Schaltet einen Verbraucher stromlos, sobald seine Leistungsaufnahme lange genug im
**Standby-Bereich** liegt: Waschmaschine fertig, Trockner fertig, Fernseher aus. Danach
wird **nachgeprüft**, ob der Aktor wirklich aus ist.

**Datei:** [`blueprints/automation/standby_abschaltung.yaml`](standby_abschaltung.yaml)
**Benötigt:** Home Assistant 2024.10 oder neuer

> **Ein Blueprint pro Verbraucher.** Schwellen und Nachlaufverhalten unterscheiden sich je
> Gerät, und ein Leistungssensor gehört immer zu genau einer Last.

## Warum

Der naive Ansatz „Leistung unter 10 W, also fertig" schaltet mitten im Waschgang ab. Eine
Waschmaschine zieht in der Heizphase über 2 kW, danach nur noch stoßweise, wenn die Trommel
dreht – zwischen zwei Stößen können Minuten liegen. Das Kriterium ist deshalb nie ein
einzelner Messwert, sondern ein **durchgehaltenes Zeitfenster**:

> Die Leistung liegt seit *20 Minuten* ununterbrochen unter *10 W*.

Steigt sie im Fenster wieder an, beginnt es von vorn. Der Blueprint hält dieses Fenster
selbst, statt sich auf ein `for:` am Trigger zu verlassen – nur so lassen sich einzelne
Spitzen gezielt tolerieren.

## Geräte mit Nachlauf: die Spitzen-Toleranz

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

## Betriebserkennung

Ohne Startschwelle (`0`) gilt: Verbraucher eingeschaltet + Leistung niedrig + Fenster voll
→ abschalten. Eine Waschmaschine, die eingeschaltet auf ihre Startzeitvorwahl wartet, wird
damit nach 20 Minuten stromlos gemacht, bevor sie überhaupt losgelegt hat.

Mit einer Startschwelle größer 0 wird erst abgeschaltet, **nachdem** die Leistung
mindestens einmal darüber lag. Faustwert: deutlich über der Standby-Schwelle, etwa die
Hälfte der Heizphasen-Leistung – 50 W passen fast immer.

Die Erkennung lebt im laufenden Automations-Durchlauf, nicht in einem Helfer pro Gerät. Das
kostet nichts an Einrichtung, hat aber eine Konsequenz: siehe *Bekannte Einschränkungen*.

## Eingaben

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

## Ablauf

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

## Trigger

| ID | Trigger | Zweck |
|---|---|---|
| `standby` | `numeric_state` unter der Schwelle | Der eigentliche Anlass |
| `ein` | `state` auf die Ziele, `to: on` | Durchlauf beginnt und wartet auf den Betrieb – nötig für die Betriebserkennung |
| `neustart` | `homeassistant` `start` | Nach Neustart oder Reload ist kein Durchlauf mehr aktiv |
| `sync` | `time_pattern` | Fängt ein, was zwischen den Triggern verloren ging |

`mode: single`, `max_exceeded: silent`: Ein Durchlauf begleitet einen kompletten
Gerätezyklus. Läuft schon einer, ist er zuständig, und jeder weitere Trigger wäre nur ein
Duplikat.

## Beispielkonfigurationen

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

## Designentscheidungen

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

## Traces lesen

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

## Bekannte Einschränkungen

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

## Migration von einer eigenen Automation

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

Teil der Blueprint-Sammlung [`ha-blueprints`](../../README.md).
