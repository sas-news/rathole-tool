# rathole 自動設定・デプロイツール

このリポジトリは、リバースプロキシツール [rathole](https://github.com/rapiz1/rathole) を使用して、NAT 内にあるサーバー（自宅サーバーなど）のサービスを公開 IP を持つサーバー（GCP など）経由で公開するための設定ファイルとデプロイスクリプトです。

`config.yaml` という単一のファイルを編集し、デプロイスクリプトを実行するだけで、サーバー側とクライアント側の設定ファイル生成、ファイアウォール設定、サービスの再起動までを自動で行います。

## 主なファイル

| ファイル名                                         | 説明                                                                                                                                                         |
| :------------------------------------------------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [`config.yaml`](config.yaml)                       | すべての設定を記述する YAML ファイルです。GCP 接続情報、公開したいサービスなどを定義します。 **基本的にユーザーはこのファイルのみを編集します。**            |
| [`deploy.sh`](deploy.sh)                           | `config.yaml` の内容に基づき、`rathole` の設定ファイル (`server.toml`, `client.toml`) を生成し、各サーバーにデプロイしてサービスを再起動するスクリプトです。 |
| [`rathole-server.service`](rathole-server.service) | 公開サーバー側で `rathole` を systemd サービスとして実行するためのユニットファイルです。                                                                     |
| [`rathole-client.service`](rathole-client.service) | 自宅サーバー側で `rathole` を systemd サービスとして実行するためのユニットファイルです。                                                                     |

## 使用手順

### 1. 前提条件

- **デプロイスクリプト実行環境 (自宅サーバーなど)**
  - `yq` と `jq` 、`ufw` がインストールされていること。
  - 公開サーバーへ SSH 鍵認証でパスワードなしで接続できること。
- **両サーバー (公開サーバーと自宅サーバー)**
  - `rathole` の実行ファイルが `/usr/local/bin/rathole` に配置されていること。
  - systemd が利用可能であること。
  - `rathole-server.service` と `rathole-client.service` がそれぞれ `/etc/systemd/system/` に配置され、有効化 (`systemctl enable`) されていること。

```sh
wget https://github.com/rathole-org/rathole/releases/download/v0.5.0/rathole-x86_64-unknown-linux-gnu.zip
unzip rathole-x86_64-unknown-linux-gnu.zip
sudo mv rathole /usr/local/bin/rathole
sudo chmod +x /usr/local/bin/rathole
```

```sh
scp rathole-server.service <GCP_USER>@<GCP_HOST>:/etc/systemd/system/rathole-server.service
ssh <GCP_USER>@<GCP_HOST> 'systemctl enable rathole-server.service; systemctl daemon-reload'
```

```sh
cp rathole-client.service /etc/systemd/system/rathole-client.service
systemctl enable rathole-client.service
systemctl daemon-reload
```

### 2. 設定ファイルの編集

[`config.yaml`](config.yaml) を開き、ご自身の環境に合わせて各項目を編集します。

- **`connection`**: 公開サーバー (GCP) のユーザー名やホスト名を設定します。
- **`rathole_global`**: `rathole` の認証トークンや待ち受けポートを設定します。
- **`services`**: 公開したいサービスをリスト形式で追加・編集します。
  - `name`: サービス名
  - `protocol`: `tcp`, `udp`, `tcp/udp` のいずれか
  - `local_addr`: 自宅サーバー上のサービスアドレス
  - `public_port`: インターネットに公開するポート番号

### 3. デプロイの実行

設定が完了したら、自宅サーバーで以下のコマンドを実行します。
スクリプトは `sudo` を使って設定ファイルの配置やサービスの再起動を行うため、必要に応じてパスワードの入力が求められます。

```sh
bash deploy.sh config.yaml
```

スクリプトは以下の処理を自動的に行います。

1.  `config.yaml` を読み込み、`server.toml` と `client.toml` を生成します。
2.  生成した `client.toml` を自宅サーバーの `/opt/rathole/client.toml` に配置します。
3.  生成した `server.toml` を公開サーバーの `/opt/rathole/server.toml` に転送します。
4.  公開サーバーのファイアウォール (UFW) を設定し、必要なポートを開放します。
5.  公開サーバーと自宅サーバーの `rathole` サービスを再起動し、新しい設定を適用します。

以上で、設定したサービスがインターネット経由で利用可能になります。
