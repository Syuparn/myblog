+++
title = "Helm Chartでマイクロサービスのマニフェストをお手軽管理"
tags = ["helm", "kubernetes", "dapr"]
date = "2022-02-19"
+++

**(この記事は、2021年にQiitaに上げた[記事](https://qiita.com/Syuparn/items/d79a3d945a18f6c34319)の再投稿です。内容が古くなっている可能性がありますのでご了承ください。)**

この記事は、[富士通クラウドテクノロジーズ Advent Calendar 2021](https://qiita.com/advent-calendar/2021/fjct) 5日目の記事です。
4日目は @o108minmin さんの 「[今年はまったレシピとその試行錯誤](https://o108minmin.hatenablog.com/entry/2021/12/04/000047)」 でした。同じレシピを繰り返し作って改良していくところに職人魂を感じました！

# TL;DR

- マイクロサービスのマニフェスト、deploymentが多くて管理が大変...
- ボイラープレートを減らすために、deploymentやserviceをまるごとテンプレート化！
- valuesを上書きできるのでchart内で環境差分の記述も不要！

[Syuparn/helm-chart-sample: Helm chart template sample of Dapr applications which can reduce boilerplates](https://github.com/Syuparn/helm-chart-sample)

# はじめに

アプリケーションのKubernetesマニフェストは、何もしないとコピペまみれになってしまいがちです。特に、マイクロサービスの場合

- アプリごとの差分
- デプロイ環境ごとの差分

のダブルパンチでマニフェストが増えていってしまいます。

マニフェストの行数が多いと修正やレビューが大変になり、設定ミスも起きやすくなります...

そこで、本記事ではボイラープレートを極力減らすHelm Chartを作ってみました。

（Helm Chartを使うのは初めてなので、「こう書いた方が綺麗になる！」等ありましたらコメントいただけるとありがたいです :innocent: ）

# バージョン

- Helm v3.7.1
- Dapr v1.5.0
- Skaffold v1.35.0

# つくったもの

https://github.com/Syuparn/helm-chart-sample

[Dapr](https://dapr.io/)の公式サンプル「[Distributed Calculator](https://github.com/dapr/quickstarts/tree/master/distributed-calculator)」のHelm Chartです。

Distributed Calculatorは4つのバックエンドサービス(加減乗除サービス)と1つのフロントエンドサービスからなるマイクロサービスです。

サービスの個数分deploymentを書くのはつらいので、**マニフェストをまるごとテンプレート化**しています。

(このアイデアはGrafanaのHelm Chartを参考にしています)

https://github.com/grafana/helm-charts/blob/main/charts/grafana/templates/_pod.tpl


```yaml:distributed-calculator/templates/_backend.tpl.yaml
{{/*
# NOTE: Values全体と対象アプリの情報量方が必要なので
# {{include "distributed-calculator.backend.deployment" (dict "Values" .Values "app" .Values.backends.hoge)}}
# のように呼び出す
*/}}
{{- define "distributed-calculator.backend.deployment" -}}
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "distributed-calculator.deploymentName" .app }}
  labels:
    app: {{ .app.name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .app.name }}
  template:
    metadata:
      labels:
        app: {{ .app.name }}
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: {{ include "distributed-calculator.deploymentName" .app | quote}}
        dapr.io/app-port: {{ .app.port | quote }}
        dapr.io/config: "appconfig"
        {{- with $.Values.podAnnotations }}
        {{- toYaml . | nindent 8 }}
        {{- end }}
    spec:
      containers:
        - name: {{ .app.name }}
          image: {{ include "distributed-calculator.deploymentImage" .app}}
          ports:
            - containerPort: {{ .app.port }}
          imagePullPolicy: {{ .Values.imagePullPolicy }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
{{- end }}
```

テンプレートさえできれば、各アプリの差分情報を流し込むだけでマニフェストの完成です。

```yaml:distributed-calculator/values.yaml
# アプリごとの情報（差分が生じる最低限の部分だけ記述）
backends:
  add:
    name: add
    image: dapriosamples/distributed-calculator-go
    tag: latest
    port: 6000
  subtract:
    name: subtract
    image: dapriosamples/distributed-calculator-csharp
    tag: latest
    port: 80
  multiply:
    name: multiply
    image: dapriosamples/distributed-calculator-python
    tag: latest
    port: 5000
  divide:
    name: divide
    image: dapriosamples/distributed-calculator-node
    tag: latest
    port: 4000
```

```yaml:distributed-calculator/templates/backends.yaml
# アプリ情報を入れるだけでマニフェスト完成
{{include "distributed-calculator.backend.deployment" (dict "Values" $.Values "app" $.Values.backends.add)}}
{{include "distributed-calculator.backend.deployment" (dict "Values" $.Values "app" $.Values.backends.subtract)}}
{{include "distributed-calculator.backend.deployment" (dict "Values" $.Values "app" $.Values.backends.multiply)}}
{{include "distributed-calculator.backend.deployment" (dict "Values" $.Values "app" $.Values.backends.divide)}}
```

「メタデータの名前とDaprのapp-idが対応しているか」や「コンテナのポートとDaprのポートが同じか」等、人手ではミスしやすい部分もテンプレートなら整合性を担保できます。

ちなみにテンプレートのファイル名 `_backend.tpl.yaml` は以下の理由で決めました。

- `_` はじまりのファイルはHelmのレンダリング対象にならない（[公式ページ](https://helm.sh/docs/chart_template_guide/named_templates/#partials-and-_-files)）
- `yaml` で終わるファイル名はVSCodeのYAMLシンタックスハイライトが効く

# chartのつくり方

chartの骨組みは、 `helm create` コマンドで自動生成されます。

```bash
$ helm create distributed-calculator
Creating distributed-calculator
```

[Helm | Getting Started](https://helm.sh/docs/chart_template_guide/getting_started/#a-starter-chart)

デプロイされるリソースのマニフェスト一覧は `helm template` で確認できます。テンプレートの修正→`helm template` で確認を繰り返して、[公式リポジトリのマニフェスト](https://github.com/dapr/quickstarts/tree/master/distributed-calculator/deploy)と同じ出力になるまで編集しました。

```bash
$ helm template ./distributed-calculator/
```

また、サンプルリポジトリに含まれていなかったtracerのマニフェストについては、kubectlのdry-runで生成しました。

tracer(Zipkin)のデプロイ手順

[How-To: Set up Zipkin for distributed tracing | Dapr Docs](https://docs.dapr.io/operations/monitoring/tracing/supported-tracing-backends/zipkin/#configure-kubernetes)

```bash
# ↑をdry-runしてマニフェストだけ出力
$ kubectl create deployment zipkin --image openzipkin/zipkin --dry-run=client -o yaml
$ kubectl expose deployment zipkin --type ClusterIP --port 9411 --dry-run=client -o yaml
```

# デプロイ

ローカルのKindクラスタにデプロイします。
注意点として、アプリのデプロイの前にdapr-systemやcomponent用ミドルウェア(今回はRedis)のデプロイが必要です。

```bash
# kindクラスター作成
$ kind create cluster

# 依存するリソースのデプロイ
# Dapr (https://docs.dapr.io/operations/hosting/kubernetes/kubernetes-deploy/#add-and-install-dapr-helm-chart 参照)
$ helm upgrade --install dapr dapr/dapr \
--version=1.5 \
--namespace dapr-system \
--create-namespace \
--wait

# Redis (https://github.com/bitnami/charts/tree/master/bitnami/redis 参照)
$ helm install redis bitnami/redis

# アプリ本体（今回作ったチャート）をデプロイ
$ helm install distributed-calculator ./distributed-calculator/
```

serviceにポートフォワードすれば、`localhost:8080` からフロントエンドの電卓アプリにアクセスできます。

```bash
$ kubectl port-forward svc/calculator-front-end 8080:80
```

{{<figure src="/images/20220219_helm_chart/calc.png">}}

Zipkinのtracerも動いているようです。

```bash
$ kubectl port-forward svc/zipkin 9411:9411
```

{{<figure src="/images/20220219_helm_chart/zipkin.png">}}


# テスト

手動でアプリを叩かなくても、`helm test` で想定通りデプロイされているかテストができます。

テストケースごとに専用のPodをデプロイし、コンテナが実行したコマンドの終了ステータスが`0`ならテスト成功です。

[Helm | チャートのテスト](https://helm.sh/ja/docs/topics/chart_tests/)

このチャートでは、curlでフロントエンドからレスポンスを受け取れるかどうかをテストしています。

```yaml:distributed-calculator/templates/tests/test-connection.yaml
# Test if frontend app "calculator-front-end" is working
apiVersion: v1
kind: Pod
metadata:
  name: "calculator-front-end-is-working"
  labels:
    {{- include "distributed-calculator.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": test
spec:
  containers:
    - name: curl
      # このイメージ色々なコマンドが入っていて便利
      image: yauritux/busybox-curl
      command: ['curl']
      args: ['calculator-front-end.default.svc.cluster.local']
  restartPolicy: Never
```

```bash
$ helm test distributed-calculator
NAME: distributed-calculator
LAST DEPLOYED: Tue Nov 23 16:53:44 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE:     calculator-front-end-is-working
Last Started:   Tue Nov 23 16:57:59 2021
Last Completed: Tue Nov 23 16:58:02 2021
Phase:          Succeeded
```

テスト用podはテストが終わると`Completed`になるので、リソースを無駄遣いすることはありません。

```bash
$ kubectl get pod calculator-front-end-is-working
NAME                              READY   STATUS      RESTARTS   AGE
calculator-front-end-is-working   0/1     Completed   0          92s
```

# Skaffoldでデプロイ

マニフェストはシンプルになりましたがデプロイ手順が煩雑なので、`skaffold run` の1コマンドでデプロイできるようにします。

```yaml:skaffold.yaml
apiVersion: skaffold/v2beta26
kind: Config
deploy:
  helm:
    releases:
      - name: dapr
        repo: https://dapr.github.io/helm-charts/
        remoteChart: dapr
        version: "1.5.0"
      - name: redis
        repo: https://charts.bitnami.com/bitnami
        remoteChart: redis
      - name: distributed-calculator
        chartPath: distributed-calculator
        overrides: # override Values.yaml
          replicaCount: 2
```

マニフェスト中にもあるように、SkaffoldはHelm Chartの `values.yaml` を上書きできます。リソースやパスワード等の環境差分をSkaffold側で吸収できるので、Chart自体をシンプルに保つことが出来ます。

Skaffoldだけでなく[Argo CDでもValue書き換えの機能がある](https://argo-cd.readthedocs.io/en/stable/user-guide/helm/)ので、環境ごとの自動デプロイも実現できそうです。

# おわりに

以上、Helm Chartでマイクロサービスのマニフェストを作ってみた紹介でした。

ChartではGo Template + [sprig](http://masterminds.github.io/sprig/)を使えるので、文字列処理の自由度が高くボイラープレート削減の手段が豊富だと感じました。
一方、テンプレートの挿入内容によってはSQLインジェクションのような構文の書き換えもできてしまうので、適切にクオーテーションを付けることを心掛けたいです（[公式](https://helm.sh/docs/howto/charts_tips_and_tricks/#quote-strings-dont-quote-integers)では`quote` 関数が推奨されています）。

明日は @ntoofu さんの 「社内CTFを開催しました」です。お楽しみに！
