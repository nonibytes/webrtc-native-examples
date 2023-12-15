Translated from original article in chinese: [here](./1-2-videostream.cn.md)

## WebRTC Video Stream

This chapter explains how to send and receive video streams using WebRTC.

Similarly, two peerconnections in the same process are used for sending and receiving video streams.

To avoid involving "camera capture" operations and to make the example code run on all platforms, I manually constructed RGB data in red, orange, yellow, green, cyan, blue, and purple as each frame of the video stream.

**Note: The actual format is I420 video data, but to avoid creating more barriers to understanding for beginners, it will be referred to as RGB data here.**

### Video Codec Factory

As mentioned in the previous section, creating a PeerConnection requires a corresponding peerconnectionfactory (abbreviated as factory).

The factory is created using the function `webrtc::CreatePeerconnctionFactory(params)`, which has many parameters. This section briefly introduces the 7th and 8th parameters: VideoEncoderFactory and VideoDecoderFactory.

When we create a video stream on a peerconnection, WebRTC queries the factory for supported video encoder types and includes the supported encoders in the SDP.

When WebRTC sends RGB data, it uses the encoder inside the factory for encoding.

When WebRTC receives "encoded" video frames from a remote source, it uses the factory's internal decoder for decoding.

VideoEncoderFactory is an interface class defined by WebRTC. Users can implement it customarily or use the default implementation provided by WebRTC.

```C++
// Omitting some non-key functions
class VideoEncoderFactory {
  // Returns a list of supported video formats in order of preference, for signaling etc.
  virtual std::vector<SdpVideoFormat> GetSupportedFormats() const = 0;  // Returns supported encoder types (SDP format)

  // Creates a VideoEncoder for the specified format.
  virtual std::unique_ptr<VideoEncoder> CreateVideoEncoder(             // Returns a video encoder of a specified type (negotiated and supported by both sides)
      const SdpVideoFormat& format) = 0;
};
```

For beginners just getting started with WebRTC, this is all you need to know. I will continue to explain how to implement custom encoders and decoders (hardware acceleration) later.

### Adding Video Streams to PeerConnection

Use the following code to add a "video source" to a peerconnection.
```C++
video_track = factory->CreateVideoTrack("video", video_source);
peerconnection->AddTrack(video_track);
```

This video source is an abstract class, and users must implement its prescribed virtual functions.

```C++
template <typename VideoFrameT>
class VideoSourceInterface {
 public:
  virtual ~VideoSourceInterface() = default;

  virtual void AddOrUpdateSink(VideoSinkInterface<VideoFrameT>* sink,
                               const VideoSinkWants& wants) = 0;
  // RemoveSink must guarantee that at the time the method returns,
  // there are no current and no future calls to VideoSinkInterface::OnFrame.
  virtual void RemoveSink(VideoSinkInterface<VideoFrameT>* sink) = 0;
};
```
`addxxxSink` is essentially setting a key callback object, `VideoSinkInterface`:

```C++
template <typename VideoFrameT>
class VideoSinkInterface {
 public:
  virtual ~VideoSinkInterface() = default;

  virtual void OnFrame(const VideoFrameT& frame) = 0;

  // Should be called by the source when it discards the frame due to rate
  // limiting.
  virtual void OnDiscardedFrame() {}
};
```

When we produce a video frame, by setting the incoming sink and using `VideoSinkInterface::OnFrame(frame)` to notify WebRTC, WebRTC will then go on to encode and transmit.

Pseudocode as follows:
```C++
class VideoSourceMock : public rtc::VideoSourceInterface<webrtc::VideoFrame> {
public:
    void start(frame)
    {
        while(true) {
            std::this_thread::sleep_for(10ms);
            auto video_frame = get_frame();   // Assume get_frame() can return a frame of video image
            broadcaster_.OnFrame(video_frame);// Deliver to WebRTC
        }
    }
private:
    void AddOrUpdateSink(rtc::VideoSinkInterface<webrtc::VideoFrame>* sink,
                                 const rtc::VideoSinkWants& wants) override {
        broadcaster_.AddOrUpdateSink(sink, wants);
        (void) video_adapter_; //we willn't use adapter at this demo
    }
    void RemoveSink(rtc::VideoSinkInterface<webrtc::VideoFrame>* sink) override {
        broadcaster_.RemoveSink(sink);
        (void) video_adapter_; //we willn't use adapter at this demo
    }
private:
    rtc::VideoBroadcaster broadcaster_;
    cricket::VideoAdapter video_adapter_;
};
```
On the sender side, setting a video track with a video source like above in peerconnection is enough to deliver

 "video frames" to WebRTC. The rest of the encoding and transmission tasks are automatically completed by WebRTC.

