# `/btw` 検証記録

日付：2026-09-10。ブランチ：`codex/btw-side-question`。ベースライン：`8dae851b9b622f2ff2631f332fde9719d0b16fba`。

独立したPostgreSQLテストDBとデータディレクトリを使用しました。実モデルの認証情報は独立テスト環境にのみ注入し、コードや本記録には保存せず、製品の既定モデルも変更していません。Go 1.26.3、norma v0.3.6、Next.js 16.2.9。

実モデルとの対話、応答オブジェクト、実装のアサーション、Qwenレビューの原文は[validation-2026-09-10.json](validation-2026-09-10.json)に保存しています。API認証情報は含みません。

## 実装のチェック

| 対象 | 結果 | 証拠 |
| --- | --- | --- |
| 構造化メッセージ・ツール引数のディープコピー | 通過 | `TestCheckpointDeepCopyAndBoundaries` |
| 要約・圧縮リクエストによる上書き防止、完全応答と終端状態の公開、部分応答の除外 | 通過 | `TestCheckpointDeepCopyAndBoundaries`、`TestSnapshotExcludesPartialStreamAndSelectsPoolMember` |
| 実際のモデルプールメンバーの識別 | 通過 | `TestSnapshotExcludesPartialStreamAndSelectsPoolMember` |
| ツール対応、20組の再送、予算による削減と超過エラー | 通過 | `TestBuildRequestCompactionToolPairingAndBudget` |
| 主処理との並列実行、双方向のキャンセル分離 | 通過 | ブロッキングProvider、`TestMainSideConcurrencyAndIndependentCancellation` |
| ツール実行なし、ストリーミング / 非ストリーミング、失敗時に得られた使用量 | 通過 | `TestServiceNoToolsAndUsageOnFailure` |
| 実際のnorma ChatAgent + ローカルReadツール、主transcript・アクティビティの分離 | 通過 | `TestSideActualChatCheckpointToolResultAndTranscriptIsolation`のストリーミング・非ストリーミングのサブケース |
| 永続化、ページング、冪等性、再起動後の部分回答保持 | 通過 | `TestSideHistoryIdempotencyPagingAndRecovery` |
| 消去と遅延書き込みの競合、親リソース削除、バージョン比較 | 通過 | `TestSideClearLateWritersAndDeletedParent` |
| MainAgent / Workerのv1/v2/v3アーカイブと復元 | 通過 | `TestSideTaskArchiveVersions` |
| 3種類の親インターフェース、認証、リソース所属、Worker論理削除 | 通過 | `TestSideHTTPGlobalLimitTaskWorkerAndDeletion`、`TestSideCheckpointPersistsBeforeAdmissionAndRestart` |
| 主セッション実行中のサイド質問、独立SSE再接続・切断、キャンセル、消去 | 通過 | `TestSideHTTPBusyIsolationClearAndReconnect` |
| 親ごとに1件 / 全体4件の同時実行 | 通過 | 2つの`TestSideHTTP…`ケース |
| 受付前のスナップショット保存、再起動後の質問継続、旧セッションの偽スナップショット防止 | 通過 | `TestSideCheckpointPersistsBeforeAdmissionAndRestart` |
| キャッシュ内の設定が削除・モデル変更された後の継続拒否 | 通過 | `TestSideRejectsDeletedOrChangedCachedProfile` |
| アーカイブ前のキャンセルと最終回答・使用量の保存待ち | 通過 | `TestSideTaskDrainPersistsBeforeArchive` |
| ストリーム消費側の早期キャンセル時に使用量を1回だけ記録し、サイド質問に帰属 | 通過 | `TestSideUsageRecordedOnceOnConsumerCancellation` |
| 再起動で自動復元したWorker / deadline実行コンテキストによる新スナップショット公開 | 通過 | `TestSideRestoredWorkerRuntimePublishesNewCheckpoint` |
| 関連パッケージのraceチェック | 通過 | 下記コマンド |
| TypeScriptと本番ビルド | 通過 | `npx tsc --noEmit`、`npm run build` |
| 新規フロントエンドモジュールのBiome | 通過 | 新規3モジュールの`biome check` |

独立した破棄可能なDBを`ARTEX_PG_DSN`に設定すると、自動チェックを再現できます（本番DBを指定しないでください）。

```sh
go test -race ./agent ./db ./server ./sidequestion ./llmrec ./llmpool \
  -run 'Test(Side|Checkpoint|Snapshot|BuildRequest|Service|MainSide|CaptureRun|TaskArchive|CompleteForwards|StopIntent|CancelIntent)' -count=1
cd web
npx tsc --noEmit
npx biome check src/lib/side-questions.ts src/hooks/use-side-questions.ts src/components/side-question-workspace.tsx
npm run build
```

Goの全回帰テストはすべて通過したわけではありません。`server`の既存2テストが一時ディレクトリのクリーンアップ時に`TempDir RemoveAll … directory not empty`で失敗しました。

- `TestInheritedActivityDetailAndRelationDeletion`
- `TestTaskMetadataPatchReturnsRenameAndPin`

