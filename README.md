# Raspberry Pi AI Kit on Ubuntu

This repository provides a method of using the [Raspberry Pi AI Kit](https://www.raspberrypi.com/documentation/accessories/ai-kit.html) on Ubuntu 24.04.
It makes use of a Docker container to bundle the correct versions of the Hailo SDK libraries.

## Install driver on host

For the SDK to work with the AI accelerator hardware, a matching version of the driver needs to be used.
Even a minor version difference will prevent the SDK from detecting the hardware.

Install requirements on host:

```
sudo apt-get install linux-headers-$(uname -r)
```

We get the exact version of the driver's source code from Github:

```
git clone https://github.com/hailo-ai/hailort-drivers.git
cd hailort-drivers
git checkout f840b6219230ec9a350444dbb903adbf0f63a373
```

Then build it and install it on the host system:

```
cd linux/pcie
make all
sudo make install
sudo modprobe hailo_pci
cd ../..
./download_firmware.sh
sudo mkdir -p /lib/firmware/hailo
sudo mv hailo8_fw.4.17.0.bin /lib/firmware/hailo/hailo8_fw.bin
sudo cp ./linux/pcie/51-hailo-udev.rules /etc/udev/rules.d/
sudo udevadm control --reload-rules && sudo udevadm trigger
```

It's better to restart after installing this driver.
After a reboot you can look at the kernel buffer to see if the device is detected and the driver loaded.

```
$ sudo dmesg | grep hailo
...
[    4.379687] hailo: Init module. driver version 4.17.0
...
[    4.545602] hailo 0000:01:00.0: Firmware was loaded successfully
[    4.572371] hailo 0000:01:00.0: Probing: Added board 1e60-2864, /dev/hailo0
```

## Build the container and open a shell

Configure host X server to accept connections from a Docker container

```
xhost +local:docker
```

Build and start the container

```
docker compose build
docker compose up -d hailo-ubuntu-pi
```

Open a shell inside the container

```
docker compose exec hailo-ubuntu-pi /bin/bash
```

## Using the container

Test that the camera is working. On Ubuntu you will have to add the `--qt-preview` argument to all `rpicam-` commands.
This is because the default windowing toolkit used by these examples is not supported on Ubuntu.

```
rpicam-hello --timeout=0 --qt-preview
```

Install Hailo libraries.
This can't be done as part of building the container, as configuring the packages depend on a running systemd.

```
apt install hailo-all
```

## Verify installation

Check that the hardware is working from inside the container

```
$ hailortcli fw-control identify
Executing on device: 0000:01:00.0
Identifying board
Control Protocol Version: 2
Firmware Version: 4.17.0 (release,app,extended context switch buffer)
Logger Version: 0
Board Name: Hailo-8
Device Architecture: HAILO8L
Serial Number: <redacted>
Part Number: <redacted>
Product Name: HAILO-8L AI ACC M.2 B+M KEY MODULE EXT TMP
```

[Test TAPPAS Core installation](https://github.com/hailo-ai/hailo-rpi5-examples/blob/main/doc/install-raspberry-pi5.md#test-tappas-core-installation-by-running-the-following-commands):

```
$ gst-inspect-1.0 hailotools
Plugin Details:
  Name                     hailotools
  Description              hailo tools plugin
  Filename                 /lib/aarch64-linux-gnu/gstreamer-1.0/libgsthailotools.so
  Version                  3.28.2
  License                  unknown
  Source module            gst-hailo-tools
  Binary package           gst-hailo-tools
  Origin URL               https://hailo.ai/

  hailoaggregator: hailoaggregator - Cascading
  hailocounter: hailocounter - postprocessing element
  hailocropper: hailocropper
  hailoexportfile: hailoexportfile - export element
  hailoexportzmq: hailoexportzmq - export element
  hailofilter: hailofilter - postprocessing element
  hailogallery: Hailo gallery element
  hailograytonv12: hailograytonv12 - postprocessing element
  hailoimportzmq: hailoimportzmq - import element
  hailomuxer: Muxer pipeline merging
  hailonv12togray: hailonv12togray - postprocessing element
  hailonvalve: HailoNValve element
  hailooverlay: hailooverlay - overlay element
  hailoroundrobin: Input Round Robin element
  hailostreamrouter: Hailo Stream Router
  hailotileaggregator: hailotileaggregator
  hailotilecropper: hailotilecropper - Tiling
  hailotracker: Hailo object tracking element

  18 features:
  +-- 18 elements
```

```
$ gst-inspect-1.0 hailo
Plugin Details:
  Name                     hailo
  Description              hailo gstreamer plugin
  Filename                 /lib/aarch64-linux-gnu/gstreamer-1.0/libgsthailo.so
  Version                  1.0
  License                  unknown
  Source module            hailo
  Binary package           GStreamer
  Origin URL               http://gstreamer.net/

  hailodevicestats: hailodevicestats element
  hailonet: hailonet element
  synchailonet: sync hailonet element

  3 features:
  +-- 3 elements
```

## GS camera on Pi 5

For the Global Shutter camera one [needs to specify the image size](https://community.hailo.ai/t/rpi5-pi-global-shutter-camera/1234), using the `--width` and `--height` arguments.

```
rpicam-vid -t 0 --post-process-file ~/rpicam-apps/assets/hailo_yolov8_inference.json --lores-width 640 --lores-height 640 --width 1456 --height 1088 --qt-preview
```

## Pi Camera v1.3 on Pi 5

### Object detection

```
rpicam-hello -t 0 --post-process-file ~/rpicam-apps/assets/hailo_yolov6_inference.json --lores-width 640 --lores-height 640 --qt-preview
```

```
rpicam-hello -t 0 --post-process-file ~/rpicam-apps/assets/hailo_yolov8_inference.json --lores-width 640 --lores-height 640 --qt-preview
```

```
rpicam-hello -t 0 --post-process-file ~/rpicam-apps/assets/hailo_yolox_inference.json --lores-width 640 --lores-height 640 --qt-preview
```

![Object Detection](media/Object%20Detection.png)

### Person and face detection

```
rpicam-hello -t 0 --post-process-file ~/rpicam-apps/assets/hailo_yolov5_personface.json --lores-width 640 --lores-height 640 --qt-preview
```

### Image segmentation

```
rpicam-hello -t 0 --post-process-file ~/rpicam-apps/assets/hailo_yolov5_segmentation.json --lores-width 640 --lores-height 640 --framerate 20 --qt-preview
```

### Pose estimation

```
rpicam-hello -t 0 --post-process-file ~/rpicam-apps/assets/hailo_yolov8_pose.json --lores-width 640 --lores-height 640 --qt-preview
```

![Pose Estimation](media/Pose%20Estimation.png)

## Advanced examples

Hailo publishes [more examples](https://github.com/hailo-ai/hailo-rpi5-examples/blob/main/README.md#configure-environment) for the Raspberry Pi 5.
We can run these too.

### Set up environment and download assets

```
cd hailo-rpi5-examples
source setup_env.sh
pip install -r requirements.txt
./compile_postprocess.sh
python basic_pipelines/detection.py --input resources/detection0.mp4
```

### Using the Raspberry Pi camera

```
python basic_pipelines/detection.py -i rpi
python basic_pipelines/detection.py --labels-json resources/barcode-labels.json --hef resources/yolov8s-hailo8l-barcode.hef -i rpi
python basic_pipelines/pose_estimation.py -i rpi
python basic_pipelines/instance_segmentation.py -i rpi
```

### A USB webcam is also supported

```
python basic_pipelines/detection.py -i /dev/video8
```

## Notes

If the Advanced Examples fail with the following error:

```
AfMode not supported by this camera, please retry with 'auto-focus-mode=AfModeManual'
```

Run these commands to change `auto-focus-mode` to manual:

```
sed -i -e 's/auto-focus-mode=2/auto-focus-mode=0/g' basic_pipelines/detection.py
sed -i -e 's/auto-focus-mode=2/auto-focus-mode=0/g' basic_pipelines/pose_estimation.py
sed -i -e 's/auto-focus-mode=2/auto-focus-mode=0/g' basic_pipelines/instance_segmentation.py
```
