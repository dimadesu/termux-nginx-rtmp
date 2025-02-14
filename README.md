# termux-nginx-rtmp

This is `nginx` build for [Termux](https://termux.dev/en/) that includes `nginx-rtmp-module`. _Plus_ ffmpeg scripts.

There will be additional instructions below on how to use ffmpeg on Android in Termux to read RTMP stream and push to SRT ingest.

It is also possible to transcode into HEVC.

Natually, it can read RTMP and push to RTMP ingest.

## What is this for?

**This allows running RTMP server on Android using Termux and rebroadcasting RTMP as SRT.**

This can be useful for outdoors streaming with action cameras as for now they only support streaming over RTMP. SRT seems to work better when sending stream over long distances.

In a more complex IRL setups this allows sending to self hosted SRT server.

I'm testing using this to make connection between action camera and [Belabox](https://belabox.net/) ingest more resilient to stutter as it's the weakest link in my setup.

Btw check out my unofficial SRT ingest for Belabox [https://github.com/dimadesu/srt-ingest-for-belabox](https://github.com/dimadesu/srt-ingest-for-belabox) as it can work with this too. You can feed your SRT stream phone a phone into SRT ingest of Belabox.

Ideally, distance between action camera and the phone should be fairly short to prevent radio interference.

At least with my phone (Samsung S20 FE) I've noticed the stream is more stable and resilient to stutter and RF interference when streaming from the phone VS. action cameras (tested with Dji Osmo Action 4 and GoPro 10).

Choose a reasonable bitrate to send from your action camera as ffmpeg will keep the same bitrate and try to send it over network.

## Prerequisites

Install [Termux from F-Droid repo](https://github.com/termux/termux-app?tab=readme-ov-file#f-droid). Google Play version is outdated.

Google is not too keen to allow all this Termux stuff for security reasons, I imagine, as anyone can copy-paste scripts from the Internet w/o knowing what they do.

But don't worry I worked out what most these scripts and commands do and they are fine.

```sh
# TODO: Why do we need root-repo? In the Termux docs it says it's for rooted phones. Instructions below do not require rooted phone
pkg install root-repo
```

## Install libraries

- `termux-services` to run Nginx as a service on boot
- `openssl-1.1` (legacy) // TODO: Is this needed for ffmpeg? Why do we need legacy version?
- `gettext` - injects environmental variable $PREFIX into Nginx config for path to XSL template

```sh
apt install termux-services openssl-1.1 gettext
ln -s $PREFIX/lib/openssl-1.1/libssl.so.1.1 $PREFIX/lib/libssl.so.1.1
ln -s $PREFIX/lib/openssl-1.1/libcrypto.so.1.1 $PREFIX/lib/libcrypto.so.1.1
```

## Install Nginx with RTMP module

```sh
apt remove nginx # remove any existing nginx installation
echo "deb https://muxable.github.io/termux-nginx-rtmp/ termux extras" > $PREFIX/etc/apt/sources.list.d/nginx-rtmp.list
apt update --allow-insecure-repositories
apt install nginx-rtmp
```

## Configure `nginx.conf`

These 2 lines create Nginx config and replace one path with $PREFIX var.

```sh
curl https://raw.githubusercontent.com/dimadesu/termux-nginx-rtmp/main/nginx-custom.conf > $PREFIX/etc/nginx/nginx.conf.template
envsubst < $PREFIX/etc/nginx/nginx.conf.template > $PREFIX/etc/nginx/nginx.conf
```

Create Nginx RTMP stats XSL template.

mkdir -p $PREFIX/www/static/ && curl https://raw.githubusercontent.com/dimadesu/termux-nginx-rtmp/main/stat.xsl > $PREFIX/www/static/stat.xsl

## Restart the phone

## Running as service

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

## Test by opening Nginx stats in the browser

[http://0.0.0.0:8080/stat](http://0.0.0.0:8080/stat)

# ffmpeg

It will read what is pushed into RTMP ingest of Nginx (pull RTMP) and push SRT where you need it.

## Install ffmpeg

```sh
apt install ffmpeg

# I think this one is to help parse XML stats of Nginx
apt install libexpat
```

## Check IP

```sh
ifconfig
```

## Configure your video encoder to push to RTMP ingest of Nginx

```sh
rtmp://IP_OF_YOUR_PHONE:1935/publish/live
```

## Run ffmpeg

Pull RTMP stream from Nginx, push to SRT ingest (w/o trasncoding to HEVC).

```sh
ffmpeg -i rtmp://localhost:1935/publish/live -c:v copy -c:a copy -f mpegts srt://IP_OF_YOUR_SRT_SERVER:PORT_NUMBER?mode=caller
```

If your phone is powerful you can try transcoding video to HEVC.
Pull RTMP stream from Nginx, encode as HEVC and push to SRT ingest.

```sh
ffmpeg -i rtmp://localhost:1935/publish/live -c:v libx265 -crf 18 -c:a copy -f mpegts srt://IP_OF_YOUR_SRT_SERVER:PORT_NUMBER?mode=caller
```

Pull RTMP stream from Nginx, push to RTMP ingest.

```sh
ffmpeg -i rtmp://localhost:1935/publish/live -c:v copy -c:a copy -f flv rtmp://IP_OF_YOUR_SRT_SERVER:1935/publish/live
```

## Script that can restart ffmpeg if it exits/errors out

```sh
nano ffmpeg.sh
```

Paste script

```sh
while true; do
ffmpeg -i rtmp://localhost:1935/publish/live -c:v copy -c:a copy -f mpegts srt://IP_OF_YOUR_SRT_SERVER:PORT_NUMBER?mode=caller
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
