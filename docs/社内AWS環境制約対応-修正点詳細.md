# 社内AWS環境制約対応 - 修正点詳細

本ドキュメントでは、社内AWS環境の制約に対応するために必要なコード修正点を制約ごとに詳細にまとめています。

## 制約1: IAMロール・ポリシー制約

**制約内容**: 自由に作成できず、管理者が事前作成したロールを使用

### 影響箇所と修正点

#### 1. API Service (`lib/constructs/dify-services/api.ts`)

**影響箇所**:
- **行399**: `storageBucket.grantReadWrite(taskDefinition.taskRole)` - S3バケットアクセス権限の自動付与
- **行401-413**: カスタムIAMポリシーの作成（Bedrockアクセス権限）

**修正点**:
```typescript
// 現在のコード（修正前）
storageBucket.grantReadWrite(taskDefinition.taskRole);
taskDefinition.taskRole.addToPrincipalPolicy(
  new PolicyStatement({
    actions: [
      'bedrock:InvokeModel',
      'bedrock:InvokeModelWithResponseStream', 
      'bedrock:ListFoundationModels',
      'bedrock:Rerank',
      'bedrock:Retrieve',
      'bedrock:RetrieveAndGenerate',
    ],
    resources: ['*'],
  }),
);

// 修正後
// 事前作成されたロールを使用
const existingTaskRole = Role.fromRoleArn(this, 'ExistingTaskRole', 
  props.existingTaskRoleArn); // 新しいpropsパラメータが必要
taskDefinition.taskRole = existingTaskRole;
// grantReadWriteとaddToPrincipalPolicyの呼び出しを削除
```

#### 2. PostgreSQL Construct (`lib/constructs/postgres.ts`)

**影響箇所**:
- **行119**: `AwsCustomResourcePolicy.fromSdkCalls()` - カスタムリソース用IAMポリシー
- **行121-122**: `cluster.secret!.grantRead()`, `cluster.grantDataApiAccess()` - 権限付与

**修正点**:
```typescript
// 現在のコード（修正前）
const query = new AwsCustomResource(this, 'Query', {
  policy: AwsCustomResourcePolicy.fromSdkCalls({ resources: [cluster.clusterArn] }),
  // ...
});
cluster.secret!.grantRead(query);
cluster.grantDataApiAccess(query);

// 修正後
// 事前作成されたロールを使用
const query = new AwsCustomResource(this, 'Query', {
  role: Role.fromRoleArn(this, 'ExistingCustomResourceRole', 
    props.existingCustomResourceRoleArn), // 新しいpropsパラメータが必要
  // ...
});
// grantReadとgrantDataApiAccessの呼び出しを削除
```

#### 3. ALB with CloudFront (`lib/constructs/alb-with-cloudfront.ts`)

**影響箇所**:
- **行193-200**: EC2権限用カスタムポリシー作成

**修正点**:
```typescript
// 現在のコード（修正前）
const query = new AwsCustomResource(this, 'Query', {
  policy: {
    statements: [
      new PolicyStatement({
        actions: ['ec2:DescribeManagedPrefixLists'],
        resources: ['*'],
      }),
    ],
  },
  // ...
});

// 修正後
// 事前作成されたロールを使用
const query = new AwsCustomResource(this, 'Query', {
  role: Role.fromRoleArn(this, 'ExistingEc2QueryRole', 
    props.existingEc2QueryRoleArn), // 新しいpropsパラメータが必要
  // ...
});
```

#### 4. CodeBuild Deploy Stack (`bin/deploy.ts`)

**影響箇所**:
- **行39**: `project.role!.addManagedPolicy()` - 管理ポリシーの追加

**修正点**:
```typescript
// 現在のコード（修正前）
project.role!.addManagedPolicy(ManagedPolicy.fromAwsManagedPolicyName('AdministratorAccess'));

// 修正後
// 事前作成されたロールを使用
const project = new Project(stack, 'Project', {
  role: Role.fromRoleArn(stack, 'ExistingCodeBuildRole', 
    process.env.EXISTING_CODEBUILD_ROLE_ARN!), // 環境変数から取得
  // 他の設定...
});
// addManagedPolicyの呼び出しを削除
```

#### 5. Email Service (`lib/constructs/email.ts`)

**影響箇所**:
- **行43**: `SesSmtpCredentials` によるIAMユーザー作成

**修正点**:
```typescript
// 修正が必要：事前作成されたSMTP認証情報を使用するよう変更
// または、SES設定を管理者側で事前構成し、このコンストラクトを削除
```

### 必要な新しいプロパティ

