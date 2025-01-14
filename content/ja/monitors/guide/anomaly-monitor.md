---
aliases:
- /ja/monitors/faq/anomaly-monitor.md
title: 異常検知モニター
---

### あらゆるものに対して異常検知を使用した方がいいですか。

いいえ。異常検知は、予測可能なパターンを持つメトリクスを視覚化し、監視するのに役立ちます。例えば、`my_site.page_views{*}` は、ユーザーのトラフィックによって駆動し、時間帯や曜日によって予測可能に変化します。そのため、異常検知に適しています。

**注**: 異常検知が適切な予測を行うには履歴データが必要です。数時間や数日分のメトリクスしか収集されていない場合、異常検知は役に立ちません。

### ダッシュボードで複数のグループに異常検知を使用できないのはなぜですか。

1 つのグラフに独立した多数の時系列を追加すると[スパゲッティ化][1]することがあり、これに異常検知の視覚化を追加すると事態が悪化します。

{{< img src="monitors/monitor_types/anomaly/spaghetti.png" alt="スパゲッティ化" style="width:80%;">}}

ただし、1 つのグラフに複数の系列を一度に 1 つ追加することは可能です。マウスを重ねたときにのみ、灰色の帯が表示されます。

{{< img src="monitors/monitor_types/anomaly/anomaly_multilines.png" alt="異常多発線" style="width:80%;" >}}

### 過去の異常値は現在の予測に影響しますか。

`basic` を除くすべてのアルゴリズムは、膨大な量の履歴データを使用しているため、ほとんどの異常に対して堅牢です。最初のグラフでは、メトリクスが 0 になった後も灰色の帯が 400K 付近に留まっています。

{{< img src="monitors/monitor_types/anomaly/anomalous_history.png" alt="anomalous_history" style="width:80%;">}}

2 つ目のグラフは、1 日後の同じメトリクスを示しています。帯の計算には前日のデータも使用されていますが、以前に発生した異常値には影響されていません。

{{< img src="monitors/monitor_types/anomaly/no_effect.png" alt="無効" style="width:80%;">}}

### ズームインすると異常値が「消える」のはなぜですか。

ズームレベルが異なると、同じクエリでも特性が異なる時系列になる可能性があります。表示する期間が長いほど、各ポイントは、さらに粒度が細かい多数のポイントの集計値を表すことになります。したがって、粒度が細かいポイントでは、観察されるノイズがこれらの集計ポイントによって隠される可能性があります。たとえば、1 週間を表示するチャートが 10 分間を表示するチャートより滑らかに (ノイズが少なく) 表示されることはよくあります。

異常監視を正確に行うためには、灰色の帯の幅が重要です。通常のノイズがほとんど帯の中に入ってしまい、異常として表示されないような帯の幅である必要があります。しかし、通常のノイズを含むほど帯が広いと、特に短いタイムウィンドウを表示する場合、いくつかの異常が隠れるほど帯が広くなってしまうことがあります。

例えば、`app.requests` というメトリクスはノイズが多いが、平均値は常に 8 です。ある日、9 時から 10 分間異常な時間があり、その間の平均値は 10 です。以下のグラフは、1 日のタイムウィンドウを持つ系列を示し、グラフの各ポイントは 5 分を要約したものです。

{{< img src="monitors/monitor_types/anomaly/disappearing_day.png" alt="消えた日" style="width:70%;" >}}

この灰色の帯は適切なものだと言えます。時系列内のノイズが収まるだけの十分な幅があり、しかも、幅が広すぎることがないので 9:00 の異常値が目立つようになっています。次のチャートは、30 分のタイムウィンドウにズームインしたビューで、上の 10 分間の異常値が含まれています。グラフの各ポイントは 10 秒間の集計値です。

{{< img src="monitors/monitor_types/anomaly/disappearing_half_hour.png" alt="消えた 30 分" style="width:70%;" >}}

ここでも、帯は合理的にサイズ変更されています。8:50 ～ 9:00 と 9:10 ～ 9:20 の正常なデータが帯の中にあるからです。これよりも帯が狭くなると、通常のデータが異常と見なされるようになってしまいます。このグラフの帯が、前のグラフより 8 倍近く広くなっていることに注目してください。9:00 ～ 9:10 の異常期間は系列の他の部分とは違って見えますが、帯の外側に出るほど極端ではありません。

一般に、ズームインしたときに異常値が消えても、それが異常値ではないという意味ではありません。つまり、ズームインしたビューでの個別のポイントは個々の異常として切り分けられなくても、同時に発生している普通とは少し違う多数のポイントが異常ということになります。