### Changes in SDP

When we add a video stream to a peerconnection, an extra item appears in the SDP during create offer:

```shell
v=0
o=- 5204290053496113649 2 IN IP4 127.0.0.1
s=-
t=0 0
a=group:BUNDLE 0
a=msid-semantic: WMS stream1
m=video 9 UDP/TLS/RTP/SAVPF 96 97 98 99 100 101 127 120 125 119 124 107 108 109 123 118 122 117 114
c=IN IP4 0.0.0.0
a=rtcp:9 IN IP4 0.0.0.0
a=ice-ufrag:qE9e
a=ice-pwd:vrZjsfbZ8njxNamShhas60+L
a=ice-options:trickle
a=fingerprint:sha-256 59:BC:FB:DA:29:42:CB:FF:99:BD:BC:92:C1:3B:2A:D1:EC:6E:1A:2F:17:CB:87:56:B7:9B:A4:81:54:59:66:31
a=setup:actpass
a=mid:0
### The following are fields related to video formats
a=extmap:1 urn:ietf:params:rtp-hdrext:toffset
a=extmap:2 http://www.webrtc.org/experiments/rtp-hdrext/abs-send-time
a=extmap:3 urn:3gpp:video-orientation
a=extmap:4 http://www.ietf.org/id/draft-holmer-rmcat-transport-wide-cc-extensions-01
a=extmap:5 http://www.webrtc.org/experiments/rtp-hdrext/playout-delay
a=extmap:6 http://www.webrtc.org/experiments/rtp-hdrext/video-content-type
a=extmap:7 http://www.webrtc.org/experiments/rtp-hdrext/video-timing
a=extmap:8 http://www.webrtc.org/experiments/rtp-hdrext/color-space
a=extmap:9 urn:ietf:params:rtp-hdrext:sdes:mid
a=extmap:10 urn:ietf:params:rtp-hdrext:sdes:rtp-stream-id
a=extmap:11 urn:ietf:params:rtp-hdrext:sdes:repaired-rtp-stream-id
a=sendrecv
a=msid:stream1 video
a=rtcp-mux
a=rtcp-rsize
### vp8
a=rtpmap:96 VP8/90000
a=rtcp-fb:96 goog-remb
a=rtcp-fb:96 transport-cc
a=rtcp-fb:96 ccm fir
a=rtcp-fb:96 nack
a=rtcp-fb:96 nack pli
a=rtpmap:97 rtx/90000
a=fmtp:97 apt=96
### vp9
a=rtpmap:98 VP9/90000
a=rtcp-fb:98 goog-remb
a=rtcp-fb:98 transport-cc
a=rtcp-fb:98 ccm fir
a=rtcp-fb:98 nack
a=rtcp-fb:98 nack pli
a=fmtp:98 profile-id=0
a=rtpmap:99 rtx/90000
a=fmtp:99 apt=98
a=rtpmap:100 VP9/90000
a=rtcp-fb:100 goog-remb
a=rtcp-fb:100 transport-cc
a=rtcp-fb:100 ccm fir
a=rtcp-fb:100 nack
a=rtcp-fb:100 nack pli
a=fmtp:100 profile-id=2
a=rtpmap:101 rtx/90000
a=fmtp:101 apt=100
### h264
a=rtpmap:127 H264/90000
a=rtcp-fb:127 goog-remb
a=rtcp-fb:127 transport-cc
a=rtcp-fb:127 ccm fir
a=rtcp-fb:127 nack
a=rtcp-fb:127 nack pli
a=fmtp:127 level-asymmetry-allowed=1;packetization-mode=1;profile-level-id=42001f
a=rtpmap:120 rtx/90000
a=fmtp:120 apt=127
a=rtpmap:125 H264/90000
a=rtcp-fb:125 goog-remb
a=rtcp-fb:125 transport-cc
a=rtcp-fb:125 ccm fir
a=rtcp-fb:125 nack
a=rtcp-fb:125 nack pli
a=fmtp:125 level-asymmetry-allowed=1;packetization-mode=0;profile-level-id=42001f
a=rtpmap:119 rtx/90000
a=fmtp:119 apt=125
a=rtpmap:124 H264/90000
a=rtcp-fb:124 goog-remb
a=rtcp-fb:124 transport-cc
a=rtcp-fb:124 ccm fir
a=rtcp-fb:124 nack
a=rtcp-fb:124 nack pli
a=fmtp:124 level-asymmetry-allowed=1;packetization-mode=1;profile-level-id=42e01f
a=rtpmap:107 rtx/90000
a=fmtp:107 apt=124
a=rtpmap:108 H264/90000
a=rtcp-fb:108 goog-remb
a=rtcp-fb:108 transport-cc
a=rtcp-fb:108 ccm fir
a=rtcp-fb:108 nack
a=rtcp-fb:108 nack pli
a=fmtp:108 level-asymmetry-allowed=1;packetization-mode=0;profile-level-id=42e01f
a=rtpmap:109 rtx/90000
a=fmtp:109 apt=108
a=rtpmap:123 AV1X/90000
a=rtcp-fb:123 goog-remb
a=rtcp-fb:123 transport-cc
a=rtcp-fb:123 ccm fir
a=rtcp-fb:123 nack
a=rtcp-fb:123 nack pli
a=rtpmap:118 rtx/90000
a=fmtp:118 apt=123
a=rtpmap:122 red/90000
a=rtpmap:117 rtx/90000
a=fmtp:117 apt=122
a=rtpmap:114 ulpfec/90000
a=ssrc-group:FID 3124827794 1259095527
a=ssrc:3124827794 cname:DDQzIip6txmbJUKJ
a=ssrc:3124827794 msid:stream1 video
a=ssrc:3124827794 mslabel:stream1
a=ssrc:3124827794 label:video
a=ssrc:1259095527 cname:DDQzIip6txmbJUKJ
a=ssrc:1259095527 msid:stream1 video
a=ssrc:1259095527 mslabel:stream1
a=ssrc:1259095527 label:video
```

