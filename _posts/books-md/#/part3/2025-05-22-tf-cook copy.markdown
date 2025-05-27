---
layout: post
title: "「Terraformクックブック」を読む"
date: 2025-05-22 12:00:00 +0900
categories: [blog]
tags: [書籍]
excerpt: ""
memo: "1-4, 5-10"
---

オライリーの`Terraformクックブック`という書籍を読んだので、気になったことを中心に簡単な感想や要約を書きます。(網羅性はありません)

`Terraform`を扱う上で基本的な内容が網羅された書籍になっています。

**目次**

- ToC
{:toc}

---

**サイト**

- <a href="https://www.oreilly.com/library/view/terraform/9798341633247/">オライリー</a>

---

### 1.Terraform を使い始める

- tfは複雑なインフラの管理に優れる
- k8s, GitLab, PostgreSQL などクラウド以外もサポート
- ハイブリッドやオンプレにも対応

#### 1.1.Terraform を使うとき

- 開発、ステージング、本番などの複数の環境を管理する場合
- 複数のクラウドを使う場合

#### 1.2.Terraform を使わない場合

- 単一サーバ(代わりに Ansible)
- 単一のクラウド(代わりに CFn や Azure Resource Manager など)
- インフラの複雑度と複数クラウドを使うかで導入を考える

#### 1.3.Terraform をインストールして設定する

- `sudo yum install terraform`
- `brew install opentofu` (オープンソース)
- パッケージマネージャー(yum, brew)は PATH の追加などまで自動化
- バイナリでの手動インストール
  - 複数のバージョンを実行する必要がある場合
  - 特殊なシステム要件がある場合
  - PATH の手動更新やアップグレードなどが面倒になる
- バージョンを指定してインストールすることが重要

#### 1.4.Terraform プロバイダを理解する

- `main.tf` に API キーやトークンを設定

#### 1.5.Terraform モジュールを理解する

- VPC を作成する場合など、自分で一から作るのは面倒
- パブリックモジュールを使えば簡単に設定できる
- 下の例では、CIDR, AZ, サブネット、NATゲートウェイなどを設定
```hcl
module "vpc" {
    source = "terraform-aws-modules/vpc/aws"
    ...
    cidr = "10.0.0.0/16"
    azs = ["us-west-2a", "us-west-2b", "us-west-2c"]
    private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
    public_subnets = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
    enable_nat_gateway = true
    tags = {
        Terraform = "true"
        Environment = "dev"
    }
}
```

#### 1.6.terraformコンソールでTerraformの状態を変更する

- デバッグのために、tfの状態をHCLで変更できる
- 本番環境では使ってはいけない
- 下の例は現在のtfの状態をロードするコマンド
```bash
terraform console
state.<module-name>.<resourcename>.<attribute-name> 
```

#### 1.7.HashiCorp Cloud Platform Terraformを使う
- 他にS3, Azure Blob, Cloud Storage, Consul, ローカルに保存できる
- 組織名とワークスペース名を指定する必要
```hcl
terraform {
    cloud {
    organization = "<your-organization-name>"

    workspaces {
        name = "<your-workspace-name>"
        }
    }
}
```
```bash
terraform init -backend-config="token=<HCP Terraform API token>"
```

#### 1.8.Visual Studio CodeでTerraformを使う

- Show Terraform Output(変更前にプレビュー)
- Show TerraformGraph(インフラリソースを表示)

#### 1.9.Terraform Providerのバージョン制約を管理する

- 下の例では5.0以上、6.0未満
- 破壊的変更を防ぐ可能性があるので、5.1.0など具体的なほうが良い
```hcl
provider "aws" {
    version = "~> 5.0"
    region = "us-west-2"
}
```
- ロックファイル(`.terraform.lock.hcl`)でバージョンを記録できる
- `terraform init- upgrade`で範囲内でアップロードできる

#### 1.10.バージョン管理されたTerraformコードリポジトリをデプロイするための戦略

- `github_branch_protection`リソースを使ってブランチ保護規則を設定
- 下の例では、CICDを通過しないとマージされない
```hcl
resource "github_branch_protection" "main" {
    repository = github_repository.terraform_repo.name
    ...
    required_status_checks {
        strict = true
        contexts = ["ci/terraform-run"]
    }
}
```

