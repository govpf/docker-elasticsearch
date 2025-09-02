################################################################################
# Build stage 0 `zlib`:
# Compile zlib for the current architecture
################################################################################
FROM govpf/debian:bullseye AS zlib
ENV ZLIB_VERSION 1.2.8

RUN apt-get update && DEBIAN_FRONTEND=noninteractive apt-get install -y \
    curl gcc make xz-utils && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /tmp
# téléchargement officiel de zlib
RUN curl --retry 10 -S -L -o zlib-${ZLIB_VERSION}.tar.gz https://zlib.net/zlib-${ZLIB_VERSION}.tar.gz

# vérification optionnelle du checksum (si souhaité, on peut décommenter et adapter)
# RUN echo "checksum_expected_here  zlib-${ZLIB_VERSION}.tar.gz" | sha256sum --check -

RUN tar -xzf zlib-${ZLIB_VERSION}.tar.gz
WORKDIR /tmp/zlib-${ZLIB_VERSION}
RUN ./configure --prefix=/usr/local/cloudflare-zlib && make && make install

################################################################################
# Build stage 1 `builder`:
# Extract Elasticsearch artifact
################################################################################
FROM govpf/debian:bullseye AS builder

# `tini` est utilisé pour gérer proprement les processus enfants
RUN set -eux; \
    tini_bin=""; \
    case "$(arch)" in \
        aarch64) tini_bin='tini-arm64' ;; \
        x86_64)  tini_bin='tini-amd64' ;; \
        *) echo >&2 "Unsupported architecture $(arch)"; exit 1 ;; \
    esac; \
    curl --retry 10 -S -L -O https://github.com/krallin/tini/releases/download/v0.19.0/${tini_bin}; \
    curl --retry 10 -S -L -O https://github.com/krallin/tini/releases/download/v0.19.0/${tini_bin}.sha256sum; \
    sha256sum -c ${tini_bin}.sha256sum; \
    rm ${tini_bin}.sha256sum; \
    mv ${tini_bin} /bin/tini; \
    chmod +x /bin/tini

RUN mkdir /usr/share/elasticsearch
WORKDIR /usr/share/elasticsearch

ARG ELASTIC_VERSION=7.14.1

RUN curl --retry 10 -S -L -o /opt/elasticsearch.tar.gz https://artifacts-no-kpi.elastic.co/downloads/elasticsearch/elasticsearch-${ELASTIC_VERSION}-linux-$(arch).tar.gz
RUN tar -xf /opt/elasticsearch.tar.gz --strip-components=1

COPY config/elasticsearch.yml config/
COPY config/log4j2.properties config/log4j2.docker.properties

RUN sed -i -e 's/ES_DISTRIBUTION_TYPE=tar/ES_DISTRIBUTION_TYPE=docker/' bin/elasticsearch-env && \
    mkdir data && \
    mv config/log4j2.properties config/log4j2.file.properties && \
    mv config/log4j2.docker.properties config/log4j2.properties && \
    find . -type d -exec chmod 0555 {} + && \
    find . -type f -exec chmod 0444 {} + && \
    chmod 0555 bin/* jdk/bin/* jdk/lib/jspawnhelper modules/x-pack-ml/platform/linux-*/bin/* && \
    chmod 0775 bin config config/jvm.options.d data logs plugins && \
    find config -type f -exec chmod 0664 {} +

################################################################################
# Build stage 2 (final Elasticsearch image)
################################################################################
FROM govpf/debian:bullseye

RUN export DEBIAN_FRONTEND=noninteractive && \
    apt-get update && \
    apt-get upgrade -y && \
    apt-get install -y --no-install-recommends \
        ca-certificates curl netcat p11-kit unzip vim-tiny zip && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

RUN groupadd -g 1000 elasticsearch && \
    adduser --uid 1000 --gid 1000 --home /usr/share/elasticsearch elasticsearch && \
    adduser elasticsearch root && \
    chown -R 0:0 /usr/share/elasticsearch

ENV ELASTIC_CONTAINER true
WORKDIR /usr/share/elasticsearch

COPY --from=builder --chown=0:0 /usr/share/elasticsearch /usr/share/elasticsearch
COPY --from=builder --chown=0:0 /bin/tini /bin/tini
COPY --from=zlib --chown=0:0 /usr/local/cloudflare-zlib /usr/local/cloudflare-zlib

ENV PATH /usr/share/elasticsearch/bin:$PATH
ENV ES_ZLIB_PATH /usr/local/cloudflare-zlib/lib

COPY bin/docker-entrypoint.sh /usr/local/bin/docker-entrypoint.sh

RUN chmod g=u /etc/passwd && \
    chmod 0555 /usr/local/bin/docker-entrypoint.sh && \
    find / -xdev -perm -4000 -exec chmod ug-s {} + && \
    chmod 0775 /usr/share/elasticsearch && \
    chown elasticsearch bin config config/jvm.options.d data logs plugins

COPY bin/docker-openjdk /etc/ca-certificates/update.d/docker-openjdk
RUN /etc/ca-certificates/update.d/docker-openjdk

EXPOSE 9200 9300

ARG ELASTIC_VERSION=7.14.1
ARG BUILD_DATE

LABEL org.label-schema.build-date="${BUILD_DATE}" \
      org.label-schema.license="Elastic-License-2.0" \
      org.label-schema.name="Elasticsearch" \
      org.label-schema.schema-version="1.0" \
      org.label-schema.url="https://www.elastic.co/products/elasticsearch" \
      org.label-schema.usage="https://www.elastic.co/guide/en/elasticsearch/reference/index.html" \
      org.label-schema.vcs-ref="66b55ebfa59c92c15db3f69a335d500018b3331e" \
      org.label-schema.vcs-url="https://github.com/elastic/elasticsearch" \
      org.label-schema.vendor="Elastic" \
      org.label-schema.version="${ELASTIC_VERSION}" \
      org.opencontainers.image.created="${BUILD_DATE}" \
      org.opencontainers.image.documentation="https://www.elastic.co/guide/en/elasticsearch/reference/index.html" \
      org.opencontainers.image.licenses="Elastic-License-2.0" \
      org.opencontainers.image.revision="66b55ebfa59c92c15db3f69a335d500018b3331e" \
      org.opencontainers.image.source="https://github.com/elastic/elasticsearch" \
      org.opencontainers.image.title="Elasticsearch" \
      org.opencontainers.image.url="https://www.elastic.co/products/elasticsearch" \
      org.opencontainers.image.vendor="Elastic" \
      org.opencontainers.image.version="${ELASTIC_VERSION}"

RUN bin/elasticsearch-plugin install analysis-icu

ENTRYPOINT ["/bin/tini", "--", "/usr/local/bin/docker-entrypoint.sh"]
CMD ["eswrapper"]
USER elasticsearch:root
