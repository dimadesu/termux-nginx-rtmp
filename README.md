# termux-nginx-rtmp

This is an `nginx` build for [Termux](https://termux.dev/en/) that includes `nginx-rtmp-module`.

There will be additional instructions below on how to use ffmpeg with it to create RTMP to SRT relay.

It is also possible to transcode into HEVC.

## Pre-Requirements

```sh
# Not too sure why we root repo, is this right?
pkg install root-repo
```

- [Termux from F-Droid repo](https://github.com/termux/termux-app?tab=readme-ov-file#f-droid) (Google Play version is outdated. Goole is not too keen to allow all this Termux stuff for security reasons.)
- termux-services
- openssl-1.1 (legacy) // TODO: Why do we need this legacy version?
- gettext (to inject env variables into nginx config)

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

## Tweak nginx.conf

```sh
curl https://raw.githubusercontent.com/dimadesu/termux-nginx-rtmp/main/nginx-custom.conf > $PREFIX/etc/nginx/nginx.conf.template
envsubst < $PREFIX/etc/nginx/nginx.conf.template > $PREFIX/etc/nginx/nginx.conf
mkdir -p $PREFIX/www/static/ && curl https://raw.githubusercontent.com/dimadesu/termux-nginx-rtmp/main/stat.xsl > $PREFIX/www/static/stat.xsl
```

# --- Restart the phone ---

## Enable and Start Service

```sh
# Configure service to start on boot
sv-enable nginx

# Start service now
sv up nginx

# Start nginx now
nginx

# Check if service is running
sv status nginx
```

## Open Nginx stats in the browser

[http://0.0.0.0:8080/stat](http://0.0.0.0:8080/stat)

# --- ffmpeg stuff ---

It will read what is pushed into RTMP ingest of Nginx (pull RTMP) and push SRT where you need it.

## Install ffmpeg

```sh
apt install ffmpeg

# I think this one is to help with XML stats of Nginx
apt install libexpat
```

## Check IP

```sh
ifconfig
```

## Configure your video encoder to push to RTMP ingest of Nginx

```sh
rtmp://ip:1935/publish/live
```

## Run ffmpeg

Relay w/o trasncoding to HEVC.

```sh
ffmpeg -i rtmp://localhost:1935/publish/live -c:v copy -c:a copy -f mpegts srt://yourip:yourport?mode=caller 
```

If your phone is powerful enough you can try transcoding video to HEVC.

```sh
ffmpeg -i rtmp://localhost:1935/publish/live -c:v libx265 -crf 18 -c:a copy -f mpegts srt://yourip:yourport?mode=caller 
```

## Script that can restart ffmpeg if it exits/errors out

```sh
nano ffmpeg.sh
```

Paste script

```sh
while true; do
ffmpeg -i rtmp://localhost:1935/publish/live -c:v copy -c:a copy -f mpegts srt://ip:port?mode=caller
echo "FFmpeg exited. Restarting in 5 seconds."
sleep 5
done
```

Give executable permission.

```sh
chmod +x ffmpeg.sh
```

Run script.

```sh
./ffmpeg.sh
```




