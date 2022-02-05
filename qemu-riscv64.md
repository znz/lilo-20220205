# qemuのriscv64にDebianを入れてみた

author
:   Kazuhiro NISHIYAMA

content-source
:   LILO&東海道らぐオンラインミーティング

date
:   2022-02-05

allotted-time
:   20m

theme
:   lightning-simple

# 自己紹介

- 西山 和広
- Ruby のコミッター
- twitter, github など: @znz
- 株式会社Ruby開発 www.ruby-dev.jp

# 環境

- ホスト側 Ubuntu 21.10 (impish), (Ubuntu 20.04.3 LTS (focal) でも動作確認)
- QEMU emulator version 6.0.0 (Debian 1:6.0+dfsg-2expubuntu1.1)
- ゲスト側: Debian GNU/Linux bookworm/sid

# 参考

- <https://wiki.debian.org/RISC-V> の情報を元に実行
- <https://blog.n-z.jp/blog/2022-01-29-debian-qemu-riscv64.html> に公開した情報を再構成

# chroot で debootstrap

```
sudo apt-get install debootstrap qemu-user-static \
binfmt-support debian-ports-archive-keyring
sudo debootstrap --arch=riscv64 --keyring \
/usr/share/keyrings/debian-ports-archive-keyring.gpg \
--include=debian-ports-archive-keyring unstable \
/tmp/riscv64-chroot http://deb.debian.org/debian-ports
```

ここは Debian Wiki のコマンドそのまま

# chroot 環境で事前設定

```
CHROOT=/tmp/riscv64-chroot
sudo chroot "$CHROOT" apt-get update
sudo chroot "$CHROOT" apt-get -y install etckeeper
sudo chroot "$CHROOT" apt-get -y full-upgrade
```

- `CHROOT` はこの後も利用
- `etckeeper` はいつも入れているのでここでも入れた

# 他の設定

```
sudo chroot "$CHROOT" ln -sf /dev/null \
 /etc/systemd/system/serial-getty@hvc0.service
sudo chroot "$CHROOT" apt-get install -y \
 linux-image-riscv64 u-boot-menu
sudo chroot "$CHROOT" apt-get install -y \
 openntpd ntpdate
sudo chroot "$CHROOT" sed -i \
 's/^DAEMON_OPTS="/DAEMON_OPTS="-s /' \
 /etc/default/openntpd
printf '\nU_BOOT_PARAMETERS="rw noquiet'\
' root=/dev/vda1"\nU_BOOT_FDT_DIR="noexist"\n' \
| sudo chroot "$CHROOT" tee -a /etc/default/u-boot
sudo chroot "$CHROOT" u-boot-update
```

(ネットワーク設定とパスワード設定は後で)

# ネットワーク設定

```
printf 'auto lo\niface lo inet loopback\n' | \
 sudo chroot "$CHROOT" tee /etc/network/interfaces.d/lo
printf 'auto eth0\niface eth0 inet dhcp\n' | \
 sudo chroot "$CHROOT" tee /etc/network/interfaces.d/eth0
echo "debian-riscv64" | sudo chroot "$CHROOT" tee /etc/hostname
echo "10.0.2.15 debian-riscv64" | sudo chroot "$CHROOT" tee -a /etc/hosts
```

- 自動化しやすいように `/etc/network/interfaces` は書き換えず `interfaces.d` にファイル作成
- `/etc/hosts`にも追加して`sudo: unable to resolve host debian-riscv64: Temporary failure in name resolution`対策

# ユーザー設定

```
NEW_USER_ID=10001
NEW_USER_NAME=user1
#NEW_USER_PASSWORD=password
NEW_USER_CRYPTED_PASSWORD='$6$(略)'
sudo chroot "$CHROOT" groupadd -g "$NEW_USER_ID" "$NEW_USER_NAME"
sudo chroot "$CHROOT" useradd -d "/home/$NEW_USER_NAME" -m \
 -g "$NEW_USER_NAME" -u "$NEW_USER_ID" -p "$NEW_USER_CRYPTED_PASSWORD" \
 -s /bin/bash "$NEW_USER_NAME"
sudo chroot "$CHROOT" apt-get install -y sudo
echo "$NEW_USER_NAME ALL=(ALL) NOPASSWD:ALL" | \
 sudo chroot "$CHROOT" tee "/etc/sudoers.d/$NEW_USER_NAME"
```

- gid や uid を固定するため `groupadd` と `useradd` で個別に追加
- `useradd` の `-p` は 生パスワードではない
- `sudo` の設定も追加

# ユーザー設定 (失敗)

```
$ echo "$NEW_USER_NAME:$NEW_USER_PASSWORD" | \
   sudo chroot "$CHROOT" chpasswd
chpasswd: (user user1) pam_chauthtok() failed, error:
Authentication token manipulation error
```

