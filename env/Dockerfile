# syntax=docker/dockerfile:1.5
FROM pytorch/pytorch:2.3.0-cuda12.1-cudnn8-runtime

# ───────────── 1. ENV 共通設定 ─────────────
ENV DEBIAN_FRONTEND=noninteractive \
    PIP_NO_CACHE_DIR=1 \
    PIP_DEFAULT_TIMEOUT=1200 \
    PIP_RETRIES=20 \
    PIP_PROGRESS_BAR=off \
    PYTHONUNBUFFERED=1

# config ファイル名（拡張子なしで指定）
ENV CFG=config

# ───────────── 2. OS パッケージ ─────────────
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        git build-essential curl cmake automake libtool zlib1g-dev && \
    rm -rf /var/lib/apt/lists/*

# ───────────── 3. llm‑leaderboard ─────────────
WORKDIR /workspace
RUN git clone --depth 1 https://github.com/wandb/llm-leaderboard.git
WORKDIR /workspace/llm-leaderboard

# ───────────── 4‑A. 依存ライブラリ ─────────────
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --upgrade pip && \
    pip install --extra-index-url https://download.pytorch.org/whl/cu124 \
      -r requirements.txt

# ───────────── 4‑B. langchain だけ別レイヤー ─────────────
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install langchain-community==0.2.17 langchain-cohere==0.2.0

# ───────────── 5. Juman++ ─────────────
RUN set -eux && \
    git clone --depth 1 https://github.com/ku-nlp/jumanpp.git /tmp/jumanpp && \
    cmake -S /tmp/jumanpp -B /tmp/jumanpp/build -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=/usr/local && \
    cmake --build /tmp/jumanpp/build -j"$(nproc)" && \
    cmake --install /tmp/jumanpp/build && \
    rm -rf /tmp/jumanpp && ldconfig

# ───────────── 6. ENTRYPOINT ─────────────
ENTRYPOINT ["/bin/sh", "-c", "python3 scripts/run_eval.py -c ${CFG}"]
