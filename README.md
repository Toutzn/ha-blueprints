# Home Assistant Blueprints

Sammlung eigener Automations-Blueprints für [Home Assistant](https://www.home-assistant.io/).

## Übersicht

| Blueprint | Domain | Kurzbeschreibung | Min. HA |
|---|---|---|---|
| [Zeitschaltuhr mit Schaltbestätigung](#zeitschaltuhr-mit-schaltbestätigung) | `automation` | Zeitschaltuhr, die nachprüft, ob der Schaltbefehl angekommen ist | 2024.10 |

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

Schaltet beliebige Entitäten nach einem `schedule.*`-Helfer ein und aus – und **prüft
nach, ob es wirklich geklappt hat**. Verlorene Funkbefehle, kurz nicht erreichbare
Aktoren und verpasste Trigger nach einem Neustart fallen damit nicht mehr durchs Raster.

**Datei:** [`blueprints/automation/zeitschaltuhr_mit_bestaetigung.yaml`](blueprints/automation/zeitschaltuhr_mit_bestaetigung.yaml)
**Benötigt:** Home Assistant 2024.10 oder neuer

### Warum

Eine gewöhnliche Zeitschaltuhr-Automation feuert den Schaltbefehl ab und hofft das Beste.
Kommt der Befehl nicht an – Funkaussetzer, Aktor gerade `unavailable`, Home Assistant hat
im Schaltmoment neu gestartet – bleibt die Last unbemerkt an. Dieser Blueprint liest den
Ist-Zustand nach dem Schalten zurück, wiederholt gezielt für die Entitäten, die nicht
reagiert haben, und meldet erst dann Erfolg oder Fehlschlag.

### Ablauf

```
Zeitschaltuhr aktiv?  ──nein──▶  Handbetrieb: nichts tun
        │                        oder Zwangs-Aus: ausschalten
        ja
        ▼
Freigaben erfüllt?    ──nein──▶  (dito)
        │
        ja
        ▼
Zeitplan sagt on/off  ──▶  Soll-Zustand
        ▼
Ist == Soll?          ──ja───▶  fertig, keine Meldung
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
| **Zeitplan** | ja | – | `schedule.*`-Helfer. `on` = Ziele an, `off` = Ziele aus. Wochentage und mehrere Blöcke pro Tag konfigurierst du im Helfer selbst. |
| **Zeitschaltuhr aktiv** | ja | – | `input_boolean` als Hauptschalter |
| **Zusätzliche Freigaben** | nein | – | Alle müssen `on` sein, z. B. `input_boolean.poolsaison` |
| **Verhalten bei deaktivierter Zeitschaltuhr** | nein | Handbetrieb | *Handbetrieb* = Automatik hält sich raus · *Zwangs-Aus* = Ziele werden ausgeschaltet |
| **Zu schaltende Entitäten** | ja | – | `switch`, `light`, `input_boolean`, `fan`, `media_player` |
| **Wartezeit vor der Nachprüfung** | nein | 10 s | Bei Funk-Aktoren großzügiger wählen |
| **Maximale Schaltversuche** | nein | 5 | Wiederholt nur für die Entitäten, die noch nicht reagiert haben |
| **Zyklische Nachprüfung aktiv** | nein | ein | Korrigiert Abweichungen laufend |
| **Intervall der Nachprüfung** | nein | 5 min | `/1`, `/5`, `/15` oder `/30` |
| **Aktion bei Erfolg** | nein | Logbuch-Eintrag | Frei definierbar |
| **Aktion bei Fehlschlag** | nein | Persistente Benachrichtigung | Frei definierbar |

### Variablen für eigene Meldungsaktionen

| Variable | Inhalt |
|---|---|
| `soll_zustand` | `on` oder `off` |
| `ziel_liste` | Alle konfigurierten Ziel-Entitäten |
| `restliche` | Entitäten, die den Soll-Zustand **nicht** erreicht haben |
| `versuche` | Anzahl der Schaltversuche |

Beispiel für eine Push-Benachrichtigung als Fehler-Aktion:

```yaml
action: notify.mobile_app_dein_handy
data:
  title: Zeitschaltuhr – Schaltfehler
  message: >-
    Konnte nach {{ versuche }} Versuch(en) nicht auf '{{ soll_zustand }}'
    schalten: {{ restliche | join(', ') }}
```

### Trigger

| ID | Trigger | Zweck |
|---|---|---|
| `zeitplan` | `state` auf den Zeitplan, `to: "on"` / `to: "off"` | Regulärer Schaltzeitpunkt |
| `freigabe` | `state` auf Aktiv-Schalter und Freigaben, `to: ["on", "off"]` | Sofortige Neubewertung |
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

### Bekannte Einschränkungen

- Solange die Zeitschaltuhr aktiv und die zyklische Nachprüfung eingeschaltet ist, wird
  manuelles Schalten innerhalb des Zeitfensters wieder korrigiert. Für Handbetrieb den
  Aktiv-Schalter ausschalten oder die Nachprüfung deaktivieren.
- `unavailable`-Ziele gelten als Abweichung, werden bis zum Versuchslimit angestoßen und
  danach als Fehler gemeldet. Das ist beabsichtigt.
- Die zusätzlichen Freigaben werden mit Default `[]` als `entity_id` in einem State-Trigger
  verwendet. Eine leere Liste registriert einfach keinen Listener. Sollte eine HA-Version das
  beim Speichern beanstanden, den Trigger mit `id: freigabe` auf die Freigaben entfernen – die
  zyklische Nachprüfung fängt Änderungen dann verzögert ab.

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
