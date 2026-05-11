# Custom nginx image with the ngx_brotli dynamic module.
# Dynamic modules must be built against the exact nginx version in the base
# image, so the builder stage uses the same image and reads its version.

ARG NGINX_IMAGE_TAG=1.27-alpine

FROM nginx:${NGINX_IMAGE_TAG} AS builder

RUN apk add --no-cache \
        build-base \
        pcre-dev \
        zlib-dev \
        openssl-dev \
        linux-headers \
        cmake \
        git \
        curl

WORKDIR /build

RUN git clone --depth=1 --recurse-submodules --shallow-submodules \
        https://github.com/google/ngx_brotli.git

RUN cd ngx_brotli/deps/brotli \
    && mkdir -p out && cd out \
    && cmake -DCMAKE_BUILD_TYPE=Release \
             -DBUILD_SHARED_LIBS=OFF \
             -DCMAKE_C_FLAGS="-O2" \
             -DCMAKE_CXX_FLAGS="-O2" \
             -DCMAKE_INSTALL_PREFIX=./installed \
             .. \
    && cmake --build . --config Release --target brotlienc

RUN NGINX_VERSION=$(nginx -v 2>&1 | sed -n 's/^nginx version: nginx\/\(.*\)$/\1/p') \
    && echo "Building modules for nginx ${NGINX_VERSION}" \
    && curl -fsSL "https://nginx.org/download/nginx-${NGINX_VERSION}.tar.gz" | tar xz \
    && cd "nginx-${NGINX_VERSION}" \
    && ./configure --with-compat --add-dynamic-module=/build/ngx_brotli \
    && make -j"$(nproc)" modules \
    && install -m 644 objs/ngx_http_brotli_filter_module.so /tmp/ \
    && install -m 644 objs/ngx_http_brotli_static_module.so /tmp/

FROM nginx:${NGINX_IMAGE_TAG}

COPY --from=builder /tmp/ngx_http_brotli_filter_module.so /etc/nginx/modules/
COPY --from=builder /tmp/ngx_http_brotli_static_module.so /etc/nginx/modules/

# load_module must be in the main context (before events{}), so prepend it
# to the base image's nginx.conf rather than dropping a file in conf.d/.
RUN printf 'load_module modules/ngx_http_brotli_filter_module.so;\nload_module modules/ngx_http_brotli_static_module.so;\n' \
        | cat - /etc/nginx/nginx.conf > /tmp/nginx.conf \
    && mv /tmp/nginx.conf /etc/nginx/nginx.conf \
    && nginx -t
