---
title: "ラズパイ上のProxmoxでcloud-initしたUbuntuの画面が表示されなかった"
emoji: "🍓"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [raspberrypi, proxmox, ubuntu]
published: true
---
# はじめに
今回、自宅Proxmoxクラスタに3台入っているラズパイにKubernetesHA構成では鉄板のコントロールプレーンにして運用しようとしていました。その際にcloud-initを使用してベースとなるUbuntuを構成したのですが、そこで色々躓いたのでそれらをご紹介します。

# 今回の環境
### Cloudimage
> https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-arm64.img
### # qm config
```bash
root@raspberrypi-proxmox-02:~# qm config 1002
boot: c
bootdisk: scsi0
cicustom: user=nfs:snippets/k8s-cp-2-user.yaml,network=nfs:snippets/k8s-cp-2-network.yaml
cores: 2
machine: virt
memory: 8192
meta: creation-qemu=9.0.2,ctime=1743041832
name: k8s-cp-2
net0: virtio=BC:24:11:27:3D:48,bridge=vmbr0
scsi0: vm-storage:vm-1002-disk-1,size=30G
scsi1: vm-storage:vm-1002-cloudinit,media=cdrom,size=4M
scsihw: virtio-scsi-pci
serial0: socket
smbios1: uuid=b78a6052-f0b8-4197-be28-4bec30303add
vga: serial0
vmgenid: 135059de-d36d-49ba-a98c-db31446c8aee
```

# 本題
## 画面が表示されない
今回1番詰まったのはcloud-initで立ち上がったubuntuにアクセスしようとしたら「starting serial terminal on interface serial0」から微動だにしない！！Enterを押しても無反応、と言う現象。
![黒い画面](/images/starting-serial-terminal-on-interface-serial0.png)
しかし、CPUは20%~30%ほどあるので何かしら起動はしてそう。
![ラズパイのCPU使用率](/images/raspberrypi-cpu-usage-proxmox.png)
## 原因
### 1. OVMF(EFI)を有効にしていなかった
これを有効にしていないとUbuntuが正常に起動しなかったっぽい。
なので有効にするために以下のコマンドを実行。
> \# qm set <vmid> --bios ovmf
そうすると、qm configにbios: ovmfが追加されます！
```bash diff
root@raspberrypi-proxmox-02:~# qm config 1002
+ bios: ovmf
boot: c
bootdisk: scsi0
cicustom: user=nfs:snippets/k8s-cp-2-user.yaml,network=nfs:snippets/k8s-cp-2-network.yaml
cores: 2
machine: virt
memory: 8192
meta: creation-qemu=9.0.2,ctime=1743041832
name: k8s-cp-2
net0: virtio=BC:24:11:27:3D:48,bridge=vmbr0
scsi0: vm-storage:vm-1002-disk-1,size=30G
scsi1: vm-storage:vm-1002-cloudinit,media=cdrom,size=4M
scsihw: virtio-scsi-pci
serial0: socket
smbios1: uuid=b78a6052-f0b8-4197-be28-4bec30303add
vga: serial0
vmgenid: 135059de-d36d-49ba-a98c-db31446c8aee
```

### 2. ノードにpve-edk2-firmwareをインストールしていなかった
`pve-edk2-firmware`にはARM用UEFIなどが入っており、Proxmox上でARM64系を動かすためには必要らしい。なので以下のコマンドでProxmoxに`pve-edk2-firmware`をインストールできます！
> \# apt install pve-edk2-firmware-aarch64 -y

## 結果
これらを設定し終わって、VMを再起動すると.....

真っ黒で無口だったコンソールから大量のカーネルからのメッセージが！！！
そして無事にcloud-initで起動したubuntuにアクセスできましたとさ。
![ログイン画面](/images/raspberrypi-login-ubuntu-proxmox.png)

# まとめ
小一時間程度ここで詰まったので、もし同じ状況に陥った方の助けになれば幸いです！
もし間違っている点などありましたら是非お知らせください！