# Comm Protocol — 複数 Claude チャット間のファイル経由通信

複数の Claude チャット（worktree レーン）が、ファイルを介して非同期にやり取りするための最小プロトコル。life-editor プロジェクトの comm プロトコルを、本プロジェクトの worktree 体制（`素材/ノート/worktree運用.md`）に合わせて移植したもの（outbox + 判断キューの構成）。

## このプロトコルの目的

身近な比喩で言うと、**冷蔵庫に貼った付箋でやり取りする家族**のようなものです。リアルタイム会話ではなく、書き置きを介して非同期に情報を共有する。

- 別チャットで進めている作業の状況を確認できる
- 別チャットに「これお願い」を残して、次に開いたとき気付いてもらえる
- 作者判断が要る点を書き溜めて、返事待ちで作業を止めない（→ `decisions/`）

## ファイル構造

```
.claude/comm/
├── README.md                  # このファイル（プロトコル定義）
├── outbox/
│   └── chat-<name>.md         # 各チャットの発信箱（書くのは本人のみ、読むのは全員）
├── decisions/
│   ├── README.md              # 判断キューのルール
│   ├── ANSWERS.md             # 作者の回答欄（書くのは作者のみ）
│   └── chat-<name>.md         # 各チャットの判断待ちキュー（書くのは本人のみ）
└── archive/
    └── YYYY-MM/               # 古くなったファイルの退避先
```

## 最重要ルール: 読み書きは main チェックアウトの絶対パスで行う

この comm/ は **main チェックアウト（`C:\Users\user\OneDrive\Desktop\dev\fantasy_novel\.claude\comm\`）の 1 箇所だけ**を使う。worktree（`fantasy_novel-worktrees\<slug>\`）の中にも merge 経由で `.claude/comm/` のコピーが現れるが、**そちらは読まない・書かない**。

冷蔵庫が家に 1 台だから付箋のやり取りが成立するのであって、各部屋に冷蔵庫のコピーを置くと「どの付箋が最新か」が分からなくなる。git のブランチ越しに同期する方式（life-editor 初期方式）は、fetch するまで他レーンの書き込みが見えない・設計ブランチの PR に comm の変更が混ざる、という二重の問題があったため、本プロジェクトでは共有パス直接方式を採る。

- worktree で作業中のチャットも、comm への読み書きだけは上記の絶対パスを使う
- comm/ の git commit は **chat-main が区切りでまとめて行う**（main ブランチに載る。他レーンは commit しない）

## 中核ルール: 単一書き込み者

**outbox / decisions のチャットファイルは 1 ファイル 1 チャット専用**。これがこのプロトコルの心臓部。

| ファイル                  | 書き込んでよい人                           | 読んでよい人 |
| ------------------------- | ------------------------------------------ | ------------ |
| `outbox/chat-story.md`    | story レーンのチャットだけ                 | 全チャット   |
| `decisions/chat-story.md` | story レーンのチャットだけ                 | 全チャット   |
| `decisions/ANSWERS.md`    | 作者（または転記を任された chat-main）のみ | 全チャット   |

身近な比喩で言うと、**各自の日記帳**です。他人の日記には書き込まない、でも借りて読むのは OK。これで同時編集の衝突が**設計レベルで起きえない**構造になる。

## チャット名（レーン固定）

チャット名は worktree 体制（`素材/ノート/worktree運用.md`）と 1:1 で固定する。セッション開始時に「このチャットは chat-story」と宣言してから作業を始める。

| チャット名           | 担当 worktree / ブランチ          | D-ID 略称 |
| -------------------- | --------------------------------- | --------- |
| `chat-main`          | 本体 `main`（オーケストレーター） | main      |
| `chat-characters`    | `design/characters`               | char      |
| `chat-organizations` | `design/organizations`            | org       |
| `chat-story`         | `design/story`                    | story     |
| `chat-magic`         | `design/magic-system`             | magic     |
| `chat-monsters`      | `design/bestiary`                 | mon       |
| `chat-audit`         | `audit/consistency`               | audit     |
| `chat-auto-writing`  | `auto/writing`                    | autow     |
| `chat-auto-verify`   | `auto/verify`                     | autov     |

- 新しいレーンを増やすときは、この表と outbox / decisions のファイルを揃って追加する（chat-main の仕事）
- チャット名を被らせない・名前なしで運用しない

## Outbox のフォーマット

各 outbox は append-only の時系列ログ。**過去のエントリは編集しない**（追記のみ）。最新エントリを上に追記する（降順）。

```markdown
## YYYY-MM-DD HH:MM → @<recipient>

