# Simple Media Annotation for Real-Time comunication
draft-chen-mmusic-smart-00.txt
  
## Abstract

While AI technology arising, automatically video annotation, speech transcribing
and similar such techs were becoming mature. This document proposes one solution 
to transfer and sync such meta data along the media in the context of RTCWeb.
 
## Introduction

  Nowdays, face tracking and even recognition have already make great progress
and were widely adopted. It can be used to smart cropping of the video avoiding 
the face being cut, It can also be used to identify who were engaged in the 
communication by labelling the name in the video. Speech recognition technology 
become mature too, it was not surprised that speech can be automatically 
transcribed to text and assist the real time communications. This can be applied 
to other object tracking, emotion detecting, simultaneous translation and so on.

````
                        +-----------------+
            +---------> | Media Processor +------------+
            |           +-----------------+            |
            |                                          |
            |                                          |
            |                                          |
            |                                          |
        +----+----+                               +-----v----+
        | Sender  |                               | Receiver |
        +---------+                               +----------+
                            Figure 1
````
                          
  In a real-time communication scenario like in Figure 1, these intellegent 
processing can be done in sender, media processor or receiver side. 
The advantage of doing it in receiver side or media processor was backward 
compatibility but there is also disadvantages: 

  1. the media could have been degraded due to network transfer,
  2. either all receivers needs to repeat the computation or extra server 
     resources are required to compute it,
  3. some meta data could be captured only on source side for privacy, 
     for example, the depth information from iPhone 3-D camera,
  
This document will focus on methods to capture the media meta data in sender 
side and define how to transfer them along with media data.

Compositing the annotation text or texture to the source picture was obviously 
way to convey them to receivers, but there were some drawbacks either:
  
  1. If the video was degraded due to network, the text will be unclear,
  2. The receiver can not turn on/off the annotation on-demand,
  3. The receiver can not customize the utilization of such meta data,
  
This document chooses a new out-of-band payload to transfer them and sync them 
through timestamp in the receiver, rather than extanding the current media 
payloads:

  1. It was easier to achieve backward compatiblity,
  2. The transportation of the meta data could be different from the media.
     For example, the face tracking information could be combined with the audio 
     transcribed text and render together.

The mechanism described in this document can also be used to transfer other 
media meta data. For example, for dekstop sharing case, the mouse moving event 
can be transfered separately through this format, and we can achieve higher 
frame rate of mouse-move than the screen content.

## Message Format

          +---------------+---------------+
          |0|1|2|3|4|5|6|7|0|1|2|3|4|5|6|7|
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          | V |   RES | P |      Count    |
          +---+-------+---+---------------+

The message will have 2 bytes header followed with an array of sub-messages. 

    V: the version of this message, it should be set to 0,
    RES: the reserved field, should be set to 0 now,
    P: priority of the message, lower priority packets can be dropped or delayed,
    Count: the number of sub-messages embedded minor 1.

All sub-message will share the common header.

### Sub-message common header

      +---------------+---------------+---------------+---------------+
      |0|1|2|3|4|5|6|7|0|1|2|3|4|5|6|7|0|1|2|3|4|5|6|7|0|1|2|3|4|5|6|7|
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |     Type      |     Len       |              SSRC             |
      +---------------+---------------+---------------+---------------+
      |0|1|2|3|4|5|6|7|0|1|2|3|4|5|6|7|0|1|2|3|4|5|6|7|0|1|2|3|4|5|6|7|
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |            ...SSRC            |          Timestamp            |
      +---------------+---------------+---------------+---------------+
      |       ...Timestamp            |
      +---------------+---------------+

Each sub-message will have at least 10 bytes headers, there are:

    Type: one-byte pre-defined sub-message type, see table 1.
    Len: the length of the sub-message, not including the common header, in bytes
    SSRC: 4 bytes, the SSRC of the corresponding source
    Timestamp: 4 bytes, the timestamp of the corresponding source

This document will define 3 sub-message types and they can be extended in future.

            +------------------+----------+
            |    Sub-message   |   Value  |  
            +------------------+----------+
            |  SMART_RECTS     |     0    |
            +------------------+----------+
            |  SMART_TEXTS     |     1    |
            +------------------+----------+