Similarly, we still selectively ignore many fields. Only focus on the fields "a=rtpmap:96 vp8/90000", along with vp9 and h264.

It indicates that the sender supports vp8, vp9, and h264 encoding (h265 is not supported due to technical constraints). These are the three encoding methods natively supported by WebRTC.

When the recipient receives this offer, it must give an answer according to its decoding capability, writing in the answer which of these three encoding formats it can support.

If all three are supported, the sender will choose the format listed first in the answer to create the encoder.

In this example, as the sender and receiver use the same WebRTC code and support all three formats, a vp8 encoder is created. The "answer" is not listed here.

### Transmitting Video Stream

In the previous DataChannel section, I detailed the SDP and candidate interaction processes. Once these processes are successfully executed, the channel is naturally established.

At this point, WebRTC automatically opens the route between the encoder and the "video sink". The video frames delivered in "broadcaster_.onframe()" reach the encoder, then go down layer by layer, eventually being split into multiple RTP format UDP (or possibly TCP) packets sent over the network.

How exactly this process is executed will be introduced in later chapters. Source code analysis is always somewhat dry, and there's no need to read it if you're just using WebRTC.

### Receiving Video Frames

Returning to the SDP interaction process. When the recipient confirms the sender's offer, WebRTC informs the user through a callback: the sender has created a video stream.

```C++
    void OnAddStream(rtc::scoped_refptr<webrtc::MediaStreamInterface> stream) override{
        std::cout<<"[info] on add stream, id:"<<stream->id()<<std::endl;
    }
    // The key function is this
    void OnAddTrack(rtc::scoped_refptr<webrtc::RtpReceiverInterface> receiver,
            const std::vector<rtc::scoped_refptr<webrtc::MediaStreamInterface>>& streams) override {
        auto track = receiver->track().get();
        if(track->kind() == "video" && video_receiver_) {
            auto cast_track = static_cast<webrtc::VideoTrackInterface*>(track);
            cast_track->AddOrUpdateSink(video_receiver_.get(), rtc::VideoSinkWants());
        }
    }
```

In the "OnAddTrack" function, WebRTC passes a receiver object, on which we can get a "track (corresponding to the sender's)". 

If we want to receive this video stream, we need to add a "sink" (video_receiver_) to this "track", which must be implemented following the interface specified by WebRTC:

```C++
class VideoStreamReceiver : public rtc::VideoSinkInterface<webrtc::VideoFrame> {
public:
    void OnFrame(const webrtc::VideoFrame& frame) override {
        // Binary data (RGB) can be obtained from this frame object; just examine its data structure
    }
};
```

WebRTC informs the user of every received frame through this interface.

If we write the data in the example to a file and open it with the corresponding video software, you will see "red, orange, yellow, green, cyan, blue, purple" playing at 30 frames.

This example does not demonstrate how to render.

### End

Source Code: https://github.com/MemeTao/webrtc-native-samples/blob/master/src/video-channel

In this example, to help beginners better understand the video stream interaction process in WebRTC:

* It does not involve screen capturing, recording, rendering, etc.
* Although WebRTC specifies the video data format as I420, I described it as RGB. The code for converting between RGB and I420 is also in this example, and readers need to understand this code by reading relevant materials.