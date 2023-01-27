# Summary
- Add notion of service protocol in libobs and OBS Studio
- Outputs for streaming are registered with one or more compatible protocols and codecs
- Only codecs compatible with the output and the service are shown
- Audio codec can be chosen if various compatible, the encoder implementation can be chosen in advanced output mode.
- Will be extended with RFC 39

# Motivation
Create a better management of outputs, encoders, and services with an approach focused on protocols.

# Design

## Outputs API
Only outputs with the `OBS_OUTPUT_SERVICE` flag are concerned.

Adding to `obs_output_info`:
- `const char *protocols`: protocols separated with a semi-colon (ex: `"RTMP;RTMPS"`), required to register the output.
- `const char *protocols_prefixes`: URL protocols prefixes separated with a semi-colon `"rtmp://;rtmps://"`, optional.
  - Only one prefix per protocol in `protocols`.
  - Those prefixes should be in the same order as `protocols`.
  - Meant to be used for protocol auto-detection.
  - While registering if present, the number of prefix shall be less or equal to the number of protocols.
  - NOTE: Prefixes (URI scheme + `://`) are prefered over URI schemes to avoid:
    - matching links without schemes (e.g. "rtmp.example.com").
    - adding `://` each time that a auto-detection is done.



Adding to the API these functions:
- `const char *obs_output_get_protocols(const obs_output_t *output)`: returns protocols supported by the output.
- `bool obs_output_is_protocol_registered(const char *protocol)`: return true if an output with the protocol is registered.
- `const char *obs_output_get_prefix_protocol(const char *prefix)`: return the protocol bound to the prefix.
- `bool obs_enum_output_service_types(const char *protocol, size_t idx, const char **id)`: enumerate all outputs types compatible with the given protocol.
- `bool obs_enum_output_protocols_prefixes(size_t idx, const char **prefix)`: enumerate all registered URL prefixes.
- `bool obs_enum_output_protocols(size_t idx, const char **prefix)`: enumerate all registered protocol.
- `const char *obs_get_output_supported_video_codecs(const char *id)`: return compatible video codecs of the given output id.
- `const char *obs_get_output_supported_audio_codecs(const char *id)`: return compatible audio codecs of the given output id.

### About protocols
RTMP and RTMPS will only be compatible only with H264 and AAC.

HLS will have no prefix set and not "auto-detectable", HLS relies on HTTP/HTTPS and it is not the only one in this case.

HLS, SRT, and RIST are codec agnostic since they use MPEG/TS container but require some set up depending on the codec.

FTL will only be compatible only with H264 and Opus because of the deprecation of the protocol in OBS Studio.

## Services API
Since a streaming service may not accept all the codecs usable by a protocol, adding a way to set supported codecs is required.

Adding to `obs_service_info`:
- `const char *(*get_protocol)(void *data)`: return the protocol used by the service. RFC 39 will allow multi-protocol services in the future.
- `const char **(*get_supported_video_codecs)(void *data)`: video codecs supported by the service. Optional, fallback to protocol supported codecs if not set.
- `const char **(*get_supported_audio_codecs)(void *data)`: audio codecs supported by the service. Optional, fallback to protocol supported codecs if not set.

Adding to the API these functions:
- `const char *obs_service_get_protocol(const obs_service_t *service)`: return the protocol used by the service.
- `obs_service_get_supported_video_codecs(const obs_service_t *service)`: return video codecs compatible with the service.
- `obs_service_get_supported_audio_codecs(const obs_service_t *service)`: return audio codecs compatible with the service.

### Services and information

Depending on the protocol, the service should provide various types of connection details (e.g. server URL, stream key, username, password).

Currently, `get_key` can provide a stream id (SRT) or an encryption passphrase (RIST) rather than a stream key.

Rather than adding a getter for each possible type of information, a getter where we can choose which type of information we want should be implemented.

Each types of connection details will be define by a macro and an even number, odd number will be reserved for potential third-party protocol (RFC 39).

