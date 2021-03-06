FROM httpd:2.4 as builder
RUN apt-get update \
 && apt-get install -y --no-install-recommends --no-install-suggests \
#RUN rm -f /etc/apt/apt.conf.d/docker-clean; echo 'Binary::apt::APT::Keep-Downloaded-Packages "true";' > /etc/apt/apt.conf.d/keep-cache
#RUN --mount=type=cache,target=/var/cache/apt --mount=type=cache,target=/var/lib/apt \
#  apt update && apt-get install -y --no-install-recommends --no-install-suggests \
      automake \
      g++ \
      libcurl4-gnutls-dev \
      libpcre++-dev \
      libtool \
      libxml2-dev \
      libyajl-dev \
      lua5.2-dev \
      make \
      openssl \
      pkgconf 

FROM builder as builder-ssdeep
ADD https://github.com/ssdeep-project/ssdeep/releases/download/release-2.14.1/ssdeep-2.14.1.tar.gz ./
RUN tar -xvzf ssdeep-2.14.1.tar.gz \
 && cd ssdeep-2.14.1 \
 && ./configure \
 && make -j$(nproc) \
 && make install \
 && make clean

# Generate self-signed certificates (if needed)
RUN mkdir -p /usr/share/TLS
COPY v2-apache/openssl.conf /usr/share/TLS
RUN openssl req -x509 -days 365 -new \
      -config /usr/share/TLS/openssl.conf \
      -keyout /usr/share/TLS/server.key \
      -out /usr/share/TLS/server.crt


FROM alpine as base-git
RUN apk --no-cache add git \
 && git clone https://github.com/SpiderLabs/ModSecurity --branch v2.9.3 --depth 1


FROM builder as builder-modsec
COPY --from=base-git /ModSecurity ./ModSecurity  
RUN cd ModSecurity \
 && ./autogen.sh \
 && ./configure \
 && make -j$(nproc) \
 && make install \
 && make clean

FROM httpd:2.4

ENV ACCESSLOG=/var/log/apache2/access.log \
    BACKEND=http://localhost:80 \
    BACKEND_WS=ws://localhost:8080 \
    ERRORLOG=/var/log/apache2/error.log \
    LOGLEVEL=warn \
    METRICS_ALLOW_FROM='127.0.0.0/255.0.0.0 ::1/128' \
    METRICS_DENY_FROM='All' \
    METRICSLOG='/dev/null combined' \
    MODSEC_AUDIT_LOG_FORMAT=JSON \
    MODSEC_AUDIT_LOG_TYPE=Serial \
    MODSEC_AUDIT_LOG=/dev/stdout \
    MODSEC_AUDIT_STORAGE=/var/log/modsecurity/audit/ \
    MODSEC_DATA_DIR=/tmp/modsecurity/data \
    MODSEC_DEBUG_LOG=/dev/null \
    MODSEC_DEBUG_LOGLEVEL=0 \
    MODSEC_PCRE_MATCH_LIMIT_RECURSION=500000 \
    MODSEC_PCRE_MATCH_LIMIT=500000 \
    MODSEC_REQ_BODY_ACCESS=on \
    MODSEC_REQ_BODY_LIMIT=13107200 \
    MODSEC_REQ_BODY_NOFILES_LIMIT=131072 \
    MODSEC_RESP_BODY_ACCESS=on \
    MODSEC_RESP_BODY_LIMIT=1048576 \
    MODSEC_RULE_ENGINE=on \
    MODSEC_TAG=modsecurity \
    MODSEC_TMP_DIR=/tmp/modsecurity/tmp \
    MODSEC_UPLOAD_DIR=/tmp/modsecurity/upload \
    PERFLOG='/dev/stdout perflogjson env=write_perflog' \
    PORT=80 \
    PROXY_PRESERVE_HOST=on \
    PROXY_SSL_CA_CERT=/etc/ssl/certs/ca-certificates.crt \
    PROXY_SSL_CERT_KEY=/usr/local/apache2/conf/server.key \
    PROXY_SSL_CERT=/usr/local/apache2/conf/server.crt \
    PROXY_SSL_CHECK_PEER_NAME=off \
    PROXY_SSL_VERIFY=none \
    PROXY_SSL=on \
    PROXY_TIMEOUT=60 \
    REMOTEIP_INT_PROXY='10.1.0.0/16' \
    REQ_HEADER_FORWARDED_PROTO='https' \
    SERVER_ADMIN=root@localhost \
    SERVER_NAME=localhost \
    SSL_ENGINE=on \
    SSL_PORT=443 \
    TIMEOUT=60 \
    WORKER_CONNECTIONS=400

