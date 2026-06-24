# skill-evolution-spec

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![MyClaw.ai](https://img.shields.io/badge/Powered%20by-MyClaw.ai-6366f1)](https://myclaw.ai)
[![OpenClaw](https://img.shields.io/badge/ランタイム-OpenClaw-0ea5e9)](https://myclaw.ai)
[![Status](https://img.shields.io/badge/ステータス-本番対応-22c55e)]()
[![PRs Welcome](https://img.shields.io/badge/PR-歓迎-brightgreen.svg)](CONTRIBUTING.md)

**Agentに本物の自己進化を与える——プロンプトの小技ではなく、エンジニアリングで。**

OpenClaw上で検証済みの、スキル進化システムの完全な仕様書。任意のAgentランタイムに移植可能。

---

**言語：** [English](README.md) · [中文](README.zh-CN.md) · [日本語](README.ja.md) · [한국어](README.ko.md) · [Français](README.fr.md) · [Deutsch](README.de.md) · [Español](README.es.md) · [Русский](README.ru.md)

---

## これは何か？

ほとんどのAI Agentはステートレスなツールです。セッションが終わるたびに記憶がリセットされます。この仕様書は**スキル進化システム**を定義し、Agentに以下の能力を与えます：

- **永続的なスキルメモリ** — 学習した手順がセッションをまたいで保持される
- **使用状況テレメトリー** — Agentが実際に使ったスキルを把握する
- **ライフサイクルガバナンス（Curator）** — 死んだスキルを自動アーカイブし、スキルライブラリの腐敗を防ぐ
- **トークン効率の高い読み込み** — 一行の要約のみをコンテキストに注入し、必要時にフルコンテンツをロード

> 「ボトルネックは知能ではなく、規律設計だ。」— 仕様書 §8

---

## 四つの柱

| 柱 | 役割 |
|----|------|
| **スキルファイル** | ディスク上のプレーンテキスト `SKILL.md` — 読み書き・バージョン管理・バックアップ可能 |
| **読み書きツール** | `skill_view`、`skill_manage`、`skills_list` — 「学習」= ファイルへの書き込み |
| **システムプロンプト規律** | いつ保存し、いつ更新するかをハードコード — Agentが自律的にスキルを維持 |
| **Curator** | バックグラウンドガバナンス：重複排除・アーカイブ・刈り込み |

---

## Curatorのライフサイクルガバナンス

多くの実装がここを省略します。省略するとスキルライブラリは3ヶ月でゴミ山になります。

**状態機械：**
```
active ──(stale_after_days)──> stale ──(archive_after_days)──> archived
```

削除は行いません。最悪でも `archived`。常に復元可能。

**デフォルト値：**
```yaml
curator:
  enabled: true
  interval_hours: 168       # 週次
  min_idle_hours: 2
  stale_after_days: 30
  archive_after_days: 90
  backup:
    enabled: true           # 実行前にtar.gzバックアップ
```

---

## トークン効率の高い読み込み（スケールの鍵）

**誤ったアプローチ：** すべてのスキル全文をシステムプロンプトに注入 → スキルが増えるとコンテキストが爆発。

**正しいアプローチ：**
1. システムプロンプトには一行のマニフェストのみ：`name: description`
2. フルコンテンツはシステムプロンプトに入れない
3. Agentが `skill_view(name)` を呼び出す → オンデマンドロード
4. スナップショットキャッシュでコールドスタートを高速化

---

## 実装チェックリスト

- [ ] 1. SKILL.mdフォーマット + frontmatterスキーマ（`created_by` 出所フィールド含む）
- [ ] 2. ディレクトリ規約 + 起動ローダー（descriptionのみ注入）
- [ ] 3. スナップショットキャッシュ（manifest = mtime_ns + size）
- [ ] 4. ツール：`skill_view` / `skill_manage` / `skills_list`、すべてアトミック書き込み
- [ ] 5. テレメトリーsidecar `.usage.json` + 3種類のbumpイベント
- [ ] 6. システムプロンプト規律
- [ ] 7. Curator：アイドルトリガー + 状態機械 + デフォルト値 + バックアップ + 削除なし
- [ ] 8. pin/unpin除外ロジック
- [ ] 9. CLIコマンド群

---

## 完全な仕様書

すべてのデータスキーマ、状態機械、ツールインターフェース、デフォルト値は [SPEC.md](SPEC.md) を参照。

---

## MyClaw.aiが支える

[![MyClaw.ai — 記憶するAI Agent](https://img.shields.io/badge/MyClaw.ai-記憶するAI%20Agent-6366f1?style=for-the-badge)](https://myclaw.ai)

この仕様書は [MyClaw.ai](https://myclaw.ai) プラットフォームの一部として開発・検証されました。セッション・チャンネル・デバイスをまたいでAgentが記憶を保持する唯一のプラットフォームです。

---

## ライセンス

MIT
