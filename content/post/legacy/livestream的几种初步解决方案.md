+++
date = '2016-02-18T20:57:19+08:00'
draft = false
title = 'livestream的几种初步解决方案'

+++


实时视频流解决方案
===================

mjpg-streamer
-----------------

    site: https://github.com/codewithpassion/mjpg-streamer/tree/master/mjpg-streamer

	    link: http://www.cnblogs.com/hnrainll/archive/2011/06/08/2074909.html
	    link: http://blog.163.com/chenhongswing@126/blog/static/1335924432011825104144612/
```bash
    #提供一个可以直接使用的demo

    git clone https://github.com/codewithpassion/mjpg-streamer/tree/master/mjpg-streamer

    cd mjpg-streamer

    ./mjpg_streamer -i "input_uvc.so -d /dev/video0 -r 640x480 -y" -o "output_http.so -w ./www"

    127.0.0.1:8080访问主页，可以获得stream、static以及通过vlc和mplayer播放
```

ffmpeg+websocket播放
-------------------------

    site: https://github.com/phoboslab/jsmpeg
    link: http://segmentfault.com/a/1190000000392586

```bash
    #提供可以直接运行的demo

    git clone https://github.com/phoboslab/jsmpeg

    nodejs stream-server.js huang	监听随机端口
        Listening for MPEG Stream on http://127.0.0.1:8082/<secret>/<width>/<height>
        Awaiting WebSocket connections on ws://127.0.0.1:8084/
        Stream Connected: 127.0.0.1:52460 size: 640x480
        New WebSocket Connection (1 total)
        Disconnected WebSocket (0 total)

    ffmpeg -s 640x480 -f video4linux2 -i /dev/video0 -f mpeg1video -b 800k -r 30 http://localhost:8082/huang/640/480/		ffmpeg采集编码并发送视频流
    google-chrome stream-example.html		使用websocket在线观看
```

vlc视频流输出和vlc视频流播放
---------------------------------

    site: http://www.cnblogs.com/fx2008/p/4315416.html
```bash
    a、关掉防火墙，至少将视频流端口开启
    b、vlc --ttl 12 -vvv --color -I telnet --telnet-password videolan --rtsp-host 0.0.0.0 --rtsp-port 5554
    c、通过telnet启动vlc的vlm管理中，telnet 127.0.0.1 4212，密码是刚才的videolan
    d、new Test vod enabled 新建一个vod，名字是Test
    e、setup Test input /path/video.file    给Test输入视频
    f、vlc rtsp://127.0.0.1:5554/Test 启动vlc观看视频流，这里还差声音
```

nginx的rtmp转播
-------------------

    site: http://itony.me/619.html
    site: https://github.com/arut/nginx-rtmp-module
    site: http://openresty.org
    site: http://blog.csdn.net/leixiaohua1020/article/details/12029543
    site: http://blog.csdn.net/leixiaohua1020/article/details/39803457
    site: http://blog.csdn.net/fireroll/article/details/18899285

```bash
    a、安装openresty和rtmp模块
    b、配置nginx，rtmp配置块与http配置块平级
        rtmp {
            server {
                listen 1935;
                application live1 {
                    live on;
                    record off;
                }
            }
        }
    c、ffmpeg转发视频流  
        ffmpeg -re -i xxx.mp4 -c copy -f flv   rtmp://localhost:1935/live1/room1
        其中的live1是应用，room1是将来要打开的节点

    d、在vlc中打开视频流： 
        rtmp://localhost:1935/live1/room1

    e、ffmpeg -f video4linux2 -i /dev/video0 -c:v libx264 -an -f flv rtmp://localhost:1935/live1/room1 试试摄像头
```

aliyun+nginx+rtmp在线转播
-------------------------------

    site: http://blog.csdn.net/xdwyyan/article/details/43198985
    注意：在nginx中配置rtmp后，reload是不够的，需要kill掉重新启动nginx。
    通过sudo netstat -tlnp | grep 1935来观察nginx是否将端口打开

```bash
    sudo ffmpeg -i /dev/video0 -acodec acc -strict experimental -ar 44100 -ac 2 -b:a 96k -r 25 -b:v 500k -s 640*480 -f flv rtmp://101.200.124.174:1935/live1/room1
    vlc rtmp://101.200.124.174:1935/live1/room1

    ffmpeg -loglevel verbose -re -i xxx.mp4 -vcodec libx264 -vprofile baseline -acodec libmp3lame -ar 44100 -ac 1 -f flv rtmp://101.200.124.174:1935/hls/movie
    vlc http://101.200.124.174/hls/movie.m3u8

    #v4l2读取摄像头，进行x264编码并将视频流发送至阿里云，进而进行hls播放
    ffmpeg -f video4linux2 -s 320x240 -i /dev/video0 -vcodec libx264 -f flv  rtmp://101.200.124.174:1935/hls/movie
    vlc http://101.200.124.174/hls/movie.m3u8
    手机浏览器打开也行

    #http config block
    rtmp {
        server {
            listen 1935;
            application live1 {
                live on;
                record off;
            }
            application hls{
                live on;
                hls on;
                hls_path /tmp/hls;
            }
    
        }
    }

    #server config block
    location /hls {
        types {
            application/vnd.apple.mpegurl m3u8;
            video/mp2t ts;
        }
        root /tmp;
        add_header Cache-Control no-cache;
    }
```

Extra Info
------------------

    防火墙设置，配置1985端口可以被外网访问
    huang@vultr:~$ sudo iptables -A INPUT -m state --state NEW -m tcp -p tcp --dport 1985 -j ACCEPT




