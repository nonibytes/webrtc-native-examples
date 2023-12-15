Translated from oroginal article in chinese: [here](./1-3-audio-stream.cn.md)

## WebRTC Audio Stream

This chapter introduces how to send and receive audio streams using WebRTC. Some of the knowledge points involved in this chapter have been covered in the previous chapter, so I won't repeat them here.

Similarly, two peerconnections in the same process are used to send and receive audio streams.

### AudioEncoderFactory

Like video, I will not go into detail here.

### Adding Audio Streams to PeerConnection
Use the following code to add an "audio source" to a peerconnection.

```C++
audio_track = factory->CreateAudioTrack("audio", audio_source);
peerconnection->AddTrack(audio_track);
```

This audio source is an abstract class, and users must implement its prescribed virtual functions.

Different from the video source, we generally use WebRTC's implementation.

```C++
factory->CreateAudioSource(cricket::AudioOptions()); // Returns a default audio source
```

It's worth noting that this is an empty implementation.
```C++
// pc/local_audio_source.(h | cpp)
class LocalAudioSource : public Notifier<AudioSourceInterface> {
 public:
  // Creates an instance of LocalAudioSource.
  static rtc::scoped_refptr<LocalAudioSource> Create(
      the cricket::AudioOptions* audio_options);

  SourceState state() const override { return kLive; }
  bool remote() const override { return false; }

  const cricket::AudioOptions options() const override { return options_; }

  void AddSink(AudioTrackSinkInterface* sink) override {}
  void RemoveSink(AudioTrackSinkInterface* sink) override {}

 protected:
  LocalAudioSource() {}
  ~LocalAudioSource() override {}

 private:
  void Initialize(const cricket::AudioOptions* audio_options);

  cricket::AudioOptions options_;
};

rtc::scoped_refptr<LocalAudioSource> LocalAudioSource::Create(
    const cricket::AudioOptions* audio_options) {
  rtc::scoped_refptr<LocalAudioSource> source(
      new rtc::RefCountedObject<LocalAudioSource>());
  source->Initialize(audio_options);
  return source;
}

void LocalAudioSource::Initialize(const cricket::AudioOptions* audio_options) {
  if (!audio_options)
    return;

  options_ = *audio_options;
}
```

WebRTC's logic for capturing video and audio data is inconsistent; video data is generated through the added "video source". In contrast, audio data is produced through an object called "audio device".

This object is specified as the third parameter in `CreatePeerConnectionFactory()`. If it is nullptr, WebRTC specifies it itself:
```C++
// media/engine/webrtc_voice_engine.cc:298
#if defined(WEBRTC_INCLUDE_INTERNAL_AUDIO_DEVICE)
  // No ADM supplied? Create a default one.
  if (!adm_) {
    adm_ = webrtc::AudioDeviceModule::Create(
        webrtc::AudioDeviceModule::kPlatformDefaultAudio, task_queue_factory_);
  }
#endif  // WEBRTC_INCLUDE_INTERNAL_AUDIO_DEVICE
  RTC_CHECK(adm());
  webrtc::adm_helpers::Init(adm());
```
If we don't provide an ADM, we need to enable the **WEBRTC_INCLUDE_INTERNAL_AUDIO_DEVICE** macro, or `RTC_CHECK(adm())` will assert.

When compiling WebRTC, you can see that this parameter is enabled by default in the gn argument list:
```shell
rtc_include_internal_audio_device
    Current value (from the default) = true
      From //webrtc.gni:243
```

In the module/audio_devices/ directory, you can see the implementation of the "audio_device" class on various platforms.

On Ubuntu 20.04, its implementation is based on "PulseAudio".

After they are initialized, audio data is automatically generated.

In this chapter, we do not introduce how to custom implement this "audio_device".

Readers may wonder why this "local_audio_source" is an empty implementation and why it is necessary. I will explain this in the subsequent source code analysis chapters.

For now, readers only need to know that they need to create such a source.

