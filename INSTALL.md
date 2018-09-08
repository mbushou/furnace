Furnace Installation Instructions
=======

Introduction
-------

These instructions install Furance on a single server/hypervisor (Fedora on Xen).  At the conclusion of these instructions, Furnace apps can be run manually from the command line using a included set of zsh functions.  Integration with a cloud management platform (OpenStack, etc) are then needed (not presently included) to expose Furnace functionality to tenants.

Furnace requires modern hardware suitable for VMI (read DRAKVUF's
or LibVMI's documentation on verifying hardware).

These instructions are spartan; they are best used by copy/pasting small blocks of commands at a time, ensuring each completes successfully before continuing.

For more information on the Furnace project, visit its [main repository](https://github.ncsu.edu/mbushou/furnace).

### Install Fedora 28 server and Xen

```
# on a fresh Fedora 28 server install:
sudo dnf upgrade --refresh -y
sudo reboot
sudo dnf install xen xen-devel -y
# add altp2m to /etc/default/grub
echo "GRUB_CMDLINE_XEN_DEFAULT=\"altp2m\"" | sudo tee -a /etc/default/grub
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
sudo reboot
# choose Fedora on Xen in grub
# ensure Fedora boots as dom0

# later on, shell scripting functions are introduced
# these are built against the zsh shell, to install and make default:
sudo dnf install zsh -y
chsh -s zsh
```

### Determine location for main directories
(note these should be in a system directory (i.e., not a home directory) because they will be accessed by the `furnace` system user)

```
FURNACE_DIR=/opt/furnace
SANDBOX_DIR=$FURNACE_DIR/furnace_sandbox
PROXY_DIR=$FURNACE_DIR/furnace_proxy
APP_STORAGE_DIR=$FURNACE_DIR/app_storage
APP_FILESYSTEM_DIR=$FURNACE_DIR/app_filesystem
VMI_FILESYSTEM_DIR=$FURNACE_DIR/vmi_filesystem
DRAKVUF_DIR=$FURNACE_DIR/drakvuf
MOCK_DIR=$SANDBOX_DIR/vmi_partition_mock

sudo mkdir -p $SANDBOX_DIR
sudo mkdir $PROXY_DIR
sudo mkdir $APP_FILESYSTEM_DIR
sudo mkdir $APP_STORAGE_DIR/tenant_data
sudo mkdir $VMI_FILESYSTEM_DIR
sudo mkdir $DRAKVUF_DIR

# current user gets permissions across Furnace, change this to taste.
sudo chown -R $USER $FURNACE_DIR
```

### Clone Furnace
```
git clone https://github.com/mbushou/furnace_sandbox $SANDBOX_DIR
git clone https://github.com/mbushou/furnace_proxy $PROXY_DIR
```

### Configure DRAKVUF
```
# This generally follows DRAKVUF's installation instructions, but cuts short recompiling Xen.
sudo dnf groupinstall "Development Tools" -y
sudo dnf install clang libtool glib2-devel bison flex json-c-devel protobuf protobuf-devel zeromq-devel autoconf-archive
# last tested against DRAKVUF commit 064450a made on 7/18/18
git clone https://github.com/tklengyel/drakvuf $DRAKVUF_DIR
cd drakvuf
git submodule init
git submodule update
# Note: if following DRAKVUF instructions, do not do the Xen recompile
```

##### Install LibVMI

```
cd $DRAKVUF_DIR/libvmi
./autogen.sh
./configure --disable-kvm
make
sudo make install

export PKG_CONFIG_PATH=$DRAKVUF_DIR/libvmi
```

##### Patch and compile DRAKVUF
```
cd $SANDBOX_DIR/drakvuf_patches
ln -s $DRAKVUF_DIR drakvuf

patch ./drakvuf/configure.ac -i configure.ac.patch
patch ./drakvuf/src/plugins/Makefile.am -i Makefile.am.patch
patch ./drakvuf/src/plugins/plugins.cpp -i plugins.cpp.patch
patch ./drakvuf/src/plugins/plugins.h -i plugins.h.patch
mkdir ./drakvuf/src/plugins/furnace
cp ../vmi_partition_drakvuf/furnace.* ./drakvuf/src/plugins/furnace/.

cd $DRAKVUF_DIR
autoreconf -vi
./configure --enable-debug
# add ZMQ and protobuf libraries
sed -i '/.*lpthread.*/ s/$/-lprotobuf -lzmq/' src/plugins/Makefile

# enable syscall tracing? copy over syscalls.cpp|h as well
# warning: this is likely an old version of the syscalls plugin
# it may be incompatible with upstream DRAKVUF
# you may have to patch in Furnace compatibility manually

# not going to use syscall tracing? comment out line 740 of furnace.cpp
```

##### Compile DRAKVUF plugin protobuf
```
cp $SANDBOX_DIR/ipc_format/vmi.proto $DRAKVUF_DIR/src/plugins/furnace/.
cd $DRAKVUF_DIR/src/plugins/furnace
protoc --proto_path=. --cpp_out=. *.proto
```

##### Compile DRAKVUF
```
cd $DRAKVUF_DIR
make
```

##### OPTIONAL: For local testing in lieu of DRAKVUF, compile and install mock VMI driver
```
cd $MOCK_DIR
mkdir bin
protoc --proto_path=../ipc_format --cpp_out=. ../ipc_format/vmi.proto
protoc --proto_path=../ipc_format --python_out=. ../ipc_format/vmi.proto
g++ -g -lprotobuf -c vmi.pb.cc
g++ -g -Wno-write-strings -Wall `pkg-config --cflags --libs glib-2.0` -lpthread -lzmq -lprotobuf mock_driver.c vmi.pb.o vmi_runtime.c -o bin/vmi_runtime
```

### Add Furnace system account
```
sudo adduser --system -M -s /sbin/nologin furnace
```

### Copy testing partitions included in the sandbox repo to $APP_STORAGE_DIR
```
cp -r $SANDBOX_DIR/tenant_data $APP_STORAGE_DIR/tenant_data
sudo chown -R furnace:furnace $APP_STORAGE_DIR/tenant_data
# rekall.json is in .gitignore, so we'll add a dummy rekall.json to the source tree here
sudo -u furnace touch $APP_STORAGE_DIR/tenant_data/22222222-2222-2222-2222-222222222222-vmi/rekall.json
```

### Make file systems
```
# set up partition's Python environment
cd $SANDBOX_DIR
python3 -m venv site
./site/bin/pip install zmq protobuf

# set up app partition (fac also uses this)
sudo dnf install --releasever=28 --repo=fedora --repo=updates --installroot=$APP_FILESYSTEM_DIR python3 libseccomp libstdc++ python3-protobuf zeromq libexplain libexplain-devel strace
cd $APP_FILESYSTEM_DIR
sudo touch ./dev/urandom
sudo mkdir -p ./opt/furnace
sudo chown -R furnace:furnace ./opt/furnace
sudo -u furnace mkdir ./opt/furnace/fac_socket
sudo -u furnace mkdir ./opt/furnace/vmi_socket
sudo -u furnace mkdir ./opt/furnace/shared
sudo -u furnace mkdir ./opt/furnace/keys
sudo -u furnace touch ./opt/furnace/keys/app.key
sudo -u furnace touch ./opt/furnace/keys/be.pub
sudo -u furnace mkdir ./opt/furnace/facilities
sudo -u furnace mkdir ./opt/furnace/app
sudo -u furnace touch ./opt/furnace/app/app.py
sudo -u furnace mkdir ./opt/furnace/venv
sudo -u furnace mkdir ./opt/furnace/frontend

# set up vmi partition
sudo dnf install --releasever=28 --repo=fedora --repo=updates --installroot=$VMI_FILESYSTEM_DIR libseccomp libstdc++ zeromq libexplain libexplain-devel bzip2-libs glib2 glibc json-c keyutils-libs krb5-libs libcap libcom_err libgcc libgcrypt libgpg-error libnl3 libselinux libstdc++ libuuid lz4-libs openpgm pcre systemd-libs xen-libs xz-libs yajl zlib strace
cd $VMI_FILESYSTEM_DIR
sudo touch ./dev/urandom
sudo mkdir -p ./opt/furnace
sudo chown -R furnace:furnace ./opt/furnace
sudo -u furnace mkdir ./opt/furnace/fac_socket
sudo -u furnace mkdir ./opt/furnace/vmi_socket
sudo -u furnace mkdir ./opt/furnace/shared
sudo -u furnace mkdir ./opt/furnace/keys
sudo -u furnace mkdir ./opt/furnace/vmi
sudo mkdir -p ./opt/kernels/rekall
sudo touch ./opt/kernels/rekall/rekall.json
sudo chown -R furnace:furnace ./opt/kernels
sudo mkdir ./dev/xen

# if using DRAKVUF, all of the following is required
# if using the mock VMI driver, only the following libprotobuf copy is required

sudo cp /usr/lib64/libprotobuf.so.15 $VMI_FILESYSTEM_DIR/usr/lib64/.

sudo cp $DRAKVUF_DIR/libvmi/libvmi/.libs/libvmi.so.0.0.13 $VMI_FILESYSTEM_DIR/usr/lib64/libvmi.so.0

sudo cp /usr/lib64/libxenctrl.so.4.10.0 $VMI_FILESYSTEM_DIR/usr/lib64/libxenctrl.so
sudo cp /usr/lib64/libxencall.so.1.0 $VMI_FILESYSTEM_DIR/usr/lib64/libxencall.so
sudo cp /usr/lib64/libxenevtchn.so.1.1 $VMI_FILESYSTEM_DIR/usr/lib64/libxenevtchn.so
sudo cp /usr/lib64/libxenforeignmemory.so.1.2 $VMI_FILESYSTEM_DIR/usr/lib64/libxenforeignmemory.so
sudo cp /usr/lib64/libxenstore.so.3.0.3 $VMI_FILESYSTEM_DIR/usr/lib64/libxenstore.so
sudo cp /usr/lib64/libxenvchan.so.4.10.0 $VMI_FILESYSTEM_DIR/usr/lib64/libxenvchan.so
```

### Compile ptracers
```
sudo dnf install libexplain-devel libseccomp-devel
cd $SANDBOX_DIR/syscall_inspector
make all
mkdir ../shared/bin
cp *_inspector ../shared/bin/.
```

### Source Furnace's convenience shell commands (if using zsh)
```
source $SANDBOX_DIR/misc/zsh_functions
```

### Install supporting packages (cgroups, bwrap)
```
sudo dnf install libcgroup-tools bubblewrap
# compile protobufs
cd $SANDBOX_DIR/ipc_format
protoc --proto_path=. --python_out=. *.proto
ln facilities_pb2.py ../shared/.
ln vmi_pb2.py ../shared/.
```

### Test by starting the partitions
```
# start the mock driver in a VMI partition
mvmi fg25-1 22222222-2222-2222-2222-222222222222-vmi
# start an app partition
papp 11111111-1111-1111-1111-111111111111-app \
22222222-2222-2222-2222-222222222222-vmi no_metadata
# start a fac partition
pfac 11111111-1111-1111-1111-111111111111-app 127.0.0.1 5561
```

### The `tenant_data` directory
* Tenant apps are stored in the $SANDBOX_DIR/tenant_data directory.  This
  directory is populated with an example app.
  The `1*1-app` directory is used by the app's APP and FAC partitions, while the
  `2*2-vmi` director is for its matching VMI partition.

```
$ find $SANDBOX_DIR/tenant_data -type f
tenant_data/11111111-1111-1111-1111-111111111111-app/be.key
tenant_data/11111111-1111-1111-1111-111111111111-app/be.pub
tenant_data/11111111-1111-1111-1111-111111111111-app/app.key
tenant_data/11111111-1111-1111-1111-111111111111-app/app.pub
tenant_data/11111111-1111-1111-1111-111111111111-app/fac_socket/fac_sub
tenant_data/11111111-1111-1111-1111-111111111111-app/fac_socket/fac_rtr
tenant_data/11111111-1111-1111-1111-111111111111-app/app.py
tenant_data/22222222-2222-2222-2222-222222222222-vmi/rekall.json
tenant_data/22222222-2222-2222-2222-222222222222-vmi/vmi_socket/vmi_rtr

# note, if rekall.json is not present, run:
# sudo -u furnace touch $SANDBOX_DIR/tenant_data/22222222-2222-2222-2222-222222222222-vmi/rekall.json
```

* `app.py` is the tenant app itself.  Used by the APP partition.
* `be.{key,pub}` and `app.{key.pub}` are ZMQ CURVE keypairs to encrypt data
  between the FAC partition, through the message proxy, and to the tenant's backend.
* The files in `fac_socket` are the sockets used to communicate (via ZMQ) between the APP
  and FAC partitions.
* `rekall.json` is used by VMI to lookup object offsets in the guest kernel.
* The file in `vmi_socket` is used to communicate (via ZMQ) between the APP and
  VMI partitions.

* Each Furnace app has its own directory.  The convention here is `UUID4-app`
  for apps.
* The VMI partition supports multiple apps interacting with it at once. Therefore it has its own directory with the convention is `UUID4-vmi`.

### Install your own apps
* Note: The Furnace hypervisor agent will (eventually) be responsible for
  staging tenant apps and starting them in partitions.
* Until then, Furnace must be used manually by staging tenant app data and starting
  partitions via the shell.
* Create and place ZMQ CURVE keys in `$SANDBOX_DIR/tenant_data/[UUID-app]` and `$PROXY_DIR/tenant_data/[UUID-app]`
*

### Install the Furnace backend
* See the furnace_backend repo for instructions
* This is inteded to be performed by a tenant on their remote system.
* For testing, it is best done on the hypervisor itself.

### Perform an end-to-end test
```
# start the VMI partition (it will idle until the app starts)
pmvmi fg25-1 \
22222222-2222-2222-2222-222222222222-vmi

# start the proxy (it will idle until the app starts)
furnace_proxy 11111111-1111-1111-1111-111111111111-app \
127.0.0.1 5561 127.0.0.1 5563

# start the FAC partition (it will idle until the app starts)
pfac 11111111-1111-1111-1111-111111111111-app \
127.0.0.1 5561

# start the backend (it will idle until the app starts)
# see furnace_backend repository for more information
./bemain.py -d \
    -c app_be \
    -m AppBE \
    --ep 127.0.0.1 \
    --et 5563 \
    --ak furnace_app_keypair_1.key_secret \
    --bk furnace_be_keypair.key_secret

# start the APP partition
papp 11111111-1111-1111-1111-111111111111-app \
22222222-2222-2222-2222-222222222222-vmi \
no_metadata
```


Troubleshooting
-------
- If using the zsh functions, removing the `p` in `pvmi` will run the VMI
  partition without
  the syscall inspector.  To poke around the inside of the partition manually, replacing
  the `p` with a `b` (e.g., `bvmi`) will invoke bash inside a new partition.
  This also works for `fac`/`pfac`/`bfac` and `app`/`papp`/`bapp`.

- Getting a ZMQ "already in use" error at runtime?  Make sure each tenant_data directory's vmi_socket and fac_socket subdirectories are chowned to furnace:furnace.

- Is a syscall inspector killing its partition when it perhaps shouldn't? Try tweaking its syscall policy.  In the relevant partition, bind a directory to which strace output can be written. Then, start the bash version of the partition (bapp, bfac, or bvmi---see ./misc/zsh_functions), then start the Furnace component behind strace instead of the syscall inspector.
By comparing the earlier crash output from the syscall inspector and aligning it with the strace output, look for a new syscall or different argument evident in the strace output at around the place the ptracer kills the partition.
```
strace -fo /tmp/app.txt /opt/furnace/venv/bin/python3 -EOO /opt/furnace/frontend/apps/femain.py -c app -m App --u2 123-123-123-app
```

```
strace -fo /tmp/trace.txt /opt/furnace/vmi/src/drakvuf -r /opt/kernels/rekall/rekall.json -x socketmon -x cpuidmon -x debugmon -x exmon -x filedelete -x filetracer -x objmon -x poolmon -x ssdtmon -x regmon -x syscalls -v -d fg25-1
```

