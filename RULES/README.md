# LearningPlatform Project RULES

この `RULES/` は、LINE Learning Platform プロジェクト固有の要件、設計意図、運用前提を管理する場所です。

共通原則の正本は `/Users/katsu/GitHub/RULES` にあり、このプロジェクトではそれを前提にしつつ、プロダクト固有の判断をここに残します。

## このフォルダの目的

- 要件定義をプロジェクト内の正本として管理する
- 実装前後で参照する前提、制約、優先順位を明文化する
- 開発指示の内容を、必要に応じて参照、改訂、廃止しやすい形へ整理する

## 運用方針

- 共通的な設計原則、責務分離、命名規則は `/Users/katsu/GitHub/RULES` を参照する
- この `RULES/` には、このプロダクトに閉じる要件、画面、権限、ドメイン前提、MVP範囲を残す
- 仕様変更時は、コードだけ先に変えず、先にこの `RULES/` を更新して差分理由を明記する
- 将来不要になった要件は削除せず、まずは廃止理由や置き換え先が分かるように改訂する

## ファイル構成

- `01_requirement_definition.md`
  プロダクト要件定義の正本。目的、利用者、権限、主要ドメイン、MVP範囲、非機能要件を管理する
- `02_mvp_information_architecture.md`
  MVPの画面、主要ユースケース、API/データ責務、実装優先順の整理
- `03_data_model_and_authorization.md`
  MVPの論理データモデル、Tenant境界、権限解決、冪等性、同期前提の整理
- `04_screen_flow_and_api_catalog.md`
  MVPの画面遷移、画面責務、主要API、実装フェーズの入口整理
- `05_er_and_table_draft.md`
  ER図、主要テーブル、制約、インデックス、集計系の下書き
- `06_gas_spreadsheet_mvp.md`
  GAS Web App と Spreadsheet を使うMVP簡易実装方針、シート責務、設定値、制約

## 参照元

- 共通 RULES: `/Users/katsu/GitHub/RULES/README.md`
- 元の開発指示: 2026-08-11 時点で受領した LINE Learning Platform 開発指示テキスト

## 改訂ルール

- 大きな変更は、少なくとも次を残す
  - 何を変えたか
  - なぜ変えたか
  - MVPへの影響
  - 将来拡張への影響
- `01_requirement_definition.md` は「何を作るか」の正本として扱う
- `02_mvp_information_architecture.md` は「どう分けて進めるか」の実務整理として扱う
- `03_data_model_and_authorization.md` は「どう壊れにくく持つか」の構造整理として扱う
- `04_screen_flow_and_api_catalog.md` は「どこから実装を始めるか」の導線整理として扱う
- `05_er_and_table_draft.md` は「どの保存構造で始めるか」の草案として扱う
- `06_gas_spreadsheet_mvp.md` は「このMVPをGASでどう成立させるか」の実装方針として扱う