### Changes in SDP
```shell
o=- 2941351251118757007 2 IN IP4 127.0.0.1
s=-
t=0 0
a=group:BUNDLE 0
a=msid-semantic: WMS stream1
m=audio 9 UDP/TLS/RTP/SAVPF 111 103 104 9 102 0 8 106 105 13 110 112 113 126
c=IN IP4 0.0.0.0
a=rtcp:9 IN IP4 0.0.0.0
a=ice-ufrag:+ykV
a=ice-pwd:OJ6ZBZBNRdI++8rpZOmDz8cW
a=ice-options:trickle
a=fingerprint:sha-256 87:57:9

E:14:EB:75:BE:48:05:A9:70:2D:BF:65:C5:81:29:F2:6E:00:44:7C:12:CE:A7:42:13:54:F7:EB:01:8D
a=setup:actpass
a=mid:0
a=extmap:1 urn:ietf:params:rtp-hdrext:ssrc-audio-level
a=extmap:2 http://www.webrtc.org/experiments/rtp-hdrext/abs-send-time
a=extmap:3 http://www.ietf.org/id/draft-holmer-rmcat-transport-wide-cc-extensions-01
a=extmap:4 urn:ietf:params:rtp-hdrext:sdes:mid
a=extmap:5 urn:ietf:params:rtp-hdrext:sdes:rtp-stream-id
a=extmap:6 urn:ietf:params:rtp-hdrext:sdes:repaired-rtp-stream-id
a=sendrecv
a=msid:stream1 audio
a=rtcp-mux
a=rtpmap:111 opus/48000/2
a=rtcp-fb:111 transport-cc
a=fmtp:111 minptime=10;useinbandfec=1
a=rtpmap:103 ISAC/16000
a=rtpmap:104 ISAC/32000
a=rtpmap:9 G722/8000
a=rtpmap:102 ILBC/8000
a=rtpmap:0 PCMU/8000
a=rtpmap:8 PCMA/8000
a=rtpmap:106 CN/32000
a=rtpmap:105 CN/16000
a=rtpmap:13 CN/8000
a=rtpmap:110 telephone-event/48000
a=rtpmap:112 telephone-event/32000
a=rtpmap:113 telephone-event/16000
a=rtpmap:126 telephone-event/8000
a=ssrc:4067769755 cname:NqeyNmZKjtdwqI6r
a=ssrc:4067769755 msid:stream1 audio
a=ssrc:4067769755 mslabel:stream1
a=ssrc:4067769755 label:audio
```

Through "a=rtpmap:111 opus/48000/2", we see that WebRTC's default audio encoding format is Opus.

### Sending Audio Data

WebRTC controls the encoding and sending itself.

### Receiving Audio Data

For the default implementation, audio data is captured and played by WebRTC.

However, if we need to process this data on the receiver side, we can add a callback to the "audio track" on the receiver side:

```C++
void OnAddTrack(rtc::scoped_refptr<webrtc::RtpReceiverInterface> receiver,
            the std::vector<rtc::scoped_refptr<webrtc::MediaStreamInterface>>& streams) override 
{
    auto track = receiver->track().get();
    if(track->kind() == "audio") {
        auto audio_track = static_cast<webrtc::AudioTrackInterface*>(track);
        audio_track->AddSink(audio_receiver_.get());
    }
}

// The implementation of audio_receiver_ is as follows:
class AudioReceiver : public webrtc::AudioTrackSinkInterface {
public:
	  virtual void OnData(const void* audio_data,
	                      int bits_per_sample,
	                      int sample_rate,
	                      size_t number_of_channels,
	                      size_t number_of_frames)
	  {
		  // We can get audio data here;
	  }
};
```

### End

After reading this chapter, readers might still have several questions:
* Where does the default audio_device get its data?
* How to add a custom "audio_device" to send the audio data we want?
* How to prevent WebRTC on the receiving end from automatically playing the received audio data and control when to play these data ourselves.

I will introduce these points in subsequent chapters.

Source Code: https://github.com/MemeTao/webrtc-native-samples/tree/master/src/audio-channle