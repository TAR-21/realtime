## RHEL上でIntel TCC/Real-Time環境を構築し、PodmanコンテナでEtherCAT通信を実装する完全ガイド

このドキュメントは、Intel TCC対応CPUを搭載したマシン上でRHEL for Real Timeをセットアップし、IgH EtherCAT Masterをインストール・設定した上で、最終的にPodmanコンテナ内のアプリケーションからEtherCAT通信を制御するまでの一連の完全な手順をまとめたものです。

-----

### Part 1: ホストOSのリアルタイム環境構築

**目的**: OSとハードウェアをチューニングし、特定のCPUコアをリアルタイムタスク専用の「聖域」として確保する。

#### **1.1. BIOS/UEFIの設定 (ハードウェアレベルの準備)**

**目的**: OSが起動する前に、CPU自体をリアルタイム処理に最適化された「本気モード」に切り替えること。

PCのCPUは通常、省電力機能や動的な性能調整を行いますが、これらはリアルタイム処理における「予測できない遅延（ジッター）」の原因となります。「Intel® TCC Mode」はこれらの変動要因をハードウェアレベルで排除または最適化する操作です。

**手順**:

1.  PCの電源投入直後、メーカーのロゴが表示されている間に特定のキー（多くは `F2`, `Del`, `F10`, `Esc`など）を押し、設定画面に入ります。
2.  メニューの中から`Advanced` -\> `CPU Configuration`や`Performance`などの項目を探します。
3.  「**Intel® TCC Mode**」やそれに類する項目を\*\*Enabled (有効)\*\*に変更し、設定を保存して再起動します。

#### **1.2. OSの準備 (OSレベルの準備)**

**目的**: リアルタイムタスクを最優先で処理できるように設計された、特殊なOSカーネルを導入すること。

標準のOSカーネルが「公平性」を重視するのに対し、リアルタイムカーネル(`kernel-rt`)は「優先度」を絶対的な基準としてタスクを処理します。

**手順**:

1.  **リポジトリの有効化**: リアルタイム関連のパッケージが格納されているリポジトリを有効化します。
    ```bash
    sudo subscription-manager repos --enable rhel-9-for-x86_64-rt-rpms
    ```
2.  **パッケージグループのインストール**: リアルタイムカーネル本体と関連ツールをインストールします。
    ```bash
    sudo dnf groupinstall "RT"
    ```
3.  **再起動と確認**: システムを再起動し、`uname -r`コマンドの出力に `-rt` が付いていることを確認します。

#### **1.3. カーネルのチューニング (CPUコアの隔離)**

`grubby`コマンドを使い、リアルタイムタスク専用のCPUコアをOSの汎用スケジューラから隔離します。

**実行例（コア2, 3を隔離）**:

```bash
sudo grubby --update-kernel=ALL --args="isolcpus=2-3 nohz_full=2-3 rcu_nocbs=2-3"
```

実行後、システムを再起動し、`cat /proc/cmdline`で設定が反映されていることを確認します。

#### **1.4. `tuned`プロファイルの適用**

OS全体の動作モードをリアルタイム向けに最適化します。

```bash
sudo tuned-adm profile realtime
```

`tuned-adm active`で`realtime`プロファイルが適用されていることを確認します。

-----

### Part 2: IgH EtherCAT Masterのインストールと設定 (Host OS)

**目的**: EtherCAT通信を制御するマスターソフトウェアをホストOSに導入し、正しく設定する。

#### **2.1. 依存パッケージのインストール**

ソースコードのビルドに必要な開発ツールをインストールします。

```bash
sudo dnf install kernel-rt-devel-$(uname -r) gcc make autoconf automake libtool wget
```

#### **2.2. ソースコードのダウンロードと展開**

EtherCATマスターのソースコードを入手します。

```bash
cd /usr/src/
wget https://download.etherlab.org/ethercat/ethercat-1.6.7.tar.bz2
tar -xvjf ethercat-1.6.7.tar.bz2
cd ethercat-1.6.7/
```

#### **2.3. ソースコードの互換性修正 (重要)**

このバージョンのEtherCATマスターを新しいRHEL 9カーネルでビルドするには、2箇所のソースコード修正が必要です。

1.  **修正点1 (`cdev.c`)**:

    ```bash
    sudo nano +233 /usr/src/ethercat-1.6.7/master/cdev.c
    ```

    233行目を以下のように `//` を付けてコメントアウトします。
    **変更前**: `vma->vm_flags |= VM_DONTDUMP;`
    **変更後**: `// vma->vm_flags |= VM_DONTDUMP;`

2.  **修正点2 (`module.c`)**:

    ```bash
    sudo nano +115 /usr/src/ethercat-1.6.7/master/module.c
    ```

    115行目を以下のように修正し、不要な引数を削除します。
    **変更前**: `class = class_create(THIS_MODULE, "EtherCAT");`
    **変更後**: `class = class_create("EtherCAT");`

#### **2.4. ビルドとインストール**

ソースコードの修正後、ビルドとインストールを実行します。

```bash
./configure --with-linux-dir=/lib/modules/$(uname -r)/build --disable-8139too --enable-generic
make
sudo make modules_install
sudo make install
sudo ldconfig
```

#### **2.5. EtherCATマスターの設定**

EtherCAT通信に専有させるNICのMACアドレスを設定します。

1.  **MACアドレスの確認** (`enp3s0`の場合):
    ```bash
    ip addr show enp3s0
    ```
2.  **設定ファイルの編集**:
    ```bash
    sudo nano /usr/local/etc/ethercat.conf
    ```
