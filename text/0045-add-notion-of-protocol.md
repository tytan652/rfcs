# Summary
- Define supported protocols for services and outputs in libobs and OBS Studio
- Outputs for streaming are registered with one or more compatible protocols and codecs
- Only codecs compatible with the output and the service are shown
- Audio codec can be chosen if various compatible, codec in simple mode and encoder advanced output mode.
- Will be extended with RFC 39

# Motivation
Create a better management of outputs, encoders, and services with an approach focused on protocols.

# Design

## Outputs API
Only outputs with the `OBS_OUTPUT_SERVICE` flag are concerned.

Adding to `obs_output_info`:
- `const char *protocols`: protocols separated with a semi-colon (ex: `"RTMP;RTMPS"`), required to register the output.
- `const char *protocols_prefixes`: URL protocols prefixes separated with a semi-colon `"rtmp://;rtmps://"`, optional.
  - Up to one URL prefix per protocol in `protocols`.
  - Those URL prefixes must be in the same order as `protocols`.
  - Each URL prefix must be unique to the protocol
  - Meant to be used for protocol auto-detection on server URL.
  - While registering if present, the number of URL prefix must be less or equal to the number of protocols.
  - NOTE: URL prefixes (URI scheme + `://`) are prefered over URI schemes to avoid:
    - matching links without prefixes (e.g. "rtmp.example.com").
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

- HLS relies on HTTP/HTTPS and it is not the only existing protocol in this case. This protocol has no unique URL prefixes.

- Because of the deprecation of the protocol, FTL will not have a URL prefix asign to its output to not make it available through custom server in the UI.

## Services API
Since a streaming service may not accept all the codecs usable by a protocol, adding a way to set supported codecs is required.

Adding to `obs_service_info`:
- `const char *(*get_protocol)(void *data)`: return the protocol used by the service. RFC 39 will allow multi-protocol services in the future.
- `const char **(*get_supported_video_codecs)(void *data)`: video codecs supported by the service. Optional, fallback to protocol supported codecs if not set.
- `const char **(*get_supported_audio_codecs)(void *data)`: audio codecs supported by the service. Optional, fallback to protocol supported codecs if not set.

Adding to the API these functions:
- `const char *obs_service_get_protocol(const obs_service_t *service)`: return the protocol used by the service.
- `const char **obs_service_get_supported_video_codecs(const obs_service_t *service)`: return video codecs compatible with the service.
- `const char **obs_service_get_supported_audio_codecs(const obs_service_t *service)`: return audio codecs compatible with the service.

### Services and connection informations

Depending on the protocol, the service should provide various types of connection details (e.g. server URL, stream key, username, password).

Currently, `get_key` can provide a stream id (SRT) or an encryption passphrase (RIST) rather than a stream key.

Rather than adding a getter for each possible type of information, a getter where we can choose which type of information we want should be implemented.

Each type of connection details will be defined by a macro with an even number value. Odd number values will be reserved for potential third-party protocols (RFC 39).

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
- `const char *(*get_connect_info)(uint32_t type, void *data)` where `type` is one of the value of the list above indicating which info we want and return `NULL` if it doesn't have it.
- `bool (*can_try_to_connect)(void *data)`, since protocols and services do not always require the same informations. This function allows the service to return if it can connect. Return true if the the service has all it needs to connect.

Adding to the API these functions:
- `const char *obs_service_get_connect_info(uint32_t type, const obs_service_t *service)`: return the connection information related to the given type (list above).
- `bool obs_service_can_try_to_connect(const obs_service_t *service)`: return if the service can try to connect.
  - return `bool (*can_try_to_connect)(void *data)` result.
  - return true if `bool (*can_try_to_connect)(void *data)` is not implemented in the service.
  - return false if the service object is invalid.

`obs_service_get_url()`, `obs_service_get_key()`, `obs_service_get_username()`, and `obs_service_get_password()` will be deprecated in favor of `obs_service_get_connect_info()`.

In the future, if we want to support a protocol that doesn't use a server URL, this scenario is covered by this design.

### About `rtmp-services`

`rtmp-services` is a plugin that provides two services `rtmp_common` and `rtmp_custom`. The use of "rtmp" in the naming no longer means that it only supports RTMP, it was and is kept to avoid breakage.

- In `rtmp_common`, services must at least support H264 as video codec and AAC or Opus as audio codec. This is a limitation required by the simple output mode.
- The service `rtmp_custom` will depend on protocol prefixes from outputs types to detect which protocol is in use.

#### About service JSON file

- For `services` objects, a `"protocol"` field will be added. This will be used to indicate the protocol for services in situation where:
  - The protocol can not be deduced from servers URL prefix.
  - The services is RTMP but has a HTTP url as its server URL have custom code in the plugin will have the protocol field enforced to RTMP.
- Services that use a protocol that is not registered will not be shown. e.g. OBS Studio without RTMPS support will not show services and servers that rely on RTMPS.
- Codecs field for audio and video will be added to allow services to limit which codec is compatible with the service.
- `"output"` field in the `"recommended"` object will be deprecated, but it will be kept for backward compatibility. `const char *(*get_output_type)(void *data)` in `obs_service_info` will be no longer used by `rtmp-services`.
  - The JSON schema will be modified to require the `"protocol"`field when the protocol is not auto-detectable. The same for the `"output"` field to keep backward compatibility.

## UI

### Custom server protocol auto-detection

The user can set a custom server through the service "Custom…", the protocol will be deduced based on URL prefixes from service outputs and default to RTMP if no matches.

NOTE: Allowing the user to manually choose the protocol (e.g. HLS) is part of RFC 39.

### Audio encoders

If the service/output is compatible with more than one audio codec, the user should be able to choose in the UI which one:
- In simple mode, the user will be able to choose the codec (AAC or Opus).
- In advanced mode, the user will be able to choose a specific encoder.

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