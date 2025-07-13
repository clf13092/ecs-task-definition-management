# ECS タスク定義ファイル管理構成設計書

## 目次
1. [概要](#概要)
2. [要件](#要件)
3. [アーキテクチャ](#アーキテクチャ)
4. [ファイル構成](#ファイル構成)
5. [認証設定](#認証設定)
6. [セットアップ手順](#セットアップ手順)
7. [具体的な実装例](#具体的な実装例)

## 概要

本設計書では、マイクロサービス環境におけるECSタスク定義ファイルの運用管理構成について説明します。
GitHub Reusable Workflowsを活用し、インフラチームによる統制と業務チームの柔軟性を両立させる構成を提案します。

## 要件

### 機能要件
- ✅ 業務アプリケーション側でタスク定義ファイル内の一部項目を編集可能
- ✅ インフラ側でCPU/メモリなどのリソース制限を統制
- ✅ GitHub Workflowsの書き方をインフラ側で統制
- ✅ 30個程度のマイクロサービスリポジトリに対応
- ✅ プライベートリポジトリ間でのワークフロー連携

### 非機能要件
- 🔒 セキュアな認証（GitHub Apps + AWS OIDC）
- 🚀 スケーラブルな運用
- 🔧 メンテナンス性の確保

## アーキテクチャ

### システム全体構成図

```mermaid
graph TB
    subgraph "Business Repositories (30+)"
        BR1[Business Repo 1<br/>user-service]
        BR2[Business Repo 2<br/>order-service]
        BR3[Business Repo 3<br/>payment-service]
        BRN[Business Repo N<br/>...]
    end
    
    subgraph "Infrastructure Repository (Private)"
        IR[Infrastructure Repo<br/>Central Management]
        RW[Reusable Workflows<br/>deploy-ecs-task-definition.yml]
        TT[Task Definition<br/>Templates]
        VS[Validation<br/>Scripts]
    end
    
    subgraph "AWS"
        S3[S3 Bucket<br/>Task Definitions]
        ECS[ECS Cluster<br/>Services]
        CD[CD Process<br/>Deployment]
    end
    
    BR1 -->|GitHub Apps API| RW
    BR2 -->|GitHub Apps API| RW
    BR3 -->|GitHub Apps API| RW
    BRN -->|GitHub Apps API| RW
    
    RW --> TT
    RW --> VS
    RW -->|OIDC Auth| S3
    
    CD -->|reads| S3
    CD -->|deploys| ECS
    
    style IR fill:#e1f5fe
    style RW fill:#f3e5f5
    style S3 fill:#fff3e0
    style ECS fill:#e8f5e8
```

### データフロー図

```mermaid
sequenceDiagram
    participant BS as Business Service
    participant GH as GitHub Apps
    participant IR as Infrastructure Repo
    participant VS as Validation
    participant S3 as S3 Bucket
    participant CD as CD Process
    participant ECS as ECS
    
    BS->>GH: 1. Generate App Token
    GH-->>BS: 2. Return access token
    BS->>BS: 3. Read params file
    BS->>IR: 4. Call workflow via API
    IR->>VS: 5. Validate parameters
    VS-->>IR: 6. Validation result
    IR->>IR: 7. Generate task definition
    IR->>S3: 8. Upload task definition (OIDC)
    CD->>S3: 9. Read task definition
    CD->>ECS: 10. Update service
```

## ファイル構成

### インフラリポジトリ構成

```
infra_repository/ (Private)
├── 📁 .github/workflows/
│   └── 📄 deploy-ecs-task-definition.yml    # Reusable Workflow
├── 📁 templates/
│   └── 📄 task-definition-template.json     # ECSタスク定義テンプレート
├── 📁 validation/
│   └── 📄 validate-params.sh                # パラメータバリデーション
├── 📁 scripts/
│   └── 📄 template-processor.sh             # テンプレート処理
└── 📄 README.md                             # 運用ドキュメント
```

### 業務リポジトリ構成

```
app_repository/
├── 📁 .github/workflows/
│   └── 📄 deploy.yml                        # GitHub Apps API でワークフロー呼び出し
├── 📄 ecs-params.json                       # 業務側パラメータ（シンプル化）
├── 📄 src/                                  # アプリケーションコード
└── 📄 README.md                             # サービス固有ドキュメント
```

## 認証設定

### GitHub Apps権限設定図

```mermaid
graph LR
    subgraph "GitHub Apps Configuration"
        GA[GitHub Apps]
        
        subgraph "Required Permissions"
            P1[📖 contents: read]
            P2[⚡ actions: write]
            P3[📊 metadata: read]
        end
        
        GA --> P1
        GA --> P2
        GA --> P3
    end
    
    subgraph "Repository Access"
        IR[Infrastructure Repo<br/>(Private)]
        BR[Business Repos]
    end
    
    GA -.->|Install| IR
    GA -.->|Install| BR
    
    style GA fill:#ff9800
    style P1 fill:#4caf50
    style P2 fill:#2196f3
    style P3 fill:#9c27b0
```

### AWS OIDC認証フロー図

```mermaid
sequenceDiagram
    participant IR as Infrastructure Repo
    participant OIDC as GitHub OIDC Provider
    participant AWS as AWS STS
    participant S3 as S3 Bucket
    
    Note over IR,S3: AWS OIDC Authentication Flow
    
    IR->>OIDC: 1. Request OIDC token
    OIDC-->>IR: 2. Return ID token
    IR->>AWS: 3. AssumeRoleWithWebIdentity
    AWS-->>IR: 4. Return temporary credentials
    IR->>S3: 5. Upload with temporary credentials
```

## セットアップ手順

### 新規リポジトリ作成フロー

```mermaid
flowchart TD
    START([新規リポジトリ作成開始]) --> REPO[📁 リポジトリ作成]
    
    REPO --> INSTALL[🔧 GitHub Apps インストール]
    INSTALL --> GETID[🆔 INSTALLATION_ID取得]
    
    GETID --> SECRETS{🔐 Secrets設定}
    SECRETS --> |APP_ID| S1[APP_ID]
    SECRETS --> |APP_PRIVATE_KEY| S2[APP_PRIVATE_KEY]
    
    S1 --> FILES{📄 ファイル配置}
    S2 --> FILES
    
    FILES --> |Workflow| F1[deploy.yml]
    FILES --> |Parameters| F2[ecs-params.json]
    
    F1 --> TEST[🧪 動作テスト]
    F2 --> TEST
    
    TEST --> |成功| SUCCESS([✅ セットアップ完了])
    TEST --> |失敗| DEBUG[🐛 トラブルシューティング]
    DEBUG --> TEST
    
    style START fill:#4caf50
    style SUCCESS fill:#4caf50
    style SECRETS fill:#ff9800
    style FILES fill:#9c27b0
```

### AWS OIDC設定手順

```mermaid
flowchart TD
    AWS_START([AWS OIDC設定開始]) --> PROVIDER[🔧 OIDC Provider作成]
    
    PROVIDER --> |URL| P1[token.actions.githubusercontent.com]
    PROVIDER --> |Audience| P2[sts.amazonaws.com]
    
    P1 --> ROLE[🎭 IAMロール作成]
    P2 --> ROLE
    
    ROLE --> TRUST[📝 信頼ポリシー設定]
    TRUST --> POLICY[📋 権限ポリシー設定]
    
    POLICY --> SECRET[🔐 AWS_ROLE_ARNシークレット設定]
    SECRET --> AWS_SUCCESS([✅ AWS設定完了])
    
    style AWS_START fill:#ff9800
    style AWS_SUCCESS fill:#4caf50
```

## 具体的な実装例

### Infrastructure Repository: Reusable Workflow

```yaml
# .github/workflows/deploy-ecs-task-definition.yml
name: Deploy ECS Task Definition

on:
  workflow_call:
    inputs:
      service_name:
        description: 'Service name'
        required: true
        type: string
      params_content:
        description: 'Parameters file content as JSON string'
        required: true
        type: string
      environment:
        description: 'Deployment environment'
        required: false
        type: string
        default: 'production'
  workflow_dispatch:
    inputs:
      service_name:
        description: 'Service name'
        required: true
        type: string
      params_content:
        description: 'Parameters file content as JSON string'
        required: true
        type: string
      environment:
        description: 'Deployment environment'
        required: false
        type: string
        default: 'production'

jobs:
  validate-and-deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - name: Checkout Infrastructure Repo
        uses: actions/checkout@v4
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ap-northeast-1
      
      - name: Create params file from input
        run: |
          echo '${{ inputs.params_content }}' > params.json
      
      - name: Install jq
        run: |
          sudo apt-get update
          sudo apt-get install -y jq
      
      - name: Validate Parameters
        run: |
          chmod +x ./validation/validate-params.sh
          ./validation/validate-params.sh params.json
      
      - name: Generate Task Definition
        run: |
          chmod +x ./scripts/template-processor.sh
          ./scripts/template-processor.sh \
            templates/task-definition-template.json \
            params.json \
            ${{ inputs.service_name }} \
            > task-definition.json
      
      - name: Upload to S3
        run: |
          aws s3 cp task-definition.json \
            s3://github-taskdefinition-test/task-definitions/${{ inputs.service_name }}/task-definition.json
```

### Business Repository: Workflow

```yaml
# .github/workflows/deploy.yml
name: Deploy ECS Task Definition

on:
  workflow_dispatch:
    inputs:
      service_name:
        description: 'Service name'
        required: true
        type: string
      params_file:
        description: 'Parameters file path'
        required: true
        type: string
        default: 'ecs-params.json'
      environment:
        description: 'Deployment environment'
        required: false
        type: string
        default: 'production'

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Generate App Token
        id: app-token
        uses: actions/create-github-app-token@v1
        with:
          app-id: ${{ secrets.APP_ID }}
          private-key: ${{ secrets.APP_PRIVATE_KEY }}
          repositories: "app_repository,infra_repository"

      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          token: ${{ steps.app-token.outputs.token }}
      
      - name: Read params file
        id: params
        run: |
          content=$(cat ${{ inputs.params_file }})
          echo "content<<EOF" >> $GITHUB_OUTPUT
          echo "$content" >> $GITHUB_OUTPUT
          echo "EOF" >> $GITHUB_OUTPUT

      - name: Call infra workflow
        uses: actions/github-script@v7
        with:
          github-token: ${{ steps.app-token.outputs.token }}
          script: |
            const response = await github.rest.actions.createWorkflowDispatch({
              owner: 'clf13092',
              repo: 'infra_repository',
              workflow_id: 'deploy-ecs-task-definition.yml',
              ref: 'main',
              inputs: {
                service_name: '${{ inputs.service_name }}',
                params_content: `${{ steps.params.outputs.content }}`,
                environment: '${{ inputs.environment }}'
              }
            });
            console.log('Workflow dispatched:', response.status);
```

### パラメータファイル例（シンプル化）

```json
{
  "image": "nginx:latest",
  "cpu": "256",
  "memory": "512",
  "environment": "production"
}
```

### Task Definition Template

```json
{
  "family": "placeholder",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::123456789012:role/ecsTaskRole",
  "containerDefinitions": [
    {
      "name": "placeholder",
      "image": "nginx:latest",
      "essential": true,
      "portMappings": [
        {
          "containerPort": 80,
          "protocol": "tcp"
        }
      ],
      "environment": [],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/placeholder",
          "awslogs-region": "ap-northeast-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

### バリデーションスクリプト例

```bash
#!/bin/bash
# validation/validate-params.sh

set -e

PARAMS_FILE="$1"

if [ -z "$PARAMS_FILE" ]; then
    echo "Error: Parameters file path is required"
    exit 1
fi

if [ ! -f "$PARAMS_FILE" ]; then
    echo "Error: Parameters file not found: $PARAMS_FILE"
    exit 1
fi

echo "Validating parameters file: $PARAMS_FILE"

if ! jq empty "$PARAMS_FILE" 2>/dev/null; then
    echo "Error: Invalid JSON format in parameters file"
    exit 1
fi

REQUIRED_FIELDS=("image" "cpu" "memory")

for field in "${REQUIRED_FIELDS[@]}"; do
    if ! jq -e ".$field" "$PARAMS_FILE" > /dev/null; then
        echo "Error: Required field '$field' is missing"
        exit 1
    fi
done

CPU_VALUE=$(jq -r '.cpu' "$PARAMS_FILE")
MEMORY_VALUE=$(jq -r '.memory' "$PARAMS_FILE")

if [[ ! "$CPU_VALUE" =~ ^[0-9]+$ ]]; then
    echo "Error: CPU value must be a number"
    exit 1
fi

if [[ ! "$MEMORY_VALUE" =~ ^[0-9]+$ ]]; then
    echo "Error: Memory value must be a number"
    exit 1
fi

echo "Parameters validation passed successfully"
```

### Template Processor Script

```bash
#!/bin/bash
# scripts/template-processor.sh

set -e

TEMPLATE_FILE="$1"
PARAMS_FILE="$2"
SERVICE_NAME="$3"

if [ -z "$TEMPLATE_FILE" ] || [ -z "$PARAMS_FILE" ] || [ -z "$SERVICE_NAME" ]; then
    echo "Usage: $0 <template_file> <params_file> <service_name>"
    exit 1
fi

if [ ! -f "$TEMPLATE_FILE" ]; then
    echo "Error: Template file not found: $TEMPLATE_FILE"
    exit 1
fi

if [ ! -f "$PARAMS_FILE" ]; then
    echo "Error: Parameters file not found: $PARAMS_FILE"
    exit 1
fi

IMAGE=$(jq -r '.image' "$PARAMS_FILE")
CPU=$(jq -r '.cpu' "$PARAMS_FILE")
MEMORY=$(jq -r '.memory' "$PARAMS_FILE")
ENVIRONMENT=$(jq -r '.environment // "production"' "$PARAMS_FILE")

if [ "$IMAGE" = "null" ] || [ "$CPU" = "null" ] || [ "$MEMORY" = "null" ]; then
    echo "Error: Required parameters (image, cpu, memory) are missing"
    exit 1
fi

jq \
  --arg service_name "$SERVICE_NAME" \
  --arg image "$IMAGE" \
  --arg cpu "$CPU" \
  --arg memory "$MEMORY" \
  --arg environment "$ENVIRONMENT" \
  '
  .family = $service_name |
  .containerDefinitions[0].name = $service_name |
  .containerDefinitions[0].image = $image |
  .cpu = $cpu |
  .memory = $memory |
  .containerDefinitions[0].environment = [
    {"name": "ENVIRONMENT", "value": $environment}
  ]
  ' "$TEMPLATE_FILE"
```

## 運用上の考慮事項

### セキュリティ対策

```mermaid
graph TD
    subgraph "Security Layers"
        L1[🔐 GitHub Apps Authentication]
        L2[🛡️ Repository Permissions]
        L3[✅ Parameter Validation]
        L4[🚨 Resource Limits]
        L5[🔒 AWS OIDC Authentication]
        L6[📋 Audit Logging]
    end
    
    L1 --> L2
    L2 --> L3
    L3 --> L4
    L4 --> L5
    L5 --> L6
    
    style L1 fill:#f44336
    style L2 fill:#ff9800
    style L3 fill:#ffeb3b
    style L4 fill:#4caf50
    style L5 fill:#2196f3
    style L6 fill:#9c27b0
```

### トラブルシューティング

| 問題 | 原因 | 解決方法 |
|------|------|---------|
| GitHub Apps認証エラー | 秘密鍵形式不正 | PEM形式の確認、改行の保持 |
| GitHub Apps権限エラー | Actions: write権限不足 | GitHub Apps設定でActions: writeを付与 |
| Workflow呼び出し失敗 | プライベートリポジトリアクセス不可 | GitHub Appsがinfra_repositoryにインストール済みか確認 |
| バリデーション失敗 | パラメータ形式不正 | ecs-params.jsonの必須フィールド確認 |
| AWS認証失敗 | OIDC設定不備 | IAMロールの信頼ポリシー確認 |
| S3アップロード失敗 | 権限不足 | IAMロールのS3権限確認 |

### 必要なSecrets設定

#### Business Repository (app_repository)
- `APP_ID`: GitHub App ID
- `APP_PRIVATE_KEY`: GitHub App Private Key（PEM形式）

#### Infrastructure Repository (infra_repository)
- `AWS_ROLE_ARN`: AWS IAMロールARN（OIDC用）

### AWS IAM設定例

#### OIDC Provider設定
- Provider URL: `token.actions.githubusercontent.com`
- Audience: `sts.amazonaws.com`

#### IAMロール信頼ポリシー例
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::YOUR_ACCOUNT:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:clf13092/infra_repository:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

### メンテナンス手順

1. **月次レビュー**
   - 各サービスのリソース使用量確認
   - バリデーションルールの見直し

2. **四半期更新**
   - テンプレートの機能追加
   - セキュリティ設定の見直し

3. **年次監査**
   - 全体アーキテクチャの見直し
   - コスト最適化の検討

---

## 付録

### 参考リンク
- [GitHub Reusable Workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows)
- [GitHub Apps](https://docs.github.com/en/developers/apps/getting-started-with-apps)
- [ECS Task Definitions](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definitions.html)
- [AWS OIDC with GitHub Actions](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services)

### 更新履歴
| 日付 | バージョン | 変更内容 |
|------|-----------|---------|
| 2025-07-12 | 1.0.0 | 初版作成 |
| 2025-07-13 | 1.1.0 | プライベートリポジトリ対応、OIDC認証対応、ワークフロー構造更新 |

---

**このドキュメントに関する質問・要望は、インフラチームまでお問い合わせください。**