---
layout: single
title:  "Share Space: Low-latency Desktop Sharing"
date:   2019-11-06 09:29:33 -0500
categories: video-streaming
---

## Background

The computer in my college dorm's common space was a thing of beauty. It was connected to a big TV, always on, and had a wireless keyboard-trackpad. Students used it to watch lectures, work on assignments, and most importantly relax together. Its convenience promoted its use, and its flexibility made it an indispensible collaboration tool.

But college is uniquely local. In reality, our connections in work and life are distributed. Conference calls and screen-sharing close some of that distance, but screen-sharing video quality is often low and without audio; and remote collaborators are unable to control the desktop being shared. Remote desktop applications like Chrome Remote Desktop and VNC solve remote control, but only for one user and with low video quality.

I built [Share Space](github.com/alexjball/share-space) to address these problems and to support the vision of a remote shared computer. Users join Share Space rooms using the Share Space web client and the room's address and passcode. Each room hosts a virtual desktop, streams high-quality video and audio to all members, and allows members to take over remote control of the desktop. [Share Space Host](https://github.com/alexjball/share-space-host) makes it easy to self-host room instances.

The most interesting part of this project has been achieving a low-latency (1-3 second) video stream of the desktop. Much of that work was panning for gold among the many specs and RFC's that make up web video. So I'll go over the implementation options for modern web video and then explain the design for Share Space's video streaming. 

## Modern Web Video

There are roughly two categories of video solutions.

### live or recorded, scalable, HTTP transport, 3-10s latency: [HLS](https://en.wikipedia.org/wiki/HTTP_Live_Streaming), [DASH](https://en.wikipedia.org/wiki/Dynamic_Adaptive_Streaming_over_HTTP), [low-latency HLS](https://developer.apple.com/documentation/http_live_streaming/protocol_extension_for_low-latency_hls_preliminary_specification), [CMAF](https://www.wowza.com/blog/low-latency-cmaf-chunked-transfer-encoding)

The Media Source Extensions define the `MediaSource` class, which can be used to pass video stream buffers from Javascript to the video playback element. This allows implementing streaming players in Javascript and is how modern players support HLS/DASH/etc.

### real-time, not scalable, TCP/UDP transport, <1s latency: [WebRTC](https://webrtc.org/)

WebRTC defines the `WebRTCPeerConnection` class, which can be used to set up RTP connections for media and data. RTP (Real-time Transport Protocol) supports lower latency and higher bandwidth than HTTP or WebSockets, and is how web-based video chat is implemented.

## Share Space Video Attempt 1: WebRTC

WebRTC initially seemed like the best option because of its latency claims. To serve the desktop stream, I would need a server that speaks WebRTC and that could share the encoded stream between all clients. The [node-webrtc](https://github.com/node-webrtc/node-webrtc) project provides a Node wrapper around [Chromium's WebRTC implementation](https://webrtc.org/native-code/native-apis/). Since browser WebRTC encodes video for each connection, though, the server would encode once for each of its clients, rather than sharing one encoded stream. Other options include the [GStreamer WebRTC](https://gstreamer.freedesktop.org/documentation/webrtc/index.html?gi-language=c) plugin, [Kurento Media Server](https://github.com/Kurento/kurento-media-server), and forking Chromium's implementation.

In every case, the WebRTC protocol stack introduced significant complexity to the streaming component. So instead, I decided to prototype a solution using MediaSource and WebSockets, which turned out to be simpler and good enough for the MVP.

WebRTC will be worth re-evaluating for video/voice chat features and further improving latency. Two paths forward:

- Use `node-webrtc` to pass the video stream over a data channel, replacing the WebSocket transport currently used.
- Investigate Kurento, which I found after implementation.

## Share Space Video Attempt 2: MediaSource and WebSockets

Traditional on-demand video streaming works by breaking up video files into segments and serving each segment over HTTP(S). Players first download a manifest with media information and a mapping from video timestamp ranges to segment URL's. Players then download the segments, buffering them as the video element decodes and displays the stream. This approach scales well since the segments can easily be cached by content delivery networks.

With this approach, latency is determined by:

1. App server: The time to encode and publish a segment for download
2. App client: The time to download and push a segment to the video element
3. Browser: The time to process and display a segment in the video element's buffer

Traditionally, the entire segment is encoded before being published, and browsers maintain a buffer of multiple segments to ensure smooth playback. With 5 second segments, this implies a latency of > 10 seconds. Modern solutions like [Low-latency HLS](https://www.theoplayer.com/blog/low-latency-hls-lhls) address this by using HTTP chunk transfer to stream the segments as they're being encoded, allowing < 5 second latencies.

I implemented that idea using WebSockets instead of HTTP chunk transfer. WebSockets provide a stream-oriented data channel, which makes it simple to pipe a video stream to clients.

## Conclusion

MediaSource and WebSockets provide a simple mechanism to serve low-latency video to web clients.

And I'd love to hear your recommendations for WebRTC server components!