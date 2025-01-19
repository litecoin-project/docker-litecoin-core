# Build stage for BerkeleyDB
FROM alpine as berkeleydb

RUN sed -i 's/http\:\/\/dl-cdn.alpinelinux.org/https\:\/\/alpine.global.ssl.fastly.net/g' /etc/apk/repositories
RUN apk --no-cache add autoconf
RUN apk --no-cache add automake
RUN apk --no-cache add build-base
RUN apk --no-cache add libressl

ENV BERKELEYDB_VERSION=db-4.8.30.NC
ENV BERKELEYDB_PREFIX=/opt/${BERKELEYDB_VERSION}

RUN wget https://download.oracle.com/berkeley-db/${BERKELEYDB_VERSION}.tar.gz
RUN tar -xzf *.tar.gz
RUN sed s/__atomic_compare_exchange/__atomic_compare_exchange_db/g -i ${BERKELEYDB_VERSION}/dbinc/atomic.h
RUN mkdir -p ${BERKELEYDB_PREFIX}

WORKDIR /${BERKELEYDB_VERSION}/build_unix

RUN CFLAGS=-Wno-error=implicit-function-declaration ../dist/configure --enable-cxx --disable-shared --with-pic --prefix=${BERKELEYDB_PREFIX}
RUN make -j4
RUN make install
RUN rm -rf ${BERKELEYDB_PREFIX}/docs

# Build stage for Litecoin Core
FROM alpine as litecoin-core

COPY --from=berkeleydb /opt /opt

RUN sed -i 's/http\:\/\/dl-cdn.alpinelinux.org/https\:\/\/alpine.global.ssl.fastly.net/g' /etc/apk/repositories
RUN apk --no-cache add autoconf
RUN apk --no-cache add automake
RUN apk --no-cache add boost-dev
RUN apk --no-cache add build-base
RUN apk --no-cache add chrpath
RUN apk --no-cache add file
RUN apk --no-cache add fmt-dev
RUN apk --no-cache add gnupg
RUN apk --no-cache add libevent-dev
RUN apk --no-cache add libressl
RUN apk --no-cache add libressl-dev
RUN apk --no-cache add libtool
RUN apk --no-cache add linux-headers
RUN apk --no-cache add zeromq-dev
RUN set -ex \
  && for key in \
  d35621d53a1cc6a3456758d03620e9d387e55666 \
  ; do \
  gpg --batch --keyserver keyserver.ubuntu.com --recv-keys "$key" || \
  gpg --batch --keyserver pgp.mit.edu --recv-keys "$key" || \
  gpg --batch --keyserver keyserver.pgp.com --recv-keys "$key" || \
  gpg --batch --keyserver ha.pool.sks-keyservers.net --recv-keys "$key" || \
  gpg --batch --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys "$key" ; \
  done

ENV LITECOIN_VERSION=0.21.4
ENV LITECOIN_PREFIX=/opt/litecoin-${LITECOIN_VERSION}

RUN wget https://download.litecoin.org/litecoin-${LITECOIN_VERSION}/SHA256SUMS.asc
RUN wget https://download.litecoin.org/litecoin-${LITECOIN_VERSION}/src/litecoin-${LITECOIN_VERSION}.tar.gz
RUN gpg --verify SHA256SUMS.asc
RUN grep " litecoin-${LITECOIN_VERSION}.tar.gz\$" SHA256SUMS.asc | sha256sum -c -
RUN tar -xzf *.tar.gz

WORKDIR /litecoin-${LITECOIN_VERSION}

RUN sed -i '/AC_PREREQ/a\AR_FLAGS=cr' src/univalue/configure.ac
RUN sed -i '/AX_PROG_CC_FOR_BUILD/a\AR_FLAGS=cr' src/secp256k1-zkp/configure.ac
RUN sed -i s:sys/fcntl.h:fcntl.h: src/compat.h

# missing includes required by alpine compiler
RUN sed -i '/#include <stdint.h>/a #include <stdexcept>' src/support/lockedpool.h
RUN sed -i '/#include <attributes.h>/a #include <cstdint>' src/util/bip32.h
RUN sed -i '/#include <cstring>/a #include <cstdint>' src/util/string.h
# fixes overflow causing segfault
RUN sed -i '/char\ scratchpad\[SCRYPT_SCRATCHPAD_SIZE\];/a\memset(scratchpad, 0, sizeof(scratchpad));' src/crypto/scrypt.cpp
RUN sed -i 's/char\ scratchpad\[SCRYPT_SCRATCHPAD_SIZE\];/thread_local &/g' src/crypto/scrypt.cpp
# update filesystem to v1.2.2: fixes alpine build error
RUN wget -O src/libmw/deps/ghc/include/ghc/filesystem.hpp https://github.com/gulrak/filesystem/releases/download/v1.2.2/filesystem.hpp

RUN ./autogen.sh
RUN ./configure LDFLAGS=-L`ls -d /opt/db*`/lib/ CPPFLAGS=-I`ls -d /opt/db*`/include/ \
  --prefix=${LITECOIN_PREFIX} \
  --mandir=/usr/share/man \
  --disable-tests \
  --disable-bench \
  --disable-ccache \
  --with-gui=no \
  --with-utils \
  --with-libs \
  --with-daemon
RUN make -j4
RUN make install
RUN strip ${LITECOIN_PREFIX}/bin/litecoin-cli
RUN strip ${LITECOIN_PREFIX}/bin/litecoin-tx
RUN strip ${LITECOIN_PREFIX}/bin/litecoind
RUN strip ${LITECOIN_PREFIX}/lib/libbitcoinconsensus.a
RUN strip ${LITECOIN_PREFIX}/lib/libbitcoinconsensus.so.0.0.0

# Build stage for compiled artifacts
FROM alpine

LABEL maintainer.0="Loshan (@losh11)"

RUN adduser -S litecoin
RUN sed -i 's/http\:\/\/dl-cdn.alpinelinux.org/https\:\/\/alpine.global.ssl.fastly.net/g' /etc/apk/repositories
RUN apk --no-cache add \
  boost-filesystem \
  boost-system \
  boost-thread \
  libevent \
  libzmq \
  su-exec \
  fmt-dev \
  libressl-dev

ENV LITECOIN_DATA=/home/litecoin/.litecoin
ENV LITECOIN_VERSION=0.21.4
ENV LITECOIN_PREFIX=/opt/litecoin-${LITECOIN_VERSION}
ENV PATH=${LITECOIN_PREFIX}/bin:$PATH

COPY --from=litecoin-core /opt /opt
COPY docker-entrypoint.sh /entrypoint.sh

VOLUME ["/home/litecoin/.litecoin"]

EXPOSE 9332 9333 19332 19335 19443 19444

ENTRYPOINT ["/entrypoint.sh"]

RUN litecoind -version | grep "Litecoin Core version v${LITECOIN_VERSION}"

CMD ["litecoind"]