- 仮想環境で弱いパスワードなのでそのまま書いておきたかったが、`chroot` 環境では使えなかった

# cloud-init

```
sudo chroot "$CHROOT" apt-get -y install cloud-init openssh-server
```

- 最終的には `cloud-init` を入れて、ネットワーク設定やユーザー設定はそちらに任せることに
- `openssh-server` の設定もしてくれる
- 一緒にインストールしておかないと、 `PasswordAuthentication yes` だけの `/etc/ssh/sshd_config` が作成されてしまっておかしなことに

# cloud-initとは

- 主に EC2 などのクラウド環境での初期設定ツール
- LXC/LXD や multipass などでも対応
- openssh のホスト鍵の再生成などディスクイメージの再利用をするときに必要な機能も入っている

# 使い分け

いまさらで物凄く恐縮ですが、cloud-initについて勉強してみた <https://qiita.com/MUCHIUCHI_OJISAN/items/9c013e87bd2dbeb4ca4a> より

![](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/133151/17565010-9d5f-0267-6e30-ac77d6aaf13b.png){:relative_height='100'}

(TODO: 公開スライドでは画像埋め込みは消す)

# ディスクイメージ作成前の最後の処理

```
sudo chroot "$CHROOT" sed -i -e 's/^# \(ja_JP\.UTF-8\)/\1/' /etc/locale.gen
sudo chroot "$CHROOT" etckeeper commit "Enable ja_JP.UTF-8"
sudo chroot "$CHROOT" etckeeper vcs gc
sudo chroot "$CHROOT" apt-get clean
```

- `cloud-init`で`/etc/locale.gen`の変更はしてくれないようなので変更しておく
- 他にもディスクイメージを小さくできる処理があればしておく

# ディスクイメージファイル作成

```
sudo apt-get install -y libguestfs-tools
sudo virt-make-fs --partition=gpt --type=ext4 \
 --size=10G "$CHROOT"/ "$HOME/riscv64/rootfs.img"
sudo chown "$USER" "$HOME/riscv64/rootfs.img"
```

- イメージ作成には時間がかかるのでゆっくり待つ
- qemu の実行ユーザーで読み書きできるようにしておく

# 実行に必要なものをインストール

```
sudo apt-get install -y qemu-system-misc
sudo apt-get install -y opensbi u-boot-qemu
sudo apt-get install -y genisoimage
```

- 実行に必要なものをインストール
- genisoimage は cloud-init の NoCloud データソース用

# cloud-init用データ作成

```
#cloud-config

ssh_pwauth: true # sshにパスワード認証で入れるようにする
chpasswd:
  list:
  #- root:debian # rootユーザーも直接使うならパスワードを設定する
  - debian:debian # debianユーザーはdefaultsで作成されるのでパスワード設定をしておく
  expire: false

manage_etc_hosts: true # sudoに必要

timezone: Asia/Tokyo
locale: ja_JP.UTF-8
package_upgrade: true
#packages:
#- qemu-guest-agent # riscv64 には未対応

mounts: # qemuの共有設定
- [ 'hostshare', '/mnt/hostshare', '9p', 'defaults,nofail,_netdev', '0', '2' ]

runcmd:
- [ mkdir, '-p', /mnt/hostshare ]
- [ mount, '-a' ]
- [ 'etckeeper', 'commit', 'cloud-init' ]
```

# cloud-init用ISOファイル作成

```
cd "$HOME/riscv64"
cat >meta-data <<EOF
instance-id: iid-local01
local-hostname: debian-riscv64
EOF
genisoimage -output seed.iso -volid cidata -joliet -rock meta-data user-data
```

- instance-id が変わるとopensshのホスト鍵の再作成などの処理が実行されるので適当な固定値に

# 起動

```
exec qemu-system-riscv64 -nographic -machine virt -m 1.9G \
 -bios /usr/lib/riscv64-linux-gnu/opensbi/generic/fw_jump.elf \
 -kernel /usr/lib/u-boot/qemu-riscv64_smode/uboot.elf \
 -object rng-random,filename=/dev/urandom,id=rng0 -device virtio-rng-device,rng=rng0 \
 -append "console=ttyS0 rw root=/dev/vda1" \
 -device virtio-blk-device,drive=hd0 -drive file=rootfs.img,format=raw,id=hd0 \
 -drive file=seed.iso,if=virtio \
 -device virtio-net-device,netdev=usernet -netdev user,id=usernet,hostfwd=tcp::22222-:22 \
 -smp cpus=4,sockets=1,cores=4,threads=1 \
 -virtfs local,path=$HOME/share,mount_tag=hostshare,security_model=mapped-xattr \
 -name debian-riscv64
```

