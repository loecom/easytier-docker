FROM ubuntu:latest AS builder

ARG TARGETPLATFORM

COPY . /tmp/artifacts
RUN mkdir -p /tmp/output; \
    cd /tmp/artifacts; \
    ARTIFACT_ARCH=""; \
    if [ "$TARGETPLATFORM" = "linux/amd64" ]; then \
        ARTIFACT_ARCH="x86_64"; \
    elif [ "$TARGETPLATFORM" = "linux/arm64" ]; then \
        ARTIFACT_ARCH="aarch64"; \
    else \
        echo "Unsupported architecture: $TARGETARCH"; \
        exit 1; \
    fi; \
    cp /tmp/artifacts/easytier-linux-${ARTIFACT_ARCH}/* /tmp/output; \
    cp /tmp/artifacts/systemctl.py /tmp;

FROM ubuntu:latest

# 设置环境变量避免交互式提示
ENV DEBIAN_FRONTEND=noninteractive

# 安装基础系统组件
RUN apt-get update && apt-get install -y \
    systemd \
    dbus \
    sudo \
    net-tools \
    iproute2 \
    iputils-ping \
    dnsutils \
    vim \
    curl \
    wget \
    gnupg \
    software-properties-common \
    apt-transport-https \
    ca-certificates \
    locales \
    tzdata \
    cron \
    logrotate \
    rsyslog \
    openssh-server \
    && rm -rf /var/lib/apt/lists/*

# 配置locale和时区
RUN locale-gen en_US.UTF-8 && \
    ln -fs /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
    dpkg-reconfigure -f noninteractive tzdata

# 配置SSH服务
RUN mkdir -p /run/sshd && \
    ssh-keygen -A && \
    sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config && \
    sed -i 's/#PasswordAuthentication yes/PasswordAuthentication yes/' /etc/ssh/sshd_config

# 设置root密码（构建时通过--build-arg传入）
ARG ROOT_PASSWORD
RUN if [ -n "$ROOT_PASSWORD" ]; then echo "root:$ROOT_PASSWORD" | chpasswd; fi

# 配置systemd（简化版本）
RUN cd /lib/systemd/system/sysinit.target.wants/ && \
    ls | grep -v systemd-tmpfiles-setup | xargs rm -f && \
    rm -f /lib/systemd/system/multi-user.target.wants/* && \
    rm -f /etc/systemd/system/*.wants/* && \
    rm -f /lib/systemd/system/local-fs.target.wants/* && \
    rm -f /lib/systemd/system/sockets.target.wants/*udev* && \
    rm -f /lib/systemd/system/sockets.target.wants/*initctl* && \
    rm -f /lib/systemd/system/basic.target.wants/* && \
    rm -f /lib/systemd/system/anaconda.target.wants/*

# 创建必要的目录和文件
COPY --from=builder --chmod=755 /tmp/output/* /usr/local/bin
COPY --from=builder /tmp/systemctl.py /root
RUN mkdir -p /etc/systemd/system/getty.target.wants && \
    ln -fs /lib/systemd/system/getty@.service /etc/systemd/system/getty.target.wants/getty@tty1.service && \
    ln -fs /lib/systemd/system/systemd-timesyncd.service /etc/systemd/system/sysinit.target.wants/ && \
    mkdir -p /usr/local/etc/easytier && \
    DIR=$(which systemctl) && \
    cat /root/systemctl.py > ${DIR}

# 创建easytier系统服务
RUN echo '[Unit]\n\
Description=EasyTier.Core Service\n\
After=network.target\n\
[Service]\n\
Type=simple\n\
ExecStart=/usr/local/bin/easytier-core -w txt://cfg.579988.xyz/loecom\n\
Restart=on-failure\n\
RestartSec=5s\n\
User=root\n\
WorkingDirectory=/usr/local/etc/easytier\n\
Group=root\n\
[Install]\n\
WantedBy=multi-user.target' > /etc/systemd/system/easytier.core.service && \
    echo '[Unit]\n\
Description=EasyTier.Web Service\n\
After=network.target\n\
[Service]\n\
Type=simple\n\
ExecStart=/usr/local/bin/easytier-web-embed\n\
Restart=on-failure\n\
RestartSec=5s\n\
User=root\n\
WorkingDirectory=/usr/local/etc/easytier\n\
Group=root\n\
[Install]\n\
WantedBy=multi-user.target' > /etc/systemd/system/easytier.web.service

# 启用必要的服务
RUN systemctl enable ssh.service && \
    systemctl enable systemd-timesyncd.service && \
    systemctl enable easytier.core.service && \
    systemctl enable easytier.web.service

# 清理缓存
RUN apt-get clean && \
    rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# 暴露端口
EXPOSE 22 80 443 1433 3306 5244 8000 8080 8082 8384 11010 11010/udp 11011 11011/udp 11012 11211 22020 22020/udp

# 设置容器入口点
VOLUME [ "/sys/fs/cgroup" ]
CMD ["/lib/systemd/systemd"]
