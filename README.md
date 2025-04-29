# mcp-rag-demo

## 実現したいこと

1. ローカル環境でRAGを構築しMCPで参照できるようにする
2. RAGにプロジェクト固有の情報を溜め込む（チーム固有のルールや規約、利用する技術の公式ドキュメント情報など）
3. Claude等のツールからMCPを経由して情報を呼び出し、生成されるコードに改善が見られるか検証する

### Ollama モデルの準備

RAG システムで使用する埋め込みモデル (`bge-m3`) を Ollama で利用可能にします。

```bash
ollama run bge-m3
```
(初回はモデルのダウンロードが実行されます)

### 5. (オプション) 環境変数による設定

API サーバーは環境変数で設定を上書きできます。必要に応じて `.env` ファイルを作成するか、環境変数を設定してください。主な設定項目は以下の通りです (デフォルト値は `rag_api_server/config.py` 参照)。

*   `OLLAMA_BASE_URL`: Ollama サーバーの URL (デフォルト: `http://localhost:11434`)
*   `EMBEDDING_MODEL_NAME`: 使用する埋め込みモデル名 (デフォルト: `bge-m3`)
*   `DB_PATH`: DuckDB データベースファイルのパス (デフォルト: `vector_store.db`)
*   `TABLE_NAME`: ベクトルストアのテーブル名 (デフォルト: `embeddings`)

## 利用可能な機能 (API エンドポイント)

API サーバーは以下のエンドポイントを提供します。Swagger UI (`http://127.0.0.1:8001/docs`) からも確認・試用できます。

*   **`GET /`**:
    *   **説明:** API サーバーのヘルスチェック。
    *   **レスポンス:** `{"status": "healthy", "version": "サーバーバージョン"}`
*   **`POST /documents/`**:
    *   **説明:** 指定されたディレクトリ内のドキュメント (.txt, .md) を処理し、チャンク分割、埋め込みベクトル生成を行い、ベクトルデータベースに保存します。
    *   **リクエストボディ:** `{"source_path": "処理対象ディレクトリのパス", "glob_pattern": "ファイルパターン (オプション, デフォルト: **/*[.md|.txt])"}`
    *   **レスポンス (成功時):** `{"status": "success", "processed_documents": 数, "processed_chunks": 数, "message": "..."}`
    *   **レスポンス (失敗時):** `{"detail": "エラーメッセージ"}`
*   **`POST /search/`**:
    *   **説明:** 指定されたクエリに基づいて、ベクトルデータベースから類似度の高いドキュメントチャンクを検索します。
    *   **リクエストボディ:** `{"query": "検索クエリ文字列", "top_k": 取得件数 (オプション, デフォルト: 5)}`
    *   **レスポンス (成功時):** `{"results": [{"text": "チャンクテキスト", "similarity": 類似度スコア}, ...]}`
    *   **レスポンス (失敗時):** `{"detail": "エラーメッセージ"}`

## 使い方 (curl 例)

### ヘルスチェック

```bash
curl http://127.0.0.1:8001/
```

### ドキュメントの登録

プロジェクト内の `docs` ディレクトリを登録する場合:

```bash
curl -X POST -H "Content-Type: application/json" \
     -d '{"source_path": "docs"}' \
     http://127.0.0.1:8001/documents/
```

### ドキュメントの検索

"DuckDB" というクエリで上位 3 件を検索する場合:

```bash
curl -X POST -H "Content-Type: application/json" \
     -d '{"query": "DuckDB", "top_k": 3}' \
     http://127.0.0.1:8001/search/
```

## 🗂️ プロジェクト構成

```text
mcp-rag/
├─ pyproject.toml               # ルート: uv ワークスペース定義
├─ src/
│  ├─ rag_api_server/           # FastAPI エンドポイント
│  │  ├─ pyproject.toml         #   src-layout / editable
│  │  └─ rag_api_server/…       #   実装
│  ├─ rag_core/                 # RAG ロジック（Embedding, VectorDB）
│  │  ├─ pyproject.toml
│  │  └─ rag_core/…
│  └─ mcp_adapter/              # (オプション) 外部 MCP プロトコルアダプタ
│     └─ …
└─ README.md                    # ← いま見ているファイル
```

* 各サブプロジェクトは **“src/パッケージ名” レイアウト**  
  （`tool.setuptools.package-dir` は省略し、`packages = ["rag_api_server"]` などで指定）
* ルート `pyproject.toml` は **ワークスペース管理専用**  
  ```toml
  [tool.uv]
  package = false                       # ルート自体は wheel 化しない

  [tool.uv.workspace]
  members = [
    "src/rag_core",
    "src/rag_api_server",
    "src/mcp_adapter",
  ]
  # workspace = true で各メンバーを editable install
  ```

---

## 🚀 クイックスタート

### 1. 前提

| Tool | Version |
|------|---------|
| Python | **3.11 以上 (CPython)** |
| uv (pipx 推奨) | `>=0.6.17` |
| DuckDB 1.2.x / Ollama 0.1.x | *rag_core を使う場合のみ* |

```bash
pipx install "uv>=0.6.17,<0.7"
```

### 2. 依存インストール

```bash
# ルートで実行
uv sync
```

* `.venv/` が自動生成され、  
  - `rag-core`, `rag-api-server`, `mcp-adapter` が **editable** で入る  
  - 依存（LangChain, duckdb, torch など）が **binary wheel** で入る
* 依存を追加したい場合は各サブプロジェクトの `pyproject.toml` に追記し  
  `uv sync` を再実行するだけ。

### 3. 開発サーバを起動

```bash
uv run uvicorn rag_api_server.main:app --reload
```

* `--reload` 付きでも **子プロセスが editable パスを解決** できる構成。  
  （multiprocessing spawn 対策済み）
* 既定で <http://127.0.0.1:8000/docs> に Swagger UI が立ち上がります。

> **Tips**  
> - `uv run -p 9000 …` でポート変更  
> - `.env` に `RAG__EMBEDDING_MODEL=all-minilm` など環境変数を置くと  
>   `rag_core` 側で `pydantic-settings` が自動読み込み

---

## ⚙️ 開発時のコマンドメモ

| タスク | コマンド |
|--------|---------|
| 依存追加 | `uv pip install -e src/rag_core[dev]`<br>（あるいは `pyproject.toml` 編集後 `uv sync`） |
| Lint / Format | `uv run ruff check .` , `uv run ruff format .` |
| テスト | `uv run pytest` |
| 仮想環境を捨てる | `rm -rf .venv uv.lock` |

---

## 📝 よくあるハマりポイント

| 症状 | 原因と対処 |
|------|-----------|
| `ModuleNotFoundError: rag_core` | サブプロジェクト側 `package-dir` の指定ミス<br>→ `packages = ["rag_core"]` だけにする |
| `ModuleNotFoundError: langchain_ollama` など | LangChain 0.3 系から **インテグレーションが分離**<br>→ `langchain-ollama`, `langchain-openai` などを `dependencies` へ追加 |
| リロード時だけ ImportError | `uv run --package` を使わず **依存として明示** (ルート `dependencies = ["rag_api_server"]`) |

---

> **Why uv?**  
> - `pip + venv` より **20–30× 高速** な解決＆インストール  
> - ワークスペースを **PEP 621 準拠**のまま editable 管理  
> - `uv pip`, `uv run`, `uv venv` で day-to-day が完結

これで新しくクローンした開発者でも  
```bash
git clone https://github.com/yourname/mcp-rag
cd mcp-rag
uv sync
uv run uvicorn rag_api_server.main:app --reload
```  
だけで環境構築～サーバ起動ができます。