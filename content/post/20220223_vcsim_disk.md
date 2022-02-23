+++
title = "vcsimの小ネタ: disk作成"
tags = ["vsphere", "vcsim"]
date = "2022-02-23"
+++

# はじめに

[vcsim](https://github.com/vmware/govmomi/tree/master/vcsim) はシングルバイナリのvCenterシミュレーターで、vCenterを操作するシステム、ツールのテストをローカル環境でも手軽に行えます。
vCenterを操作できるCLI [govc](https://github.com/vmware/govmomi/tree/master/govc) の練習にももってこいです。

シミュレーターなので当然実際のvSphereが動いているわけではないのですが、高い再現度で動作する機能も多くあります。

今回はその中のディスク作成について紹介します。

# バージョン

```bash
$ vcsim version
Build Version: 0.27.4
Build Commit: 285e80cd
Build Date: 2022-02-11T04:56:04Z

$ govc version
govc 0.27.4
```

# まずはvcsim起動

デフォルトでは以下のURLで起動します。

```bash
$ vcsim
export GOVC_URL=https://user:pass@127.0.0.1:8989/sdk GOVC_SIM_PID=191563
```

- `GOVC_URL=https://user:pass@127.0.0.1:8989/sdk`
- `GOVC_INSECURE=true`

を環境変数に設定すれば、govcからvcsimにリクエストできます。

```bash
$ govc about
FullName:     VMware vCenter Server 6.5.0 build-5973321 (govmomi simulator)
Name:         VMware vCenter Server
Vendor:       VMware, Inc.
Version:      6.5.0
Build:        5973321
OS type:      linux-amd64
API type:     VirtualCenter
API version:  6.5
Product ID:   vpx
UUID:         dbed6e0c-bd88-4ef6-b594-21283e1c677f
```

# データストアとディスクを確認

vcsimでははじめからホストやvm等が作成されており（個数はオプションで指定可能）、vmにもdiskが1つささった状態になっています。

```bash
$ govc find .
/
/DC0
/DC0/vm
/DC0/vm/DC0_H0_VM0
/DC0/vm/DC0_H0_VM1
/DC0/vm/DC0_C0_RP0_VM0
/DC0/vm/DC0_C0_RP0_VM1
/DC0/host
/DC0/host/DC0_H0
/DC0/host/DC0_H0/DC0_H0
/DC0/host/DC0_H0/Resources
/DC0/host/DC0_C0
/DC0/host/DC0_C0/DC0_C0_H0
/DC0/host/DC0_C0/DC0_C0_H1
/DC0/host/DC0_C0/DC0_C0_H2
/DC0/host/DC0_C0/Resources
/DC0/datastore
/DC0/datastore/LocalDS_0
/DC0/network
/DC0/network/VM Network
/DC0/network/DVS0
/DC0/network/DVS0-DVUplinks-9
/DC0/network/DC0_DVPG0
```

```bash
$ govc device.info -vm DC0_H0_VM0 disk-*
Name:           disk-202-0
  Type:         VirtualDisk
  Label:        disk-202-0
  Summary:      10,485,760 KB
  Key:          255511522
  Controller:   pvscsi-202
  Unit number:  0
  File:         [LocalDS_0] DC0_H0_VM0/disk1.vmdk
```

ディスクはこちらのデータストア `LocalDS_0` に入っています。

```bash
$ govc datastore.info
Name:        LocalDS_0
  Path:      /DC0/datastore/LocalDS_0
  Type:      OTHER
  URL:       /tmp/govcsim-DC0-LocalDS_0-312572150
  Capacity:  10240.0 GB
  Free:      10200.0 GB
```

# vm関連のファイル

実は、ディスク等vm関連のファイルは、上記データストアのURLにちゃんと作成されています。

```bash
# データストアのURLはディレクトリパスに対応している
$ ls -l /tmp/govcsim-DC0-LocalDS_0-312572150
合計 16
drwx------ 2 syuparn syuparn 4096  2月 23 10:20 DC0_C0_RP0_VM0
drwx------ 2 syuparn syuparn 4096  2月 23 10:20 DC0_C0_RP0_VM1
drwx------ 2 syuparn syuparn 4096  2月 23 10:20 DC0_H0_VM0
drwx------ 2 syuparn syuparn 4096  2月 23 10:20 DC0_H0_VM1

$ ls -l /tmp/govcsim-DC0-LocalDS_0-312572150/DC0_H0_VM0
合計 4
-rw-r--r-- 1 syuparn syuparn   0  2月 23 10:20 DC0_H0_VM0.nvram
-rw-r--r-- 1 syuparn syuparn   0  2月 23 10:20 DC0_H0_VM0.vmx
-rw-r--r-- 1 syuparn syuparn   0  2月 23 10:20 disk1-flat.vmdk
-rw-r--r-- 1 syuparn syuparn   0  2月 23 10:20 disk1.vmdk
-rw-r--r-- 1 syuparn syuparn 118  2月 23 10:20 vmware.log
```

（中身こそ空ですが）vmxファイルやvmdkファイルが作成されています。ログファイルには、vcsim起動準備時の動作ログも記載されています。

```bash
$ cat /tmp/govcsim-DC0-LocalDS_0-312572150/DC0_H0_VM0/vmware.log
vmx 2022/02/23 10:20:30 created
vmx 2022/02/23 10:20:30 running power task: requesting poweredOn, existing poweredOff
```

逆に、この仕様のために存在しないディレクトリをデータストアパスに指定することはできません
（以前ハマりました）。

```bash
# ok
$ ls myds
$ govc datastore.create -type local -path ~/myds DC0_H0
# ng
$ govc datastore.create -type local -path ~/notfoundds DC0_H0
govc: ServerFaultCode: stat /home/syuparn/notfoundds: no such file or directory
```

また、ブロックストレージを用いるVMFSデータストアもファイル格納方式を再現できないため非対応です。

```bash
$ govc datastore.create -type vmfs -path ~/myds DC0_H0
govc: ServerFaultCode: HostDatastoreSystem:hostdatastoresystem-15 does not implement: QueryAvailableDisksForVmfs
```

参考： [create vmfs datastore using govc · Issue #1511 · vmware/govmomi](https://github.com/vmware/govmomi/issues/1511#issuecomment-506398383)

# おわりに（次回予告？）

vcsim小ネタはまだいくつかストックがあるので、シリーズ化して記事にしたいと思います。
