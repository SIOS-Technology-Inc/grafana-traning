# Azure VMの操作コマンド（作成/接続/削除）
## 環境
以下の環境で実行したAzure VMの操作コマンドを記録した。
ほとんどのコマンドは、azure cliが利用可能な環境であれば動作するはずだが、必要に応じてコマンドを読み替えて実行すること。
- WSL2上で動作するUbuntu20.04.5 LTS
- azure cli 2.50.0 インストール済

### azureへのログイン
以降のコマンドを実行する前に、以下のコマンドでAzureにログインしておく。ブラウザを使ってAzureへの対話型ログインができる。
```bash
az login
```

## VM以外のリソースの作成
###リソースグループの作成
```bash
resourcegroup="rg-grafana-seminar-2"
location="eastus"
az group create --name $resourcegroup --location $location
```
### 秘密鍵の作成
sshキーは、他のVMを建てる際にも同じものを共用したいので、Azure上に事前に作成しておく
```bash
az sshkey create --name "${resourcegroup}_key" --resource-group $resourcegroup
```

秘密鍵を作成するコマンドを実行すると、```No public key is provided. A key pair is being generated for you.
Private key is saved to "/home/develop/.ssh/1697770064_9876912".
Public key is saved to "/home/develop/.ssh/1697770064_9876912.pub".``` というメッセージが表示され、キーファイルが保存されたパスが表示される。

これらの情報を基に、後ほど管理がしやすいように保存された秘密鍵と公開鍵の名前を変更しておく。

```bash
# ~/.ssh ディレクトリ配下での操作
key=1697770064_9876912

mv $key "${resourcegroup}_key"
mv "${key}.pub" "${resourcegroup}_key.pub"
```

プライバシーのために、秘密キーファイルのアクセス許可を変更しておく。
```bash
# ~/.ssh ディレクトリ配下での操作
chmod 600 "${resourcegroup}_key"
```

## VMリソースの作成
作成するVM名、VMのユーザー名、VMのイメージを変数で指定して、仮想マシンを作成する。
秘密鍵には、先ほど作成した秘密鍵を指定している。

バックアップ用に作成するマシンの名前は、```vm-grafana-postgres-backup```、リストア用に作成するマシンの名前は```vm-grafana-postgres-restore```として、同様のコマンドで2つのVMを作成する。
アップグレードのデモ用には、```vm-grafana-postgres-upgrade```という名前のVMを作成する。
```bash
vmname="vm-grafana-postgres-backup"
username="azureuser"
image="Canonical:0001-com-ubuntu-server-jammy:22_04-lts-gen2:22.04.202309190"
az vm create \
	--resource-group $resourcegroup \
	--name $vmname \
	--image $image \
	--public-ip-sku Standard \
  --admin-username $username \
	--ssh-key-name "${resourcegroup}_key"
```
## DNSの設定 (任意)
作成されたVMへのアクセスをDNS名で行えるようにするため、VMと共に作成されたパブリックIPアドレスに対してドメイン名を設定する。
VMと共に作成されたパブリックIPアドレスのドメイン名を取得するためには、以下のコマンドを実行する。
```bash
az network public-ip list --resource-group $resourcegroup --output table
```
上記のコマンドにより表示されたパブリックIPアドレスの名前を指定し、以下のコマンドでドメイン名を設定する。以下のコマンドは、VM名の変数が保存された状態で実行すると、自動的にパブリックIPの名前を補完する。
```bash
az network public-ip update --resource-group $resourcegroup --name "${vmname}PublicIP" --dns-name $vmname
```

## 仮想マシンへの接続
以下のコマンドで秘密鍵を使って仮想マシンへの接続を行うことができる。筆者はRloginというツールを用いてVMに接続した。
```bash
ssh -i ~/.ssh/rg-grafana-seminar-2_key  azureuser@<接続先VMのIPアドレス or DNS名>
```

## リソースの削除
デモが終了し、リソースが不要になった場合は以下のコマンドで削除できる。（必要に応じて変数は再定義する。）
### VMの削除
```bash
az vm delete \
  --resource-group $resourcegroup \
  --name $vmname
```
### リソースグループの削除
```bash
az group delete \
	--name $resourcegroup
```

### VMの初期設定
- タイムゾーンの設定