+++
title = "vcsimの小ネタ: vCenterをコピーして再現"
tags = ["vsphere", "vcsim"]
date = "2022-02-24"
+++

# はじめに

前回に引き続き [vcsim](https://github.com/vmware/govmomi/tree/master/vcsim) の小ネタです。

vcsimはデフォルトでもホスト、分散スイッチ、VM等を作ってくれますが、「運用しているvCenterと全く同じ構成で検証したい」という場面もあると思います。今回は、vCenterの構成を記録してvcsim上で再現する方法を紹介します。

# バージョン

```bash
$ vcsim version
Build Version: 0.27.4
Build Commit: 285e80cd
Build Date: 2022-02-11T04:56:04Z

$ govc version
govc 0.27.4
```

# 環境の再現

`govc object.save` で記録したリソースをvcsim起動時に読み込むことで再現できます。

参考 [vcsim features · vmware/govmomi Wiki](https://github.com/vmware/govmomi/wiki/vcsim-features#record-and-playback)


## vCenterリソースの記録

まずは実際のvCenterのリソースを記録します。

```bash
$ govc about
FullName:     VMware vCenter Server 7.0.3 build-18700403
Name:         VMware vCenter Server
Vendor:       VMware, Inc.
Version:      7.0.3
Build:        18700403
OS type:      linux-x64
API type:     VirtualCenter
API version:  7.0.3.0
Product ID:   vpx
UUID:         dbbe889e-ee2f-402b-95ba-aef9308c696f

# 中身確認
$ govc find .
/
/DC0
/DC0/vm
/DC0/host
/DC0/datastore
/DC0/network
/DC0/network/VM Network
/DC0/network/DVS0-DVUplinks-26
/DC0/network/DC0_DVPG0
/DC0/network/DVS0
/DC0/datastore/datastore1
/DC0/host/192.168.1.200
/DC0/host/DC0_C0
/DC0/host/DC0_C0/Resources
/DC0/host/192.168.1.200/Resources
/DC0/host/192.168.1.200/192.168.1.200
/DC0/vm/vCLS
/DC0/vm/myvm
/DC0/vm/VMware vCenter Server

# 記録
$ govc object.save -d my_vcenter
Saved 71 total objects to "my_vcenter", including:
DistributedVirtualPortgroup: 2
Folder: 6
ResourcePool: 2
VirtualMachine: 2
```

各リソースの情報がxml形式で保存されています。

```bash
$ ls my_vcenter
0000-ServiceInstance-ServiceInstance.xml
0001-AlarmManager-AlarmManager.xml
0002-AuthorizationManager-AuthorizationManager.xml
0003-ClusterProfileManager-ClusterProfileManager.xml
...
```

## vcsimで読み込み

先ほど作ったディレクトリを、vcsim起動時に読み込ませます。

```bash
$ vcsim -load my_vcenter
```

実際のvCenterと同じリソースが作成されていることが確認できます。

```bash
$ govc about -u https://user:pass@127.0.0.1:8989/sdk
FullName:     VMware vCenter Server 7.0.3 build-18700403
Name:         VMware vCenter Server
Vendor:       VMware, Inc.
Version:      7.0.3
Build:        18700403
OS type:      linux-x64
API type:     VirtualCenter
API version:  7.0.3.0
Product ID:   vpx
UUID:         dbbe889e-ee2f-402b-95ba-aef9308c696f

$  govc find -u https://user:pass@127.0.0.1:8989/sdk
/
/DC0
/DC0/vm
/DC0/vm/myvm
/DC0/vm/VMware vCenter Server
/DC0/vm/vCLS
/DC0/host
/DC0/host/192.168.1.200
/DC0/host/192.168.1.200/192.168.1.200
/DC0/host/192.168.1.200/Resources
/DC0/host/DC0_C0
/DC0/host/DC0_C0/Resources
/DC0/datastore
/DC0/datastore/datastore1
/DC0/network
/DC0/network/VM Network
/DC0/network/DVS0-DVUplinks-26
/DC0/network/DC0_DVPG0
/DC0/network/DVS0
```

見たところ記録されるのはmoidを持つリソースだけのようなので、タグ等は再現できませんでした。

```bash
# vCenter
$ govc tags.ls
my_tag  my_category

# vcsim
$ govc tags.ls -u https://user:pass@127.0.0.1:8989/sdk
```
