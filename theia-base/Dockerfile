# https://github.com/theia-ide/theia-apps/tree/master/theia-cpp-docker

ARG NODE_VERSION=18
FROM node:${NODE_VERSION}-bullseye-slim as base

ARG DIST=bullseye
ENV TZ=America/New_York \
    PUID=1001 \
    GUID=1001 \
    KEYBOARD=en-us-qwerty \
    CUSTOM_PORT=5000

ARG PY_VERSION=3.10.6
ENV PY_VERSION=$PY_VERSION \
    NODE_VERSION=$NODE_VERSION \
    TERM=xterm-256color \
    DEBIAN_FRONTEND=noninteractive \
    REPO_NAME="" \
    ANUBIS_ASSIGNMENT_TESTS_REPO="" \
    ANUBIS_ADMIN="OFF" \
    ANUBIS=1 \
    ANUBIS_IDE=1 \
    ANUBIS_ASSIGNMENT_NAME="" \
    ANUBIS_WEBTOP=OFF \
    HOME=/home/anubis \
    SHELL=/usr/bin/zsh \
    THEIA_DEFAULT_PLUGINS="local-dir:/opt/theia/plugins/" \
    LANG="en_US.UTF-8" \
    LC_ALL="en_US.UTF-8" \
    DOCKER_HOST="tcp://localhost:2376" \
    DOCKER_TLS_CERTDIR="/certs" \
    DOCKER_TLS_VERIFY="1" \
    DOCKER_CERT_PATH="/certs/client" \
    ANUBIS_RUN_DOCKERD="0"

WORKDIR /opt/theia

