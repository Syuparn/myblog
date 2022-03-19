+++
title = "govcの小ネタ: ポートグループの各ポートを確認する"
tags = ["vsphere", "govc"]
date = "2022-03-19"
+++

vcsimに引き続き、[govc](https://github.com/vmware/govmomi/tree/master/govc)の小ネタ集やります。
個人的にはまったコマンド、便利なコマンドをいくつか紹介していきたいと思います。

## TL; DR

```bash
# 分散ポートグループのポート一覧

# アクティブなポート
$ govc dvs.portgroup.info -active -connected -pg ${portgroup} ${switch}
PortgroupKey: dvportgroup-28
DvsUuid:      50 21 44 2c 15 ff a7 d2-28 b0 2a c5 2c 26 45 5d
VlanId:       0
PortKey:      0

# アクティブでない（でも接続はされている）ポート
$ govc dvs.portgroup.info -connected -pg ${portgroup} ${switch}
# 空いているポート
$ govc dvs.portgroup.info -pg ${portgroup} ${switch}
# アクティブなアップリンクポート
$ govc dvs.portgroup.info -active -connected -uplinkPort -pg ${portgroup} ${switch}
```

# はじめに

分散ポートグループのポートを調べようとしたときにはまったのでメモ。

分散スイッチのポートグループの各ポートは `govc dvs.portgroup.info -pg ${portgroup} ${switch}` の形式で取得可能です。ポートグループを指定しなければ、分散スイッチの全ポートグループが返ってきます。

が、**指定した条件をすべて満たすものしか表示されません！**

- 指定できる条件
  - 接続しているか (true/false)
  - アクティブかどうか (true/false)
  - アップリンクポートかどうか (true/false)

デフォルトでは「接続していない」「アクティブでない」「アップリンクポートでない」ポートだけが取得されます。
言葉でいってもわけわかめなので、実際にvSphere Clientの表示と見比べながら紹介します。

# 実際に試してみる

分散スイッチ `DVS0` のポートグループ `DC0_DVPG0` は以下のポートを持っています。

{{<figure src="/images/20220319/pg.png">}}

ポート0はホストのvmkernelアダプタ、ポート1はvm(パワーオフ)のNICに接続しています。

この状態でgovcを使ってみます。オプションなしだと、デフォルトの「接続していない」「アクティブでない」「アップリンクポートでない」ポートが取得されます。空いているポート2~7が取得できました。

```bash
$ govc dvs.portgroup.info -pg DC0_DVPG0 DVS0
PortgroupKey: dvportgroup-28
DvsUuid:      50 21 44 2c 15 ff a7 d2-28 b0 2a c5 2c 26 45 5d
VlanId:       0
PortKey:      2

PortgroupKey: dvportgroup-28
DvsUuid:      50 21 44 2c 15 ff a7 d2-28 b0 2a c5 2c 26 45 5d
VlanId:       0
PortKey:      3

PortgroupKey: dvportgroup-28
DvsUuid:      50 21 44 2c 15 ff a7 d2-28 b0 2a c5 2c 26 45 5d
VlanId:       0
PortKey:      4

PortgroupKey: dvportgroup-28
DvsUuid:      50 21 44 2c 15 ff a7 d2-28 b0 2a c5 2c 26 45 5d
VlanId:       0
PortKey:      5

PortgroupKey: dvportgroup-28
DvsUuid:      50 21 44 2c 15 ff a7 d2-28 b0 2a c5 2c 26 45 5d
VlanId:       0
PortKey:      6

PortgroupKey: dvportgroup-28
DvsUuid:      50 21 44 2c 15 ff a7 d2-28 b0 2a c5 2c 26 45 5d
VlanId:       0
PortKey:      7
```

`-connected` を指定すると、「接続している」「アクティブでない」「アップリンクポートでない」ポートが取得されます。パワーオフのVMに接続しているポート1だけが取得できました。

```bash
# 接続している、アクティブでない(デフォルト)、アップリンクポートでない（デフォルト）ポート
$ govc dvs.portgroup.info -connected -pg DC0_DVPG0 DVS0
PortgroupKey: dvportgroup-28
DvsUuid:      50 21 44 2c 15 ff a7 d2-28 b0 2a c5 2c 26 45 5d
VlanId:       0
PortKey:      1
```

`-connected` と `-active` を指定すると、「接続している」「アクティブな」「アップリンクポートでない」ポートが取得されます。こちらはホストに接続しているポート0が取得できました。

```bash
# 接続している、アクティブな、アップリンクポートでない（デフォルト）ポート
$ govc dvs.portgroup.info -active -connected DVS0
PortgroupKey: dvportgroup-28
DvsUuid:      50 21 44 2c 15 ff a7 d2-28 b0 2a c5 2c 26 45 5d
VlanId:       0
PortKey:      0
```

(`-active` だけ指定すると「接続していない」「アクティブな」「アップリンクポートでない」ポートが取れますが、そんなポートは（おそらく）ありえないので意味なし)

また、`-connected`、 `-active`、`-uplinkPort` をすべて指定すると、「接続している」「アクティブな」「アップリンクポート」が取得でき、今使っているアップリンクポートが確認できます。
（~~ちなみに自宅ラボではVMNICを標準スイッチに取られているのでアップリンクポートはすべて未接続です~~）

```bash
# 接続している、アクティブな、アップリンクポート
$ govc dvs.portgroup.info -uplinkPort -active -connected DVS0
```
