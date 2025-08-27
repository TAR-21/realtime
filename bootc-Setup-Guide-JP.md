-----

# RHEL with Intel TCC on `bootc`: A Setup Guide

このドキュメントは、Red Hat Enterprise Linux (RHEL)システムを`bootc`を用いてイミュータブルOSとして構成し、Intel® Time Coordinated Computing (TCC)を有効化してリアルタイム性能を確保するための手順をまとめたものです。

## Prerequisites

  - **ハードウェア**: Intel TCCをサポートするCPUを搭載したベアメタルマシン
  - **ソフトウェア**:
      - Red Hatサブスクリプション (RHELリポジトリおよび`bootc`インストーラーISOへのアクセスに必要)
      - Podmanなどのコンテナビルドツール
  - **ネットワーク**:
      - カスタム`bootc`イメージをホストするためのコンテナレジストリ (Quay.io, Docker Hub, プライベートレジストリなど)
      - インストール時にコンテナレジストリへアクセスできるネットワーク環境

-----

## Step 1: ハードウェアとBIOS/UEFIの設定

最初に、OSが起動する前のハードウェアレベルでTCCを有効化します。

1.  ターゲットマシンを再起動し、**BIOS/UEFI設定ユーティリティ**を起動します。
2.  CPU設定やパフォーマンス関連のメニューに移動します。
3.  **Intel® Time Coordinated Computing (TCC) Mode** という項目を探し、**Enable (有効)** に設定します。
      - この設定により、レイテンシやジッターの原因となるCPUの省電力機能 (C-statesなど) が無効化・最適化され、予測可能な応答時間が得られます。
4.  設定を保存して終了します。

-----

## Step 2: TCC対応`bootc`イメージのビルド

`bootc`システムの核となるのはコンテナイメージです。リアルタイムカーネルと必要なチューニングツールを含むカスタムイメージを作成します。

### 2.1. `Containerfile`の作成

作業ディレクトリに`Containerfile`という名前で以下のファイルを作成します。

```dockerfile
# ベースイメージとしてRHEL 9を指定
FROM registry.redhat.io/rhel9/rhel:latest

# リアルタイムカーネル、tunedプロファイル、Intel CATツールをインストール
RUN dnf install -y kernel-rt tuned-profile-realtime intel-cmt-cat && \
    dnf clean all

# デフォルトのtunedプロファイルを「realtime」に設定
# このイメージから起動したシステムは、自動的にこのプロファイルが適用される
RUN tuned-adm profile realtime
```

### 2.2. イメージのビルドとプッシュ

Podmanを使い、作成した`Containerfile`からイメージをビルドし、コンテナレジストリにプッシュします。

```bash
#【変数】自身のレジストリとイメージ名に置き換えてください
IMAGE_URL="quay.io/your-username/rhel-tcc-rt:latest"

# イメージをビルド
podman build -t "${IMAGE_URL}" .

# レジストリにログインしてイメージをプッシュ
podman login quay.io
podman push "${IMAGE_URL}"
```

-----

## Step 3: ベアメタルマシンへのOSインストール

次に、作成したイメージを使って物理マシンにRHELをインストールします。

1.  Red Hatカスタマーポータルから**RHEL 9の`bootc`用インストーラーISO**をダウンロードし、USBメモリなどの起動メディアを作成します。
2.  作成したメディアからマシンを起動し、RHELのAnacondaインストーラーを開始します。
3.  インストーラーの「インストール概要」画面で、**「インストールソース」** を選択します。
4.  「Bootable Container Image」のセクションで、**Step 2でプッシュしたイメージのURL** (例: `quay.io/your-username/rhel-tcc-rt:latest`) を入力します。 **このステップが最も重要です。**
5.  ディスクのパーティショニングなど、残りの設定を完了し、インストールを開始します。

-----

## Step 4: インストール後の永続的なチューニング

インストールが完了し、システムが再起動したら、最後にカーネルブートパラメータを設定します。この設定は`bootc-kargs`コマンドで行い、永続化されます。

1.  システムにログインし、ターミナルを開きます。

2.  `bootc-kargs`コマンドを使い、リアルタイムタスク専用に隔離するCPUコアなどのパラメータを追加します。

    ```bash
    # 例: CPUコア2〜5をリアルタイムタスク用に隔離する
    sudo bootc-kargs --append 'isolcpus=2-5 nohz_full=2-5 rcu_nocbs=2-5'
    ```

3.  設定を反映させるため、システムを再起動します。

    ```bash
    sudo systemctl reboot
    ```

    これで、指定したカーネルパラメータが常に適用された状態でシステムが起動します。

-----

## Step 5: 設定の確認とOSのアップデート

### 設定の確認

システムが正しく設定されているかを確認します。

```bash
# 現在適用されているカーネルパラメータを確認
cat /proc/cmdline

# 現在アクティブなtunedプロファイルを確認
tuned-adm active
```

### OSのアップデート

OSのアップデートは、新しいバージョンのコンテナイメージをビルド＆プッシュし、`bootc upgrade`コマンドを実行するだけで完了します。カーネルパラメータは自動的に引き継がれます。

```bash
# システムのアップデートを確認・実行
sudo bootc upgrade
```
