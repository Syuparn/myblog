+++
title = "govcの小ネタ: 実行したAPIのリクエスト/レスポンスを確認する"
tags = ["vsphere", "govc"]
date = "2022-04-15"
+++

# TL; DR

- `-trace` オプションを付けると、vCenterへのHTTPリクエスト/レスポンスがすべて表示される！

# はじめに

govcを実行したとき、実際にはどんなAPIでvCenterへリクエストしているのでしょうか？

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

今回は、govc内部で叩いたAPIリクエスト/レスポンスを確認する方法を紹介します。

（~~[コード](https://github.com/vmware/govmomi/blob/master/govc/about/command.go#L75)読めば分かるやろというツッコミは無しで :innocent:~~）

# -trace オプション

**`-trace` オプションを付ける**と、APIのHTTPリクエスト、レスポンスを出力できます。

```
-trace=false                        Write SOAP/REST traffic to stderr
```

```bash
# stderrにリクエスト/レスポンスを出力
# stdoutは相変わらず表示されるので、 /dev/null に捨てると見やすい
$ govc about -trace > /dev/null
POST /sdk HTTP/1.1
Host: 127.0.0.1:8989
Content-Type: text/xml; charset="utf-8"
Soapaction: urn:vim25/7.0
User-Agent: govc/0.27.4

<?xml version="1.0" encoding="UTF-8"?>
<Envelope xmlns="http://schemas.xmlsoap.org/soap/envelope/"><Body><RetrieveProperties xmlns="urn:vim25"><_this type="PropertyCollector">propertyCollector</_this><specSet><propSet><type>SessionManager</type><pathSet>currentSession</pathSet></propSet><objectSet><obj type="SessionManager">SessionManager</obj><skip>false</skip></objectSet></specSet></RetrieveProperties></Body></Envelope>2022-04-15T11-14-20.011233266 - 0001:      1ms (*methods.RetrievePropertiesBody)
HTTP/1.1 200 OK
Content-Length: 1038
Content-Type: text/xml; charset=utf-8
Date: Fri, 15 Apr 2022 02:14:20 GMT

<?xml version="1.0" encoding="UTF-8"?>
<soapenv:Envelope xmlns:soapenc="http://schemas.xmlsoap.org/soap/encoding/" xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"><soapenv:Body><RetrievePropertiesResponse xmlns="urn:vim25"><returnval><obj type="SessionManager">SessionManager</obj><propSet><name>currentSession</name><val xmlns:XMLSchema-instance="http://www.w3.org/2001/XMLSchema-instance" XMLSchema-instance:type="UserSession"><key>0a8e5793-bc37-4ff3-8a34-916244ed78ad</key><userName>user</userName><fullName>user</fullName><loginTime>2022-04-15T02:06:55.124232819Z</loginTime><lastActiveTime>2022-04-15T11:14:20.010917828+09:00</lastActiveTime><locale>en_US</locale><messageLocale>en_US</messageLocale><extensionSession>false</extensionSession><ipAddress>127.0.0.1</ipAddress><userAgent>govc/0.27.4</userAgent><callCount>19</callCount></val></propSet></returnval></RetrievePropertiesResponse></soapenv:Body></soapenv:Envelope>
```

`SessionManager` に対して `RetrieveProperties` をしているようです。

`-trace` オプションはすべてのサブコマンドで使えます。

```bash
# 長いので省略
$ govc vm.info -trace DC0_H0_VM0 > /dev/null
...
HTTP/1.1 200 OK
Content-Length: 1726
Content-Type: text/xml; charset=utf-8
Date: Fri, 15 Apr 2022 02:26:24 GMT

<?xml version="1.0" encoding="UTF-8"?>
<soapenv:Envelope xmlns:soapenc="http://schemas.xmlsoap.org/soap/encoding/" xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"><soapenv:Body><RetrievePropertiesResponse xmlns="urn:vim25"><returnval><obj type="VirtualMachine">vm-54</obj><propSet><name>summary</name><val xmlns:XMLSchema-instance="http://www.w3.org/2001/XMLSchema-instance" XMLSchema-instance:type="VirtualMachineSummary"><vm type="VirtualMachine">vm-54</vm><runtime><host type="HostSystem">host-21</host><connectionState>connected</connectionState><powerState>poweredOn</powerState><toolsInstallerMounted>false</toolsInstallerMounted><bootTime>2022-04-15T11:06:15.031569698+09:00</bootTime><numMksConnections>0</numMksConnections></runtime><guest><guestId>otherGuest</guestId><toolsStatus>toolsNotInstalled</toolsStatus></guest><config><name>DC0_H0_VM0</name><template>false</template><vmPathName>[LocalDS_0] DC0_H0_VM0/DC0_H0_VM0.vmx</vmPathName><memorySizeMB>32</memorySizeMB><numCpu>1</numCpu><numEthernetCards>1</numEthernetCards><numVirtualDisks>1</numVirtualDisks><uuid>265104de-1472-547c-b873-6dc7883fb6cb</uuid><instanceUuid>b4689bed-97f0-5bcd-8a4c-07477cc8f06f</instanceUuid><guestId>otherGuest</guestId><guestFullName>otherGuest</guestFullName></config><storage><committed>0</committed><uncommitted>0</uncommitted><unshared>0</unshared><timestamp>2022-04-15T11:06:15.011905714+09:00</timestamp></storage><quickStats><guestHeartbeatStatus>gray</guestHeartbeatStatus></quickStats><overallStatus>green</overallStatus></val></propSet></returnval></RetrievePropertiesResponse></soapenv:Body></soapenv:Envelope>
...
```

長いので端折りましたが、全リクエスト、レスポンスが時系列で表示されます。

パラメータの意味は、[vSphere Web Services API のリファレンス](https://developer.vmware.com/apis/1192/vsphere) で確かめられます。

## パスワードは隠される

ログイン時のパスワードにはマスクをかけてくれます。**調査結果を共有するときにうっかりパスワードを貼り付けずに済む**ので安全ですね。

```bash
...
POST /sdk HTTP/1.1
Host: 192.168.1.201
Content-Type: text/xml; charset="utf-8"
Soapaction: urn:vim25/7.0
User-Agent: govc/0.27.4

<?xml version="1.0" encoding="UTF-8"?>
<Envelope xmlns="http://schemas.xmlsoap.org/soap/envelope/"><Body><Login xmlns="urn:vim25"><_this type="SessionManager">SessionManager</_this><userName>User@MYDOMAIN.LOCAL</userName><password>********</password><locale>en_US</locale></Login></Body></Envelope>2022-04-15T11-41-31.321479500 - 0002:     43ms (*methods.LoginBody)
```

## -verboseオプションと組み合わせるとGoの構造体を表示

`-verbose` オプションも `-trace` 同様リクエストとレスポンスを出力します。こちらは人が見やすい形式に整形してくれます。

```
  -verbose=false                      Write request/response data to stderr
```

```bash
$ govc vm.info -verbose DC0_H0_VM0 > /dev/null
RetrieveProperties(propertyCollector, []types.PropertyFilterSpec)...
...group-d1 name: Datacenters

RetrieveProperties(propertyCollector, []types.PropertyFilterSpec)...
...datacenter-2 name: DC0

RetrieveProperties(propertyCollector, []types.PropertyFilterSpec)...
...datacenter-2 name:            DC0
...datacenter-2 vmFolder:        Folder:group-3
...datacenter-2 hostFolder:      Folder:group-4
...datacenter-2 datastoreFolder: Folder:group-5
...datacenter-2 networkFolder:   Folder:group-6

RetrieveProperties(propertyCollector, []types.PropertyFilterSpec)...
...group-3      name:   vm
...group-3      parent: Datacenter:datacenter-2
...datacenter-2 name:   DC0
...datacenter-2 parent: Folder:group-d1
...group-d1     name:   Datacenters

RetrieveProperties(propertyCollector, []types.PropertyFilterSpec)...
...vm-54 name: DC0_H0_VM0
...vm-57 name: DC0_H0_VM1
...vm-60 name: DC0_C0_RP0_VM0
...vm-63 name: DC0_C0_RP0_VM1

RetrieveProperties(propertyCollector, []types.PropertyFilterSpec)...
...vm-54 summary: `govc object.collect -dump VirtualMachine:vm-54 summary`

RetrieveProperties(propertyCollector, []types.PropertyFilterSpec)...
...host-21 name: DC0_H0
```

さらに、 **`-verbose` と `-trace` を同時に指定すると、Goの構造体コードがそのまま出力されます**。~~これはワザップ~~

```bash
$ govc vm.info -verbose -trace DC0_H0_VM0 > /dev/null
types.RetrieveProperties{
    This:    types.ManagedObjectReference{Type:"PropertyCollector", Value:"propertyCollector"},
    SpecSet: []types.PropertyFilterSpec{
        {
            PropSet: []types.PropertySpec{
                {
                    Type:    "ManagedEntity",
                    All:     (*bool)(nil),
                    PathSet: []string{"name", "parent"},
                },
                {
                    Type:    "VirtualMachine",
                    All:     (*bool)(nil),
                    PathSet: []string{"parentVApp"},
                },
            },
            ObjectSet: []types.ObjectSpec{
                {
                    Obj:       types.ManagedObjectReference{Type:"Folder", Value:"group-d1"},
                    Skip:      types.NewBool(false),
                    SelectSet: []types.BaseSelectionSpec{
                        &types.TraversalSpec{
                            SelectionSpec: types.SelectionSpec{
                                Name: "traverseParent",
                            },
                            Type:      "ManagedEntity",
                            Path:      "parent",
                            Skip:      types.NewBool(false),
                            SelectSet: []types.BaseSelectionSpec{
                                &types.SelectionSpec{
                                    Name: "traverseParent",
                                },
                            },
                        },
                        &types.TraversalSpec{
                            SelectionSpec: types.SelectionSpec{},
                            Type:          "VirtualMachine",
                            Path:          "parentVApp",
                            Skip:          types.NewBool(false),
                            SelectSet:     []types.BaseSelectionSpec{
                                &types.SelectionSpec{
                                    Name: "traverseParent",
                                },
                            },
                        },
                    },
                },
            },
            ReportMissingObjectsInResults: (*bool)(nil),
        },
    },
}
# ... （長いので省略）
```

# おわりに

以上、govcでリクエスト/レスポンスを確認する方法でした。govcの動作確認だけでなく、vSphere操作ツールを開発する際にも参考にできそうです。
