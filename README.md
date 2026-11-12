# scaling_pve

CNDW 2026 のデモ用 StackStorm pack。Proxmox 上の Web 層を Terraform でスケールし、HAProxy のランタイム API でバックエンドを追従させる。

## コンポーネント

| ファイル | 役割 |
| --- | --- |
| `actions/scale.yaml` | Orquesta ワークフロー `scaling_pve.scale` の定義 |
| `actions/workflows/scale.yaml` | スケーリング処理本体 |
| `actions/remote_command.yaml` | SSH 経由で単一ホストのコマンドを実行 |
| `rules/scale.yaml` | `/api/v1/webhooks/scale` を受けてワークフローを起動 |

## 全体の流れ

1. `terraform output -raw total_machines` で現在の台数を取得
2. `direction` と `total_machines` から目標台数を決定（`auto`・範囲外・変化なしは no-op）
3. Terraform ホストで `terraform apply -var total_machines=<target>`
4. scale out: 追加された VM の IP を `worker_ips` から取得し、HAProxy に `add server`
5. scale in: 減らした分の HAProxy サーバーを `del server`
6. 追加時は `set server ... state ready` と `enable health` で有効化

## 前提

### StackStorm

- pack がインストールされ、action / rule が登録済み
- 実行ユーザーから Terraform ホスト・LB ホストへ SSH できる
- `st2kv` に接続情報が登録済み

### Terraform ホスト

- `terraform` と `jq` がインストール済み
- 対象ディレクトリで `terraform apply` できる
- `total_machines` 変数を持ち、VM 台数に反映する
- `total_machines` 出力（数値）と `worker_ips` 出力（文字列リスト）を持つ
- `worker_ips` は VM のインデックス順で、新規 VM が末尾に追加されること（scale out 時に `[current:target]` で切り出すため）

### HAProxy ホスト

- `haproxy`、`socat`、`jq` がインストール済み
- ランタイム API のソケットが `level admin` で有効
- backend（既定 `web_back`）が動的サーバー対応のバランシング（`roundrobin` など）を使っている
- `socat` を sudo で実行できる

### Web VM

- HAProxy から 80/tcp で到達でき、HTTP 応答を返す

## セットアップ

### 1. pack のインストール

```bash
st2 pack install file:///path/to/cndw-st2
st2ctl reload --register-all
```

確認:

```bash
st2 action get scaling_pve.scale
st2 rule get scaling_pve.scale
```

旧名 `scaling_pve.on_request` を登録済みの環境では、リロード後に旧 action / rule を削除してください。

```bash
st2 action delete scaling_pve.on_request
st2 rule delete scaling_pve.on_request
```

### 2. 接続情報の登録

```bash
st2 key set work_dir /home/user/terraform
st2 key set tf_host 192.168.1.10
st2 key set lb_host 192.168.1.20
st2 key set ssh_username stackstorm
st2 key set ssh_password 'your-password' --encrypt
```

`work_dir` は Terraform ディレクトリ、`tf_host` / `lb_host` はそれぞれ Terraform ホストと HAProxy ホストです。action の既定値として使われます。

### 3. sudoers

SSH ユーザーがパスワードなしで `socat` を実行できるようにします。

```bash
sudo visudo -f /etc/sudoers.d/scaling-pve
```

```sudoers
<ssh_user> ALL=(root) NOPASSWD: /usr/bin/socat
```

`which socat` でパスを確認してください。

### 4. HAProxy の設定

```bash
sudo apt update
sudo apt install -y haproxy socat jq
sudo mkdir -p /run/haproxy
sudo chown haproxy:haproxy /run/haproxy
```

`/etc/haproxy/haproxy.cfg` の例:

```haproxy
global
    stats socket /run/haproxy/admin.sock mode 660 level admin

defaults
    mode http
    timeout connect 5s
    timeout client  50s
    timeout server  50s

frontend web_front
    bind *:80
    default_backend web_back

backend web_back
    balance roundrobin
    server web01 10.0.0.11:80 check
    server web02 10.0.0.12:80 check
```

注意:

- `level admin` がないと `add server` / `del server` が拒否されます。
- `balance` は `roundrobin` など動的サーバー対応の方式にしてください（`static-rr` は不可）。
- 初期状態として `min_machines` と同じ台数（例: `web01`, `web02`）を静的に定義します。
- 動的に追加したサーバーは HAProxy の reload / restart で消えます。PoC 中は reload しないでください。

設定の反映と確認:

```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl restart haproxy
sudo systemctl enable haproxy

ls -l /run/haproxy/admin.sock
echo 'show info' | sudo socat stdio /run/haproxy/admin.sock
echo 'show stat' | sudo socat stdio /run/haproxy/admin.sock | grep '^web_back,'
```

### 5. Terraform 出力の確認

```bash
st2 run scaling_pve.remote_command \
  hosts=<terraform_host> cwd=<terraform_dir> \
  cmd='terraform output -raw total_machines'

st2 run scaling_pve.remote_command \
  hosts=<terraform_host> cwd=<terraform_dir> \
  cmd='terraform output -json worker_ips'
```

## PoC 手順

### 初期状態

例として Terraform の `total_machines` を `2` にし、HAProxy に `web01` / `web02` を定義した状態から始めます。

```bash
st2 run scaling_pve.remote_command \
  hosts=<terraform_host> cwd=<terraform_dir> \
  cmd='terraform output -raw total_machines'
# => 2
```

