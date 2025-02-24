# Produce and Consume to Mediamtx server by RTSP hoặc 'Whip và Whep' (WebRTC) protocol
## 1. Thiết lập
### 1.1 Thiết lập server
```bash
https://github.com/bluenviron/mediamtx/releases/download/v1.11.3/mediamtx_v1.11.3_linux_amd64.tar.gz
tar -xf mediamtx_v1.11.3_linux_amd64.tar.gz
./mediamtx
```

### 1.2 Thông tin về Mediamtx server
- Bạn có thể chỉnh cấu hình của server trong file mediamtx.yml  
- Mediamtx định nghĩa các luồn qua mount point  
  VD: `rtsp://127.0.0.1:8554/mystream`  --> mount point: `mystream`  
           `http://127.0.0.1:8889/stream/whip` --> mount point: `stream`  
           `http://127.0.0.1:8889/stream/whep` --> mount point: `stream`  

- Mỗi mount point sẽ tiếp nhận 1 luồng producer gửi và nhiều luồng consumer nhận cùng một lúc
- Nên để luồng producer gửi lên server trước khi cosumer nhận luồng 

### 1.3 Thiết lập producer và consumer
- Producer và consumer phụ thuộc chính vào gstreamer 
- Các thư viện cần tải: gstreamer1.0, gstreamer plugin {base, bad, good, urly, rtsp server}. Khuyến khích phiên bản >=1.20
Trên ubuntu chạy lệnh
```bash
sudo apt-get install libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev libgstreamer-plugins-bad1.0-dev gstreamer1.0-plugins-base gstreamer1.0-plugins-good gstreamer1.0-plugins-bad gstreamer1.0-plugins-ugly gstreamer1.0-libav gstreamer1.0-tools gstreamer1.0-x gstreamer1.0-alsa gstreamer1.0-gl gstreamer1.0-gtk3 gstreamer1.0-qt5 gstreamer1.0-pulseaudio gstreamer1.0-rtsp libgstrtspserver-1.0-0 gstreamer1.0-rtsp libgstrtspserver-1.0-dev alsa-utils
```
Với các thư viện trên thì có thể chạy giao thức RTSP
- Đối với giao thức Whip và Whep (WebRTC) thì cần biên dịch thêm [simple-whip-clien](https://github.com/meetecho/simple-whip-client.git) và [simple-whep-client](https://github.com/meetecho/simple-whep-client.git). Bạn có thể lấy sẵn file thực thi [tại đây](./Whip_Whep_application)
### 1.3 Đánh giá
- Độ trễ media từ producer đến consumer dùng giao thức RTSP ~ 800ms còn dùng giao thức Whip và Whep (WebRTC) ~ 550ms
- Các câu lệnh cho producer và consumer bên dưới được tối ưu cho thiết bị IoT stream voice audio, bạn có thể thay đổi câu lệnh phù hợp với nhu cầu sử dụng với tài liệu tham khảo [elements of pipeline gstreamer](https://gstreamer.freedesktop.org/documentation/plugins_doc.html#)
## 2. Producer và Consumer dùng giao thức RTSP
- Tài liệu tham khảo [elements of pipeline gstreamer](https://gstreamer.freedesktop.org/documentation/plugins_doc.html#)
### 2.1 Producer
#### Phát audio từ 1 file 

```bash
gst-launch-1.0 filesrc location=/home/lamhung/Downloads/mat_ket_noi.mp3 ! decodebin ! audioconvert ! audioresample ! audio/x-raw,format=S16LE,rate=16000,channels=1 ! opusenc audio-type=voice bandwidth=wideband bitrate=16000 bitrate-type=constrained-vbr complexity=5 frame-size=20 ! rtspclientsink location=rtsp://127.0.0.1:8554/mystream
```

#### Phát audio từ MIC

```bash
gst-launch-1.0 alsasrc ! audioconvert ! audioresample ! audio/x-raw,format=S16LE,rate=16000,channels=1 ! opusenc audio-type=voice bandwidth=wideband bitrate=16000 bitrate-type=constrained-vbr complexity=5 frame-size=20 ! rtspclientsink location=rtsp://127.0.0.1:8554/mystream
```

### 2.1 Consumer


```bash
gst-launch-1.0 rtspsrc location=rtsp://127.0.0.1:8554/mystream latency=0 ! rtpjitterbuffer latency=400 drop-on-latency=true ! queue max-size-buffers=200 ! application/x-rtp,media=audio,encoding-name=OPUS ! rtpopusdepay ! opusdec ! autoaudiosink sync=true
```
## 3. Producer và Consumer dùng giao thức Whip và Whep (WebRTC)
- Tài liệu tham khảo [document simple-whip-client](https://github.com/meetecho/simple-whip-client?tab=readme-ov-file#building-the-whip-client)
- Tài liệu tham khảo [document simple-whep-client](https://github.com/meetecho/simple-whep-client?tab=readme-ov-file#building-the-whep-client)

### 3.1 Producer
```bash
./whip-client -u http://127.0.0.1:8889/stream/whip -A "filesrc location=/home/lamhung/Downloads/du_cho_tan_the.mp3 ! decodebin ! audioconvert ! audioresample ! audio/x-raw,format=S16LE,rate=16000,channels=1 ! opusenc audio-type=voice bandwidth=wideband bitrate=16000 bitrate-type=constrained-vbr complexity=5 frame-size=20 ! rtpopuspay " -V "" -n -b 0
```
### 3.1 Consumer
```bash
./whep-client -u http://127.0.0.1:8889/stream/whep -A "application/x-rtp,media=audio,encoding-name=opus,clock-rate=48000,encoding-params=(string)2,payload=111 " -n -b 200
```