# LLM ベンチマーク環境（llm‑benchmark‑docker）

日本語 LLM を **vLLM + llm‑leaderboard** で再現評価するための Docker 構成です。

- 推論エンジン : vLLM OpenAI 互換サーバー（GPU）  
- 評価エンジン : llm‑leaderboard (`wandb/llm-leaderboard`)（GPU）
- **完全コンテナ化** → 依存衝突を極力低減し環境再現性を向上

---

## 1 . 必要環境

| 項目 | 必要バージョン | 確認コマンド |
|------|---------------|--------------|
| NVIDIA Driver | 525 以降 | `nvidia-smi` |
| CUDA 対応 Docker | 24.0 以降 | `docker -v` |
| nvidia‑container‑toolkit | 最新 | `docker info | jq '.Runtimes.nvidia'` |
| Git / curl 等 | – | – |

> **補足** : 公式ドキュメントに沿って `nvidia-container-toolkit` を導入し、Docker デーモンを再起動してください。
```bash
> distribution=$(. /etc/os-release; echo ${ID}${VERSION_ID})
> curl -fsSL https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.list | \
>   sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
> sudo apt-get update && sudo apt-get install -y nvidia-container-toolkit nvidia-container-runtime
> sudo systemctl restart docker
> ```

---

## 2 . このリポジトリを取得

```bash
# ホストで
$ git clone https://github.com/nejumi/llm-benchmark-docker.git
$ cd llm-benchmark-docker
```

```
llm-benchmark-docker/
├─ docker-compose.yml   # サービス定義
├─ Dockerfile.eval      # 評価エンジン用
├─ config/              # 各モデルごとの YAML を置く
└─ README.md            # ← いま読んでいるファイル
```

---

## 3 . モデルの指定方法

### 3‑A : Hugging Face 上のモデルを直接使う

1. **Hugging Face アクセストークン**を `.env` または環境変数で渡します。
   ```bash
   echo "HUGGINGFACE_HUB_TOKEN=hf_xxxxxxxxx" > .env
   ```
2. `docker-compose.yml` → `vllm:` サービスの `command:` を編集
   ```yaml
   --model nejumi/Llama-3.1-Swallow-70B-Instruct-v0.3-GPTQ-Int4-calib-ja-1k
   ```
3. 起動時に vLLM が自動でモデルをダウンロードします。

### 3‑B : **ローカルに既に重みがある** 場合

1. 重みが置いてあるディレクトリを把握（例 `/data/models/swallow-70b/`）。  
   `…/snapshots/<commit_hash>/` など *config.json がある階層* を指します。
2. Compose にボリュームマウントを追加
   ```yaml
   volumes:
     - /data/models/swallow-70b:/models/swallow-70b:ro
   ```
3. `command:` では **フルパスを渡す**
   ```yaml
   --model /models/swallow-70b
   ```
   > オフライン環境でも動作します。

### 3‑C : 例の Swallow 以外のモデルを使うには？

| 変更箇所 | 内容 |
|-----------|------|
| `vllm.command` | `--model <repo_id or /local/path>`<br>`--served-model-name <表示名>` |
| `config/<your>.yaml` | `wandb.run_name` `model.pretrained_model_name_or_path` を同じ名前に |
| GPU 数 | `--tensor-parallel-size` と `CUDA_VISIBLE_DEVICES` を合わせる |

> **GPTQ / AWQ / GGUF** など量子化フォーマットは vLLM がサポートしていればそのまま使えます。

---

## 4 . イメージのビルド

```bash
# 初回のみ（数十分）
$ docker compose build --pull
```

- **評価エンジン**: `llm-benchmark-docker-eval`  
- **vLLM サーバ**: 公式イメージ (`vllm/vllm-openai:latest`)

キャッシュが効くので 2 回目以降は数十秒です。

---

## 5 . サービスの起動

```bash
# モデルを先にロード
$ docker compose up -d vllm
$ docker compose logs -f vllm   # "Uvicorn running" を確認

# ベンチマーク開始
$ docker compose up -d eval
$ docker compose logs -f eval   # 進捗を監視
```

完了すると **Weights & Biases** に run がアップロードされます。

---

## 6 . config.yaml のポイント

```yaml
api: vllm-external           # ← サーバ経由で評価
base_url: "http://vllm:8000/v1"  # Compose のサービス名
num_gpus: 2                  # vLLM と合わせる
batch_size: 128              # GPU メモリに応じて調整
```

- **`wandb:`** セクションの `project` や `entity` を自分の値に変更
- 日本語タスクを試す場合は `tasks:` リストを編集

---

## 7 . よくあるトラブル

| 症状 | 解決策 |
|------|--------|
| `nvidia runtime unknown` | 1) nvidia‑container‑toolkit を入れる<br>2) `/etc/docker/daemon.json` で `default-runtime"nvidia"` |
| `HFValidationError` | ローカルパスを渡すときは **必ずボリュームマウント** する |
| CUDA OOM | `batch_size` を半分に / `tensor-parallel-size` を増やす |
| WANDB にログが出ない | `.env` の API キー、Outbound port 443 を確認 |

---

## 8 . クリーンアップ

```bash
# サービス停止
$ docker compose down

# 未使用イメージ・キャッシュ削除（任意）
$ docker system prune -af
```