#### 1.11.TerraformでDockerコンテナをデプロイする

- コンテナ、イメージ、ネットワークを定義できる
```hcl
provider "docker" {}

resource "docker_container" "example" {
    image = "nginx"
    ...
}
```

#### 1.12.Terraformをアップグレードする：0.xから1.xへ、そしてその先へ

- リストアに備えて、状態ファイルのバックアップを作成
- レガシーでは、v0.12→v0.12.31→v0.13のように少しずつアップグレードしていた
- 以前は徹底的なレビューが必要だった
- v1.xxでは後方互換性に対処

#### 1.13.Terraform変数を使う

- varaiableで定義
- `terraform apply -var region=us-east-1`のように上書きできる
- モジュール化では入力と出力を設定して抽象化
- 変数のスコープに注意し、グローバル変数は避ける

#### 1.14.一貫した経験のための一貫したコード

- `required_version`を使う
```hcl
terraform {
     required_version = ">= 1.0.0"
     ...
  }
```


### 2.Terraformの基本

#### 2.1.Terraformコードの形式と検証

- `terraform fmt`は推奨スタイルに変更
- `terraform validate`は文法チェックを行うが、不十分

#### 2.2.Terraformコンソールによる迅速な実験

- nodejsやpythonのようなREPL
```bash
terraform console

> lookup({a="foo", b="bar"}, "a", "default")
"foo"
```

#### 2.3.TFLintでコード品質を向上させる

- `terraform validate `では検出できない潜在的な問題を検出
- `tflint`コマンドをダウンロード
- 例：AWSの寛容すぎるセキュリティグループの検出など
- `tflint`コマンドをCI/CDやGit precommitフックで使用

#### 2.4.TFSecでコード品質を向上させる

- `tfsec`コマンドをダウンロード
- tfファイルにコメントを追加して、特定のチェックを無視できる
- これはなるべく使用せず、使用する場合は文書化する
```hcl
#tfsec:ignore:aws-iam-no-policy-wildcards resource "aws_iam_policy" "example" 
```
- `.tfsec.yml`ファイルで設定を構成できる

#### 2.5.前提条件と事後条件を使ってコードを検証する

```hcl
# precondition
variable "instance_type" {
    ...
    validation {
        condition = contains(["t2.micro", "t2.small"], var.instance_type)
        error_message = "The instance_type must be one of: t2.micro, t2.small."
    }
}

# Postcondition
resource "aws_instance" "example" {
    ...
     lifecycle {
        postcondition {
            condition = self.tags["Name"] == "example-instance"
            error_message = "The Name tag must be set to 'example-instance'."
        }
    }
}
```
- `local-exec`プロビジョナーの`null_resource`で事後条件を定義
- 事後条件はローカルマシンで実行されるので、本番環境では適さない
- 実際はスクリプトやテストフレームワークで事後条件を検証

#### 2.6.Open Policy Agentでコードを検証する

- OPAを使用してポリシーアズコード検証を実装
- インフラをポリシーとして定義
- OPAポリシーは定期的に最新基準に応じて変更する
```plaintext
# terraform.rego

package terraform
# Define a policy to restrict allowed EC2 instance types
deny["Instance type must be t2.micro or t2.small"] {
    ...
}
```

```bash
$ terraform init
$ terraform plan -out=tfplan
$ terraform show -json tfplan > tfplan.json
$ opa eval --data terraform.rego --input tfplan.json "data.terraform.deny"
```
- CI/CDに組み込む
```yaml
- name: OPA Evaluation
  run: |
    terraform show -json tfplan > tfplan.json
    violations=$(opa eval --data terraform.rego --input tfplan.json "data.terraform.deny")
    if [ -n "$violations" ]; then
        ...
    fi
```

#### 2.7.Terraform-docsでコードをドキュメント化する

- ドキュメントを自動生成できる
- 下の例では、現在のディレクトリを指定して出力
```bash
$ terraform-docs markdown ./ > README.md
$ terraform-docs asciidoc table ./ > docs.adoc
```
- md, json, yaml, htmlなどに変換できる
- ascii記法にも対応している
- `.terraformdocs.yml`で設定
- CI/CDに組み込むと、自動でドキュメントを更新できる

