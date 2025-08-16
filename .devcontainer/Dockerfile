FROM ubuntu:22.04

# Установка базовых пакетов
RUN apt-get update && apt-get install -y \
    software-properties-common \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release \
    git \
    vim \
    nano \
    tree \
    jq \
    htop \
    wget \
    unzip \
    ssh \
    sshpass \
    iputils-ping \
    net-tools \
    telnet \
    netcat \
    python3 \
    python3-pip \
    python3-venv \
    python3-setuptools \
    && rm -rf /var/lib/apt/lists/*

# Установка Ansible и дополнительных инструментов
RUN pip3 install --no-cache-dir \
    ansible>=6.0.0 \
    ansible-lint \
    molecule \
    molecule-docker \
    yamllint \
    jinja2-cli \
    netaddr \
    kubernetes \
    openshift \
    requests \
    urllib3

# Установка Ansible коллекций
RUN ansible-galaxy collection install \
    kubernetes.core \
    community.general \
    ansible.posix \
    community.docker

# Установка kubectl
RUN curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" \
    && chmod +x kubectl \
    && mv kubectl /usr/local/bin/

# Установка helm
RUN curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Установка Docker CLI
RUN curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg \
    && echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null \
    && apt-get update \
    && apt-get install -y docker-ce-cli \
    && rm -rf /var/lib/apt/lists/*

# Установка kind для локального тестирования
RUN curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64 \
    && chmod +x ./kind \
    && mv ./kind /usr/local/bin/kind

# Создание пользователя
RUN useradd -m -s /bin/bash vscode \
    && usermod -aG sudo vscode \
    && echo 'vscode ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers

# Настройка SSH для пользователя
USER vscode
RUN mkdir -p /home/vscode/.ssh \
    && chmod 700 /home/vscode/.ssh \
    && ssh-keygen -t rsa -b 4096 -f /home/vscode/.ssh/id_rsa -N "" \
    && chmod 600 /home/vscode/.ssh/id_rsa \
    && chmod 644 /home/vscode/.ssh/id_rsa.pub

# Настройка рабочей директории
WORKDIR /workspace

USER vscode