# Setup high level dependencies (mainly python)
RUN set -ex; \
    apt update; \
    apt upgrade -y; \
    apt-get install -y --no-install-recommends \
      wget gpg gnupg apt-transport-https ca-certificates  \
      apt-utils pkg-config curl lsb-release; \
    \
    `# Add LLVM to apt`  \
    wget -O - https://apt.llvm.org/llvm-snapshot.gpg.key | apt-key add -; \
    echo "deb http://apt.llvm.org/${DIST}/ llvm-toolchain-${DIST} main" \
      > /etc/apt/sources.list.d/llvm.list; \
    \
    `# Add Docker to apt`  \
    mkdir -p /etc/apt/keyrings; \
    curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg; \
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
        $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null; \
    \
    `# ctop.sh`  \
    curl -fsSL https://azlux.fr/repo.gpg.key | gpg --dearmor -o /usr/share/keyrings/azlux-archive-keyring.gpg; \
    echo \
      "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/azlux-archive-keyring.gpg] http://packages.azlux.fr/debian \
      $(lsb_release -cs) main" | tee /etc/apt/sources.list.d/azlux.list >/dev/null; \
    \
    apt-get update; \
    apt-get install -y --no-install-recommends \
      git curl wget build-essential tree clangd zsh exa locales \
      libffi-dev libexpat1-dev \
      libgdbm-dev liblzma-dev zlib1g-dev \
      libsqlite3-dev libssl-dev openssl \
      libsecret-1-0 libsecret-1-dev sqlite3 libbz2-dev bzip2 fzf \
      nano htop less mariadb-client openssh-client pigz procps \
      docker-ce-cli docker-compose-plugin docker-ctop libncurses6 libncurses-dev; \
    apt remove -y python python-pip python3-pip python3; \
    cd /; \
    \
    `# Locale`  \
    echo 'LANG=en_US.UTF-8' > /etc/locale.conf; \
    echo 'en_US.UTF-8 UTF-8' > /etc/locale.gen; \
    echo 'LANG=en_US.UTF-8' > /etc/default/locale; \
    locale-gen en_US.UTF-8; \
    \
    `# Python from source `  \
    wget https://www.python.org/ftp/python/$PY_VERSION/Python-$PY_VERSION.tgz; \
    wget https://bootstrap.pypa.io/get-pip.py; \
    tar xvf Python-$PY_VERSION.tgz; \
    cd Python-$PY_VERSION; \
    ./configure \
      --enable-loadable-sqlite-extensions \
      --enable-optimizations \
      --enable-option-checking=fatal \
      --enable-shared \
      --with-system-expat \
      --with-system-ffi \
      --without-ensurepip; \
    make -j $(nproc) LDFLAGS="-Wl,--strip-all"; \
    make install; \
    ldconfig; \
    ln -sf /usr/local/bin/python3 /usr/local/bin/python; \
    ln -sf /usr/local/bin/python3 /usr/bin/python3; \
    ln -sf /usr/local/bin/python3 /usr/bin/python; \
    cd ..; \
    rm -rf Python-$PY_VERSION; \
    rm Python-$PY_VERSION.tgz; \
    python3 get-pip.py \
      --disable-pip-version-check \
      --no-cache-dir; \
    rm get-pip.py; \
    pip3 install --upgrade --no-cache-dir pip; \
    pip3 install --upgrade --no-cache-dir python-language-server supervisor setuptools pylint; \
    pip3 install --upgrade --no-cache-dir pyyaml requests gitpython parse lorem; \
    \
    `# Users, permissions, etc` \
    adduser --disabled-password --gecos '' --uid ${PUID} anubis; \
    chown -R ${PUID}:${GUID} /home/anubis; \
    cd /home/anubis; \
    cp /etc/skel/.bash_logout /etc/skel/.bashrc /etc/skel/.profile /home/anubis/; \
    echo 'cat /etc/motd' >> /etc/skel/.bashrc; \
    cd /home/anubis; \
    chmod g+rw /home; \
    chown -R anubis:anubis /home/anubis; \
    \
    `# Cleanup`  \
    apt remove -y libffi-dev libexpat1-dev \
      libgdbm-dev liblzma-dev zlib1g-dev; \
    apt autoclean -y; \
    apt autoremove -y; \
    rm -rf /tmp/*; \
    rm -rf /usr/share/doc; \
    rm -rf /var/cache/apt/*; \
    rm -rf /var/lib/apt/lists/*; \
    rm -rf /home/node/*; \
    find / -depth \
      \( -name .cache -o -name __pycache__ -o -name '*.pyc' -o -name .git -o -name .github \) \
      -exec 'rm' '-rf' '{}' '+';

# Build Theia Application
ARG BUILD_THEIA=ON
ARG MS_PYTHON_VERSION=2020.10.332292344
ARG MS_JUPYTER_VERSION=2021.8.0
COPY latest.package.json package.json
RUN set -ex; \
  if [ "${BUILD_THEIA}" = "ON" ]; then  \
    apt update; \
    apt install -y --no-install-recommends pkg-config libsecret-1-dev unzip zip; \
    yarn --pure-lockfile; \
    SCRIPT_LINE='<script>function ping(){if (document.hasFocus()){fetch("/ide/ping")}};document.addEventListener("focus",ping);setInterval(ping,60000)</script>'; \
    NODE_OPTIONS="--max-old-space-size=4096" yarn theia build; \
    HEAD_LINE_NUMBER=$(grep -n '</head>' lib/index.html | cut -f1 -d:); \
    sed -i "${HEAD_LINE_NUMBER}i${SCRIPT_LINE}" lib/index.html; \
    yarn theia download:plugins; \
    wget https://open-vsx.org/api/ms-python/python/${MS_PYTHON_VERSION}/file/ms-python.python-${MS_PYTHON_VERSION}.vsix -O /opt/theia/plugins/ms-python.python-${MS_PYTHON_VERSION}.vsix; \
    mkdir -p /opt/theia/plugins/ms-python.python; \
    unzip -d /opt/theia/plugins/ms-python.python /opt/theia/plugins/ms-python.python-${MS_PYTHON_VERSION}.vsix; \
    rm /opt/theia/plugins/ms-python.python-${MS_PYTHON_VERSION}.vsix; \
    sed -i 's/"ms-toolsai.jupyter,ms-python.vscode-pylance"/"ms-toolsai.jupyter"/' /opt/theia/plugins/ms-python.python/extension.vsixmanifest; \
    sed -i '122 s/.*/    match = re.match(r"(\\d+)(?:\\.(\\d+)(?:\\.\\d+)?)?$", version)/' /opt/theia/plugins/ms-python.python/extension/pythonFiles/lib/python/parso/utils.py; \
    wget https://raw.githubusercontent.com/davidhalter/parso/master/parso/python/grammar39.txt -O /opt/theia/plugins/ms-python.python/extension/pythonFiles/lib/python/parso/python/grammar39.txt; \
    wget https://raw.githubusercontent.com/davidhalter/parso/master/parso/python/grammar310.txt -O /opt/theia/plugins/ms-python.python/extension/pythonFiles/lib/python/parso/python/grammar310.txt; \
    wget https://raw.githubusercontent.com/davidhalter/parso/master/parso/python/grammar311.txt -O /opt/theia/plugins/ms-python.python/extension/pythonFiles/lib/python/parso/python/grammar311.txt; \
    wget https://raw.githubusercontent.com/davidhalter/parso/master/parso/python/grammar312.txt -O /opt/theia/plugins/ms-python.python/extension/pythonFiles/lib/python/parso/python/grammar312.txt; \
    wget https://open-vsx.org/api/ms-toolsai/jupyter/${MS_JUPYTER_VERSION}/file/ms-toolsai.jupyter-${MS_JUPYTER_VERSION}.vsix -O /opt/theia/plugins/ms-toolsai.jupyter-${MS_JUPYTER_VERSION}.vsix; \
    mkdir -p /opt/theia/plugins/ms-toolsai.jupyter; \
    unzip -d /opt/theia/plugins/ms-toolsai.jupyter /opt/theia/plugins/ms-toolsai.jupyter-${MS_JUPYTER_VERSION}.vsix; \
    rm /opt/theia/plugins/ms-toolsai.jupyter-${MS_JUPYTER_VERSION}.vsix; \
    yarn --production; \
    yarn autoclean --init; \
    echo *.ts >> .yarnclean; \
    echo *.ts.map >> .yarnclean; \
    echo *.spec.* >> .yarnclean; \
    yarn autoclean --force; \
    yarn cache clean; \
    apt autoclean -y; \
    apt autoremove -y; \
    rm -rf /tmp/*; \
    rm -rf /usr/share/doc; \
    rm -rf /var/lib/apt/lists/*; \
    rm -rf /home/node/*; \
    find / -depth \
      \( -name .cache -o -name __pycache__ -o -name '*.pyc' -o -name .git -o -name .github \) \
      -exec 'rm' '-rf' '{}' '+';  \
  fi

