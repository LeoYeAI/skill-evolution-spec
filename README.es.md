# skill-evolution-spec

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![MyClaw.ai](https://img.shields.io/badge/Desarrollado%20con-MyClaw.ai-6366f1)](https://myclaw.ai)
[![OpenClaw](https://img.shields.io/badge/Runtime-OpenClaw-0ea5e9)](https://myclaw.ai)
[![Status](https://img.shields.io/badge/Estado-Listo%20para%20producción-22c55e)]()
[![PRs Welcome](https://img.shields.io/badge/PRs-bienvenidos-brightgreen.svg)](CONTRIBUTING.md)

**Dale a tu agente una verdadera auto-evolución — no trucos de prompt, ingeniería.**

Una especificación completa para un sistema de evolución de habilidades, validada en OpenClaw y lista para implementar en cualquier runtime de agente.

---

**Idiomas:** [English](README.md) · [中文](README.zh-CN.md) · [日本語](README.ja.md) · [한국어](README.ko.md) · [Français](README.fr.md) · [Deutsch](README.de.md) · [Español](README.es.md) · [Русский](README.ru.md)

---

## ¿Qué es esto?

La mayoría de los agentes de IA son herramientas sin estado. Olvidan todo después de cada sesión. Esta especificación describe un **sistema de evolución de habilidades** que da a los agentes:

- **Memoria de habilidades persistente** — los procedimientos aprendidos sobreviven entre sesiones
- **Telemetría de uso** — el agente sabe qué habilidades usa realmente
- **Gobernanza del ciclo de vida (Curator)** — archiva automáticamente habilidades muertas, previene la corrupción
- **Carga eficiente en tokens** — solo inyecta resúmenes de una línea en el contexto, carga contenido completo bajo demanda

> «El cuello de botella no es la inteligencia. Es el diseño de la disciplina.» — §8

---

## Los Cuatro Pilares

| Pilar | Función |
|-------|---------|
| **Archivos de habilidades** | `SKILL.md` en texto plano en disco — legibles, versionables, respaldables |
| **Herramientas R/W** | `skill_view`, `skill_manage`, `skills_list` — «aprender» = escribir archivos |
| **Disciplina del prompt del sistema** | Reglas fijas sobre cuándo guardar, cuándo actualizar |
| **Curator** | Gobernanza en segundo plano: deduplicación, archivado, poda |

---

## Curator — Gobernanza del Ciclo de Vida

La parte que la mayoría de las implementaciones omite. Sin él, la biblioteca se pudre en 3 meses.

**Máquina de estados:**
```
active ──(stale_after_days)──> stale ──(archive_after_days)──> archived
```

Nunca elimina. Peor caso: `archived`. Siempre recuperable.

**Valores predeterminados:**
```yaml
curator:
  enabled: true
  interval_hours: 168       # semanal
  min_idle_hours: 2
  stale_after_days: 30
  archive_after_days: 90
  backup:
    enabled: true
```

---

## Lista de implementación

- [ ] 1. Formato SKILL.md + esquema frontmatter (con campo de procedencia `created_by`)
- [ ] 2. Convención de directorio + cargador de inicio (solo inyectar description)
- [ ] 3. Caché de instantáneas (manifest = mtime_ns + size)
- [ ] 4. Herramientas: `skill_view` / `skill_manage` / `skills_list`, escrituras atómicas
- [ ] 5. Sidecar de telemetría `.usage.json` + 3 tipos de eventos
- [ ] 6. Disciplina del prompt del sistema
- [ ] 7. Curator: disparador inactivo + máquina de estados + valores predeterminados + copia de seguridad + sin eliminación
- [ ] 8. Lógica de exención pin/unpin
- [ ] 9. Verbos CLI: status / run / pause / resume / pin / unpin / archive / restore / prune / backup / rollback

---

## Especificación completa

Ver [SPEC.md](SPEC.md) para todos los esquemas de datos, máquinas de estados, interfaces de herramientas y valores predeterminados.

---

## Desarrollado con MyClaw.ai

[![MyClaw.ai — Agentes de IA que recuerdan](https://img.shields.io/badge/MyClaw.ai-Agentes%20de%20IA%20que%20recuerdan-6366f1?style=for-the-badge)](https://myclaw.ai)

Esta especificación fue desarrollada y validada como parte de la plataforma [MyClaw.ai](https://myclaw.ai).

---

## Licencia

MIT
