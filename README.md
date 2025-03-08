# Rebroadcast RTMP stream from action camera as SRT using Android phone

## What is this for?

**This allows running RTMP server on Android using Termux and rebroadcasting RTMP as SRT.**

This can be useful when streaming outdoors with action cameras as for now they only support streaming over RTMP. SRT seems to work better when sending stream over long distances.

In a more complex streaming setups this allows sending to self hosted SRT server.

I'm testing using this to make connection between action camera and [Belabox](https://belabox.net/) ingest more resilient to stutter as it's the weakest link in my setup.

Btw check out my unofficial SRT ingest for Belabox [https://github.com/dimadesu/srt-ingest-for-belabox](https://github.com/dimadesu/srt-ingest-for-belabox) as it can work with this too. You can feed your SRT stream from a phone into SRT ingest of Belabox. If you don't want to bother installing SRT ingest for Belabox even rebroadcasting RTMP as RTMP into RTMP ingest of Belabox seems to make signal stronger too.

Ideally, distance between action camera and the phone should be fairly short to prevent radio interference.

At least with my phone (Samsung S20 FE) I've noticed the stream is more stable and resilient to stutter and RF interference when streaming from the phone VS. action cameras (tested with Dji Osmo Action 4 and GoPro 10).

Choose a reasonable bitrate to send from your action camera as ffmpeg will keep the same bitrate and try to send it over network.

## Disclaimer. This isn't mine

It's based on this guy's fork https://github.com/NeepOwO/termux-nginx-rtmp and [his video](https://www.youtube.com/watch?v=977_AtGC2sQ) (in Russian).

I just cleaned up README and provided detailed explanations. Also, I also added extra TODO comments when I was trying to make sense of it and understand how it works.

Hopefully, we can further improve upon this and polish it. I wonder if we can swap Nginx for [Node Media Server](https://www.npmjs.com/package/node-media-server) as Node is [supported by Termux](https://wiki.termux.com/index.php?title=Node.js) or smth like MediaMTX.

# termux-nginx-rtmp

This is `nginx` build for [Termux](https://termux.dev/en/) that includes `nginx-rtmp-module`. _Plus_ ffmpeg scripts.

There will be additional instructions below on how to use ffmpeg on Android in Termux to read RTMP stream and push to SRT ingest.

It is also possible to transcode into HEVC (needs more testing).

Natually, it can read RTMP and push to RTMP ingest too. This can be usefull too for some cases detailed below.

## Prerequisites

Install [Termux from F-Droid repo](https://github.com/termux/termux-app?tab=readme-ov-file#f-droid). Google Play version is outdated.

Google is not too keen to allow all this Termux stuff for security reasons, I imagine, as anyone can copy-paste scripts from the Internet w/o knowing what they do.

But don't worry I worked out what most these scripts and commands do and they are fine.

```sh
# TODO: Why do we need root-repo? In the Termux docs it says it's for rooted phones. Instructions below do not require rooted phone
# TODO: Try running w/o this
pkg install root-repo
```

## Install libraries

- `termux-services` to run Nginx as a service on boot
- `openssl-1.1` (legacy) // TODO: Why do we need this? Why legacy version? Can we get rid of this? This seems to be needed if Nginx will use HTTPS TBC
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

```
mkdir -p $PREFIX/www/static/ && curl https://raw.githubusercontent.com/dimadesu/termux-nginx-rtmp/main/stat.xsl > $PREFIX/www/static/stat.xsl
```

## Restart the phone

## Running as service

```sh
# Configure service to start on boot
sv-enable nginx

# Start service now
sv up nginx

# Check if service is running
sv status nginx
```

## Test by opening Nginx stats in the browser

// TODO: Can this be changed to `localhost`?

[http://0.0.0.0:8080/stat](http://0.0.0.0:8080/stat)


# ffmpeg

It can read what is pushed into RTMP ingest of Nginx (pull RTMP) and push to SRT or RTMP ingest.

## Install ffmpeg

```sh
apt install ffmpeg
```

## Install libexpat

I think this one is to help parse XML stats of Nginx.

```
# TODO: Try running w/o it. I think it's not used
apt install libexpat
```

## Find out your phone's IP address

Most likely you'll be running hotspot on your phone, so run this command first

```sh
ifconfig
```

and look for IP under `swlan0` > `inet`.

## Configure your camera / video encoder to push to RTMP ingest of Nginx

```sh
rtmp://IP_OF_YOUR_PHONE:1935/publish/live
```

## Run ffmpeg

Pick one of the options bellow.

- Option 1. **RTMP-SRT H264.** Pull RTMP stream from Nginx, push to SRT ingest (w/o trasncoding to HEVC).

  ```sh
  ffmpeg -i rtmp://localhost:1935/publish/live -c:v copy -c:a copy -f mpegts srt://IP_OF_YOUR_SRT_SERVER:PORT_NUMBER?mode=caller
  ```

- Option 2. **RTMP-SRT HEVC.** Pull RTMP stream from Nginx, encode as HEVC and push to SRT ingest.

  If your phone is powerful you can try transcoding video to HEVC.
  **I've tested this and the performance is pretty bad.** The solution is to use Mediacodec to get hardware accelation. Please refer to instructions [here](https://github.com/dimadesu/android-rtmp-ingest).
  
  ```sh
  ffmpeg -i rtmp://localhost:1935/publish/live -c:v libx265 -crf 18 -c:a copy -f mpegts srt://IP_OF_YOUR_SRT_SERVER:PORT_NUMBER?mode=caller
  ```

- Option 3. **RTMP-RTMP H264.** Pull RTMP stream from Nginx, push to RTMP ingest.

  ```sh
  ffmpeg -i rtmp://localhost:1935/publish/live -c:v copy -c:a copy -f flv rtmp://IP_OF_YOUR_SRT_SERVER:1935/publish/live
  ```

## Script that can restart ffmpeg if it exits or errors out

For convenience you can create a script and manually run it.

```sh
nano ffmpeg.sh
```

Paste script.

```sh
while true; do
ffmpeg -i rtmp://localhost:1935/publish/live -c:v copy -c:a copy -f mpegts srt://IP_OF_YOUR_SRT_SERVER:PORT_NUMBER?mode=caller
echo "FFmpeg exited. Restarting in 5 seconds."
sleep 5
done
```

Give executable permissions.

```sh
chmod +x ffmpeg.sh
```

Run script.

```sh
./ffmpeg.sh
```
