# syntax=docker/dockerfile:1

###
FROM ubuntu:22.04 AS dind

# https://docs.docker.com/engine/install/ubuntu/
RUN apt-get update && apt-get install -y \
	ca-certificates \
	curl \
	&& install -m 0755 -d /etc/apt/keyrings \
	&& curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc \
	&& chmod a+r /etc/apt/keyrings/docker.asc \
	&& echo \
		"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
		$(. /etc/os-release && echo "$VERSION_CODENAME") stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null

RUN apt-get update && apt-get install -y \
	docker-ce \
	docker-ce-cli \
	containerd.io \
	docker-buildx-plugin \
	docker-compose-plugin \
	&& rm -rf /var/lib/apt/lists/* \
	&& apt-get clean \
	&& apt-get autoremove -y

###
FROM dind AS openhands

ENV DEBIAN_FRONTEND=noninteractive

#
RUN apt-get update && apt-get install -y \
	bash \
    build-essential \
    curl \
	git \
	git-lfs \
	software-properties-common \
	make \
    netcat \
    sudo \
	wget \
	&& rm -rf /var/lib/apt/lists/* \
	&& apt-get clean \
	&& apt-get autoremove -y

# https://github.com/cli/cli/blob/trunk/docs/install_linux.md
RUN curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg | dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg \
	&& chmod go+r /usr/share/keyrings/githubcli-archive-keyring.gpg \
	&& echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | tee /etc/apt/sources.list.d/github-cli.list > /dev/null \
	&& apt-get update && apt-get -y install \
    gh \
  && rm -rf /var/lib/apt/lists/* \
  && apt-get clean \
  && apt-get autoremove -y

# Python 3.12
RUN add-apt-repository ppa:deadsnakes/ppa \
    && apt-get update \
    && apt-get install -y python3.12 python3.12-venv python3.12-dev python3-pip \
    && ln -s /usr/bin/python3.12 /usr/bin/python

# NodeJS >= 22.0
RUN curl -fsSL https://deb.nodesource.com/setup_22.x | bash - \
    && apt-get install -y nodejs

# Poetry >= 1.8
RUN curl -fsSL https://install.python-poetry.org | python3.12 - \
    && ln -s ~/.local/bin/poetry /usr/local/bin/poetry

# Google cloud sdk
RUN curl -sSL https://sdk.cloud.google.com | bash

ENV PATH $PATH:/root/google-cloud-sdk/bin

#
RUN <<EOF
#!/bin/bash
printf "#!/bin/bash
set +x
uname -a
docker --version
gh --version | head -n 1
git --version
#
python --version
echo node `node --version`
echo npm `npm --version`
poetry --version
netcat -h 2>&1 | head -n 1
" > /version.sh
chmod a+x /version.sh
EOF

ENV SANDBOX_RUNTIME_CONTAINER_IMAGE=ghcr.io/all-hands-ai/runtime:0.31-nikolaik
ENV SANDBOX_API_HOSTNAME=host.docker.internal

RUN git clone https://github.com/kshivvy/OpenHands.git /app && cd /app && git checkout main

ENV HOME=/app

WORKDIR /app

# cache build dependencies
RUN <<EOF
#!/bin/bash
make -s clean
make -s check-dependencies
make -s install-python-dependencies
EOF

RUN <<EOF
#!/bin/bash
printf "#!/bin/bash
echo '--- [Cloud Batch Task Start - User Cred Workaround] ---'
echo '1. Fetching user credentials from Secret Manager...'
gcloud secrets versions access latest --secret=\"\$USER_ADC_SECRET_NAME\" --project=\"docker-rlef-exploration\" > /tmp/user_adc.json
chmod 600 /tmp/user_adc.json # Good practice for credential files
echo '2. Setting GOOGLE_APPLICATION_CREDENTIALS environment variable...'
export GOOGLE_APPLICATION_CREDENTIALS=/tmp/user_adc.json
echo '3. Verifying authentication (should show YOUR user account)...'
gcloud auth list
# Capture the active account WITHIN the container's shell
ACTIVE_ACCOUNT=\$(gcloud config get-value account)
echo \"Active Account for Task: \${ACTIVE_ACCOUNT}\"

export DEBUG=1; export DEBUG_LLM=0; export LOG_ALL_EVENTS=1; export LOG_TO_FILE=1;

BACKEND_HOST='0.0.0.0'
SANDBOX_USER_ID=\$(id -u)
MODEL=\${MODEL:-'gemini/gemini-1.5-flash-002'}
SELECTED_IDS=\${SELECTED_IDS:-\"['sphinx-doc__sphinx-8721']\"}
printf \"selected_ids = \${SELECTED_IDS}\n\" > /app/evaluation/benchmarks/swe_bench/config.toml
# Create LLM config.
printf \"[llm.gemini]\n model = '\${MODEL}'\n api_key = '\${GEMINI_API_KEY}'\n temperature = 1.0\n top_p = 0.95\n max_output_tokens = 65536\n caching_prompt = false\n num_retries = 100\n timeout = 600\n seed = 42\n\" > /app/config.toml
# Run evaluation.
# See README at https://github.com/All-Hands-AI/OpenHands/tree/main/evaluation/benchmarks/swe_bench
# [model_config] [git-version] [agent] [eval_limit] [max_iter] [num_workers] [dataset] [dataset_split]
/app/evaluation/benchmarks/swe_bench/scripts/run_infer.sh llm.gemini HEAD CodeActAgent 500 100 1 princeton-nlp/SWE-bench_Verified test
" > /app/entrypoint.sh
chmod a+x /app/entrypoint.sh
EOF

ENTRYPOINT "/app/entrypoint.sh"