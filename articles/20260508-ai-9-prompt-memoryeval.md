---
title: "【全体マップ】AI 時代に新しく生まれた『9 つのエンジニアリング』を一気通貫で整理 — Prompt から Memory、Eval、..."
emoji: "📝"
type: "tech"
topics: ["ai", "tech"]
published: true
---

「Prompt Engineering って結局なに？」「Context Engineering って何が違うの？」「Harness ってよく聞くけどあれ何？」

ここ 2 年で **LLM/AI 周りのエンジニアリング領域は 9 つに分裂**しました。それぞれ独立した設計分野で、独立したベストプラクティス・独立したツールチェーンを持ち、現場ではこれらを総動員して「動くエージェント」「使われる AI 機能」を作っています。

この記事では **9 種を 1 つの全体マップにまとめ**、それぞれ何を扱い、どこから手を着け、どうつながるのかを整理します。

## 9 大エンジニアリングの全体像

| # | 領域 | やること | 代表ツール / フレームワーク |
|---|---|---|---|
| 1 | **Prompt Engineering** | 1 ターン入力の設計 (system / user / few-shot / CoT) | プロンプト集、Anthropic Console |
| 2 | **Context Engineering** | コンテキストウィンドウの中身設計 | プロンプトキャッシュ、Anthropic Skills |
| 3 | **Harness Engineering** | 推論を回すループ・足場の設計 | Claude Code、Cursor、Aider |
| 4 | **Tool / MCP Engineering** | ツール/関数の設計と接続 | Function Calling、MCP |
| 5 | **Output Engineering** | 構造化出力・パース・制約 | JSON mode、grammars |
| 6 | **Feature Engineering (RAG)** | 検索特徴量・埋め込み | embedding、re-ranker |
| 7 | **Memory Engineering** | 短期/長期/エピソード記憶 | MemGPT、Letta、Anthropic Memory |
| 8 | **Eval Engineering** | 評価ハーネス・golden set | lm-evaluation-harness、LLM-as-judge |
| 9 | **Token / Cost Engineering** | 経済性最適化 | プロンプトキャッシュ、ルーティング |

それぞれ詳細を見ていきます。

## ① Prompt Engineering — 1 ターン入力の設計

最も古くて最も誤解されている領域です。「Prompt Engineering = 呪文集」というイメージは 2024 年で終わりました。

**いま現場でやっていること:**

- **role 分離**: system に役割と制約、user にタスク、assistant に few-shot 例。これを混ぜると LLM が混乱する。
- **CoT (Chain of Thought)**: 「ステップごとに考えて」だけでも精度が上がる。Claude 4.6 以降は extended thinking がオプションで、明示的な思考トークンを得られる。
- **Few-shot 例の選び方**: 5 件適当に並べるより、**バリエーション 3 件 + エッジケース 1 件 + 反例 1 件**のほうが汎化する。
- **失敗パターンの明示**: 「こうしないで」を書いたほうが「こうして」だけより安定する場合が多い (LLM は否定を理解できないという俗説は誇張)。

身に着ける優先度: **★★★★★** (基礎中の基礎、避けて通れない)

## ② Context Engineering — コンテキストウィンドウの中身設計

2025 年に Anthropic が提唱して一気に広まった概念。Prompt Engineering が「1 ターンの文章」なら、Context Engineering は「**ウィンドウ全体に何を入れて何を切り捨てるか**」の設計学です。

**やること:**

- **コンテキストの予算管理**: 200K トークンあるからといって全部入れるとモデルが迷子になる。本当に必要な情報だけを精選。
- **プロンプトキャッシュ設計**: 静的部分 (system + tool 定義 + 共通文書) を前方に固定し、動的部分 (ユーザー入力) を末尾に置く → キャッシュヒット率を最大化。
- **Skills / Subagent の設計**: 巨大なシステムプロンプトを 1 つ持つより、分割して必要時にロードする (Anthropic Skills のパターン)。
- **長文圧縮**: 過去履歴の自動要約、関連箇所だけの抽出、Cross-Encoder による reranking。

**Prompt Engineering との違い**: Prompt Engineering は **how to ask**、Context Engineering は **what to include**。

身に着ける優先度: **★★★★★** (エージェント時代の最重要分野)

## ③ Harness Engineering — エージェントループの設計学

「Harness (ハーネス)」は元々 lm-evaluation-harness のように評価フレームワークを指していましたが、最近は **エージェントの足場 (loop + tool wiring + context management)** 全般を指すようになりました。

Claude Code、Cursor、Aider、Devin、Manus — これらは全部 **Harness** です。LLM 本体は同じ Claude/GPT でも、Harness の作りで使い心地と精度が劇的に変わる。

**Harness 設計で扱うこと:**

- **メインループ**: 「LLM 呼び出し → tool 実行 → 結果を context に追加 → 再度 LLM 呼び出し」をいつ止めるか。停止条件、ステップ上限、コスト上限。
- **エラーリカバリ**: tool が失敗したらどうする？ retry？プロンプト書き換え？ユーザーに聞く？
- **状態管理**: 進捗、過去のステップ、すでに試したパス。
- **インタラクティブ性**: ユーザーの割り込み (中断、修正、やり直し) を loop にどう組み込むか。

