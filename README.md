# termux-nginx-rtmp

This is an `nginx` build for Termux that includes `nginx-rtmp-module`.

## Pre-Requirements

```sh
pkg install root-repo
```
+ termux from f-droid repo (google play is outdated)
+ termux-services
+ openssl-1.1 (legacy)
+ gettext (to inject env variables into nginx config)
```sh
apt install termux-services openssl-1.1 gettext
ln -s $PREFIX/lib/openssl-1.1/libssl.so.1.1 $PREFIX/lib/libssl.so.1.1
ln -s $PREFIX/lib/openssl-1.1/libcrypto.so.1.1 $PREFIX/lib/libcrypto.so.1.1
```

## Installation

```sh
apt remove nginx # remove any existing nginx installation.
echo "deb https://muxable.github.io/termux-nginx-rtmp/ termux extras" > $PREFIX/etc/apt/sources.list.d/nginx-rtmp.list
apt update --allow-insecure-repositories
apt install nginx-rtmp
```

# Tweak nginx.conf
```sh
curl https://raw.githubusercontent.com/NeepOwO/termux-nginx-rtmp/main/nginx-custom.conf > $PREFIX/etc/nginx/nginx.conf.template
envsubst < $PREFIX/etc/nginx/nginx.conf.template > $PREFIX/etc/nginx/nginx.conf
mkdir -p $PREFIX/www/static/ && curl https://raw.githubusercontent.com/NeepOwO/termux-nginx-rtmp/main/stat.xsl > $PREFIX/www/static/stat.xsl
```
# Restart phone

## Enable and Start Service
```sh
sv-enable nginx
sv up nginx
nginx
```
## test link
```sh
0.0.0.0:8080/stat 
```
# install ffmpeg
```sh
apt install ffmpeg
apt install libexpat
```
# Start stream
## check ip
```sh
ifconfig
```
## streamlink
```sh
rtmp://ip:1935/publish/live
```
## ffmpeg start string

without coding to HEVC
```sh
ffmpeg -i rtmp://localhost:1935/publish/live -c:v copy -c:a copy -f mpegts srt://yourip:yourport?mode=caller 
```

with coding to HEVC
```sh
ffmpeg -i rtmp://localhost:1935/publish/live -c:v libx265 -crf 18 -c:a copy -f mpegts srt://yourip:yourport?mode=caller 
```




