# Summary
- Add notion of service protocol in libobs and OBS Studio
- Outputs for streaming are registered with one or more compatible protocols and codecs
- Only codecs compatible with the output and the service are shown
- Audio encoder can be chosen if various compatible
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

Asside from getters for protocol related attributes, adding to the API these functions:
- `bool obs_output_is_protocol_registered(const char *protocol)`: return true if an output with the protocol is registered.
- `const char *obs_output_get_prefix_protocol(const char *prefix)`: return the protocol bound to the prefix.
- `bool obs_enum_output_service_types(const char *protocol, size_t idx, const char **id)`: enumerate all outputs types compatible with the given protocol.
- `bool obs_enum_output_protocols_prefixes(size_t idx, const char **prefix)`: enumerate all registered URL prefixes.
- `bool obs_enum_output_protocols(size_t idx, const char **prefix)`: enumerate all registered protocol.

### About protocols
RTMP and RTMPS will only be compatible only with H264 and AAC.

HLS will have no prefix set and not "auto-detectable".

HLS, SRT, and RIST are codec agnostic since they use MPEG/TS container but require some set up depending on the codec.

FTL will only be compatible only with H264 and Opus because of the deprecation of the protocol in OBS Studio.

## Services API
Since a streaming service may not accept all the codecs usable by a protocol, adding a way to set supported codecs is required.

Adding to `obs_service_info`:
- `const char *(*get_protocols)(void *data)`: return the protocol used by the service. It will allow multi-protocol services in the future.
- `const char **(*get_supported_video_codecs)(void *data)`: video codecs supported by the service. Optional, fallback to protocol supported codecs if not set.
- `const char **(*get_supported_audio_codecs)(void *data)`: audio codecs supported by the service. Optional, fallback to protocol supported codecs if not set.

### Services and information

Depending on the protocol, the service should provide various types of connection details (e.g. server URL, stream key, username, password).

Currently, `get_key` can provide a stream id (SRT) or an encryption passphrase (RIST) rather than a stream key.

Rather than adding a getter for each possible type of information, a getter where we can choose which type of information we want should be implemented.

Adding to Services API:
- `const char *(*get_connect_info)(uint32_t type, void *data)` where `type` is an integer indicating which info we want and return `NULL` if it doesn't have it.
- `bool (*can_try_to_connect)(void *data)`, since protocols and services do not always require the same informations. This function allows the service to return if it can connect. Returns true if not implemented.
- Each types of connection details will be define by a macro and an even number, odd number will be reserved for future third-party protocol.
- `obs_service_get_url`, `obs_service_get_key`, `obs_service_get_username` and `obs_service_get_password` will be deprecated in favor of `obs_service_get_info`.

List of types:
```c
#define OBS_SERVICE_INFO_SERVER_URL 0
#define OBS_SERVICE_INFO_STREAM_KEY 2
#define OBS_SERVICE_INFO_USERNAME 4 
#define OBS_SERVICE_INFO_PASSWORD 6
#define OBS_SERVICE_INFO_STREAM_ID 8
#define OBS_SERVICE_INFO_ENCRYPT_PASSPHRASE 10
```

In the future, if we want to support a protocol that doesn't use a server URL, this scenario is covered by this design.

### About `rtmp-services`

This plugin will:
- for `rtmp_common` only accept H264 or HEVC depending on the protocol, AAC and opus for FTL protocol.
- for `rtmp_custom` it will depends on the auto-detected protocol.

## UI

### Audio encoders
If the service/output is compatible with various audio encoders, the user should be able to choose in the UI which one.

For AAC in simple mode, the will have only one AAC option.
In advanced mode the user could choose between various AAC implementation.


### Settings loading order

Service settings need to be loaded first and output ones afterward to show only compatible encoder.

### About service JSON file

- A field `"protocol"` will be added for HLS/FTL services, other protocol can be detected through prefixes, further changes will be done in RFC 39.
  - Services still mono-protocol, RTMP+RTMPS combo when the field is not set is the only exception.
  - RTMP services with custom code and HTTP(S) URLs will have the protocol enforced to RTMP.
- Services that ask for a protocol or only codecs that are not registered will not be shown.
- `"output"` in `"recommended"` is no longer required (kept for backward compatibility) and `const char *(*get_output_type)(void *data)` in `obs_service_info` will be no longer used.
    - It will be required through the JSON Schema if protocol different from RTMP(S) is set to avoid breakage of the backward compatibility.
- Codecs field for audio and video will be added to enable services to allow some of codec compatible with the protocol.

### Service and Output settings interactions

NOTE: Since Services API is not usable by third-party for now, this RFC will not consider a scenario where there is no simple encoder preset for the service.

If changing the service results in the selected protocol changing, the Output page should be updated to list only compatible encoders.

If the currently selected encoder (codec) is not supported by the service, the encoder will be changed and the user will be notified about the change.

## What if there are various registered outputs for one protocol ?

The improbable situation where a plugin register a output for an already registered protocol could happen, so managing this possibility is required.

- In `obs_service_info`:
- A `const char *(*get_preferred_output_type)(void *data)` will replace `const char *(*get_output_type)(void *data)`.
- A function in the UI to prefer first-party outputs will be considered.
- An option in advanced output to allow to choose an output type will not be considered for now.