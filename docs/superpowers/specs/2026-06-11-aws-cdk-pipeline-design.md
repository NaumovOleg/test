# AWS CDK Pipeline Design

**Date:** 2026-06-11  
**Status:** Approved

## Overview

Single CodeBuild project with GitHub webhook monitoring two branches (`dev`, `stage`). Push to either branch triggers deployment to the corresponding CDK stack in one AWS account.

## Architecture

```
GitHub repo
  ├── branch: dev  ──push──┐
  └── branch: stage ─push──┤
                            ▼
                  CodeBuild (webhook filter: dev|stage)
                            │
                  reads CODEBUILD_WEBHOOK_TRIGGER
                            │
              ┌─────────────┴─────────────┐
              ▼                           ▼
       AppStack-dev               AppStack-stage
       (Lambda-dev)               (Lambda-stage)
```

## Stacks

### PipelineStack
Single CDK stack containing:
- `CodeBuild Project` — GitHub source, Node.js runtime
- `GitHub webhook` — filter pattern `^refs/heads/(dev|stage)$`
- `GitHubSourceCredentials` — hardcoded token (practice only)
- IAM role with CloudFormation/Lambda/S3/SSM deploy permissions

### AppStack
Parameterized stack accepting `environment: 'dev' | 'stage'`:
- Node.js 20.x Lambda function returning `{ message: "hello from <env>" }`
- Stack name: `AppStack-dev` or `AppStack-stage`

## File Structure

```
aws-stack/
├── bin/
│   └── aws-stack.ts          # entry point
├── lib/
│   ├── pipeline-stack.ts     # CodeBuild + webhook
│   └── app-stack.ts          # parameterized Lambda stack
├── lambda/
│   └── hello/
│       └── index.ts          # hello-world handler
├── buildspec.yml             # branch detection + cdk deploy
├── cdk.json
├── package.json
└── tsconfig.json
```

## Buildspec Logic

```yaml
version: 0.2
phases:
  install:
    commands:
      - npm ci
  build:
    commands:
      - BRANCH=$(echo $CODEBUILD_WEBHOOK_TRIGGER | sed 's/branch\///')
      - npx cdk deploy AppStack-$BRANCH --require-approval never
```

## IAM Permissions (CodeBuild role)

- `cloudformation:*`
- `iam:PassRole`
- `lambda:*`
- `s3:*` (CDK asset bucket)
- `ssm:GetParameter` (CDK bootstrap params)

## Configuration

- **CDK_DEFAULT_ACCOUNT** and **CDK_DEFAULT_REGION** set in buildspec env vars
- GitHub token: passed directly to `codebuild.GitHubSourceCredentials` (practice only — use Secrets Manager in production)
- CDK bootstrap: already done in target account

## Deployment Flow

1. `cdk bootstrap` — already done
2. `cdk deploy --all` — initial deploy of all 3 stacks (PipelineStack + AppStack-dev + AppStack-stage)
3. Push to `dev` → CodeBuild triggers → deploys `AppStack-dev`
4. Push to `stage` → CodeBuild triggers → deploys `AppStack-stage`

## Out of Scope

- Manual approval gates between environments
- Multi-account setup
- CDK Pipelines self-mutating pipeline
- Tests
