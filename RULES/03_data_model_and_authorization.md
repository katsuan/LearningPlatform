# MVPデータモデルと認可整理

## 1. この文書の目的

この文書は、MVPで必要な論理データモデルと、Tenant境界および権限解決の前提を固定するための資料です。
実装開始時にモデル名や責務がぶれないことを目的とします。

## 2. モデル設計の基本原則

- Platform権限とTenant権限を分離する
- Userの本人情報とTenant所属情報を分離する
- Tenantに属するデータは、原則として `tenantId` を持つ
- 学習単位はQuestionではなくLearningSessionを中心に扱う
- 回答結果の一次データはAnswerEventとして保存する
- 課金や機能差分はPlan名でなくFeature Entitlementで表現する

## 3. MVPの主要エンティティ

### 3.1 Platform系

#### PlatformAdmin

プラットフォーム運営者。
Tenantユーザー体系とは別管理とする。

主な項目:

- `platformAdminId`
- `email`
- `displayName`
- `status`
- `createdAt`
- `updatedAt`

#### Tenant

契約組織の最上位単位。

主な項目:

- `tenantId`
- `tenantKey`
- `name`
- `status`
- `timezone`
- `locale`
- `planCode`
- `createdAt`
- `updatedAt`

#### FeatureEntitlement

Tenantごとの機能利用条件。

主な項目:

- `featureEntitlementId`
- `tenantId`
- `featureKey`
- `enabled`
- `limitValue`
- `periodType`
- `effectiveFrom`
- `effectiveTo`

### 3.2 ユーザー / 所属 / 権限

#### User

利用者本人の共通主体。
複数Tenant所属を前提とする。

主な項目:

- `userId`
- `lineUserId`
- `displayName`
- `photoUrl`
- `status`
- `createdAt`
- `updatedAt`

#### Membership

UserとTenantの関係を表す所属モデル。

主な項目:

- `membershipId`
- `tenantId`
- `userId`
- `role`
- `status`
- `joinedAt`
- `lastAccessedAt`

#### Group

Tenant内の学習者分類単位。

主な項目:

- `groupId`
- `tenantId`
- `name`
- `status`
- `createdAt`
- `updatedAt`

#### GroupMembership

GroupとMembershipの関係。
User直結ではなくMembership基準にしてTenant境界を明確にする。

主な項目:

- `groupMembershipId`
- `tenantId`
- `groupId`
- `membershipId`
- `createdAt`

### 3.3 LINE / LIFF 接続

#### LineConnection

Tenant単位のLINE接続設定。

主な項目:

- `lineConnectionId`
- `tenantId`
- `channelId`
- `channelSecretRef`
- `channelAccessTokenRef`
- `liffId`
- `status`
- `createdAt`
- `updatedAt`

`channelSecret` や `channelAccessToken` は平文値ではなく、Secret参照名や安全な保管先識別子を持つ前提にする。

### 3.4 教材

#### Course

教材の最上位単位。

主な項目:

- `courseId`
- `tenantId`
- `sourceType`
- `title`
- `description`
- `status`
- `createdAt`
- `updatedAt`

`sourceType` は `COMMON` または `TENANT` を想定する。

#### Category

コース内の大分類。

主な項目:

- `categoryId`
- `tenantId`
- `courseId`
- `name`
- `sortOrder`

#### Unit

問題を束ねる学習単元。

主な項目:

- `unitId`
- `tenantId`
- `courseId`
- `categoryId`
- `name`
- `sortOrder`

#### Question

問題本体。

主な項目:

- `questionId`
- `tenantId`
- `courseId`
- `categoryId`
- `unitId`
- `questionType`
- `gradingType`
- `questionText`
- `explanation`
- `difficulty`
- `status`
- `requiredFeatureKey`
- `createdAt`
- `updatedAt`

#### QuestionOption

選択肢を持つ問題形式用の子要素。

主な項目:

- `questionOptionId`
- `tenantId`
- `questionId`
- `optionKey`
- `label`
- `sortOrder`

#### QuestionGradingConfig

採点設定の詳細。
Question Typeと採点設定を分離するための構造。

主な項目:

- `questionGradingConfigId`
- `tenantId`
- `questionId`
- `answerSchema`
- `correctAnswerPayload`
- `tolerance`
- `maxScore`
- `rubricPayload`

### 3.5 学習実行

#### LearningSession

学習実行の中心単位。

主な項目:

