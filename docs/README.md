# Dify on AWS CDK ドキュメント

このディレクトリには、Dify on AWS CDKプロジェクトに関する詳細な分析・説明ドキュメントが含まれています。

## ドキュメント一覧

### 1. [Dify-on-AWS-CDK-プロジェクト分析.md](./Dify-on-AWS-CDK-プロジェクト分析.md)
**概要**: DifyをAWS上にホスティングするためのCDKプロジェクト全体の詳細分析

**主な内容**:
- `/bin`と`/lib`配下のモジュール説明
- オプション機能（CloudFront、WAF、ACM証明書）の制御方法
- 用途別のデプロイメント設定例
- 作成されるIAMロール・ポリシーの一覧
- npmスクリプトの詳細解説

### 2. [PostgreSQL-Construct-詳細分析.md](./PostgreSQL-Construct-詳細分析.md)
**概要**: `lib/constructs/postgres.ts`の詳細分析

**主な内容**:
- Aurora PostgreSQL Serverless v2の設定詳細
- pgvector拡張機能の自動セットアップ
- Bastionホストの構成
- 依存関係管理とクエリ実行制御
- セキュリティ・コスト最適化の仕組み

### 3. [社内AWS環境対応-修正点一覧.md](./社内AWS環境対応-修正点一覧.md)
**概要**: 社内AWS環境での運用に向けた修正点と対応策

**主な内容**:
- リージョン設定の変更（東京リージョン対応）
- タグ付与による統一的なリソース管理
- セキュリティ・ガバナンス要件への対応

