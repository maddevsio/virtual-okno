# Raspberry Pi streaming

## Hardware

Hardware to use:
* Raspberry Pi 3 B+
* Case
* Heatsinks
* MicroSD card (16 GB, UHS-I recommended)
* Power supply (2.5 A, 5 V DC, MicroUSB)
* Logitech C920 webcam
* USB sound adapter
* TV or monitor
* HDMI cable
* Headset
* Microphone

Why we use Logitech C920? This webcam can get H.264-encoded 1080p/30fps stream, which can be transmitted without encoding/decoding. Raspberry Pi can decode H.264 stream to display on screen.

## Preparation

*Warning! To prevent damage, microSD card should been inserted into a slot after board installation into a case.*

The latest version of Raspbian and installation tutorial can be found at https://www.raspberrypi.org/downloads/raspbian/

*Note! We used the Stretch release, the latest Raspbian release is Buster, so we haven't tested the code on Buster yet.*

We used the [Lite](https://downloads.raspberrypi.org/raspbian_lite/images/raspbian_lite-2019-04-09/2019-04-08-raspbian-stretch-lite.zip) variant of Raspbian installation. Unpack it and write to microSD card using a proper software in your OS.

To enable Wi-Fi and SSH access at first boot add some modifications. On Linux host it would be the following:

```touch /media/<username>/boot/ssh```

```nano /media/<username>/boot/wpa_supplicant.conf```

and fill up it with the following:

```ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
country=GB

network={
    ssid="*your netwok SSID*"
    psk="*your key*"
    key_mgmt=WPA-PSK
    # for hidden network:
    #scan_ssid=1
}
```

If your network uses WPA-Enterprise:
```
network={
    ssid="*your netwok SSID*"
    identity="*your login*"
    password="*your password*"
    key_mgmt=WPA-EAP
    eap=TTLS
    phase2="auth=MSCHAPV2"
}
```

Complete documentation: https://www.systutorials.com/docs/linux/man/5-wpa_supplicant.conf/

It's better to use a monitor and a keyboard at the first boot. 

Default user: pi

Default password: raspberry


After login let's change a password:

```sudo passwd pi```

To check if the date has been updated:

```timedatectl```

Install updates:

```sudo apt update && sudo apt upgrade -y && sudo apt autoremove -y```

Update firmware:

```sudo rpi-update```

and reboot:

```sudo reboot```

To configure Raspberry (keyboard layout, time zone, etc) use:

```sudo raspi-config```

## Camera

Install GStreamer:

```sudo apt install gstreamer1.0-tools```

List of cameras:

```v4l2-ctl --list-devices```

Sample output:

```
bcm2835-codec (platform:bcm2835-codec):
	/dev/video10
	/dev/video11
	/dev/video12

HD Pro Webcam C920 (usb-3f980000.usb-1.2):
	/dev/video0
	/dev/video1
```

Camera capabilities:

```v4l2-ctl --list-formats-ext --device /dev/video0```

Sample output:

```
ioctl: VIDIOC_ENUM_FMT
	Index       : 0
	Type        : Video Capture
	Pixel Format: 'YUYV'
	Name        : YUYV 4:2:2
		Size: Discrete 640x480
			Interval: Discrete 0.033s (30.000 fps)
			Interval: Discrete 0.042s (24.000 fps)
...
	Index       : 1
	Type        : Video Capture
	Pixel Format: 'H264' (compressed)
	Name        : H.264
		Size: Discrete 640x480
			Interval: Discrete 0.033s (30.000 fps)
			Interval: Discrete 0.042s (24.000 fps)
			Interval: Discrete 0.050s (20.000 fps)
			Interval: Discrete 0.067s (15.000 fps)
			Interval: Discrete 0.100s (10.000 fps)
			Interval: Discrete 0.133s (7.500 fps)
			Interval: Discrete 0.200s (5.000 fps)
...
		Size: Discrete 1920x1080
			Interval: Discrete 0.033s (30.000 fps)
			Interval: Discrete 0.042s (24.000 fps)
			Interval: Discrete 0.050s (20.000 fps)
			Interval: Discrete 0.067s (15.000 fps)
			Interval: Discrete 0.100s (10.000 fps)
			Interval: Discrete 0.133s (7.500 fps)
			Interval: Discrete 0.200s (5.000 fps)

```


## Using GStreamer for video

A receiver should be started before a sender.

A sender command:

```gst-launch-1.0 v4l2src device=/dev/video0 ! queue ! video/x-h264,width=1920,height=1080,framerate=24/1 ! fdsink | nc -u <receiverIP> 5001```

A receiver command:

```nc -l -u 5001 > video.stream | omxplayer -o hdmi video.stream```


## Using GStreamer for audio

A receiver should be started before a sender.

A receiver command:

```gst-launch-1.0 udpsrc port=4444 caps="application/x-rtp,channels=1" ! queue ! rtpjitterbuffer latency=100  ! rtpopusdepay ! opusdec plc=true ! queue ! audioconvert ! audioresample ! alsasink device=hw:2```

A sender command:

```gst-launch-1.0 alsasrc device=hw:2 ! queue ! audiorate ! audioconvert ! audioresample ! opusenc ! rtpopuspay ! udpsink host=<receiverIP> port=4444```


## Running Docker

Install Docker:

```sudo apt update && sudo apt upgrade -y```

```curl -sSL https://get.docker.com | sh```

Add user to a group:

```sudo usermod -aG docker pi```

After it, exit termintal and login again.

Enable daemon:

```sudo systemctl enable docker```

Start daemon:

```sudo systemctl start docker```

Check if Docker installed correctly:

```docker run hello-world```

Install docker-compose:

```sudo apt install python3-pip -y```

```sudo pip3 install docker-compose```

Check:

```docker-compose --version```

Set up a repository:

```sudo apt install git```

```git clone https://github.com/maddevsio/virtual-okno.git```

```cd virtual-okno```

Edit .env file:

```nano Docker/.env```

Sample content of .env:

```
RECEIVE_IP=<peerIP>
AUDIO_PORT=5003
VIDEO_PORT=5001
ALSA_OUT_DEV=hw:2
ALSA_IN_DEV=hw:1
VIDEO_DEV=/dev/video0
```

```RECEIVE_IP``` is an IP address of another installation

Run:

```cd Docker```

```docker-compose up -d```

To restart: 

```docker-compose restart restart```


## To do

* Connection via VPN
* Remote management

