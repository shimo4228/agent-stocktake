Language: [English](README.md) | 日本語

# agent-stocktake

**agent 定義**（`~/.claude/agents/*.md`）の品質を棚卸しする [Agent Skill](https://agentskills.io/specification)。[skill-stocktake](https://github.com/shimo4228/skill-stocktake)・[rules-stocktake](https://github.com/shimo4228/rules-stocktake) に続く第 3 の stocktake で、agent 定義が両者のコストモデルを**同時に**払っていることが、独立スキルとして存在する理由。

## ハイブリッドコストモデル

skill のコストはトリガー汚染、rule のコストは常駐（residency）。agent は両方を持つ:

- `description` は "Available agent types" 一覧として**毎セッション**注入される——rule と同じ**常駐**。監査は各 description が濃く・本文に忠実で・委譲判断を可能にするかを問い、**合計語数**も追跡する。一覧が長いほど、各エントリの選択シグナルは薄まる。
- 本文は agent が起動された時だけロードされる——skill と同じ**呼び出し課金**。ただしトリガーは description マッチではなくモデルの委譲判断。監査は本文が最新・唯一で、現行モデル世代を静かに劣化させる指示を含まないかを問う。

## 本文スクリーンが検出するもの

- **抑制指示** — 確信度閾値（「N% 以上確信がある指摘のみ報告」）、深刻度フロア、「保守的に」フレーミング。現行世代の指針は*全部報告して別パスでフィルタ*であり、抑制指示は文字通り実行されて指摘を静かに落とす。これらは **Improve-by-inversion** 候補: 指示を逆方向に書き換える——削除だけでは抑制的フレームが残る。
- **旧世代の過剰制約** — 現行モデルが素で持つ判断への網羅的手順、反復強調、ALWAYS/NEVER 対。
- **Substrate 吸収** — harness がその agent の仕事を native に持つようになった。fresh/rich context 軸で判定する: *fresh* context が効く役割（レビュー・敵対的検証・本質評価）は subagent に住む正当性があり、*rich* context が効く役割（計画・生成）はメインループの仕事——後者にとってはメインループ自体が吸収者に数えられる。

## モード

| モード | トリガー | 動作 |
|------|---------|--------------|
| **full** | 既定、または `/agent-stocktake full` | 全 agent 定義を読んで評価 |
| **changed** | `/agent-stocktake changed` | 前回実行以降に変更されたファイルのみ再評価し、残りは台帳から引き継ぐ。機械的整合チェックは常に全件——参照切れは mtime に映らない |

## 動作

1. **Phase 1 — 棚卸し + 機械的整合チェック**: `~/.claude/agents/*.md` を列挙し、description 語数と本文行数を実測。frontmatter・name/ファイル名一致・tool 実在・参照解決を検査。使用回数（invocation logging hook があれば）は下限値の但し書き付きで証拠に入る——未計測は `—` で描画し、0 とは描画しない。
2. **Phase 2 — 評価**: 2 段階のバイナリスクリーン。Stage 1 は agent ごとの 7 問 Yes/No チェックリスト（description 層 2 問 + 本文層 5 問）。Stage 2 は非 Keep 草案 verdict を**反証**しにいく agent 固有の質問を生成——Dissolve 候補は吸収者を具体名で挙げられなければ反証される。バイナリ回答は holistic verdict の証拠であり、スコアに集計しない。
3. **Phase 3 — サマリ**: `Agent | Desc words | Body lines | Usage 90d | Verdict | Reason` 表。末尾に description 合計語数と前回からの差分。
4. **Phase 4 — 整理実行**: 候補は **1 件ずつ確認**——証拠を先に、次に `[y/n/skip]`。一括承認はしない。承認された編集はセッション内で適用、Demote は skill-creator へ、Dissolve は why の ADR 化を提案する。

## Verdict 基準

| Verdict | 意味 |
|---------|---------|
| **Keep** | 両層で価値: description は濃く忠実、本文は最新で唯一 |
| **Improve** | 残す価値はあるが締めが要る——抑制指示の**反転**を含む |
| **Update** | 参照している技術・tool・モデルが古い（証拠付きで検証済み） |
| **Merge into [X]** | 他 agent との実質的重複 |
| **Demote to skill** | 価値は指示内容にあり、独立 context/process にはない |
| **Dissolve** | substrate に吸収された。*成功*による退役——古い本文が新しい既定を上書きする前に削除し、why を ADR に記録 |
| **Retire** | 欠陥ベースの除去: 低品質・陳腐化・修復不能 |

## インストール

### Claude Code

```bash
cp -r skills/agent-stocktake ~/.claude/skills/agent-stocktake
```

### SkillsMP

```bash
/skills add shimo4228/agent-stocktake
```

## 必要環境

- **Glob** / **Read** / **Edit** / **Bash** tool を持つ Claude Code（監査はメイン context 1 つで走る——subagent 不要）。
- 任意: changed モードのタイムスタンプ判定に `jq`、usage 列に agent-usage logging hook。どちらも無くても劣化動作する。

## harness からの同期

このスキルの正本は著者の生きた Claude Code harness にある。この repo は一方向の公開ミラー:

```bash
scripts/sync-from-local.sh --dry-run   # 差分の報告のみ
scripts/sync-from-local.sh             # working tree に適用（commit はしない）
```

## References

2 段階バイナリ質問設計（スクリーン → verdict 反証テスト、holistic verdict、スコア集計なし）は skill-stocktake / rules-stocktake から継承し、checklist 分解評価の研究線に従う: [BinEval "Ask, Don't Judge"](https://arxiv.org/abs/2606.27226)、CheckEval (arXiv:2403.18771)、TICK (arXiv:2410.03608)——過剰分解は holistic 品質との相関を劣化させる。ゆえに 7 問・スコアなし。吸収質問と Dissolve verdict は Agent Knowledge Cycle の **Scaffold Dissolution**（モデル世代交代トリガーを含む）を実装する——世代交代時は [generation-audit](https://github.com/shimo4228/generation-audit) が runtime 層の証拠を収集し、agents スライスをこのスキルに渡す。

## About this skill

このスキルは [Agent Knowledge Cycle (AKC)](https://github.com/shimo4228/agent-knowledge-cycle)——Zenodo で引用可能な 6 phase 双方向成長ループ（[DOI 10.5281/zenodo.19200726](https://doi.org/10.5281/zenodo.19200726)）——の **Curate** phase を agent 定義層に拡張し、stocktake セットを完成させる: [skill-health](https://github.com/shimo4228/skill-health) が構造的負債、[skill-stocktake](https://github.com/shimo4228/skill-stocktake) が skill、[rules-stocktake](https://github.com/shimo4228/rules-stocktake) が常駐 rule、agent-stocktake が毎セッション description を載せる agent 定義を担う。AKC は [@shimo4228](https://github.com/shimo4228) の 3 研究線の 1 つ（他: [Contemplative Agent](https://github.com/shimo4228/contemplative-agent) [DOI 10.5281/zenodo.19212118](https://doi.org/10.5281/zenodo.19212118)、[Agent Attribution Practice (AAP)](https://github.com/shimo4228/agent-attribution-practice) [DOI 10.5281/zenodo.19652013](https://doi.org/10.5281/zenodo.19652013)）。

## License

MIT