#### 2.8.GitHub アクションでコード検証を自動化する

- `tfsec`などは`curl`を使わなくてもアクションが存在する
- 例: `uses: aquasecurity/tfsec-action@v1.0.3`

#### 2.9.プロバイダのバージョンアップに Dependabot を使用する

- `Dependabot`はtfにも対応しており、プロバイダを自動更新できる
```yaml
# .github/dependabot.yml
version: 2
updates:
 - package-ecosystem: "terraform"
   ...
```
- PRは必ずレビュー、変更に対して検証や自動テスト、破壊的変更を考慮
- プロバイダーの定期的アップデートはセキュリティのため非常に重要
- 本番環境へのアップロード前は必ず十分なテストを行う

#### 2.10.GitHub コードスペースと DevContainers を使う

- `.devcontainer`以下に、`devcontainer.json`と`Dockerfile`を作成

#### 2.11.Terraformでブラスト半径を制限する

- 変更の内容を局所化する


### 3.Terraform構文パターン

#### 3.1.trimspaceでユーザ入力をクリーニングする

- 空白などは予期せぬ動作を起こす可能性
    - `chomp`や`trimspace`関数
```hcl
locals {
    clean_user_input = trimspace(var.user_input)
 }

output "clean_user_input" {
    value = local.clean_user_input
 }
```

#### 3.2.接頭辞と接尾辞を削除する

```hcl
locals {
    prefix = "prefix_"
    suffix = "_suffix"
  
    without_prefix   = trimprefix(var.user_input, local.prefix)
    clean_user_input = trimsuffix(local.without_prefix, local.suffix)
 }
```

#### 3.3.正規表現を使う

- `regex`か`regexall`のreplace正規表現関数をサポート
```hcl
locals {
    masked_input = replace(var.user_input, 
        regex("\\d{3}-\\d{2}-(\\d{4})", var.user_input),
        "XXX-XX-$1"
  )
 }
```
- 使いすぎると可読性が落ちることに注意

#### 3.4.高度な文字列操作

- ネストされた`replace`関数や`format`関数
```hcl
locals {
    processed_input = replace(
        replace(
        upper(var.user_input),
        "WORLD",
        "TERRAFORM"
        ),
        "!",
        "!!!"
    )
 }
```

- 複雑な場合は`format`関数と組み合わせる
```hcl
locals {
    complex_transformation = format(
        "%s: %s",
        upper(replace(var.user_input, "hello", "greetings")),
        replace(var.user_input, "\\d+", "NumbersRemoved")
    )
}
```

#### 3.5.大文字と小文字を区別する文字列を扱う title、upper、lowerを使う

```hcl
locals {
    title_case = title(var.input_string)
    upper_case = upper(var.input_string)
    lower_case = lower(var.input_string)
}
```

#### 3.6.リストのアルファベットソート

```hcl
locals {
    sorted_list = sort(var.unsorted_list)
}
```
- 小文字の前に大文字が来る
- カスタムソートやロケールはサポートしていない
- 複雑なソートは外部で処理

#### 3.7.CIDRブロックからサブネットを作成する

```hcl
locals {
    base_cidr_block = "10.0.0.0/16"
  
    # Create four /24 subnets
    subnets = [for i in range(4) : cidrsubnet(local.base_cidr_block, 8, i)]
}
```
- 上記では "10.0.0.0/24"- "10.0.3.0/24"
- エラーを減らす、サブネットを簡単に調整できる

```hcl
locals {
    first_ip_in_each_subnet = [for subnet in local.subnets : cidrhost(subnet, 1)]
}
```
- ゲートウェイIPアドレスや固定リソースに代入
- 一貫したIPアドレス割り当てが可能になる

#### 3.8.ローカルファイルシステムを操作する

- 外部データのインポートや前提条件の検証などに利用
- `file`と`fileexsits`関数を利用

```hcl
locals {
    file_content = fileexists(var.config_file_path) ? file(var.config_file_path) : "Default content"
 }
```
- パスは相対パス
- 外部ファイルの変更を自動検知しない
    - `replaceflag`や`terraform taint`を利用
```hcl
locals {
    config_contents = {
        for path in var.config_files :
        path => fileexists(path) ? file(path) : "File not found: ${path}"
  }
 }
```

