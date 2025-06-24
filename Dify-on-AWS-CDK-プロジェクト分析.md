# Dify on AWS CDK プロジェクト分析

## プロジェクト概要

このプロジェクトは、オープンソースのLLMアプリケーション開発プラットフォーム「Dify」をAWS上にホスティングするためのAWS CDK（Cloud Development Kit）コードです。コンテナベースのマイクロサービスアーキテクチャを採用し、スケーラブルで高可用性なDifyインスタンスをAWS上に構築します。

## /bin ディレクトリ

### [`bin/cdk.ts`](bin/cdk.ts:1)
**役割**: CDKアプリケーションのエントリーポイント

**主な機能**:
- 環境設定の定義（[`EnvironmentProps`](lib/environment-props.ts:4)）
- Difyのバージョン設定（デフォルト: 1.4.3）
- 複数スタックの初期化と連携
  - [`DifyOnAwsStack`](lib/dify-on-aws-stack.ts:26): メインのDifyインフラストラクチャ
  - [`UsEast1Stack`](lib/us-east-1-stack.ts:14): CloudFront用のus-east-1リージョンリソース（**オプション**）

**作成される主要リソース**:
- メインアプリケーションスタック（指定リージョン）
- CloudFront用証明書・WAFスタック（us-east-1）- **条件付きで作成**

### [`bin/deploy.ts`](bin/deploy.ts:1)
**役割**: CloudShellからのデプロイメント用スタック

**主な機能**:
- CodeBuildプロジェクトの作成
- S3バケットでのソースコード管理
- CloudWatchログ設定

**作成される主要リソース**:
- CodeBuild プロジェクト
- S3 ソースバケット
- CloudWatch ロググループ

## /lib ディレクトリ

### メインスタック

#### [`lib/dify-on-aws-stack.ts`](lib/dify-on-aws-stack.ts:26)
**役割**: Difyアプリケーション全体のメインスタック

**作成される主要リソース**:
- ECS Fargate クラスター
- Aurora PostgreSQL データベース（pgvector拡張付き）
- ElastiCache Valkey（Redis互換）クラスター
- Application Load Balancer（ALB）
- S3 ストレージバケット
- CloudFront ディストリビューション（**オプション**）
- Dify API・Webサービス

#### [`lib/us-east-1-stack.ts`](lib/us-east-1-stack.ts:14)
**役割**: CloudFront用のus-east-1リージョン専用リソース（**オプション**）

**作成される主要リソース**:
- SSL/TLS証明書（ACM）- **条件付きで作成**
- Web Application Firewall（WAF）- **条件付きで作成**

#### [`lib/environment-props.ts`](lib/environment-props.ts:4)
**役割**: 設定パラメータの型定義

**主要な設定項目**:
- AWSリージョン・アカウント設定
- ネットワーク設定（VPC、サブネット、NAT）
- ドメイン・SSL設定
- データベース・キャッシュ設定
- コンテナイメージ設定
- セキュリティ設定

## オプション機能の制御方法

### 1. CloudFrontを無効にする

**修正箇所**: [`bin/cdk.ts`](bin/cdk.ts:8)の`props`オブジェクト

```typescript
export const props: EnvironmentProps = {
  awsRegion: 'us-west-2',
  awsAccount: process.env.CDK_DEFAULT_ACCOUNT!,
  difyImageTag: '1.4.3',
  difyPluginDaemonImageTag: '0.1.2-local',
  
  // CloudFrontを無効にする
  useCloudFront: false,  // この行を追加
};
```

**影響**:
- [`UsEast1Stack`](lib/us-east-1-stack.ts:14)が作成されない
- CloudFront、WAF、ACM証明書が作成されない
- ALBが直接インターネットに公開される

### 2. WAFを無効にする

**修正箇所**: [`bin/cdk.ts`](bin/cdk.ts:8)の`props`オブジェクト

```typescript
export const props: EnvironmentProps = {
  // 既存の設定...
  
  // IP制限を削除してWAFを無効にする
  // allowedIPv4Cidrs: ['1.1.1.1/30'],  // この行をコメントアウト
  // allowedIPv6Cidrs: ['2001:db8:0:7::5/64'],  // この行をコメントアウト
};
```

