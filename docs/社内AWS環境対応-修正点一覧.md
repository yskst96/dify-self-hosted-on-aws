# Dify on AWS - 社内AWS環境対応 修正点一覧

## 概要
社内AWSアカウントでの制約（IAMロール・VPCリソースの事前作成必須）に対応するための修正点を整理しています。

## 制約事項
- **IAMロール・ポリシー制約**: 自由に作成できず、管理者が事前作成したロールを使用
- **VPC・サブネット制約**: 自由に作成できず、管理者が事前作成したネットワークリソースを使用

---

## 1. IAMロール関連の修正点

### 1.1 明示的に作成されるIAMロール・ポリシー

#### 🔴 HIGH PRIORITY - 管理者権限が必要なロール

**ファイル**: `bin/deploy.ts:52`
```typescript
project.role!.addManagedPolicy(ManagedPolicy.fromAwsManagedPolicyName('AdministratorAccess'));
```
**修正内容**: CodeBuildプロジェクト用の管理者権限を、事前作成された適切なロールに変更

#### 🔴 HIGH PRIORITY - SES SMTP認証用IAMユーザー

**ファイル**: `lib/constructs/email.ts:46-50`
```typescript
const smtpCredentials = new SesSmtpCredentials(this, 'SmtpCredentials', {
  user: new User(this, 'SmtpCredentialsUser', {
    managedPolicies: [
      ManagedPolicy.fromAwsManagedPolicyName('AmazonSESFullAccess'),
    ],
  }),
});
```
**修正内容**: SES SMTP用のIAMユーザー作成を、事前作成されたSMTP認証情報に変更

#### 🟡 MEDIUM PRIORITY - カスタムリソース用の権限

**ファイル**: `lib/constructs/postgres.ts:104-106`
```typescript
AwsCustomResourcePolicy.fromSdkCalls({ resources: [cluster.clusterArn] })
```
**修正内容**: RDS Data API用のカスタムリソース権限を事前作成されたロールに変更

**ファイル**: `lib/constructs/alb-with-cloudfront.ts:30-40`
```typescript
policy: {
  statements: [
    new PolicyStatement({
      actions: ['ec2:DescribeManagedPrefixLists'],
      resources: ['*'],
    }),
  ],
}
```
**修正内容**: CloudFrontプレフィックリスト取得用のポリシーを事前作成されたロールに変更

### 1.2 自動生成されるIAMロール

#### 🔴 HIGH PRIORITY - ECS Fargate用ロール

**必要なロール**:
- **ECSタスク実行ロール**: ECR・CloudWatch・Secrets Manager アクセス権限
- **ECSタスクロール**: S3・Bedrock・アプリケーション固有の権限

**ファイル**: `lib/constructs/dify-services/api.ts:370-390`
```typescript
// 現在の実装
storageBucket.grantReadWrite(taskDefinition.taskRole);
taskDefinition.taskRole.addToPrincipalPolicy(
  new PolicyStatement({
    actions: ['bedrock:InvokeModel', 'bedrock:InvokeModelWithResponseStream', ...],
    resources: ['*'],
  }),
);
```

**修正内容**: 
- `taskDefinition.fromTaskRoleArn()` を使用して事前作成されたロールを指定
- S3・Bedrock権限を含む統合されたタスクロールを事前作成

#### 🟡 MEDIUM PRIORITY - その他のサービスロール

- **RDS Aurora**: 自動バックアップ・Data API権限
- **ElastiCache**: サブネットグループ管理権限  
- **ALB**: ヘルスチェック・ターゲットグループ管理権限
- **CloudFront**: オリジンアクセス・ログ配信権限

**修正内容**: 各サービス用の事前作成されたサービスロールを設定で指定

---

## 2. VPC・ネットワークリソース関連の修正点

### 2.1 VPC作成の修正

#### 🔴 HIGH PRIORITY - VPC作成ロジックの変更

**ファイル**: `lib/constructs/vpc.ts:12-52`

**現在の実装**:
```typescript
// 新規VPC作成
vpc = new Vpc(scope, 'Vpc', {
  maxAzs: 2,
  subnetConfiguration: [...],
});
```

