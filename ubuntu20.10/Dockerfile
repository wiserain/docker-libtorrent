FROM ubuntu:20.10 AS ubuntu
FROM ubuntu AS build-base

ARG PY_MAJOR_VER
ARG LIBTORRENT_VER
ARG TARGETARCH
ARG DEBIAN_FRONTEND="noninteractive"

RUN \
    echo "**** install build-deps ****" && \
    apt-get update && \
    apt-get install -yq \
        autoconf \
        automake \
        build-essential \
        git \
        libboost-chrono-dev \
        libboost-python-dev \
        libboost-random-dev \
        libboost-system-dev \
        libssl-dev \
        libtool \
        pkg-config

RUN \
    echo "**** clone source ****" && \
    git clone https://github.com/arvidn/libtorrent.git /tmp/libtorrent -b "v${LIBTORRENT_VER}" --depth 1

RUN \
    echo "**** install python-deps ****" && \
    if [ $PY_MAJOR_VER = "2" ]; then \
        pydeps="python-setuptools python-all-dev"; \
    elif [ $PY_MAJOR_VER = "3" ]; then \
        pydeps="python3-setuptools python3-all-dev"; \
    fi && \
    apt-get install -yq ${pydeps}

RUN \
    echo "**** build libtorrent ****" && \
    PYTHON_VERSION=$(python${PY_MAJOR_VER} -c 'import sys; print(".".join([str(x) for x in sys.version_info[:2]]))') && \
    _PYTHON_VERSION="$(echo "${PYTHON_VERSION}" | sed 's/\.//g')" && \
    PYTHON_INSTALL_LIBDIR="$(python${PY_MAJOR_VER} -c 'import sys; print(sys.path[-1]);' | sed 's/\/usr//g')" && \
    if [ $TARGETARCH = "amd64" ]; then \
        BOOST_LIB_DIR="x86_64-linux-gnu"; \
    elif [ $TARGETARCH = "arm64" ]; then \
        BOOST_LIB_DIR="aarch64-linux-gnu"; \
    elif [ $TARGETARCH = "arm" ]; then \
        BOOST_LIB_DIR="arm-linux-gnueabihf"; \
    fi && \
    cd /tmp/libtorrent && \
    PYTHON=/usr/bin/python${PYTHON_VERSION} \
    PYTHON_INSTALL_PARAMS='--prefix=$(DESTDIR)$(prefix) --install-lib=$(DESTDIR)$(prefix)'${PYTHON_INSTALL_LIBDIR} \
    ./bootstrap.sh \
        --prefix=/usr \
        --libdir="/usr/lib/${BOOST_LIB_DIR}" \
        --enable-python-binding \
        --disable-static \
        --with-libiconv \
        --with-boost-python="boost_python${_PYTHON_VERSION}" \
        --with-boost-libdir="/usr/lib/${BOOST_LIB_DIR}" && \
    make VERBOSE=1 -j$(nproc) && \
    make DESTDIR=/libtorrent-build install

# 
# TEST
# 
FROM ubuntu AS test

ARG LIBTORRENT_VER
ARG PY_MAJOR_VER
ARG TARGETARCH
ARG DEBIAN_FRONTEND="noninteractive"

COPY --from=build-base /libtorrent-build/usr/lib/ /usr/lib/

RUN \
    echo "**** install runtime packages ****" && \
    apt-get update && \
    apt-get install -yq \
        libboost-chrono-dev \
        libboost-python-dev \
        libboost-random-dev \
        libboost-system-dev \
        libboost-tools-dev \
        python$(if [ ${PY_MAJOR_VER} = "3" ]; then echo 3; fi)-minimal

RUN \
    BUILD_VER=$(python${PY_MAJOR_VER} -c 'import libtorrent as lt; print(lt.version)') && \
    if [ $LIBTORRENT_VER = ${BUILD_VER%.*} ]; then \
        echo "Successfully built with version: ${BUILD_VER}"; \
    else \
        echo "Something went wrong: ${BUILD_VER}"; \
        exit 1; \
    fi

# 
# RELEASE
# 
FROM ubuntu
LABEL maintainer "wiserain"
COPY --from=build-base /libtorrent-build/usr/lib/ /libtorrent-build/usr/lib/