**影響**:
- [`UsEast1Stack`](lib/us-east-1-stack.ts:14)内でWAFが作成されない（[35行目](lib/us-east-1-stack.ts:35)の条件により）
- すべてのIPアドレスからアクセス可能になる

### 3. ACM証明書を無効にする

**修正箇所**: [`bin/cdk.ts`](bin/cdk.ts:8)の`props`オブジェクト

```typescript
export const props: EnvironmentProps = {
  // 既存の設定...
  
  // ドメイン名を削除してACM証明書を無効にする
  // domainName: 'example.com',  // この行をコメントアウト
};
```

**影響**:
- [`UsEast1Stack`](lib/us-east-1-stack.ts:14)内でACM証明書が作成されない（[21行目](lib/us-east-1-stack.ts:21)の条件により）
- [`Alb`](lib/constructs/alb.ts:50)でHTTPS証明書が作成されない（[69行目](lib/constructs/alb.ts:69)の条件により）
- HTTP接続のみとなる

### 4. 完全にシンプルな構成にする

**修正箇所**: [`bin/cdk.ts`](bin/cdk.ts:8)の`props`オブジェクト

```typescript
export const props: EnvironmentProps = {
  awsRegion: 'us-west-2',
  awsAccount: process.env.CDK_DEFAULT_ACCOUNT!,
  difyImageTag: '1.4.3',
  difyPluginDaemonImageTag: '0.1.2-local',
  
  // オプション機能をすべて無効にする
  useCloudFront: false,        // CloudFront無効
  // domainName: undefined,    // カスタムドメイン無効（デフォルト）
  // allowedIPv4Cidrs: undefined, // IP制限無効（デフォルト）
  // allowedIPv6Cidrs: undefined, // IP制限無効（デフォルト）
};
```

## 条件分岐の仕組み

### [`bin/cdk.ts`](bin/cdk.ts:27-37)での条件分岐
```typescript
let virginia: UsEast1Stack | undefined = undefined;
if ((props.useCloudFront ?? true) && (props.domainName || props.allowedIPv4Cidrs || props.allowedIPv6Cidrs)) {
  virginia = new UsEast1Stack(app, `DifyOnAwsUsEast1Stack${props.subDomain ? `-${props.subDomain}` : ''}`, {
    // UsEast1Stackの設定
  });
}
```

**条件**:
- `useCloudFront`が`true`（デフォルト）**かつ**
- 以下のいずれかが設定されている場合のみ[`UsEast1Stack`](lib/us-east-1-stack.ts:14)を作成
  - `domainName`
  - `allowedIPv4Cidrs`
  - `allowedIPv6Cidrs`

### [`lib/dify-on-aws-stack.ts`](lib/dify-on-aws-stack.ts:116-133)での条件分岐
```typescript
const alb = useCloudFront
  ? new AlbWithCloudFront(this, 'Alb', {
      // CloudFront付きALBの設定
    })
  : new Alb(this, 'Alb', {
      // 通常のALBの設定
    });
```

**条件**:
- `useCloudFront`が`true`の場合: [`AlbWithCloudFront`](lib/constructs/alb-with-cloudfront.ts:1)を使用
- `useCloudFront`が`false`の場合: 通常の[`Alb`](lib/constructs/alb.ts:50)を使用

## インフラストラクチャコンストラクト

### [`lib/constructs/vpc.ts`](lib/constructs/vpc.ts:5)
**役割**: VPCネットワークの構築

**作成される主要リソース**:
- VPC（新規作成または既存VPC利用）
- パブリック・プライベートサブネット
- NAT Gateway または NAT Instance
- VPCエンドポイント（分離環境用）

### [`lib/constructs/postgres.ts`](lib/constructs/postgres.ts:26)
**役割**: PostgreSQLデータベースの構築

**作成される主要リソース**:
- Aurora PostgreSQL Serverless v2 クラスター
- データベース認証情報（Secrets Manager）
- pgvector拡張機能の自動セットアップ
- Bastionホスト（オプション）