**修正内容**:
```typescript
// 既存VPC参照のみ
vpc = Vpc.fromLookup(scope, 'Vpc', { 
  vpcId: props.vpcId  // 必須パラメータ化
});
```

#### 🔴 HIGH PRIORITY - サブネット参照の修正

**現在の実装**: サブネットタイプによる自動選択
```typescript
// postgres.ts:64
vpcSubnets: vpc.selectSubnets({ 
  subnets: vpc.privateSubnets.concat(vpc.isolatedSubnets) 
})

// alb.ts:78-80
vpcSubnets: vpc.selectSubnets({
  subnets: internal ? vpc.privateSubnets.concat(vpc.isolatedSubnets) : vpc.publicSubnets,
})
```

**修正内容**: 特定のサブネットIDを指定
```typescript
// 設定ファイルで事前定義されたサブネットIDを使用
vpcSubnets: vpc.selectSubnets({ 
  subnetIds: props.privateSubnetIds  // 新規パラメータ
})
```

### 2.2 セキュリティグループの修正

#### 🟡 MEDIUM PRIORITY - セキュリティグループ作成

**ファイル**: `lib/constructs/redis.ts:30-32`
```typescript
const securityGroup = new SecurityGroup(this, 'SecurityGroup', {
  vpc,
});
```

**修正内容**: 事前作成されたセキュリティグループを参照
```typescript
const securityGroup = SecurityGroup.fromSecurityGroupId(
  this, 'SecurityGroup', 
  props.redisSecurityGroupId
);
```

### 2.3 VPCエンドポイントの修正

#### 🟡 MEDIUM PRIORITY - VPCエンドポイント作成

**ファイル**: `lib/constructs/vpc-endpoints.ts:49-61`
```typescript
new InterfaceVpcEndpoint(this, item.service.shortName, {
  vpc,
  service: item.service,
});
```

**修正内容**: 事前作成されたVPCエンドポイントを参照または作成をスキップ

---

## 3. 設定ファイルの修正

### 3.1 環境設定の拡張

**ファイル**: `lib/environment-props.ts`

**追加必要パラメータ**:
```typescript
export interface EnvironmentProps {
  // 既存パラメータ...
  
  // 🔴 必須: 事前作成リソース
  vpcId: string;  // 既存だが必須化
  privateSubnetIds: string[];  // 新規追加
  publicSubnetIds: string[];   // 新規追加
  isolatedSubnetIds?: string[]; // 新規追加
  
  // 🔴 必須: 事前作成IAMロール
  ecsTaskRoleArn: string;        // 新規追加
  ecsExecutionRoleArn: string;   // 新規追加
  codeBuildRoleArn: string;      // 新規追加
  
  // 🟡 オプション: セキュリティグループ
  albSecurityGroupId?: string;    // 新規追加
  redisSecurityGroupId?: string;  // 新規追加
  rdsSecurityGroupId?: string;    // 新規追加
  
  // 🟡 オプション: SMTP設定
  smtpEndpoint?: string;          // 新規追加
  smtpCredentialsSecretArn?: string; // 新規追加
}
```

### 3.2 デプロイ設定の修正

**ファイル**: `bin/cdk.ts`

**修正内容**:
```typescript
export const props: EnvironmentProps = {
  // 既存設定...
  
  // 🔴 必須: ネットワークリソース
  vpcId: 'vpc-xxxxxxxxx',  // 管理者提供
  privateSubnetIds: ['subnet-xxxxxxxxx', 'subnet-yyyyyyyyy'],
  publicSubnetIds: ['subnet-aaaaaaaa', 'subnet-bbbbbbb'],
  
  // 🔴 必須: IAMロール
  ecsTaskRoleArn: 'arn:aws:iam::123456789012:role/DifyEcsTaskRole',
  ecsExecutionRoleArn: 'arn:aws:iam::123456789012:role/DifyEcsExecutionRole',
  codeBuildRoleArn: 'arn:aws:iam::123456789012:role/DifyCodeBuildRole',
  
  // 🟡 オプション: セキュリティグループ（既存利用の場合）
  albSecurityGroupId: 'sg-xxxxxxxxx',
  redisSecurityGroupId: 'sg-yyyyyyyyy',
  rdsSecurityGroupId: 'sg-zzzzzzzzz',
};
```

