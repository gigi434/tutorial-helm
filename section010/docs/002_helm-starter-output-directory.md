# Helm Create の出力ディレクトリ制御

## helm createの仕様

`helm create`コマンドは**現在のディレクトリに直接チャートを作成**します。出力先を指定するオプションはありません。

```bash
# 現在のディレクトリに作成される
helm create mychart
# → ./mychart/ が作成される

# Starterを使用しても同じ
helm create mychart --starter mycompany-starter
# → ./mychart/ が作成される
```

## 出力先を制御する方法

### 方法1: ディレクトリを移動してから実行

```bash
# 目的のディレクトリに移動
cd /path/to/charts
helm create mychart --starter mycompany-starter

# または一行で
(cd /path/to/charts && helm create mychart --starter mycompany-starter)
```

### 方法2: 作成後に移動

```bash
# 一時的な場所で作成
helm create mychart --starter mycompany-starter
# 目的の場所に移動
mv mychart /path/to/charts/

# または
helm create mychart --starter mycompany-starter && mv mychart /desired/location/
```

### 方法3: シェル関数でラップ

```bash
# ~/.bashrc や ~/.zshrc に追加
helm-create() {
    local chart_name="$1"
    local output_dir="${2:-.}"  # デフォルトは現在のディレクトリ
    local starter="${3}"
    
    # 出力ディレクトリの作成
    mkdir -p "$output_dir"
    
    # 元のディレクトリを保存
    local original_dir=$(pwd)
    
    # 指定ディレクトリに移動して実行
    cd "$output_dir"
    
    if [ -n "$starter" ]; then
        helm create "$chart_name" --starter "$starter"
    else
        helm create "$chart_name"
    fi
    
    # 元のディレクトリに戻る
    cd "$original_dir"
    
    echo "Chart created at: $output_dir/$chart_name"
}

# 使用例
helm-create mychart /path/to/charts mycompany-starter
helm-create myapp ./apps  # starterなし
```

### 方法4: Makefileで管理

```makefile
# Makefile
CHART_DIR ?= ./charts
STARTER ?= mycompany-starter

.PHONY: create-chart
create-chart:
	@if [ -z "$(NAME)" ]; then \
		echo "Usage: make create-chart NAME=chartname [CHART_DIR=./charts] [STARTER=starter-name]"; \
		exit 1; \
	fi
	@mkdir -p $(CHART_DIR)
	@cd $(CHART_DIR) && helm create $(NAME) $(if $(STARTER),--starter $(STARTER))
	@echo "Chart created: $(CHART_DIR)/$(NAME)"

# 使用例
# make create-chart NAME=mychart
# make create-chart NAME=myapp CHART_DIR=/opt/charts
# make create-chart NAME=myservice STARTER=microservice
```

### 方法5: スクリプトでラップ

```bash
#!/bin/bash
# helm-create-wrapper.sh

usage() {
    cat <<EOF
Usage: $0 [OPTIONS] CHART_NAME
Options:
    -d, --dir DIR        Output directory (default: current directory)
    -s, --starter NAME   Starter template name
    -h, --help          Show this help message

Examples:
    $0 mychart
    $0 -d /path/to/charts mychart
    $0 -s mycompany-starter -d ./charts mychart
EOF
    exit 1
}

# デフォルト値
OUTPUT_DIR="."
STARTER=""
CHART_NAME=""

# オプション解析
while [[ $# -gt 0 ]]; do
    case $1 in
        -d|--dir)
            OUTPUT_DIR="$2"
            shift 2
            ;;
        -s|--starter)
            STARTER="$2"
            shift 2
            ;;
        -h|--help)
            usage
            ;;
        -*)
            echo "Unknown option: $1"
            usage
            ;;
        *)
            CHART_NAME="$1"
            shift
            ;;
    esac
done

# チャート名の確認
if [ -z "$CHART_NAME" ]; then
    echo "Error: Chart name is required"
    usage
fi

# ディレクトリ作成
mkdir -p "$OUTPUT_DIR"

# チャート作成
echo "Creating chart '$CHART_NAME' in '$OUTPUT_DIR'"
if [ -n "$STARTER" ]; then
    (cd "$OUTPUT_DIR" && helm create "$CHART_NAME" --starter "$STARTER")
    echo "Chart created with starter '$STARTER'"
else
    (cd "$OUTPUT_DIR" && helm create "$CHART_NAME")
    echo "Chart created with default template"
fi

echo "Chart location: $OUTPUT_DIR/$CHART_NAME"
```

