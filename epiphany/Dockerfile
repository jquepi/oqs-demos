# Default location where all binaries wind up:
ARG OSSLGIODIR=/opt/ossl-gio

# 2-step build: First create binaries
FROM ubuntu as intermediate
ARG OSSLGIODIR
ENV DEBIAN_FRONTEND noninteractive

# everything to build liboqs, oqs-openssl, glib-networking:

RUN apt update && apt upgrade -y && apt install -y build-essential clang meson gnome-pkg-tools libglib2.0-dev libproxy-dev gsettings-desktop-schemas-dev ca-certificates epiphany-browser libtool make gcc ninja-build libssl-dev cmake libtool wget

WORKDIR /opt

RUN git clone --branch main https://github.com/open-quantum-safe/liboqs.git && git clone --depth 1 --branch OQS-OpenSSL_1_1_1-stable https://github.com/open-quantum-safe/openssl ossl-src && git clone https://gitlab.gnome.org/GNOME/glib-networking.git

# make sure the liboqs library is distributable:
RUN cd liboqs && mkdir build && cd build && cmake -GNinja -DOQS_DIST_BUILD=ON -DCMAKE_INSTALL_PREFIX=/opt/ossl-src/oqs .. && ninja && ninja install && cd ../..

RUN cd ossl-src && ./config no-shared -lm && make -j 2 && make install_sw && cd ..

# build glib-networking such as to use (OQS-)OpenSSL and not GnuTLS:
RUN cd glib-networking && git checkout 2.60.4 && mkdir build && cd build && PKG_CONFIG_PATH=${OSSLGIODIR}/lib/pkgconfig CPATH=${OSSLGIODIR}/include LIBRARY_PATH=${OSSLGIODIR}/lib  meson --prefix=${OSSLGIODIR} -Dopenssl=enabled -Dgnutls=disabled .. && CPATH=${OSSLGIODIR}/include ninja && ninja install

# obtain and register test.openquantumsafe.org and letsencrypt certs
COPY certlink.sh /usr/local/ssl/certs/certlink.sh
RUN cd /usr/local/ssl/certs && wget https://letsencrypt.org/certs/isrgrootx1.pem && wget https://test.openquantumsafe.org/CA.crt -O oqsrootca.pem && ./certlink.sh isrgrootx1.pem && ./certlink.sh oqsrootca.pem

# 2nd build step: Only retain what's necessary:
FROM ubuntu
ARG OSSLGIODIR
ENV DEBIAN_FRONTEND noninteractive

RUN apt update && apt upgrade -y && apt install -y epiphany-browser 

COPY --from=intermediate ${OSSLGIODIR} ${OSSLGIODIR}
COPY --from=intermediate /usr/local /usr/local

# Run everything under a limited user account:
RUN groupadd -g 1000 oqs && useradd -u 1000 -d /home/oqs -g oqs oqs 
COPY openssl-client.cnf /home/oqs/openssl-client.cnf
COPY startepiphany.sh /home/oqs/startepiphany.sh
RUN chown -R oqs.oqs /home/oqs

USER oqs

ENV OPENSSL_CONF=/home/oqs/openssl-client.cnf
ENV LD_LIBRARY_PATH=${OSSLGIODIR}/lib
ENV GIO_MODULE_DIR=${OSSLGIODIR}/lib/x86_64-linux-gnu/gio/modules

ENTRYPOINT [ "/home/oqs/startepiphany.sh" ]
STOPSIGNAL SIGTERM
