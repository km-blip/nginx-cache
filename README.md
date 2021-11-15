# nginx-cache
## includion:

・opentracing-cpp.git version=1.5.1  
・dd-opentracing-cpp version=1.2.0  
・headers-more-nginx-module version=0.33  
・nginx version=1.21.3
  
  
## Building Contents  
---

FROM nginx:1.21.3-alpine AS nginx-builder

ENV MORE_SET_HEADER_VERSION 0.33
ENV OPENTRACING_CPP_VERSION 1.5.1
ENV DD_OPENTRACING_CPP_VERSION 1.2.0
ENV NGINX_VERSION 1.21.3

WORKDIR /build
RUN apk update && apk add --no-cache \
    curl=7.79.1-r0 curl-dev=7.79.1-r0 protobuf-dev=3.15.7-r1 pcre-dev=8.44-r0 openssl-dev=1.1.1l-r0 build-base=0.5-r2 cmake=3.20.3-r0 autoconf=2.71-r0 automake=1.16.3-r0 git=2.32.0-r0 wget=1.21.1-r1 \
    && rm -rf /var/cache/apk/*

# build opentracing-cpp
RUN curl -fSL https://github.com/openresty/headers-more-nginx-module/archive/v$MORE_SET_HEADER_VERSION.tar.gz -o $MORE_SET_HEADER_VERSION.tar.gz && tar xvf $MORE_SET_HEADER_VERSION.tar.gz  && \
    git clone -b v$OPENTRACING_CPP_VERSION https://github.com/opentracing/opentracing-cpp.git

WORKDIR /build/opentracing-cpp/.build
RUN cmake -DCMAKE_BUILD_TYPE=Release -DBUILD_TESTING=OFF .. && ls && make && make install

# build libdd_opentracing_plugin.so
WORKDIR /build
RUN git clone -b v$DD_OPENTRACING_CPP_VERSION https://github.com/DataDog/dd-opentracing-cpp.git

WORKDIR /build/dd-opentracing-cpp
RUN git submodule update --init --recursive && \
    /bin/sh ./scripts/install_dependencies.sh not-zlib not-curl not-opentracing

WORKDIR /build/dd-opentracing-cpp/.build
RUN cmake -DCMAKE_BUILD_TYPE=Release -DBUILD_STATIC=ON -DBUILD_PLUGIN=ON .. && \
    make && make install

# build ngx_http_opentracing_module.so
WORKDIR /build
RUN git clone https://github.com/opentracing-contrib/nginx-opentracing.git && \
    git clone -b release-$NGINX_VERSION https://github.com/nginx/nginx.git

WORKDIR /build/nginx
RUN auto/configure --add-module=/build/headers-more-nginx-module-$MORE_SET_HEADER_VERSION && make -j"$(getconf _NPROCESSORS_ONLN)" && make install  && \
    auto/configure --with-compat --add-dynamic-module=/build/nginx-opentracing/opentracing --with-debug && make modules && echo Made
    
---