### 方法6: エイリアスの活用

```bash
# ~/.bashrc または ~/.zshrc に追加

# 特定ディレクトリでチャート作成
alias helm-create-charts='cd ~/charts && helm create'
alias helm-create-work='cd ~/work/helm && helm create'

# 使用例
helm-create-charts mychart --starter mycompany-starter
```

## Docker/Podmanを使った方法

```bash
# Dockerコンテナ内で実行
docker run --rm -v /desired/path:/charts -w /charts alpine/helm:latest \
    create mychart --starter mycompany-starter

# エイリアス化
alias helm-create-at='docker run --rm -v $(pwd):/charts -w /charts alpine/helm:latest create'

# 使用例
cd /path/to/destination
helm-create-at mychart --starter mycompany-starter
```

## CI/CDでの活用

### GitHub Actions

```yaml
name: Create Helm Chart

on:
  workflow_dispatch:
    inputs:
      chart_name:
        description: 'Chart name'
        required: true
      output_dir:
        description: 'Output directory'
        default: 'charts'
      starter:
        description: 'Starter template'
        default: ''

jobs:
  create:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Helm
      uses: azure/setup-helm@v3
    
    - name: Install Starters
      if: ${{ github.event.inputs.starter != '' }}
      run: |
        mkdir -p ~/.local/share/helm/starters
        # Starterのインストール処理
    
    - name: Create Chart
      run: |
        mkdir -p ${{ github.event.inputs.output_dir }}
        cd ${{ github.event.inputs.output_dir }}
        
        if [ -n "${{ github.event.inputs.starter }}" ]; then
          helm create ${{ github.event.inputs.chart_name }} \
            --starter ${{ github.event.inputs.starter }}
        else
          helm create ${{ github.event.inputs.chart_name }}
        fi
    
    - name: Commit Chart
      run: |
        git config --global user.name 'GitHub Actions'
        git config --global user.email 'actions@github.com'
        git add .
        git commit -m "Create chart: ${{ github.event.inputs.chart_name }}"
        git push
```

### Jenkins Pipeline

```groovy
pipeline {
    agent any
    
    parameters {
        string(name: 'CHART_NAME', description: 'Chart name')
        string(name: 'OUTPUT_DIR', defaultValue: 'charts', description: 'Output directory')
        string(name: 'STARTER', defaultValue: '', description: 'Starter template')
    }
    
    stages {
        stage('Create Chart') {
            steps {
                script {
                    sh """
                        mkdir -p ${params.OUTPUT_DIR}
                        cd ${params.OUTPUT_DIR}
                        
                        if [ -n "${params.STARTER}" ]; then
                            helm create ${params.CHART_NAME} --starter ${params.STARTER}
                        else
                            helm create ${params.CHART_NAME}
                        fi
                    """
                }
            }
        }
    }
}
```

## プロジェクト構造の推奨パターン

```bash
# モノレポ構造
project/
├── charts/              # helm-createの実行場所
│   ├── frontend/
│   ├── backend/
│   └── database/
├── applications/
└── scripts/
    └── create-chart.sh  # ラッパースクリプト

# 実行例
cd project/charts
helm create newservice --starter mycompany-starter
```

## VS Code タスクの設定

```json
// .vscode/tasks.json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "Create Helm Chart",
      "type": "shell",
      "command": "cd ${input:outputDir} && helm create ${input:chartName} ${input:starterFlag}",
      "problemMatcher": [],
      "presentation": {
        "echo": true,
        "reveal": "always",
        "focus": false,
        "panel": "shared"
      }
    }
  ],
  "inputs": [
    {
      "id": "chartName",
      "type": "promptString",
      "description": "Chart name"
    },
    {
      "id": "outputDir",
      "type": "promptString",
      "description": "Output directory",
      "default": "./charts"
    },
    {
      "id": "starterFlag",
      "type": "pickString",
      "description": "Select starter",
      "options": [
        "",
        "--starter mycompany-web",
        "--starter mycompany-api",
        "--starter mycompany-job"
      ],
      "default": ""
    }
  ]
}
```

## まとめ

`helm create`自体に出力ディレクトリ指定オプションはありませんが：

1. **シンプルな解決策**: `cd`してから実行
2. **自動化**: シェル関数やスクリプトでラップ
3. **チーム共有**: Makefile やCI/CDで標準化

最も実用的なのは、シェル関数やスクリプトでラップして、チーム内で共有することです。