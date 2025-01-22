# key-valueが並んでいるテーブルをカラム分けするSQL

## 1. 課題

例えばアンケートを格納するテーブルで、`user_id`、`question`、`answer`というカラムがあり、各`user_id`と`question`のペアで回答が格納されているとする。questionが有限個 (かつ既知) の場合に、ユーザーごとに1つのレコードに集約したい場合が稀によくある。

例えば以下のようなデータがあるとする：

| user_id | question  | answer   |
|---------|-----------|----------|
| 1       | 質問1     | はい     |
| 1       | 質問2     | いいえ   |
| 2       | 質問1     | いいえ   |
| 3       | 質問1     | はい     |
| 3       | 質問2     | はい     |

これを次のような形式に変換し、ユーザーごとに統合されたレコードを得たい：

| user_id | 質問1 | 質問2 |
|---------|-------|-------|
| 1       | はい  | いいえ|
| 2       | いいえ| NULL  |
| 3       | はい  | はい  |

## 2. 解決策

この課題を解決するために、SQLクエリを利用する。以下に示すSQL文で、ユーザーごとに回答を集約し、`question`ごとに個別のカラムとして表示する。

```sql
SELECT
    user_id,
    MAX(CASE WHEN question = '質問1' THEN answer END) AS 質問1,
    MAX(CASE WHEN question = '質問2' THEN answer END) AS 質問2
FROM
    survey_results
GROUP BY
    user_id;
```

このクエリでは、`GROUP BY user_id`を用いてユーザーごとに集約されたレコードから目的の質問を抽出するのに`CASE`文を使用している。`MAX`関数は、同じ`user_id`と`question`の組み合わせが複数あった場合のハンドリングのために使われる。未回答の質問に対しては`NULL`が返される。
