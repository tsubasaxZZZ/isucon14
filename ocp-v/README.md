# ISUCON14 on OpenShift Virtualization

ISUCON14 の競技環境を OpenShift Virtualization (OCP-V) 上の VM として Ansible で構築します。
[matsuu/aws-isucon](https://github.com/matsuu/aws-isucon/tree/main/isucon14) の AMI 構築手順を OCP-V 向けに移植したもので、
VM 内のセットアップは本リポジトリの `provisioning/ansible/roles` (上流ロール) をそのまま使います。

## 構成

| VM | 役割 | vCPU | メモリ | ディスク | IP |
|---|---|---|---|---|---|
| isu1〜isu3 | 競技サーバ (c5.large 相当) | 2 (1 core x 2 threads) | 4Gi | 20Gi | 192.168.2.131〜133 |
| isu-bench | ベンチマーカー | 2 | 4Gi | 20Gi | 192.168.2.134 |
| isu-builder | ゴールデンイメージ作成用 (一時) | 4 | 8Gi | 20Gi | 192.168.2.135 |

- ネットワーク: `lab-localnet` (lab-lan に L2 直結する localnet)。NIC 1 枚、cloud-init で固定 IP。
- ストレージ: `lvms-worker` (worker-virt のローカルディスク)。
- CPU: requests 500m / limits 2 の overcommit。ゲストからは 2 vCPU に見えるが、ノード上の他 VM の負荷でスコアが揺れる。
- IP / スペック / StorageClass などは `inventory.yml` と `group_vars/` で変更できる。

## 流れ

```
00_base_image.yml  Namespace 作成 + Ubuntu 24.04 クラウドイメージを DataVolume に取り込み
10_golden.yml      isu-builder を起動 → frontend/bench をビルド → 上流ロールで構築
                   → 証明書差し替え・cloud-init clean → 停止してディスクを isucon14-golden にクローン → builder 削除
20_deploy.yml      isucon14-golden から isu1〜3 + isu-bench をクローンして起動
99_destroy.yml     片付け
```

ビルド対象のソースは **このリポジトリの git HEAD** (`git archive`) です。未コミットの変更は含まれません。

## 前提

- `oc login` 済みで、current-context が対象クラスタ
- `~/.ssh/id_ed25519.pub` (cloud-init で `ubuntu` ユーザに登録)
- 実行マシンから lab-lan (192.168.2.0/24) に到達できること
- [uv](https://docs.astral.sh/uv/)
- (推奨) StorageProfile `lvms-worker` の `cloneStrategy` を `csi-clone` にしておく。既定の copy だとディスクのクローン 1 本に約 10 分かかる (csi-clone なら数秒)。詳細は [STORAGE.md](STORAGE.md)

## 使い方

```sh
cd ocp-v
uv sync
uv run ansible-playbook site.yml          # 全部 (初回は 30〜40 分程度)

# 個別に
uv run ansible-playbook 20_deploy.yml     # ゴールデンイメージから VM を作り直す
uv run ansible-playbook 99_destroy.yml    # VM 削除 (-e destroy_images=true / -e destroy_namespace=true)
```

## ベンチマーク

```sh
ssh ubuntu@192.168.2.134
sudo -i -u isucon
./bench run --addr 192.168.2.131:443 --target https://isuride.xiv.isucon.net \
  --payment-url http://192.168.2.134:12346 --payment-bind-port 12346
```

## ブラウザで確認

手元の hosts に `192.168.2.131 isuride.xiv.isucon.net` を追記して `https://isuride.xiv.isucon.net/` を開く (自己署名証明書)。

## 本番と異なる点

- CPU は AMD EPYC ホスト上の overcommit (本番は Xeon の c5.large)。スコアの絶対値は本番と比較できない
- ディスクは LVMS (ローカル NVMe/SSD)。gp3 (3000 IOPS / 125MB/s) とは性能特性が異なる
- SSL 証明書は自己署名
- isuadmin ユーザ・envcheck は導入しない (aws-isucon と同じ)
- 参考実装は **Go のみ** (isuride-go)。他言語の処理系 (xbuildwebapp) とサービスは入れない
