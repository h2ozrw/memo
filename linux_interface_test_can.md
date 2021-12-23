#   Linux: Interface Test: CAN

##  Build can-utils

Get can-utils from `https://github.com/linux-can/can-utils`.

Build can-utils.

```
./autogen.sh

# gnu or musl are both okay.
./configure --prefix=/mem/bin --host=aarch64-linux-musl

# build.
make -j 16 && make install

# we use static build of musl, strip is enough.
aarch64-linux-musl-strip *

```

##  Linux Enable CAN Interface.

Check `Documentation/networking/can.rst` for CAN documentation.

```
# baudrate.
ip link set can0 type can bitrate 1000000

# auto re-start time. not mandatory, maybe useful while testing.
ip link set can0 type can restart-ms 100

# enable CAN interface.
ip link set can0 up

# show CAN interface detail statistics.
ip -d -s link show can0
```

##  Test using can-utils.

```
# send packet, to can0, id is 123, data is DEADBEEF.
cansend can0 123#DEADBEEF
```
