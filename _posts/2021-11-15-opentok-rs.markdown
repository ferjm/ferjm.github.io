---
layout:     post
title:      "opentok-rs: easy WebRTC with Rust"
date:       2021-11-17
summary:    Introducing opentok-rs, Rust bindings of the OpenTok SDK
categories: ["rust", "opentok", "webrtc"]
author: "Fernando Jiménez"
---

[OpenTok](https://www.vonage.com/communications-apis/video/) is [Vonage](https://www.vonage.com)'s
(formerly [TokBox's](https://en.wikipedia.org/wiki/TokBox))
[PaaS](https://en.wikipedia.org/wiki/Platform_as_a_service) (Platform as a Service)
that enables developers to easily build custom video experiences within any mobile,
web, or desktop application, on top of a [WebRTC](https://en.wikipedia.org/wiki/WebRTC)
stack.

One of the customer projects that I am working on at [Igalia](https://igalia.com)
requires publishing and subscribing to streams to and from OpenTok sessions.
The main application of this project needs to run on a Linux box and Vonage already
provides a nice [OpenTok C++ SDK for Linux](https://tokbox.com/developer/sdks/linux/).
However, the entire application for this customer project is written in Rust so,
together with my colleague [Philippe Normand](https://base-art.net/), we decided to
write Rust bindings for the OpenTok C++ SDK.

[opentok-rs](https://github.com/ferjm/opentok-rs) contains the result of this work.
There you can find the [FFI](https://doc.rust-lang.org/book/ch19-01-unsafe-rust.html#calling-rust-functions-from-other-languages)
bindings, mostly generated with [bindgen](https://rust-lang.github.io/rust-bindgen/),
and the safe wrapper API.

We recently published a first version in [crates.io](https://crates.io/crates/opentok).

There is really not much documentation yet, apart from the [rustdoc](https://doc.rust-lang.org/rustdoc/what-is-rustdoc.html)
published [here](https://ferjm.github.io/opentok-rs/opentok/), that is mostly a copy & paste
of the C++ documentation. But there are a few
[examples](https://github.com/ferjm/opentok-rs/tree/main/examples/src/bin)
that demonstrate how easy and fast you can write your own custom video experiences.

# Basic video chat application

With opentok-rs you can write a very basic video chat application like
[this one](https://github.com/ferjm/opentok-rs/blob/09ac4d8f38dcb5aa443308a2f9e82444530e745e/examples/src/bin/basic_video_chat.rs)
in only a few dozen lines of code.

<div style="text-align:center;">
<iframe width="560" height="315" src="https://www.youtube-nocookie.com/embed/i48iw0GYgcA" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</div>

If you are not familiar with the basic concepts of OpenTok, I recommend reading
the [official documentation](https://tokbox.com/developer/guides/basics/) at Vonage's developer
site.

In a nutshell, all OpenTok activity occurs within a session, which is somewhat like a "room"
where clients interact with one another in real-time. Each participant in a session can publish
streams to the session or subscribe to other participants' streams.

To connect to OpenTok sessions you need its identifier and a token. For testing purposes, you
can obtain a session ID and a token from the project page in your [Vonage Video API](https://tokbox.com/developer/) account.
However, in a production application, you will need to dynamically obtain the session ID and
token from a web service that uses one of the [Vonage Video API server SDKs](https://tokbox.com/developer/sdks/server/).

For a basic chat application you need to create a [Publisher](https://ferjm.github.io/opentok-rs/opentok/publisher/struct.Publisher.html)
instance, to publish your video stream, and a [Subscriber](https://ferjm.github.io/opentok-rs/opentok/subscriber/struct.Subscriber.html)
instance, likely in a different thread, to subscribe to the rest of the streams in the session. Each entity
may connect to the session separately.

### Publisher

The OpenTok SDK is heavily based on callbacks. Starting with the session, you need to provide
a [SessionCallbacks](https://ferjm.github.io/opentok-rs/opentok/session/struct.SessionCallbacks.html)
instance to the [Session](https://ferjm.github.io/opentok-rs/opentok/session/struct.Session.html) constructor.
For the sake of simplicity, we only care about the `on_connected` and `on_error` callbacks in this case.

You also need to provide the session credentials. This is the Vonage API key, the session ID and its token.

```rust
let session_callbacks = SessionCallbacks::builder()
    .on_connected(move |session| {
        // At this point, we can start publishing
        session.publish(&*publisher.lock().unwrap())
    })
    .on_error(|_, error, _| {
        eprintln!("on_error {:?}", error);
    })
    .build();
let session = Session::new(
    &credentials.api_key,
    &credentials.session_id,
    session_callbacks,
)?;
session.connect(&credentials.token)?;
```

The Publisher constructor gets a [PublisherCallbacks](https://ferjm.github.io/opentok-rs/opentok/publisher/struct.PublisherCallbacks.html)
instance and optionally a [VideoCapturer](https://ferjm.github.io/opentok-rs/opentok/video_capturer/struct.VideoCapturer.html) instance.
If you do not provide a custom video capturer, the default one capturing audio and video from your local mic and webcam will be used.

```rust
let publisher_callbacks = PublisherCallbacks::builder()
    .on_stream_created(move |_, stream| {
        println!("Publishing stream with ID {}", stream.id());
    })
    .on_error(|_, error, _| {
        eprintln!("on_error {:?}", error);
    })
    .build();
let publisher = Arc::new(Mutex::new(Publisher::new(
    "publisher" /* Publisher name */,
    None, /* Use WebRTC's video capturer */,
    publisher_callbacks,
)));

```

The [basic video chat example](https://github.com/ferjm/opentok-rs/blob/09ac4d8f38dcb5aa443308a2f9e82444530e745e/utils/src/publisher.rs#L122)
demonstrates how to add a custom video capturer. [In this case](https://github.com/ferjm/opentok-rs/blob/main/utils/src/capturer.rs#L23),
it uses a [GStreamer](https://gstreamer.freedesktop.org/) [videotestsrc](https://gstreamer.freedesktop.org/documentation/videotestsrc/index.html?gi-language=c)
element to produce test video data. You can use whatever mechanism to produce video that you prefer though.

### Subscriber

The subscriber part is somewhat similar. It needs to connect to the session, providing the credentials
and the session callbacks. In this case, the callback that we care about the most is the `on_stream_received`
callback. Within this callback, you can set the stream on your Subscriber instance and instruct the session
to use it.

```rust
let session_callbacks = SessionCallbacks::builder()
    .on_stream_received(move |session, stream| {
        if subscriber.set_stream(stream).is_ok() {
            if let Err(e) = session.subscribe(&subscriber) {
                eprintln!("Could not subscribe to session {:?}", e);
            }
        }
    })
    .on_error(|_, error, _| {
        eprintln!("on_error {:?}", error);
    })
    .build();
```

The Subscriber gets the [video frames](https://ferjm.github.io/opentok-rs/opentok/video_frame/struct.VideoFrame.html)
through repeated calls to the `on_render_frame` callback.

```rust
let subscriber_callbacks = SubscriberCallbacks::builder()
    .on_render_frame(move |_, frame| {
        let width = frame.get_width().unwrap() as u32;
        let height = frame.get_height().unwrap() as u32;

        let get_plane_size = |format, width: u32, height: u32| match format {
            FramePlane::Y => width * height,
            FramePlane::U | FramePlane::V => {
                let pw = (width + 1) >> 1;
                let ph = (height + 1) >> 1;
                pw * ph
            }
            _ => unimplemented!(),
        };

        let offset = [
            0,
            get_plane_size(FramePlane::Y, width, height) as usize,
            get_plane_size(FramePlane::Y, width, height) as usize
                + get_plane_size(FramePlane::U, width, height) as usize,
        ];

        let stride = [
            frame.get_plane_stride(FramePlane::Y).unwrap(),
            frame.get_plane_stride(FramePlane::U).unwrap(),
            frame.get_plane_stride(FramePlane::V).unwrap(),
        ];
        renderer_
            .lock()
            .unwrap()
            .as_ref()
            .unwrap()
            .push_video_buffer(
                frame.get_buffer().unwrap(),
                frame.get_format().unwrap(),
                width,
                height,
                &offset,
                &stride,
            );
    })
    .on_error(|_, error, _| {
        eprintln!("on_error {:?}", error);
    })
    .build();
```

The snippet above uses a [video renderer](https://github.com/ferjm/opentok-rs/blob/main/utils/src/renderer.rs)
based on the GStreamer [autovideosink](https://gstreamer.freedesktop.org/documentation/autodetect/autovideosink.html?gi-language=c#autovideosink-page)
element. But just like with the custom video capturer, you can use whatever you like to render your video frames.

### Audio

The OpenTok SDK handles audio and video in different ways. While video streams are independently tied to each
publisher and each subscriber in a session, audio is tied to a global
[audio device](https://ferjm.github.io/opentok-rs/opentok/audio_device/struct.AudioDevice.html)
that is shared by all publishers and subscribers.

This design imposes two hard limitations:

- There is no way to obtain the independent audio stream from each participant.
OpenTok provides a single audio stream which is a mix of every participant's audio,
so there is no way to do things like speech-to-text, moderation or any kind of audio
processing per participant, unless you create a somewhat
[complex workaround](https://github.com/opentok/opentok-linux-sdk-samples/issues/25#issuecomment-916155032)
where you run each audio subscriber in its own dedicated process.

- It is not possible to run two instances of the OpenTok SDK in the same process.
A second instance of the OpenTok SDK overwrites the audio callbacks set from the previous instance.

Vonage claimed to be working on improving this design.

# There is more

Everything in opentok-rs is meant to run on client applications, but as mentioned before, Vonage also
provides [server side OpenTok SDKs](https://tokbox.com/developer/sdks/server/).

[opentok-server-rs](https://github.com/ferjm/opentok-server-rs) wraps a minimal subset of the OpenTok
REST API. It lets developers to securely create sessions and generate tokens for their OpenTok applications.

I started it only to be able to write automatic tests for opentok-rs, so the functionality is limited and will
hopefully be extended soon.

# Acknowledgements
* I would like to thanks [Televic Conference](https://www.televic-conference.com/en) for sponsoring this work.
* Huge thanks to [José Antonio Olivera](https://github.com/joliveraortega) from Vonage for
his continuous guidance and support while writting the bindings.
