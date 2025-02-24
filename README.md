# Produce and Consume to Mediamtx server by RTSP or 'WHIP and WHEP' (WebRTC) protocol

## 1. Setup

### 1.1 Server Setup

```bash
wget https://github.com/bluenviron/mediamtx/releases/download/v1.11.3/mediamtx_v1.11.3_linux_amd64.tar.gz
tar -xf mediamtx_v1.11.3_linux_amd64.tar.gz
./mediamtx
```

### 1.2 Information about the Mediamtx server

- You can adjust the server's configuration in the `mediamtx.yml` file.
- Mediamtx defines streams through mount points.
  Example: `rtsp://127.0.0.1:8554/mystream`  --> mount point: `mystream`
           `http://127.0.0.1:8889/stream/whip` --> mount point: `stream`
           `http://127.0.0.1:8889/stream/whep` --> mount point: `stream`

- Each mount point will accept 1 producer stream and multiple consumer streams simultaneously.
- It is recommended to have the producer stream send to the server before consumers start receiving.

### 1.3 Setting up Producer and Consumer

- Producer and consumer rely heavily on GStreamer.
- Required libraries: gstreamer1.0, gstreamer plugins {base, bad, good, ugly, rtsp server}.  Version >= 1.20 is recommended.
  On Ubuntu, run the following command:

```bash
sudo apt-get install libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev libgstreamer-plugins-bad1.0-dev gstreamer1.0-plugins-base gstreamer1.0-plugins-good gstreamer1.0-plugins-bad gstreamer1.0-plugins-ugly gstreamer1.0-libav gstreamer1.0-tools gstreamer1.0-x gstreamer1.0-alsa gstreamer1.0-gl gstreamer1.0-gtk3 gstreamer1.0-qt5 gstreamer1.0-pulseaudio gstreamer1.0-rtsp libgstrtspserver-1.0-0 gstreamer1.0-rtsp libgstrtspserver-1.0-dev alsa-utils
```
With the above libraries, you can run the RTSP protocol.

- For the WHIP and WHEP (WebRTC) protocols, you need to compile [simple-whip-client](https://github.com/meetecho/simple-whip-client.git) and [simple-whep-client](https://github.com/meetecho/simple-whep-client.git).  You can find pre-built binaries [here](./Whip_Whep_application).

### 1.4 Evaluation

- Media latency from producer to consumer using the RTSP protocol is ~800ms, while using the WHIP and WHEP (WebRTC) protocols it's ~550ms.
- The commands for producer and consumer below are optimized for IoT devices streaming voice audio.  You can modify the commands according to your needs, referring to the [GStreamer pipeline elements documentation](https://gstreamer.freedesktop.org/documentation/plugins_doc.html#).

## 2. Producer and Consumer using RTSP Protocol

- Reference: [GStreamer pipeline elements documentation](https://gstreamer.freedesktop.org/documentation/plugins_doc.html#)

### 2.1 Producer

#### Play audio from a file

```bash
gst-launch-1.0 filesrc location=/home/lamhung/Downloads/mat_ket_noi.mp3 ! decodebin ! audioconvert ! audioresample ! audio/x-raw,format=S16LE,rate=16000,channels=1 ! opusenc audio-type=voice bandwidth=wideband bitrate=16000 bitrate-type=constrained-vbr complexity=5 frame-size=20 ! rtspclientsink location=rtsp://127.0.0.1:8554/mystream
```

#### Play audio from MIC

```bash
gst-launch-1.0 alsasrc ! audioconvert ! audioresample ! audio/x-raw,format=S16LE,rate=16000,channels=1 ! opusenc audio-type=voice bandwidth=wideband bitrate=16000 bitrate-type=constrained-vbr complexity=5 frame-size=20 ! rtspclientsink location=rtsp://127.0.0.1:8554/mystream
```

### 2.2 Consumer

```bash
gst-launch-1.0 rtspsrc location=rtsp://127.0.0.1:8554/mystream latency=0 ! rtpjitterbuffer latency=400 drop-on-latency=true ! queue max-size-buffers=200 ! application/x-rtp,media=audio,encoding-name=OPUS ! rtpopusdepay ! opusdec ! autoaudiosink sync=true
```

## 3. Producer and Consumer using WHIP and WHEP (WebRTC) Protocols

- Reference: [simple-whip-client documentation](https://github.com/meetecho/simple-whip-client?tab=readme-ov-file#building-the-whip-client)
- Reference: [simple-whep-client documentation](https://github.com/meetecho/simple-whep-client?tab=readme-ov-file#building-the-whep-client)

### 3.1 Producer

```bash
./whip-client -u http://127.0.0.1:8889/stream/whip -A "filesrc location=/home/lamhung/Downloads/du_cho_tan_the.mp3 ! decodebin ! audioconvert ! audioresample ! audio/x-raw,format=S16LE,rate=16000,channels=1 ! opusenc audio-type=voice bandwidth=wideband bitrate=16000 bitrate-type=constrained-vbr complexity=5 frame-size=20 ! rtpopuspay " -V "" -n -b 0
```

### 3.2 Consumer

```bash
./whep-client -u http://127.0.0.1:8889/stream/whep -A "application/x-rtp,media=audio,encoding-name=opus,clock-rate=48000,encoding-params=(string)2,payload=111 " -n -b 200
```