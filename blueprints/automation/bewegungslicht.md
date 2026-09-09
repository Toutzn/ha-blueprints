# 🚶💡 Bewegungslicht mit Dunkelheitsprüfung und Schaltbestätigung

[![Blueprint importieren](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FToutzn%2Fha-blueprints%2Fblob%2Fmain%2Fblueprints%2Fautomation%2Fbewegungslicht.yaml)

Braucht Home Assistant **2024.10** oder neuer.

## Zweck

Licht bei Bewegung einschalten und nach einer Nachlaufzeit ohne Bewegung wieder ausschalten
— mit drei Dingen, die der handgeschriebenen Variante meist fehlen:

1. **Mehrere Melder für ein Licht.** Jede Meldung schaltet ein, die Nachlaufzeit beginnt
   erst, wenn *alle* Melder wieder frei melden.
2. **Dunkelheitsprüfung wahlweise über Sonnenstand, Lux-Sensor oder beides.** So lässt sich
   mit der bewährten Sonnen-Bedingung starten und die Lux-Schwelle in Ruhe dazunehmen.
3. **Schaltbestätigung in beide Richtungen.** Nach jedem Befehl wird der Ist-Zustand
   zurückgelesen, bei Bedarf wiederholt und ein endgültiger Fehlschlag gemeldet.

Dazu kommt der Umgang mit manuellem Schalten: Wer am Wandschalter einschaltet, bekommt die
automatische Abschaltung trotzdem — und wer während der Wartezeit von Hand ausschaltet,
wird nicht überstimmt.

## Eingaben (25)

### Bewegung und Licht

| Feld | Pflicht | Default | Beschreibung |
|---|---|---|---|
| `bewegungsmelder` | ja | – | Ein oder mehrere `binary_sensor` (motion / occupancy / presence) |
| `lichter` | ja | – | Zu schaltende **Entitäten** (`light`, `switch`, `input_boolean`) |
| `nachlaufzeit` | nein | `60` s | Brenndauer, nachdem der letzte Melder frei gemeldet hat |
| `sicherheits_timeout` | nein | `0` min | Notbremse für hängende Melder; `0` = aus (harte Grenze 24 h) |

### Wann ist es dunkel genug?

| Feld | Pflicht | Default | Beschreibung |
|---|---|---|---|
| `dunkel_modus` | nein | `sonne` | `sonne` \| `helligkeit` \| `beides_und` \| `beides_oder` \| `immer` |
| `offset_sonnenuntergang` | nein | `-00:10:00` | Versatz zum Sonnenuntergang, negativ = früher |
| `offset_sonnenaufgang` | nein | `00:30:00` | Versatz zum Sonnenaufgang, positiv = länger dunkel |
| `helligkeitssensoren` | nein | `[]` | Lux-Sensoren; bei mehreren entscheidet der **kleinste** Wert |
| `lux_schwelle` | nein | `20` lx | Darunter gilt der Raum als dunkel genug |

### Einschaltverhalten

| Feld | Pflicht | Default | Beschreibung |
|---|---|---|---|
| `einschalt_helligkeit` | nein | `0` % | Helligkeit für `light.*`; `0` = keine Vorgabe |
| `uebergangszeit` | nein | `0` s | Weiches Aufblenden, nur zusammen mit einer Helligkeit |
| `nachtmodus` | nein | `false` | Zweiter Helligkeitswert in einem Zeitfenster |
| `nacht_start` | nein | `23:00:00` | Beginn des Nachtfensters |
| `nacht_ende` | nein | `06:00:00` | Ende des Nachtfensters (Übergang über Mitternacht erlaubt) |
| `nacht_helligkeit` | nein | `20` % | Helligkeit im Nachtfenster |

### Manuelles Schalten

| Feld | Pflicht | Default | Beschreibung |
|---|---|---|---|
| `manueller_trigger` | nein | `true` | Auf manuelles Einschalten reagieren |
| `manuelles_aus_respektieren` | nein | `true` | Manuelles Aus während der Wartezeit beendet den Durchlauf |

### Sperren

| Feld | Pflicht | Default | Beschreibung |
|---|---|---|---|
| `sperr_entitaeten` | nein | `[]` | Sperre greift, solange eine davon `on` ist |
| `sperr_verhalten` | nein | `beides` | `beides` \| `nur_ausschalten` \| `nur_einschalten` |

### Schaltbestätigung und Meldungen

| Feld | Pflicht | Default | Beschreibung |
|---|---|---|---|
| `pruefung_verzoegerung` | nein | `5` s | Wartezeit vor dem Zurücklesen des Ist-Zustands |
| `max_versuche` | nein | `3` | Maximale Schaltversuche je Richtung |
| `push_geraete_fehler` | nein | `[]` | Geräte-Selektor (`integration: mobile_app`), nur bei Fehlschlag |
| `push_titel` | nein | `Bewegungslicht` | Überschrift der Push-Meldung |
| `push_kritisch` | nein | `false` | Fehler-Push als kritischer iOS-Alert |
| `aktion_bei_fehler` | nein | `persistent_notification.create` | Zusatzaktion, läuft **neben** dem Push |

Variablen für eigene Aktionen: `phase` (`einschalten` / `ausschalten`), `restliche` (Liste
der Entitäten, die nicht reagiert haben), `versuche`, `melde_text`.

## Ablauf

0. **Torwächter im `conditions`-Block** (nicht als Aktion, siehe unten): Ein
   `manuell`-Auslöser zählt nur, wenn er wirklich von Hand kam
   (siehe [Designentscheidungen](#warum-der-kontext-des-zustandswechsels-geprüft-wird)),
   und bei einer Vollsperre wird der Auslöser gar nicht erst angenommen.
1. **Einschalten** — nur bei Bewegung, nur wenn es dunkel genug ist, nur für Ziele, die
   gerade aus sind. `light.*`-Ziele bekommen dabei die eingestellte Helligkeit,
   alles andere `homeassistant.turn_on`. Danach Ist-Zustand zurücklesen, für die
   Nachzügler wiederholen, bei endgültigem Fehlschlag melden.
2. **Warten, bis alle Melder frei melden** (mit Sicherheits-Timeout).
3. **Nachlaufzeit** — als Wartebedingung, damit ein manuelles Aus sofort abbricht.
4. **Ausschalten**, sofern noch etwas nicht aus ist und keine Sperre greift — wieder mit
   Zurücklesen, Wiederholung und Meldung.

Jede neue Bewegung startet den Durchlauf neu (`mode: restart`), die Nachlaufzeit beginnt
also von vorn. Ein Timer-Helfer ist dafür nicht nötig.

## Trigger

| ID | Trigger | Zweck |
|---|---|---|
| `bewegung` | `state` auf die Melder, `to: 'on'` | Einschalten und Zeit zurücksetzen |
| `manuell` | `state` auf die Lichter, `from: 'off'` `to: 'on'` | Abschaltung auch nach manuellem Einschalten |

`mode: restart`, `max_exceeded: silent`.

## Designentscheidungen

### Die Dunkelheitsprüfung sitzt im Einschaltzweig, nicht im `conditions`-Block

Läge sie oben, würde ein Durchlauf am helllichten Tag sofort abbrechen — und von Hand
eingeschaltetes Licht bliebe ewig an. So kommt jeder Durchlauf bis zum Ausschalten; nur das
*Einschalten* hängt an der Helligkeit. Wird es während der Brenndauer hell, geht das Licht
trotzdem regulär nach der Nachlaufzeit aus.

Denselben Effekt hat es beim Lux-Sensor: Weil nur im Moment des Einschaltens gemessen wird,
stört es nicht, dass das eigene Licht den Messwert danach hochzieht.

### Fünf Modi statt zweier getrennter Bedingungen

Sonnenstand und Lux sind je nach Raum unterschiedlich brauchbar. `beides_oder` ist der
sichere Einstieg, solange die Lux-Schwelle noch nicht sitzt: es schaltet ein, sobald *eine*
der beiden Bedingungen zutrifft. `beides_und` spart am meisten, verzeiht aber keinen
defekten Sensor. Umgesetzt ist das als eine ODER-Verknüpfung aus drei Zweigen (Modus
`immer`, Sonnen-Zweig, Lux-Zweig), weil die Sonnenbedingung eine echte HA-Condition mit
`!input`-Offsets ist und nicht in ein Template passt.

### Warum die beiden Torwächter im `conditions`-Block stehen und nicht in `actions`

Das ist keine Stilfrage, sondern der Unterschied zwischen „funktioniert" und „das Licht
bleibt an". Home Assistant arbeitet einen Auslöser in dieser Reihenfolge ab
(`automation/__init__.py`, `async_trigger`):

```
variables rendern  →  conditions prüfen  →  bei false: RETURN
                                         →  sonst: action_script.async_run()
```

Erst in `async_run` greift `mode: restart` und bricht einen laufenden Durchlauf ab. Eine
Prüfung, die als **erste Aktion** formuliert ist, kommt also zu spät: Der wartende
Durchlauf ist bereits abgeräumt, wenn die Bedingung ihn ablehnt — und mit ihm die
Ausschaltung. Im `conditions`-Block dagegen wird der Auslöser verworfen, bevor überhaupt
etwas abgebrochen wird, und der laufende Durchlauf zählt in Ruhe seine Nachlaufzeit
herunter.

Genau daran ist die erste Fassung gescheitert: Die Automation schaltete das Licht ein, ihr
eigener Zustandswechsel löste den `manuell`-Trigger aus, `mode: restart` beendete den
gerade gestarteten Durchlauf, und der neue Durchlauf stieg bei der Auslöser-Prüfung sofort
wieder aus. Zurück blieb brennendes Licht ohne wartende Automation. Sichtbar wird das im
Trace als Durchlauf, der nach `Nur echte Auslöser verarbeiten` mit `result: false` endet —
direkt hinter einem abgebrochenen Durchlauf.

Nebenwirkung, die man kennen sollte: Auch die Vollsperre nimmt jetzt nur noch neue Auslöser
nicht mehr an. Ein Durchlauf, der schon wartet, läuft weiter — schaltet am Ende aber nicht,
weil die Sperre kurz vor dem Ausschalten erneut geprüft wird.

### Warum der Kontext des Zustandswechsels geprüft wird

Der `manuell`-Trigger horcht auf dieselben Lichter, die die Automation selbst schaltet —
ohne Gegenmaßnahme würde sie sich beim eigenen Einschalten sofort neu starten und die
laufende Erfolgsprüfung abbrechen. Ein Zustandswechsel, den eine Automation oder ein Skript
verursacht hat, trägt eine `context.parent_id`; ein Druck auf den Wandschalter, ein Klick in
der App oder ein Sprachbefehl nicht. Geprüft wird deshalb:

```jinja
{{ trigger.id != 'manuell'
   or (v_manueller_trigger and trigger.to_state.context.parent_id is none) }}
```

Nebenwirkung: Schaltet eine *andere* Automation oder eine Szene das Licht ein, übernimmt
dieser Blueprint die Abschaltung nicht.

### Nachlaufzeit als `wait_template`, nicht als `delay`

Ein `delay` läuft stur ab. Die Wartebedingung dagegen endet sofort, wenn während der
Nachlaufzeit von Hand ausgeschaltet wird — dann bleibt der spätere Ausschaltbefehl samt
Erfolgsprüfung und möglicher Fehlermeldung aus. In beiden Wartebedingungen steht der
Lichter-Teil vorne, damit die Entitäten auch bei abgeschalteter Option als Listener
registriert werden.

### Entitäten statt Geräte

Nur mit einer Entity-ID lässt sich der Ist-Zustand zurücklesen — mit einer Geräte-ID, wie
sie der Automations-Editor anbietet, wäre die Schaltbestätigung unmöglich. Der *Push* nutzt
umgekehrt einen Geräte-Selektor, weil die Companion-App keine Notify-Entitäten anlegt.

### Helligkeit nur für ausgeschaltete Lichter

Gesetzt wird die Helligkeit ausschließlich beim Einschalten eines Lichts, das gerade aus
ist. Wer von Hand gedimmt hat, wird also nicht überstimmt — auch dann nicht, wenn eine neue
Bewegung den Durchlauf neu startet. Die Übergangszeit wird nur mitgesendet, wenn sie größer
`0` ist; manche Leuchtmittel quittieren ein unbekanntes Attribut sonst mit einem Fehler.

### `unavailable` gilt als Abweichung

Ein Aktor, der weder sauber `on` noch `off` meldet, wird angestoßen und nach `max_versuche`
als Fehlschlag gemeldet. Ein hängendes Gerät bleibt so nicht unbemerkt an.

### Kein Push im Normalbetrieb

Ein Bewegungslicht schaltet zu oft für Erfolgsmeldungen. Gemeldet wird nur, was nach allen
Versuchen nicht reagiert hat.

## Beispielkonfigurationen

### Flur mit zwei Meldern

| Feld | Wert |
|---|---|
| `bewegungsmelder` | `binary_sensor.bw_flur_1_occupancy`, `binary_sensor.bw_flur_2_occupancy` |
| `lichter` | `light.flur_decke` |
| `nachlaufzeit` | `45` s |
| `dunkel_modus` | `beides_oder` (Einstieg) → später `helligkeit` |
| `offset_sonnenuntergang` / `offset_sonnenaufgang` | `-00:10:00` / `00:30:00` |
| `helligkeitssensoren` | `sensor.bw_flur_1_illuminance` |
| `lux_schwelle` | aus dem Verlauf ablesen, Startwert `20` |
| `sicherheits_timeout` | `30` min |

### WC (Ersatz für die handgeschriebene Automation)

| Feld | Wert |
|---|---|
| `bewegungsmelder` | `binary_sensor.bw_wc_occupancy` |
| `lichter` | die **Entity-ID** des WC-Aktors statt der bisherigen Geräte-ID |
| `nachlaufzeit` | `30` s |
| `dunkel_modus` | `sonne` |
| `push_geraete_fehler` | eigenes Handy |

### Bad, nachts gedimmt

| Feld | Wert |
|---|---|
| `nachlaufzeit` | `180` s |
| `einschalt_helligkeit` | `100` % |
| `nachtmodus` | an, `nacht_start` `23:00:00`, `nacht_ende` `06:00:00` |
| `nacht_helligkeit` | `15` % |
| `uebergangszeit` | `2` s |
| `sperr_entitaeten` | `input_boolean.putzmodus`, `sperr_verhalten` `nur_ausschalten` |

## Die Lux-Schwelle finden

1. Sensor im Verlauf öffnen (Entwicklerwerkzeuge → Statistiken oder der Verlauf der
   Entität) und **zwei bis drei Tage** laufen lassen, ohne die Lux-Bedingung zu nutzen.
2. Im Verlauf die Stelle suchen, an der man selbst Licht machen würde — meist die
   Dämmerung. Der Wert dort ist die Untergrenze.
3. Prüfen, wie hoch der Wert an einem trüben Vormittag steigt. Liegt er über der
   Untergrenze, muss die Schwelle dazwischen liegen.
4. Schwelle setzen, `dunkel_modus` zunächst auf `beides_oder` — dann schaltet die Sonne
   weiterhin zuverlässig, während sich zeigt, ob die Schwelle passt.
5. Passt sie, auf `helligkeit` oder `beides_und` umstellen.

Anhaltswerte: heller Tag im Raum 200–1000 lx, bedeckter Tag am Fenster 50–200 lx, Dämmerung
5–50 lx, Nacht ohne Licht 0–2 lx. Der Sensor sitzt im Melder an der Decke und misst
Reflexion, nicht das Fensterlicht — die Werte fallen deshalb niedriger aus als erwartet.

## Bekannte Einschränkungen

- **Andere Automationen und Szenen** lösen den `manuell`-Trigger nicht aus (Kontextprüfung,
  siehe oben). Wer sein Licht per Szene einschaltet, bekommt keine automatische Abschaltung.
- **Ein hängender Melder** hält das Licht an, bis der Sicherheits-Timeout greift. Der steht
  ab Werk auf `0` (aus), damit die Automation niemandem das Licht ausmacht, der nur ruhig
  sitzt. In Fluren lohnt sich ein Wert von 15–60 Minuten.
- **Kein Neustart-Trigger.** Fällt HA aus, während das Licht brennt, läuft kein Durchlauf
  mehr, der es ausschaltet. Erst die nächste Bewegung setzt wieder auf. Ein
  `homeassistant`-`start`-Trigger würde das abdecken, kostet aber die Unterscheidung, ob
  das Licht bewusst von Hand an ist — bewusst zurückgestellt.
- **Ein Blueprint pro Lichtgruppe.** Melder und Licht gehören zusammen; wer zwei Räume in
  eine Automation packt, teilt sich Nachlaufzeit und Wartezeit.
- **Die Lux-Bedingung gilt als erfüllt**, wenn kein Sensor ausgewählt ist oder keiner eine
  Zahl liefert. Ein defekter Sensor blockiert also nicht das Licht — er macht die Prüfung
  wirkungslos.
- **Der Text der Sonnen-Offsets wird nicht geprüft.** Ein Tippfehler im Format `HH:MM:SS`
  lässt die Automation nicht laden.

## Validierung

Offline geprüft mit PyYAML und Jinja2 (`validate.py`, 193 Prüfungen): YAML geparst, alle
25 `!input` in beide Richtungen abgeglichen, 51 Templates gegen simulierte HA-Zustände
gerendert, dazu Wahrheitstabellen für Auslöser-Erkennung, Lux-Bedingung, die fünf
Dunkelheits-Modi, Nachtfenster (auch über Mitternacht), Sperren in allen drei Varianten,
Warte- und Timeout-Bedingungen, Zielaufteilung beim Dimmen sowie Ablaufsimulationen für
Sofort-Erfolg, Nachzügler im dritten Versuch, toten Aktor und `unavailable`.

Dazu ein Regressionstest für die Selbstauslösung: Er bildet die Reihenfolge aus
`async_trigger` nach und prüft mit Gegenprobe, dass der Torwächter im `conditions`-Block
liegt — als erste Aktion ginge der wartende Durchlauf verloren.

Nicht abgedeckt: Home Assistants eigene Config-Validierung (Selektor- und Trigger-Schema)
und das Laufzeitverhalten von `mode: restart` — dafür ist eine echte Instanz nötig.

---

[← zurück zur Übersicht](../../README.md)
