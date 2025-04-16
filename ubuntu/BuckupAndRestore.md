# Linux バックアップとレストア

## バックアップ手順

1. フルバックアップ（レベル0）

```bash
sudo dump -0 -f /mnt/backup/full_backup.dump /dev/sdX1
```

- `-0` はフルバックアップ（レベル0）
- `-f /mnt/backup/full_backup.dump` でバックアップファイルの保存先を指定
- `/dev/sdX1` はバックアップ対象のディスクパーティション（確認が必要）

1. 増分バックアップ

   - 例えば、前回のバックアップとの差分だけを保存する場合：

```bash
sudo dump -1 -f /mnt/backup/incremental_backup.dump /dev/sdX1
```

- `-1` はレベル1の増分バックアップ（前回のバックアップとの差分）

- できたファイルをUSBなどに保存

```bash
sudo cp /mnt/backup/full_backup.dump /media/usb/
```

## 復元（restoreコマンド）

1. フルバックアップから復元

```bash
sudo restore -r -f /mnt/backup/full_backup.dump
```

- `-r` でリストアモードを指定

1. 増分バックアップの適用

- 例えば、増分バックアップも適用する場合：

```bash
sudo restore -r -f /mnt/backup/incremental_backup.dump
```

## 注意点

- バックアップ対象のパーティションを確認（`lsblk` や `df -h` を使用）
- USBや外部ストレージの容量を確保（特にフルバックアップ時）
- 復元前にデータをチェック（`restore` で誤って上書きしないように）

## リストア手順の流れ

1. 新規ディスクのパーティショニング

- `fdisk` や `parted` を使って以下のように構成：
  - `/efi` パーティション（FAT32, 例: `/dev/sdX1`）
  - `/` パーティション（ext4, 例: `/dev/sdX2`）

```bash
sudo parted /dev/sdX mklabel gpt
sudo parted /dev/sdX mkpart ESP fat32 1MiB 512MiB
sudo parted /dev/sdX set 1 boot on
sudo parted /dev/sdX mkpart primary ext4 512MiB 100%
   ```

1. ファイルシステムの作成

```bash
sudo mkfs.fat -F32 /dev/sdX1  # EFIパーティション
sudo mkfs.ext4 /dev/sdX2      # ルートパーティション
```

1. Dumpから `/` のリストア

```bash
sudo mount /dev/sdX2 /mnt
cd /mnt
sudo restore -r -f /path/to/full_backup.dump
```

1. EFIパーティションの設定

- `EFI` は `/boot/efi` にマウント：

```bash
sudo mkdir -p /mnt/boot/efi
sudo mount /dev/sdX1 /mnt/boot/efi
```

1. GRUBの再インストール

```bash
sudo mount --bind /dev /mnt/dev
sudo mount --bind /sys /mnt/sys
sudo mount --bind /proc /mnt/proc
sudo chroot /mnt
sudo grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ubuntu
sudo update-grub
exit
```

## レストア時の注意点

- バックアップしたときのファイルシステムと同じパーティションであることが必要
  - ダンプファイルからファイルシステムを読み取ることはできないのでメモっておく。
  - 以下のようにスクリプト化して、ファイルシステム情報といっしょに保存させる、など。

### バックアップ スクリプト

```bash
#!/bin/bash
BACKUP_DIR="/mnt/backup"
FS_INFO="$BACKUP_DIR/filesystem_info.txt"
DUMP_FILE="$BACKUP_DIR/full_backup.dump"

# ファイルシステム情報を取得して保存
lsblk -f > $FS_INFO

# Dumpでバックアップ
sudo dump -0 -f $DUMP_FILE /dev/sdX2
```

### レストア スクリプト

```bash
#!/bin/bash

# バックアップ時に取得したファイルシステム情報のファイル
FS_INFO="/mnt/backup/filesystem_info.txt"

# ターゲットディスク（適宜変更）
TARGET_DISK="/dev/sdX"

# パーティション情報を抽出
EFI_PART=$(grep "fat32" $FS_INFO | awk '{print $1}')
ROOT_PART=$(grep "ext4" $FS_INFO | awk '{print $1}') ## これ"ext4"で検索してはメモの意味がない、要検討

# パーティショニング（GPTを設定）
sudo parted $TARGET_DISK mklabel gpt

# EFIパーティション作成（512MB）
sudo parted $TARGET_DISK mkpart ESP fat32 1MiB 512MiB
sudo parted $TARGET_DISK set 1 boot on

# ルートパーティション作成（残りの容量）
sudo parted $TARGET_DISK mkpart primary ext4 512MiB 100%

# ファイルシステムの作成
sudo mkfs.fat -F32 ${TARGET_DISK}1
sudo mkfs.ext4 ${TARGET_DISK}2

# マウント＆リストア
sudo mount ${TARGET_DISK}2 /mnt
cd /mnt
sudo restore -r -f /mnt/backup/full_backup.dump

# EFIブートパーティション設定 （Legacy BIOSの場合は別途メモる）
sudo mkdir -p /mnt/boot/efi
sudo mount ${TARGET_DISK}1 /mnt/boot/efi

# GRUBの再インストール
sudo mount --bind /dev /mnt/dev
sudo mount --bind /sys /mnt/sys
sudo mount --bind /proc /mnt/proc
sudo chroot /mnt
sudo grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ubuntu
sudo update-grub
exit

echo "restore completed"
```

## ルートパーティションのファイルシステムのメモ方法

1. `findmnt` を活用

- `findmnt` でマウントされているファイルシステムをシンプルに表示できるので、ルートパーティションを確実に特定可能。

```bash
findmnt -n -o SOURCE / 
```

- これにより、現在のルート (`/`) のデバイス名 を取得。例えば `/dev/sdX2` など。

1. `blkid` と組み合わせてファイルシステムを抽出

- ルートパーティションを特定した上で、そのファイルシステムタイプを取得：

```bash
ROOT_DEV=$(findmnt -n -o SOURCE /)
FS_TYPE=$(blkid -o value -s TYPE $ROOT_DEV)
echo "ルートパーティション: $ROOT_DEV, ファイルシステム: $FS_TYPE"
```

1. スクリプトに組み込んでバックアップ

```bash
ROOT_DEV=$(findmnt -n -o SOURCE /)
FS_TYPE=$(blkid -o value -s TYPE $ROOT_DEV)

echo "$ROOT_DEV $FS_TYPE" > /mnt/backup/filesystem_info.txt

sudo dump -0 -f /mnt/backup/full_backup.dump $ROOT_DEV
```

- これで抽出が楽になる、と思う。

## UUIDの注意事項

- `/etc/fstab`にuuidを使ったマウント制御をしている場合している場合、
  - 新規Diskにリストアするとuuidが変わってしまうため、マウントしなくなる
  - grubも同じ

### 対応方法

- 記録しておいたUUIDを再利用する場合は、新しいUUIDを元のものに書き換える。

```bash
sudo tune2fs -U <元のUUID> /dev/sdX2
```

- 新しいUUIDを使う場合は、`fstab`を新しいUUIDに変更する

```bash
sudo nano /etc/fstab
```

- リブートする

```bash
sudo reboot
```
