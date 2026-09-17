# 🔌🛡️ Zustandswächter mit Schaltbestätigung

[![Blueprint importieren](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FToutzn%2Fha-blueprints%2Fblob%2Fmain%2Fblueprints%2Fautomation%2Fzustandswaechter.yaml)

Braucht Home Assistant **2024.10** oder neuer.

## Zweck

Eine Entität dauerhaft in einem festen Zustand halten — `on` oder `off` — und nach jedem
Eingriff nachprüfen, ob es geklappt hat.

Der Anwendungsfall, aus dem der Blueprint entstanden ist: eine Steckdose, an der ein
kritischer Verbraucher hängt — Kühlschrank, Gefriertruhe, Server, Netzwerktechnik. Fällt
der Strom aus und kommt wieder, stellen nicht alle Aktoren ihren letzten Zustand wieder
her; manche kommen grundsätzlich aus. Der Wächter sorgt dafür, dass sie es nicht bleiben.

Die Gegenrichtung ist genauso abgedeckt: ein Gerät, das dauerhaft **aus** bleiben soll und
weder durch einen Spannungswiederkehr-Standard noch durch einen Fehlgriff wieder anlaufen
darf.

> **Erst am Gerät nachsehen.** Die meisten Aktoren können das Verhalten nach
> Spannungswiederkehr selbst festlegen — `power_on_behavior: on` in zigbee2mqtt,
> „Restore/Always ON" bei Shelly, `PowerOnState 1` bei Tasmota. Das wirkt auch dann, wenn
> Home Assistant selbst noch bootet oder gar nicht läuft, und ist deshalb die erste
> Verteidigungslinie. Dieser Blueprint ist das Netz darunter, nicht der Ersatz dafür.

### Warum ein Zustandswechsel als Auslöser nicht reicht

Der eigentliche Punkt sind die Auslöser, nicht das Schalten. War Home Assistant während
des Stromausfalls selbst aus — was bei einem Hausausfall der Normalfall ist —, dann gab es
aus seiner Sicht nie einen Zustandswechsel: Es fährt hoch und sieht die Steckdose einfach
als „aus". Eine Automation, die nur auf `to: "off"` hört, wird nie ausgelöst.

Der Blueprint hört deshalb auf vier verschiedene Dinge, und jedes deckt eine andere Lücke:

| Auslöser | deckt ab |
|---|---|
| Abweichung vom Soll-Zustand | den Normalfall: jemand oder etwas hat geschaltet |
| Rückkehr aus `unavailable` | Stromausfall am Gerät — genau das ist seine Signatur |
| Neustart von Home Assistant | Stromausfall, der HA selbst getroffen hat |
| zyklische Nachprüfung | alles andere: verlorener Funkbefehl, hängende Integration, verpasster Trigger |

Das zyklische Nachprüfen ist das eigentliche Sicherheitsnetz; die übrigen Auslöser sorgen
nur dafür, dass es schnell geht.

## Eingaben (18)

### Überwachung

| Feld | Pflicht | Default | Beschreibung |
|---|---|---|---|
| `ziele` | ja | – | Zu überwachende **Entitäten** (`switch`, `light`, `input_boolean`, `fan`, `humidifier`, `siren`) |
| `soll_zustand` | nein | `on` | `on` = dauerhaft ein · `off` = dauerhaft aus |

### Handbetrieb und Ausnahmen

| Feld | Pflicht | Default | Beschreibung |
|---|---|---|---|
| `pause_entitaeten` | nein | `[]` | Solange eine davon `on` ist, hält sich der Wächter komplett raus |
| `toleranz` | nein | `0` min | So lange darf von Hand abgewichen werden, bevor korrigiert wird; `0` = sofort |

### Schalten und Nachprüfen

| Feld | Pflicht | Default | Beschreibung |
|---|---|---|---|
| `anlauf_verzoegerung` | nein | `5` s | Wartezeit nach Rückkehr aus `unavailable`, bevor geschaltet wird |
| `pruefung_verzoegerung` | nein | `10` s | Wartezeit zwischen Schaltbefehl und Nachprüfung |
| `max_versuche` | nein | `5` | Maximale Schaltversuche je Eingriff |
| `nachpruefung` | nein | `true` | Zyklische Nachprüfung aktiv |
| `sync_intervall` | nein | `/5` | Intervall der Nachprüfung (1 / 5 / 15 / 30 Minuten) |

### Erreichbarkeit

| Feld | Pflicht | Default | Beschreibung |
|---|---|---|---|
| `unerreichbar_melden` | nein | `true` | Eigene Warnung, wenn ein Ziel gar nicht antwortet |
| `unerreichbar_nach` | nein | `10` min | Karenzzeit, bevor `unavailable` als Problem gilt |

### Push-Benachrichtigungen

| Feld | Pflicht | Default | Beschreibung |
|---|---|---|---|
| `push_geraete_erfolg` | nein | `[]` | Geräte für die Erfolgsmeldung |
| `push_geraete_fehler` | nein | `[]` | Geräte für die Fehlermeldung |
| `push_titel` | nein | `Zustandswächter` | Überschrift der Meldung |
| `fehler_melden` | nein | `ereignisse` | `ereignisse` = Nachprüf-Ticks schweigen · `immer` = jeder Durchlauf meldet |
| `push_kritisch` | nein | `false` | Fehler-Push als kritischer Hinweis (durchbricht „Nicht stören") |

### Meldungen

| Feld | Pflicht | Default | Beschreibung |
|---|---|---|---|
| `aktion_bei_erfolg` | nein | `logbook.log` | Zusätzliche Aktion nach erfolgreichem Eingriff |
| `aktion_bei_fehler` | nein | `persistent_notification.create` | Zusätzliche Aktion nach Fehlschlag |

## Ablauf

1. **Auslöser bewerten.** Je nach Einstellung werden Auslöser verworfen: der Sofort-Trigger
   bei gesetzter Toleranz, der verzögerte Trigger ohne Toleranz, der Nachprüf-Tick bei
   abgeschalteter Nachprüfung, die Erreichbarkeitswarnung bei abgeschalteter Warnung.
2. **Lage erheben.** Pause-Schalter, Soll-Zustand, Auslöser im Klartext. Ist ein
   Pause-Schalter `on` — oder gar nicht auswertbar —, endet der Durchlauf hier.
3. **Fällige Ziele bestimmen.** Wer weicht ab, und weicht er lange genug ab? Die
   Toleranzzeit wird genau einmal angewendet; danach steht die Liste fest und wird bis zum
   Erfolg bearbeitet. Ziele mit Zustand `unavailable` kommen nicht in die Liste — sie lassen
   sich nicht schalten — sondern werden getrennt gezählt.
4. **Abbruch, wenn es nichts zu tun gibt.** Der übliche Fall; kostet praktisch keine Leistung.
5. **Anlaufverzögerung**, aber nur auf dem Rückkehr- und Neustart-Weg.
6. **Schalten mit Wiederholung.** `homeassistant.turn_on`/`turn_off` auf die noch
   abweichenden Ziele, warten, nachprüfen, wiederholen — bis alle im Soll sind oder
   `max_versuche` erreicht ist.
7. **Ergebnis auswerten** und melden: Push an die gewählten Geräte, dazu die frei
   definierbare Zusatzaktion.

## Auslöser im Einzelnen

| ID | Trigger | Bemerkung |
|---|---|---|
| `rueckkehr` | `from: unavailable` → `on`/`off` | umgeht die Toleranzzeit |
| `abweichung` | Zustandswechsel der Ziele | nur wirksam bei Toleranz `0` |
| `abweichung_verzoegert` | derselbe Wechsel mit `for: <Toleranz>` | nur wirksam bei Toleranz > 0 |
| `weg` | `to: unavailable` mit `for: <Karenzzeit>` | meldet nur, schaltet nicht |
| `pause` | Pause-Schalter wechselt | damit das Ende der Pause sofort nachgezogen wird |
| `neustart` | `homeassistant: start` | umgeht die Toleranzzeit |
| `sync` | `time_pattern` | das eigentliche Sicherheitsnetz |

## Designentscheidungen

### Zwei Trigger für dieselbe Abweichung

Der naheliegende Weg wäre *ein* Trigger mit variablem `for:`. Bei Toleranz `0` hinge das
Verhalten dann an einer Null-Wartezeit — formal gültig, aber eine Abhängigkeit von einem
Randfall im Trigger-Schema. Stattdessen gibt es zwei Trigger, von denen immer genau einer
beachtet wird. Der jeweils andere feuert zwar mit, wird aber gleich in der ersten Aktion
verworfen.

### Toleranz einmal anwenden, nicht in der Schleife

Ob ein Ziel „lange genug" abweicht, wird einmal entschieden und in `faellig` festgehalten.
Die Wiederholschleife arbeitet danach nur noch auf dieser Liste. Andernfalls würde die
Schleife die Toleranzprüfung bei jedem Durchgang neu rechnen und könnte ein Ziel mitten im
Korrigieren wieder aus den Augen verlieren — und der `last_changed`-Zeitstempel verschiebt
sich mit jedem Schaltversuch ohnehin.

### Rückkehr und Neustart umgehen die Toleranz

Die Toleranzzeit ist für Menschen gedacht: kurz ausstecken, umstöpseln, abtauen. Beim
Stromausfall war niemand am Werk, und ein Kühlschrank soll nicht eine halbe Stunde warten,
nur weil jemand die Toleranz für den Handbetrieb gesetzt hat. Die Unterscheidung fällt
entlang des Zustandsverlaufs: `unavailable → off` ist ein Ausfall, `on → off` ein Eingriff.

### Pause-Schalter ohne Zustand pausiert ebenfalls

Ist der Pause-Schalter `unavailable` oder `unknown` — direkt nach dem HA-Start der
Normalfall —, hält sich der Wächter raus. „Weiß ich nicht" als „nicht pausiert" zu deuten
wäre die riskantere Auslegung: Der Wächter würde gegen einen laufenden Handbetrieb
anschalten, von dem er nichts weiß. Wer keinen Handbetrieb braucht, lässt das Feld leer;
dann greift die Prüfung nie.

### `unavailable` ist kein Fehlschlag, sondern eine eigene Meldung

Ein Gerät mit Zustand `unavailable` lässt sich nicht schalten; es wiederholt anzufunken
kostet nur Zeit. Es wird deshalb getrennt gezählt und getrennt gemeldet. Bei einem
kritischen Verbraucher ist das ohnehin die wichtigere Information: Entweder der Aktor ist
defekt, oder es liegt dort gerade überhaupt keine Spannung an — was bedeutet, dass der
Kühlschrank *jetzt* aus ist, unabhängig davon, was der Schaltbefehl gemacht hätte.

Die Karenzzeit `unerreichbar_nach` ist dabei nicht nur Komfort: Ohne sie meldet jeder
Home-Assistant-Start einen Fehlschlag, weil direkt nach dem Start alle Entitäten kurz
`unknown` sind.

### `mode: queued`

Die Auslöser können dicht aufeinander folgen — ein Stromausfall lässt Rückkehr, Abweichung
und Neustart fast gleichzeitig feuern. Mit `restart` würde ein laufender Wiederholzyklus
abgeräumt, mit `single` gingen die Folgeläufe verloren. `queued` arbeitet sie nacheinander
ab; die nachrückenden Läufe finden dann nichts mehr zu tun und beenden sich sofort.

Deshalb steht auch jede Entscheidung in den **Aktionen**, nicht in den `variables:` auf
oberster Ebene: Die werden zum *Auslösezeitpunkt* gerendert, nicht beim Ausführen. Ein Lauf,
der in der Warteschlange gewartet hat, würde sonst mit dem Zustand von vorher rechnen.

### Warum das Verwerfen hier in den Aktionen steht und nicht in `conditions:`

Der sonst geltenden Faustregel (*verwerfen → `conditions:`*) liegt zugrunde, dass
`mode: restart` einen laufenden Durchlauf bereits abgeräumt hat, bevor die erste Aktion
ablehnen kann. Bei `mode: queued` passiert das nicht: Ein neuer Lauf reiht sich ein, statt
den alten zu killen. Die Prüfung darf deshalb in den Aktionen stehen — und muss es sogar,
weil sie die `v_*`-Variablen braucht.

## Beispielkonfigurationen

### Kühlschrank — muss immer laufen

```yaml
ziele: [switch.ps_kuehlschrank]
soll_zustand: "on"
toleranz: 0
anlauf_verzoegerung: 5
nachpruefung: true
sync_intervall: "/5"
unerreichbar_melden: true
unerreichbar_nach: 10
push_geraete_erfolg: [<Handy>]
push_geraete_fehler: [<Handy>]
push_titel: Kühlschrank
push_kritisch: true
```

Die Erfolgsmeldung ist hier keine Spielerei: Sie ist die Nachricht „hier war gerade ein
Stromausfall, ich habe es gerichtet".

### Serverschrank — mit Wartungsschalter

```yaml
ziele: [switch.ps_serverschrank, switch.ps_switch_keller]
soll_zustand: "on"
pause_entitaeten: [input_boolean.wartung_serverschrank]
toleranz: 0
push_geraete_fehler: [<Handy>]
push_titel: EDV
```

Vor dem Umbau `input_boolean.wartung_serverschrank` einschalten — dann schweigt der Wächter
komplett. Beim Ausschalten des Helfers zieht er den Soll-Zustand sofort nach.

### Gefriertruhe — Handgriffe von bis zu 15 Minuten erlaubt

```yaml
ziele: [switch.ps_gefriertruhe]
soll_zustand: "on"
toleranz: 15
push_kritisch: true
```

Kurzes Ausschalten von Hand bleibt möglich; nach 15 Minuten schaltet der Wächter wieder ein.
Ein Stromausfall wird davon unberührt sofort korrigiert.

### Gerät, das nicht von selbst anlaufen darf

```yaml
ziele: [switch.heizluefter_werkstatt]
soll_zustand: "off"
toleranz: 0
push_geraete_fehler: [<Handy>]
```

Achtung: Ohne Pause-Schalter oder Toleranz lässt sich das Gerät dann auch von Hand nicht
mehr einschalten — der Wächter schaltet sofort zurück. Für gelegentlichen Handbetrieb einen
Pause-Schalter setzen.

## Bekannte Einschränkungen

- **Der Wächter setzt sich durch.** Ohne Pause-Schalter und ohne Toleranz lässt sich das
  Ziel nicht von Hand in den anderen Zustand bringen. Das ist der Sinn der Sache, führt aber
  zu Verwunderung, wenn man es vergisst — auch beim Schalten über die HA-Oberfläche.
- **Der Neustart umgeht die Toleranz.** Wer von Hand ausgeschaltet hat und dann Home
  Assistant neu startet, findet das Ziel wieder im Soll-Zustand. Für planbaren Handbetrieb
  ist der Pause-Schalter das richtige Mittel, nicht die Toleranz.
- **Nur `on`/`off`.** Zustände jenseits davon (`playing`, `heat`, Helligkeitsstufen) kann der
  Wächter nicht halten; `media_player` und Klimageräte sind deshalb nicht im Selektor.
- **Mehrere Ziele teilen sich alles.** Soll-Zustand, Toleranz, Meldungen und Push-Tag gelten
  für die ganze Automation. Ziele mit unterschiedlichen Regeln gehören in getrennte
  Automationen — schon damit die Meldungen auseinanderzuhalten sind.
- **Ein Aktor, der `off` meldet, obwohl er `on` ist**, lässt sich nicht erkennen. Der
  Blueprint liest den Zustand zurück, den die Integration liefert; er misst nicht.
- **Die Rückkehr-Erkennung hängt daran, dass die Integration `unavailable` setzt.** Manche
  Integrationen lassen den letzten bekannten Zustand stehen, wenn ein Gerät verschwindet.
  Dann greift nur die zyklische Nachprüfung — die aber greift.

## Validierung

Offline geprüft mit PyYAML und Jinja2, ohne laufende Home-Assistant-Instanz:

- alle `!input`-Verweise beidseitig gegen die Deklaration abgeglichen (18/18),
- alle 31 Templates gegen simulierte Zustände gerendert,
- 24 Ablaufszenarien durchgespielt: Stromausfall mit und ohne Toleranz, Handbetrieb
  innerhalb und außerhalb der Toleranz, beide Toleranz-Trigger in beiden Einstellungen,
  Pause-Schalter an / aus / `unavailable`, Nachprüfung an und aus, Nachzügler im dritten
  Versuch, toter Aktor, frisch und lange `unavailable`, HA-Start mit `unknown`-Entitäten,
  Soll-Zustand `off`, manuelles Ausführen im Editor, kritischer Push,
- Trigger-IDs gegen das Verwerfungs-Template abgeglichen, damit keine ID stumm durchrutscht,
- jeder State-Trigger auf explizites `to:` geprüft.

Nicht abgedeckt: Home Assistants eigene Config-Validierung (Selektor- und Trigger-Schema)
und das Laufzeitverhalten der Warteschlange — dafür ist eine echte Instanz nötig.

---

[← zurück zur Übersicht](../../README.md)
