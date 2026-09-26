# upstream追従の終了と次世代エンジンとしての独立判断

実施時刻: 2026-09-26
対象 Issue: #1 / #39 / 上流 RVC-Project/Retrieval-based-Voice-Conversion-WebUI #2854
branch / HEAD: `claude/fox-next-gen-engine-y9ja38`
実行環境: 文書判断のみ。実機試験なし
結果: success（方針決定と文書改訂）

担当: 齋藤みつる / Claude Code

## 目的

upstream へ差分を戻す前提の開発方針を終了し、このforkをRVC WebUIを祖先に持つ独自の次世代声変換エンジンとして扱う判断を記録する。

## 入力・前提

- `experiments/20260905-upstream-attribution-boundary-freeze.ja.md`
- `experiments/20260905-upstream-pr-readiness-audit.ja.md`
- 上流への実装提案 Issue #2854
- 開発者note「GarageBandを生かしたまま、AIランナーだけを3回殺した話」（2026-09-05） https://note.com/fusamofu326/n/n1a29ca1ef393
- 開発者note「元ベンチャー社長、現職NEETの、私が求める雇用主（正確にはPatient Capital / Impact Patron像）を説明する」（2026-09-25）  https://note.com/fusamofu326/n/neb54d4397dc5

## 実行コマンド

```text
なし（README.md / AGENTS.md の文書改訂のみ）
```

## 観測事実

- 上流repositoryは `has_pull_requests: false` であり、提出用分岐 `upstream/macos-au-webui-runtime` をPRにできなかった（2026-09-05の記録）
- 実装済み提案を上流Issue #2854として提出したが、2026-09-26時点で開発者は反応なしと認識している
- 上流側にIntel Mac、macOS/AU、DAW実機で統合検証する経路は観測されていない
- fork側はML stack再設計、CPU学習の完走モデル3本（開発者申告。receiptは既存experimentsに一部）、AU v2、WebUI所有runtime、RSVC protocol、Bonjour、GarageBand offline Bounce実用合格まで到達している
- 未検証: Windows実機回帰、Apple Silicon、Logic Pro、別Mac間Bonjour、Wi-Fi断、realtime monitoring、複数client

## 解釈 / 仮説

- upstreamへ差分を寄せ続ける分離コストに対し、受け取り側の経路がないため技術的な利益がない
- 検証が閉じない原因は設計ではなく検証機材と電力であり、上流ではなく独自の資金・物資・contributor・ミュージシャン支援で解く問題である
- 次世代エンジンとしてのalpha（手元実機での通し経路と実用）は完了しており、残りは一発installerと互換性整備のbeta領域である

## 決定

- READMEをfork独自次世代エンジンとして書き換え、alpha完了とbeta移行のためのコミュニティ支援（コード・物資・資金・ミュージシャン/スタジオ）募集を明記した
- AGENTS.mdの「upstream PRへ戻せる形に保つ」を撤回し、上流をrevision固定の参照元とする方針へ改訂した
- 上流の著作権表示・MIT License・来歴は保持する。#39の非攻性防壁は維持する
- 上流がPRを再開し取込方法を示した場合、汎用差分の提供は拒まない
- repository名称変更とGitHub fork networkからの切り離しは、検証機材と電力の調達後に行う

## Recovery / 次の一手

- 投げ銭窓口（GitHub Sponsors等）の開設後、READMEへ直リンクを追加する
- 資金・機材調達後に名称（候補: OpenRVC）の空き確認、fork network切り離し、AU manufacturer/subtype codeの移行方針を決める
- beta: 一発installer、Windows回帰、Apple Silicon backend

## unknown

- 上流Issue #2854が今後反応されるか
- 上流のPR無効化の背景（#39と同じくUNKNOWN、事実として扱わない）
- 名称候補の商標・PyPI・GitHub org名の空き