#### 3.9.レンダリングテンプレート

- 変数入力の基づいて設定ファイルやスクリプトを生成
```hcl
data "template_file" "bash_script" { 
　  template = <<-EOF
            #!/bin/bash
            echo "Hello, ${user_name}"
            EOF
    vars = {
      user_name = var.user_name
    }
}

output "rendered_script" {
    description = "Rendered bash script"
    value       = data.template_file.bash_script.rendered
}
```

#### 3.10.入力文字列の検証

```hcl
variable "environment" {
    type        = string
    ...
    validation {
        condition     = contains(["dev", "staging", "prod"], var.environment)
        error_message = "The environment must be dev, staging, or prod."
  }
 }
```
- 複数のバリエーションを使うこともできる
```hcl
variable "vpc_cidr" {
    type        = string
    description = "CIDR block for the VPC"
  
    validation {
        condition     = can(cidrhost(var.vpc_cidr, 0))
        error_message = "Must be a valid IPv4 CIDR block."
    }

    validation {
        condition     = split("/", var.vpc_cidr)[1] <=  "24" && split("/", var.vpc_cidr)[1] >= "16"
        error_message = "VPC CIDR block must be between a /16 and /24."
  }
}
```

- tfのバリエーション機能に頼るのは注意
    - 検証のために外部のデータソースやスクリプトの利用
    - メルアドのような複雑な検証は外部ライブラリの利用

#### 3.11.canとtryを使って楽観的にデータを取り出す

```hcl
variable "security_rules" {
  type = map(object({
    type        = string
    from_port   = number
    to_port     = number
    protocol    = string
    cidr_blocks = list(string)
  }))
  
  default = {
    http = {
      type        = "ingress"
      from_port   = 80
      to_port     = 80
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    },
    https = {
      type        = "ingress"
      from_port   = 443
      to_port     = 443
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }
}

resource "aws_security_group" "example" {
  name        = "example"
  description = "Example security group"
  
  dynamic "ingress" {
    for_each = var.security_rules
    content {
      type        = try(ingress.value.type, "ingress")
      from_port   = ingress.value.from_port
      to_port     = ingress.value.to_port
      protocol    = ingress.value.protocol
      cidr_blocks = can(ingress.value.cidr_blocks) ? ingress.value.cidr_blocks : null
    }
  }
}
```
- 上記では`try`と`can`を使用している

- オプションプロパティを持つ例
    - `can`, `try`を使ってデフォルト値を与えている
```plaintext
cidr_blocks = optional(list(string))
...
cidr_blocks = can(rule.cidr_blocks) ? rule.cidr_blocks : ["0.0.0.0/0"]
```

#### 3.12.入力データを順次処理する

```hcl
resource "aws_iam_role" "example" {
  count               = length(var.app_names)
  name                = "role-${var.app_names[count.index]}"
  assume_role_policy  = jsonencode({
    Version   = "2012-10-17"
    Statement = [
      {
        Action    = "sts:AssumeRole"
        Effect    = "Allow"
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      }
    ]
  })
}

resource "aws_iam_instance_profile" "example" {
  count = length(var.app_names)
  name  = "profile-${var.app_names[count.index]}"
  role  = aws_iam_role.example[count.index].name
}

resource "aws_instance" "example" {
  count                  = length(var.app_names)
  ami                    = data.aws_ami.example.id
  instance_type          = "t2.micro"
  iam_instance_profile   = aws_iam_instance_profile.example[count.index].name
  vpc_security_group_ids = [aws_security_group.example.id]
  tags = {
    Name = var.app_names[count.index]
  }
  depends_on = [aws_iam_role.example]
}
```
#### 3.13.悪い入力に対する良いエラーメッセージ

#### 3.14.Terraformステート間でデータをコンシューマする

- 他のtfステートのリソースや出⼒を参照する必要がある場合
- `terraform_remote_state`データソースを使⽤
```hcl
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "my-terraform-state"
    key    = "network/terraform.tfstate"
    region = "us-west-2"
  }
}

resource "aws_subnet" "example" {
  vpc_id     = data.terraform_remote_state.network.outputs.vpc_id
  cidr_block = "10.0.1.0/24"
}
```
- 頼りすぎないことが重要
- s3の場合、アクセス制限、バージョニング、暗号化を確認
- ステートのロックも確認

