# syntax=docker/dockerfile:1
ARG MAJOR_VERSION=6.2
ARG RELEASE=7
ARG ZBX_VERSION=${MAJOR_VERSION}.7
ARG BUILD_BASE_IMAGE=zabbix-build-base:rhel-${ZBX_VERSION}

FROM ${BUILD_BASE_IMAGE} as builder

ARG MAJOR_VERSION
ARG RELEASE
ARG ZBX_VERSION
ARG ZBX_SOURCES=https://git.zabbix.com/scm/zbx/zabbix.git

LABEL description="Zabbix build base for MySQL based images" \
      maintainer="alexey.pustovalov@zabbix.com" \
      name="zabbix/zabbix-build-mysql-62" \
      release="${RELEASE}" \
      summary="Zabbix build base (MySQL)" \
      url="https://www.zabbix.com/" \
      vendor="Zabbix LLC" \
      version="${MAJOR_VERSION}" \
      io.k8s.description="Zabbix build base for MySQL based images" \
      io.k8s.display-name="Zabbix build base (MySQL)" \
      io.openshift.expose-services="" \
      io.openshift.tags="zabbix,build,mysql" \
      org.label-schema.description="Zabbix build base for MySQL based images" \
      org.label-schema.name="zabbix-build-mysql-rhel" \
      org.label-schema.url="https://zabbix.com/" \
      org.label-schema.vcs-ref="${VCS_REF}" \
      org.label-schema.vendor="Zabbix LLC"

ENV MONGODB_PLUGIN_SOURCES=https://git.zabbix.com/scm/ap/mongodb.git MONGODB_PLUGIN_VERSION=${ZBX_VERSION} \
    POSTGRESQL_PLUGIN_SOURCES=https://git.zabbix.com/scm/ap/postgresql.git POSTGRESQL_PLUGIN_VERSION=${ZBX_VERSION}

COPY ["conf/chromedp_no_sandbox.patch", "/tmp/chromedp_no_sandbox.patch"]

RUN set -eux && \
    ARCH_SUFFIX="$(arch)"; \
    case "$ARCH_SUFFIX" in \
        x86_64) \
            additional_components='--enable-java'; \
            ;; \
        aarch64) \
            additional_components='--enable-java'; \
            ;; \
        ppc64le) \
            additional_components=''; \
            ;; \
        *) echo "Unknown ARCH_SUFFIX=${ARCH_SUFFIX-}"; exit 1 ;; \
    esac; \
    cd /tmp/ && \
    git -c advice.detachedHead=false clone ${ZBX_SOURCES} --branch ${ZBX_VERSION} --depth 1 --single-branch /tmp/zabbix-${ZBX_VERSION} && \
    cd /tmp/zabbix-${ZBX_VERSION} && \
    patch -p1 < /tmp/chromedp_no_sandbox.patch && \
    zabbix_revision=`git rev-parse --short HEAD` && \
    sed -i "s/{ZABBIX_REVISION}/$zabbix_revision/g" include/version.h && \
    sed -i "s/{ZABBIX_REVISION}/$zabbix_revision/g" src/go/pkg/version/version.go && \
    sed -i "s/{ZABBIX_REVISION}/$zabbix_revision/g" src/zabbix_java/src/com/zabbix/gateway/GeneralInformation.java && \
    ./bootstrap.sh && \
    export CFLAGS="-fPIC -pie -Wl,-z,relro,-z,now,-z,defs" && \
    export CFLAGS="$CFLAGS -D_FORTIFY_SOURCE=2 -fexceptions -O2 -pipe" && \
    ./configure \
            --datadir=/usr/lib \
            --libdir=/usr/lib/zabbix \
            --prefix=/usr \
            --sysconfdir=/etc/zabbix \
            --enable-ipv6 \
            --enable-agent \
            --enable-agent2 \
            --enable-proxy \
            --enable-server \
            --enable-webservice \
            --with-ldap \
            --with-libcurl \
#            --with-libmodbus \
            --with-libpcre2 \
            --with-libxml2 \
            --with-mysql \
            --with-net-snmp \
            --with-openipmi \
            --with-openssl \
            --with-ssh \
            --with-unixodbc \
            $additional_components \
            --silent && \
    make -j"$(nproc)" -s dbschema && \
    make -j"$(nproc)" -s && \
    make -j"$(nproc)" -s gettext && \
    cat database/mysql/schema.sql > database/mysql/create.sql && \
    cat database/mysql/images.sql >> database/mysql/create.sql && \
    cat database/mysql/data.sql >> database/mysql/create.sql && \
    gzip -c database/mysql/create.sql > database/mysql/create_server.sql.gz && \
    rm -rf database/mysql/create.sql && \
    cat database/mysql/schema.sql > database/mysql/create.sql && \
    gzip -c database/mysql/create.sql > database/mysql/create_proxy.sql.gz && \
    rm -rf database/mysql/create.sql && \
    strip /tmp/zabbix-${ZBX_VERSION}/src/zabbix_agent/zabbix_agentd && \
    strip /tmp/zabbix-${ZBX_VERSION}/src/zabbix_server/zabbix_server && \
    strip /tmp/zabbix-${ZBX_VERSION}/src/zabbix_proxy/zabbix_proxy && \
    strip /tmp/zabbix-${ZBX_VERSION}/src/go/bin/zabbix_agent2 && \
    strip /tmp/zabbix-${ZBX_VERSION}/src/zabbix_get/zabbix_get && \
    strip /tmp/zabbix-${ZBX_VERSION}/src/zabbix_sender/zabbix_sender && \
    strip /tmp/zabbix-${ZBX_VERSION}/src/go/bin/zabbix_web_service && \
    cd /tmp/ && \
    git -c advice.detachedHead=false clone ${MONGODB_PLUGIN_SOURCES} --branch ${MONGODB_PLUGIN_VERSION} --depth 1 --single-branch /tmp/mongodb_plugin && \
    cd /tmp/mongodb_plugin && \
    make && \
    strip /tmp/mongodb_plugin/zabbix-agent2-plugin-mongodb && \
    cd /tmp/ && \
    git -c advice.detachedHead=false clone ${POSTGRESQL_PLUGIN_SOURCES} --branch ${POSTGRESQL_PLUGIN_VERSION} --depth 1 --single-branch /tmp/postgresql_plugin && \
    cd /tmp/postgresql_plugin && \
    make && \
    strip /tmp/postgresql_plugin/zabbix-agent2-plugin-postgresql && \
    rm -rf $(go env GOCACHE)
