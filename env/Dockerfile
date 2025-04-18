# syntax=docker/dockerfile:1.5
FROM pytorch/pytorch:2.3.0-cuda12.1-cudnn8-runtime

ARG  CFG=config   # デフォルト値（拡張子なし）
ENV  DEBIAN_FRONTEND=noninteractive \
     PIP_NO_CACHE_DIR=1 \
     PIP_DEFAULT_TIMEOUT=1200 \
     PIP_RETRIES=20 \
     PIP_PROGRESS_BAR=off \
     PYTHONUNBUFFERED=1 \
     CFG=${CFG}

# 1. OS パッケージ
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        git build-essential curl cmake automake libtool zlib1g-dev sudo && \
    rm -rf /var/lib/apt/lists/*

# 2. llm‑leaderboard 取得
WORKDIR /workspace
RUN git clone --depth 1 https://github.com/wandb/llm-leaderboard.git
WORKDIR /workspace/llm-leaderboard

# 3. Python 依存 (2 ステップ)
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --upgrade pip && \
    pip install \
        --extra-index-url https://download.pytorch.org/whl/cu124 \
        -r requirements.txt
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install \
        langchain-community==0.2.17 \
        langchain-cohere==0.2.0

# 4. Juman++
RUN bash scripts/install_jumanpp.sh

# Juman++ を評価スクリプトから確実に見えるように
ENV PATH="/usr/local/bin:${PATH}" \
    JUMANPP_COMMAND=/usr/local/bin/jumanpp \
    JUMANPP_DICDIR=/usr/local/share/jumandic

# 5. ENTRYPOINT（シェル経由で変数展開）
ENTRYPOINT ["/bin/sh", "-c", "python scripts/run_eval.py -c ${CFG}"]
