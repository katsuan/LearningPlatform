# GAS / Spreadsheet MVP実装方針

## 1. この文書の目的

この文書は、LearningPlatform のMVPを `GitHub Pages + GAS Web App + Spreadsheet` で簡易実装するための方針を固定する資料です。
将来の本格バックエンド移行を妨げないようにしつつ、最短で成立させる実装単位を明確にします。

## 2. 採用方針

MVPのフロント公開面は GitHub Pages とし、バックエンドは Google Apps Script の Web App、永続化には Spreadsheet を使う。

構成:

- LINE: 入口
- GitHub Pages: LIFF フロントの配信
- LIFF: 学習UI
- GAS Web App: API、認証補助、学習ロジック、採点、保存
- Spreadsheet: DB代替
- Script Properties: 環境差分、秘密情報、接続先設定

## 3. この構成を採る理由

- 初期構築が速い
- Spreadsheet で教材や学習データを目視確認しやすい
- 小規模MVPの運用に向く
- GitHub Pages と LINE / LIFF の相性がよく、公開URLを安定して持ちやすい
- GAS と組み合わせて、最初の検証導線を作りやすい

## 4. 先に受け入れる制約

- RDB のような厳密な制約は使えない
- 複雑な検索や集計は重くなりやすい
- 同時更新や大量データには弱い
- CORS や公開構成は先に整理しないと詰まりやすい

このため、MVPでは次を守る。

- 更新系の責務を明確にする
- 取得回数を減らす
- シート責務を混ぜない
- Script Properties と Spreadsheet ID をコード直書きしない

## 5. システム境界

### 5.1 LIFF側

- 学習画面表示
- ローカル途中保存
- AnswerEvent 再送
- GitHub Pages から静的配信
- GAS Web App 呼び出し

### 5.2 GAS側

- `doGet` `doPost` を入口とする
- action でAPI責務を分岐する
- Membership解決
- Tenant境界チェック
- Session取得 / 生成
- 自動採点
- Spreadsheet 永続化

### 5.3 Spreadsheet側

- Tenant、User、Membership、Question、Session、AnswerEventなどの保存
- 管理者向け簡易集計の元データ保持

## 6. GASディレクトリ責務

GAS実装は次の責務で分ける。

- `entry`
  - `doGet` `doPost`
- `router`
  - action 解決、入力解釈
- `usecase`
  - ユースケース本体
- `domain`
  - role、session type、採点ポリシー
- `infra`
  - Spreadsheet、Properties、LockService、Utilities
- `presenter`
  - JSON レスポンス整形
- `setup`
  - シート初期化、ヘッダ生成、サンプルデータ投入

## 7. Spreadsheetシート構成

MVPでは原則 1 エンティティ 1 シートとする。

推奨シート:

- `tenants`
- `users`
- `memberships`
- `groups`
- `group_memberships`
- `feature_entitlements`
- `line_connections`
- `courses`
- `categories`
- `units`
- `questions`
- `question_options`
- `question_grading_configs`
- `assignments`
- `learning_sessions`
- `session_questions`
- `answer_events`

後回し候補:

- `learning_profiles`
- `admin_daily_summaries`

## 8. シート設計ルール

- 1 行目はヘッダ固定
- `*_id` は文字列で扱う
- 日時は ISO 8601 文字列で保存する
- JSON構造は文字列化して保存する
- enum は文字列で保存する
- 削除は物理削除より `status` 管理を優先する

## 9. Script Properties で持つ値

- `APP_BASE_URL`
- `GITHUB_PAGES_URL`
- `LIFF_ID`
- `LINE_CHANNEL_ID`
- `LINE_CHANNEL_SECRET`
- `LINE_CHANNEL_ACCESS_TOKEN`
- `SPREADSHEET_ID`
- `DEFAULT_TENANT_ID`
- `ALLOWED_ORIGIN`

必要に応じて追加:

- `DEBUG_ENABLED`
- `SYSTEM_ADMIN_EMAILS`

## 10. API設計のGAS向け読み替え

OpenAPI では REST 風に整理しているが、GAS では `doGet` `doPost` の制約に合わせて action ベースで実装してよい。

例:

- `GET ?action=getMe`
- `GET ?action=getHome`
- `POST action=resolveDailySession`
- `POST action=getLearningSession`
- `POST action=resumeLearningSession`
- `POST action=createAnswerEvent`
- `GET ?action=getLearningHistorySummary`
- `GET ?action=listLearningHistorySessions`
- `GET ?action=listAdminLearners`
- `GET ?action=getAdminLearnerDetail`

重要なのはURL形状より責務分離を守ること。

## 10.1 GitHub Pages 配信前提

- LIFF URL には GitHub Pages の公開URLを設定する
- フロントで保持する設定値は公開可能なものだけに絞る
- GAS Web App URL、LIFF ID、公開Originなどは build 時または配信前生成ファイルで注入する
- Secret は GitHub Pages 側へ置かない

## 10.2 接続上の注意

- `GitHub Pages -> GAS Web App` のブラウザ直呼びは CORS 制約確認が必要
- そのため、MVPでも最初に `health` 相当の疎通確認を用意する
- もし直結が安定しない場合は、公開構成自体は維持したまま、接続方式だけを見直す
- ただし現時点では、公開面を GitHub Pages に置く方針は固定する

## 11. Tenant境界のGAS実装方針

- リクエストの `tenantId` は信用しない
- LINEユーザー識別またはセッション文脈から Membership を解決する
- Spreadsheet 読み取り時も `tenant_id` 条件を必ず通す
- `sheet.getDataRange().getValues()` の後に広く絞るのではなく、読み取りとフィルタ責務を repository に閉じる

## 12. AnswerEvent の冪等性

- クライアントが `idempotencyKey` を生成する
- GAS 側で保存前に `tenant_id + learning_session_id + idempotency_key` の重複を確認する
- 更新競合を減らすため、保存前後の狭い範囲で `LockService` 利用を検討する

## 13. 今日の5問の簡易実装

MVPでは高度な最適化をせず、次の優先順で出題してよい。

1. 課題の未回答問題
2. 復習対象
3. 未学習問題
4. 通常問題

最初はルールベースで十分とし、ロジックは usecase に閉じる。

## 14. 最初に作るべきGASユースケース

1. `getMe`
2. `getHome`
3. `resolveDailySession`
4. `getLearningSession`
5. `createAnswerEvent`
6. `getLearningHistorySummary`
7. `listAdminLearners`

## 15. 移行前提

この構成はMVPの簡易実装として採用する。
将来 RDB や別バックエンドへ移行する際も、次を維持していれば置き換えやすい。

- usecase と repository の分離
- Spreadsheet 依存を infra に閉じる
- Question / Grading / Session / AnswerEvent の責務分離
- Membership 基準の権限判定
