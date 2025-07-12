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

### 非機能要件
- 🔒 セキュアな認証（GitHub Apps）
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
    
    subgraph "Infrastructure Repository"
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
    
    BR1 -->|calls| RW
    BR2 -->|calls| RW
    BR3 -->|calls| RW
    BRN -->|calls| RW
    
    RW --> TT
    RW --> VS
    RW -->|uploads| S3
    
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
    participant GH as GitHub
    participant IR as Infrastructure Repo
    participant VS as Validation
    participant S3 as S3 Bucket
    participant CD as CD Process
    participant ECS as ECS
    
    BS->>GH: 1. Push code changes
    GH->>IR: 2. Call Reusable Workflow
    IR->>VS: 3. Validate parameters
    VS-->>IR: 4. Validation result
    IR->>IR: 5. Generate task definition
    IR->>S3: 6. Upload task definition
    CD->>S3: 7. Read task definition
    CD->>ECS: 8. Update service
```

## ファイル構成

### インフラリポジトリ構成

```
infrastructure-repo/
├── 📁 .github/workflows/
│   └── 📄 deploy-ecs-task-definition.yml    # Reusable Workflow
├── 📁 templates/
│   ├── 📄 task-definition-template.json     # ECSタスク定義テンプレート
│   └── 📄 service-template.json             # サービス設定テンプレート
├── 📁 validation/
│   ├── 📄 validate-params.sh                # パラメータバリデーション
│   ├── 📄 validate-resources.sh             # リソース制限チェック
│   └── 📄 security-check.sh                 # セキュリティチェック
├── 📁 scripts/
│   ├── 📄 template-processor.sh             # テンプレート処理
│   └── 📄 s3-uploader.sh                    # S3アップロード
└── 📄 README.md                             # 運用ドキュメント
```

### 業務リポジトリ構成

```
business-app-repo/
├── 📁 .github/workflows/
│   └── 📄 deploy.yml                        # Reusable Workflowを呼び出し
├── 📄 ecs-params.json                       # 業務側パラメータ
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
            P2[⚡ actions: read]
            P3[📊 metadata: read]
            P4[🔄 pull-requests: write]
        end
        
        GA --> P1
        GA --> P2
        GA --> P3
        GA --> P4
    end
    
    subgraph "Repository Access"
        IR[Infrastructure Repo]
        BR[Business Repos]
    end
    
    GA -.->|Install| IR
    GA -.->|Install| BR
    
    style GA fill:#ff9800
    style P1 fill:#4caf50
    style P2 fill:#2196f3
    style P3 fill:#9c27b0
    style P4 fill:#f44336
```

### 認証フロー図

```mermaid
sequenceDiagram
    participant BR as Business Repo
    participant GA as GitHub Apps
    participant IR as Infrastructure Repo
    participant S3 as S3 Bucket
    
    Note over BR,S3: Authentication Flow
    
    BR->>GA: 1. Request token with APP_ID
    GA->>GA: 2. Validate private key
    GA-->>BR: 3. Return access token
    BR->>IR: 4. Call reusable workflow
    Note over IR: 5. Process with token
    IR->>S3: 6. Upload with AWS credentials
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
    SECRETS --> |INSTALLATION_ID| S3[INSTALLATION_ID]
    
    S1 --> VARS{📝 Variables設定}
    S2 --> VARS
    S3 --> VARS
    
    VARS --> |INFRASTRUCTURE_REPO| V1[INFRASTRUCTURE_REPO]
    VARS --> |SERVICE_NAME| V2[SERVICE_NAME]
    
    V1 --> FILES{📄 ファイル配置}
    V2 --> FILES
    
    FILES --> |Workflow| F1[deploy-ecs.yml]
    FILES --> |Parameters| F2[ecs-params.json]
    
    F1 --> TEST[🧪 動作テスト]
    F2 --> TEST
    
    TEST --> |成功| SUCCESS([✅ セットアップ完了])
    TEST --> |失敗| DEBUG[🐛 トラブルシューティング]
    DEBUG --> TEST
    
    style START fill:#4caf50
    style SUCCESS fill:#4caf50
    style SECRETS fill:#ff9800
    style VARS fill:#2196f3
    style FILES fill:#9c27b0
