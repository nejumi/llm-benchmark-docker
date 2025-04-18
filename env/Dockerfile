# syntax=docker/dockerfile:1.5
# ───────────────────────────────────────────────
# 基盤: CUDA‑12.1 + PyTorch‑2.3 公式ランタイム
# ───────────────────────────────────────────────
FROM pytorch/pytorch:2.3.0-cuda12.1-cudnn8-runtime

# ──────────────── 1. ENV ────────────────
ENV DEBIAN_FRONTEND=noninteractive \
    PIP_NO_CACHE_DIR=1 \
    PIP_DEFAULT_TIMEOUT=1200 \
    PIP_RETRIES=20 \
    PIP_PROGRESS_BAR=off \
    PYTHONUNBUFFERED=1

# run_eval.py に渡す **拡張子なし** の設定ファイル名
ENV CFG=my_config

# ──────────────── 2. OS パッケージ ────────────────
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        git build-essential curl cmake automake libtool zlib1g-dev && \
    rm -rf /var/lib/apt/lists/*

# ──────────────── 3. ソース取得 ────────────────
WORKDIR /workspace
RUN git clone --depth 1 https://github.com/wandb/llm-leaderboard.git
WORKDIR /workspace/llm-leaderboard

# ──────────────── 4. Python 依存 (2 段) ────────────────
# 4‑A requirements.txt（pytorch CUDA 版 index を追加）
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --upgrade pip && \
    pip install \
      --extra-index-url https://download.pytorch.org/whl/cu124 \
      -r requirements.txt

# 4‑B langchain 系はバージョン衝突回避のため別 layer
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install \
      langchain-community==0.2.17 \
      langchain-cohere==0.2.0

# ──────────────── 5. Juman++ ────────────────
RUN bash scripts/install_jumanpp.sh

# ──────────────── 6. ローカル config をイメージに取り込む ────────────────
# ※ bind‑mount 運用なら丸ごと削除して OK
COPY ./config/*.yaml /workspace/llm-leaderboard/configs/

# ──────────────── 7. 実行 ────────────────
# bash 経由で ENV を展開し、-c に拡張子無の CFG を渡す
ENTRYPOINT ["bash", "-c", "python scripts/run_eval.py -c ${CFG}"]
