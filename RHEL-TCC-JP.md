## RHEL上でIntel TCCを有効化し、Podmanコンテナでリアルタイム性能を検証する手順書

このドキュメントは、Intel TCC対応CPUを搭載したマシン上でRHEL for Real Timeをセットアップし、Podmanコンテナ内のアプリケーションでそのリアルタイム性能を実証するための一連の手順をまとめたものです。

### Part 1: ホストOSのリアルタイム環境構築

**目的**: OSとハードウェアをチューニングし、特定のCPUコアをリアルタイムタスク専用の「聖域」として確保する。

#### **Step 1. BIOS/UEFIの設定**

  - マシンを起動しBIOS/UEFI設定画面に入り、「**Intel® Time Coordinated Computing (TCC) Mode**」を**有効 (Enable)** にする。

#### **Step 2. OSの準備**

  - OSとして「**RHEL for Real Time**」をインストールし、リアルタイムカーネル (`kernel-rt`) でシステムを稼働させる。

#### **Step 3. カーネルのチューニング (CPUコアの隔離)**

  - `grubby`コマンドを使い、リアルタイムタスク専用のCPUコアをOSの汎用スケジューラから隔離する。
  - **実行例（コア2, 3を隔離）**:
    ```bash
    sudo grubby --update-kernel=ALL --args="isolcpus=2-3 nohz_full=2-3 rcu_nocbs=2-3"
    ```
  - **反映と確認**:
    システムを再起動後、`cat /proc/cmdline`を実行し、上記パラメータが含まれていることを確認する。

#### **Step 4. `tuned`プロファイルの適用**

  - OS全体の動作モードをリアルタイム向けに最適化する。
  - **実行コマンド**:
    ```bash
    sudo tuned-adm profile realtime
    ```
  - **確認**:
    `tuned-adm active`を実行し、`Current active profile: realtime`と表示されることを確認する。

-----

### Part 2: リアルタイム検証用コンテナの作成

**目的**: リアルタイム性能を測定する標準ツール`cyclictest`を、ソースコードからビルドしてコンテナイメージに含める。

#### **Step 1. `Containerfile`の作成**

  - 以下の内容で`Containerfile`を作成する。これにより、サブスクリプションに依存せずに`rt-tests`ツールを導入できる。
    ```dockerfile
    # ベースイメージとしてUBI 9を指定
    FROM registry.redhat.io/ubi9/ubi:latest

    # STEP 1: rt-testsのビルドに必要なツールとライブラリをインストール
    RUN dnf install -y \
        git \
        make \
        gcc \
        procps-ng \
        numactl-devel \
        libcap-devel \
        && dnf clean all

    # STEP 2: rt-testsのソースコードをダウンロードし、ビルドしてインストール
    RUN git clone https://git.kernel.org/pub/scm/utils/rt-tests/rt-tests.git && \
        cd rt-tests && \
        make all && \
        make install

    # デフォルトのコマンドを設定
    CMD ["cyclictest", "--help"]
    ```

#### **Step 2. Podmanイメージのビルド**

  - 上記`Containerfile`を元に、`rt-test-app`という名前のコンテナイメージを作成する。
    ```bash
    sudo podman build -t rt-test-app .
    ```

-----

### Part 3: リアルタイム性能の測定と検証

**目的**: OSに高負荷をかけながら、隔離したCPUコア上で動作するリアルタイムタスクの性能が劣化しないことを証明する。

#### **Step 1. 負荷ツールのインストール（ホストOS側）**

  - 負荷生成ツール`stress-ng`をホストOSにインストールする。
    ```bash
    sudo dnf install -y epel-release
    sudo dnf install -y stress-ng
    ```

#### **Step 2. 性能測定の実行（ターミナル2つ使用）**

  - **ターミナル① - `cyclictest`の起動**:
    隔離したCPUコア（2,3）上で、リアルタイム権限を与えて`cyclictest`を起動する。

    ```bash
    podman run --rm --cap-add=SYS_NICE --cpuset-cpus="2,3" --ulimit memlock=-1:-1 rt-test-app chrt -f 80 cyclictest -t1 -p 80 -i 1000 -m
    ```

  - **ターミナル② - `stress-ng`による負荷印加**:
    `cyclictest`が動いていない\*\*隔離されていないCPUコア（例: 0,1）\*\*に対して高負荷をかける。

    ```bash
    taskset -c 0,1 stress-ng --cpu 2 --vm 2 --timeout 120s
    ```

#### **Step 3. 結果の評価**

  - `stress-ng`が動作している間、ターミナル①の`cyclictest`の`Max`（最大遅延時間）の値を確認する。
  - **成功の基準**: ホストOSに高い負荷がかかっていても、`Max`の値が**常に低い値（数μs〜20μs程度）で安定していること**。
  - これが確認できれば、カーネルチューニングによってリアルタイムタスクが他のタスクの影響から完全に保護されていることの証明となります。
