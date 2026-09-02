# base image
FROM python:3.12-slim-bookworm@sha256:782412e85d0f0984994c290652577d4018aff08145c85b262bb63dc0c7522254 AS base

WORKDIR /app/api

# Install uv
ENV UV_VERSION=0.8.9

RUN pip install --no-cache-dir uv==${UV_VERSION}


FROM base AS packages

# if you located in China, you can use aliyun mirror to speed up
# RUN sed -i 's@deb.debian.org@mirrors.aliyun.com@g' /etc/apt/sources.list.d/debian.sources

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
          # basic environment
          git g++ \
          # for building gmpy2
          libmpfr-dev libmpc-dev

# Install Python dependencies (workspace members under providers/vdb/)
COPY api/pyproject.toml api/uv.lock ./
COPY api/providers ./providers
COPY dify-agent/pyproject.toml dify-agent/README.md /app/dify-agent/
COPY dify-agent/src /app/dify-agent/src
# Trust the checked-in lock during image builds; local path sources are copied from the repository context.
RUN uv sync --frozen --no-dev --no-editable

# production stage
FROM base AS production

ENV FLASK_APP=app.py
ENV DEPLOY_ENV=PRODUCTION
ENV CONSOLE_API_URL=http://127.0.0.1:5001
ENV CONSOLE_WEB_URL=http://127.0.0.1:3000
ENV SERVICE_API_URL=http://127.0.0.1:5001
ENV APP_WEB_URL=http://127.0.0.1:3000

EXPOSE 5001

# set timezone
ENV TZ=UTC

# Set UTF-8 locale
ENV LANG=en_US.UTF-8
ENV LC_ALL=en_US.UTF-8
ENV PYTHONIOENCODING=utf-8

WORKDIR /app/api

# Create non-root user
ARG dify_uid=1001
ARG NODE_MAJOR=22
ARG NODE_PACKAGE_VERSION=22.21.0-1nodesource1
ARG NODESOURCE_KEY_FPR=6F71F525282841EEDAF851B42F59B5F99B1BE0B4
ARG NLTK_DATA_COMMIT=550b6625bcef1f2abff2ff770a5a0d272c9c6b2a
ARG NLTK_PUNKT_TAB_SHA256=e57f64187974277726a3417ca6f181ec5403676c717672eef6a748a7b20e0106
ARG NLTK_TAGGER_SHA256=6025f530624335c67d6547d44757b357b4e79bae030a0383e9887a92c1718f0b
ARG NLTK_STOPWORDS_SHA256=48c0e52d8b52546e827f53761fb30300c0ab94f70660d28bd65ba0a86270946b
RUN groupadd -r -g ${dify_uid} dify && \
    useradd -r -u ${dify_uid} -g ${dify_uid} -s /bin/bash dify && \
    chown -R dify:dify /app

