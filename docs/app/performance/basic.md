# 基礎知識
## どのような時にパフォーマンス問題が起きるのか？
一口にパフォーマンスと言っても、CPU使用率・メモリ使用率・レスポンスタイム（レイテンシー）等の様々な指標がある。
これらの指標と、どのような時に遅くなるのか？を解説していく。

## CPU使用率
Railsアプリケーションが高いCPU使用率を示す場合、以下のような問題が発生することがある。

### 問題

- ActiveRecordによる複雑なクエリ
  - 複数のテーブル結合や、膨大なデータを処理するクエリが頻繁に実行されると、CPU使用率が急激に増加する可能性があります。特に、インデックスが適切に設定されていない場合、データベース検索のパフォーマンスが悪化し、結果としてCPUリソースが消費されることがあります。
- JSON生成
  - APIのレスポンスとして大量のデータをJSON形式で生成する場合、その処理もCPU負荷を増加させる可能性があります。特にas_jsonやto_jsonメソッドを使用した大規模データの変換はCPUを消費します。
- スレッド切り替えのオーバーヘッド
  - Pumaなどのマルチスレッド対応アプリケーションサーバーで、スレッドが多数起動されている場合、頻繁なスレッド切り替えによりCPU使用率が高くなる可能性があります。

### 根本原因
- 複雑なクエリや大量のデータを処理する際に、データベースとのやり取りが同期I/Oで行われている場合、クエリの結果が返ってくるまで各スレッドがI/O待ち状態となり、結果的にスレッドが専有されてします。
  - https://gihyo.jp/admin/serial/01/rdbms/0007
  - https://gihyo.jp/admin/serial/01/rdbms/0004

```
## 1. 同期I/Oによる待ち時間の増加

- 複数のテーブル結合や大規模データに対してインデックスが適切に設定されていない場合、データベースはクエリを処理するのに時間がかかります。このクエリが処理される間、アプリケーションはデータベースからの応答を待つため、CPUリソースは他の処理を行うことができず、スレッドはブロックされた状態になります。

## 2. スレッドの専有

- PumaやUnicornなどのマルチスレッド・マルチプロセス型のRailsサーバーは、リクエストごとにスレッドやプロセスを割り当てます。同期I/Oが多発すると、クエリ処理を待っている間、スレッドがI/O待ち状態でブロックされるため、新たなリクエストを処理するためのスレッドが不足します。
- 結果として、他のリクエストがスレッド/プロセスを待つことになり、リクエスト全体のレスポンスタイムが増加します。

## 3. スケーラビリティへの影響

- 同期I/Oが多発してスレッドが専有されると、スケーラビリティに問題が生じます。スレッド数が限られている環境では、すべてのスレッドがデータベースI/Oの待機状態になると、新規リクエストを処理できなくなり、スループット（処理可能なリクエスト数）が大幅に低下します。
```

一つの解決策として、非同期処理の導入がある。rubyだと、async gemがあったりする。
（メッセージングキュー使うのもあり。）

https://github.com/socketry/async

## メモリ使用率
Railsアプリケーションが高いメモリ使用率を示す場合、以下のような問題が発生することがあります。

問題の兆候:

- メモリリーク: Ruby on Railsでは、長時間稼働しているサーバープロセスでメモリリークが発生する可能性があります。例えば、インスタンス変数に過剰にデータを保持したり、不要になったオブジェクトが解放されない場合、プロセス全体のメモリ使用量が増加し続けます。
- データベースからの大量データの読み込み: 大量のレコードを一度にメモリにロードする場合（例: allやfind_eachを使わずに全てのデータを一括で読み込む）、一時的にメモリを大量に消費することがあります。特に、数十万件以上のレコードを扱う場合は、メモリ不足のリスクが高まります。
- スワッピングの発生: メモリ不足により、Railsサーバーがスワップを発生させると、ディスクI/Oが発生し、アプリケーションの応答速度が大幅に低下します。これは、特に高負荷状態で多くのリクエストを同時に処理している場合に顕著です。

Railsでの具体的な問題例:

- ActiveRecordでの巨大なクエリ結果: データベースから大量のレコードを一度に取得し、処理を行うと、メモリ使用量が急激に増加します。例えば、User.allのような操作で、全てのユーザーを一度にメモリにロードするとメモリ使用率が急増します。
- 大規模ファイルの読み込み/書き込み: Railsアプリケーションが大きなファイル（例: 画像やCSVファイル）をメモリ上で読み書きする場合、システムのメモリ消費量が増加します。これにより、メモリが圧迫されてスワップが発生するリスクがあります。
- 複数の長時間稼働プロセス: 長時間稼働するバックグラウンドジョブ（例: SidekiqやResque）がある場合、メモリ使用量が増加し、最終的にシステム全体のメモリが不足することがあります。

## レスポンスタイム
- I/O待ち: ディスクI/Oやデータベースクエリの処理が遅いと、システム全体の応答速度が低下する。
  - DB のI/Oだと、バッファプールをうまく使えていなくて、ディスクI/Oが発生し、遅くなる。
  - アプリケーションでも、`File.write`(rubyの場合)を実行したりすると、ディスクI/Oが発生するので、処理時間が長くなったりする。
    - OSのファイルに書き込んだり、読み込んだりするためである。
    - `StringIO`などのメモリに書き込むようにすると防げる。
  - 例えば、外部のAPIにアクセスする等でもネットワークI/Oが発生するので、レスポンスタイムがかなり悪くなったりする。
    - rubyなどの手続き型の言語では、レスポンスを待ってから処理を実行するためである。
  - これらのI/Oを少なくするために、Redisなどのインメモリのミドルウェアにキャッシュしたりする。
    - Redisはメモリ上にデータを持つため、データの取り出しがかなり速いという特徴がある。
- スレッド数の不足: スレッドプールが枯渇し、新たなリクエストに対応できなくなることがある。

(非同期処理に流したりもするよ！)
