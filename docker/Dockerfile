FROM centos:5.11
MAINTAINER The ManyLinux project

# Python versions installed in /opt/python. Could add more, if desired.
ENV PY_VERS="2.7.11 3.4.4 3.5.1"

ENV LC_ALL=en_US.UTF-8
ENV LANG=en_US.UTF-8
ENV LANGUAGE=en_US.UTF-8

# Dependencies for compiling Python that we want to remove from the final
# image after compiling Python
ENV PYTHON_COMPILE_DEPS="zlib-devel bzip2-devel ncurses-devel sqlite-devel readline-devel tk-devel gdbm-devel db4-devel libpcap-devel xz-devel"

# Libraries that are allowed as part of the manylinux1 profile
ENV MANYLINUX1_DEPS="glibc-devel libstdc++-devel glib2-devel libX11-devel libXext-devel libXrender-devel  mesa-libGL-devel libICE-devel libSM-devel ncurses-devel"

# EPEL support
ADD https://dl.fedoraproject.org/pub/epel/5/x86_64/epel-release-5-4.noarch.rpm /
# Dev toolset (for LLVM and other projects requiring C++11 support)
ADD http://people.centos.org/tru/devtools-2/devtools-2.repo /etc/yum.repos.d/
RUN rpm -Uvh --replacepkgs epel-release-5*.rpm && rm -f epel-release-5*.rpm


# Development tools and libraries
RUN yum -y install curl wget bzip2 make git patch unzip bison yasm autoconf automake which file \
                   kernel-devel-`uname -r` \
                   devtoolset-2-binutils devtoolset-2-gcc \
                   devtoolset-2-gcc-c++ devtoolset-2-gcc-gfortran \
                   ${PYTHON_COMPILE_DEPS}
ENV PATH /opt/rh/devtoolset-2/root/usr/bin:$PATH
ENV LD_LIBRARY_PATH /opt/rh/devtoolset-2/root/usr/lib64:/opt/rh/devtoolset-2/root/usr/lib


# Compile the latest Python releases.
# (In order to have a proper SSL module, Python is compiled against openssl-1.0.2e
# which is linked statically, and then we delete openssl afterwards)

RUN export OPENSSL_VERSION=openssl-1.0.2f && \
    export OPENSSL_SHA256=932b4ee4def2b434f85435d9e3e19ca8ba99ce9a065a61524b429a9d5e9b2e9c && \
    echo "${OPENSSL_SHA256}  ${OPENSSL_VERSION}.tar.gz" > ${OPENSSL_VERSION}.tar.gz.sha256 && \
    wget http://www.openssl.org/source/${OPENSSL_VERSION}.tar.gz && \
    sha256sum -c ${OPENSSL_VERSION}.tar.gz.sha256 && \
    tar -xzf ${OPENSSL_VERSION}.tar.gz && \
    (cd ${OPENSSL_VERSION} && ./config no-ssl2 no-shared -fPIC --prefix=/usr/local/ssl && make && make install) && \
    rm -rf ${OPENSSL_VERSION} ${OPENSSL_VERSION}.tar.gz ${OPENSSL_VERSION}.tar.gz.sha256 && \
    for PYVER in ${PY_VERS}; do  \
        wget -q https://www.python.org/ftp/python/$PYVER/Python-$PYVER.tgz && \
        tar -xzf Python-$PYVER.tgz && \
        (cd Python-$PYVER && \
            mkdir -p /opt/$PYVER/lib && \
            LDFLAGS="-Wl,-rpath /opt/$PYVER/lib" ./configure --prefix=/opt/$PYVER --enable-shared --with-ensurepip=install && \
            make -j2 && \
            make install \
        ) && \
        if [[ "$(echo $PYVER | cut -d. -f 1)" == "3" ]]; then \
            ln -s /opt/$PYVER/bin/python3 /opt/$PYVER/bin/python; \
            ln -s /opt/$PYVER/bin/pip3 /opt/$PYVER/bin/pip; \
        fi; \
        ln -s /opt/$PYVER/ /opt/$(echo $PYVER | cut -d. -f 1,2); \
        /opt/$PYVER/bin/pip install pip --upgrade; \
        /opt/$PYVER/bin/pip install wheel; \
        rm -rf Python-$PYVER.tgz Python-$PYVER; \
    done && \
    rm -rf /usr/local/ssl


# Install patchelf and auditwheel (latest)
RUN wget http://nixos.org/releases/patchelf/patchelf-0.8/patchelf-0.8.tar.gz && \
    tar -xzvf patchelf-0.8.tar.gz && \
    (cd patchelf-0.8 && ./configure && make && make install) && \
    rm -rf patchelf-0.8.tar.gz patchelf-0.8 && \
    /opt/3.5/bin/pip install git+git://github.com/manylinux/auditwheel.git && \
    ln -s /opt/3.5/bin/auditwheel /usr/local/bin/auditwheel


# Clean up development headers and other unnecessary stuff for final image
RUN yum -y erase wireless-tools gtk2 libX11 hicolor-icon-theme \
                 avahi freetype bitstream-vera-fonts \
                 ${PYTHON_COMPILE_DEPS}  > /dev/null 2>&1 && \
    yum -y install ${MANYLINUX1_DEPS} && \
    yum -y clean all > /dev/null 2>&1 && \
    yum list installed