- cloud-init用のseed.isoや共有のvirtfsなどは不要なら省略可能

# 接続

```
ssh -p 22222 debian@localhost
```

- 起動して `cloud-init` での設定が終わるまでしばらく待ってから接続


# 共有ディレクトリ設定

- ホストとゲストでファイルのやりとりのために `$HOME/share` を共有ディレクトリに設定
- ホスト側でパーミッションを 1777 に
- `security_model` は <https://wiki.qemu.org/Documentation/9psetup> の推奨設定
- ゲストで `sudo mount -t 9p hostshare /mnt/hostshare` のようにマウント
- 早すぎるとマウント失敗するので fstab に `_netdev` で遅めにマウント

# poweroff

```
[  169.345939] systemd-shutdown[1]: Powering off.
[  169.348326] reboot: Power down
```

- なぜか `poweroff` をしても上の行までで `qemu-system-riscv64` が終了せずに止まってしまう
- `C-a x` で終了
- ホスト側が amd64 の Ubuntu 20.04 だと問題なし

# 困りごと

```
debian@debian-riscv64:~$ echo 'int main(){return 0;}' | gcc -xc -
/usr/bin/ld: warning: /usr/lib/gcc/riscv64-linux-gnu/11/crti.o: mis-matched ISA version 2.0 for 'i' extension, the output version is 2.1
/usr/bin/ld: warning: /usr/lib/gcc/riscv64-linux-gnu/11/crti.o: mis-matched ISA version 2.0 for 'a' extension, the output version is 2.1
/usr/bin/ld: warning: /usr/lib/gcc/riscv64-linux-gnu/11/crti.o: mis-matched ISA version 2.0 for 'f' extension, the output version is 2.2
/usr/bin/ld: warning: /usr/lib/gcc/riscv64-linux-gnu/11/crti.o: mis-matched ISA version 2.0 for 'd' extension, the output version is 2.2
/usr/bin/ld: warning: /usr/lib/gcc/riscv64-linux-gnu/11/crtbeginS.o: mis-matched ISA version 2.0 for 'i' extension, the output version is 2.1
/usr/bin/ld: warning: /usr/lib/gcc/riscv64-linux-gnu/11/crtbeginS.o: mis-matched ISA version 2.0 for 'a' extension, the output version is 2.1
/usr/bin/ld: warning: /usr/lib/gcc/riscv64-linux-gnu/11/crtbeginS.o: mis-matched ISA version 2.0 for 'f' extension, the output version is 2.2
/usr/bin/ld: warning: /usr/lib/gcc/riscv64-linux-gnu/11/crtbeginS.o: mis-matched ISA version 2.0 for 'd' extension, the output version is 2.2
/usr/bin/ld: warning: /usr/lib/gcc/riscv64-linux-gnu/11/crtendS.o: mis-matched ISA version 2.0 for 'i' extension, the output version is 2.1
/usr/bin/ld: warning: /usr/lib/gcc/riscv64-linux-gnu/11/crtendS.o: mis-matched ISA version 2.0 for 'a' extension, the output version is 2.1
/usr/bin/ld: warning: /usr/lib/gcc/riscv64-linux-gnu/11/crtendS.o: mis-matched ISA version 2.0 for 'f' extension, the output version is 2.2
/usr/bin/ld: warning: /usr/lib/gcc/riscv64-linux-gnu/11/crtendS.o: mis-matched ISA version 2.0 for 'd' extension, the output version is 2.2
/usr/bin/ld: warning: /usr/lib/gcc/riscv64-linux-gnu/11/crtn.o: mis-matched ISA version 2.0 for 'i' extension, the output version is 2.1
/usr/bin/ld: warning: /usr/lib/gcc/riscv64-linux-gnu/11/crtn.o: mis-matched ISA version 2.0 for 'a' extension, the output version is 2.1
/usr/bin/ld: warning: /usr/lib/gcc/riscv64-linux-gnu/11/crtn.o: mis-matched ISA version 2.0 for 'f' extension, the output version is 2.2
/usr/bin/ld: warning: /usr/lib/gcc/riscv64-linux-gnu/11/crtn.o: mis-matched ISA version 2.0 for 'd' extension, the output version is 2.2
debian@debian-riscv64:~$
```

- なぜかリンクで警告がでる
- (`/usr/bin/ld: warning: /usr/lib/gcc/riscv64-linux-gnu/11/crti.o: mis-matched ISA version 2.0 for 'i' extension, the output version is 2.1` など)

# まとめ

- qemu で riscv64 の Debian 環境が作成できた
- cloud-init での初期設定もできた
- qemuのvirtfsの共有も使えた
- gccで警告がでる (普通に実行するだけで出るのでそのうち直るはず?)
