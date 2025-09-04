+++
title = 'Build'
date = 2025-09-04T11:50:15+08:00
draft = false
+++

## build openwrt on debian 13 (trixie)

### install dependencies

```bash
sudo apt update
sudo apt install -y ack antlr3 asciidoc autoconf automake autopoint binutils bison build-essential \
bzip2 ccache clang cmake cpio curl device-tree-compiler flex gawk gettext \
genisoimage git gperf haveged help2man intltool libelf-dev libfuse-dev libglib2.0-dev \
libgmp3-dev libltdl-dev libmpc-dev libmpfr-dev libncurses5-dev libncursesw5-dev libpython3-dev \
libreadline-dev libssl-dev libtool llvm lrzsz msmtp ninja-build p7zip p7zip-full patch pkgconf \
python3  python3-pyelftools python3-setuptools qemu-utils rsync scons squashfs-tools subversion \
swig texinfo uglifyjs upx-ucl unzip vim wget xmlto xxd zlib1g-dev \
python3-dev python-is-python3 libdebuginfod-dev systemtap-sdt-dev
```

### clone openwrt

```bash
git clone https://github.com/openwrt/openwrt.git
cd openwrt
git checkout openwrt-24.10
git pull
```

### build openwrt

```bash
wget https://downloads.openwrt.org/releases/24.10.0/targets/x86/64/config.buildinfo -O .config
make defconfig
make download -j$(nproc)
make -j$(nproc)
```