- 出力しない場合に備えて、リモートアクセスは`try`を検討
```plaintext
vpc_id = try(data.terraform_remote_state.network.outputs.vpc_id, null)
```


### 4.Terraformモジュールとプロバイダー

- AWSなどだけでなく、GitHub, Datadogなどもサポート

#### 4.1.パブリックモジュールを使用してEKSクラスタを作成する

- `variables.tf`ファイルで受け取る関数
    - `cluster_version`
    - `cluster_instance_type`
    - `cluster_asg_desired_capacity`
    - `cluster_asg_max_size`
    - `cluster_enabled_log_types`
    - `cluster_write_kubeconfig`
- EKSは内部的にEC2やASGを使っていることが分かる

- 暗号化のために、KMSの設定
```hcl
# kms.tf
resource "aws_kms_key" "eks" {
    description = "EKS Secret Encryption Key"
    deletion_window_in_days = 7
    enable_key_rotation = true
}
```

```hcl
data "aws_eks_cluster" "cluster" {
  name = module.eks.cluster_id
}

data "aws_eks_cluster_auth" "cluster" {
  name = module.eks.cluster_id
}

provider "kubernetes" {
  host                   = data.aws_eks_cluster.cluster.endpoint
  cluster_ca_certificate = base64decode(data.aws_eks_cluster.cluster.ca.0.data)
  token                  = data.aws_eks_cluster_auth.cluster.token
}

module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "20.24.0"

  cluster_name            = var.project_name
  cluster_version         = var.cluster_version
  subnets                 = module.vpc.private_subnets
  vpc_id                  = module.vpc.vpc_id
  cluster_enabled_log_types = var.cluster_enabled_log_types
  write_kubeconfig        = var.cluster_write_kubeconfig
  cluster_encryption_config = [
    {
      provider_key_arn = aws_kms_key.eks.arn
      resources        = ["secrets"]
    }
  ]
  worker_groups = [
    {
      instance_type         = var.cluster_instance_type
      asg_desired_capacity  = var.cluster_asg_desired_capacity
      asg_max_size          = var.cluster_asg_max_size
    }
  ]
}
```

#### 4.2.GitHubアクションでTerraformをリントする

#### 4.3.Terraformプロバイダの認証

```hcl
provider "aws" {
  region     = "us-west-2"
  access_key = var.aws_access_key
  secret_key = var.aws_secret_key
 }
```
- この変数は以下の方法で与える
    - CLIの利用、`.tfvars`ファイル
    - 環境変数の使用
        - `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`
    - `~/.aws/credentials`ファイル
    - EC2で実行する場合、インスタンスにIAMロールを代入
    - Hashicorp Vault

```hcl
provider "aws" {
  region                   = "us-west-2"
  shared_credentials_file  = "~/.aws/credentials"
  profile                  = "dev"
 }
```

#### 4.4.プライベートモジュールの認証

```hcl
# Terraform Cloudのプライベートモジュール
 module "my_module" {
  source  = "app.terraform.io/my-org/my-module"
  version = "1.0.0"
 }

 # Example usage of a private VPC module
 module "vpc" {
    ...
 }
```
- 認証を行うには資格情報を提供
    - `./terraformrc`ファイルの資格情報ブロックを使う
```hcl
credentials "app.terraform.io" {
    token = "your-api-token"
}
```
- 別の資格情報ファイルを指すように環境変数をTERRAFORM_CONFIGにセット
```bash
export TERRAFORM_CONFIG=/path/to/terraform.rc
```
- 通常、HCP Terraform のプライベートモジュールレジストリのようなプライベートモジュールレジストリに保存される

#### 4.5.Terraformモジュールを作成する

- main.tf, variables.tf, outputs.tfファイルの作成
- 別のファイルで呼び出す
```hcl
 module "ec2_instance" {
  source        = "./my_module"
  ami           = "ami-abc123"
  instance_type = "t2.micro"
  instance_name = "my-instance"
 }

 output "instance_id" {
  value = module.ec2_instance.instance_id
 }
 ```
- READMEには、モジュールの目的、入力、出力を記述
- モジュールのテストのために、/testsフォルダを作成する場合

