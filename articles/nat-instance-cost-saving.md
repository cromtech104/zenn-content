---
title: "個人開発のAWSコスト削減: NAT GatewayをEC2自前NATに置き換えた話"
emoji: "💸"
type: "tech"
topics: ["aws", "terraform", "lambda", "個人開発", "vpc"]
published: true
---

個人で[RepoCarta](https://repocarta.jp/)というSaaSを作っていて、LambdaをVPC内に置いてRDSに接続する構成にした。しばらくしてAWSの請求を確認したら、NAT Gatewayの固定費だけで月$45かかっていた。サービスがまだ収益ゼロの段階でこれはきつい。

EC2 t3.nanoで代替したら月$4になったので、その手順とTerraformのコードをまとめておく。Amazon Linux 2023はiptablesではなくnftablesを使うので、そこだけ注意が必要だった。

## なぜNATが要るのか

LambdaをプライベートサブネットのVPC内に置くと、デフォルトではインターネットに出られない。GitHub APIやAnthropic APIへのリクエストがすべて失敗する。NAT経由で外に出る経路を作る必要がある。

NAT Gatewayを使えば設定は簡単だが、固定費だけで月$45かかる。データ転送量によってはさらに上乗せされる。EC2 t3.nanoで自前NATを立てれば月$4で済む。単一障害点になるのは許容して、コストを優先した。

## Terraformの実装

### VPCとサブネット

```hcl
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true
}

# パブリックサブネット（NATインスタンスを置く）
resource "aws_subnet" "public" {
  count                   = 2
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.${count.index + 1}.0/24"
  availability_zone       = ["ap-northeast-1a", "ap-northeast-1c"][count.index]
  map_public_ip_on_launch = true
}

# プライベートサブネット（LambdaとRDSを置く）
resource "aws_subnet" "private" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index + 11}.0/24"
  availability_zone = ["ap-northeast-1a", "ap-northeast-1c"][count.index]
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
}

resource "aws_route_table_association" "public" {
  count          = 2
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}
```

### NATインスタンスのセキュリティグループ

Lambdaからの通信だけ受け付けて、外への通信は全部許可する。

```hcl
resource "aws_security_group" "nat" {
  name   = "${var.project}-${var.env}-nat"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port       = 0
    to_port         = 0
    protocol        = "-1"
    security_groups = [aws_security_group.lambda.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

### `source_dest_check = false` を忘れると動かない

EC2インスタンス本体で一番はまったのがここ。デフォルトでEC2は「宛先が自分でないパケット」を捨てる。これをオフにしないとNATとして機能しない。

```hcl
resource "aws_instance" "nat" {
  ami                         = var.nat_ami_id
  instance_type               = "t3.nano"
  subnet_id                   = aws_subnet.public[0].id
  vpc_security_group_ids      = [aws_security_group.nat.id]
  associate_public_ip_address = true
  source_dest_check           = false  # ← これがないとNATにならない
  iam_instance_profile        = aws_iam_instance_profile.nat.name

  user_data = file("${path.module}/nat_user_data.sh")
}
```

### プライベートサブネットのルートテーブル

デフォルトルートをNATインスタンスのENIに向ける。

```hcl
resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block           = "0.0.0.0/0"
    network_interface_id = aws_instance.nat.primary_network_interface_id
  }
}

resource "aws_route_table_association" "private" {
  count          = 2
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private.id
}
```

### Amazon Linux 2023はnftablesを使う

ここも詰まった。AL2023ではiptablesが使えず、nftablesで書く必要がある。ネット上のサンプルはiptablesのものが多いので注意。

```bash
#!/bin/bash
set -e

echo "net.ipv4.ip_forward = 1" > /etc/sysctl.d/99-ip-forward.conf
sysctl -p /etc/sysctl.d/99-ip-forward.conf

dnf install -y nftables

# AL2023のnftables.serviceは /etc/sysconfig/nftables.conf を読む
cat > /etc/sysconfig/nftables.conf << 'NFTEOF'
#!/usr/sbin/nft -f
flush ruleset

table ip nat {
    chain POSTROUTING {
        type nat hook postrouting priority srcnat; policy accept;
        oifname != "lo" masquerade
    }
}
NFTEOF

systemctl enable --now nftables
```

`masquerade`でプライベートIPをEC2のパブリックIPに変換して外へ出す。

### SSMでSSHなしに入れるようにしておく

NAT インスタンスにSSHポートを開けたくないので、SSM Session Managerのロールを付けておく。EC2にそのまま入って設定を確認するときに使う。

```hcl
resource "aws_iam_role" "nat_instance" {
  name = "${var.project}-${var.env}-nat"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Action    = "sts:AssumeRole"
      Principal = { Service = "ec2.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "nat_ssm" {
  role       = aws_iam_role.nat_instance.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

resource "aws_iam_instance_profile" "nat" {
  name = "${var.project}-${var.env}-nat"
  role = aws_iam_role.nat_instance.name
}
```

## AMI IDの取得

```bash
aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=al2023-ami-2023.*-x86_64" \
  --query "sort_by(Images, &CreationDate)[-1].ImageId" \
  --output text \
  --region ap-northeast-1
```

Terraformで自動追従させると`apply`のたびにEC2が再作成されることがあるので、`variables.tf`に固定値で書いておく方が安全。

## やっておかないといけないこと

**EC2が止まると外に出られなくなる。** 単一障害点なので、インスタンスが落ちたらLambdaから外への通信が全部止まる。止まったときはコンソールかTerraformで再起動する。

**EC2を再作成するとENIが変わる。** インスタンスタイプを変えるなど`terraform apply`でEC2が再作成されるとネットワークインターフェースIDが変わり、ルートテーブルの参照が壊れる。AMI IDを固定しておけば基本的には起きない。

---

月$40、年$480の差は個人開発の初期には大きい。可用性が必要になったタイミングでNAT Gatewayに戻せばいい。