- `learningSessionId`
- `tenantId`
- `userId`
- `membershipId`
- `courseId`
- `assignmentId`
- `sessionType`
- `status`
- `questionCount`
- `startedAt`
- `completedAt`
- `expiresAt`
- `createdAt`

#### SessionQuestion

SessionとQuestionの関連。
表示順、出題時点スナップショット、再取得用参照を持てるようにする。

主な項目:

- `sessionQuestionId`
- `tenantId`
- `learningSessionId`
- `questionId`
- `displayOrder`
- `questionSnapshotPayload`
- `createdAt`

#### AnswerEvent

回答行動の一次データ。

主な項目:

- `answerEventId`
- `tenantId`
- `learningSessionId`
- `sessionQuestionId`
- `questionId`
- `userId`
- `membershipId`
- `attemptNo`
- `idempotencyKey`
- `answerPayload`
- `isCorrect`
- `score`
- `elapsedMs`
- `hintUsed`
- `explanationViewed`
- `answeredAt`
- `syncedAt`

#### LearningProfile

将来の最適化と分析のための集約モデル。

主な項目:

- `learningProfileId`
- `tenantId`
- `userId`
- `membershipId`
- `courseId`
- `categoryId`
- `unitId`
- `accuracyRate`
- `recentAccuracyRate`
- `averageElapsedMs`
- `reviewCount`
- `lastLearnedAt`
- `updatedAt`

### 3.6 課題 / 管理

#### Assignment

管理者が付与する課題。

主な項目:

- `assignmentId`
- `tenantId`
- `courseId`
- `targetType`
- `targetId`
- `questionCount`
- `startAt`
- `dueAt`
- `createdByMembershipId`
- `status`
- `createdAt`
- `updatedAt`

`targetType` は `USER` または `GROUP` を想定する。
MVPでは複数ターゲット一括保持より、Assignmentごとに対象を明示する形を優先してよい。

## 4. Tenant境界のルール

### 4.1 基本ルール

- Tenant配下モデルはすべて `tenantId` で境界を持つ
- 認証後に確定した `tenantId` を、以後のRepositoryアクセス条件へ必ず含める
- `id` 単独検索を避け、`tenantId + id` を基本キーとして扱う

### 4.2 越境防止

- API入力の `tenantId` をそのまま信用しない
- LIFF / 認証 / Membershipから確定したTenant文脈を正とする
- 管理画面でも、一覧条件と詳細取得条件の双方でTenant制約をかける

### 4.3 共通教材との両立

共通教材を扱う場合でも、取得条件は曖昧にしない。

例:

- `tenantId = currentTenantId` の教材
- または `sourceType = COMMON` の教材

この判定はRepositoryまたは専用Query層に閉じ込め、画面ごとに分散させない。

## 5. 権限解決の考え方

### 5.1 判定単位

権限は `User` ではなく、Tenant文脈を持つ `Membership` を基準に判定する。

### 5.2 MVPの権限制御

- `LEARNER`
  - 自分のSession、自分の履歴、自分に紐づく課題のみ参照
- `INSTRUCTOR`
  - 担当対象の閲覧と課題系操作
- `TENANT_ADMIN`
  - ユーザー、グループ、教材、課題の管理
- `TENANT_OWNER`
  - Tenant設定、契約系操作

### 5.3 SYSTEM_ADMIN

SYSTEM_ADMINはTenant権限と混在させない。
運営者向け機能は別の認証経路または別アプリケーション境界を推奨する。

## 6. 冪等性と同期

### 6.1 AnswerEvent

LIFFの通信断や再送を考慮し、`AnswerEvent` には `idempotencyKey` を持たせる。

推奨ルール:

- 1回答送信につきクライアントで一意キーを生成
- サーバーは `tenantId + learningSessionId + idempotencyKey` で重複登録を防ぐ

### 6.2 ローカル保存

クライアント側では少なくとも次を保持できるようにする。

- 現在の `learningSessionId`
- 現在の問題位置
- 未同期のAnswerEvent
- 一時的な回答状態

## 7. MVPで後回しにしてよい点

- 詳細な監査ログ専用モデル
- 複雑な組織階層モデル
- AI採点用の詳細評価モデル
- 従量課金専用の請求モデル

ただし、名称や主キーの置き方が将来追加を阻害しないようにする。

## 8. 実装開始時の最小セット

最初の実装では、少なくとも次を先に固定すると進めやすい。

1. Tenant
2. User
3. Membership
4. FeatureEntitlement
5. Course / Category / Unit / Question
6. LearningSession
7. SessionQuestion
8. AnswerEvent
9. LineConnection