RUN DEBIAN_FRONTEND=noninteractive \
    apt-get update -qq \
 && apt-get install -qq -y --no-install-recommends --no-install-suggests \
      ca-certificates \
      libcurl3-gnutls \
      libxml2 \
      libyajl2 \
 && apt-get clean \
 && rm -rf /var/lib/apt/lists/* \
 && mkdir -p /etc/modsecurity.d/ \
 && mkdir -p /var/log/apache2/

COPY --from=builder-ssdeep /usr/local/lib/libfuzzy.so.2.1.0                             /usr/local/lib/libfuzzy.so.2.1.0
COPY --from=builder-ssdeep /usr/local/bin/ssdeep                                        /usr/local/bin/ssdeep
COPY --from=builder-ssdeep /usr/share/TLS/server.key                                    /usr/local/apache2/conf/server.key
COPY --from=builder-ssdeep /usr/share/TLS/server.crt                                    /usr/local/apache2/conf/server.crt
COPY src/etc/modsecurity.d/*.conf /etc/modsecurity.d/
COPY v2-apache/conf/extra/*.conf /usr/local/apache2/conf/extra/

RUN ln -s libfuzzy.so.2.1.0 /usr/local/lib/libfuzzy.so \
 && ln -s libfuzzy.so.2.1.0 /usr/local/lib/libfuzzy.so.2 \
 && ldconfig

RUN sed -i -E 's|(Listen) [0-9]+|\1 ${PORT}|' /usr/local/apache2/conf/httpd.conf \
 && sed -i -E 's|(ServerTokens) Full|\1 Prod|' /usr/local/apache2/conf/extra/httpd-default.conf \
 && sed -i -E 's|#(ServerName) www.example.com:80|\1 ${SERVER_NAME}|' /usr/local/apache2/conf/httpd.conf \
 && sed -i -E 's|#(LoadModule unique_id_module)|\1|' /usr/local/apache2/conf/httpd.conf \
 && sed -i -E 's|#(LoadModule rewrite_module modules/mod_rewrite.so)|\1|' /usr/local/apache2/conf/httpd.conf \
 && sed -i -E 's|#(LoadModule proxy_module modules/mod_proxy.so)|\1|' /usr/local/apache2/conf/httpd.conf \
 && sed -i -E 's|#(LoadModule proxy_http_module modules/mod_proxy_http.so)|\1|' /usr/local/apache2/conf/httpd.conf \
 && sed -i -E 's|#(LoadModule remoteip_module modules/mod_remoteip.so)|\1|' /usr/local/apache2/conf/httpd.conf \
 && sed -i -E 's|#(LoadModule socache_shmcb_module modules/mod_socache_shmcb.so)|\1|' /usr/local/apache2/conf/httpd.conf \
 && sed -i -E 's|#(LoadModule ssl_module modules/mod_ssl.so)|\1|' /usr/local/apache2/conf/httpd.conf \
 && sed -i -E 's|#(Include conf/extra/httpd-default.conf)|\1|' /usr/local/apache2/conf/httpd.conf \
 && sed -i -E 's|#(Include conf/extra/httpd-proxy.conf)|\1|' /usr/local/apache2/conf/httpd.conf \
 && sed -i -E 's|#(Include conf/extra/httpd-ssl.conf)|\1|' /usr/local/apache2/conf/httpd.conf \
 && sed -i -E 's|#(Include conf/extra/httpd-vhosts.conf)|\1|' /usr/local/apache2/conf/httpd.conf \
 && echo 'Include conf/extra/httpd-locations.conf' >> /usr/local/apache2/conf/httpd.conf \
 && echo 'Include conf/extra/httpd-modsecurity.conf' >> /usr/local/apache2/conf/httpd.conf \
 && sed -i -E 's|(MaxRequestWorkers[ ]*)[0-9]*|\1${WORKER_CONNECTIONS}|' /usr/local/apache2/conf/extra/httpd-mpm.conf

RUN chgrp -R 0 /var/log/ /usr/local/apache2/ \
 && chmod -R g=u /var/log/ /usr/local/apache2/

COPY --from=builder-modsec /usr/local/apache2/modules/mod_security2.so                  /usr/local/apache2/modules/mod_security2.so
COPY --from=builder-modsec /usr/local/apache2/ModSecurity/modsecurity.conf-recommended  /etc/modsecurity.d/modsecurity.conf
COPY --from=builder-modsec /usr/local/apache2/ModSecurity/unicode.mapping               /etc/modsecurity.d/unicode.mapping

CMD ["/usr/local/apache2/bin/apachectl", "-D", "FOREGROUND"]
