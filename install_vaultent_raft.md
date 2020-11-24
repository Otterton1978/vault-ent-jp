# Vault Enterprise 導入・初期設定 (Raft Storage)
[Vault Enterprise](https://www.vaultproject.io/) を初めて利用するユーザーが、初期導入・設定をスムーズに進めるための日本語のガイド。[HashiCorpオフィシャルのドキュメント](https://learn.hashicorp.com/tutorials/vault/raft-deployment-guide?in=vault/day-one-raft)をベースに作成しています。

## 前提
* Storage BackendとしてIntegrated Storageを利用
* Vaultが稼働するLinuxサーバーインスタンスを事前作成済み(サイジングについては[こちらを参照](https://learn.hashicorp.com/tutorials/vault/raft-reference-architecture?in=vault/day-one-raft#sizing-for-vault-servers))

## リファレンスアーキテクチャー
https://learn.hashicorp.com/tutorials/vault/raft-reference-architecture?in=vault/day-one-raft#design-summary

## ネットワーク要件
以下のポートにて通信が可能なようにFirewall/Security Group等の設定を行うこと
* 8200/tcp - クライアントよりVault API, GUIへのアクセスポート
* 8201/tcp - Vaultクラスター内のノード間Gossip通信, リクエスト転送, Vaultクラスター間のレプリケーション

詳細は[コチラ](https://learn.hashicorp.com/tutorials/vault/raft-reference-architecture?in=vault/day-one-raft#network-connectivity-details)
## 導入

#### Enterprise バイナリのダウンロード
[HashiCorpのVaultバイナリのリポジトリ](https://releases.hashicorp.com/vault/)より、Enterprise版のバイナリをダウンロード
``` Vault Binary Download
export VAULT_URL="https://releases.hashicorp.com/vault" \
         VAULT_VERSION="1.6.0+ent"
$ curl \
      --silent \
      --remote-name \
     "${VAULT_URL}/${VAULT_VERSION}/vault_${VAULT_VERSION}_linux_amd64.zip"

$ curl \
      --silent \
      --remote-name \
      "${VAULT_URL}/${VAULT_VERSION}/vault_${VAULT_VERSION}_SHA256SUMS"

$ curl \
      --silent \
      --remote-name \
      "${VAULT_URL}/${VAULT_VERSION}/${VAULT_VERSION}/vault_${VAULT_VERSION}_SHA256SUMS.sig"
```


* VAULT_VERSIONには、`導入バージョン+ent`を指定(i.e. 1.6.0+ent)
* Linuxに導入する場合、プラットフォームは`linux_amd64`を指定

#### zipファイルを解凍、/usr/local/bin以下に配置
``` Vault unzip
$ unzip vault_${VAULT_VERSION}_linux_amd64.zip
$ sudo chown root:root vault
$ sudo mv vault /usr/local/bin/
$ vault -version
Vault v1.6.0+ent
```
* 上記以外のディレクトリに配置する場合はPATHの設定を行う

#### Vault実行に関連する事前設定

``` Vault setup
# vaultコマンドのAuto completeを有効化
$ vault -autocomplete-install
$ complete -C /usr/local/bin/vault vault

# mlock syscallの非rootユーザーによる実行を許可
$ sudo setcap cap_ipc_lock=+ep /usr/local/bin/vault

# vault実行ユーザーの作成
$ sudo useradd --system --home /etc/vault.d --shell /bin/false vault
```

## サーバー構成
#### Vaultサーバーの構成ファイルを作成
```
$ sudo mkdir --parents /etc/vault.d
$ sudo touch /etc/vault.d/vault.hcl
$ sudo chown --recursive vault:vault /etc/vault.d
$ sudo chmod 640 /etc/vault.d/vault.hcl
```

#### スタンザの設定
上のステップで作成した`/etc/vault.d/vault.hcl`の編集を行う。以下、環境の違いにより、最低限必要な構成例（

##### A. 開発・サンドボックスなど、シングル構成、ノンセキュア通信が許容できる環境
``` vault.hcl
# リスナー設定
listener "tcp" {
  address       = "0.0.0.0:8200"

  # 開発・サンドボックスなどでNon-TLS通信を許可する場合
  tls_disable = "true"
}

# Storage Backendの構成（raft storage）
storage "raft" {
  path = "/opt/raft"
  node_id = "raft_node_1"
}

# GUIを有効化
ui = true

# Integrated Storageを使用する場合の推奨
disable_mlock = true
```
##### B. ステージング・本番など、クラスター構成、セキュア通信が必要な環境（各ノード毎に構成）
``` vault.hcl
# リスナー設定
listener "tcp" {
  address       = "0.0.0.0:8200"

  # クラスター環境では必須（一台構成の場合、不要）
  cluster_address = "xxx.xxx.xxx.xxx:8201"

  # TLSによるEnd-to-Endの暗号化のため、本番・Stagingでは必須
  tls_cert_file = "/path/to/fullchain.pem" # 証明書
  tls_key_file  = "/path/to/privkey.pem" # 秘密鍵

}

# Storage Backendの構成（raft storage）
storage "raft" {
  path = "/opt/raft"
  node_id = "raft_node_1"

  # 本番クラスター構成の場合に必須（Vaultサーバー台数分記述）
  retry_join {
    leader_api_addr = "http://xxx.xxx.xxx.xxx:8200"
  }
  retry_join {
    leader_api_addr = "http://xxx.xxx.xxx.xxx:8200"
  }
  retry_join {
    leader_api_addr = "http://xxx.xxx.xxx.xxx:8200"
  }

}

# GUIを有効化
ui = true

# Integrated Storageを使用する場合の推奨
disable_mlock = true

# Client Redirection用のURL
api_addr = "Full URL of Vault API endpoint"

# クラスター間通信に使用されるアドレス
cluster_addr = "https://xxx.xxx.xxx.xxx:8201"
```


#### Raft storage DB の保管先ディレクトリを作成
```
$ sudo mkdir /opt/raft
$ sudo chown -R vault:vault /opt/raft
```

## Vaultサービスの構成（systemd）
#### Vaultサービス定義ファイル作成
```
$ sudo touch /etc/systemd/system/vault.service
```
#### サービス定義ファイル編集
```
[Unit]
Description="HashiCorp Vault - A tool for managing secrets"
Documentation=https://www.vaultproject.io/docs/
Requires=network-online.target
After=network-online.target
ConditionFileNotEmpty=/etc/vault.d/vault.hcl
StartLimitIntervalSec=60
StartLimitBurst=3

[Service]
User=vault
Group=vault
ProtectSystem=full
ProtectHome=read-only
PrivateTmp=yes
PrivateDevices=yes
SecureBits=keep-caps
AmbientCapabilities=CAP_IPC_LOCK
Capabilities=CAP_IPC_LOCK+ep
CapabilityBoundingSet=CAP_SYSLOG CAP_IPC_LOCK
NoNewPrivileges=yes
ExecStart=/usr/local/bin/vault server -config=/etc/vault.d/vault.hcl
ExecReload=/bin/kill --signal HUP $MAINPID
KillMode=process
KillSignal=SIGINT
Restart=on-failure
RestartSec=5
TimeoutStopSec=30
LimitNOFILE=65536
LimitMEMLOCK=infinity

[Install]
WantedBy=multi-user.target
```

## Vault起動

```
$ sudo systemctl enable vault
$ sudo systemctl start vault
$ sudo systemctl status vault
```
## Vaultの初期化処理
Vaultサービス起動後、初期化処理を1回実行することで、seal/unsealのオペレーションに必要なunseal key、及びVaultの特権操作を行うことができるrootトークンを生成。[リンク先](https://github.com/hashicorp-japan/vault-workshop-jp/blob/master/contents/hello-vault.md#vault%E3%81%AE%E5%88%9D%E6%9C%9F%E5%8C%96%E5%87%A6%E7%90%86)の"Vaultの初期化処理"のセクションに詳細な手順あり



## ライセンス適用
メールにて送付される"Vault Enterprise Onboarding Guide"に添付されているライセンスキーをコピーし、以下のvaultコマンドを実行
```
$ vault write /sys/license "text=<ライセンスキー>"
```
実行後、ライセンスの適用状況を確認
```
$ vault read /sys/license
Key Value
--- -----
expiration_time 2020-01-02T12:02:47.792487-08:00
features [HSM Performance Replication ... ]
license_id 7abeb972-7fd6-8d3b-46cb-597...
performance_standby_count 9999
start_time 2020-01-02T11:27:47.792487-08:00
```
* Vaultサービス起動後、30分以内にライセンスの適用を実施すること

## 注意事項
* このガイドはクイックにVaultを導入して機能確認できることを主目的としています。本番環境で考慮する設計要素（ハードニング、モニタリング、バックアップ、監査ログ、DR etc.）は対象外としています。[Vaultの公式ドキュメント](https://www.vaultproject.io/docs)や[HashiCorp Learn](https://learn.hashicorp.com/vault)を参照してください
