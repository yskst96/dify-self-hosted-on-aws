# PostgreSQL Construct 詳細分析

## 概要

[`lib/constructs/postgres.ts`](lib/constructs/postgres.ts:26)は、DifyアプリケーションのためのAurora PostgreSQL Serverless v2データベースクラスターを構築するCDKコンストラクトです。pgvector拡張機能を含む高可用性・自動スケーリング対応のデータベース環境を提供します。

## クラス構造

### PostgresProps インターフェース

```typescript
export interface PostgresProps {
  vpc: IVpc;                    // デプロイ先VPC
  createBastion?: boolean;      // Bastionホスト作成フラグ（デフォルト: false）
  scalesToZero: boolean;        // ゼロスケーリング有効化フラグ
}
```

### Postgres クラス

```typescript
export class Postgres extends Construct implements IConnectable
```

**実装インターフェース**:
- `IConnectable`: ネットワーク接続管理機能を提供

**パブリックプロパティ**:
- `connections: Connections` - セキュリティグループ管理
- `cluster: rds.DatabaseCluster` - Aurora PostgreSQLクラスター
- `secret: ISecret` - データベース認証情報
- `databaseName: string` - メインデータベース名（'main'）
- `pgVectorDatabaseName: string` - pgvector用データベース名（'pgvector'）

## 主要機能

### 1. Aurora PostgreSQL Serverless v2 クラスター

#### エンジン設定
```typescript
const engine = rds.DatabaseClusterEngine.auroraPostgres({
  version: rds.AuroraPostgresEngineVersion.VER_15_7,
});
```
- **PostgreSQLバージョン**: 15.7
- **エンジンタイプ**: Aurora PostgreSQL

#### スケーリング設定
```typescript
serverlessV2MinCapacity: props.scalesToZero ? 0 : 0.5,
serverlessV2MaxCapacity: 2.0,
```
- **最小容量**: 0 ACU（ゼロスケーリング時）または 0.5 ACU
- **最大容量**: 2.0 ACU
- **自動スケーリング**: 負荷に応じて自動調整

#### セキュリティ設定
```typescript
writer: rds.ClusterInstance.serverlessV2(this.writerId, {
  autoMinorVersionUpgrade: true,
  publiclyAccessible: false,
}),
storageEncrypted: true,
```
- **パブリックアクセス**: 無効
- **ストレージ暗号化**: 有効
- **自動マイナーバージョンアップグレード**: 有効

### 2. データベース初期化

#### パラメータグループ
```typescript
parameterGroup: new rds.ParameterGroup(this, 'ParameterGroup', {
  engine,
  parameters: {
    idle_session_timeout: '60000',  // 60秒でアイドルセッション終了
  },
}),
```
- **アイドルセッションタイムアウト**: 60秒
- **目的**: Aurora Serverless v2の自動一時停止対応

#### ネットワーク設定
```typescript
vpcSubnets: vpc.selectSubnets({ 
  subnets: vpc.privateSubnets.concat(vpc.isolatedSubnets) 
}),
```
- **配置サブネット**: プライベート + 分離サブネット
- **セキュリティ**: インターネットから直接アクセス不可

### 3. pgvector拡張機能の自動セットアップ

#### データベース・拡張機能作成
```typescript
this.runQuery(`CREATE DATABASE ${this.pgVectorDatabaseName};`, undefined);
this.runQuery('CREATE EXTENSION IF NOT EXISTS vector;', this.pgVectorDatabaseName);
```

**実行順序**:
1. `pgvector`データベースの作成
2. `vector`拡張機能のインストール

#### カスタムリソースによる実行
```typescript
private runQuery(sql: string, database: string | undefined) {
  const query = new AwsCustomResource(this, `Query${this.queries.length}`, {
    onUpdate: {
      service: 'rds-data',
      action: 'ExecuteStatement',
      parameters: {
        resourceArn: cluster.clusterArn,
        secretArn: cluster.secret!.secretArn,
        database: database,
        sql: sql,
      },
    },
  });
}
```

**特徴**:
- **RDS Data API使用**: サーバーレス環境での安全なSQL実行
- **シーケンシャル実行**: 依存関係を考慮した順次実行
- **待機機能**: Data API有効化まで60秒待機

### 4. Bastionホスト（オプション）

