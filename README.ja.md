Language: [English](README.md) | 日本語

# agent-stocktake

[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/shimo4228/agent-stocktake)

**agent 定義**（`~/.claude/agents/*.md`、サブエージェントを定義するファイル）を棚卸しし、ファイルごとに Keep・Improve・Demote to skill・Dissolve などの判定を出す Claude Code 向けの Agent Skill（[仕様](https://agentskills.io/specification)は英語）です。agent 定義に専用の監査があるのは、1 つの定義がコンテキストを 2 回使うからです。description は毎セッション読み込まれる一覧に載り、本文は agent が呼ばれるたびに読み込まれます（[ハイブリッドコストモデル](#ハイブリッドコストモデル)を参照）。著者の stocktake（棚卸し）スキルの 3 つ目で、スキル向けの [skill-stocktake](https://github.com/shimo4228/skill-stocktake)、常駐ルール向けの [rules-stocktake](https://github.com/shimo4228/rules-stocktake) に続きます。

監査の最後には agent ごとに 1 行の表 `Agent | Desc words | Body lines | Usage 90d | Verdict | Reason` が出て、Keep 以外の理由は証拠を挙げるので、表を見て判断できます。スキル自身が挙げる Improve の理由の例です:

> L23 'only report issues you are 80%+ confident in' suppresses findings per current-generation guidance — invert to 'report everything; caller filters in a separate pass'.

監査が agent 定義を編集・降格・削除するのは、あなたがそのファイルを 1 件ずつ承認した後だけです。著者の関連する記事とリポジトリは[末尾の節](#著者のほかの仕事)にまとめています。

## インストール

インストールの方法は 2 つあります。このリポジトリを clone する方法（1 つ目のブロック）では agent-stocktake だけが入ります。akc-cycle プラグイン（2 つ目のブロック）では、監査の最後のフェーズが仕事を引き渡す 2 つのスキル、`skill-creator`（agent 定義からスキルを作る）と `adr-writer`（agent を削除した理由を ADR に記録する）も一緒に入ります（clone でこれらが無い場合は[必要環境](#必要環境)を参照）。

```bash
git clone https://github.com/shimo4228/agent-stocktake
mkdir -p ~/.claude/skills
cp -r agent-stocktake/skills/agent-stocktake ~/.claude/skills/agent-stocktake
```

clone で入れた場合、スキルは証拠スクリプトを `~/.claude/skills/agent-stocktake` から実行するので、フォルダはこの場所に置いてください。実行は `/agent-stocktake` と入力します。このスキルは `disable-model-invocation: true` を設定しているので、Claude が自分から起動することはなく、呼ぶまでセッションのコンテキストにも載りません。

同じスキルは、Agent Knowledge Cycle（AKC: コーディングエージェントが繰り返した経験をスキルとルールに変える、人が承認する著者の 6 フェーズのサイクル）のほかのスキルと一緒に、Claude Code プラグイン [akc-cycle](https://github.com/shimo4228/akc-cycle)（英語）にも入っています。agent-stocktake はこのサイクルの Curate（整理）フェーズに属し、プラグインでは `/akc-cycle:agent-stocktake` という名前で呼びます。このリポジトリは同じ元から一方向に同期しているので、同期と同期の間はプラグインより古いことがあります。

```
/plugin marketplace add shimo4228/akc-cycle
/plugin install akc-cycle@akc-cycle
```

## 必要環境

必須なのは最初の 2 つだけで、残りはすべて任意です。

- **Glob** / **Read** / **Edit** / **Write** / **Bash** ツールを持つ Claude Code（Write は結果の台帳の更新に使います。監査自体はメインの会話の中で走り、サブエージェントは起動しません）。
- 同梱の証拠スクリプト用に [`uv`](https://docs.astral.sh/uv/)（英語）と Python 3.11 以上。
- 任意: changed モードのタイムスタンプ判定に `jq`。
- 任意（clone で入れる場合）: `skill-creator` と `adr-writer` という名前のスキル（akc-cycle プラグインには両方入っています）。使うのは Phase 4 だけです（[動作](#動作)を参照）。これらが無くても監査はすべての判定を出し、Dissolve と判定した agent も確認後に削除しますが、Demote 先のスキルの作成と ADR の記録は自分で行うことになります。
- 任意（著者のハーネスにあり、このリポジトリには入っていないもの。ハーネスは著者自身の Claude Code の設定で、このリポジトリの同期元です）: 各 agent の frontmatter、`name` がファイル名と一致するか、agent 本文の Markdown リンクが解決するかを検査する `harness_lint.py`（監査はこれらの検査を自分では繰り返さずにその結果を読むので、無いときはこれらの検査は行われません）と、usage 列のための agent-usage logging hook（ログが無いとき usage は 0 ではなく `—`、つまり未計測になります）。

## ハイブリッドコストモデル

agent 定義はコンテキストを 2 通りに使います。1 つは常に読み込まれるルールと同じ使い方、もう 1 つは呼ばれたときに読み込まれるスキルと同じ使い方です:

- `description` は "Available agent types" 一覧として**毎セッション**注入されます。ルールと同じ**常駐**です。監査は各 description が短い語数で多くを伝えているか、本文に忠実か、モデルが委譲先を選ぶ手がかりになるかを問い、**合計語数**も追跡します。一覧が長いほど、各エントリの選択シグナルは薄まります。
- 本文は agent が起動された時だけロードされます。スキルと同じく**呼び出し時**のコストですが、トリガーは description との照合ではなくモデルの委譲判断です。監査は、本文が最新か、ほかの agent と役割が重複していないか、現行モデル世代を静かに劣化させる指示を含まないかを問います。ここでの現行モデル世代は Claude 5 です。このスキルは 2026 年 7 月の著者による Claude 4 → Claude 5 の監査の中で作られ、モデルの振る舞いについての前提もそのときに定めました。

## 監査が agent 本文で探すもの

- **抑制指示**: 確信度の閾値（「N% 以上確信がある指摘のみ報告」）、重大度の下限（「重大な指摘のみ」）、「保守的に」というフレーミング。この監査は*全部報告して別パスでフィルタ*を自身の原則とします。現行モデル（Claude 5）は抑制指示を文字通り実行し、指摘を静かに落とすからです。これらは **Improve-by-inversion** 候補で、指示を逆方向に書き換えます。削除だけでは抑制的なフレームが残るからです。
- **旧世代の過剰制約**: 現行モデル（Claude 5）がもともと自分で下せる判断を一歩ずつ細かく指示する手順、同じ強調の繰り返し、ALWAYS と NEVER を対にした指示。
- **Claude Code 本体による吸収**: Claude Code 本体（組み込みの機能とメインの会話ループ）が、その agent の仕事をすでに引き受けている状態です。役割に要るのがどちらのコンテキストかで判定します。*新しい*コンテキストが効く役割（レビュー・敵対的検証・本質評価。本質評価とは、そもそも作るべきかを判定することです）はサブエージェントとして置く正当性があり、メイン会話の*豊富な*コンテキストが効く役割（計画・生成）は既定ではメインループの仕事なので、後者にとってはメインループ自体が吸収者に数えられます。SKILL.md は、こうした役割をサブエージェントに残してよい例外を 2 つ挙げています。呼び出し側が自己完結した入力をまとめて渡す場合と、作業の量がメインのコンテキストをあふれさせるほど大きい場合です。

## モード

| モード | トリガー | 動作 |
|------|---------|--------------|
| **full** | 既定、または `/agent-stocktake full` | 全 agent 定義を読んで評価 |
| **changed** | `/agent-stocktake changed` | 前回実行以降に変更されたファイルのみ再評価し、残りは台帳（実行ごとに判定を記録する `results.json`）から引き継ぐ。ほかの場所で起きた参照切れは mtime に映らないため、参照チェックは常に全件 |

## 動作

1. **Phase 1 — 証拠・棚卸し・使用回数**: 同梱スクリプト（`scripts/agent_evidence.py`）が description 語数と本文行数を測り、列挙されたツールを分類し、似通った description の組と、抑制指示・ALWAYS/NEVER 表現の候補を行番号付きで挙げます。出力は判定を含まない JSON です。続いて監査は agent 本文に書かれたファイルパスが解決するかを確かめ、使用回数（使用回数を記録する hook があれば）を下限値の但し書き付きで証拠に入れます。
2. **Phase 2 — 評価**: agent ごとに 8 問の Yes/No 質問に答えます（description について 2 問、本文について 6 問）。Keep 以外の仮の判定には、それを**反証**しにいく agent 固有の質問を続けて当てます。Dissolve 候補は吸収者を具体名で挙げられなければ反証されます。回答は判定の証拠であり、スコアには集計しません。
3. **Phase 3 — サマリ**: `Agent | Desc words | Body lines | Usage 90d | Verdict | Reason` 表を出し、末尾に description 合計語数と前回からの差分を添えます。
4. **Phase 4 — 整理実行**: 候補は **1 件ずつ確認**します。証拠を先に、次に `[y/n/skip]` で、一括承認はしません。承認された編集はセッション内で適用し、Demote は `skill-creator` という名前のスキルへ、Dissolve は `adr-writer` で理由を ADR に記録することを提案します。

## 判定基準

| 判定 | 意味 |
|---------|---------|
| **Keep** | 一覧の行としても本文としても役に立っている: description は短く忠実、本文は最新でほかと重複しない |
| **Improve** | 残す価値はあるが、引き締める必要がある。抑制指示の**反転**を含む |
| **Update** | 参照している技術・ツール・モデルが古い（証拠付きで検証済み） |
| **Merge into [X]** | 他 agent との実質的重複 |
| **Demote to skill** | 価値は指示内容にあり、独立したコンテキストや処理にはない |
| **Dissolve** | Claude Code 本体に吸収された。*成功*による退役で、古い本文が新しい既定を上書きする前に削除し、理由を ADR に記録 |
| **Retire** | 欠陥による除去: 品質が低い、古くなった、直せないほど壊れている |

## 参考研究

Yes/No 質問による設計は skill-stocktake / rules-stocktake から継承し、チェックリストによる評価の研究に従っています（いずれも英語）: [BinEval "Ask, Don't Judge"](https://arxiv.org/abs/2606.27226)、CheckEval (arXiv:2403.18771)、TICK (arXiv:2410.03608)。

## 著者のほかの仕事

- **[akc-cycle](https://github.com/shimo4228/akc-cycle)**: このスキルをサイクルのほかのスキルと一緒に、1 つの Claude Code プラグインとして入れます（英語）。
- **[Agent Knowledge Cycle](https://github.com/shimo4228/agent-knowledge-cycle)**: Curate を含むサイクルの各フェーズがなぜあるのかを、日付付きの設計判断として記録しています。
- **[skill-stocktake](https://github.com/shimo4228/skill-stocktake)**: インストール済みのスキルについての同種の監査で、古さ・矛盾・重複を見つけてスキルごとに判定を出します。
- **[rules-stocktake](https://github.com/shimo4228/rules-stocktake)**: 常駐ルールについての同種の監査で、各ルールが毎セッション払わせるコストを量ります。
- **[generation-audit](https://github.com/shimo4228/generation-audit)**: 新しい Claude モデルが役割を引き継ぐときに自作のルール・スキル・agent を点検し直し、agent については集めた証拠で `/agent-stocktake` を実行するよう提案します。
- **[shimo4228](https://github.com/shimo4228/shimo4228)**: 著者の拠点リポジトリです。AKC をほかの長期プロジェクトとその DOI と並べています。

## ライセンス

MIT

<details>
<summary>ツールと AI アシスタント向けの資料</summary>

agent-stocktake は、`~/.claude/agents/` の全サブエージェント定義を 1 つのコンテキストで読み、ファイルごとに判定を出す Claude Code 向けの Agent Skill です。自分のサブエージェントを持っていて、毎セッション載る一覧を短く、本文を現行に保ちたい人のためのものです。判定は Keep・Improve・Update・Merge into [X]・Demote to skill・Dissolve・Retire の 7 つで、1 件ずつの `[y/n/skip]` の確認なしには agent 定義の編集・降格・削除をしません。確認なしに書く自分のファイルはスキルのフォルダの中だけで、結果の台帳と、証拠スクリプトの初回実行時に `uv` がそこに作る `.venv` です。フォルダの外に書くのは `uv` 自身のキャッシュだけで、初回実行時に `uv` がビルドバックエンド（hatchling）とテスト用の依存（pytest）をそこへダウンロードします。

存在する理由は、agent 定義が 2 つのコストを払うことです。description はルールと同じく毎セッション "Available agent types" 一覧に載り、一覧が長いほど各エントリの選択シグナルが弱まります。本文はスキルと同じく呼ばれたときだけ読み込まれますが、トリガーはモデルの委譲判断です。そのため description は常駐の密度と本文への忠実さで、本文は呼び出しの質で監査します。本文で見るのは、現行モデル（Claude 5）が文字通りに従って指摘を静かに落とす抑制指示（確信度の閾値・重大度の下限）、旧世代向けの過剰制約、Claude Code 本体の機能やメインループによる吸収です。Dissolve の判定は Agent Knowledge Cycle の Scaffold Dissolution の考え方（Claude Code 本体が仕事を引き受けた agent を退役させること）を実装しています。モデル世代が変わったときは [generation-audit](https://github.com/shimo4228/generation-audit) が証拠を集め、それを使って `/agent-stocktake` を実行するようユーザーに提案します。Claude はこのスキルを自分から起動できないからです。

基本的な事実: MIT ライセンス。`SKILL.md` と Python スクリプト 1 本 `scripts/agent_evidence.py`（Python 3.11 以上、同梱の `uv.lock` で `uv` から実行、テストは `tests/`）からなり、スクリプトは JSON の証拠だけを出して判定は出しません。著者 1 人（@shimo4228）が保守しています。状態: 稼働中で、著者の Claude Code ハーネスから `scripts/sync-from-local.sh`（`--dry-run` は差分の報告だけ。commit はしません）で一方向に同期しており、akc-cycle プラグインにも `/akc-cycle:agent-stocktake` として入っているため、同期と同期の間はこのリポジトリがプラグインより古いことがあります。要件: Glob・Read・Edit・Write・Bash を使える Claude Code、`uv`、`changed` モード用の `jq`。有料の鍵は不要です。`/agent-stocktake` と呼んだときだけ動きます（`disable-model-invocation: true`）。2 つの入力は著者のハーネスから来ていて、このリポジトリには入っていません。frontmatter、name/ファイル名、Markdown リンクの検査は `harness_lint.py` の結果を読み、usage 列は `log-agent-usage.sh` hook が書く `~/.claude/metrics/agent-usage.jsonl` を読みます（Agent tool の起動しか記録しないので、回数は下限値です）。スクリプトは、どの MCP サーバが設定されているかを見るためにローカルの MCP 設定ファイル（`~/.claude.json`、`~/.claude/.mcp.json`）を読みます。承認された編集は自分で適用し、Demote は `skill-creator` という名前のスキルに引き渡し、Dissolve には `adr-writer` を提案し、台帳をスキルのフォルダの `results.json` に書きます（clone で入れた場合は `~/.claude/skills/agent-stocktake/results.json`、プラグインでは `${CLAUDE_PLUGIN_ROOT}/skills/agent-stocktake/results.json`）。

例: `uv run --frozen --project ~/.claude/skills/agent-stocktake --directory ~/.claude/skills/agent-stocktake python scripts/agent_evidence.py --root ~/.claude` は agent ごとの `desc_words`・`body_lines`・ツールの状態に加えて `total_desc_words`・`description_near_duplicates`・`suppression_candidates`・`always_never_candidates` を出力し、指摘の数にかかわらず終了コード 0 を返します（コーパス、または `--known-tools` で渡したファイルが読めないときだけ 2）。監査はその後 `Agent | Desc words | Body lines | Usage 90d | Verdict | Reason` の表を描画します。Improve の理由は例えば "L23 'only report issues you are 80%+ confident in' suppresses findings per current-generation guidance — invert to 'report everything; caller filters in a separate pass'." のように書かれます。

リンク: [skills/agent-stocktake/SKILL.md](skills/agent-stocktake/SKILL.md)（英語）がスキル本体、[llms.txt](llms.txt)（英語）が機械可読の要約です。このスキルは [Agent Knowledge Cycle (AKC)](https://github.com/shimo4228/agent-knowledge-cycle) の Curate フェーズを agent 定義の層へ広げるもので、AKC の concept DOI（常に最新版へつながる代表 DOI）は [10.5281/zenodo.19200726](https://doi.org/10.5281/zenodo.19200726) です。引用はこの DOI で行ってください。サイクル全体をインストールできる形は [akc-cycle](https://github.com/shimo4228/akc-cycle)（英語）です。

</details>
