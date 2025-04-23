Vivado 2018.2 installation in a container
==================================

1. Create Dockerfile in a separate folder with the following content:
    ```
    # With ubuntu 22.04 vivado 2018.2 exits quickly after start
    # Ubuntu 20.04 seems to be more stable
    FROM ubuntu:20.04

    ARG DEBIAN_FRONTEND=noninteractive
    ENV TZ=Europe/London
    # gedit is useful to test that graphics passthrough is working
    # iproute2 is useful to check mac address
    # libtinfo5 and libncurses5 seem to be needed for Vivado
    RUN apt-get update && apt-get install -y gedit libtinfo5 libncurses5-dev iproute2

    # packages needed for building the risc-v toolchain
    RUN apt-get update && apt-get install -y build-essential python3 cmake help2man device-tree-compiler python3-pip
    RUN apt-get update && apt-get install -y autoconf automake autotools-dev curl git libmpc-dev libmpfr-dev libgmp-dev gawk build-essential bison flex texinfo gperf libtool bc zlib1g-dev

    # packages for cva6-sdk
    RUN apt-get update && apt-get install -y autoconf automake autotools-dev curl libmpc-dev libmpfr-dev libgmp-dev libusb-1.0-0-dev gawk build-essential bison flex texinfo gperf libtool patchutils bc zlib1g-dev device-tree-compiler pkg-config libexpat-dev wget cpio rsync

    # just useful
    RUN apt-get update && apt-get install -y vim
    ```

2. Run `podman build --tag vivado .` in that folder
3. To get graphics passthrough with X, you'll need to create xauth file.
   You can use the following script that I called gen_xauthority.sh:
    ```
    #!/usr/bin/env bash

    CONTAINERNAME="${1:-vivado}"
    touch .Xauthority && \
    xauth -f .Xauthority add $CONTAINERNAME/unix:0 MIT-MAGIC-COOKIE-1 1ee043605d90ec22c2975d91a6b0798b && \
    xauth                add $CONTAINERNAME/unix:0 MIT-MAGIC-COOKIE-1 1ee043605d90ec22c2975d91a6b0798b
    # hex numbers are random
    ```
4. To run the container you can use the following script that I called run.sh:
    ```
    #!/usr/bin/env bash

    ./gen_xauthority.sh && \
    mkdir -p dotXilinx && \
    exec \
    podman run -it --rm -v .Xauthority:/root/.Xauthority:ro -v /tmp/.X11-unix:/tmp/.X11-unix:ro -v dotXilinx:/root/.Xilinx -v .:/mnt -e "DISPLAY" -h vivado --workdir /mnt --network podman --mac-address e2:fa:df:52:9f:78 vivado
    # mac-address is random, you can use any
    ```
5. Inside container run Vivado 2018.2 installer, and point it to /mnt folder (as / will be destroyed)

6. Activate Vivado

7. Install Genesys2 board from https://github.com/Digilent/vivado-boards

8. Apply this workaround https://adaptivesupport.amd.com/s/question/0D52E00006hpi31SAA/unexpected-error-has-occured-11-vivado-20182

9. Now inside container you can cd into cva6 directory and run "LD_PRELOAD=/lib/x86_64-linux-gnu/libudev.so.1 make fpga".
    LD_PRELOAD is to work around this issue https://adaptivesupport.amd.com/s/article/000034450
