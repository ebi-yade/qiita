---
title: OpenTelemetry の Sampler を Go で実装するときに見るところ
tags:
  - 'OpenTelemetry'
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

この記事は、[OpenTelemetry Advent Calendar 2023](https://qiita.com/advent-calendar/2023/otel) 23日目の記事です。

登録した時点では、metric の話になると思いましたが、なりませんでした。すみません。

![エントリ宣言](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/317486/54fc5f7e-7096-f5b7-8a1c-c13f6fbbc81d.png)

# ということで季節のご挨拶

都会の道はまばゆい光に包まれ、マジで寒いだけの高くて微妙な路面店を横目に「やっぱり家が一番だな」となる季節がやってまいりました。皆様いかがお過ごしでしょうか。

私はありがたいことに婚約をさせていただいて、将来への不安を抱えつつ、マジで情緒がアレになっております。最近はマジでアーキテクトやりたいなと思っており、ご祝儀代わりのオファーもお待ちしております。お応えできるか分かりませんが......。

根がクマさんなので体は勝手に冬眠モードに突入するし、そのくせ仕事はアレコレ湧いてきて、そんなこんなで思いの外 OpenTelemetry Metrics に関する進捗を出せませんでした。

# Sampler for Go やっていき

ということで、今回はトレース文脈の Sampler の話をします。

これもこれで参考になる方はいらっしゃるかと思いますので、ご容赦ください。

## 目線合わせ

まず、そもそも Sampler とは、[サンプリング](https://opentelemetry.io/docs/concepts/sampling/) してくれるマンです。トレースまたはスパンを処理・出力するかどうかを決めるものです。

Go においては、`sdktrace` の [Sampler インタフェース](https://pkg.go.dev/go.opentelemetry.io/otel/sdk/trace#Sampler) です。多分みんな `otel/trace` パッケージを `trace` と呼んでいて、 `otel/sdk/trace` パッケージのことは `sdktrace` と呼んでいると思います。もし違っても、私はそう呼んでいます（頑固）

ところで、本年ご好評いただいた [CNDT供養ブログ](https://techblog.kayac.com/cndt-operations-suite-observability) にて私は「Collector の誘惑を振り払って、ParentBased/RatioBasedで我慢しよう！」などと申し上げており、つまるところ Collector の活用やテイル・サンプリングには乗り出しておりません。

ですので、そのような高度なことをバキバキにやっている方々には、今回の記事はあまり参考にならないかもしれません。あくまでもアプリと同一のプロセスで動作する Sampler についての話です。

## Godoc に目を通す

そんなこんなで [Sampler インタフェース](https://pkg.go.dev/go.opentelemetry.io/otel/sdk/trace#Sampler) のドキュメントを見てみると、これでもかとばかりに `ShouldSample` というメソッドがありまして、要するに本丸です。読んで字の如く「`SamplingParameters` からサンプルするかどうかを決めてくれや」という話です。

```go
type Sampler interface {

	// ShouldSample returns a SamplingResult based on a decision made from the
	// passed parameters.
	ShouldSample(parameters SamplingParameters) SamplingResult

	// Description returns information describing the Sampler.
	Description() string
}
```

そこで [SamplingParameters 構造体](https://pkg.go.dev/go.opentelemetry.io/otel/sdk/trace#SamplingParameters) を除いてみると、結構魅力的なメンバが見当たります。

```go
type SamplingParameters struct {
	ParentContext context.Context
	TraceID       trace.TraceID
	Name          string
	Kind          trace.SpanKind
	Attributes    []attribute.KeyValue
	Links         []trace.Link
}
```

うおお！！`Attributes` って対象スパンの属性？！しかも `ParentContext` ってやつから親の情報も引き回せるんじゃね？！そんじゃ、こっから根掘り葉掘り情報引っ張ってきたら神サンプラー実装できるじゃんwwwwwwww余裕wwwwww

と見せかけて、実はそうでもないんですよね。

## 超重要: ShouldSample してる場所

焦らず、しっかり `ShouldSample` が呼ばれている場所を探してみましょう。

答えは [ここ](https://github.com/open-telemetry/opentelemetry-go/blob/885210bf33238f8c6cc94c3f3711b0036fbe77b1/sdk/trace/tracer.go#L98-L105) です。`sdktrace` における `tracer` 構造体の `newSpan` メソッドです。どちらも unexported なので、もしかしたら変更が入るかもしれませんが、要するに **スパンを開始する度に呼ばれている** 訳です。

少しだけ俯瞰してみると、一般的な呼び出しにおいては以下のような関係になっています。

```
(あなたのアプリ) -> trace.Tracer.Start 実態は sdktrace.tracer.Start -> sdktrace.tracer.newSpan
```

そして大事なのが、`SamplingParameters` の中身です。 `newSpan` 内のコードを抜粋すると、こんな感じです。

```go
samplingResult := tr.provider.sampler.ShouldSample(SamplingParameters{
	ParentContext: ctx,
	TraceID:       tid,
	Name:          name,
	Kind:          config.SpanKind(),
	Attributes:    config.Attributes(),
	Links:         config.Links(),
})
```

`Attributes` の `config` というのが目につくと思うのですが、これは言っちゃえばスパン開始（＝`trace.Tracer.Start`）時にオプションで渡した `trace.WithAttributes` です。もちろん渡さなければ「そこには何もないですね」というだけです。

## これも重要: Parent の詳細情報は引き出せない

`Attributes` がダメなら `ParentContext` から情報を引き出せばいいじゃん、と思うかもしれません。確かに、`trace` パッケージには [SpanFromContext](https://pkg.go.dev/go.opentelemetry.io/otel/trace#SpanFromContext) という魅力的な関数が用意されています。

ここで、良いニュースと悪いニュースがあります。アメリカ人みたいな言い回しはさておき、親の `Span` を引き出すことができました。良いニュースはここで終わりです。

悪いニュースは、`trace.Span` はインタフェースであり、詳細情報を与えてくれない設計になっていることです。これは [Godoc](https://pkg.go.dev/go.opentelemetry.io/otel/trace#Span) を見ていただければ明らかなことですが、なんと `IsRecording() bool` および `TracerProvider() TracerProvider` を除く全てが更新系です。

ここで運よく、似通った名前の [SpanContextFromContext](https://pkg.go.dev/go.opentelemetry.io/otel/trace#SpanContextFromContext) を見つける方もいらっしゃるかもしれませんが、それで引き出した `SpanContext` を目の当たりにして考えつくような事をやったのが `sdktrace` の `ParentBased` な訳です。[この辺り](https://github.com/open-telemetry/opentelemetry-go/blob/885210bf33238f8c6cc94c3f3711b0036fbe77b1/sdk/trace/sampling.go#L266-L282) にその全てがあります。

何にしても、**`ShouldSample` のシグニチャだけ見て「こいつから情報引き回せば多分良い感じにサンプリング決定できて、良い感じに継承されるんやろな〜」みたいな甘い認識でいると、普通に間違った実装をすることになります。**

## 改めて ParentBased って何だっけ

先ほど、こんなことを申し上げました。

> 焦らず、しっかり `ShouldSample` が呼ばれている場所を探してみましょう。
> 〜（略）〜
> 要するに **スパンを開始する度に呼ばれている** 訳です。

ということで、ShouldSample 自体はスパンを開始するたび刹那的に呼ばれている訳ですね。そこで適当に「`RatioBased` 召喚！！これが俺の答えや！！」みたいなことやってたら、親のスパンはサンプルされていないのに子のスパンはサンプルされてる」的な、ゾンビ現象があちこち起こって収拾がつかなくなる訳です。

そこで、先ほどの「良い感じに継承されるんやろな〜」という甘い認識に基づく緩い期待を実現してくれるのが `ParentBased` なのです。逆にいうと、アプリと同一のプロセスで動作する Sampler について、「何でもかんでも `AlwaysSample`」するような場合を除いて `ParentBased` を使わないなんてことは論外に近いのです。

強い言葉を使ってしまって恐縮ではあるんですが、そもそも Sampler は通常グローバルに運用される TracerProvider に登録するものなので、「特定のURLへのアクセスでは常にサンプルしたい」ぐらいの話では `ParentBased` を拡張して使うなり、同様のものを再発明することになります。繰り返しますが、**「何でもかんでも `AlwaysSample`」以外は `ParentBased` なんです。**

ということで、ユーザー視点で ParentBased を爆速理解していきましょう。早い話、 [ソースコード](https://github.com/open-telemetry/opentelemetry-go/blob/885210bf33238f8c6cc94c3f3711b0036fbe77b1/sdk/trace/sampling.go#L170-L195) を読んでください。16行なので貼り付けちゃいます。

```go
func ParentBased(root Sampler, samplers ...ParentBasedSamplerOption) Sampler {
	return parentBased{
		root:   root,
		config: configureSamplersForParentBased(samplers),
	}
}

type parentBased struct {
	root   Sampler
	config samplerConfig
}

func configureSamplersForParentBased(samplers []ParentBasedSamplerOption) samplerConfig {
	c := samplerConfig{
		remoteParentSampled:    AlwaysSample(),
		remoteParentNotSampled: NeverSample(),
		localParentSampled:     AlwaysSample(),
		localParentNotSampled:  NeverSample(),
	}

	for _, so := range samplers {
		c = so.apply(c)
	}

	return c
}
```

分かりやすい話ですね。基本はリモートだろうがローカルだろうが、親がサンプルされていれば子もサンプルされるし、親がサンプルされていなければ子もサンプルされない、ということです。もちろんオプションで置き換えることができます。まぁこれを無理やり拡張しなくても良いとは思いますが、結局要求を整理して実装していったら同じようなことに辿り着くはずです。

## そういえばリクエスト分類はどう実装できるんだっけ

ということで、さっきの「特定のURLへのアクセスでは常にサンプルしたい」とか、そこまで行かなくとも「特定のURLへのアクセスではサンプルレートを変更したい」みたいなやつって、`otelhttp` を使う場合はどうすれば良いんだっけ？ということが気になる方もいらっしゃると思います。

結論から申し上げますと、あまり綺麗な方法は存在しません。ぶっちゃけ `otelhttp` が微妙なことが大きな原因です。自分自身トレースのみならずメトリクス文脈でも `otelhttp` を使ってゴニョゴニョしてみましたが、残念ながら欲しいものを揃えるにはダーティハックが必要となる現状があります。「頑張らないオブザーバビリティ」のブログでも、`otelhttp`を使う人への注意喚起をしましたが、それから二ヶ月、「使わないのも良い選択肢だよ」が今の心境です。

何なら `otelhttp` 自体が（有名な開発者とはいえ）個人リポジトリに依存しており、「準公式」と捉える程でもないと感じています。ディスみたいになると嫌ですが、読めば読むほど大したことをしている訳でもないと分かるので、そのインタフェースに縛られてダーティハックを仕込むぐらいなら自前実装も良い選択だという話です。

そんなこんなで、そのうち `otelhttp` の代替実装を用意したいな〜と思いつつ、それは乞うご期待として、今回は一応ダーティハックの紹介をしておきます。

二ヶ月前のブログで申し上げたとおり、`otelhttp.NewHandler` はリクエストスコープで呼ぶわけにはいきません。リクエストスコープで小細工をできるのはオプションの `otelhttp.WithSpanNameFormatter` で渡すことができる `func(string, *http.Request) string` だけです。これを使って、リクエストのパスを一定の規則で加工して **スパン名** にできます。また、同時に `otelhttp.WithSpanOptions` で **スパン開始時に渡すAttributes** も仕込むことができます。

これら二つによって、`ShouldSample` に渡される `SamplingParameters` の `Name` と `Attributes` を加工できることになります。なので、後者で「これはHTTPリクエストを受け付けているレイヤのスパンだよ」ということを示すシグナルを与えて、`ShouldSample` 内で引っかかった場合にのみ「`ParentBased` とは違う動き」をさせます。それは前者の `Name` からリクエストURL を抽出して、それに応じたサンプリングレートを返すというものです。

# まとめ

リクエスト分類の話を見て「何が何だか分からん」という方は、もう少しお待ちください。今のプロジェクトでは、割にダーティな方で何とかしてしまった節があり反省まみれなので、年明け以降に「俺が考えた最強の HTTP 計装 for Go」を何かしらの形でドドンと公開したいと思います。ちょっと社内事情に左右されそうなのですが、流石にどこかしらで機会は掴めると見ております。

もし、「いやいや、俺はすでにクリーンな方法でリクエスト分類を実装しているが？」という方がいらっしゃったら教えてください。めっちゃ気になります。

# 【宣伝】会社のアドベントカレンダー

私が働いている[面白法人カヤック](https://www.kayac.com/)でも、Qiita のアドベントカレンダー企画をやっています。

Qiita に投稿している記事は会社と直接関係ありませんが、企画の運営をしているのは私です。

おかげさまで、今年もバラエティ豊かなエントリが出揃いました。ぜひご確認ください！

https://qiita.com/advent-calendar/2023/kayac-group

# 修正リクエストを受け付けています

この記事は GitHub で管理しています。以下のリポジトリに Pull Request を送っていただければ、記事内容の修正を検討いたします。（コメント欄に書いていただいても構いません。）

https://github.com/ebi-yade/qiita/blob/main/public/otel-2023-day23.md