#### 作成条件
```typescript
if (props.createBastion) {
  const host = new ec2.BastionHostLinux(this, 'BastionHost', {
    vpc,
    machineImage: ec2.MachineImage.latestAmazonLinux2023({ 
      cpuType: ec2.AmazonLinuxCpuType.ARM_64 
    }),
    instanceType: ec2.InstanceType.of(ec2.InstanceClass.T4G, ec2.InstanceSize.NANO),
  });
}
```

**仕様**:
- **OS**: Amazon Linux 2023 (ARM64)
- **インスタンスタイプ**: t4g.nano
- **ストレージ**: 8GB EBS（暗号化済み）

#### 管理コマンド出力
```typescript
new CfnOutput(this, 'PortForwardCommand', {
  value: `aws ssm start-session --region ${Stack.of(this).region} --target ${
    host.instanceId
  } --document-name AWS-StartPortForwardingSessionToRemoteHost --parameters '{"portNumber":["${
    cluster.clusterEndpoint.port
  }"], "localPortNumber":["${cluster.clusterEndpoint.port}"], "host": ["${cluster.clusterEndpoint.hostname}"]}'`,
});
```

**提供される管理コマンド**:
- **ポートフォワーディング**: SSM経由でのデータベース接続
- **シークレット取得**: 認証情報の取得コマンド

## 技術的特徴

### 1. 高可用性・耐障害性
- **マルチAZ配置**: 複数のアベイラビリティゾーンに分散
- **自動フェイルオーバー**: プライマリインスタンス障害時の自動切り替え
- **自動バックアップ**: ポイントインタイム復旧対応

### 2. コスト最適化
- **Serverless v2**: 使用量に応じた自動スケーリング
- **ゼロスケーリング**: 非使用時の完全停止（オプション）
- **最小インスタンス**: t4g.nano Bastionホスト

### 3. セキュリティ
- **ネットワーク分離**: プライベートサブネット配置
- **暗号化**: 保存時・転送時データ暗号化
- **認証情報管理**: Secrets Managerによる安全な管理
- **最小権限**: 必要最小限のIAM権限

### 4. 運用性
- **Data API**: サーバーレス環境での安全なデータベース操作
- **SSM接続**: セキュアなリモートアクセス
- **CloudWatch統合**: 自動ログ・メトリクス収集

## 依存関係管理

### 1. クエリ実行の順序制御
```typescript
if (this.queries.length > 0) {
  query.node.defaultChild!.node.addDependency(this.queries.at(-1)!.node.defaultChild!);
} else {
  const sleep = new TimeSleep(this, 'WaitForHttpEndpointReady', {
    createDuration: Duration.seconds(60),
  });
  query.node.defaultChild!.node.addDependency(sleep);
}
```

**制御メカニズム**:
- **シーケンシャル実行**: 前のクエリ完了後に次のクエリ実行
- **初期化待機**: Data API有効化まで60秒待機
- **依存関係管理**: CDKの依存関係グラフによる制御

### 2. IAM権限の自動付与
```typescript
cluster.secret!.grantRead(query);
cluster.grantDataApiAccess(query);
```

**付与される権限**:
- **Secrets Manager読み取り**: データベース認証情報アクセス
- **RDS Data API**: SQL実行権限

## 使用例

### 基本的な使用方法
```typescript
const postgres = new Postgres(this, 'Database', {
  vpc: vpc,
  scalesToZero: false,  // 本番環境では false 推奨
});

// 他のサービスからの接続許可
postgres.connections.allowDefaultPortFrom(ecsService);
```

### 開発環境での使用
```typescript
const postgres = new Postgres(this, 'Database', {
  vpc: vpc,
  scalesToZero: true,      // コスト削減
  createBastion: true,     // 直接アクセス用
});
```

## ベストプラクティス

### 1. 本番環境設定
- `scalesToZero: false` - 安定したパフォーマンス確保
- `createBastion: false` - セキュリティリスク軽減
- マルチAZ配置の維持

### 2. 開発環境設定
- `scalesToZero: true` - コスト最適化
- `createBastion: true` - 開発・デバッグ用アクセス
- 必要に応じてインスタンスサイズ調整

### 3. セキュリティ
- プライベートサブネット配置の維持
- セキュリティグループの適切な設定
- 定期的なパスワードローテーション

このPostgreSQLコンストラクトは、Difyアプリケーションに最適化された高性能・高可用性・コスト効率的なデータベース環境を提供します。