### SMART_RECTS

      +---------------+---------------+---------------+---------------+
      |0|1|2|3|4|5|6|7|0|1|2|3|4|5|6|7|0|1|2|3|4|5|6|7|0|1|2|3|4|5|6|7|
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |S|I| Name Len  |               ID              |       Left    |
      +---------------+---------------+---------------+---------------+
      |    ...Lef     |             Top               |      Width    |
      +---------------+---------------+---------------+---------------+
      |    ...Width   |            Height             |     Name      |
      +---------------+---------------+---------------+---------------+
      
This pattern can be repeated, the total number of rectangles could be determined
from the length field of the sub-message common header. Each filed of this 
sub-message is defined as:

      S: controls the Width/Height fields. If it is 0, there is no Width/Height.
      I: controls the ID field. If it is 0, there is no ID filed.
      Name Len: the length of the name field, it can between 0 - 63.
      ID: the identifier of the rectangle, value is defined by the app.
      Left/Top: 2 bytes each, the left top point of the rectangle.
      Width/Height: 2 bytes each, the width and height of the rectangle.
      Name: the UTF-8 string of the name tagging that rectangle, up to 63 bytes.

### SMART_TEXTS

      +---------------+---------------+---------------+---------------+
      |0|1|2|3|4|5|6|7|0|1|2|3|4|5|6|7|0|1|2|3|4|5|6|7|0|1|2|3|4|5|6|7|
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |T|   Text Len  |          Time Offset          |     Text      |
      +---------------+---------------+---------------+---------------+

Similar to the SMART_RECTS, each filed of this sub-message is defined as:

    T: control if Time Offset field was set or not,
    Text Len: the length of the Name field minor 1, the name length can be 1-128,
    Time Offset: the millisecond time offset corresponding to the timestamp,
    Text: A string in UTF-8 encoding,

## SDP Negotiation

The message can be sent along with RTP stream as RTP payload, it can also be 
sent through Data Channel in SCTP.

When it was sent through RTP stream, it can be multiplexed with either Audio or 
Video stream with its own sequence space and separate SSRC, the payload type 
can be negotiated with dynamic payloads.

### Media Type Registration

Media type registration is done according to [RFC6838] and [RFC4855].

   Type name: application
   Subtype name: annotation

   Required parameters:
    
      type: supported sub-message type values, use vertical bar to concatenate, 
      for example, type=0|1, means it will generate both SMART_RECTS and 
      SMART_TEXTS. The type indicates how many payloads it could be encoded in 
      the source side.
   
   Optional parameters:
      
      minptime: the minimum interval of such meta data
      maxbitrate: the maximum bitrate of such meta data, in kbps units
      
### Example

```   
    m=audio 5004 UDP/TLS/RTP/SAVPF 111 104
    c=IN IP4 47.98.240.10
    a=rtcp:9 IN IP4 0.0.0.0
    a=mid:audio
    a=sendrecv
    a=rtcp-mux
    a=rtpmap:111 opus/48000/2
    a=fmtp:111 minptime=10;sprop-stereo=0;stereo=0;usedtx=0;useinbandfec=1
    a=rtpmap:104 annotation/1000
    a=fmtp:104 type=0|1;maxbitrate=2
```

In this example, 104 was selected for the media annotation and it will send 
along with audio stream.

### Send Through Data Channel

## RTCWeb implicition

To achieve the synced video annotation, RTCWeb needs to open interface for 
application operating the rendering of the annotation with the video. Exposure 
of the video raw data to JS layer and let application render it with Canvas or 
WebGL would be a good choice.

## Compatible

When the legacy endpoints received such an OFFER, it can just reject the payload.

## Security Considerations

Such meta data could be privacy as the same level of the media, sender can 
decide to not distribute them. The meta data should be encrypted in the transer 
and it should also support end to end encription.

## IANA Considerations

"application/annotation" media type should be registered as defined in [RFC6838].

## Acknowledgements


https://www.iana.org/assignments/media-types/media-types.xhtml
http://asciiflow.com/
