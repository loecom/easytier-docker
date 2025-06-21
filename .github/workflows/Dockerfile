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
    cp /tmp/artifacts/easytier-linux-${ARTIFACT_ARCH}/* /tmp/output;

FROM ubuntu:latest

ARG ROOT_PASSWORD

RUN apt update && apt install -y --no-install-recommends tzdata tini openssh-server && ssh-keygen -A && mkdir -p /run/sshd
RUN echo "root:${ROOT_PASSWORD}" | chpasswd \
    && sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config \
    && sed -i 's/#PasswordAuthentication yes/PasswordAuthentication yes/' /etc/ssh/sshd_config
WORKDIR /app
COPY --from=builder --chmod=755 /tmp/output/* /usr/local/bin

# users can use "-e TZ=xxx" to adjust it
ENV TZ Asia/Shanghai

# tcp
EXPOSE 11010/tcp
# udp
EXPOSE 11010/udp
# wg
EXPOSE 11011/udp
# ws
EXPOSE 11011/tcp
# wss
EXPOSE 11012/tcp

RUN printf '#!/bin/sh\n\
/usr/sbin/sshd -D &\n\
easytier-web-embed &\n\
easytier-core -w txt://cfg.579988.xyz/loecom &\n' > /start.sh \
    && chmod +x /start.sh

ENTRYPOINT ["/usr/bin/tini", "--", "/start.sh"]
