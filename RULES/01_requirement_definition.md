# LINE Learning Platform 要件定義

## 1. この文書の位置づけ

この文書は、LINE Learning Platform のプロダクト要件定義の正本です。

2026-08-11 時点で受領した開発指示を基に、実装と運用で参照しやすいよう再整理しています。
設計や実装で変更提案が必要な場合は、黙って仕様変更せず、この文書へ変更案と影響を反映して扱います。

## 2. 目的

LINEを入口とし、学習体験を学習画面内で完結させるマルチテナント型学習プラットフォームを構築する。

対象は個人学習に閉じず、将来的に塾、企業、ケアセンター、教育施設などが、自組織の既存LINE公式アカウントへ学習機能を追加できるSaaSへ拡張する。

MVPでは過剰実装を避けるが、将来のマルチテナント、法人利用、課金、AI採点、ホワイトラベルを妨げない構造を維持する。

## 3. スコープと基本思想

### 3.1 役割分離

- LINE: 入口と通知
- 学習画面: 学習UIと管理UI
- Backend: 認証、学習ロジック、データ管理
- Learning Engine: 問題選定、採点、個人最適化
- Tenant: 契約組織単位
- Feature Entitlement: 契約による機能制御

### 3.2 体験方針

- 問題演習は原則としてLINEチャットではなく学習画面内で完結させる
- LINEチャットを問題と回答のログで汚さない
- 「今日の5問」を中心体験として設計する
- 問題演習中の待ち時間を極力減らす
- LIFF は実装・配信方式として扱い、学習者向け表現では原則として前面に出さない

### 3.3 境界条件

- SYSTEM_ADMINはTenant側権限と分離する
- UserとTenantの関係はMembershipで管理する
- Tenant境界はAPI層だけでなくデータアクセス層でも保証する
- プラン名ではなくFeature Entitlementで機能制御する
- Question TypeとGrading Typeは分離する

## 4. 想定利用者

同一システムで次を扱える構造とする。

- 個人
- Family
- 学習塾
- 企業
- ケアセンター
- 教育施設

用途ごとに別システムを作らず、Tenantで分離する。
1人のUserが複数Tenantへ所属できることを前提とする。

## 5. 権限要件

### 5.1 Platform権限

SYSTEM_ADMINはプラットフォーム運営権限であり、Tenantユーザー体系に含めない。

SYSTEM_ADMINに想定する責務:

- Tenant作成、停止、状態確認
- 契約管理、Plan管理、Feature管理
- LINE接続状態確認
- システム利用量確認
- エラーログ確認
- 共通教材管理
- システム設定

Tenant内部を参照する必要がある場合も、通常権限としての無条件アクセスではなく、明示的なサポート操作と監査ログを前提にする。

### 5.2 Tenant権限

- `TENANT_OWNER`: 契約、組織全体設定、管理者管理
- `TENANT_ADMIN`: ユーザー管理、グループ管理、教材管理、課題管理
- `INSTRUCTOR`: 担当学習者確認、課題、将来的な記述採点
- `LEARNER`: 問題演習、自分の履歴と成績確認

## 6. LINE / LIFF 要件

### 6.1 LINE連携

- 既存LINE公式アカウントへの導入を可能にする
- Tenant単位で `LineConnection` を管理する
- 新しいLINE公式アカウント作成を必須にしない
- 機密情報はDBへ平文保存せず、Secret Manager等へ移行可能な構造にする

想定導線:

既存LINE公式アカウント -> リッチメニュー「学習する」 -> Tenant固有の学習画面 -> Learning Platform

### 6.2 学習画面

実装上は LIFF を利用できるが、学習者や一般利用者向けの文言では「LIFF」を主語にせず、「学習画面」「管理画面」など目的ベースの表現を優先する。

学習者向けの想定画面:

- Home
- 今日の5問
- 追加学習
- 復習
- 課題
- コース
- 学習履歴
- 成績
- 設定

管理者向けの想定画面:

- Dashboard
- 学習者一覧
- 学習者詳細
- Group管理
- Assignment管理
- 教材管理
- 採点
- Tenant設定

MVPでは必要最低限に絞るが、後続画面を増やせる構造を前提にする。

## 7. ドメイン要件

### 7.1 マルチテナント

Tenant Aの利用者は、Tenant Bの次のデータへアクセスできてはならない。

- User
- Group
- Question
- Course
- Assignment
- LearningSession
- AnswerEvent
- Analytics

Tenant境界は認可だけでなく、RepositoryやData Accessの設計でも保証する。

### 7.2 学習セッション

学習はQuestion単体ではなく `LearningSession` 単位で管理する。

想定するSession Type:

- `DAILY`
- `EXTRA`
- `REVIEW`
- `ASSIGNMENT`
- `COURSE`
- `EXAM`

「今日の5問」「もう5問」「復習」「課題20問」を同じモデルで扱えることを要件とする。

### 7.3 今日の5問

「今日の5問」はサービスの中心機能とする。
ユーザーごとにDaily Learning Sessionを生成し、問題選定ロジックは将来的に交換可能な構造で分離する。

将来考慮する選定要素:

1. 管理者指定課題
2. 復習タイミング
3. 苦手分野
4. 現在の習熟度
5. 未学習分野
6. 通常問題

