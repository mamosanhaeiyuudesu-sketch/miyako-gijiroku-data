# 宮古島市議会 議事録処理システム

宮古島市議会の議事録PDFをダウンロードし、テキスト抽出・分析・AI検索用データベース構築を行うスクリプト群。

---

## ディレクトリ構成

```
miyako/
├── 0_download.py            # Step 0: PDFをダウンロード
├── 1_extract_text.py        # Step 1: PDFからテキスト抽出
├── 2_extract_features.py    # Step 2: TF-IDF特徴量抽出
├── 3_upload_vectorstore.py  # Step 3: OpenAI Vector Storeにアップロード
├── 4_split_sessions.py      # Step 4: テキストを会期ごとに分割
├── 5_upload_sessions.py     # Step 5: 会期ファイルをOpenAIにアップロード
├── analyze_speakers.py      # 発話者分析（独立スクリプト）
├── extract_gijiroku.py      # PDFをSQLiteに取り込む（DBパイプライン用）
├── export_to_d1.py          # SQLiteからCloudflare D1へ移行
├── schema.sql               # DBスキーマ定義
│
├── gijiroku/                # ダウンロードしたPDF（.gitignore対象）
├── sessions/                # 会期ごとのテキストファイル（.gitignore対象）
└── output/                  # 生成ファイル（.gitignore対象）
    ├── gijiroku_all.txt              # 全会期結合テキスト
    ├── features.json                 # 会期ごとのTF-IDFキーワード
    ├── vectorstore_id.txt            # OpenAI Vector Store ID
    ├── miyako-file-ids.json          # 会期名 → OpenAI file_id マッピング
    ├── speakers_meta.json            # 発話者統計
    ├── tfidf_words.csv               # 発話者×単語 TF-IDF
    ├── tfidf_categories.csv          # 発話者×カテゴリ TF-IDF
    └── miyakojima_council_members_20years.json  # 議員プロフィール（入力ファイル）
```

---

## パイプライン

### RAGパイプライン（OpenAI検索）

PDFから全文テキストを抽出し、OpenAI APIを使ったRAG（検索拡張生成）用データを構築する。

```
0_download.py
    ↓  gijiroku/*.pdf
1_extract_text.py
    ↓  gijiroku_all.txt（全会期結合テキスト）
2_extract_features.py
    ↓  features.json（会期ごとのキーワード上位30件）
3_upload_vectorstore.py
    ↓  vectorstore_id.txt（全文をVector Storeにアップロード）
4_split_sessions.py
    ↓  sessions/*.txt（会期ごとに分割）
5_upload_sessions.py
    ↓  miyako-file-ids.json（会期ごとにFiles APIへアップロード）
```

### DBパイプライン（Cloudflare D1）

PDFのメタデータと全文をSQLiteに格納し、Cloudflare D1に移行する。

```
0_download.py
    ↓  gijiroku/*.pdf
extract_gijiroku.py
    ↓  gijiroku.db / gijiroku.sql（1行1PDF のシンプルなテーブル）
export_to_d1.py
    ↓  Cloudflare D1（sessions / bills / utterances テーブル）
```

> **注意**: `export_to_d1.py` は `gijiroku_all.db`（sessions/bills/utterances 構造）を入力とする。このDBを生成するスクリプトは別途必要。

---

## 各ファイルの説明

### `0_download.py`
宮古島市の公式サイトから議事録PDFを一括ダウンロードする。

- 取得先: `https://www.city.miyakojima.lg.jp/gyosei/gikai/gijiroku.html`
- 出力: `gijiroku/*.pdf`
- サーバー負荷軽減のため1秒間隔でダウンロード

### `1_extract_text.py`
`gijiroku/` 内のPDFからテキストを抽出し、全会期を1ファイルに結合する。

- スキャンPDF（テキスト取得不可）は自動スキップ
- 元号（令和・平成・昭和等）を西暦に変換してヘッダーに付与
- 出力: `gijiroku_all.txt`（約100MB超）

**出力フォーマット:**
```
==== 令和5年 第4回 定例会 2023-09-04〜2023-09-22 ====
（本文テキスト）

==== 令和5年 第3回 定例会 ...
```

### `2_extract_features.py`
`gijiroku_all.txt` を会期ごとに分割し、TF-IDFで各会期の特徴語を抽出する。

