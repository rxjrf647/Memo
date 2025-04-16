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