`lib/environment-props.ts` に以下のプロパティを追加:
```typescript
export interface EnvironmentProps {
  // 既存のプロパティ...
  
  // IAMロール関連（事前作成済みロールのARN）
  readonly existingTaskRoleArn?: string;
  readonly existingExecutionRoleArn?: string;
  readonly existingCustomResourceRoleArn?: string;
  readonly existingEc2QueryRoleArn?: string;
  readonly existingCodeBuildRoleArn?: string;
}
```

---

## 制約2: ネットワークリソース制約

**制約内容**: VPC、Subnet、SecurityGroup、NAT Gateway、Elastic IPは管理者が事前作成した既存リソースを使用

### 影響箇所と修正点

#### 1. VPC構築 (`lib/constructs/vpc.ts`)

**影響箇所**:
- **行13-48**: VPC作成ロジック全体

**修正点**:
```typescript
// 現在のコード（修正前）
export function createVpc(scope: Construct, props: VpcProps): IVpc {
  if (props.vpcId) {
    return Vpc.fromLookup(scope, 'Vpc', { vpcId: props.vpcId });
  }
  if (props.isolated) {
    return new Vpc(scope, 'Vpc', {
      // VPC作成設定...
    });
  }
  return new Vpc(scope, 'Vpc', {
    // VPC作成設定...
  });
}

// 修正後
export function createVpc(scope: Construct, props: VpcProps): IVpc {
  // 既存VPCの使用を必須化
  if (!props.vpcId) {
    throw new Error('vpcId must be provided for corporate AWS environment');
  }
  return Vpc.fromLookup(scope, 'Vpc', { vpcId: props.vpcId });
}
```

#### 2. セキュリティグループ作成 (`lib/constructs/redis.ts`)

**影響箇所**:
- **行30**: `new SecurityGroup()` - Redisセキュリティグループ作成

**修正点**:
```typescript
// 現在のコード（修正前）
const securityGroup = new SecurityGroup(this, 'SecurityGroup', {
  vpc,
  allowAllOutbound: false,
});

// 修正後
// 事前作成されたセキュリティグループを使用
const securityGroup = SecurityGroup.fromSecurityGroupId(this, 'SecurityGroup', 
  props.existingRedisSecurityGroupId); // 新しいpropsパラメータが必要
```

#### 3. ロードバランサー作成 (`lib/constructs/alb.ts`, `lib/constructs/alb-with-cloudfront.ts`)

**影響箇所**:
- **alb.ts 行76**: `new ApplicationLoadBalancer()` - ALB作成
- **alb-with-cloudfront.ts 行61**: `new ApplicationLoadBalancer()` - 内部ALB作成

**修正点**:
```typescript
// 現在のコード（修正前）
const alb = new ApplicationLoadBalancer(this, 'Alb', {
  vpc,
  vpcSubnets: internal 
    ? { subnetType: SubnetType.PRIVATE_WITH_EGRESS }
    : { subnetType: SubnetType.PUBLIC },
  internetFacing: !internal,
});

// 修正後
// 事前作成されたALBを使用（またはセキュリティグループのみ既存使用）
const alb = new ApplicationLoadBalancer(this, 'Alb', {
  vpc,
  vpcSubnets: {
    subnetIds: props.existingSubnetIds, // 事前作成されたサブネットID配列
  },
  internetFacing: !internal,
  securityGroup: SecurityGroup.fromSecurityGroupId(this, 'AlbSecurityGroup',
    props.existingAlbSecurityGroupId), // 事前作成されたセキュリティグループ
});
```

#### 4. VPCエンドポイント作成 (`lib/constructs/vpc-endpoints.ts`)

**影響箇所**:
- **行50-60**: VPCエンドポイント作成

**修正点**:
```typescript
// VPCエンドポイントも事前作成済みを前提とする場合、
// このコンストラクト全体を条件付きで無効化するか、
// 既存エンドポイントのインポート機能を追加
```

### 必要な新しいプロパティ

`lib/environment-props.ts` に以下のプロパティを追加:
```typescript
export interface EnvironmentProps {
  // 既存のプロパティ...
  
  // ネットワークリソース関連（事前作成済みリソース）
  readonly existingSubnetIds: string[];
  readonly existingRedisSecurityGroupId: string;
  readonly existingRdsSecurityGroupId: string;
  readonly existingAlbSecurityGroupId: string;
  readonly existingEcsSecurityGroupId: string;
  readonly existingNatGatewayIds?: string[];
  readonly existingElasticIpIds?: string[];
}
```

---

## 制約3: S3リソース制約

**制約内容**: S3バケットやバケットポリシーは自由に作成できず、管理者が事前に作成した既存S3バケットを使用

### 影響箇所と修正点