- 形態素解析: [Fugashi](https://github.com/polm/fugashi) + UniDic
- 議会用ストップワード（「議員」「質問」「答弁」等）を除外
- 各会期の上位30キーワードを抽出
- 出力: `features.json`

### `3_upload_vectorstore.py`
`gijiroku_all.txt` をOpenAI Vector Storeにアップロードする。

- OpenAI Files APIにテキスト全体をアップロード
- Vector Storeを作成してファイルを紐付け
- 出力: `vectorstore_id.txt`（Vector Store IDを保存）
- 環境変数 `OPENAI_API_KEY` が必要

### `4_split_sessions.py`
`gijiroku_all.txt` を会期ごとの個別ファイルに分割する。

- `==== ... ====` ヘッダーを区切りとして分割
- ファイル名例: `R5-4-定例会_2023-09-04.txt`
- 出力: `sessions/*.txt`（100件超）

### `5_upload_sessions.py`
`sessions/` 内の会期ファイルをOpenAI Files APIへアップロードし、Vector Storeに追加する。

- すでにアップロード済みの会期はスキップ（再実行対応）
- 会期名とfile_idのマッピングを保存
- 出力: `miyako-file-ids.json`
- 環境変数 `OPENAI_API_KEY` が必要

### `analyze_speakers.py`
議事録テキストを発話者ごとに集計し、TF-IDFで各議員の関心テーマを分析する。

- `gijiroku_all.txt` から話者と発言を抽出
- `miyakojima_council_members_20years.json`（議員プロフィール）と突合
- 13カテゴリ（環境・防災、農業・水産業、観光 等）でスコアを集計
- カレントディレクトリから実行すること（相対パスを使用）

**出力:**
| ファイル | 内容 |
|---------|------|
| `output/speakers_meta.json` | 発話者一覧・発言数・性別・党派等の統計 |
| `output/tfidf_words.csv` | 発話者×単語のTF-IDFスコア（上位200語） |
| `output/tfidf_categories.csv` | 発話者×カテゴリのスコア |

### `extract_gijiroku.py`
`gijiroku/` 内のPDFを一括でSQLiteに取り込む（DBパイプライン用）。

- 表紙から年度・回・会期種別・開閉会日を抽出
- スキャンPDFは `is_readable=0` として記録
- 出力: `gijiroku.db`（SQLite）、`gijiroku.sql`（D1 import用SQLダンプ）

**テーブル構造 (`gijiroku`):**
| カラム | 説明 |
|--------|------|
| filename | PDFファイル名 |
| nendo | 年度（例: 令和5年） |
| kai | 回次 |
| session_type | 定例会 / 臨時会 |
| date_start / date_end | 開閉会日（ISO 8601） |
| content | 全文テキスト |
| is_readable | スキャンPDFは 0 |

### `export_to_d1.py`
SQLiteデータベースをCloudflare D1にバッチ移行する。

- `wrangler d1 execute` コマンドを使用
- 100件ずつバッチ処理（D1のサイズ制限対応）
- 接続エラー時は自動リトライ（最大5回）
- `--utterance-start-batch N` で途中から再開可能
- 入力: `gijiroku_all.db`（sessions/bills/utterances 構造）
- 前提: `wrangler` CLIがインストールされていること

### `schema.sql`
Cloudflare D1（および互換SQLite）用のスキーマ定義。

**テーブル:**
| テーブル | 内容 |
|---------|------|
| sessions | 会期情報（開閉会日・種別・年度・議員期） |
| bills | 議案情報（番号・タイトル・提案者・結果） |
| utterances | 発言情報（話者・役職・党派・発言種別・内容） |

---

## セットアップ

```bash
pip install pymupdf fugashi unidic-lite requests beautifulsoup4 openai
```

OpenAI APIを使う場合:
```bash
export OPENAI_API_KEY="sk-..."
```

---

## 実行手順（RAGパイプライン）

```bash
# プロジェクトルートで実行
python3 0_download.py
python3 1_extract_text.py
python3 2_extract_features.py
python3 3_upload_vectorstore.py
python3 4_split_sessions.py
python3 5_upload_sessions.py

# 発話者分析（別途実行）
python3 analyze_speakers.py
```
