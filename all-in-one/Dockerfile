ARG HUB=higress-registry.cn-hangzhou.cr.aliyuncs.com/higress
ARG BASE_VERSION=2022-10-27T19-02-22
ARG CORE_VERSION=2.2.0
ARG CONSOLE_VERSION=2.2.1
ARG APISERVER_VERSION=0.0.29
ARG ENVOY_VERSION=2.2.1
ARG PLUGIN_SERVER_VERSION=2.2.1

FROM ${HUB}/api-server:${APISERVER_VERSION} AS apiserver
FROM ${HUB}/higress:${CORE_VERSION} AS controller
FROM ${HUB}/pilot:${CORE_VERSION} AS pilot
FROM ${HUB}/gateway:${CORE_VERSION} AS gateway
FROM swr.cn-east-3.myhuaweicloud.com/btc8_public/higress/higress-console:1.0.0 AS console
FROM ${HUB}/eclipse-temurin:21-jre AS jdk
FROM ${HUB}/plugin-server:${PLUGIN_SERVER_VERSION} AS plugin-server

FROM ${HUB}/base:${BASE_VERSION}

# Install API server
COPY --from=apiserver /apiserver /usr/local/bin/apiserver

# Install controller
COPY --from=controller /usr/local/bin/higress /usr/local/bin/higress

# Install pilot
COPY --from=pilot /usr/local/bin/pilot-discovery /usr/local/bin/pilot-discovery
COPY --from=pilot /usr/local/bin/higress-pilot-start.sh /usr/local/bin/higress-pilot-start.sh

# Install gateway
COPY --from=gateway /var/lib/istio/envoy/*.json /var/lib/istio/envoy/
COPY --from=gateway /var/lib/istio/envoy/*.so /var/lib/istio/envoy/
COPY --from=gateway /usr/local/bin/envoy /usr/local/bin/envoy
COPY --from=gateway /usr/local/bin/pilot-agent /usr/local/bin/pilot-agent
COPY --from=gateway /usr/local/bin/higress-proxy-*.sh /usr/local/bin/
RUN chmod a+x /usr/local/bin/higress-proxy-container-init.sh; \
  sed -i 's/1337/0/g' /usr/local/bin/higress-proxy-container-init.sh; \
  /usr/local/bin/higress-proxy-container-init.sh
COPY --from=gateway /usr/local/bin/supercronic* /usr/local/bin/

# Install console
COPY --from=console /app /app

# Install plugin-server (nginx with wasm plugins)
COPY --from=plugin-server /usr/share/nginx/html/plugins /usr/share/nginx/html/plugins

# Install JDK required by console
ENV JAVA_HOME=/opt/java/openjdk
COPY --from=jdk $JAVA_HOME $JAVA_HOME
ENV PATH="${JAVA_HOME}/bin:${PATH}"

# Install supervisord, logrotate, cron and initialize related folders
RUN unset http_proxy https_proxy HTTP_PROXY HTTPS_PROXY; \
  if grep -qi 'ubuntu' /etc/os-release; then \
        sed -i 's@http://\(security\|archive\).ubuntu.com@http://mirrors.aliyun.com@g' /etc/apt/sources.list; \
    else \
        sed -i 's@http://\(security\|deb\).debian.org@http://mirrors.aliyun.com@g' /etc/apt/sources.list; \
    fi && \
  apt-get update --allow-unauthenticated; \
  apt-get install --no-install-recommends -y --allow-unauthenticated \
    supervisor logrotate cron nginx; \
  mkdir -p /var/log/higress; \
  mkdir -p /opt/data/defaultConfig && \
  mkdir -p /opt/data/defaultConfig/gateways;

# Copy local envoy and yq files (faster than downloading)
COPY --chown=root:root ./envoy-amd64.tar.gz /tmp/envoy.tar.gz
COPY --chown=root:root ./yq_linux_amd64 /usr/local/bin/yq
RUN tar -xzf /tmp/envoy.tar.gz -C / && \
  rm /tmp/envoy.tar.gz && \
  chmod +x /usr/local/bin/envoy; \
  chmod +x /usr/local/bin/yq;

COPY ./supervisord/supervisord.conf /etc/supervisor/conf.d/supervisord.conf

# Initialize configurations
COPY ./meshConfig /etc/istio/config
COPY ./gateway/podinfo /etc/istio/pod
COPY ./scripts /usr/local/bin
COPY ./apiserver/config /app/kubeconfig
COPY ./config /opt/data/defaultConfig
COPY ./plugin-server/nginx.conf /etc/nginx/plugin-server/nginx.conf

EXPOSE 8080 8443 8001

ENTRYPOINT ["supervisord", "-c", "/etc/supervisor/conf.d/supervisord.conf"]
