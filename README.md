# k3sをインストールする方法 (1M+2W)
このマニュアルではk8sの軽量版であるk3sを簡単にインストール、クラスタを構築します。  
参考にさせていただきましたありがとうございます。    
[Raspberry Piでk3sクラスタを建ててみた](https://qiita.com/to-fmak/items/696eb97a454111435337)  
[RK3sでクラスターを作ってみよう](https://qiita.com/ogawa_pro/items/55a9dadad90d595a85a5)
k8sについてよくわかる動画[挫折したエンジニア向け-Kubernetesの仕組みをちゃんと理解する](https://www.youtube.com/watch?v=r0NpHb-6IvY)

[![](https://img.youtube.com/vi/8fFXmBuscak/0.jpg)](https://www.youtube.com/watch?v=8fFXmBuscak)

OSはUbuntuServer24.04です。
他のOSでもほとんど変わらないと思います。

## 0.前準備
### 1.rootユーザーになる
```bash
sudo su
```
### 2.apt-update
```bash
apt update && apt upgrade -y
```

## 1.WireGurdをインストールする
### 1.aptでインストール
```bash
apt install wireguard -y
```
### 2.キーを生成
```bash
wg genkey | tee wg.key | wg pubkey > wg-pub.key
```
### 3.設定ファイルの作成
`/etc/wireguard/wg0.conf`    
**Master**
```
[Interface]
Address = 10.0.0.1
ListenPort = 51871
PrivateKey = <Masterで作成した秘密鍵>

[Peer]
PublicKey = <エージェント1で作成した公開鍵>
AllowedIPs = 10.0.0.2

[Peer]
PublicKey = <エージェント2で作成した公開鍵>
AllowedIPs = 10.0.0.3
```
**Worker1**
```
[Interface]
Address = 10.0.0.2
ListenPort = 51871
PrivateKey = <Worker1で作成した秘密鍵>

[Peer]
PublicKey = <Masterで作成した公開鍵>
Endpoint = <MasterのIP>:51871
AllowedIPs = 10.0.0.1
```
**Worker2**
```
[Interface]
Address = 10.0.0.3
ListenPort = 51871
PrivateKey = <Worker2で作成した秘密鍵>

[Peer]
PublicKey = <Masterで作成した公開鍵>
Endpoint = <MasterのIP>:51871
AllowedIPs = 10.0.0.1
```
### 4.WireGurdの起動+自動起動設定
```bash
systemctl enable wg-quick@wg0
```
```bash
systemctl start wg-quick@wg0
```

### 5.grubの設定(WireGurdではないけど)
メモリサブシステムの有効化
`/etc/default/grub`のGRUB_CMDLINE_LINUX_DEFAULTに`cgroup_enable=memory`を追記
すべてのノードでおこなう。    
```
update-grub
```    
grubをアップデート  

```
reboot now
```    
再起動
    
## 2.Master側の作業
こちらもrootで実行してください。
`22,6443/tcp,7946/tcp,10250/tcp,51820/udp,51871/udp`のポートが開いていることを確認
### 1.インストール
```bash
curl -sfL https://get.k3s.io | sh -s - --node-name master --tls-san <MasterのIP> --tls-san 10.0.0.1 --tls-san 127.0.0.1 --flannel-backend wireguard-native --flannel-external-ip --flannel-iface wg0
```
### 2.トークンの確認
```bash
cat /var/lib/rancher/k3s/server/node-token
```
トークンはあとで使います
## 2.Worker側の作業
`22,6443/tcp,7946/tcp,10250/tcp,51820/udp,51871/udp`のポートが開いていることを確認
### 1.参加する
**Worker1**
```
curl -sfL https://get.k3s.io | K3S_URL=https://10.0.0.1:6443 K3S_TOKEN=<トークン> sh -s - --node-name worker1 --node-ip 10.0.0.2
```
**Worker2**
```
curl -sfL https://get.k3s.io | K3S_URL=https://10.0.0.1:6443 K3S_TOKEN=<トークン> sh -s - --node-name worker2 --node-ip 10.0.0.3
```

## 3.その他
### 1.Kubectlにworkerからもアクセスできるようにする
Masterの`/etc/rancher/k3s/k3s.yaml`をWorkerの`~/.kube/config`にコピーしてk3s-agentを再起動する    
MasterNodeも同じようにやってください    
この時にWorkerのconfigは`server: https://10.0.0.1:6443`のようにMasterのIP(VPN越し)に設定する
```bash
systemctl restart k3s #MasterNode
```
```bash
systemctl restart k3s-agent #WorkerNode
```
ノードの一覧を取得
```
kubectl get nodes
```
ノードの一覧を取得(もっと)
```
kubectl get nodes -o wide
```
## 4.Rancherのインストール
### 1.Helmのインストール
```bash
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
```
### 2.Helmにリポジトリを追加
```bash
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
```
```bash
helm repo update
```
### 3.Cert-ManagerのCRDをインストールする
```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.crds.yaml
```
### 4.名前空間を作成
```
kubectl create namespace cattle-system
```
### 5.デプロイする
```
helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --set hostname=rancher.local
```
### 6.RancherのWebUIにアクセス
アクセスしたいPCの`/etc/hosts`を編集する    
Winなら'C:\Windows\System32\drivers\etc\hosts'
LinuxやMacなら`/etc/hosts`
```
<masterのIP(VPN越しではない)> rancher.local
例:192.168.11.212 rancher.local
```
### 7.初期パスワードを取得
```bash
kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}{{"\n"}}'
```
を実行したら得られる
初期のユーザー名は`admin`
あとはブラウザで
`https://rancher.local`にアクセスし、サイトの指示に従っていけばOK
## <おまけ>アンインストール
### k3sのアンインストール
```bash
sudo /usr/local/bin/k3s-uninstall.sh #Master
```
```bash
sudo /usr/local/bin/k3s-agent-uninstall.sh #Worker
```
### ファイルの削除(kube)
```bash
rm -r ~/.kube/
```
### wireguardのアンインストール
```bash
sudo apt remove wireguard
```
### ファイルの削除(wireguard)
```bash
sudo rm /etc/wireguard/wg0.conf
```
## 制作
**sskrc**

---

今後の改良に関する提案やバグ報告は、お気軽にIssueを通してご連絡ください。
