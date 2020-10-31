---
title: "Why you should force fixed & closed GOPs (and how to do it in ffmpeg)"
date: 2020-10-18T01:33:55+01:00
categories:
- ffmpeg
- x264
- transcoding
- video coding
---
Today I'm going cover why you would want to use fixed & closed GOPs in HLS streaming -- and how to do it using ffmpeg.

### Open and closed groups of pictures

If you came here you probably already know that in video coding a _group of pictures_ (GOP) is a term that defines how I, P and B frames are arranged -- or more specifically, the interval between the I-frames. I frames contain full image data, while P and B frames are so-called predictive frames and only have partial data. They make use of the fact that large parts of the videos (such as the background) remain effectively unchanged and only encode the changes between them -- i.e. motion and scene changes. P frames are only looking back, while B frames can also look ahead.

There are two types of GOPs: open and closed.

Open GOPs allow the predictive frames to reference frames beyond their current group of pictures.

Closed GOPs, on the other hand, constrain the references to the current group of pictures. They do so by introducing another type of I-frame, the Instantaneous Decoding Refresh (IDR) frame, which forbids predictive frames from referencing past their bounds.

Since frames in open GOPs can reference more frames, this results in better compression and better image quality -- compared to the same bitrate video using closed GOPs. Sounds great -- in theory.

### Open GOPs are problematic

Unfortunately, open GOPs also introduce problems with seeking and streaming. When seeking to a specific timestamp in a video with open GOPs we might arrive at a frame that references data that we haven't read yet -- ultimately stalling playback, dropping frames or introducing artifacts.

While for local playback having to fetch more frames might not be an issue at all, this suddenly gets worse in the streaming scenario. We might seek to a segment that contains the part of the video we're interested in, only to find out that we need another segment to decode the current one correctly. Network latency is going to be _much_ more noticeable than when simply reading from disk.

HTTP Live Streaming (HLS) makes these problems much more evident. The idea behind HLS is to quickly adapt the bitrate of the video to current network conditions (due to change over time) so that the client can watch it without hiccups. However, switching between segments of different bitrates is very much like seeking in the streaming scenario, only now it can occur regularly during normal playback. Let's break it down.

On every bitstream switch, we're requesting a segment from that new stream, just ahead of our buffered data. We then decode this segment. Since our video was encoded with open GOPs, we may need the previous segment's data[^1]. However, the P and B frames in different bitstreams are referring to the frames in their respective bitstreams, so we cannot reuse frames we have received -- and have to fetch yet another segment.

### The specification

Contrary to what you can find in many places on the Internet, the RFC 8216 (the document behind the HLS specification) doesn't mention anything about _requiring_ closed or fixed GOPs. It merely suggests the presence of an IDR frame in the decoded segment[^2].

> Any Media Segment that contains video SHOULD include enough information to initialize a video decoder and decode a continuous set of frames that includes the final frame in the Segment; network efficiency is optimized if there is enough information in the Segment to decode all frames in the Segment.  For example, any Media Segment containing H.264 video SHOULD contain an IDR; frames prior to the first IDR will be downloaded but possibly discarded.

It does mention that closed GOPs (i.e. _"enough information to decode all frames"_) improve performance -- which makes sense given the whole class of issues we're running into by using open GOPs.

Also, since decoding open GOPs is more involved, some players are bound to have buggy implementations that don't play nicely with open GOPs (thus limiting the audience that can watch such content).

Regardless, closed GOP videos have become the de-facto standard for HLS streaming. In fact, Apple even [mandates](https://developer.apple.com/documentation/http_live_streaming/hls_authoring_specification_for_apple_devices) that all streams to be consumed on Apple devices have closed GOPs.

### Fixed GOPs

Another practice that doesn't stem from the specification itself is fixing GOP size. RFC 8216 requires that matching content in different bitstreams have matching timestamps -- and that on its own is enough to decode videos successfully.

Here we're facing another Apple [recommendation](https://developer.apple.com/documentation/http_live_streaming/hls_authoring_specification_for_apple_devices):

> For maximum interoperability, all audio/video variants and renditions SHOULD have segment boundaries at the same points in time.

Reading further through that document we can find that if we want our streams to be playable on AirPlay 2-Enabled TVs, it becomes a requirement. Moreover, Twitch has also reported [playback issues](https://blog.twitch.tv/en/2017/10/10/live-video-transmuxing-transcoding-f-fmpeg-vs-twitch-transcoder-part-i-489c1c125f28/) on Chromecasts when playing HLS streams with misaligned segments, so it's likely to happen on other devices as well.

Thus, for maximum compatibility, our segments should be timestamp-aligned.

What we haven't touched on yet so far is the relation between segment alignment and fixed GOPs.

Segments are usually divided on keyframe boundaries, as it makes the most sense. Without enforcing fixed GOPs, keyframes are going to be naturally aligned on scene changes. At first, it seems there shouldn't be any issues with this approach. However, scene detection for different resolutions, bitrates and encoding profiles can happen at different timestamps (albeit usually fairly close to one another), causing segment misalignment.

This is why so many people recommend switching scene detection off. However, scene detection is very useful. It allows us to deliver a better quality video at the same bitrate.

If we can force the keyframes to be at specific intervals we can then have the segmenter split our video on these intervals (or their multiples) -- all while keeping scene detection enabled.

### Making it work with ffmpeg

Now that we know what we need to do and why, let's see how.

For H.264, we can use x264's `-g` and `-keyint_min` options. The first switch specifies the maximum keyframe interval, while the latter the minimum interval. If we set both to the same value we should achieve fixed GOPs. Except we won't. The actual [code](https://github.com/mirror/x264/blob/db0d417728460c647ed4a847222a535b00d3dbcb/encoder/encoder.c#L1057) within libx264 clips the minimum keyframe interval to half the maximum interval plus 1:

```
h->param.i_keyint_min = x264_clip3( h->param.i_keyint_min, 1, h->param.i_keyint_max/2+1 );
```

This will inevitably lead us to the issues that scene detection introduces mentioned above. We could stop here and simply disable scene detection (by setting `sc_threshold` x264 param to `0` as it's frequently done), effectively leaving keyframes at the maximum interval we specified (`-g` option). But, there's another way.

It's the `-force_key_frames` option. We can use it to specify raw timestamps where keyframes are to be inserted. Luckily, instead of providing these values manually, we can just use an expression which selects those frames for us. 

```
-force_key_frames "expr:if(isnan(prev_forced_n),1,eq(n,prev_forced_n+$GOP))"
```

What this expression does is:
1. if there are no previously forced keyframes it forces one (happens on the first keyframe); and
2. it forces a keyframe on every multiple of the GOP size thereafter (here supplied as shell variable).

This way, we can both have keyframes inserted at scene changes and guaranteed keyframes at set positions -- which coupled with setting segment size to a multiple of the GOP length will allow us to achieve aligned segments while reaping the benefits of scene change detection.

[^1]: This can be a future segment as well, but it's usually less of an issue as we'd be fetching it in time anyway.
[^2]: The initial version of the draft RFC 8216 is based upon didn't actually have this recommendation. It's only been included in the subsequent revisions [draft-pantos-http-live-streaming-01](https://tools.ietf.org/html/draft-pantos-http-live-streaming-01#section-4); for full reference of how different drafts map to different protocol versions and RFC 2816 please follow this [link](https://www.gpac-licensing.com/2014/12/01/apple-hls-comparing-versions/).
