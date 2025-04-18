# syntax=docker/dockerfile:1.5
# ───────────────────────────────────────────────
# 1. CUDA 12.1 + PyTorch 2.3 ランタイム
# ───────────────────────────────────────────────
FROM pytorch/pytorch:2.3.0-cuda12.1-cudnn8-runtime

# ───────────────────────────────────────────────
# 2. 変数
#    * CFG は run_eval.py へ渡す設定ファイル名（拡張子なし）
#      → docker‑compose.yml で上書きできます
# ───────────────────────────────────────────────
ARG  CFG=my_config          # `docker build --build‑arg CFG=foo …`
ENV  DEBIAN_FRONTEND=noninteractive \
     PIP_NO_CACHE_DIR=1 \
     PIP_DEFAULT_TIMEOUT=1200 \
     PIP_RETRIES=20 \
     PIP_PROGRESS_BAR=off \
     PYTHONUNBUFFERED=1 \
     CFG=${CFG}

# ───────────────────────────────────────────────
# 3. Linux パッケージ
# ───────────────────────────────────────────────
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        git build-essential curl cmake automake libtool zlib1g-dev && \
    rm -rf /var/lib/apt/lists/*

# ───────────────────────────────────────────────
# 4. llm‑leaderboard 取得
# ───────────────────────────────────────────────
WORKDIR /workspace
RUN git clone --depth 1 https://github.com/wandb/llm-leaderboard.git
WORKDIR /workspace/llm-leaderboard

# ───────────────────────────────────────────────
# 5. Python 依存ライブラリ（2 レイヤー）
# ───────────────────────────────────────────────
# 5‑A: requirements.txt
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --upgrade pip && \
    pip install \
      --extra-index-url https://download.pytorch.org/whl/cu124 \
      -r requirements.txt

# 5‑B: langchain 系だけ別レイヤー
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install \
      langchain-community==0.2.17 \
      langchain-cohere==0.2.0

# ───────────────────────────────────────────────
# 6. Juman++ を root でビルド（sudo 不要）
# ───────────────────────────────────────────────
RUN set -eux; \
    git clone --depth 1 https://github.com/ku-nlp/jumanpp.git /tmp/jumanpp; \
    mkdir -p /tmp/jumanpp/build; \
    cd       /tmp/jumanpp/build; \
    cmake .. -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=/usr/local; \
    make -j"$(nproc)"; make install; ldconfig; \
    rm -rf /tmp/jumanpp

# ───────────────────────────────────────────────
# 7. ENTRYPOINT
#    -c には拡張子を付けない  (run_eval.py が .yaml を補完)
# ───────────────────────────────────────────────
ENTRYPOINT ["bash", "-c", "python llm-leaderboard/scripts/run_eval.py -c ${CFG}"]
