# APサーバーで集計するよりも、DBサーバーで集計した方が高速になる
## 比較した時のSQLとコード
### 集計型SQL
DB側で集計までやってしまう例。
この場合だと、DBからAPPに集計した値を三つ返すだけで良くなる。
```sql
SELECT SUM(billing), MAX(billing), AVG(billing)
FROM orders
WHERE order_date = '2017/3/1';
```

以下のような処理順序になる。

1. HDDからメモリへデータを読み出す。
2. HDDから読み出したままの生データを扱いやすいようにテーブル形式に整形する。(HDDにはレコードのデータはテーブル形式で保存されているわけではない。)
3. CPU内で、SUM, MAX, AVGの集計関数で集計する。
4. 3.で集計した三つのデータのみをDBサーバからAPPサーバに転送する。

### 非集計型SQL
APP側で合計・最大値・平均値を算出する例。
この場合だと、DBから、APPにbillingカラムの値を全て渡す必要が出てくる。
```sql
SELECT billing
FROM orders
WHERE order_date = '2017/3/1';
```

以下のような処理順序になる。

1. HDDからメモリへデータを読み出す。
2. HDDから読み出したままの生データを扱いやすいようにテーブル形式に整形する。(HDDにはレコードのデータはテーブル形式で保存されているわけではない。)
3. メモリ上のテーブル形式のデータから、billingカラムのデータのみを取り出す。
4. 3.のデータをDBサーバからAPPサーバに転送する。

## なぜDB側で集計した方がDB側でも低負荷になるのか？
- DBからAPPへのデータ転送量がかなり減るので、比較すると集計型SQLの方が、データの転送時間がかなり短くなるため。
- 【集計型SQL】DBサーバーでの集計関数の処理の負荷 < 【非集計型SQL】billingカラムの切り出し処理
  - SQLでの集計関数の処理は、DBサーバーのCPU内で行われるので、とても高速なため。
  - 集計型SQLでは、メモリ内でのbillingカラムの切り出し処理が不要になるため。  

## 前提知識
- 帯域幅について
  - 単位はビット毎秒。
- I/Fとは？
  - 「インターフェース」の省略表現のこと。
  - https://wa3.i-3-i.info/word110409.html
- オーバーヘッドとは？
  - 処理や通信に際して間接的に生じる負荷などを指す。
  - https://wa3.i-3-i.info/word12471.html
  - https://e-words.jp/w/%E3%82%AA%E3%83%BC%E3%83%90%E3%83%BC%E3%83%98%E3%83%83%E3%83%89.html