#### 4.6.GitHubのシークレットをTerraformで管理する

```hcl
resource "github_actions_secret" "example_secret" {
  repository     = var.github_repository
  secret_name    = "MY_SECRET"
  plaintext_value = var.my_secret
}

variable "my_secret" {
  description = "The value of the GitHub secret"
  type        = string
  sensitive   = true
 }
```

- `sensitive`がついた変数は、ログに出力されない
- シークレットに保存することが重要
    - .tfvarsに保存し、コミットするなどはNG
- 暗号化を有効化したリモートステートの利用

#### 4.7.GitHubリポジトリをTerraformで管理する

```hcl
provider "github" {
    ...
}

resource "github_repository" "example" {
    ...
}

variable "github_owner" {
    ...
}

output "repository_url" {
  value       = github_repository.example.html_url
  ...
}
```
- オーナー、リポジトリ名などは変更されるので、変数を使う
- 耐久性、初期化、issue追跡、ウィキ、マージ戦略など
- テンプレート、トピックの追加
- 標準化された設定で多くのリポジトリを管理する場合に有効

#### 4.8.Consul KVによるダイナミックコンフィギュレーション

- 異なるステート間で依存関係を広げる必要がある場合など
```hcl
# Configure the Consul provider
 provider "consul" {
  address = "localhost:8500"
  scheme  = "http"
 }

 # Store the VPC ID in Consul KV
 resource "consul_key_prefix" "vpc" {
  path_prefix = "terraform/vpc/"
  subkeys = {
    "id" = aws_vpc.main.id
  }
 }

 # Retrieve the VPC ID from Consul KV
 data "consul_keys" "vpc" {
  key {
    name = "vpc_id"
    path = "terraform/vpc/id"
  }
 }

 # Use the retrieved VPC ID
 resource "aws_subnet" "example" {
  vpc_id     = data.consul_keys.vpc.var.vpc_id
  cidr_block = "10.0.1.0/24"
 }
```

- Consulで値を変更し、実行時に反映できるようになる

#### 4.9.サービス・ヘルス・アウェア・プロバイダの構成

- ConsulのHTTPAPIで正常性を確認

```hcl
# Data source to check service health via Consul HTTP API
 data "http" "service_health_check" {
  url = "http://localhost:8500/v1/health/service/my-service"
 }

 # Local values to process the health check response
 locals {
  service_health = jsondecode(data.http.service_health_check.body)[0]
  active_node    = local.service_health.Status == "passing" ? local.service_health.Service.Address : 
    "fallback_address"
 }
 ...
```
- httpデータソースと`jsondecode`関数を使用してAPIリクエストを処理
- 複数のサービスがあり、健全なものを使用する場合に役立つ

#### 4.10.Terraformの状態をプロバイダで消費する

- 別々のTerraformプロジェクトやモジュール間で情報を共有する場合
- `terraform_remote_state`の利用(書き込みはできない)
- リモートアクセスに必要な権限に注意
- インフラを論理的に分離する場合に役立つ(ネットワーク、データベースなど)

#### 4.11.複数の同一プロバイダーを使用する

- 1個の設定で複数のリージョンやアカウントを管理する場合

```hcl
# Configure the default AWS provider
provider "aws" {
  region = "us-west-2"
}

# Configure an additional AWS provider for the US East region
provider "aws" {
  alias  = "east"
  region = "us-east-1"
}

# EC2 instance in the US East region
resource "aws_instance" "east_server" {
  provider      = aws.east
  ...
}
```
- 二番目以降にはエイリアスを設定
- AMIはリージョン固有であることに注意
- 冗長性のために、複数の地域に同様のインフラをデプロイする場合に役立つ
- 必ずしも最適な方法ではないことに注意
    - リージョンごとに別々のtfにする
    -   tfワークスペースの使用


### 5.Terraformによるコンテナ管理
<!--
#### 5.1.ローカルとリモートのDockerイメージを使い分ける

#### 5.2.クラスタデプロイとクラスタ構成を区別する

#### 5.3.Terraformのクラスタ運用を許可する

#### 5.4.YAMLを使ってKubernetes上でコンテナをスケジューリングする

#### 5.5.HCLでKubernetes上のコンテナをスケジューリングする

