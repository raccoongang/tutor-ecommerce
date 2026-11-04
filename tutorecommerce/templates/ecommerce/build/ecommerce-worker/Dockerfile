# syntax=docker/dockerfile:1.4
###### Minimal image with base system requirements for most stages
FROM docker.io/ubuntu:20.04 AS minimal

RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    sed -i 's|http://archive.ubuntu.com|https://archive.ubuntu.com|g; s|http://security.ubuntu.com|https://security.ubuntu.com|g' /etc/apt/sources.list && \
    apt -o Acquire::https::Verify-Peer=false update && \
    apt -o Acquire::https::Verify-Peer=false install -y ca-certificates && \
    apt update && \
    apt install -y curl git-core language-pack-en libmysqlclient-dev \
    libssl-dev libffi-dev build-essential

# Newer Ubuntu releases distrust the emSign ECC Root CA - G3 over the compromised eMudhra
# intermediates (LP: #2171507), but 20.04 is out of standard support and its ca-certificates
# still trusts it, so deselect it the same way. https://emudhra.we-have-a-problem.com/
RUN sed -i 's|^mozilla/emSign_ECC_Root_CA_-_G3\.crt$|!&|' /etc/ca-certificates.conf && \
    grep -qx '!mozilla/emSign_ECC_Root_CA_-_G3.crt' /etc/ca-certificates.conf && \
    update-ca-certificates

# Constrain the setuptools that pip uses in isolated (PEP 517) build environments; otherwise
# the latest one is used, which no longer ships pkg_resources. pip 26 no longer applies
# PIP_CONSTRAINT to build dependencies, hence PIP_BUILD_CONSTRAINT too.
# https://github.com/overhangio/tutor/issues/1350
RUN echo "setuptools==69.1.1" > /etc/ecommerce-build-constraints.txt
ENV PIP_CONSTRAINT=/etc/ecommerce-build-constraints.txt
ENV PIP_BUILD_CONSTRAINT=/etc/ecommerce-build-constraints.txt

ARG APP_USER_ID=1000
RUN useradd --home-dir /openedx --create-home --shell /bin/bash --uid ${APP_USER_ID} app
USER ${APP_USER_ID}

RUN mkdir /openedx/ecommerce_worker && \
    git clone https://github.com/raccoongang/ecommerce-worker.git --branch release/teak --depth 1 /openedx/ecommerce_worker
WORKDIR /openedx/ecommerce_worker

# Install pyenv
# https://www.python.org/downloads/
# https://github.com/pyenv/pyenv/releases
ARG PYTHON_VERSION=3.12.2
ENV PYENV_ROOT=/opt/pyenv
# root user is required for below 2 steps, as app user gets permission denied.
USER root
RUN git clone https://github.com/pyenv/pyenv $PYENV_ROOT --branch v2.3.36 --depth 1
# Install Python
RUN $PYENV_ROOT/bin/pyenv install $PYTHON_VERSION
USER app

# Create virtualenv
RUN $PYENV_ROOT/versions/$PYTHON_VERSION/bin/python -m venv ../venv/
ENV PATH=/openedx/venv/bin:$PATH
RUN --mount=type=cache,target=/openedx/.cache/pip,sharing=shared pip install \
    # https://pypi.org/project/setuptools/
    # https://pypi.org/project/pip/
    # https://pypi.org/project/wheel/
    setuptools==69.1.1 pip==26.2.2 wheel==0.43.0
RUN --mount=type=cache,target=/openedx/.cache/pip,sharing=shared pip install -r requirements/production.txt

# Upgrade certifi until the ecommerce-worker requirements pin a version without the compromised emSign roots
# https://emudhra.we-have-a-problem.com/
RUN --mount=type=cache,target=/openedx/.cache/pip,sharing=shared pip install "certifi>=2026.10.19"

ENV WORKER_CONFIGURATION_MODULE=ecommerce_worker.settings.production
CMD celery --app=ecommerce_worker.celery_app:app worker --loglevel=info --max-tasks-per-child=100 --queues=fulfillment,email_marketing
