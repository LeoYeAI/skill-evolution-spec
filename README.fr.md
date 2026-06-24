# skill-evolution-spec

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![MyClaw.ai](https://img.shields.io/badge/Propulsé%20par-MyClaw.ai-6366f1)](https://myclaw.ai)
[![OpenClaw](https://img.shields.io/badge/Runtime-OpenClaw-0ea5e9)](https://myclaw.ai)
[![Status](https://img.shields.io/badge/Statut-Prêt%20pour%20la%20production-22c55e)]()
[![PRs Welcome](https://img.shields.io/badge/PRs-bienvenus-brightgreen.svg)](CONTRIBUTING.md)

**Donnez à votre agent une véritable auto-évolution — pas des astuces de prompt, de l'ingénierie.**

Une spécification complète pour un système d'évolution de compétences, validée sur OpenClaw et prête à être implémentée sur n'importe quel runtime d'agent.

---

**Langues :** [English](README.md) · [中文](README.zh-CN.md) · [日本語](README.ja.md) · [한국어](README.ko.md) · [Français](README.fr.md) · [Deutsch](README.de.md) · [Español](README.es.md) · [Русский](README.ru.md)

---

## Qu'est-ce que c'est ?

La plupart des agents IA sont des outils sans état. Ils oublient tout après chaque session. Cette spécification décrit un **système d'évolution de compétences** qui donne aux agents :

- **Mémoire de compétences persistante** — les procédures apprises survivent entre les sessions
- **Télémétrie d'utilisation** — l'agent sait quelles compétences il utilise réellement
- **Gouvernance du cycle de vie (Curator)** — archive automatiquement les compétences mortes, prévient la corruption
- **Chargement efficace en tokens** — injecte uniquement des résumés d'une ligne dans le contexte, charge le contenu complet à la demande

> « Le goulot d'étranglement n'est pas l'intelligence. C'est la conception de la discipline. » — §8

---

## Les Quatre Piliers

| Pilier | Rôle |
|--------|------|
| **Fichiers de compétences** | `SKILL.md` en texte brut sur disque — lisibles, versionnés, sauvegardables |
| **Outils R/W** | `skill_view`, `skill_manage`, `skills_list` — « apprendre » = écrire des fichiers |
| **Discipline du prompt système** | Règles strictes sur quand stocker, quand mettre à jour |
| **Curator** | Gouvernance en arrière-plan : déduplication, archivage, élagage |

---

## Curator — Gouvernance du Cycle de Vie

La partie que la plupart des implémentations sautent. Sans elle, la bibliothèque de compétences pourrit en 3 mois.

**Machine à états :**
```
active ──(stale_after_days)──> stale ──(archive_after_days)──> archived
```

Ne supprime jamais. Pire cas : `archived`. Toujours récupérable.

**Valeurs par défaut :**
```yaml
curator:
  enabled: true
  interval_hours: 168       # hebdomadaire
  min_idle_hours: 2
  stale_after_days: 30
  archive_after_days: 90
  backup:
    enabled: true
```

---

## Liste de contrôle d'implémentation

- [ ] 1. Format SKILL.md + schéma frontmatter (avec champ de provenance `created_by`)
- [ ] 2. Convention de répertoire + chargeur de démarrage (injecter description uniquement)
- [ ] 3. Cache d'instantané (manifest = mtime_ns + size)
- [ ] 4. Outils : `skill_view` / `skill_manage` / `skills_list`, écritures atomiques
- [ ] 5. Sidecar de télémétrie `.usage.json` + 3 types d'événements
- [ ] 6. Discipline du prompt système
- [ ] 7. Curator : déclencheur inactif + machine à états + valeurs par défaut + sauvegarde + jamais de suppression
- [ ] 8. Logique d'exemption pin/unpin
- [ ] 9. Verbes CLI : status / run / pause / resume / pin / unpin / archive / restore / prune / backup / rollback

---

## Spécification complète

Voir [SPEC.md](SPEC.md) pour tous les schémas de données, machines à états, interfaces d'outils et valeurs par défaut.

---

## Propulsé par MyClaw.ai

[![MyClaw.ai — Agents IA qui se souviennent](https://img.shields.io/badge/MyClaw.ai-Agents%20IA%20qui%20se%20souviennent-6366f1?style=for-the-badge)](https://myclaw.ai)

Cette spécification a été développée et validée dans le cadre de la plateforme [MyClaw.ai](https://myclaw.ai).

---

## Licence

MIT