**評価ハーネス (lm-evaluation-harness 方面)** は別物として ⑧ に分類。

身に着ける優先度: **★★★★** (自分でエージェントを作るなら必須、使うだけなら ★★★)

## ④ Tool / MCP Engineering — ツールの設計と接続

LLM に道具を持たせる領域。Function Calling から MCP (Model Context Protocol) まで進化中です。

**Tool 設計の勘所:**

- **命名は LLM 視点で決める**: 関数名・引数名・docstring は LLM が読むので、人間視点ではなく LLM が間違えにくい言葉を選ぶ。
- **Schema は厳密に**: required / optional、enum、description を JSON Schema で明示。曖昧だと LLM が幻覚する。
- **失敗時のメッセージ**: tool が失敗したら、LLM が「次にどう試すべきか」分かるエラーメッセージを返す。スタックトレースだけ渡すと LLM がパニックする。
- **粒度**: 1 つの巨大ツールより、5 つの小さい合成可能ツールのほうが LLM は使いこなす。

**MCP (Model Context Protocol)** は「ツールを LLM に提供する標準プロトコル」。Filesystem / Git / Slack / GitHub などのサーバーがすでに大量にあり、Claude Code や Cursor から接続できる。

身に着ける優先度: **★★★★** (LLM を実用機能に組み込むなら必須)

## ⑤ Output Engineering — 構造化出力・パース・制約

「LLM の出力を確実にプログラムで読める形にする」領域。地味だけど現場で一番ハマるポイント。

**主な技法:**

- **JSON mode / Structured Output**: OpenAI / Anthropic / Gemini がそれぞれ提供。schema を渡すと型保証された JSON が返ってくる。
- **Grammar / Regex 制約**: llama.cpp / vLLM などローカル推論では BNF / 正規表現で出力を強制できる。
- **XML タグ分離**: `<thinking>...</thinking><answer>...</answer>` のように出力を構造化してパースしやすくする (Anthropic 推奨パターン)。
- **多段パース**: LLM 1 回目で粗く、2 回目で構造化。1 段で全部やらない。

**ハマりどころ:**

- 「JSON で返して」だけだと前置きが付く (`もちろんです、以下が JSON です: {...}`)。Structured Output API を使う。
- LLM が schema に違反した値を入れる (enum 外、null 突入)。検証 → リトライのループを組む。

身に着ける優先度: **★★★★** (production で運用するなら必須)

## ⑥ Feature Engineering (RAG / Embedding 時代) — 検索特徴量

伝統的 ML の「特徴量設計」が **embedding + retrieval** の文脈でリバイバルした領域。

**RAG パイプラインで設計すること:**

- **チャンキング戦略**: 固定長？意味境界？階層構造？文書タイプによってベストが違う。コードなら関数単位、論文なら段落単位、対話なら turn 単位。
- **Embedding model 選び**: text-embedding-3-large、Voyage、Cohere、BGE。**ドメイン適合性 > サイズ** が経験則。
- **Re-ranking**: ベクトル検索で 50 件取って、Cross-Encoder で 5 件に絞る。これだけで精度が 1.5-2 倍になる。
- **メタデータ filtering**: 「2026 年以降」「日本語のみ」のような事前フィルタ。LLM に投げる前に絞る。
- **ハイブリッド検索**: BM25 (キーワード) + ベクトル (意味)。固有名詞や日付は BM25 のほうが強い。

**評価指標:**

- **Recall@k**: 正解が上位 k に入る確率
- **MRR**: 正解の順位の逆数平均
- **Faithfulness**: 生成された答えがソースに忠実か (RAGAS で測る)

身に着ける優先度: **★★★★** (ドキュメント系・検索系プロダクトなら必須)

## ⑦ Memory Engineering — 短期/長期/エピソード記憶

「会話を超えて覚える」設計領域。Claude / ChatGPT の Memory 機能、MemGPT、Letta が代表。

**記憶の階層:**

- **Working memory (短期)**: 直近のターン。LLM が直接見る。
- **Episodic memory (出来事)**: 「先週ユーザーがこう言った」のような時系列。
- **Semantic memory (知識)**: 「ユーザーの好み」「ユーザーの仕事」のような恒常的事実。
- **Procedural memory (手続き)**: 「いつもこの手順でやる」「このフォーマットを好む」。

**設計トレードオフ:**

- **入れすぎ**: コンテキスト膨張、関係ない記憶が混入してノイズ化、コスト増。
- **入れなさすぎ**: 「この前話したじゃん」と言われる。
- **更新タイミング**: いつ記憶を書く？毎ターン？セッション終了時？明示的に「覚えて」と言われた時だけ？
- **忘却**: 古い記憶を消す/減衰させる。さもないと記憶が肥大化して逆に検索精度が落ちる。

