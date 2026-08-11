# MVP画面遷移とAPIカタログ

## 1. この文書の目的

この文書は、MVPの画面責務とAPI責務を対応づけ、実装の入口を明確にするための資料です。
画面とAPIのどちらから着手しても、責務が混ざらないように整理します。

## 2. 想定システム境界

- LINE: 起点、通知、LIFF起動導線
- LIFF Frontend: 学習UI、管理UI、ローカル保存、表示制御
- Backend API: 認証文脈解決、権限判定、学習ロジック、永続化
- Data Store: Tenant配下データ、学習履歴、設定

## 3. 画面遷移の最小構成

### 3.1 学習者導線

1. LINE リッチメニュー
2. LIFF Home
3. 今日の5問 開始 / 再開
4. 設問回答
5. 即時採点 / 解説表示
6. 次問題へ遷移
7. セッション完了
8. 学習履歴確認

### 3.2 管理者導線

1. 管理HomeまたはDashboard
2. 学習者一覧
3. 学習者詳細

MVPでは、管理者向けの教材管理や課題管理は画面定義だけ先に置き、実装対象を絞ってよい。

## 4. 画面ごとの責務

### 4.1 Home

役割:

- ユーザー識別状態を確認する
- 今日の5問への主導線を見せる
- 再開可能Sessionの有無を見せる
- 学習履歴や設定への導線を置く

必要データ:

- 現在のMembership
- 今日のSession概要
- 再開状態
- 学習サマリ

### 4.2 今日の5問

役割:

- Session取得または生成
- 設問表示
- 回答送信
- 採点結果表示
- 解説表示
- 次問題への遷移

必要データ:

- Sessionメタ情報
- 出題済みQuestion一覧
- 現在問題
- 回答状態
- 未同期イベント状態

### 4.3 学習履歴

役割:

- 累計問題数
- 正答率
- 最終学習日時
- 直近Session一覧

を確認できるようにする。

### 4.4 学習者一覧

役割:

- Tenant配下学習者の一覧表示
- 基本サマリ表示
- 学習者詳細への導線

### 4.5 学習者詳細

役割:

- 個別学習者の基本状況表示
- 直近学習状況の確認

MVPでは詳細分析は不要。

## 5. APIカタログ

エンドポイント名は仮置きであり、ここでは責務を固定する。

### 5.1 認証 / コンテキスト

#### `GET /api/me`

目的:

- LIFFログイン済みユーザー情報を返す
- 現在のTenant文脈とMembershipを返す
- 利用可能な基本権限を返す

返却例の責務:

- `user`
- `membership`
- `tenant`
- `entitlements`

### 5.2 Home表示用

#### `GET /api/home`

目的:

- Home画面に必要なサマリ情報を返す

主な返却内容:

- 今日のSession概要
- 再開可能Session
- 学習サマリ

### 5.3 学習セッション

#### `POST /api/learning-sessions/daily:resolve`

目的:

- 今日の5問Sessionを取得または新規生成する

備考:

- 既存Sessionがあれば再利用
- なければ生成

#### `GET /api/learning-sessions/:sessionId`

目的:

- Session詳細と出題一覧を返す

#### `POST /api/learning-sessions/:sessionId/resume`

目的:

- 再開時の現在位置や未完了状態を返す

### 5.4 回答

#### `POST /api/answer-events`

目的:

- 回答を受け取り採点する
- AnswerEventを永続化する
- 冪等性を保証する

入力責務:

- `learningSessionId`
- `sessionQuestionId`
- `idempotencyKey`
- `answerPayload`
- `elapsedMs`

返却責務:

- 正誤
- スコア
- 解説
- 次問題へ進めるか

### 5.5 学習履歴

#### `GET /api/learning-history/summary`

目的:

- 学習者本人の累計学習サマリを返す

#### `GET /api/learning-history/sessions`

目的:

- 直近Session一覧を返す

### 5.6 管理者向け

#### `GET /api/admin/learners`

目的:

- Tenant配下学習者の一覧を返す

主な返却内容:

- 学習者名
- 累計問題数
- 正答率
- 最終学習日時

#### `GET /api/admin/learners/:membershipId`

目的:

- 個別学習者の基本情報と直近サマリを返す

## 6. 画面とAPIの対応

- Home
  - `GET /api/me`
  - `GET /api/home`
- 今日の5問
  - `POST /api/learning-sessions/daily:resolve`
  - `GET /api/learning-sessions/:sessionId`
  - `POST /api/answer-events`
- 学習履歴
  - `GET /api/learning-history/summary`
  - `GET /api/learning-history/sessions`
- 学習者一覧
  - `GET /api/admin/learners`
- 学習者詳細
  - `GET /api/admin/learners/:membershipId`

## 7. 実装フェーズの切り方

### Phase 1

- `GET /api/me`
- Tenant / User / Membership 解決
- Homeの最低限表示

### Phase 2

- 今日の5問Session取得 / 生成
- Session詳細取得
- Question表示の最低限UI

### Phase 3

- 回答送信
- 自動採点
- 解説表示
- AnswerEvent保存

### Phase 4

- 途中保存
- 再開
- 未同期イベント再送

### Phase 5

- 学習履歴サマリ
- 管理者向け学習者一覧

## 8. 実装時の注意

- 画面が直接採点ロジックを持たない
- API Controllerが問題選定ロジックを抱え込まない
- `tenantId` をクライアント入力のまま信用しない
- 再開や再送を考慮して、回答APIは冪等にする
- Home用集計と履歴画面用集計を無理に1APIへ詰め込まない

## 9. 次の実装資料候補

ここまで定義したら、次は次のどれかに着手しやすい。

- ER図
- OpenAPIたたき台
- LIFF画面ワイヤー
- 認証とTenant解決シーケンス