### [`lib/constructs/redis.ts`](lib/constructs/redis.ts:13)
**役割**: Redisキャッシュ・メッセージキューの構築

**作成される主要リソース**:
- ElastiCache Valkey レプリケーショングループ
- 認証トークン（Secrets Manager）
- セキュリティグループ
- SSMパラメータ（Broker URL）

### [`lib/constructs/alb.ts`](lib/constructs/alb.ts:50)
**役割**: Application Load Balancerの構築

**作成される主要リソース**:
- Application Load Balancer
- SSL/TLS証明書（ACM）- **条件付き**
- Route53 Aレコード - **条件付き**
- セキュリティグループ

### Difyサービスコンストラクト

#### [`lib/constructs/dify-services/api.ts`](lib/constructs/dify-services/api.ts:46)
**役割**: Dify APIサービスの構築

**作成される主要リソース**:
- ECS Fargate タスク定義
- 複数コンテナ構成:
  - **Main**: Dify APIサーバー
  - **Worker**: バックグラウンドタスク処理
  - **Sandbox**: コード実行環境
  - **PluginDaemon**: プラグイン管理
  - **ExternalKnowledgeBaseAPI**: 外部ナレッジベース連携

#### [`lib/constructs/dify-services/web.ts`](lib/constructs/dify-services/web.ts:26)
**役割**: Dify Webフロントエンドの構築

**作成される主要リソース**:
- ECS Fargate タスク定義
- Next.js Webアプリケーション

## アーキテクチャ特徴

### 高可用性設計
- **マルチAZ配置**: データベース、キャッシュ、コンテナサービス
- **自動スケーリング**: Aurora Serverless v2、ECS Fargate
- **ロードバランシング**: ALB による負荷分散

### セキュリティ
- **ネットワーク分離**: VPC、プライベートサブネット
- **暗号化**: 保存時・転送時データ暗号化
- **認証**: IAM ロール、Secrets Manager
- **WAF**: Web Application Firewall（**オプション**）

### コスト最適化オプション
- **NAT Instance**: NAT Gateway の代替（t4g.nano）
- **Aurora ゼロスケーリング**: 非使用時の自動停止
- **Fargate Spot**: スポットインスタンス利用
- **Redis シングルAZ**: 開発環境向け
- **CloudFront無効**: CDN不要な場合のコスト削減

## デプロイメント設定例

### 最小構成（開発環境向け）
```typescript
export const props: EnvironmentProps = {
  awsRegion: 'us-west-2',
  awsAccount: process.env.CDK_DEFAULT_ACCOUNT!,
  difyImageTag: '1.4.3',
  difyPluginDaemonImageTag: '0.1.2-local',
  
  // コスト削減オプション
  useCloudFront: false,
  isRedisMultiAz: false,
  useNatInstance: true,
  enableAuroraScalesToZero: true,
  useFargateSpot: true,
};
```

### 本番環境向け（カスタムドメイン付き）
```typescript
export const props: EnvironmentProps = {
  awsRegion: 'us-west-2',
  awsAccount: process.env.CDK_DEFAULT_ACCOUNT!,
  difyImageTag: '1.4.3',
  difyPluginDaemonImageTag: '0.1.2-local',
  
  // 本番環境設定
  domainName: 'example.com',
  subDomain: 'dify',
  useCloudFront: true,
  allowedIPv4Cidrs: ['0.0.0.0/0'], // 必要に応じて制限
  setupEmail: true,
};
```

### 閉域ネットワーク向け
```typescript
export const props: EnvironmentProps = {
  awsRegion: 'ap-northeast-1',
  awsAccount: '123456789012',
  difyImageTag: '1.4.3',
  difyPluginDaemonImageTag: '0.1.2-local',
  
  // 閉域ネットワーク設定
  allowedIPv4Cidrs: ['10.0.0.0/16'],
  useCloudFront: false,        // CloudFront無効
  internalAlb: true,           // 内部ALB
  vpcIsolated: true,           // インターネット接続なし
  customEcrRepositoryName: 'dify-images',
};
```

このCDKプロジェクトにより、用途に応じて柔軟にカスタマイズされたDifyインスタンスをAWS上に構築できます。