実装としては **embedding + retrieval (RAG の応用)** + **要約 + ピン留め** を組み合わせるのが主流。

身に着ける優先度: **★★★** (パーソナル系プロダクトを作るなら必須)

## ⑧ Eval Engineering — 評価ハーネス設計

「いい LLM プロダクトを作る」≠「いい LLM を選ぶ」。**自分のユースケースで何が良いかを測る**設計領域。

**やること:**

- **Golden set 作成**: 100-500 件の手動ラベル付き入出力ペア。これがすべての出発点。
- **評価メトリクス選定**: 完全一致 (exact match) / LLM-as-judge / F1・BLEU・ROUGE / 人間評価。
- **回帰テスト化**: PR ごとに golden set を流して、退行がないかチェック。CI に組み込む。
- **ハーネス**: lm-evaluation-harness、HELM、OpenAI evals、promptfoo などで自動化。

**ハマりどころ:**

- LLM-as-judge は「自分自身を甘く採点する」バイアスがある (Claude が Claude を採点すると高めに出る)。複数モデルでクロスチェック。
- Golden set がコンテキストドリフトする。3 ヶ月ごとに見直し。

身に着ける優先度: **★★★★★** (production 投入するなら絶対避けて通れない)

## ⑨ Token / Cost Engineering — 経済性最適化

「精度 99% でも、1 リクエスト $1 かかったら使えない」領域。スケールするほど利く。

**主な技法:**

- **プロンプトキャッシュ**: 静的部分 (system + tool 定義 + 共通文書) を前方に固定。Anthropic / OpenAI / Google が対応済み。**最大 90% コスト削減**できる場面も。
- **モデルルーティング**: 簡単なタスクは Haiku、複雑なタスクだけ Opus。前段ルーターを 1 つ挟むだけで月額が半減することも。
- **バッチング**: OpenAI / Anthropic の Batch API で 24h 以内処理 → **50% 引き**。リアルタイム不要なら超有効。
- **トークン削減**: 不要な空白除去、JSON minify、共通テンプレート省略 — 各々小さいが効いてくる。
- **モデル蒸留**: 大規模モデルの出力を小規模モデルに学習させる。production で安定したら検討。

**観測すべき KPI:**

- リクエストあたり平均トークン数 (input / output 別)
- キャッシュヒット率
- モデル別コスト分布
- 月額の予測曲線

身に着ける優先度: **★★★★** (スケールするなら避けて通れない)

## どの順番で身に着けるか

「全部一気には無理」という方への現実的な道のり:

1. **入門** → ① Prompt + ⑤ Output (1 週間で基礎を固める)
2. **応用** → ② Context + ④ Tool/MCP (1 ヶ月で「使われる機能」が作れる)
3. **エージェント自作** → ③ Harness + ⑦ Memory (3 ヶ月)
4. **検索プロダクト** → ⑥ Feature/RAG (並行)
5. **production 運用** → ⑧ Eval + ⑨ Token/Cost (運用フェーズ必須)

①③⑤⑧ は **「誰がやっても必須」**、②④⑥⑦⑨ は **「やりたいプロダクト次第」** という分類で覚えておくとブレません。

## 2026 年に脚光を浴びてる順 (筆者の主観)

1. **Context Engineering** — Anthropic 発の新概念で、Skills / Subagent の登場で一気に必修化。
2. **Harness Engineering** — Claude Code の爆発的普及で「自分専用 Harness」を組む人が急増。
3. **Memory Engineering** — Letta / MemGPT に加えて、ChatGPT / Claude のネイティブ Memory 機能が出揃って実用フェーズへ。
4. **Eval Engineering** — production 事故の多発で、評価ハーネスを後回しにできなくなった。
5. **Token Engineering** — プロンプトキャッシュが広く対応して、コスト構造が大きく変わった。

## まとめ

LLM 時代のエンジニアリングは「Prompt」だけでは説明できなくなりました。**9 つの独立した設計領域**が並列に存在し、それぞれにベストプラクティス・ツールチェーン・評価指標があります。

最初から 9 つ全部覚える必要はありません。まず ①⑤ で基礎、次にやりたいプロダクトに合わせて広げる。3 ヶ月後には自然と全部触っているはずです。

「自分はどこまで来てるか」確認しながら、欠けてる領域を 1 つずつ埋めていくのが、結局いちばんの近道です。

## 参考文献

- [Anthropic Skills](https://docs.anthropic.com/en/docs/agents-and-tools/skills) - 取得日: 2026-05-08
- [Model Context Protocol (MCP)](https://modelcontextprotocol.io/) - 取得日: 2026-05-08
- [lm-evaluation-harness (EleutherAI)](https://github.com/EleutherAI/lm-evaluation-harness) - 取得日: 2026-05-08
- [Letta (formerly MemGPT)](https://github.com/letta-ai/letta) - 取得日: 2026-05-08
- [RAGAS evaluation framework](https://github.com/explodinggradients/ragas) - 取得日: 2026-05-08