---

## 4. 実装優先度と対応順序

### Phase 1: 🔴 HIGH PRIORITY（必須対応）
1. **VPC作成の無効化**: 既存VPC参照のみに変更
2. **サブネット参照の修正**: 固定サブネットID指定
3. **ECSタスクロールの修正**: 事前作成ロール使用
4. **CodeBuildロールの修正**: 管理者権限の適切な制限
5. **環境設定の必須パラメータ追加**

### Phase 2: 🟡 MEDIUM PRIORITY（推奨対応）
1. **セキュリティグループの修正**: 事前作成SG使用
2. **カスタムリソース権限の修正**: 統合ロール使用
3. **VPCエンドポイント作成の制御**: 既存利用オプション
4. **SMTP認証の修正**: 外部SMTP使用

### Phase 3: 🟢 LOW PRIORITY（オプション）
1. **その他サービスロールの最適化**
2. **ログ・監視権限の統合**
3. **デプロイメント自動化の改善**

---

## 5. 管理者への要求事項

### 5.1 事前作成が必要なIAMロール

#### ECSタスク実行ロール
**ロール名**: `DifyEcsExecutionRole`
**必要な権限**:
- `AmazonECSTaskExecutionRolePolicy`
- ECR読み取り権限
- CloudWatch Logs書き込み権限
- Secrets Manager読み取り権限

#### ECSタスクロール  
**ロール名**: `DifyEcsTaskRole`
**必要な権限**:
- S3読み書き権限（Difyストレージバケット）
- Bedrock実行権限
- RDS接続権限
- ElastiCache接続権限

#### CodeBuildロール
**ロール名**: `DifyCodeBuildRole`  
**必要な権限**:
- ECR読み書き権限
- S3読み書き権限（アーティファクト用）
- CloudWatch Logs書き込み権限

### 5.2 事前作成が必要なネットワークリソース

#### VPC要件
- **パブリックサブネット**: 最低2つ（ALB用）
- **プライベートサブネット**: 最低2つ（ECS/RDS/ElastiCache用）
- **インターネットゲートウェイ**: パブリックサブネット用
- **NAT Gateway**: プライベートサブネット用（またはVPCエンドポイント）

#### セキュリティグループ要件
- **ALB用**: HTTP/HTTPS受信許可
- **ECS用**: ALBからの通信許可
- **RDS用**: ECSからのPostgreSQL通信許可
- **ElastiCache用**: ECSからのRedis通信許可

---

## 6. 検証・テスト項目

### 6.1 デプロイメント検証
- [ ] VPC・サブネット参照の正常動作確認
- [ ] IAMロール権限の正常動作確認  
- [ ] セキュリティグループ通信の確認
- [ ] ECSタスク起動の確認

### 6.2 アプリケーション検証
- [ ] Dify Webアプリケーションアクセス確認
- [ ] データベース接続確認
- [ ] Redis接続確認
- [ ] S3ストレージアクセス確認
- [ ] Bedrock API呼び出し確認

### 6.3 セキュリティ検証  
- [ ] 不要な権限の除去確認
- [ ] ネットワーク分離の確認
- [ ] 暗号化設定の確認
- [ ] ログ・監査設定の確認

---

## 7. 補足情報

### 7.1 既存の設定オプション活用
現在のコードでは `vpcId` パラメータによる既存VPC使用が部分的にサポートされているため、これを全面的に活用する方向で修正を行います。

### 7.2 段階的な移行アプローチ
一度に全ての修正を行うのではなく、Phase別に段階的に対応することで、リスクを最小限に抑えた移行が可能です。

### 7.3 将来の拡張性
社内AWS環境の制約に対応しつつ、将来的な要件変更にも柔軟に対応できるような設計とします。