### パラメータ

`scaling_pve.scale` の主なパラメータ:

| パラメータ | 既定値 | 説明 |
| --- | --- | --- |
| `direction` | `auto` | `scale_out` / `scale_in` / `auto`。`auto` は何もしない |
| `total_machines` | `null` | 目標台数。省略時は ±1 |
| `min_machines` | `2` | 下限 |
| `max_machines` | `5` | 上限 |
| `terraform_host` | `st2kv.system.tf_host` | Terraform 実行ホスト |
| `lb_host` | `st2kv.system.lb_host` | HAProxy ホスト |
| `terraform_dir` | `st2kv.system.work_dir` | Terraform ディレクトリ |
| `terraform_ip_output` | `worker_ips` | IP リストの output 名 |
| `lb_socket` | `/run/haproxy/admin.sock` | HAProxy ランタイムソケット |
| `lb_backend` | `web_back` | backend 名 |
| `lb_server_prefix` | `web` | サーバー名の接頭辞 |

`total_machines` を省略する場合は、`total_machines=null` を渡さず引数自体を省略してください。

### scale out

```bash
# 現在 2 → 3（+1）
st2 run scaling_pve.scale direction=scale_out

# 現在 3 → 5（絶対値）
st2 run scaling_pve.scale direction=scale_out total_machines=5
```

明示的にホストを指定する場合:

```bash
st2 run scaling_pve.scale \
  direction=scale_out \
  total_machines=4 \
  terraform_host=192.168.1.10 \
  lb_host=192.168.1.20 \
  terraform_dir=/home/user/terraform
```

### scale in

```bash
# 現在 5 → 4（-1）
st2 run scaling_pve.scale direction=scale_in

# 現在 4 → 2（絶対値）
st2 run scaling_pve.scale direction=scale_in total_machines=2
```

### no-op になる例

以下は成功扱いで終了しますが、Terraform も HAProxy も変更されません。

```bash
# auto は常に no-op
st2 run scaling_pve.scale direction=auto

# 現在値と同じ
st2 run scaling_pve.scale direction=scale_out total_machines=2

# max_machines を超える
st2 run scaling_pve.scale direction=scale_out total_machines=6

# min_machines を下回る
st2 run scaling_pve.scale direction=scale_in total_machines=1
```

## 動作確認

### StackStorm 側

```bash
st2 execution list -a scaling_pve.scale -n 10
st2 execution get <execution_id>
st2 execution tail <execution_id>
```

最終出力（`result`）に `current_machines` / `target_machines` / `direction` が表示されます。

期待されるタスク遷移:

- scale out: `read_current_count` → `inspect_capacity` → `apply_scale_out` → `capture_scale_out_ip` → `register_lb` → `finish`
- scale in: `read_current_count` → `inspect_capacity` → `apply_scale_in` → `deregister_lb` → `finish`
- no-op: `read_current_count` → `inspect_capacity` → `finish`（`target_machines` は null）

### Terraform 側

```bash
st2 run scaling_pve.remote_command \
  hosts=<terraform_host> cwd=<terraform_dir> \
  cmd='terraform output -raw total_machines'
```

### HAProxy 側

```bash
# サーバー一覧と状態（UP / MAINT など）
echo 'show stat' | sudo socat stdio /run/haproxy/admin.sock | grep '^web_back,'

# サーバーの状態詳細
echo 'show servers state' | sudo socat stdio /run/haproxy/admin.sock | grep web_back
```

scale out 後に `web03` などが追加され、`status` が `UP` になっていれば成功です。`MAINT` の場合は以下を手動で実行してください。

```bash
echo 'set server web_back/web03 state ready' | sudo socat stdio /run/haproxy/admin.sock
echo 'enable health web_back/web03'      | sudo socat stdio /run/haproxy/admin.sock
```

### 疎通確認

```bash
for i in $(seq 1 10); do
  curl -s http://<lb_host>/
done
```

各 VM で異なるレスポンス（ホスト名など）を返すようにしておくと、分散されていることを確認しやすくなります。

## タスクと実行内容

| タスク | 内容 |
| --- | --- |
| `read_current_count` | `terraform output -raw total_machines` で現在値を取得 |
| `inspect_capacity` | 方向・目標値・上下限を判定 |
| `apply_scale_out` | `terraform apply -var total_machines=<target>` |
| `capture_scale_out_ip` | `worker_ips` から追加分の IP を取得 |
| `register_lb` | `add server` → `set server ... state ready` → `enable health` |
| `apply_scale_in` | `terraform apply -var total_machines=<target>` |
| `deregister_lb` | 減った分を `del server` |
| `finish` | 終了 |

## webhook から実行

```bash
curl -X POST https://<st2-ip>/api/v1/webhooks/scale \
  -H "St2-Api-Key: <api-key>" \
  -H "Content-Type: application/json" \
  -d '{"direction":"scale_out","total_machines":5,"terraform_host":"terraform-host","lb_host":"lb-host"}'
```

`direction: scale_in` と `total_machines` で絶対値の scale in もできます。

```bash
curl -X POST https://<st2-ip>/api/v1/webhooks/scale \
  -H "St2-Api-Key: <api-key>" \
  -H "Content-Type: application/json" \
  -d '{"direction":"scale_in","total_machines":2}'
```
