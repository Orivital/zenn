---
title: "VECTOR 型"
---

https://dev.mysql.com/doc/refman/9.7/en/vector.html

MySQL 9.0 から VECTOR 型（The VECTOR Type）が出ました。浮動小数点の配列を1つの値として持つデータ型です。公式の定義では、`VECTOR(N)` は最大 N 個のエントリを持つ構造で、各エントリは 4 バイトの単精度 float です。デフォルトは 2048、最大は 16383 です。

通常の列は `INT` なら整数 1 個、`VARCHAR` なら文字列 1 個です。`VECTOR` は、その並び全体を 1 つの値として持ちます。
例えば `[0.12, -0.45, 0.88, ...]` のような並びを、JSON やカンマ区切りの文字列ではなく、専用のバイナリとして保存します。

`VECTOR(N)` の N は、「必ずその個数」ではなく、最大個数（容量）です。

| 記法 | 意味 | 容量の目安 |
| --- | --- | --- |
| `VECTOR(384)` | 最大 384 個まで入る | 最大 1536 バイト |
| `VECTOR`（括弧なし） | 省略時のデフォルト、最大 2048 | 最大 8192 バイト |
| `VECTOR(16383)` | 指定できる上限 | 最大 65532 バイト（64KB 弱） |

`VECTOR()` は構文エラーです。`VECTOR(16384)` は上限（16383）を超えるため使えません。

各エントリは IEEE 754 の単精度（float32）です。`DOUBLE`（8 バイト）ではありません。精度はおおよそ小数点以下 6〜7 桁です。

## ユースケース

**セマンティック検索**
「返品のやり方」で検索して、「返品ポリシー」「交換手続き」のような、文言が違う文書も拾うことを狙う。

**RAG（社内データ付きのチャット）**
質問に近い社内 PDF / マニュアルを先に取り出し、その抜粋を LLM に渡して答える。公開データだけで訓練された LLM に、自社データを足すための仕組み。

**レコメンド**
商品・記事・ユーザーの埋め込みを近い順に並べて、「似た商品」「似た記事」を出す。

**チャットボット / Q&A**
FAQ やチケット履歴をベクトル化して、自然言語の問い合わせに近い回答を返す。

## VECTOR はどう役立つか

例えば、上記で挙げた「セマンティック検索」は VECTOR でどのように役立つか、次の検証で動かします。
通常の検索（`LIKE` や `FULLTEXT`）は文字列を見ます。「返品のやり方」という句ではヒットしません。`LIKE '%返品%'` なら返品ポリシーは出ますが、交換手続きは出ません。

セマンティック検索は、文章を **埋め込み（embedding）** という数値の並びに変換します。
埋め込みモデルは「近い意味は近い位置」に置くので、「返品のやり方」と「返品ポリシー」はベクトル空間上で近くなります。
VECTOR 型は、その数値の並びを列として持つための型です。

流れはこうです。

1. 文章をモデルでベクトル化し、`VECTOR` 列に保存する
2. 検索クエリも同じモデルでベクトル化する
3. 近い順に取る。距離なら小さいほど近く、類似度なら大きいほど近い

Community 版と Commercial 版で DB 単体できるのは、すでにできたベクトルを `VECTOR` 列へ保存することです。ステップ 1・2 の「モデルでベクトル化する」処理と、ステップ 3 の距離計算（`DISTANCE()`）は、MySQL の中では HeatWave / MySQL AI の機能です。
ただし埋め込み生成は、Python や API などアプリ側で行うのが一般的です。その場合、Community / Commercial でも 1 と 2 は実現でき、3 はアプリ側（または自前の SQL）で距離を計算することになります。

## 検証

Docker で MySQL 9.7 を使います。ホストの 33067 番に出します。

```bash
docker run --name mysql97-vector-demo \
  -e MYSQL_ROOT_PASSWORD=demo \
  -e MYSQL_DATABASE=vecdemo \
  -p 33067:3306 \
  -d mysql:9.7
```

普通にコンテナに入ろうとすると、ロケールが `C` / `POSIX` となり、`mysql` の入力処理が ASCII しか受け付けなくなり、日本語入力が全て空文字になってしまうため、ロケールを UTF-8 に設定します。

```bash
docker exec -it -e LANG=C.UTF-8 -e LC_ALL=C.UTF-8 mysql97-vector-demo mysql -uroot -pdemo vecdemo
```

返品に関する FAQ のテーブルを作成します。

```sql
CREATE TABLE faqs (
  id INT NOT NULL AUTO_INCREMENT,
  title VARCHAR(100) NOT NULL,
  body VARCHAR(255) NOT NULL,
  PRIMARY KEY (id)
);
```

```sql
INSERT INTO faqs (title, body) VALUES
  ('返品ポリシー', '購入後30日以内なら未使用品を返品できます。'),
  ('交換手続き', 'サイズや色の交換はマイページから申請してください。'),
  ('配送状況の確認', '伝票番号で荷物の現在地を追跡できます。'),
  ('支払い方法', 'クレジットカードとコンビニ払いが使えます。');
```

