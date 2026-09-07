# Home Assistant Blueprints

Sammlung eigener Automations-Blueprints für [Home Assistant](https://www.home-assistant.io/).

## Übersicht

| Blueprint | Was er tut | Doku | Import |
|---|---|---|---|
| **🗓️⏰ Zeitschaltuhr mit Schaltbestätigung** | Schaltet Entitäten nach frei wählbaren Zeitquellen und prüft nach, ob der Befehl angekommen ist | [Details](blueprints/automation/zeitschaltuhr_mit_bestaetigung.md) · [YAML](blueprints/automation/zeitschaltuhr_mit_bestaetigung.yaml) | [![Blueprint importieren](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FToutzn%2Fha-blueprints%2Fblob%2Fmain%2Fblueprints%2Fautomation%2Fzeitschaltuhr_mit_bestaetigung.yaml) |
| **🔌📉 Standby-Abschaltung mit Betriebserkennung** | Schaltet einen Verbraucher ab, wenn seine Leistungsaufnahme lange genug im Standby-Bereich liegt | [Details](blueprints/automation/standby_abschaltung.md) · [YAML](blueprints/automation/standby_abschaltung.yaml) | [![Blueprint importieren](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FToutzn%2Fha-blueprints%2Fblob%2Fmain%2Fblueprints%2Fautomation%2Fstandby_abschaltung.yaml) |

Alle Blueprints sind Automations-Blueprints und brauchen Home Assistant **2024.10** oder
neuer. Die vollständige Dokumentation steht jeweils als `.md` neben der YAML-Datei –
Einstellungen, Ablauf, Designentscheidungen, Beispielkonfigurationen und bekannte
Einschränkungen.

## Aufbau

```
README.md                      # dieser Index
blueprints/
├── automation/                # Automations-Blueprints
│   ├── <name>.yaml            # der Blueprint
│   └── <name>.md              # seine Dokumentation
├── script/                    # Script-Blueprints
└── template/                  # Template-Blueprints
```

Das Layout spiegelt Home Assistants eigenes `/config/blueprints/`-Verzeichnis. Doku und
Blueprint liegen bewusst nebeneinander: ein `git mv` verschiebt beides, und wer den
Ordner öffnet, sieht sofort, was dokumentiert ist.

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
3. Dokumentation als `blueprints/<domain>/<name>.md` daneben anlegen: Titel = der
   `name` aus dem `blueprint:`-Block, darunter das Import-Badge, dann Zweck,
   Eingabentabelle, Ablauf, Trigger, Designentscheidungen, Beispielkonfigurationen
   und bekannte Einschränkungen. Am Ende ein Rücklink auf `../../README.md`.
4. Zeile in die Übersichtstabelle oben aufnehmen – mit Link auf die `.md`, die
   YAML-Datei und dem Import-Badge. Die README bleibt reiner Index; Details gehören
   in die Doku-Datei.
5. Import-Badge einfügen:
   ```markdown
   [![Blueprint importieren](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=<URL-encodierte Blob-URL>)
   ```

## Lizenz

MIT – siehe [LICENSE](LICENSE).