<body>

---
```

### 宛先タグの使い分け

- `@all` — 全チャット宛（broadcast。多用しない）
- `@chat-<name>` — 特定チャット宛（unicast）
- `@self` — 自分への備忘録（他チャットも読めるが宛先ではない）

### エントリの書き方の例

```markdown
## 2026-08-02 14:32 → @chat-main

c-004 の肉付けが一段落したので PR #14 を出しました。レビューと merge をお願いします。

- 対象: 設計/キャラクター/主要人物/c-004.md
- 論点: 姉弟関係の設定が MEMORY.md の家族構成と接触する（矛盾はないはず）
- 未処理: _index.md の更新は merge 後に main 側でお願いしたい

---
```

## 判断キュー（decisions/）

「作者に聞かないと進めない」で止まらないための書き溜め場。ルールと書式は `decisions/README.md` を参照。連絡（outbox）と判断依頼（decisions）は混ぜない — outbox に質問を書いても回答レーンには乗らない。

## 衝突対策（3 層防御）

| 層      | 対策                                       | 効果                                      |
| ------- | ------------------------------------------ | ----------------------------------------- |
| 1. 設計 | 単一書き込み者 + 共有パスは物理的に 1 箇所 | 衝突が起きえない                          |
| 2. 構造 | append-only・降順追記                      | 万一衝突しても損失は新規 1 エントリに限定 |
| 3. git  | chat-main が main ブランチで定期 commit    | 履歴から復元可能                          |

## 運用フロー

### 1. セッション開始時（全チャット共通）

1. チャット名を宣言する（例:「このチャットは chat-story」。/session-start がこの確認を含む）
2. 自分の outbox / decisions ファイルの存在を確認する（無ければレーン表と照合の上で作成）
3. 他チャットの outbox から自分宛（`@chat-<自分>` と `@all`）の最新エントリを確認する
4. `decisions/ANSWERS.md` に自分の D-ID への回答が来ていないか確認する

### 2. 作業中に他チャットへ連絡したいとき

自分の outbox の先頭（ヘッダの直下）にエントリを追記する。相手はファイル監視できないため、即応してほしい場合は作者に「◯◯チャットで outbox を確認して」と伝えてもらう。

### 3. 作業の区切り

- 引き継ぎ・完了報告・作業宣言を自分の outbox に残す
- chat-main は区切りで comm/ の変更を main に commit する

## 重要な制約: Claude はファイルを監視できない

**最重要の落とし穴**。誰かが outbox に書いても、もう片方の Claude は**自動では気付かない**。ポストに手紙を入れても、相手がポストを開けに行かないと届かないのと同じ。

- 基本は「セッション開始時」と「作業の区切り」に読みに行く
- 長時間ループ（/loop）で回すレーンは、ループ 1 周の頭に自分宛確認を組み込む

## アーカイブ運用

outbox / decisions が肥大化したら（目安: 100 エントリ超 or 1 ヶ月経過）、`archive/YYYY-MM/` に退避して新しいファイルを作り直す（chat-main の仕事）。

## アンチパターン

- ❌ 他チャットの outbox / decisions を編集する
- ❌ 過去エントリを書き換える（履歴破壊）
- ❌ worktree 内の `.claude/comm/` コピーを読み書きする（古い版を掴む）
- ❌ 大量の生データ（設計文書全文・原稿全文）を貼る（パスと行番号で示す）
- ❌ `@all` を多用する（ノイズ）
- ❌ 正典の変更を outbox だけで済ませる（正典の更新は従来どおり PR / 承認フロー。outbox はあくまで連絡）

## 本プロジェクトで導入していないもの（将来拡張の候補）

life-editor には存在するが、本プロジェクトでは意図的に見送った仕組み。必要になったら追加を検討する。

- **`.session-name` / `.session-branch`**: hook 連携・per-chat tracker と組で効く仕組み。本プロジェクトは hook 連携なしのため、チャット名は宣言 + レーン固定表で足りる
- **per-chat memory/history 分割**: MEMORY.md が設定正典を兼ねるため見送り（2026-08-02 作者決定）。タスク追跡は従来どおり MEMORY.md（更新は main 集約）
- **Issue dispatch（GitHub Issues でのタスク分配）**: タスクの正本は MEMORY.md のタスクトラッカー。レーン間のタスク受け渡しは当面 outbox で行う
- **digest/（日次ダイジェスト）**: レーン数が増えて chat-main の見通しが苦しくなったら導入