未変更の上記ベースラインを同じ隔離環境で実行しても、この2件を再現しました。ベースラインではさらに`TestCoreTaskLifecyclePG`の目標ノード数アサーションが失敗しましたが、最終変更後の`server`回帰ではその失敗はありませんでした。他パッケージ、および今回のサイド質問関連ケースとraceチェックは通過しました。ベースラインの問題を今回の合格として扱わず、隠蔽のための既存アサーション変更も行っていません。

Next.jsビルドには既存の複数lockfile / workspace root推定の警告が出ますが、ビルドと全ページの生成は成功しました。

## ブラウザー検証

Codex In-app Browserから独立したローカルGoサービスとNext.js開発サーバーへ接続しました。デスクトップと390×844の狭い画面で次の操作を自動化し、スクリーンショットとブラウザーログを確認しました。

- 通常チャット実行中に`/btw`を入力し、主内容とサイド質問が同時に表示され、デスクトップのサイドバーが正常であること。
- 追加質問を続けられ、サイド質問の停止後も生成済み部分を保持し、主処理は継続すること。
- パネルを閉じてもリクエストが続き、再表示時に完了回答を復元すること。再読み込み後の空`/btw`で履歴を復元すること。
- 狭い画面のDrawerで入力、ボタン、履歴、閉じる操作が正常で、横方向にはみ出さないこと。
- 消去は確認ダイアログを経由し、履歴だけが消え、主transcriptとスナップショットが保持されること。
- タスクのMainAgentと2つのWorkerで個別に質問・切り替えを行い、Agentラベルと履歴が混線しないこと。
- ブロッキングのローカルモデルフィクスチャでWorkerを実行中に保ち、その主入力欄から`/btw`を送信。サイド質問停止後もWorkerは実行中表示と自身の停止ボタンを保持し、サイド質問は部分回答を保存すること。
- ブラウザーのエラー・警告ログが空であること。

同時実行のタイミングは制御可能なフィクスチャで検証し、実モデルの出力速度には依存していません。調整中の2回はタスク終了・回答の早期終了で有効な同時実行区間を作れませんでした。フィクスチャ修正後に再検証して通過し、初期の操作は有効な通過として数えていません。

## 実モデルとの対話

まず`grok-4.6`をOpenAI互換エンドポイント`http://127.0.0.1:12580/tingly/openai`で確認しました。HTTP 200、モデル名`grok-4.6`と`READY`、所要2.82秒。第一候補が利用可能だったため、Tingly `glm`と智譜`glm-5.3`の予備チェーンは使用・検証していません。

| シナリオ | 実際の結果 |
| --- | --- |
| 主セッション実行中にアセット・目標・マーカーを質問 | `redhaze.top`、トップページの読み取りと要約という目標、`BTW-REAL-0910`を回答。完了まで16.97秒 |
| 主セッションのトップページ読み取り後にツールの根拠を質問 | WebFetch 200、curlの301 → 302 → 200、ページタイトルを正しく引用。7.24秒 |
| サイド質問でBashによるテストファイル作成を要求 | 実行を拒否し、対象ファイルは未作成。7.74秒 |
| 完了したサイド質問が主コンテキストを変更しない | 主transcriptのSHA-256と主アクティビティが不変。サイド質問のツール実行0回 |
| Goサービスを実際に停止・再起動して追加質問 | 3件の既存履歴を保持し、主Agentの再実行なしに永続化スナップショットからアセット・マーカー・タイトルを回答 |
| 新規セッションでGrokの非ストリーミング設定を使用 | アセットと`ATOMIC-0910`を正しく回答。input 11734、output 138、cache_read 11520を取得・保存 |

アセット事例の主セッションはWebFetchとBash/curlで公開トップページを読み取り、到達先は`https://id.redhaze.top/home`、タイトルは「红幕科技 RedHaze Group · 全球综合集团门户」でした。Bashは応答をローカルのテストファイルに一時保存しましたが、リモートへの書き込みはしていません。この事実は「サイド質問がツールを実行していないこと」と別に検証しています。

主transcriptのハッシュ：`e7e61f135a4a120954b539f357e8c4205d7d5cd7460dcaf3dc0fd066463e1d00`。

**使用量の制限：** TinglyのGrokストリーム応答はusageを返しませんでした。別途`stream_options.include_usage=true`を直接送信してもHTTP 200、データ12フレーム、usage 0フレームでした。したがってストリームテストの0は「エンドポイントが使用量を提供しない」という意味で、無料を意味しません。非ストリーミング使用量と、フィクスチャの失敗・キャンセル時の使用量は正しく保存されました。

## Qwenレビュー

`qwen-flash`をOpenAI互換エンドポイント`https://dashscope.aliyuncs.com/compatible-mode/v1`で使用し、HTTP 200でした。最初の3件の実対話、主セッションのツール根拠、実装アサーションを提供した結果、`verdict: accept`、`concerns: []`を返しました。回答はアセット・マーカー・ページ読み取りの証拠と整合し、サイド質問のツール拒否も制約どおりと評価しました。使用量はprompt 6625、completion 312、total 6937でした。

このレビューは後から追加したサービス再起動・非ストリーミングテストを含みません。Qwenによる「書き込みなし」の総括は広すぎます。主セッションのcurlはローカル応答一時ファイルを作成しており、上記に明記しました。同時実行、ツール実行0回、transcript分離は実装アサーションで判断し、モデルレビューは回答品質の補助評価に限ります。