実際に LIKE すると、「返品のやり方」は 0 件、「返品」は返品ポリシーだけです。

```text
mysql> SELECT id, title FROM faqs
    -> WHERE title LIKE '%返品のやり方%' OR body LIKE '%返品のやり方%';
Empty set (0.011 sec)
```

```text
mysql> SELECT id, title FROM faqs
    -> WHERE title LIKE '%返品%' OR body LIKE '%返品%';
+----+--------------------+
| id | title              |
+----+--------------------+
|  1 | 返品ポリシー       |
+----+--------------------+
1 row in set (0.023 sec)
```

次は列を足して、埋め込みを入れます。
今回、`sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2` という日本語を含む多言語の小型モデルをアプリ側で使います。このモデルは 1 入力を 384 個の数値にするため、`VECTOR(384)` と指定します。

```sql
ALTER TABLE faqs ADD COLUMN embedding VECTOR(384);
```

Community / Commercial には `DISTANCE()` がないので、保存と近い順の計算は Python 側です。

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install sentence-transformers pymysql numpy
```

```python
import json
import numpy as np
import pymysql
from sentence_transformers import SentenceTransformer

# 文書もクエリも、同じモデルでベクトル化する
MODEL = "sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2"
QUERY = "返品のやり方"

# MySQL の STRING_TO_VECTOR() が受け付ける形式 '[0.12,-0.03,...]' にする
def literal(vec):
    return "[" + ",".join(f"{float(x):.8g}" for x in vec) + "]"

# 長さ 1 に揃えたベクトル同士なら、内積が余弦類似度になる
def cosine(a, b):
    return float(np.dot(a, b))

# モデルを読み込む（初回は Hugging Face からダウンロードする）
model = SentenceTransformer(MODEL)

# ホストからコンテナの MySQL へ接続する（コンテナ内なら port=3306）
conn = pymysql.connect(
    host="127.0.0.1", port=33067,
    user="root", password="demo", database="vecdemo", charset="utf8mb4",
)
cur = conn.cursor()

# FAQ の本文を取る
cur.execute("SELECT id, title, body FROM faqs")
rows = cur.fetchall()

# タイトルと本文を 1 入力にして埋め込みにする
# normalize_embeddings=True で長さを 1 に揃える（内積が類似度になる）
texts = [f"{title}。{body}" for _, title, body in rows]
vecs = model.encode(texts, normalize_embeddings=True)

# 文字列化したベクトルを STRING_TO_VECTOR() で VECTOR 列へ保存する
for (id_, title, body), vec in zip(rows, vecs):
    cur.execute(
        "UPDATE faqs SET embedding = STRING_TO_VECTOR(%s) WHERE id = %s",
        (literal(vec), id_),
    )
conn.commit()

# 保存できたか、次元が 384 かを確認する
cur.execute("SELECT id, title, VECTOR_DIM(embedding) FROM faqs")
print("dims:", cur.fetchall())

# 検索クエリも同じモデルでベクトル化する
q = model.encode([QUERY], normalize_embeddings=True)[0]

# VECTOR はバイナリなので、VECTOR_TO_STRING() で数値の配列に戻してから比べる
cur.execute("SELECT id, title, VECTOR_TO_STRING(embedding) FROM faqs")
ranked = []
for id_, title, s in cur.fetchall():
    ranked.append((cosine(q, np.array(json.loads(s), dtype=np.float32)), id_, title))

# 類似度が大きい順（近い順）に出す
for score, id_, title in sorted(ranked, reverse=True):
    print(f"{score:.4f}  [{id_}] {title}")
conn.close()
```

結果はこちら。

```text
dims: ((1, '返品ポリシー', 384), (2, '交換手続き', 384), (3, '配送状況の確認', 384), (4, '支払い方法', 384))
0.4119  [1] 返品ポリシー
0.3115  [4] 支払い方法
0.2748  [3] 配送状況の確認
0.2611  [2] 交換手続き
```

数字は余弦類似度で、大きいほど近いです。HeatWave の `DISTANCE(..., 'COSINE')` は距離なので、近いほど 0 に近くなります。ここの 0.4119 とは向きが逆です。

| 順位 | 類似度 | 文書 | 備考 |
| --- | --- | --- | --- |
| 1 | 0.4119 | 返品ポリシー | `LIKE '%返品のやり方%'` では 0 件だった行が首位 |
| 2 | 0.3115 | 支払い方法 | 返品とは無関係だが、このモデルでは 2 位 |
| 3 | 0.2748 | 配送状況の確認 | 同上 |
| 4 | 0.2611 | 交換手続き | 冒頭の例では拾う想定だったが、ここでは最下位 |

VECTOR 列に入れて近い順に並べる、までは確認できました。
一方「交換手続きも上がる」は、この短い文とこのモデルでは成り立ちませんでした。VECTOR 列は箱で、近い順に並ぶかどうかは埋め込みモデルと入力文の話です。
