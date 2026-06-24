# skill-evolution-spec

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![MyClaw.ai](https://img.shields.io/badge/Powered%20by-MyClaw.ai-6366f1)](https://myclaw.ai)
[![OpenClaw](https://img.shields.io/badge/Laufzeit-OpenClaw-0ea5e9)](https://myclaw.ai)
[![Status](https://img.shields.io/badge/Status-Produktionsreif-22c55e)]()
[![PRs Welcome](https://img.shields.io/badge/PRs-willkommen-brightgreen.svg)](CONTRIBUTING.md)

**Gib deinem Agent echte Selbstentwicklung — kein Prompt-Trick, sondern Engineering.**

Eine vollständige Spezifikation für ein Skill-Evolutionssystem, validiert auf OpenClaw und bereit für jede Agent-Laufzeit.

---

**Sprachen:** [English](README.md) · [中文](README.zh-CN.md) · [日本語](README.ja.md) · [한국어](README.ko.md) · [Français](README.fr.md) · [Deutsch](README.de.md) · [Español](README.es.md) · [Русский](README.ru.md)

---

## Was ist das?

Die meisten KI-Agenten sind zustandslose Werkzeuge. Nach jeder Sitzung ist alles vergessen. Diese Spezifikation beschreibt ein **Skill-Evolutionssystem**, das Agenten folgendes gibt:

- **Persistenter Skill-Speicher** — erlernte Verfahren überleben sitzungsübergreifend
- **Nutzungstelemetrie** — der Agent weiß, welche Skills er tatsächlich nutzt
- **Lebenszyklus-Governance (Curator)** — archiviert tote Skills automatisch, verhindert Bibliotheksfäulnis
- **Token-effizientes Laden** — injiziert nur einzeilige Zusammenfassungen in den Kontext, lädt Vollinhalt bei Bedarf

> „Der Engpass ist nicht Intelligenz. Es ist Disziplin-Design." — §8

---

## Die Vier Säulen

| Säule | Funktion |
|-------|----------|
| **Skill-Dateien** | Klartext-`SKILL.md` auf Disk — lesbar, versionierbar, sicherbar |
| **R/W-Werkzeuge** | `skill_view`, `skill_manage`, `skills_list` — „Lernen" = Dateien schreiben |
| **System-Prompt-Disziplin** | Feste Regeln wann speichern, wann aktualisieren |
| **Curator** | Hintergrund-Governance: Deduplizierung, Archivierung, Bereinigung |

---

## Curator — Lebenszyklus-Governance

Der Teil, den die meisten Implementierungen überspringen. Ohne ihn fault die Skill-Bibliothek in 3 Monaten.

**Zustandsautomat:**
```
active ──(stale_after_days)──> stale ──(archive_after_days)──> archived
```

Löscht niemals. Schlimmster Fall: `archived`. Immer wiederherstellbar.

**Standardwerte:**
```yaml
curator:
  enabled: true
  interval_hours: 168       # wöchentlich
  min_idle_hours: 2
  stale_after_days: 30
  archive_after_days: 90
  backup:
    enabled: true
```

---

## Implementierungscheckliste

- [ ] 1. SKILL.md-Format + Frontmatter-Schema (mit `created_by`-Herkunftsfeld)
- [ ] 2. Verzeichniskonvention + Startlader (nur description injizieren)
- [ ] 3. Snapshot-Cache (manifest = mtime_ns + size)
- [ ] 4. Werkzeuge: `skill_view` / `skill_manage` / `skills_list`, alle atomare Schreibvorgänge
- [ ] 5. Telemetrie-Sidecar `.usage.json` + 3 Ereignistypen
- [ ] 6. System-Prompt-Disziplin
- [ ] 7. Curator: Leerlauf-Auslöser + Zustandsautomat + Standardwerte + Backup + kein Löschen
- [ ] 8. Pin/Unpin-Ausnahmelogik
- [ ] 9. CLI-Verben: status / run / pause / resume / pin / unpin / archive / restore / prune / backup / rollback

---

## Vollständige Spezifikation

Siehe [SPEC.md](SPEC.md) für alle Datenschemata, Zustandsautomaten, Werkzeugschnittstellen und Standardwerte.

---

## Powered by MyClaw.ai

[![MyClaw.ai — KI-Agenten mit Gedächtnis](https://img.shields.io/badge/MyClaw.ai-KI--Agenten%20mit%20Ged%C3%A4chtnis-6366f1?style=for-the-badge)](https://myclaw.ai)

Diese Spezifikation wurde als Teil der [MyClaw.ai](https://myclaw.ai)-Plattform entwickelt und validiert.

---

## Lizenz

MIT
