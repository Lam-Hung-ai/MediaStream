# Consume and produce to Mediamtx serve by RTSP and Whip/Whep protocol
## 1. Server
### 1.1 Thiết lập server
```bash
https://github.com/bluenviron/mediamtx/releases/download/v1.11.3/mediamtx_v1.11.3_linux_amd64.tar.gz
tar -xf mediamtx_v1.11.3_linux_amd64.tar.gz
./mediamtx
```

### 1.2 Thông tin về Mediamtx server
- Bạn có thể chỉnh cấu hình của server trong file mediamtx.yml  
- Mediamtx định nghĩa các luồn qua mount point
VD: rtsp://27.71.17.174:8554/mystream  --> mount point: mystream
    http://27.71.17.174:8889/stream/whip --> muont point: stream
    http://27.71.17.174:8889/stream/whep --> muont point: stream

- Mỗi mount point sẽ tiếp nhận 1 luồng producer gửi và nhiều luồng consumer nhận cùng một lúc
- Nên để luồng producer gửi lên serve trước khi cosumer nhận luồng 
-  
## 2. Producer và Consumer dùng giao thức RTSP
Mediamtx định nghĩa các luồn qua mount point 
Các thư viện cần tải: gstreamer, gstreamer plugin {base, bad, good, urly, rtsp server}
Trên ubuntu chạy lệnh
```bash
sudo apt-get install libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev libgstreamer-plugins-bad1.0-dev gstreamer1.0-plugins-base gstreamer1.0-plugins-good gstreamer1.0-plugins-bad gstreamer1.0-plugins-ugly gstreamer1.0-libav gstreamer1.0-tools gstreamer1.0-x gstreamer1.0-alsa gstreamer1.0-gl gstreamer1.0-gtk3 gstreamer1.0-qt5 gstreamer1.0-pulseaudio libgstrtspserver-1.0-0 gstreamer1.0-rtsp libgstrtspserver-1.0-dev
```
### 2.1 Producer
#### Phát audio từ 1 file 
- Lệnh GStreamer
```bash
gst-launch-1.0 filesrc location=/home/lamhung/Downloads/mat_ket_noi.mp3 ! decodebin ! audioconvert ! audioresample ! audio/x-raw,format=S16LE,rate=16000,channels=1 ! opusenc audio-type=voice bandwidth=wideband bitrate=16000 bitrate-type=constrained-vbr complexity=5 frame-size=20 ! rtspclientsink location=rtsp://27.71.17.174:8554/mystream
```

#### Phát audio từ MIC

- Lệnh GStreamer
```bash
gst-launch-1.0 alsasrc ! audioconvert ! audioresample ! audio/x-raw,format=S16LE,rate=16000,channels=1 ! opusenc audio-type=voice bandwidth=wideband bitrate=16000 bitrate-type=constrained-vbr complexity=5 frame-size=20 ! rtspclientsink location=rtsp://27.71.17.174:8554/mystream
```

## Consumer
### Lệnh FFmpeg (sử dụng 1 trong 2, lệnh chưa tối ưu cho thiết bị IoT) 
```bash
ffmpeg -i rtsp://{server_ip}:{port}/test -f alsa default
```

```bash
ffmpeg -i rtsp://{server_ip}:{port}/test -f wav - | aplay
```

### Lệnh GStreamer ( lệnh đã tối ưu cho thiết bị IoT nhưng trễ khoảng 0.8s)
```bash
gst-launch-1.0 rtspsrc location=rtsp://10.0.0.1:8554/test latency=0 ! rtpjitterbuffer latency=200 drop-on-latency=true ! queue max-size-buffers=100 ! application/x-rtp,media=audio,encoding-name=OPUS ! rtpopusdepay ! opusdec plc=true use-inband-fec=true ! autoaudiosink sync=true
```  
# *Mediasoup server
## Producer
### Phát audio từ 1 file 
- Lệnh FFmpeg ( tùy chọn gửi rtcp hoặc không )
```bash
ffmpeg -re -i /home/lamhung/Downloads/dung_lam_trai_tim_anh_dau.mp3 -acodec libopus -b:a 32k -vbr on -ar 48000 -ac 2 -payload_type 96 -ssrc 12345678 -f rtp "rtp://{server_ip}:{rtp_port}?rtcpport={rtcp_port}&localrtpport={rtp_port máy local}&localrtcpport={rtcp_port_máy_local}"
```  