#### 5.6.k2tfを使ってKubernetes YAMLをHCLに変換する

#### 5.7.Kubernetesデプロイのためにアノテーションを調整する

#### 5.8.Kubernetesデプロイのための設定を調整する

#### 5.9.KubernetesのNetworkPoliciesをTerraformで適用する

#### 5.10.Helmでコンテナをデプロイする

#### 5.11.Helmを使用してKubernetesデプロイの監視を有効にする

#### 5.12.HashiCorp Nomadでコンテナをスケジューリングする
-->


### 6.HCP TerraformとTerraform Enterprise
<!--
#### 6.1.HCP Terraformをセットアップする

#### 6.2.HCP TerraformとVCSを統合する

#### 6.3.HCP TerraformでTerraformの状態をコンシューマする

#### 6.4.GitHub ActionsでHCP Terraformを使う

#### 6.5.HCP Terraformを使った共同ワークフロー

#### 6.6.HCP TerraformとTerraform Enterpriseのトラブルシューティング

#### 6.7.ポリシーをコードとして施行する

#### 6.8.HCP Terraformでコストを管理する

#### 6.9.大規模デプロイにおけるリモート演算子の活用

#### 6.10.高度な状態管理と回復
-->


### 7.Terraformでシークレットを消費し管理する
<!--
#### 7.1.Terraformで機密データを扱う

#### 7.2.HashiCorp VaultからKey-Valueを取得する。

#### 7.3.Kubernetesネイティブ関数でKubernetesの秘密を管理する

#### 7.4.VaultとTerraformでKubernetesのシークレットを管理する

#### 7.5.秘密を環境変数として保存する

#### 7.6.秘密の監査とローテーション

#### 7.7.Terraformとパスワードマネージャで秘密を管理する

#### 7.8.Terraform Secretsによるコンプライアンスとガバナンス

#### 7.9.HashiCorp Vaultによるダイナミック・シークレット

#### 7.10.CI/CDパイプラインでシークレット・インジェクションを安全にする
-->


### 8.Terraformによる構成管理
<!--
#### 8.1.Terraformでシェルスクリプトを書く

#### 8.2.Terraformを使ってAnsibleの設定を書く

#### 8.3.Consul Key-Valueを使った動的コンフィギュレーション

#### 8.4.HTTPインタフェースからデータを消費する

#### 8.5.Terraformで条件付きロジックを適用する

#### 8.6.既存のインフラをTerraformにインポートする

#### 8.7.Terraformワークスペースを活用する

#### 8.8.Terraformテンプレートを利用する

#### 8.9.Terraformリソース間の依存関係を管理する

#### 8.10.ブルーグリーンデプロイにTerraformを使う
-->


### 9.高度なTerraformテクニック
<!--
#### 9.1.Terraformで構成ファイルを書く

#### 9.2.Cloud-Initコンフィギュレーションを書く

#### 9.3.CI/CDパイプラインにTerraformモジュールを実装する

#### 9.4.Terraformの高度な状態管理

#### 9.5.Terraformとマルチクラウド戦略

#### 9.6.スケーラブルなアーキテクチャのためのTerraform

#### 9.7.Terraformカスタムプロバイダーの開発

#### 9.8.Terraformと監視ツールを統合する

#### 9.9.Terraformでセキュリティとコンプライアンスを管理する

#### 9.10.Terraformの高度なデバッグテクニック
-->


### 10.実際の使用例
<!--
#### 10.1.Terraform Workspacesで複数の環境を管理する

#### 10.2.高可用性Webアプリケーションを地域間でデプロイする

#### 10.3.AWS EKS上でスケーラブルなKubernetesクラスタをプロビジョニングする

#### 10.4.ブルーグリーンデプロイをTerraformで実装する

#### 10.5.TerraformとAWS RDSでデータベース移行を自動化する

#### 10.6.AWS LambdaとAPIゲートウェイでサーバーレスアプリケーションをデプロイする

#### 10.7.GitOpsワークフローにコードとしてのインフラを実装する

#### 10.8.Terraformとスポットインスタンスでコストを最適化する

#### 10.9.Terraformでマルチクラウド監視ソリューションをデプロイする

#### 10.10.TerraformとAWSでディザスターリカバリーを自動化する
-->

