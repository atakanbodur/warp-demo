# Warp
Segmented live media delivery protocol utilizing QUIC streams. See the [Warp draft](https://datatracker.ietf.org/doc/draft-lcurley-warp/).

Warp works by delivering each audio and video segment as a separate QUIC stream. These streams are assigned a priority such that old video will arrive last and can be dropped. This avoids buffering in many cases, offering the viewer a potentially better experience.

# Limitations
## Browser Support
This demo currently only works on Chrome for two reasons:

1. WebTransport support.
2. [Media underflow behavior](https://github.com/whatwg/html/issues/6359).

The ability to skip video abuses the fact that Chrome can play audio without video for up to 3 seconds (hardcoded!) when using MSE. It is possible to use something like WebCodecs instead... but that's still Chrome only at the moment.

## Streaming
This demo works by reading pre-encoded media and sleeping based on media timestamps. Obviously this is not a live stream; you should plug in your own encoder or source.

The media is encoded on disk as a LL-DASH playlist. There's a crude parser and I haven't used DASH before so don't expect it to work with arbitrary inputs.

## QUIC Implementation
This demo uses a fork of [quic-go](https://github.com/lucas-clemente/quic-go). There are two critical features missing upstream:

1. ~~[WebTransport](https://github.com/lucas-clemente/quic-go/issues/3191)~~
2. [Prioritization](https://github.com/lucas-clemente/quic-go/pull/3442)

## Congestion Control
This demo uses a single rendition. A production implementation will want to:

1. Change the rendition bitrate to match the estimated bitrate.
2. Switch renditions at segment boundaries based on the estimated bitrate.
3. or both!

Also, quic-go ships with the default New Reno congestion control. Something like [BBRv2](https://github.com/lucas-clemente/quic-go/issues/341) will work much better for live video as it limits RTT growth.


# Setup
## Requirements
* Go
* ffmpeg
* openssl
* Chrome Canary

## Media
This demo simulates a live stream by reading a file from disk and sleeping based on media timestamps. Obviously you should hook this up to a real live stream to do anything useful.

Download your favorite media file:
```
wget http://commondatastorage.googleapis.com/gtv-videos-bucket/sample/BigBuckBunny.mp4 -O media/source.mp4
```

Use ffmpeg to create a LL-DASH playlist. This creates a segment every 2s and MP4 fragment every 10ms.
```
ffmpeg -i media/source.mp4 -f dash -use_timeline 0 -r:v 24 -g:v 48 -keyint_min:v 48 -sc_threshold:v 0 -tune zerolatency -streaming 1 -ldash 1 -seg_duration 2 -frag_duration 0.01 -frag_type duration media/playlist.mpd
```

You can increase the `frag_duration` (microseconds) to slightly reduce the file size in exchange for higher latency.

## TLS
Unfortunately, QUIC mandates TLS and makes local development difficult.

If you have a valid certificate you can use it instead of self-signing. The go binaries take a `-tls-cert` and `-tls-key` argument. Skip the remaining steps in this section and use your hostname instead.

Otherwise, use [mkcert](https://github.com/FiloSottile/mkcert) to install a self-signed CA:
```
mkcert -install
```

With no arguments, the server will generate self-signed cert using this root CA.

## Server
The Warp server defaults to listening on UDP 4443. It supports HTTP/3 and WebTransport, pushing media over WebTransport streams once a connection has been established. A more refined implementation would load content based on the WebTransport URL or some other messaging scheme.

```
cd server
go run main.go
```

## Web Player
The web assets need to be hosted with a HTTPS server. If you're using a self-signed certificate, you may need to ignore the security warning in Chrome (Advanced -> proceed to localhost).

```
cd player
yarn install
yarn serve
```

These can be accessed on `https://127.0.0.1:4444` by default.

If you use a custom domain for the Warp server, make sure to override the server URL with the `url` query string parameter, e.g. `https://localhost:4444/?url=https://warp.demo`.

## Chrome
Now we need to make Chrome accept these certificates, which normally would involve trusting a root CA but this was not working with WebTransport when I last tried.

Instead, we need to run a *fresh instance* of Chrome, instructing it to allow our self-signed certificate. This command will not work if Chrome is already running, so it's easier to use Chrome Canary instead.

Launch a new instance of Chrome Canary:
```
/Applications/Google\ Chrome\ Canary.app/Contents/MacOS/Google\ Chrome\ Canary --allow-insecure-localhost --origin-to-force-quic-on=127.0.0.1:4443 https://127.0.0.1:4444
```