#### 1. メインスタック (`lib/dify-on-aws-stack.ts`)

**影響箇所**:
- **行90-95**: `new Bucket()` - アクセスログ用バケット作成
- **行109-114**: `new Bucket()` - ストレージ用バケット作成

**修正点**:
```typescript
// 現在のコード（修正前）
const accessLogBucket = new Bucket(this, 'AccessLogBucket', {
  enforceSSL: true,
  removalPolicy: cdk.RemovalPolicy.DESTROY,
  objectOwnership: ObjectOwnership.OBJECT_WRITER,
  autoDeleteObjects: true,
});

const storageBucket = new Bucket(this, 'StorageBucket', {
  autoDeleteObjects: true,
  enforceSSL: true,
  removalPolicy: cdk.RemovalPolicy.DESTROY,
  blockPublicAccess: BlockPublicAccess.BLOCK_ALL,
});

// 修正後
// 事前作成されたバケットを使用
const accessLogBucket = Bucket.fromBucketName(this, 'AccessLogBucket', 
  props.existingAccessLogBucketName); // 新しいpropsパラメータが必要

const storageBucket = Bucket.fromBucketName(this, 'StorageBucket', 
  props.existingStorageBucketName); // 新しいpropsパラメータが必要
```

#### 2. CodeBuild Deploy Stack (`bin/deploy.ts`)

**影響箇所**:
- **行19**: `new Bucket()` - ソースコード用バケット作成

**修正点**:
```typescript
// 現在のコード（修正前）
const bucket = new Bucket(stack, 'SourceBucket');

// 修正後
// 事前作成されたバケットを使用
const bucket = Bucket.fromBucketName(stack, 'SourceBucket', 
  process.env.EXISTING_SOURCE_BUCKET_NAME!); // 環境変数から取得
```

#### 3. S3権限付与の変更 (`lib/constructs/dify-services/api.ts`)

**影響箇所**:
- **行399**: `storageBucket.grantReadWrite(taskDefinition.taskRole)` - S3権限付与

**修正点**:
```typescript
// 現在のコード（修正前）
storageBucket.grantReadWrite(taskDefinition.taskRole);

// 修正後
// 既存バケット使用時は権限付与を削除（事前設定済みを前提）
// または、明示的にバケットポリシーでアクセス制御
// grantReadWriteの呼び出しを削除し、事前設定されたIAMロールを使用
```

### 必要な新しいプロパティ

`lib/environment-props.ts` に以下のプロパティを追加:
```typescript
export interface EnvironmentProps {
  // 既存のプロパティ...
  
  // S3リソース関連（事前作成済みバケット）
  readonly existingAccessLogBucketName: string;
  readonly existingStorageBucketName: string;
  readonly existingSourceBucketName?: string; // CodeBuild用（オプション）
}
```

---

## 統合的な修正アプローチ

### 1. 設定ファイル分離

社内環境用の専用設定ファイルを作成:
```typescript
// lib/corporate-environment-props.ts
export interface CorporateEnvironmentProps extends EnvironmentProps {
  // 必須の既存リソース
  readonly existingTaskRoleArn: string;
  readonly existingAccessLogBucketName: string;
  readonly existingStorageBucketName: string;
  readonly existingSubnetIds: string[];
  readonly existingRedisSecurityGroupId: string;
  readonly existingAlbSecurityGroupId: string;
  
  // VPCは必須（新規作成不可）
  readonly vpcId: string;
}
```

### 2. デプロイスクリプトの分離

社内環境用の専用デプロイスクリプトを作成:
```bash
# scripts/deploy-corporate.ts
# 社内環境の制約に合わせた専用のデプロイロジック
```

### 3. 条件分岐による対応

既存コードに社内環境フラグを追加し、条件分岐で制御:
```typescript
if (props.corporateEnvironment) {
  // 既存リソース使用ロジック
} else {
  // 従来の新規作成ロジック
}
```

### 4. 必要な事前準備

社内環境でのデプロイ前に以下のリソースが事前作成されている必要があります:

1. **IAMロール**:
   - ECSタスクロール（Bedrock、S3アクセス権限付き）
   - ECS実行ロール
   - カスタムリソース用ロール
   - CodeBuildロール

2. **ネットワークリソース**:
   - VPC
   - パブリック/プライベートサブネット
   - セキュリティグループ（ALB、ECS、RDS、Redis用）
   - NAT Gateway（必要に応じて）

3. **S3バケット**:
   - アプリケーション用ストレージバケット
   - ALBアクセスログバケット
   - CodeBuildソースバケット（必要に応じて）

これらの修正により、社内AWS環境の制約に準拠したデプロイが可能になります。