# KMS WebRTC Endpoint Events

The WebRtcEndpoint class manages a WebRTC connection between KMS and a remote peer. This includes the ICE Gathering process, used to find local and remote addresses which provide a working connection between both peers.

Table of contents:

<!-- TOC -->

- [KMS WebRTC Endpoint Events](#kms-webrtc-endpoint-events)
  - [Basic usage](#basic-usage)
  - [Events](#events)
    - [MediaElement events](#mediaelement-events)
      - [ElementConnected](#elementconnected)
      - [ElementDisconnected](#elementdisconnected)
      - [MediaFlowInStateChange](#mediaflowinstatechange)
      - [MediaFlowOutStateChange](#mediaflowoutstatechange)
    - [BaseRtpEndpoint events](#basertpendpoint-events)
      - [ConnectionStateChanged](#connectionstatechanged)
      - [MediaStateChanged](#mediastatechanged)
    - [WebRtcEndpoint events](#webrtcendpoint-events)
      - [IceComponentStateChange](#icecomponentstatechange)
      - [IceCandidateFound](#icecandidatefound)
      - [NewCandidatePairSelected](#newcandidatepairselected)
      - [IceGatheringDone](#icegatheringdone)
      - [DataChannelClose](#datachannelclose)
      - [DataChannelOpen](#datachannelopen)
  - [Sample sequence of events](#sample-sequence-of-events)

<!-- /TOC -->


## Basic usage

Once an instance of the WebRtcEndpoint is created inside a Media Pipeline, an event handler should be added for each one of the events that can be emitted by the endpoint. Later, the endpoint should be instructed to either generate a SDP Offer -when KMS is the caller- or process an offer generated by the remote peer -when KMS is the callee-. The later option would in turn generate an SDP Answer. As a last step, the endpoint should start the ICE gathering process.

This example code shows the typical usage for the WebRtcEndpoint:

```
KurentoClient kurento;
MediaPipeline pipeline = kurento.createMediaPipeline();
WebRtcEndpoint webRtcEndpoint = new WebRtcEndpoint.Builder(pipeline).build();
webRtcEndpoint.addIceComponentStateChangedListener(<...>);
webRtcEndpoint.addIceCandidateFoundListener(<...>);
webRtcEndpoint.addNewCandidatePairSelectedListener(<...>);
webRtcEndpoint.addIceGatheringDoneListener(<...>);

// Receive an SDP Offer, via the application's signaling mechanism
String sdpAnswer = webRtcEndpoint.processOffer(sdpOffer);
// Send the SDP Answer, via the application's signaling mechanism

webRtcEndpoint.gatherCandidates();
```


## Events

This is a list of all events that can be emitted by an instance of WebRtcEndpoint. This class belongs to a chain of inherited classes, so the list includes events from all of them, starting from the topmost class in the inheritance tree:
- MediaObject:
    - Error
- MediaElement:
    - ElementConnected
    - ElementDisconnected
    - MediaFlowInStateChange
    - MediaFlowOutStateChange
- SessionEndpoint:
    - MediaSessionStarted
    - MediaSessionTerminated
- BaseRtpEndpoint:
    - ConnectionStateChanged
    - MediaStateChanged
- WebRtcEndpoint:
    - IceComponentStateChange
    - IceCandidateFound
    - NewCandidatePairSelected
    - IceGatheringDone
    - DataChannelOpen
    - DataChannelClose


### MediaElement events

These events indicate some low level information about the state of [GStreamer](https://gstreamer.freedesktop.org), the underlying multimedia framework.

Note that the `MediaFlowInStateChange` and `MediaFlowOutStateChange` events are not 100% reliable to check if a RTP connection is flowing: RTCP packets do not usually flow at a constant rate. For example, minimizing a browser window with an `RTCPeerConnection` might affect this interval.


#### ElementConnected

[TODO]


#### ElementDisconnected

[TODO]


#### MediaFlowInStateChange

- State = _Flowing_: There are GStreamer's buffers flowing _into_ the Endpoint's `sink` pad, in the GStreamer pipeline. For example, with a Recorder element this event will fire when media starts arriving from the pipeline.
- State = _NotFlowing_: The element is not receiving any input data from the GStreamer pipeline.


#### MediaFlowOutStateChange

- State = _Flowing_: There are GStreamer's buffers flowing _from_ the Endpoint's `src` pad, in the GStreamer pipeline. For example, with a Player element this event will fire when media starts being pushed to the pipeline.
- State = _NotFlowing_: The element is not generating any output data to the GStreamer pipeline.


### BaseRtpEndpoint events

These events provide information about the state of the RTP connection for each stream in the WebRtc call.


#### ConnectionStateChanged

- State = _Connected_: All of the `KmsIRtpConnection` objects have been created [TODO: explain what this means].
- State = _Disconnected_: At least one of the `KmsIRtpConnection` objects is not created yet.

Call sequence:

```
signal KmsIRtpConnection::"connected"
  -> signal KmsSdpSession::"connection-state-changed"
    -> signal KmsBaseRtpEndpoint::"connection-state-changed"
      -> BaseRtpEndpointImpl::updateConnectionState
```


#### MediaStateChanged

- State = _Connected_: At least _one_ of the audio or video RTP streams in the session is still alive (receiving RTCP packets).
- State = _Disconnected_: None of the RTP streams belonging to the session is alive (ie. no RTCP packets are received for any of them).

Call sequence:

These signals from GstRtpBin will trigger the `MediaStateChanged` event:
- `GstRtpBin::"on-bye-ssrc"`: State = _Disconnected_.
- `GstRtpBin::"on-bye-timeout"`: State = _Disconnected_.
- `GstRtpBin::"on-timeout"`: State = _Disconnected_.
- `GstRtpBin::"on-ssrc-active"`: State = _Connected_.

```
<Signals from GstRtpBin>
  -> signal KmsBaseRtpEndpoint::"media-state-changed"
    -> BaseRtpEndpointImpl::updateMediaState
```

**Note**: `MediaStateChanged` (State = _Connected_) will happen after these other events have been emitted:
1. `NewCandidatePairSelected`.
2. `IceComponentStateChanged` (State: _Connected_).
3. `MediaFlowOutStateChange` (State: _Flowing_).


### WebRtcEndpoint events

These events provide information about the state of [libnice](https://nice.freedesktop.org), the underlying library in charge of the ICE Gathering process. The ICE Gathering is typically done before attempting any WebRTC call.

For further reference, see the libnice's Agent [documentation](https://nice.freedesktop.org/libnice/NiceAgent.html) and [source code](https://cgit.freedesktop.org/libnice/libnice/tree/agent/agent.h).


#### IceComponentStateChange

This event carries the state values from the signal [NiceAgent::"component-state-changed"](https://nice.freedesktop.org/libnice/NiceAgent.html#NiceAgent-component-state-changed).

- State = _Disconnected_: There is no active connection, and the ICE process is stopped. NiceAgent state: `NICE_COMPONENT_STATE_DISCONNECTED`, "No activity scheduled".
- State = _Gathering_: The Endpoint has started finding all possible local candidates, which will be notified through the event `IceCandidateFound`. NiceAgent state: `NICE_COMPONENT_STATE_GATHERING`, "Gathering local candidates".
- State = _Connecting_: The Endpoint has started the connectivity checks between at least one pair of local and remote candidates. NiceAgent state: `NICE_COMPONENT_STATE_CONNECTING`, "Establishing connectivity".
- State = _Connected_: At least one candidate pair resulted in a successful connection. This happens right after the event `NewCandidatePairSelected`. NiceAgent state: `NICE_COMPONENT_STATE_CONNECTED`, "At least one working candidate pair".
- State = _Ready_: All local candidates have been gathered, all pairs of local and remote candidates have been tested for connectivity, and a successful connection was established. NiceAgent state: `NICE_COMPONENT_STATE_READY`, "ICE concluded, candidate pair selection is now final".
- State = _Failed_: All local candidates have been gathered, all pairs of local and remote candidates have been tested for connectivity, but still none of the connection checks was successful, so no connectivity was reached to the remote peer. NiceAgent state: `NICE_COMPONENT_STATE_FAILED`, "Connectivity checks have been completed, but connectivity was not established".

This graph shows the possible state changes:

![Source: libnice/docs/reference/libnice/states.png](kms_webrtc_endpoint_events-libnice_states.png)

**Note**: The states _Ready_ and _Failed_ indicate that the ICE transport has completed gathering and is currently idle. However, since events such as adding a new interface or a new TURN server will cause the state to go back, _Ready_ and _Failed_ are **not** terminal states.


#### IceCandidateFound

A new local candidate has been found, after the ICE Gathering process was started. Equivalent to the signal [NiceAgent::"new-candidate-full"](https://nice.freedesktop.org/libnice/NiceAgent.html#NiceAgent-new-candidate-full).


#### NewCandidatePairSelected

During the connectivity checks one of the pairs happened to provide a successful connection, and the pair had a higher preference than the previously selected one (or there was no previously selected pair yet). Equivalent to the signal [NiceAgent::"new-selected-pair"](https://nice.freedesktop.org/libnice/NiceAgent.html#NiceAgent-new-selected-pair-full).


#### IceGatheringDone

All local candidates have been found, all remote candidates have been received from the remote peer, and all pairs of local-remote candidates have been tested for connectivity. When this happens, all activity of the ICE agent stops. Equivalent to the signal [NiceAgent::"candidate-gathering-done"](https://nice.freedesktop.org/libnice/NiceAgent.html#NiceAgent-candidate-gathering-done).


#### DataChannelClose

[TODO]


#### DataChannelOpen

[TODO]


## Sample sequence of events

When the WebRtcEndpoint instance has been created, and all event handlers have been aded, starting the ICE process will generate a sequence of events very similar to this one:

1. Event(s): **IceCandidateFound**. Typically, candidates of type `host` -corresponding to the local network- are almost immediately found after starting the ICE gathering, and this event can arrive even before the event `IceComponentStateChanged` is triggered.

2. Event: **`IceComponentStateChanged`** (State: _Gathering_). At this point, the local peer is gathering more candidates, and it is also waiting for the candidates gathered by the remote peer, which could start arriving at any time.

3. Function call: **`AddIceCandidate`**. The remote peer found some initial candidates, and started sending them. Typically, the first candidate received is of type `host`, because those are found the fastest.

4. Event: **`IceComponentStateChanged`** (State: _Connecting_). After receiving the very first of the remote candidates, the ICE agent starts with the connectivity checks.

5. Function call(s): **`AddIceCandidate`**. The remote peer will continue sending its own gathered candidates, of any type: `host`, `srflx` (STUN), `relay` (TURN).

6. Event: **`IceCandidateFound`**. The local peer will also continue finding more of the available local candidates.

7. **`NewCandidatePairSelected`**. The ICE agent makes local and remote candidate pairs. If one of those pairs pass the connectivity checks, it is selected for the WebRTC connection.

8. **`IceComponentStateChanged`** (State: _Connected_). After selecting a candidate pair, the connection is stablished. _At this point, the media stream(s) can start flowing_.

9. **`NewCandidatePairSelected`**. Typically, better candidate pairs will be found over time. The old pair will be abandoned in favour of the new one.

10. **`IceGatheringDone`**. When all candidate pairs have been tested, no more work is left to do for the ICE agent. The gathering process is finished.

11. **`IceComponentStateChanged`** (State: _Ready_). As a consequence of finishing the ICE gathering, the component state gets updated.