List of types:
```c
#define OBS_SERVICE_INFO_SERVER_URL 0
#define OBS_SERVICE_INFO_STREAM_KEY 2
#define OBS_SERVICE_INFO_USERNAME 4 
#define OBS_SERVICE_INFO_PASSWORD 6
#define OBS_SERVICE_INFO_STREAM_ID 8
#define OBS_SERVICE_INFO_ENCRYPT_PASSPHRASE 10
```

Adding to `obs_service_info`:
- `const char *(*get_connect_info)(uint32_t type, void *data)` where `type` is an integer indicating which info we want and return `NULL` if it doesn't have it.
- `bool (*can_try_to_connect)(void *data)`, since protocols and services do not always require the same informations. This function allows the service to return if it can connect. Return true if the the service has all it needs to connect.

Adding to the API these functions:
- `const char *obs_service_get_connect_info(uint32_t type, const obs_service_t *service)`: return the connection information related to the given type.
- `bool obs_service_can_try_to_connect(const obs_service_t *service)`: return if the service can try to connect.
  - return `bool (*can_try_to_connect)(void *data)` result.
  - return true if `bool (*can_try_to_connect)(void *data)` is not implemented in the service.
  - return false if the service object is invalid.

And `obs_service_get_url()`, `obs_service_get_key()`, `obs_service_get_username()` and `obs_service_get_password()` will be deprecated in favor of `obs_service_get_connect_info()`.

In the future, if we want to support a protocol that doesn't use a server URL, this scenario is covered by this design.

### About `rtmp-services`

`rtmp-services` is plugin that provides two services `rtmp_common` and `rtmp_custom`. The use of "rtmp" in the naming no longer means that it only supports RTMP, it was and is kept to avoid breakage.

This plugin will:
- for `rtmp_common` services will to at least support H264 as video codec, AAC or Opus as audio codec. This is a limitation required by the simple output mode.
- for `rtmp_custom` it will depends on the auto-detected protocol.

#### About service JSON file

- A field `"protocol"` will be added mostly for HLS/FTL services, other protocol can be detected through prefixes, further changes will be done in RFC 39.
  - Services still mono-protocol, RTMP+RTMPS combo when the field is not set is the only exception.
  - RTMP services with custom code and HTTP(S) URLs will have the protocol enforced to RTMP.
- Services that ask for a protocol that is not registered will not be shown.
- Codecs field for audio and video will be added to allow services to limit which codec is compatible with the service.
- `"output"` field in the `"recommended"` object will be deprecated, but it will be kept for backward compatibility. `const char *(*get_output_type)(void *data)` in `obs_service_info` will be no longer used by `rtmp-services`.
    - It will be required through the JSON Schema if protocol different from RTMP(S) is set to avoid breakage of the backward compatibility.

## UI

### Audio encoders
If the service/output is compatible with various audio encoders, the user should be able to choose in the UI which one.

For AAC in simple mode, the will have only one AAC option.
In advanced mode the user could choose between various AAC implementation.


### Settings loading order

Service settings need to be loaded first and output ones afterward to show only compatible encoder.

### Service and Output settings interactions

NOTE: Since Services API is not usable by third-party for now, this RFC will not consider a scenario where there is no simple encoder preset for the service.

If changing the service results in the selected protocol changing, the Output page should be updated to list only compatible encoders.

If the currently selected encoder (codec) is not supported by the service, the encoder will be changed and the user will be notified about the change.

## What if there are various registered outputs for one protocol ?

The improbable situation where a plugin register a output for an already registered protocol could happen, so managing this possibility is required.

- In `obs_service_info`:
- `const char *(*get_output_type)(void *data)` will be renamed `const char *(*get_preferred_output_type)(void *data)` when an API break is possible.
- `const char *obs_service_get_output_type(const obs_service_t *service)` will be deprecated and replaced by `const char *obs_service_get_preferred_output_type(const obs_service_t *service)`.

A function in the UI to prefer first-party outputs will be considered.
An option in advanced output to allow to choose an output type will not be considered for now.