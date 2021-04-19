FROM ubuntu:18.04

ENV TENGINE_VERSION 2.3.3
ENV DEBIAN_FRONTEND noninteractive

WORKDIR /root/

RUN apt update && \
    apt install -y gcc make git vim curl wget gzip tar zlib1g zlib1g-dev libssl-dev libpcre3 libpcre3-dev libgeoip-dev libgeoip1 zlib1g zlib1g-dev tzdata && \
    cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && echo "Asia/Shanghai" > /etc/timezone && \
    wget http://tengine.taobao.org/download/tengine-${TENGINE_VERSION}.tar.gz && \
    wget https://kkb-op.oss-cn-beijing.aliyuncs.com/kkb-test/openssl-OpenSSL_1_1_1b.tar.gz && \
    git clone https://github.com/vozlt/nginx-module-vts.git && \
    mkdir -p /usr/local/nginx && mkdir -p /var/log/nginx && \
    tar xf tengine-${TENGINE_VERSION}.tar.gz && tar xf openssl-OpenSSL_1_1_1b.tar.gz && \
    cd tengine-${TENGINE_VERSION} && ./configure --prefix=/usr/local/nginx \
        --with-http_gzip_static_module \
        --with-pcre \
        --with-http_ssl_module \
        --with-http_realip_module \
        --with-http_mp4_module \
        --with-http_v2_module \
        --with-http_sub_module \
        --with-http_geoip_module \
        --add-module=modules/ngx_http_upstream_check_module \
        --with-openssl=/root/openssl-OpenSSL_1_1_1b \
        --add-module=/root/nginx-module-vts && \
    make -j4 && make install && \
    rm -rf /usr/share/GeoIP/* && \
    curl --create-dirs -fsSLko /usr/share/GeoIP/GeoIP.dat https://github.com/kubernetes/ingress-nginx/raw/master/images/nginx/rootfs/etc/nginx/geoip/GeoIP.dat && \
    curl --create-dirs -fsSLko /usr/share/GeoIP/GeoIPASNum.dat https://github.com/kubernetes/ingress-nginx/raw/master/images/nginx/rootfs/etc/nginx/geoip/GeoIPASNum.dat && \
    curl --create-dirs -fsSLko /usr/share/GeoIP/GeoLiteCity.dat https://github.com/kubernetes/ingress-nginx/raw/master/images/nginx/rootfs/etc/nginx/geoip/GeoLiteCity.dat && \
    cp /usr/local/nginx/sbin/nginx /usr/local/sbin/ && \
    apt remove gcc make git -y && apt-get clean && \
    rm -rf /root/* /var/lib/apt/lists/* /var/cache/apt/archives/*

WORKDIR /usr/local/nginx/conf/

STOPSIGNAL SIGQUIT

CMD ["nginx", "-g", "daemon off;"]

# 本次版本
# docker build -t registry.cn-beijing.aliyuncs.com/aoisola/nginx-server:base-2.0 .