```

### セットアップチェックリスト

| ステップ | 項目 | 担当 | 状態 |
|---------|------|------|------|
| 1 | GitHub Apps インストール | インフラ | ⬜ |
| 2 | INSTALLATION_ID 取得 | インフラ | ⬜ |
| 3 | Repository Secrets 設定 | 業務 | ⬜ |
| 4 | Repository Variables 設定 | 業務 | ⬜ |
| 5 | Workflow ファイル配置 | 業務 | ⬜ |
| 6 | パラメータファイル作成 | 業務 | ⬜ |
| 7 | 動作テスト | 業務 | ⬜ |

## 具体的な実装例

### Reusable Workflow

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
      params_file:
        description: 'Parameters file path'
        required: true
        type: string
      environment:
        description: 'Deployment environment'
        required: false
        type: string
        default: 'production'
    secrets:
      app_id:
        description: 'GitHub App ID'
        required: true
      app_private_key:
        description: 'GitHub App Private Key'
        required: true
      installation_id:
        description: 'GitHub App Installation ID'
        required: true
      aws_access_key_id:
        description: 'AWS Access Key ID'
        required: true
      aws_secret_access_key:
        description: 'AWS Secret Access Key'
        required: true

jobs:
  validate-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Infrastructure Repo
        uses: actions/checkout@v4
        with:
          repository: ${{ github.repository_owner }}/infrastructure-repo
          token: ${{ steps.app-token.outputs.token }}
      
      - name: Generate App Token
        id: app-token
        uses: actions/create-github-app-token@v1
        with:
          app-id: ${{ secrets.app_id }}
          private-key: ${{ secrets.app_private_key }}
          installation-id: ${{ secrets.installation_id }}
      
      - name: Checkout Business Repo
        uses: actions/checkout@v4
        with:
          path: business-repo
          token: ${{ steps.app-token.outputs.token }}
      
      - name: Validate Parameters
        run: |
          bash validation/validate-params.sh business-repo/${{ inputs.params_file }}
      
      - name: Generate Task Definition
        run: |
          bash scripts/template-processor.sh \
            templates/task-definition-template.json \
            business-repo/${{ inputs.params_file }} \
            ${{ inputs.service_name }} \
            > task-definition.json
      
      - name: Upload to S3
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.aws_access_key_id }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.aws_secret_access_key }}
          AWS_DEFAULT_REGION: ap-northeast-1
        run: |
          aws s3 cp task-definition.json \
            s3://your-bucket/task-definitions/${{ inputs.service_name }}/task-definition.json
```

### 業務リポジトリのWorkflow

```yaml
# .github/workflows/deploy-ecs.yml
name: Deploy ECS Task Definition

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  deploy:
    uses: your-org/infrastructure-repo/.github/workflows/deploy-ecs-task-definition.yml@main
    with:
      service_name: ${{ vars.SERVICE_NAME }}
      params_file: ecs-params.json
      environment: production
    secrets:
      app_id: ${{ secrets.APP_ID }}
      app_private_key: ${{ secrets.APP_PRIVATE_KEY }}
      installation_id: ${{ secrets.INSTALLATION_ID }}
      aws_access_key_id: ${{ secrets.AWS_ACCESS_KEY_ID }}
      aws_secret_access_key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

### パラメータファイル例

```json
{
  "cpu": "256",
  "memory": "512",
  "image_tag": "v1.2.3",
  "port": 8080,
  "environment_vars": [
    {
      "name": "ENV",
      "value": "production"
    },
    {
      "name": "LOG_LEVEL",
      "value": "info"
    },
    {
      "name": "DATABASE_URL",
      "value": "postgresql://prod-db:5432/myapp"
    }
  ],
  "health_check": {
    "path": "/health",
    "interval": 30,
    "timeout": 5,
    "retries": 3,
    "start_period": 60
  },
  "resource_limits": {
    "cpu_limit": "512",
    "memory_limit": "1024"
  }
}
```

### バリデーションスクリプト例

```bash
#!/bin/bash
# validation/validate-params.sh

PARAMS_FILE=$1

# JSONファイルの存在確認
if [[ ! -f "$PARAMS_FILE" ]]; then
    echo "❌ ERROR: Parameters file not found: $PARAMS_FILE"
    exit 1
fi

# JSON形式の検証
if ! jq empty "$PARAMS_FILE" 2>/dev/null; then
    echo "❌ ERROR: Invalid JSON format in $PARAMS_FILE"
    exit 1
fi

# CPUの制限チェック
CPU=$(jq -r '.cpu' "$PARAMS_FILE")
if [[ "$CPU" -gt 1024 ]]; then
    echo "❌ ERROR: CPU exceeds limit (1024): $CPU"
    exit 1
fi

# メモリの制限チェック
MEMORY=$(jq -r '.memory' "$PARAMS_FILE")
if [[ "$MEMORY" -gt 2048 ]]; then
    echo "❌ ERROR: Memory exceeds limit (2048MB): $MEMORY"
    exit 1
fi

# 必須フィールドの確認
REQUIRED_FIELDS=("cpu" "memory" "image_tag" "port")
for field in "${REQUIRED_FIELDS[@]}"; do
    if [[ $(jq -r ".$field" "$PARAMS_FILE") == "null" ]]; then
        echo "❌ ERROR: Required field missing: $field"
        exit 1
    fi
done

echo "✅ Validation passed for $PARAMS_FILE"
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
        L5[📋 Audit Logging]
    end
    
    L1 --> L2
    L2 --> L3
    L3 --> L4
    L4 --> L5
    
    style L1 fill:#f44336
    style L2 fill:#ff9800
    style L3 fill:#ffeb3b
    style L4 fill:#4caf50
    style L5 fill:#2196f3
```

### トラブルシューティング

| 問題 | 原因 | 解決方法 |
|------|------|---------|
| Workflow実行エラー | 認証失敗 | INSTALLATION_IDの再確認 |
| バリデーション失敗 | パラメータ不正 | ecs-params.jsonの形式確認 |
| S3アップロード失敗 | AWS権限不足 | IAMポリシーの確認 |
| テンプレート処理エラー | 必須パラメータ欠如 | パラメータファイルの補完 |

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

### 更新履歴
| 日付 | バージョン | 変更内容 |
|------|-----------|---------|
| 2025-07-12 | 1.0.0 | 初版作成 |

---

**このドキュメントに関する質問・要望は、インフラチームまでお問い合わせください。**