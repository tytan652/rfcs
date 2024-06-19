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

The output system, service management, settings windows and the auto-config wizard are the main components that will have a part of their code isolated to allow this path to be created.

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
   - Add Service APIs for multi-audio-track behavior

50. Re-implement OAuth integration (except YouTube) without docks

50. Implement service-agnostic "Broadcast Flow"
    - Re-implement YouTube integration without the docks

50. Implement service-agnostic bandwidth test APIs

70. Implement service-agnostic multi-video-track support
    - Add Service APIs for multi-video-track behavior
    - Twitch Enhanced Broadcast and WHIP Simulcast

99. Re-implement browser docks for service integration

99. Re-implement services without custom behavior
    - A service ID per service

99. Re-implement services with custom behavior

99. Make already existing service front-end API functional

## WIP

# Drawbacks

The overhaul will not be a 1:1 change, some features might not be portable to the more service-agnostic paradigm.

# Additional Information

This is a re-write of [Service Overhaul #39](https://github.com/obsproject/rfcs/pull/39) trying to mitigate the lack of being to incrementally merge changes without causing regressions.
