# Linux-Fake-Background-Webcam

## Background
Video conferencing software support for background blurring and background
replacement under Linux is relatively poor. The Linux version of Zoom only
supports background replacement via chroma key. The Linux version of Microsoft
Team does not support background blur. Over at Webcamoid, we tried to figure out
if we can do these reliably using open source software
([issues/250](https://github.com/webcamoid/webcamoid/issues/250)).

This repository started of as a tidy up of Benjamen Elder's
[blog post](https://elder.dev/posts/open-source-virtual-background/). His
blogpost described a background replacement solution using Python, OpenCV,
Bodypix neural network, which is only available under Tensorflow.js. The scripts
in Elder's blogpost do not work out of box. This repository originally provided
a turn-key solution for creating a virtual webcam with background replacement
and additionally foreground object placement, e.g. a podium.

Over time this repository got strangely popular. However it has been clear over
time that Bodypix is slow and difficult to set up. Various users wanted to use
their GPU with Tensorflow.js, this does not always work. The extra code that
provided GPU support sometimes created problems for CPU-only users.

Recently Google released selfie segmentation support for
[Mediapipe](https://github.com/google/mediapipe/releases/tag/v0.8.5). This
repository has been updated to use Mediapipe for image segmentation. This
significantly increased the performance. The older version of this repository
is now stored in the ``bodypix`` branch. 

## Prerequisite
You need to install either v4l2loopback or akvcam. This repository was
originally written with v4l2loopback in mind. However, there has been report
that v4l2loopback does not work with certain versions of Ubuntu. Additionally,
the author has never really managed to get v4l2loopback to work with Microsoft
Team. Therefore support for akvcam has been added.

### v4l2loopback
If you are on Debian Buster, you can do the
following:

    sudo apt install v4l2loopback-dkms

I added module options for v4l2loopback by creating
``/etc/modprobe.d/v4l2loopback.conf`` with the following content:

    options v4l2loopback devices=1  exclusive_caps=1 video_nr=2 card_label="v4l2loopback"

``exclusive_caps`` is required by some programs, e.g. Zoom and Chrome.
``video_nr`` specifies which ``/dev/video*`` file is the v4l2loopback device.
In this repository, I assume that ``/dev/video2`` is the virtual webcam, and
``/dev/video0`` is the physical webcam.

I also created ``/etc/modules-load.d/v4l2loopback.conf`` with the following content:

    v4l2loopback

This automatically loads v4l2loopback module at boot, with the specified module
options.

If you get an error like
```
OSError: [Errno 22] Invalid argument
```

when opening the webcam from Python, please try the latest version of
v4l2loopback from the its
[Github repository](https://github.com/umlaeute/v4l2loopback),
as the version from your package manager may be too old.

#### Ubuntu 18.04
If you are using Ubuntu 18.04, and if you want to use v4l2loopback, please compile
v4l2loopback from the source. You need to do the following: 
1. Remove the ``v4l2loopback`` package
    - `sudo rmmod -r v4l2loopback`
    - `sudo apt-get remove v4l2loopback-dkms`
2. Install DKMS and the Linux kernel headers
    - ``sudo apt-get install dkms linux-headers-`uname -r` ``
3. Install v4l2loopback from the repository
    - `git clone https://github.com/umlaeute/v4l2loopback.git`
    - `cd v4l2loopback`
4. Install the module via DKMS
    - `sudo cp -R . /usr/src/v4l2loopback-1.1`
    - `sudo dkms add -m v4l2loopback -v 1.1`
    - `sudo dkms build -m v4l2loopback -v 1.1`
    - `sudo dkms install -m v4l2loopback -v 1.1`
5. Load the module
    - `sudo modprobe v4l2loopback`

This may apply for other versions of Ubuntu as well. For more information,
please refer to the following Github
[issue](https://github.com/jremmons/pyfakewebcam/issues/7#issuecomment-616617011).

### Akvcam
To install akvcam, you need to do the following:
1. Install the driver by following the instruction at
[Akvcam wiki](https://github.com/webcamoid/akvcam/wiki/Build-and-install). I
recommend installing and managing the driver via DKMS.
2. Configure the driver by copying ``akvcam`` to ``/etc/``, for more
information, please refer to
[Akvcam wiki](https://github.com/webcamoid/akvcam/wiki/Configure-the-cameras)

The configuration file I supplied was originally generated by Webcamoid, I
added the ``rw`` attributes to do the virtual camera devices. If you already
have already configured akvcam via webcamoid, you need to modify the
``/etc/akvcam/config.ini`` to add the ``rw`` attributes.

### Disabling UEFI Secure boot
Both v4l2loopback and Akvcam require custom kernel module. This might not be possible 
if you have secure boot enabled. Please refer to your device manufacturer's manual 
on disabling secure boot. 

### Python 3
You will need Python 3. You need to have pip installed. Please make sure that
you have installed the correct version pip, if you have both Python 2 and
Python 3 installed. Please make sure that the command ``pip3`` runs.

In Debian, you can run

    sudo apt-get install python3-pip

I am assuming that you have set up your user environment properly, and when you
install Python packages, they will be installed locally within your home
directory.

You might want to add the following line in your ``.profile``. This line is
needed for Debian Buster.

    export PATH="$HOME/.local/bin":$PATH
configuration files yourself.

### Upgrading pip
Mediapipe requires pip version 19.3 or above. (Please refer to
[here](https://pypi.org/project/mediapipe/#files) and
[here](https://github.com/pypa/manylinux)). However, the pip distributed with
some Linux distributions is outdated, e.g.
[Debian Buster](https://packages.debian.org/buster/python3-pip).

If you are on Debian Buster please make sure ``.local/bin`` is in your ``PATH``.
You can make sure this is the case by adding:

    PATH="$HOME/.local/bin":$PATH

in your ``~/.profile``.

You can then upgrade pip by running:

    pip3 install --upgrade pip

## Installation
The actual installation can be done by simply running

    ./install.sh

### Installing with Docker
The use of Docker is no longer supported. I no longer see any reason for using
Docker with this software. However I have left behind the files
related to Docker, for those who want to fix Docker support.
Please also refer to [DOCKER.md](DOCKER.md). The Docker related files were
provided by [liske](https://github.com/liske).

Docker made starting up and shutting down the virtual webcam more convenient
for when Bodypix was needed. The ability to change background and foreground
images on-the-fly is unsupported when running under Docker.

## Usage
In the terminal window, do the following (if using v4l2loopback) :

    python3 fake.py

or (if using Akvcam) :

    python3 fake.py --akvcam

The files that you might want to replace are the followings:

  - ``background.jpg`` - the background image
  - ``foreground.jpg`` - the foreground image
  - ``foreground-mask.jpg`` - the foreground image mask

If you want to change the files above in the middle of streaming, replace them
and press ``CTRL-C``

Note that animated background is supported. You can use any video file that can
be read by OpenCV.

### fake.py

``fakecam.py`` supports the following options:

    usage: fake.py [-h] [-W WIDTH] [-H HEIGHT] [-F FPS] [-C CODEC]
                [-S SCALE_FACTOR] [-w WEBCAM_PATH] [-v V4L2LOOPBACK_PATH]
                [--akvcam] [-i IMAGE_FOLDER] [-b BACKGROUND_IMAGE]
                [--tile-background] [--no-background]
                [--background-blur BACKGROUND_BLUR] [--background-keep-aspect]
                [--no-foreground] [-f FOREGROUND_IMAGE]
                [-m FOREGROUND_MASK_IMAGE] [--hologram]

    Faking your webcam background under GNU/Linux. Please make sure your bodypix
    network is running. For more information, please refer to:
    https://github.com/fangfufu/Linux-Fake-Background-Webcam

    optional arguments:
    -h, --help            show this help message and exit
    -W WIDTH, --width WIDTH
                            Set real webcam width
    -H HEIGHT, --height HEIGHT
                            Set real webcam height
    -F FPS, --fps FPS     Set real webcam FPS
    -C CODEC, --codec CODEC
                            Set real webcam codec
    -w WEBCAM_PATH, --webcam-path WEBCAM_PATH
                            Set real webcam path
    -v V4L2LOOPBACK_PATH, --v4l2loopback-path V4L2LOOPBACK_PATH
                            V4l2loopback device path
    --akvcam              Use an akvcam device rather than a v4l2loopback device
    -i IMAGE_FOLDER, --image-folder IMAGE_FOLDER
                            Folder which contains foreground and background images
    -b BACKGROUND_IMAGE, --background-image BACKGROUND_IMAGE
                            Background image path, animated background is
                            supported.
    --tile-background     Tile the background image
    --no-background       Disable background image, blurry background
    --background-blur BACKGROUND_BLUR
                            Set background blur level
    --background-keep-aspect
                            Crop background if needed to maintain aspect ratio
    --no-foreground       Disable foreground image
    -f FOREGROUND_IMAGE, --foreground-image FOREGROUND_IMAGE
                            Foreground image path
    -m FOREGROUND_MASK_IMAGE, --foreground-mask-image FOREGROUND_MASK_IMAGE
                            Foreground mask image path
    --hologram            Add a hologram effect

## License
The soure code of this file are released under GPLv3.

    Linux Fake Background Webcam
    Copyright (C) 2020-2021  Fufu Fang

    This program is free software: you can redistribute it and/or modify
    it under the terms of the GNU General Public License as published by
    the Free Software Foundation, either version 3 of the License, or
    (at your option) any later version.

    This program is distributed in the hope that it will be useful,
    but WITHOUT ANY WARRANTY; without even the implied warranty of
    MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
    GNU General Public License for more details.

    You should have received a copy of the GNU General Public License
    along with this program.  If not, see <https://www.gnu.org/licenses/>.