3.  以下の2行を探し出し、MACアドレスを設定します。
    ```ini
    # 変更前
    MASTER0_DEVICE=""
    DEVICE_MODULES=""

    # 変更後 (MACアドレスはご自身の環境に合わせてください)
    MASTER0_DEVICE="cc:82:7f:92:d2:37"
    DEVICE_MODULES="generic"
    ```

#### **2.6. NetworkManagerの無効化**

EtherCATがNICを占有できるように、NetworkManagerの管理対象から除外します。

1.  設定ファイルを作成します。
    ```bash
    sudo nano /etc/NetworkManager/conf.d/99-unmanaged-devices.conf
    ```
2.  以下の内容を記述します (MACアドレスはご自身のものを使用)。
    ```ini
    [keyfile]
    unmanaged-devices=mac:cc:82:7f:92:d2:37
    ```
3.  NetworkManagerを再起動します。
    ```bash
    sudo systemctl restart NetworkManager
    ```

#### **2.7. systemdサービスの有効化**

EtherCATサービスを登録し、起動します。

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now ethercat
```

`systemctl status ethercat.service`で`active (exited)`と表示されれば成功です。

-----

### Part 3: Podmanコンテナアプリケーションの準備

**目的**: EtherCAT通信を行うサンプルアプリを、自己完結したコンテナイメージとしてビルドする。

#### **3.1. ビルド用ファイルの準備**

作業ディレクトリ（例: `/root/test`）を作成し、以下の4つのファイルを用意します。

1.  **`ethercat-1.6.7.tar.bz2`**
    `/usr/src/`からこのファイルをカレントディレクトリにコピーします。

    ```bash
    # cd /root/test/
    cp /usr/src/ethercat-1.6.7.tar.bz2 .
    ```

2.  **`simple_test.c`**

    ```c
    #include <stdio.h>
    #include "ecrt.h"

    int main(int argc, char **argv) {
        ec_master_t *master;
        ec_master_info_t master_info;

        master = ecrt_request_master(0);
        if (!master) {
            fprintf(stderr, "Failed to request master 0!\n");
            return -1;
        }
        if (ecrt_master(master, &master_info)) {
            fprintf(stderr, "Failed to get master info.\n");
            ecrt_release_master(master);
            return -1;
        }
        printf("Found %u slaves on the bus.\n", master_info.slave_count);
        ecrt_release_master(master);
        return 0;
    }
    ```

3.  **`Makefile`**

    ```makefile
    all: simple_test
    simple_test: simple_test.c
    	gcc -o simple_test simple_test.c -lethercat
    clean:
    	rm -f simple_test
    ```

4.  **`Containerfile`**

    ```dockerfile
    # ベースイメージとしてUBI 9を指定
    FROM registry.redhat.io/ubi9/ubi:latest

    # --- Part 1: EtherCATマスターのビルド ---
    # STEP 1: ビルドに必要なツールとライブラリを全てインストール
    RUN dnf install -y dnf-plugins-core && \
        dnf config-manager --set-enabled rhel-9-for-x86_64-rt-rpms && \
        dnf install -y \
        make gcc gcc-c++ autoconf automake libtool bzip2 diffutils file \
        kernel-rt-devel numactl-devel libcap-devel \
        && dnf clean all

    # STEP 2: ホストからソースコードのアーカイブをコンテナ内にコピー
    WORKDIR /usr/src/
    COPY ethercat-1.6.7.tar.bz2 .

    # STEP 3: アーカイブを展開
    RUN tar -xvjf ethercat-1.6.7.tar.bz2

    # STEP 4: ソースコードのディレクトリに移動
    WORKDIR /usr/src/ethercat-1.6.7/

    # STEP 5: ソースコードの互換性問題を修正
    RUN sed -i '233s/^/\/\//' master/cdev.c && \
        sed -i 's/class_create(THIS_MODULE, "EtherCAT")/class_create("EtherCAT")/' master/module.c

    # STEP 6: EtherCATマスターをビルドしてインストール（コンテナ内に）
    RUN KERNEL_SOURCE_DIR=/usr/src/kernels/$(ls /usr/src/kernels) && \
        ./configure --with-linux-dir=${KERNEL_SOURCE_DIR} --disable-8139too --enable-generic && \
        make && \
        make install && \
        echo "/usr/local/lib" > /etc/ld.so.conf.d/ethercat.conf && \
        ldconfig

    # --- Part 2: サンプルアプリケーションのビルド ---
    # STEP 7: アプリケーション用のディレクトリに移動
    WORKDIR /app

    # STEP 8: アプリケーションのソースコードをコピー
    COPY simple_test.c Makefile ./

    # STEP 9: アプリケーションをビルド
    RUN make

    # STEP 10: 実行するコマンドを指定
    CMD ["./simple_test"]
    ```

#### **3.2. コンテナイメージのビルド**

```bash
podman build -t my-ethercat-app .
```

-----

### Part 4: 最終的な実行と動作確認

**目的**: 構築した環境で、コンテナアプリからEtherCAT通信を成功させる。

#### **4.1. コンテナの実行**

ホストのハードウェアにアクセスするための権限を与えてコンテナを起動します。

```bash
podman run -it --rm \
  --network=host \
  --device=/dev/EtherCAT0 \
  --cap-add=SYS_NICE \
  --cpuset-cpus="2,3" \
  --ulimit memlock=-1:-1 \
  my-ethercat-app
```

*注: このコンテナは自己完結しているため、ホストのライブラリをマウント (`-v`や`-e`) する必要はありません。*

#### **4.2. 動作確認**

コンテナが正常に起動し、`Found X slaves on the bus.` (Xはスレーブの数) と表示されれば、全てのセットアップは成功です。