### 一部の関数を異常検知と組み合わせようとするとクエリパースエラーが発生するのはなぜですか。

`anomalies()` 関数の呼び出しの中にすべての関数をネストできるわけではありません。特に、`cumsum()`、`integral()`、`outliers()`、`piecewise_constant()`、`robust_trend()`、`trend_line()` の各関数は、異常検知モニターまたはダッシュボードクエリに組み込めません。

異常検知は、履歴データを使用して系列の正常な挙動のベースラインを確立します。上記の関数は、クエリウィンドウの配置に影響されます。あるタイムスタンプでの系列の値は、それがクエリウィンドウ内のどこにあるかによって大幅に変化する可能性があります。この不安定性のため、異常検知機能は安定した系列のベースラインを決定できなくなります。

### adaptive アルゴリズムはどうなったのですか？

Datadog はアルゴリズムを進化させ、adaptive アルゴリズムを使用しないようにしました。`adaptive` アルゴリズムを使用する既存のモニターはそのままで、引き続き動作します。

### count_default_zero 引数とは何ですか？

以前は、Datadog はカウントメトリクスをゲージとして扱い、報告されたポイント間を補間していました。しかし、レガシーモニターでは、`count_default_zero` 引数を使用することで、以前の動作が保持されます。

### カウントメトリクスをゲージとして扱いたい場合はどうすればよいですか？

エラーのようなカウントメトリクスを使用する場合、カウントの間を補間しないことは理にかなっています。しかし、1 時間ごとに発生する定期的なジョブがある場合、実行の間にメトリクスが 0.0 の値を報告しない方がより理にかなっているかもしれません。これを実現するには、2 つの異なる方法があります。

1. ロールアップ (高度なオプションのセクションにあります) を 1 時間に設定します。
2. API を使って明示的に `count_default_zero='false'` を設定します。

### "Advanced Options" でロールアップ期間を設定する方法と、.rollup() を使用してクエリで設定する方法では、どのような違いがありますか？

ロールアップをクエリで明示的に設定すると、異常値モニターのロールアップ期間オプションが無視されます。

### メトリクスの値が X 未満の場合は異常でもかまいません。このような異常値を無視する方法はありますか。

範囲を超える値をアラートする異常値モニター **A** を作成し、さらにしきい値アラートを使用して X より大きい値でトリガーする別の[メトリクスモニター][2] **B** を作成します。続いて **A && B** の[複合条件モニター][3]を作成します。

### モニターを保存しようとしたら、"alert and alert recovery criteria are such that the monitor can be simultaneously in alert and alert recovery states" (アラートとアラートリカバリの基準が、モニターが同時にアラート状態およびアラートリカバリ状態になり得る内容です) というメッセージが表示され、保存できませんでした。どうしてですか？

アラート期間とアラートリカバリ期間に異なるウィンドウを設定すると、あいまいな状態になる可能性があります。アラートとアラートリカバリのウィンドウサイズは、両方が同時に満たされないように設定する必要があります。たとえば、2 時間のウィンドウに対してアラートしきい値 50% を設定し (つまり、異常が 1 時間続けばアラートをトリガーする)、10 分間のウィンドウに対して[リカバリしきい値][4] 50% を設定すると (つまり、非異常が 5 分間続けば回復する)、アラート状態とアラートリカバリ状態が同時にトリガーされる可能性があります。たとえば、最後の 5 分間が異常ではなく、1 時間前が異常だった場合は、アラートとアラートリカバリの両方がトリガーされます。

### 夏時間は異常検知モニターにどのように影響しますか？

Datadog のモニターは UTC 時間を使用し、デフォルトではローカルタイムゾーンを考慮しません。ユーザーアクティビティは、ユーザーのローカルタイムに対して変わらないことが普通なので、UTC 時間からは相対的にシフトします。これが予期しない異常値として検出される可能性があります。

Datadog では、タイムシフトに対して自動的に修正が行われるタイムゾーンを異常検知モニターごとに構成することができます。詳細については、[ローカルタイムゾーンを考慮した異常検知モニターの更新方法][5]を参照してください。

[1]: https://www.datadoghq.com/blog/anti-patterns-metric-graphs-101
[2]: /ja/monitors/create/types/metric/
[3]: /ja/monitors/create/types/composite/
[4]: /ja/monitors/guide/recovery-thresholds/
[5]: /ja/monitors/guide/how-to-update-anomaly-monitor-timezone/