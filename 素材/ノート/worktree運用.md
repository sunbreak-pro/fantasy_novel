# worktree 運用メモ（オーケストレーター体制）

作成日: 2026-08-02。main のチェックアウト（`dev/fantasy_novel`）を司令塔、worktree を担当別の作業机とする体制。執筆が完了するまで全 worktree を存続させる。

## 配置と役割

| worktree                                | ブランチ               | 役割                                                                      |
| --------------------------------------- | ---------------------- | ------------------------------------------------------------------------- |
| `dev/fantasy_novel`（本体）             | `main`                 | オーケストレーター。方針決定・PR の集約・MEMORY.md の更新はここに集約する |
| `fantasy_novel-worktrees/characters`    | `design/characters`    | キャラクター設定（c-004 肉付け・新キャラ三人・ガルド/友人の残り）         |
| `fantasy_novel-worktrees/organizations` | `design/organizations` | 組織・社会設定（討伐同盟・教会・少年兵制度・才能社会）                    |
| `fantasy_novel-worktrees/story`         | `design/story`         | ストーリー設定（謎の答えの先行設計・話単位分解・伏線）                    |
| `fantasy_novel-worktrees/audit`         | `audit/consistency`    | 監査。正典間の矛盾・反映漏れの定期チェック                                |
| `fantasy_novel-worktrees/auto-writing`  | `auto/writing`         | 自動執筆パイプライン（設計 → 執筆 → セルフレビュー → commit）             |
| `fantasy_novel-worktrees/auto-verify`   | `auto/verify`          | 自動検証。auto/writing の成果を別コンテキストで監査                       |

## 運用ルール

1. 起点はすべて main。各 worktree の成果は PR で main に集約する。
2. PR マージ後、他の worktree は `git fetch origin && git merge origin/main` で最新の正典を取り込んでから作業を続ける。
3. 正典（`.claude/MEMORY.md`・`設計/`）は競合しやすい。担当領域外のファイルを同時に触らない。MEMORY.md の更新は原則オーケストレーター（main）で行う。
4. worktree とブランチはマージ後も削除しない。同じ机を使い回す。
5. 各 worktree でのセッション開始時は `/session-start` を通し、担当領域を最初に宣言する。

## チャット間の連絡（comm プロトコル・2026-08-02 導入）

レーン間の書き置き・作者への判断依頼は `.claude/comm/` 経由で行う（life-editor から移植。正本 = `.claude/comm/README.md`）。

- チャット名は上の表の worktree と 1:1（`chat-main` / `chat-characters` / `chat-organizations` / `chat-story` / `chat-audit` / `chat-auto-writing` / `chat-auto-verify`）。
- 読み書きは必ず main チェックアウト（`dev/fantasy_novel/.claude/comm/`）の絶対パス。worktree 内のコピーはブランチ差で古いため使わない。
- 連絡 = 自分の `outbox/chat-<自分>.md` に降順追記。作者判断待ち = `decisions/chat-<自分>.md` に書き溜めて次の作業へ（回答は `decisions/ANSWERS.md` に届く）。
- comm/ の commit は chat-main が区切りでまとめて行う（他レーンは commit しない）。

## Orca（ADE）との連携

fantasy_novel は Orca に git リポジトリとして登録済み（2026-08-02。既存フォルダの取り込み方式・プロジェクト ID `github:sunbreak-pro/fantasy_novel`）。

- 新しい worktree を git CLI で作ったときは、`orca worktree set --worktree "path:<worktreeの絶対パス>" --display-name "<名前>"` を一発叩くと Orca の台帳に登録されて UI に表示される。
- 外部 worktree の自動表示はリポジトリ設定の externalWorktreeVisibility（Show/Hide）に従う。表示されないときはまずこのスイッチを確認する。
- 過去の注意: このリポジトリはかつて「フォルダ」として Orca に登録されており（移行残骸 `legacy-repo`）、その状態では git 機能・worktree 検出が一切働かなかった。同じ症状が出たら `orca repo show --repo "name:<リポジトリ名>"` で `kind` が `git` かを確認する。

## 自動執筆パイプライン（auto-writing / auto-verify）

役割分担: auto-writing が「作る側」、auto-verify が「別の目で検品する側」。同じセッションに書かせて自分で採点させない（自己評価バイアス回避）。

起動例（各 worktree で Claude セッションを開いて貼る。/loop は Claude に実行させず、作者が貼る）:

- auto-writing:
  `/loop 45m 設計正典（MEMORY.md・設計/）に従い、次の未執筆の話を1本執筆する。章設計が無ければ設計から始め、執筆後に /manuscript-review を通して指摘を修正してから commit する。第1章ぶんが完了したら停止する`
- auto-verify:
  `/loop 60m origin/auto/writing を fetch して新規コミットを確認し、新しい原稿があれば正典との整合と文章品質を監査して、指摘を素材/参照/指摘ログ.md の形式で記録し commit する。新規原稿が無ければ何もしない`

ループには必ず停止条件を添える。PC を閉じても回したい場合は /schedule（クラウド定期実行）に切り替える。

## 自動執筆の解禁条件（重要）

MEMORY.md の規則により【未定】の設定で本文を書いてはならない。以下が確定するまで auto-writing は起動しない:

- 謎の答えの先行設計（F-002・F-003 の残り）
- 新キャラ三人の創出（c-004 肉付け・兄弟姉妹・悪意の法術使い）
- 第一部の話単位分解（少なくとも第1章ぶん）
- 文体の方向性・話の開始/終了の型（writing.md の【未定】欄）
