# syntax=docker/dockerfile:1.5
FROM pytorch/pytorch:2.3.0-cuda12.1-cudnn8-runtime

# ========== 共通 ENV ==========
ENV DEBIAN_FRONTEND=noninteractive \
    PIP_NO_CACHE_DIR=1 \
    PIP_DEFAULT_TIMEOUT=1200 \
    PIP_RETRIES=20 \
    PIP_PROGRESS_BAR=off \
    PYTHONUNBUFFERED=1 \
    CFG=config        # ← compose 側で上書き可

# ========== 必要ツール ==========
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        git build-essential curl cmake automake libtool zlib1g-dev && \
    rm -rf /var/lib/apt/lists/*

# ========== llm‑leaderboard ==========
WORKDIR /workspace
RUN git clone --depth 1 https://github.com/wandb/llm-leaderboard.git
WORKDIR /workspace/llm-leaderboard

# ========== Python 依存（2 ステップ） ==========
## 5‑A. requirements.txt
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --upgrade pip && \
    pip install \
      --extra-index-url https://download.pytorch.org/whl/cu124 \
      -r requirements.txt

## 5‑B. langchain 系（依存衝突を避けるため別レイヤ）
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install \
      langchain-community==0.2.17 \
      langchain-cohere==0.2.0

# ========== Juman++ ==========
RUN set -eux && \
    cd /tmp && \
    git clone --depth 1 https://github.com/ku-nlp/jumanpp.git && \
    cd jumanpp && \
    mkdir build && cd build && \
    cmake .. -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=/usr/local && \
    make -j"$(nproc)" && make install && ldconfig && \
    rm -rf /tmp/jumanpp

# ========== エントリポイント ==========
#   run_eval.py は “configs/${CFG}.yaml” を自動で解決する
ENTRYPOINT ["/bin/sh","-c","python3 scripts/run_eval.py -c ${CFG}"]
