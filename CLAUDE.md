# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a CDK project that deploys Dify (an LLM app development platform) on AWS using managed services. The architecture uses Aurora PostgreSQL, ElastiCache Valkey, ECS Fargate, and optional CloudFront distribution.

## Essential Commands

### Development Commands
```bash
# Install dependencies
npm ci

# Build TypeScript
npm run build

# Format code
npm run format

# Check formatting
npm run format:check

# Run tests
npm test

# Watch for changes during development
npm run watch
```

### CDK Commands
```bash
# Bootstrap AWS account (required once per account/region)
npx cdk bootstrap

# Deploy all stacks
npx cdk deploy --all

# Deploy specific stack
npx cdk deploy DifyOnAwsStack-sata

# Generate CloudFormation template
npm run synth

# Destroy all resources
npx cdk destroy --force
```

### Specialized Commands
```bash
# Copy Dify images to private ECR (for closed networks)
npx ts-node scripts/copy-to-ecr.ts

# Simple deployment script (for CloudShell)
./simple-deploy.sh
```

## Architecture Overview

### Stack Structure
- **Main Stack**: `DifyOnAwsStack` - Primary infrastructure deployment
- **US East 1 Stack**: `UsEast1Stack` - CloudFront certificates and WAF (when using CloudFront)

### Core Infrastructure Components

#### Networking Layer (`lib/constructs/vpc.ts`)
- Creates VPC with public/private subnets or imports existing VPC
- Supports three modes: standard networking, isolated (no internet), or existing VPC import
- Automatically provisions VPC endpoints for isolated deployments
- NAT Gateway vs NAT Instance options for cost optimization

#### Load Balancing Layer
- **Standard ALB** (`alb.ts`): Direct Application Load Balancer
- **CloudFront + ALB** (`alb-with-cloudfront.ts`): ALB with CloudFront distribution
- Both implement `IAlb` interface for consistent service registration

#### Application Services (`lib/constructs/dify-services/`)
- **API Service** (`api.ts`): Multi-container ECS service running:
  - Main API container (Dify backend)
  - Worker container (background jobs)
  - Sandbox container (secure code execution)
  - Plugin daemon (plugin management)
  - External knowledge API (Bedrock Knowledge Bases integration)
- **Web Service** (`web.ts`): Next.js frontend

#### Data Layer
- **PostgreSQL** (`postgres.ts`): Aurora Serverless v2 with pgvector extension
- **Redis** (`redis.ts`): ElastiCache Valkey cluster for caching

### Key Configuration Files

#### Primary Configuration
- **`bin/cdk.ts`**: Main deployment configuration and stack instantiation
- **`lib/environment-props.ts`**: Interface defining all configurable parameters
- **`lib/dify-on-aws-stack.ts`**: Stack orchestrator that wires all constructs together

#### Environment Management
- **`dify-services/environment-variables.ts`**: Utility for managing secrets and environment variables across container types
- Container types: `'web' | 'api' | 'worker' | 'sandbox'`

### Deployment Modes

The system supports multiple deployment scenarios through configuration:

#### Standard Deployment
- Public/private subnets with NAT Gateway
- CloudFront distribution (default)
- Multi-AZ Redis and Aurora

#### Cost-Optimized Deployment
```typescript
// In bin/cdk.ts
useNatInstance: true,        // Use t4g.nano instead of NAT Gateway
isRedisMultiAz: false,       // Single AZ Redis
useFargateSpot: true,        // Use Spot capacity
enableAuroraScalesToZero: true // Auto-pause Aurora
```

#### Closed Network Deployment
```typescript
// In bin/cdk.ts
vpcIsolated: true,           // No internet access
useCloudFront: false,        // No CloudFront
internalAlb: true,           // Internal ALB only
customEcrRepositoryName: 'dify-images' // Use private ECR
```

## Common Development Tasks

### Modifying Dify Configuration
1. Update environment variables in `dify-services/environment-variables.ts`
2. Use `additionalEnvironmentVariables` in `bin/cdk.ts` for custom variables
3. Reference secrets via `{ secretName: 'name', field?: 'field' }`
4. Reference SSM parameters via `{ parameterName: 'name' }`

### Adding New AWS Services
1. Create new construct in `lib/constructs/`
2. Follow existing patterns for security groups and IAM roles
3. Add configuration parameters to `EnvironmentProps` interface
4. Wire into main stack in `dify-on-aws-stack.ts`

### Network Configuration Changes
- VPC modifications: `lib/constructs/vpc.ts`
- Security group rules: Individual construct files
- VPC endpoints: `lib/constructs/vpc-endpoints.ts`

### Container Configuration
- Main Dify containers: Modify `api.ts` and `web.ts`
- Custom containers: Add to `dify-services/docker/` directory
- Python packages for sandbox: Edit `dify-services/docker/sandbox/python-requirements.txt`

## Important Notes

### Security Considerations
- All databases deploy in private/isolated subnets
- Secrets managed through AWS Secrets Manager and SSM
- Security groups follow principle of least privilege
- Enable WAF for production deployments

### ElasticIP Creation
The system automatically creates ElasticIPs when using:
- `SubnetType.PRIVATE_WITH_EGRESS` (default behavior in `vpc.ts:45`)
- NAT Gateways (when `useNatInstance: false`)
To avoid ElasticIP charges, set `useNatInstance: true` or `vpcIsolated: true`

### Cost Optimization
- Default configuration prioritizes managed services over cost
- Use `useNatInstance: true` to replace NAT Gateway ($32/month) with NAT Instance ($3/month)
- Enable `useFargateSpot: true` for 60-70% Fargate cost reduction
- Set `enableAuroraScalesToZero: true` for automatic Aurora pause

### Version Management
- Pin `difyImageTag` to specific versions for production
- Update `difySandboxImageTag` and `difyPluginDaemonImageTag` as needed
- For closed networks, run `copy-to-ecr.ts` after version changes

### Testing
- Unit tests focus on construct configuration and IAM policies
- Integration tests deploy actual stacks (see `test/` directory)
- Snapshot tests verify CloudFormation template consistency