- Lệnh GStreamer
```bash
gst-launch-1.0 -v rtpbin name=rtpbin     filesrc location=/home/lamhung/Downloads/me_yeu_con.mp3 ! decodebin ! audioconvert ! audioresample ! audio/x-raw,format=S16LE,rate=48000,channels=2 ! opusenc bitrate=32000 bitrate-type=vbr complexity=6 frame-size=20 bandwidth=wideband dtx=true ! rtpopuspay ssrc=12345678 ! rtpbin.send_rtp_sink_0     rtpbin.send_rtp_src_0 ! udpsink host=10.0.0.1 port=40001 bind-port=10001     rtpbin.send_rtcp_src_0 ! udpsink host=10.0.0.1 port=40002 bind-port=10002 sync=false async=false
```

### Phát audio từ MIC
- Lệnh FFmpeg
```bash
ffmpeg -re -f alsa -i hw:0,0 -acodec libopus -b:a 32k -vbr on -ar 48000 -ac 2 -payload_type 96 -ssrc {ssrc của producer} -f rtp "rtp://{server_ip}:{rtp_port}?rtcpport={rtcp_port}&localrtpport={rtp_port máy local}&localrtcpport={rtcp_port_máy_local}"
```
- Lệnh GStreamer
```bash
gst-launch-1.0 alsasrc ! audioconvert ! audioresample ! audio/x-raw,format=S16LE,rate=48000,channels=2 ! opusenc bitrate=32000 bitrate-type=vbr complexity=6 frame-size=20 bandwidth=wideband dtx=true ! rtpopuspay ssrc={ssrc của producer} ! udpsink host={server_ip} port={server_port}
```

## Consumer
Trước khi nhận audio từ mediasoup server thì ta cần phải gửi một chút audio để server xác định địa chỉ ip và cổng của client trong cùng mạng VPN hoặc mạng LAN
### Lệnh FFmpeg 
- Gửi audio nhỏ tới server
```bash
ffmpeg -f lavfi -i "sine=frequency=440" -t 0.1 -acodec libopus -b:a 32k -vbr on -ar 48000 -ac 2 -payload_type 96 -f rtp "rtp://{server_ip}:{rtp_port}?rtcpport={rtcp_port}&localrtpport={rtp_port_máy_local}&localrtcpport={rtcp_port_máy_local}"
```
- Tạo file consumer.sdp với nội dung
```text
v=0
o=- 0 0 IN IP4 127.0.0.1
s=AudioStream
c=IN IP4 127.0.0.1
t=0 0
m=audio {port_nhận_audio_từ_server} RTP/AVP 96
a=rtpmap:96 opus/48000/2
a=fmtp:96 minptime=10
```
port_nhận_audio_từ_server tương ứng với {rtp_port máy local} khi gửi audio nhở tới server
- Nhận audio từ Mediasoup server
```bash
ffmpeg -protocol_whitelist file,udp,rtp -i consumer.sdp -f alsa default
```
```bash
ffmpeg -protocol_whitelist file,udp,rtp -i consumer.sdp -f wav - | aplay
```

### Lệnh GStreamer
- Gửi audio nhỏ tới server
```bash
gst-launch-1.0 -v rtpbin name=rtpbin audiotestsrc freq=440 num-buffers=5 wave=sine ! audioconvert ! audioresample ! opusenc ! rtpopuspay ! rtpbin.send_rtp_sink_0 rtpbin.send_rtp_src_0 ! udpsink host=10.0.0.1 port=40003 bind-port=10003 rtpbin.send_rtcp_src_0 ! udpsink host=10.0.0.1 port=40004 bind-port=10004 sync=false async=false
```
- Nhận audio từ Mediasoup server
```bash
gst-launch-1.0 udpsrc port=10003 ! application/x-rtp,media=audio,encoding-name=OPUS,payload=96 ! queue max-size-buffers=200  ! rtpopusdepay ! opusdec ! autoaudiosink sync=true
```  
port_nhận_dữ_liệu = bind-port_gửi dữ liệu rtp lúc gửi audio nhỏ
