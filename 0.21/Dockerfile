FROM debian:bullseye-slim

LABEL maintainer.0="Loshan (@losh11)"

RUN useradd -r litecoin \
  && apt-get update -y \
  && apt-get install -y curl gnupg gosu \
  && apt-get clean \
  && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

ARG TARGETPLATFORM
ENV LITECOIN_VERSION=0.21.5.5
ENV LITECOIN_DATA=/home/litecoin/.litecoin
ENV PATH=/opt/litecoin-${LITECOIN_VERSION}/bin:$PATH

RUN set -ex \
  && case "${TARGETPLATFORM}" in \
  linux/amd64) LITECOIN_PLATFORM=x86_64-linux-gnu ;; \
  linux/arm64) LITECOIN_PLATFORM=aarch64-linux-gnu ;; \
  linux/arm/v7) LITECOIN_PLATFORM=arm-linux-gnueabihf ;; \
  *) echo "Unsupported platform: ${TARGETPLATFORM}" && exit 1 ;; \
  esac \
  && for key in \
  d35621d53a1cc6a3456758d03620e9d387e55666 \
  ; do \
  gpg --batch --keyserver keyserver.ubuntu.com --recv-keys "$key" || \
  gpg --batch --keyserver pgp.mit.edu --recv-keys "$key" || \
  gpg --batch --keyserver keyserver.pgp.com --recv-keys "$key" || \
  gpg --batch --keyserver ha.pool.sks-keyservers.net --recv-keys "$key" || \
  gpg --batch --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys "$key" ; \
  done \
  && curl -SLO https://download.litecoin.org/litecoin-${LITECOIN_VERSION}/linux/litecoin-${LITECOIN_VERSION}-${LITECOIN_PLATFORM}.tar.gz \
  && curl -SLO https://download.litecoin.org/litecoin-${LITECOIN_VERSION}/SHA256SUMS.asc \
  && gpg --verify SHA256SUMS.asc \
  && grep " litecoin-${LITECOIN_VERSION}-${LITECOIN_PLATFORM}.tar.gz" SHA256SUMS.asc | sha256sum -c - \
  && tar -xzf *.tar.gz -C /opt \
  && rm *.tar.gz *.asc \
  && rm -rf /opt/litecoin-${LITECOIN_VERSION}/bin/litecoin-qt

COPY docker-entrypoint.sh /entrypoint.sh

VOLUME ["/home/litecoin/.litecoin"]

EXPOSE 9332 9333 19332 19335 19443 19444


ENTRYPOINT ["/entrypoint.sh"]

RUN litecoind -version | grep "Litecoin Core version v${LITECOIN_VERSION}"

CMD ["litecoind"]
