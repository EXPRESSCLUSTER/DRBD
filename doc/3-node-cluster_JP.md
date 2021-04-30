3ノード DRBD クラスタ
===

はじめに
---

本ガイドでは、DRBD 9.0 を使った分散ストレージクラスタを構築する手順について説明します。
CLUSTERPRO X の詳細については、[こちら](https://jpn.nec.com/clusterpro/clpx/index.html)をご参照ください。

構成
---

### 概要

本構成では、同一のサイト内に配置された Node1、Node2 と、遠隔地のサイトに配置された Node3 の間で DRBD を利用したデータ同期を行います。
同一サイト内のデータ同期は Protocol C で行い、異なるサイト間のデータ同期は Protocol A で行います。

```
||==========[ Site1 ]==========||                        ||==[ Site2 ]==||
||                             ||                        ||             ||
|| [ Node1 ]         [ Node2 ] ||                        ||  [ Node3 ]  ||
     ^   ^             ^   ^                                   ^   ^
     |   |             |   |                                   |   |
     |   +=============+   +-----------------------------------+   |
     |   (Replication:           (Replication: Protocol A)         |
     |       Protocol C)                                           |
     |                                                             |
     +-------------------------------------------------------------+
                                 (Replication: Protocol A)
```

### 使用ソフトウェア

- Redhat Enterprise Linux 7.5
- CLUSTERPRO X 3.3.5 / 4.0.0
- DRBD 9.0

### クラスタ構成

- グループリソース
  - exec-drbd: DRBDデバイスをマウント・アンマウントします。
- モニタリソース
  - diskw-drbd: DRBDデバイスに対する WRITE 監視を行います。
  - genw-drbd: DRBDのステータスを監視します。
  - genw-drbd-service: DRBDサービスのステータスを監視します。

DRBD のインストールと設定
---

クラスタを構成する各ノードで以下の作業を実施してください。

1. DRBD 用のリポジトリを追加する
   ```sh
   # rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
   # rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm

   # yum info *drbd* | grep Name
   Name        : drbd84-utils
   Name        : drbd84-utils-sysvinit
   Name        : drbd90-utils
   Name        : drbd90-utils-sysvinit
   Name        : kmod-drbd84
   Name        : kmod-drbd90
   ```
1. DRBD をインストールする
   ```sh
   # yum install drbd90-utils kmod-drbd90
   ```
1. カーネルモジュールをロードする
   ```sh
   # insmod $(rpm -ql kmod-drbd90 | grep drbd.ko)
   ```
1. DRBDのリソースを設定する
   ```sh
   # vi /etc/drbd.d/test.res
   ```
   設定例は以下のとおりです。
   (Node1), (Node2), (Node3) は、各サーバのホスト名に置換してください。
   また、(Address1), (Address2), (Address3) は、各サーバのIPアドレスに置換してください。
   ```
   resource test {
        device /dev/drbd1;
        disk /dev/sdb1;
        meta-disk internal;

        on (Node1) {
                node-id 0;
                address (Address1):7700;
        }
        on (Node2) {
                node-id 1;
                address (Address2):7700;
        }
        on (Node3) {
                node-id 2;
                address (Address3):7700;
        }
        connection {
                net {
                        protocol C;
                }
                host (Node1) port 7710;
                host (Node2) port 7711;
        }
        connection {
                net {
                        protocol A;
                }
                host (Node1) port 7720;
                host (Node3) port 7722;
        }
        connection {
                net {
                        protocol A;
                }
                host (Node2) port 7731;
                host (Node3) port 7732;
        }
   }
   ```
1. メタデータストレージを初期化する
   ```sh
   # drbdadm create-md test
   ```
1. DRBD サービスを有効化 & 起動する
   ```sh
   # systemctl enable drbd
   # systemctl start drbd
   ```

DRBD の動作確認
---

1. (Node1 で実施) DRBD リソースを初期化する
   ```sh
   # drbdadm primary --force test
   ```
1. (Node1 で実施) ファイルシステムを作成する
   ```sh
   # mkfs.xfs -f /dev/drbd1
   ```
1. (Node1 で実施) 読み書き可能かテストする
   ```sh
   # mount /dev/drbd1 /mnt/drbd1
   # hostname >> /mnt/drbd1/hostnames
   # cat /mnt/drbd1/hostnames
   (Node1)
   # umount /mnt/drbd1
   # drbdadm secondary test
   ```
1. (Node2 で実施) 読み書き可能かテストする
   ```sh
   # mount /dev/drbd1 /mnt/drbd1
   # hostname >> /mnt/drbd1/hostnames
   # cat /mnt/drbd1/hostnames
   (Node1)
   (Node2)
   # umount /mnt/drbd1
   ```
   (補足: auto-promoteオプションが有効であれば、mount 時に自動的に primary に昇格されます。また、umount 時に自動的に secondary に降格されます)
1. (Node3 で実施) 読み書き可能かテストする
   ```sh
   # mount /dev/drbd1 /mnt/drbd1
   # hostname >> /mnt/drbd1/hostnames
   # cat /mnt/drbd1/hostnames
   (Node1)
   (Node2)
   (Node3)
   # umount /mnt/drbd1
   ```

CLUSTERPRO のインストールと設定
---

1. CLUSTERPRO をインストールし、ライセンスを登録する
1. サーバグループを設定する
   - Site1: Node1, Node2
   - Site2: Node3
1. フェイルオーバグループ failover-drbd を設定する
   - 基本設定
     - サーバグループ設定を使用する: 有効
   - 起動可能なサーバグループ
     - 1: Site1
     - 2: Site2
   - グループ属性
     - 自動フェイルオーバ
       - サーバグループ内のフェイルオーバポリシーを優先する
         - サーバグループ間では手動フェイルオーバのみを有効とする: 有効
1. EXECリソース exec-drbd を設定する
   - 詳細
     - 開始スクリプト:
       ```sh
       #! /bin/sh
       #***************************************
       #*              start.sh               *
       #***************************************

       echo "exec-drbd: start"
       mount /dev/drbd1 /mnt/drbd1
       result=$?
       echo "exit status: $result"
       exit $result
       ```
     - 終了スクリプト:
       ```sh
       #! /bin/sh
       #***************************************
       #*               stop.sh               *
       #***************************************

       echo "exec-drbd: stop"
       umount /mnt/drbd1
       drbdadm secondary test   # just for confirmation
       result=$?
       echo "exit status: $result"
       exit $result
       ```
1. ディスクモニタリソース diskw-drbd を設定する
   - 監視(共通)
     - 監視タイミング: 活性時 (対象リソース: exec-drbd)
   - 監視(固有)
     - 監視方法: WRITE(FILE)
     - 監視先: /mnt/drbd1/diskw
   - 回復動作
     - 回復動作: 回復対象を再起動、効果がなければフェイルオーバ実行
     - 回復対象: failover-drbd
1. カスタムモニタリソース genw-drbd を設定する
   - 監視(共通)
     - 監視タイミング: 活性時 (対象リソース: exec-drbd)
   - 監視(固有)
     - 監視スクリプト:
       ```sh
       #! /bin/sh
       #***********************************************
       #*                   genw.sh                   *
       #***********************************************

       echo "genw-drbd: monitor"

       role=`drbdadm role test`
       echo "role: $role"
       if [ $role != "Primary" ]; then
           exit 1
       fi

       drbdadm cstate test | grep Connected
       cstate=$?
       if [ $cstate -ne 0 ]; then
           echo "Not connected"
           exit 1
       fi

       df /mnt/drbd1 | grep /mnt/drbd1
       mstate=$?
       echo "mount status: $mstate"
       exit $mstate
       ```
   - 回復動作
     - 回復動作: 回復対象を再起動、効果がなければフェイルオーバ実行
     - 回復対象: failover-drbd
1. カスタムモニタリソース genw-drbd-service を設定する
   - 監視(共通)
     - 監視タイミング: 常時
   - 監視(固有)
     - 監視スクリプト:
       ```sh
       #! /bin/sh
       #***********************************************
       #*                   genw.sh                   *
       #***********************************************

       echo "genw-drbd-service: monitor"
       systemctl status drbd
       result=$?
       echo "exit status: $result"
       exit $result
       ```
   - 回復動作
     - 回復動作: カスタム設定
     - 回復対象: LocalServer
     - 回復スクリプト実行回数: 1
     - 回復スクリプト:
       ```sh
       #! /bin/sh
       #***********************************************
       #*	          preaction.sh                 *
       #***********************************************

       systemctl restart drbd
       result=$?
       echo "exit status: $result"
       exit $result
       ```
     - 最終動作: クラスタサービス停止とOS再起動

参考
---

- The DRBD9 User's Guide: https://docs.linbit.com/docs/users-guide-9.0/