RUN \
    apt-get update \
    && apt-get install -y --no-install-recommends \
        ca-certificates \
        curl \
        gnupg \
    && mkdir -p /etc/apt/keyrings \
    && curl --retry 5 --retry-all-errors --connect-timeout 20 -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key -o /tmp/nodesource.gpg \
    && gpg --show-keys --with-colons /tmp/nodesource.gpg \
        | awk -F: '/^fpr:/ {print $10}' \
        | grep -Fx "${NODESOURCE_KEY_FPR}" \
    && gpg --dearmor -o /etc/apt/keyrings/nodesource.gpg /tmp/nodesource.gpg \
    && rm -f /tmp/nodesource.gpg \
    && echo "deb [signed-by=/etc/apt/keyrings/nodesource.gpg] https://deb.nodesource.com/node_${NODE_MAJOR}.x nodistro main" \
        > /etc/apt/sources.list.d/nodesource.list \
    && apt-get update \
    # Install dependencies
    && apt-get install -y --no-install-recommends \
        # basic environment
        nodejs=${NODE_PACKAGE_VERSION} \
        # for gmpy2 \
        libgmp-dev libmpfr-dev libmpc-dev \
        # For Security
        expat libldap-2.5-0=2.5.13+dfsg-5 perl libsqlite3-0=3.40.1-2+deb12u2 zlib1g=1:1.2.13.dfsg-1 \
        # install fonts to support the use of tools like pypdfium2
        fonts-noto-cjk \
        # install a package to improve the accuracy of guessing mime type and file extension
        media-types \
        # install libmagic to support the use of python-magic guess MIMETYPE
        libmagic1 \
    && apt-get autoremove -y \
    && rm -rf /var/lib/apt/lists/*

# Copy Python environment and packages
ENV VIRTUAL_ENV=/app/api/.venv
COPY --from=packages --chown=dify:dify ${VIRTUAL_ENV} ${VIRTUAL_ENV}
ENV PATH="${VIRTUAL_ENV}/bin:${PATH}"

RUN mkdir -p /usr/local/share/nltk_data/tokenizers /usr/local/share/nltk_data/taggers /usr/local/share/nltk_data/corpora \
    && curl --retry 5 --retry-all-errors --connect-timeout 20 -fsSL \
        "https://raw.githubusercontent.com/nltk/nltk_data/${NLTK_DATA_COMMIT}/packages/tokenizers/punkt_tab.zip" -o /tmp/punkt_tab.zip \
    && echo "${NLTK_PUNKT_TAB_SHA256}  /tmp/punkt_tab.zip" | sha256sum --check - \
    && python -m zipfile -e /tmp/punkt_tab.zip /usr/local/share/nltk_data/tokenizers \
    && curl --retry 5 --retry-all-errors --connect-timeout 20 -fsSL \
        "https://raw.githubusercontent.com/nltk/nltk_data/${NLTK_DATA_COMMIT}/packages/taggers/averaged_perceptron_tagger_eng.zip" -o /tmp/averaged_perceptron_tagger_eng.zip \
    && echo "${NLTK_TAGGER_SHA256}  /tmp/averaged_perceptron_tagger_eng.zip" | sha256sum --check - \
    && python -m zipfile -e /tmp/averaged_perceptron_tagger_eng.zip /usr/local/share/nltk_data/taggers \
    && curl --retry 5 --retry-all-errors --connect-timeout 20 -fsSL \
        "https://raw.githubusercontent.com/nltk/nltk_data/${NLTK_DATA_COMMIT}/packages/corpora/stopwords.zip" -o /tmp/stopwords.zip \
    && echo "${NLTK_STOPWORDS_SHA256}  /tmp/stopwords.zip" | sha256sum --check - \
    && python -m zipfile -e /tmp/stopwords.zip /usr/local/share/nltk_data/corpora \
    && rm -f /tmp/punkt_tab.zip /tmp/averaged_perceptron_tagger_eng.zip /tmp/stopwords.zip \
    && NLTK_DATA=/usr/local/share/nltk_data python -c "import nltk; nltk.data.find('tokenizers/punkt_tab'); nltk.data.find('taggers/averaged_perceptron_tagger_eng'); nltk.data.find('corpora/stopwords')" \
    && chmod -R 755 /usr/local/share/nltk_data

ENV TIKTOKEN_CACHE_DIR=/app/api/.tiktoken_cache

RUN python -c "import tiktoken; tiktoken.encoding_for_model('gpt2')" \
    && chown -R dify:dify ${TIKTOKEN_CACHE_DIR}

# Copy source code
COPY --chown=dify:dify api /app/api/

# Prepare entrypoint script
COPY --chown=dify:dify --chmod=755 api/docker/entrypoint.sh /entrypoint.sh


ARG COMMIT_SHA
ENV COMMIT_SHA=${COMMIT_SHA}
ENV NLTK_DATA=/usr/local/share/nltk_data

USER dify

ENTRYPOINT ["/bin/bash", "/entrypoint.sh"]