MVPではルールベースでよいが、UIやControllerに選定ロジックを直書きしない。

### 7.4 問題と採点

MVP必須の問題形式:

- `TEXT_EXACT`
- `SINGLE_CHOICE`
- `MULTIPLE_CHOICE`
- `TRUE_FALSE`
- `NUMBER`
- `ORDER`
- `MATCHING`

これらはサーバーAIに依存せず自動採点可能とする。

将来拡張する問題形式:

- `SHORT_ANSWER`
- `LONG_ANSWER`

採点方式:

- `AUTO`
- `MANUAL`
- `AI`
- `AI_WITH_REVIEW`

Question TypeとGrading Typeは別概念として扱い、採点方式追加時にQuestionモデル全体を書き換えなくてよい構造にする。

### 7.5 教材構造

基本構造は次を前提とする。

`Course -> Category -> Unit -> Question`

共通教材とTenant独自教材を扱える構造にする。
MVPの登録手段は手入力またはSeedデータでよい。

### 7.6 学習履歴と分析

`AnswerEvent` を学習行動の一次データとして記録する。

記録対象の例:

- 回答内容
- 正誤
- スコア
- 回答時間
- ヒント利用
- 解説閲覧
- 回答時刻
- 同期時刻

`LearningProfile` はカテゴリや単元ごとの習熟状態を保持できる構造を想定する。
MVPでは高度な最適化は不要だが、将来の問題選定や分析へ利用できるようにする。

### 7.7 課題

管理者は学習者またはGroupへ `Assignment` を設定できるようにする。

想定項目:

- 対象種別と対象ID
- 教材
- 問題数
- 開始日時
- 締切日時
- 作成者
- ステータス

AssignmentはLearningSessionとの関連を考慮して設計する。

## 8. プランと機能制御

料金プラン名を直接アプリの条件分岐へ埋め込まない。
契約による機能差分はFeature Entitlementで表現する。

例:

- `dailyQuestions`
- `historyDays`
- `managedUsers`
- `groupLimit`
- `assignments`
- `customContent`
- `manualGrading`
- `aiGradingMonthly`
- `analyticsLevel`
- `customLine`
- `whiteLabel`
- `apiAccess`

MVPで完全な課金導線は不要だが、将来のToC/B2Bマネタイズを妨げない構造とする。

## 9. MVP実装範囲

MVPで優先する項目:

1. マルチテナント基盤
2. User / Membership
3. SYSTEM_ADMIN分離
4. Tenant権限
5. LINE / LIFFユーザー識別
6. LIFF学習画面
7. 今日の5問
8. LearningSession
9. 7種類の自動採点問題
10. 即時採点
11. 解説表示
12. AnswerEvent
13. 回答途中保存
14. キャッシュ
15. Prefetch可能な構造
16. 基本学習履歴
17. 学習問題数
18. 正答率
19. 管理者用学習者一覧
20. Feature Entitlement

MVPで完全実装しない項目:

- AI採点
- 記述問題
- PDF教材生成
- AI問題生成
- 高度な個人最適化
- SSO
- API公開
- Enterprise機能
- 完全なホワイトラベル
- 本番課金

## 10. 非機能要件

### 10.1 Security

- Tenant間データ分離
- LINE認証情報の安全な管理
- 権限チェック
- SYSTEM_ADMIN分離
- API入力Validation
- 不正なTenant ID指定による越境防止

### 10.2 Performance

- Daily Sessionキャッシュ
- Prefetch
- Local State
- API呼び出し最小化
- 問題遷移時の待ち時間最小化

### 10.3 Reliability

- AnswerEventの重複登録防止
- 通信切断からの復旧
- Session途中再開
- 同期失敗時のRetry

### 10.4 Maintainability

- UI
- Domain
- Application
- Infrastructure

上記の責務を可能な範囲で分離する。
過度な抽象化は避けるが、Business LogicをControllerやUIへ直接集中させない。

## 11. 将来拡張要件

将来的にTenant単位で次を変更可能にする。

- 組織名
- ロゴ
- LIFFテーマ
- 表示文言
- LINE通知文
- 教材

また、将来の拡張候補として以下を考慮する。

- AI一次採点とレビュー
- AI教材生成
- 詳細分析
- 保護者レポート
- API連携
- SSO
- 多階層組織
- 監査ログ

MVPでは未実装でも、追加時に大幅な作り直しが発生しないことを要件とする。

## 12. 実装判断時の最重要原則

次は設計判断で崩さない。

1. SYSTEM_ADMINとTenantユーザーを分離する
2. UserとTenant所属をMembershipで管理する
3. Tenant境界を全データアクセスで保証する
4. 問題演習はLIFF完結を基本とする
5. Question TypeとGrading Typeを分離する
6. 学習をLearningSession単位で扱う
7. 回答履歴をAnswerEventとして蓄積する
8. プラン名ではなくFeature Entitlementで機能制御する
9. 「今日の5問」を中心体験とする
10. 問題演習中のローディングを極力発生させない
11. 既存LINE公式アカウントへ後付けできる構造を維持する
12. 将来拡張を守りつつ、MVPで過剰実装しない
