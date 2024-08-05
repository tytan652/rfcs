# Summary
- Make OBS able to accept third-party service plugins
- Register services with a unique id rather than a common one
- A service can be provided with multiple protocols
- Separate Twitch, Restream and YouTube integration and make them plugins
- Forbid service-specific feature in the UI code

# Motivation
Actually even if OBS has the Service API, developer can't create third-party service plugin because there is no mechanism to use them at all.

Before in OBS in the Stream settings page, property views were used for services but with only two registered services `rtmp_common` which contain all services and `rtmp_custom` for custom servers.

Currently in OBS, this page shows the list of services with many new elements shown or hidden depending on the selection (like recommended settings). With also Twitch and Restream OAuth integration. And no use of the property views provided by `rtmp-services`.
Also multiple actors pushed their own features for their service which increase the UI code complexity.

This need to be refactored to re-introduce property views for the service and for the protocol output.

This will also provide the ability for some stream services to be able to make their own plugin.

# Design
## Dual code-path
Like it was said in [Motivation](#motivation), various actors want to push their own features causing potential overhaul to be redone since the code base can be heavily changed by an actor's new feature.

Blocking the actors does not seem like a possibility at all, so the overhaul needs to try to enable avoiding blocking them at the price that they might see they work re-done/worked later.

So OBS Studio will have two code path, the original path where actors can push their changes and the experimental one.

This only concerns the UI side of OBS Studio, libobs and plugins will not have two paths.

Since the pushed feature can cause re-basing issue in a one path, using a dual path could mitigate a little this issue.

### Original path
This path will only receive isolation and API-based changes, most of the path will not change except if an actors push a change which will only affect this path.

### Experimental path
In this path, features will implemented progressively since we do not directly replace the original path, this makes the overhaul more incremental.

The output system, service management, settings windows and the auto-configuration wizard are the main components that will have a part of their code isolated to allow this path to be created.

Note about previously mentioned actors, until the experimental path becomes the default/only path. If those actors happens to want to push changes to the experimental path, those changes, if not bug fixes, will be ignored as the path is experimental and not open for external contribution.

### How to select the path
Until the experimental path can be considered to be defaulted to, the original path will always be used unless:

1. The user add a custom option to opt-in to the experimental path.
2. A menu option once the experimental is considered testable by a wider range of users.

## Pseudo-road-map
This is road-map of potentials steps to implement the overhaul through the experimental path.

The order of some steps is not definitive.

1. Creation of the experimental path
   - Isolation of the original path
   - Enabling the experimental path will only disable the streaming feature

2. Enable streaming with only "Custom…" re-implemented
   - A custom service that supports all first-party support protocols in its own plugin
   - The custom service will not support "Twitch Enhanced Broadcast". (TODO: Move this sentence)

3. Re-implement service that have integration but without their integration
   - Integration will be progressively re-added in other steps
   - Add Service API for multi-audio-track behavior

50. Re-implement OAuth integration (except YouTube) without docks

50. Implement service-agnostic "Broadcast Flow"
    - Re-implement YouTube integration without the docks

50. Implement service-agnostic bandwidth test API

70. Implement service-agnostic multi-video-track support
    - Add Service API for multi-video-track behavior
    - Twitch Enhanced Broadcast and WHIP Simulcast

99. Re-implement Auto-Configuration Wizard streaming parts

99. Re-implement browser docks for service integration

99. Re-implement services without custom behavior
    - A service ID per service

99. Re-implement services with custom behavior

99. Make already existing service front-end API functional

99. Implement settings migration from original path to experimental

## Settings UI
Many elements of the settings UI related to services will be moved inside service properties view like:

- Stream key field
- Username and password fields
- OAuth connect disconnect button
- Text with clickable link (e.g., YouTube integration links)
- Maximum and recommended settings information
- "Get Stream Key" button
- "More Info" button

The streaming output properties view will be shown below the service properties.

If there multiple streaming output type available for the same protocol, this will be select-able through the settings windows.

The service output settings (if any) will be saved as a JSON string in the profile config file.

Advanced network settings will also be replaced by this new properties view, since those were only meant for `"rtmp_output"` (RTMP(S) output).

## Front-end API
Those functions will be modified:

- `obs_frontend_set_streaming_service()` because while the output selection is done in the settings windows and is no longer done while the stream is starting. The API will swap in a opinionated way the output if the service and the actual output protocols do not match.
- `obs_frontend_save_streaming_service()` will save service output settings.

## Services
### API

The following is subject to change.

Adding to `obs_service_info`:

  - `uint32_t flags` with the following flags:
    - `OBS_SERVICE_DEPRECATED`: The service is marked as deprecated.
    - `OBS_SERVICE_INTERNAL`: The service is meant to be used internally in some plugin (e.g., WebSocket, Scripting) and usually not directly exposed in the UI.
    - `OBS_SERVICE_UNCOMMON`: The service can be hidden behind a "Show All/More" option UI/UX-wise.
  - `const char *supported_protocols`: Protocol supported by the service.
  - `enum obs_service_audio_track_cap (*get_audio_track_cap)(void *data)`: Returns the service audio track capability with the following possible values:
    - `OBS_SERVICE_AUDIO_SINGLE_TRACK` - Only a single audio track is used by the service
    - `OBS_SERVICE_AUDIO_ARCHIVE_TRACK` - A second audio track is accepted and is meant to become the archive/VOD audio
    - `OBS_SERVICE_AUDIO_MULTI_TRACK` - Supports multiple audio tracks
  - `bool (*can_bandwidth_test)(void *data)`: Return if the service is able to do bandwidth test, there is situations were a service is not always able to do it (e.g. the YouTube integration only does when an is account connected)
  - `void (*enable_bandwidth_test)(void *data, bool enabled)`: Enable bandwidth test on the service (TODO: Error pointer)
  - `bool (*bandwidth_test_enabled)(void *data)`: Return if the service has the bandwidth test enabled
  - `void (*get_supported_resolutions2)(void *data, struct obs_service_resolution **resolutions,size_t *count, bool *with_fps)`: Replace its non-two variant to enable framerate value
  - `int (*get_max_video_bitrate)(void *data, const char *codec struct obs_service_resolution resolution)`: Return a maximum bitrate based on a video codec and a resolution
  - `int (*get_max_codec_bitrate)(void *data, const char *codec)`: Return a maximum bitrate for a specific codec
  - `void (*apply_encoder_settings2)(void *data, const char *encoder_id, obs_data_t *encoder_settings)`: Replace its non-two variant to enable settings per encoder id and codec

Adding to the Services API:

  - `enum obs_service_audio_track_cap obs_service_get_audio_track_cap(const obs_service_t *service)`: Returns the service audio track capability with the following possible values:
    - `OBS_SERVICE_AUDIO_SINGLE_TRACK` - Only a single audio track is used by the service
    - `OBS_SERVICE_AUDIO_ARCHIVE_TRACK` - A second audio track is accepted and is meant to become the archive/VOD audio
    - `OBS_SERVICE_AUDIO_MULTI_TRACK` - Supports multiple audio tracks
  - `uint32_t obs_get_service_flags(const char *id)` and `uint32_t obs_service_get_flags(const obs_service_t *service)`: Return services flags
  - `const char *obs_get_service_supported_protocols(const char *id)`: Return all protocols that the service can support
  - `bool obs_service_can_bandwidth_test(const obs_service_t *service)`: Return if the service has bandwidth test capability
  - `void obs_service_enable_bandwidth_test(const obs_service_t *service, bool enabled)`: Enable/disable the service bandwidth test
  - `bool obs_service_bandwidth_test_enabled(const obs_service_t *service)`; Return if the service bandwidth test is enabled
  - `int obs_service_get_max_codec_bitrate(const obs_service_t *service, const char *codec)`: Return the maximum bit-rate supported by the service depending on the codec
  - `void obs_service_get_supported_resolutions2( const obs_service_t *service, struct obs_service_resolution **resolutions, size_t *count, bool *with_fps)`: Return resolutions  supported by the service with optional frame-rate
  - `int obs_service_get_max_video_bitrate(const obs_service_t *service, const char *codec, struct obs_service_resolution resolution)`: Return the maximum bitrate supported by the service depending on the codec and a resolution
  - `void obs_service_apply_encoder_settings2(obs_service_t *service, const char *encoder_id, obs_data_t *encoder_settings)`: Apply encoder settings for a specific encoder id

Deprecating in the Services API:

  - `obs_service_apply_encoder_settings`: replaced by `obs_service_apply_encoder_settings2` to allow per encoder id/codec settings
  - `obs_service_get_supported_resolutions`: replaced by `obs_service_get_supported_resolutions2` to enable returning frame-rate
  - `obs_service_get_max_bitrate`: replaced by `obs_service_get_max_codec_bitrate` and  `obs_service_get_max_video_bitrate` for per codec (and resolution for the second) bit-rate

### Plugins
#### `rtmp-services`

Services provided by this plugin (`"rtmp_custom"`, `"rtmp_common"`) will be deprecated (`OBS_SERVICE_DEPRECATED` and `OBS_SERVICE_INTERNAL` applied) and completely unused in OBS Studio experimental path.

Those are deprecated rather than completely removed to allow scripting and plugins to migrate if they happen to use them.

Its service JSON will no longer be updated.

This plugin will be replaced and its two services will be replaced by two plugins.

#### `obs-webrtc`
`"whip_custom"` will be deprecated (`OBS_SERVICE_DEPRECATED` and `OBS_SERVICE_INTERNAL` applied) and completely unused in OBS Studio experimental path.

The service will be replaced by one implemented in another plugin to avoid co-dependency between the output and the service implementation.

#### `custom-service`
This plugin is meant to provide replacement to `"rtmp_custom"` and `"whip_custom"` and extend it to every protocol that OBS Studio support.

#### `obs-services`
This plugin is meant to provide a replacement for `"rtmp_common"` type for services who don't rely on custom behavior nor integration.

Each service will be registered with its own id and not the same id for all services.

This new plugin will be able to provide multi-protocol services so no more "Service - HLS" and "Service - RTMP".

# Drawbacks
The overhaul will not be a 1:1 change, some features might not be portable as-is to the more service-agnostic paradigm.

As an example, URI scheme detection will be completely dropped since not all protocol can have this applied (e.g., HLS and WHIP are both HTTP based and share same URI schemes).

# Additional Information
This is a re-write of [Service Overhaul #39](https://github.com/obsproject/rfcs/pull/39) trying to mitigate the lack of being able to incrementally merge changes without causing regressions.

Required addition for browser-based features:

- https://github.com/obsproject/obs-browser/pull/431
- https://github.com/obsproject/obs-studio/pull/10516
