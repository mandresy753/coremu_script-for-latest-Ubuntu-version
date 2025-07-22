#!/bin/bash
set -e

echo "Mise à jour des paquets..."
sudo apt update -y

echo "Installation des dépendances système..."
sudo apt install -y \
    ca-certificates \
    xterm \
    psmisc \
    python3 \
    python3-tk \
    python3-pip \
    python3-venv \
    wget \
    git \
    unzip \
    automake \
    gawk \
    g++ \
    libreadline-dev \
    libtool \
    make \
    pkg-config \
    iproute2 \
    iputils-ping \
    tcpdump \
    libpcap-dev \
    libpcre3-dev \
    libxml2-dev \
    uuid-dev \
    net-tools \
    libffi-dev \
    libssl-dev \
    python3-dev \
    python3-wheel \
    python3-setuptools \
    libsqlite3-dev \
    liblzma-dev

echo "Suppression de protobuf incompatible..."
sudo apt remove --purge -y libprotobuf-dev libprotobuf-lite* libprotobuf* protobuf-compiler || true
sudo apt autoremove -y

echo "Téléchargement et installation de protobuf 3.19.6..."
cd ~/Documents
wget -q https://github.com/protocolbuffers/protobuf/releases/download/v3.19.6/protobuf-all-3.19.6.tar.gz
tar -xzf protobuf-all-3.19.6.tar.gz
cd protobuf-3.19.6
./configure --prefix=/usr/local
make -j$(nproc)
sudo make install
sudo ldconfig

echo "protoc version installée : $(protoc --version)"

echo "Clonage et compilation d'OSPF-MDR..."
cd ~/Documents
git clone https://github.com/USNavalResearchLaboratory/ospf-mdr.git
cd ospf-mdr
./bootstrap.sh
./configure --disable-doc --enable-user=root --enable-group=root \
    --with-cflags=-ggdb --sysconfdir=/usr/local/etc/quagga --enable-vtysh \
    --localstatedir=/var/run/quagga
make -j$(nproc)
sudo make install

echo "Téléchargement et installation d'EMANE (1.5.1)..."
cd ~/Documents
EMANE_RELEASE=emane-1.5.1-release-1
EMANE_PACKAGE=${EMANE_RELEASE}.ubuntu-22_04.amd64.tar.gz
wget -q https://adjacentlink.com/downloads/emane/${EMANE_PACKAGE}
tar xf ${EMANE_PACKAGE}
cd ${EMANE_RELEASE}/debs/ubuntu-22_04/amd64
rm emane-spectrum-tools*.deb emane-model-lte*.deb
rm *dev*.deb
sudo apt install -y ./emane*.deb ./python3-emane_*.deb

echo "Installation des bindings Python EMANE"
cd ~/Documents
git clone https://github.com/adjacentlink/emane.git
cd emane
git checkout v1.5.1
./autogen.sh
./configure --prefix=/usr/local

echo "Nettoyage des anciens fichiers .pb.*..."
find . -name "*.pb.h" -delete
find . -name "*.pb.cc" -delete

echo "Regénération des fichiers .proto..."
cd src/libemane
protoc *.proto --cpp_out=.

echo "Compilation EMANE depuis les sources..."
cd ~/Documents/emane
make -j$(nproc)
sudo make install
sudo ldconfig

echo "Installation de CORE (coreemu)..."
cd ~/Documents
CORE_VERSION=9.2.1
CORE_PACKAGE=core_${CORE_VERSION}_amd64.deb
wget -q https://github.com/coreemu/core/releases/download/release-${CORE_VERSION}/${CORE_PACKAGE}
sudo apt install -y ./${CORE_PACKAGE}

echo "Installation terminée avec succès."
echo "Tu peux maintenant tester avec : emane --version ou lancer core-gui"

