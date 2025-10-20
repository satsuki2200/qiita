---
title: i915が原因でRocky Linux 9.6が起動しない問題の対処法
tags:
  - i915
  - RockyLinux
private: false
updated_at: '2025-10-20T16:14:53+09:00'
id: b699afd5e8b82a06f842
organization_url_name: null
slide: false
ignorePublish: false
---

:::note warn
この問題は複数のLinuxディストリビューションで報告されていますが、他のLinuxディストリビューションでは以下の対処法を検証していません。
:::

:::note warn
当方環境ではCLIのみ復旧できたため、CLI環境のみ復旧の対象法記事になります。
GUI環境の復旧は現在試みている最中ですので、もし対処法をご存知の方がいらっしゃいましたら、教えていただけますと幸いです🙇
:::


# 発生までの流れ

Rocky Linux 9.4をインストール後、9.6へアップデート。
アップデート後に再起動を試みたところ、起動時に以下のエラーが発生して起動できない状態になった。
```
[    1.485049] i915 0000:00:02.0: [drm] *ERROR* Unexpected child device config size 40 (expected 39 for VBT version 258)
[    3.273843] i915 0000:00:02.0: [drm] *ERROR* GT1: Request submission for GSC load failed -ETIME
[    3.273845] i915 0000:00:02.0: [drm] *ERROR* GT1: Failed to load GSC firmware i915/mtl_gsc_1.bin
〜中略〜
[    9.426172] i915 0000:00:02.0: [drm] *ERROR* GT1: GUC: Engine reset failed on 5:6 (other0) because 0x00000000
```

## i915ドライバが原因でRocky Linux 9.6が起動しない問題の対処法

起動時の実行コマンドに`systemd.unit=multi-user.target`を追加する。

既存コマンド

```
load_video
set gfxpayload=keep
insmod gzio
linux ($root)/vmlinuz-5.14.0-570.49.1.el9_6.x86_64 root=/dev/mapper/r1-root\
 ro crashkernel=1G-4G:192M,4G-64G:256M,64G-:512M resume=/dev/mapper/r1-swap\
 rd.lvm.lv=r1/root rd.lvm.lv=r1/swap rhgb quiet  rd.driver.blacklist=nouvea\
 u
initrd ($root)/initramfs-5.14.0-570.49.1.el9_6.x86_64.img $tuned_initrd
```

追加後のコマンド

```
load_video
set gfxpayload=keep
insmod gzio
linux ($root)/vmlinuz-5.14.0-570.49.1.el9_6.x86_64 root=/dev/mapper/r1-root\
 ro crashkernel=1G-4G:192M,4G-64G:256M,64G-:512M resume=/dev/mapper/r1-swap\
 rd.lvm.lv=r1/root rd.lvm.lv=r1/swap rhgb quiet  rd.driver.blacklist=nouvea\
 u systemd.unit=multi-user.target
initrd ($root)/initramfs-5.14.0-570.49.1.el9_6.x86_64.img $tuned_initrd
```


## 参考資料
https://bbs.archlinux.org/viewtopic.php?id=270304
