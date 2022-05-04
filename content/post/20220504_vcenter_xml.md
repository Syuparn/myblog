+++
title = "vCenterの小ネタ: APIリクエストのXMLサイズ上限"
tags = ["vsphere", "govc"]
date = "2022-05-04"
+++

# TL; DR

[vSphere Web Services API](https://developer.vmware.com/apis/1192/vsphere)のリクエストにはサイズ上限があり、超えるとエラーが返る

- XMLの要素数: `500,000` (ネストされた分含む)
    - エラー: `ServerFaultCode: XML document element count exceeds configured maximum 500000`
- リクエストボディの長さ `20,000,000B` (`20MB`)
    - エラー: `ServerFaultCode: Length of HTTP request body exceeds configured maximum; 20000462 > 20000000` 

# 調査方法

正直記事で伝えたいことは↑で終わりですが、一応調べた方法も記載します。

## 注意！

vCenterに負荷がかかります！**本番環境で実行しないように！**（本記事は自宅の検証用vCenterで試しました）

## 要素数

XMLの要素数が(ネストされた中身も含めて)50万を超えるとエラーが発生しました。

実験には `RetrieveProperties` を使います。`objectSet` でManaged Objectを複数指定できるので、増やしてみてどこでエラーになるか試します。

```bash
# RetrieveProperties
$ govc vm.info myvm
Name:           myvm
  Path:         /DC0/vm/myvm
  UUID:         422100bc-69af-3a08-ff2d-dee92685cf2a
  Guest name:   VMware Photon OS (64-bit)
  Memory:       256MB
  CPU:          1 vCPU(s)
  Power state:  poweredOn
  Boot time:    2022-03-27 05:30:40 +0000 UTC
  IP address:
  Host:         192.168.1.200
# リクエストボディを確認 (詳細は以前の記事参照)
$ govc vm.info -trace myvm > /dev/null
...
<?xml version="1.0" encoding="UTF-8"?>
<Envelope xmlns="http://schemas.xmlsoap.org/soap/envelope/"><Body><RetrieveProperties xmlns="urn:vim25"><_this type="PropertyCollector">propertyCollector</_this><specSet><propSet><type>VirtualMachine</type><pathSet>summary</pathSet><pathSet>guest.ipAddress</pathSet></propSet><objectSet><obj type="VirtualMachine">vm-15</obj><skip>false</skip></objectSet></specSet></RetrieveProperties></Body></Envelope>
...

# vmを2つ指定
# 重複していても objectSet に2つ分指定される
$ govc vm.info -trace myvm myvm > /dev/null
...
<?xml version="1.0" encoding="UTF-8"?>
<Envelope xmlns="http://schemas.xmlsoap.org/soap/envelope/"><Body><RetrieveProperties xmlns="urn:vim25"><_this type="PropertyCollector">propertyCollector</_this><specSet><propSet><type>VirtualMachine</type><pathSet>summary</pathSet><pathSet>guest.ipAddress</pathSet></propSet><objectSet><obj type="VirtualMachine">vm-15</obj><skip>false</skip></objectSet><objectSet><obj type="VirtualMachine">vm-15</obj><skip>false</skip></objectSet></specSet></RetrieveProperties></Body></Envelope>
...
```

このまま増やして行きたかったのですが、`govc`だと引数が10000個を超えたあたりからレスポンスが不安定になり検証できませんでした。

```bash
$ yes myvm | head -n 10000 | xargs -n 10000 govc vm.info > /dev/null
```

仕方なく方針を変え、[govcの実装](https://github.com/vmware/govmomi/blob/master/govc/vm/info.go#L101)を参考にクライアントを実装しました。

```go
package main

import (
	"context"
	"fmt"
	"net/url"

	"github.com/samber/lo"
	"github.com/vmware/govmomi"
	"github.com/vmware/govmomi/property"
	"github.com/vmware/govmomi/vim25/mo"
	"github.com/vmware/govmomi/vim25/types"
)

func main() {
	ctx := context.TODO()
	url, _ := url.Parse(vCenterURL)

    // MOIDをn個作成することで、objectSetがn個指定される
	refs := lo.Times(1000000, func(int) types.ManagedObjectReference {
		return types.ManagedObjectReference{
			Type:  "VirtualMachine",
			Value: "vm-15",
		}
	})
	var vms []mo.VirtualMachine
	props := []string{"summary", "guest.ipAddress"}

	c, err := govmomi.NewClient(ctx, url, true)
	if err != nil {
		fmt.Printf("%v\n", err)
		return
	}

	pc := property.DefaultCollector(c.Client)
	if err := pc.Retrieve(ctx, refs, props, &vms); err != nil {
		fmt.Printf("%v\n", err)
		return
	}

	fmt.Printf("succeeded\n")
}
```

結果、objectSetを `166664` 個指定したときにエラーが発生しました。

```bash
$ go run ./main.go
ServerFaultCode: XML document element count exceeds configured maximum 500000
```

先ほど `govc` で確認したようにXMLの要素数（開始タグの合計）はobjectSetが `n` 個のとき `3n + 9` 個あったので、ネストされたものも含めた要素数と考えると計算が合います。

## リクエストボディ(XML)の長さ

つづいてXMLの長さです。要素数が上限に達していなくても、`20MB` を超えるとエラーになりました。

実験には同じく `RetrieveProperties` を使い、ダミーのpathSetに長い文字列を入れ検証します。

```go
func length() {
	ctx := context.TODO()
	url, _ := url.Parse(vCenterURL)

	refs := lo.Times(1, func(int) types.ManagedObjectReference {
		return types.ManagedObjectReference{
			Type:  "VirtualMachine",
			Value: "vm-15",
		}
	})
	var vms []mo.VirtualMachine
    // ダミーのpathSetで文字数を稼ぐ
	props := []string{"summary", "guest.ipAddress", strings.Repeat("a", 20000000)}

	c, err := govmomi.NewClient(ctx, url, true)
	if err != nil {
		fmt.Printf("%v\n", err)
		return
	}

	pc := property.DefaultCollector(c.Client)
	if err := pc.Retrieve(ctx, refs, props, &vms); err != nil {
		fmt.Printf("%v\n", err)
		return
	}

	fmt.Printf("succeeded\n")
}
```

`20000000` 文字(=20MB)でエラーが出ました。

```bash
$ go run ./main.go
ServerFaultCode: Length of HTTP request body exceeds configured maximum; 20000462 > 20000000
```

# おわりに

実運用でお目にかかることはめったに無いと思いますが、ライブラリによってはハマることもあるようなので([issue例](https://github.com/influxdata/telegraf/issues/5041))エラーメッセージを覚えておいて損はないでしょう。