# PRETTY SHELL NECESSARY
RUN set -ex; \
  mkdir -p /etc/anubis/skel/; \
  mkdir -p /etc/anubis/init.d/; \
  chsh -s /usr/bin/zsh ${USER}; \
  git clone https://github.com/robbyrussell/oh-my-zsh.git /opt/oh-my-zsh; \
  git clone https://github.com/Aloxaf/fzf-tab.git /opt/oh-my-zsh/custom/plugins/fzf-tab; \
  git clone https://github.com/zsh-users/zsh-syntax-highlighting.git /opt/oh-my-zsh/custom/plugins/zsh-syntax-highlighting; \
  git clone https://github.com/zsh-users/zsh-autosuggestions.git /opt/oh-my-zsh/custom/plugins/zsh-autosuggestions;
COPY zshrc /etc/anubis/skel/.zshrc
COPY anubis.zsh-theme /opt/oh-my-zsh/custom/themes/anubis.zsh-theme

# Anubis CLI
RUN set -eux; \
  git clone https://github.com/AnubisLMS/CLI.git /cli; \
  pip3 install --no-cache-dir /cli; \
  rm -rf /cli

COPY motd.txt /etc/motd
COPY supervisord.conf initialize-ide.py /

USER ${USER}
WORKDIR /opt/theia
ENTRYPOINT ["/usr/local/bin/supervisord", "--nodaemon", "-c", "/supervisord.conf"]
