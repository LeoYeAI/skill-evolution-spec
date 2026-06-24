# skill-evolution-spec

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![MyClaw.ai](https://img.shields.io/badge/Powered%20by-MyClaw.ai-6366f1)](https://myclaw.ai)
[![OpenClaw](https://img.shields.io/badge/런타임-OpenClaw-0ea5e9)](https://myclaw.ai)
[![Status](https://img.shields.io/badge/상태-프로덕션_준비-22c55e)]()
[![PRs Welcome](https://img.shields.io/badge/PR-환영-brightgreen.svg)](CONTRIBUTING.md)

**Agent에 진정한 자기진화 능력을 부여하라 — 프롬프트 트릭이 아닌, 엔지니어링으로.**

OpenClaw에서 검증된 스킬 진화 시스템의 완전한 명세서. 모든 Agent 런타임에 이식 가능.

---

**언어：** [English](README.md) · [中文](README.zh-CN.md) · [日本語](README.ja.md) · [한국어](README.ko.md) · [Français](README.fr.md) · [Deutsch](README.de.md) · [Español](README.es.md) · [Русский](README.ru.md)

---

## 이것은 무엇인가?

대부분의 AI Agent는 무상태(stateless) 도구입니다. 세션이 끝나면 모든 기억이 사라집니다. 이 명세서는 Agent에게 다음 능력을 부여하는 **스킬 진화 시스템**을 정의합니다:

- **지속적인 스킬 메모리** — 학습한 절차가 세션을 넘어 유지됨
- **사용 텔레메트리** — Agent가 실제로 어떤 스킬을 사용했는지 파악
- **생명주기 거버넌스(Curator)** — 죽은 스킬 자동 아카이브, 스킬 라이브러리 부패 방지
- **토큰 효율적 로딩** — 한 줄 요약만 컨텍스트에 주입, 필요 시 전체 내용 로드

> "병목은 지능이 아니라 규율 설계다." — 명세서 §8

---

## 네 가지 기둥

| 기둥 | 역할 |
|------|------|
| **스킬 파일** | 디스크의 플레인텍스트 `SKILL.md` — 읽기/쓰기, 버전 관리, 백업 가능 |
| **읽기/쓰기 도구** | `skill_view`, `skill_manage`, `skills_list` — "학습" = 파일 쓰기 |
| **시스템 프롬프트 규율** | 언제 저장하고 언제 업데이트할지 하드코딩 — Agent가 자율적으로 스킬 유지 |
| **Curator** | 백그라운드 거버넌스: 중복 제거, 아카이브, 정리 |

---

## Curator — 생명주기 거버넌스

대부분의 구현에서 생략되는 부분입니다. 생략하면 스킬 라이브러리는 3개월 만에 쓰레기 더미가 됩니다.

**상태 기계:**
```
active ──(stale_after_days)──> stale ──(archive_after_days)──> archived
```

삭제하지 않습니다. 최악의 경우 `archived`. 항상 복원 가능.

**기본값:**
```yaml
curator:
  enabled: true
  interval_hours: 168       # 주 1회
  min_idle_hours: 2
  stale_after_days: 30
  archive_after_days: 90
  backup:
    enabled: true
```

---

## 구현 체크리스트

- [ ] 1. SKILL.md 형식 + frontmatter 스키마(`created_by` 출처 필드 포함)
- [ ] 2. 디렉토리 규약 + 시작 로더(description만 주입)
- [ ] 3. 스냅샷 캐시(manifest = mtime_ns + size)
- [ ] 4. 도구: `skill_view` / `skill_manage` / `skills_list`, 모두 원자적 쓰기
- [ ] 5. 텔레메트리 sidecar `.usage.json` + 3가지 bump 이벤트
- [ ] 6. 시스템 프롬프트 규율
- [ ] 7. Curator: 유휴 트리거 + 상태 기계 + 기본값 + 백업 + 삭제 없음
- [ ] 8. pin/unpin 제외 로직
- [ ] 9. CLI 동사 세트

---

## 완전한 명세서

모든 데이터 스키마, 상태 기계, 도구 인터페이스, 기본값은 [SPEC.md](SPEC.md) 참조.

---

## MyClaw.ai와 함께

[![MyClaw.ai — 기억하는 AI Agent](https://img.shields.io/badge/MyClaw.ai-기억하는%20AI%20Agent-6366f1?style=for-the-badge)](https://myclaw.ai)

이 명세서는 [MyClaw.ai](https://myclaw.ai) 플랫폼의 일부로 개발·검증되었습니다.

---

## 라이선스

MIT
