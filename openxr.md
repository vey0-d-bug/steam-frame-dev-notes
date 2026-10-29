# OpenXR

## The runtime

- **SteamVR/OpenXR 2.18.4**, host manifest `/opt/steamvr/steamxr_linuxarm64.json` (via `~/.config/openxr/1/active_runtime.json`). Inside Steam's pressure-vessel containers the manifest is at `/usr/lib/pressure-vessel/overrides/etc/xdg/openxr/1/active_runtime.json`. System name "SteamVR/OpenXR : cv", vendor 0x28de. Recommended 1728x1728 per eye, max swapchain 8192x8192, 16 layers, blend modes opaque and alpha-blend.
- Vulkan through `XR_KHR_vulkan_enable2` v4, accepting Vulkan 1.0 to 1.2.
- **49 instance extensions.** Notable: `XR_FB_display_refresh_rate`, `XR_FB_foveation(_vulkan)`, `XR_FB_space_warp`, `XR_EXT_user_presence`, `XR_EXT_hand_tracking`, `XR_EXT_eye_gaze_interaction` (v2), `XR_VALVE_frame_controller_interaction`, `XR_KHR_generic_controller`, `XR_EXT_local_floor`, `XR_EXT_render_model`, `XR_EXT_interaction_render_model`, `XR_META_foveation_eye_tracked`.
- **Not offered:** `XR_KHR_composition_layer_cylinder`, `_equirect`, `_equirect2`, `_cube`. Curved screens and 180/360 video must be rendered by the app into projection layers (or onto quads, see [compositor-layers.md](compositor-layers.md)).
- An OpenXR app started over ssh (with `DISPLAY=:1`, the session's `XDG_RUNTIME_DIR` and Steam's `VK_INSTANCE_LAYERS`) gets a normal session on the headset's runtime, the same as a library launch.
- Several sessions one after another in separate processes from one launch work cleanly.

## Views and spaces

- Eye views are **asymmetric**. Left eye: left -58.9°, right 50.5°, up 49.4°, down -60.0°. Right eye: left -50.4°, right 59.1°, up 49.7°, down -59.7°. Straight ahead is not the centre of either eye's image. Eye separation measured 61.9 mm. The views move slightly frame to frame with the head still.
- The STAGE space's floor was **not usable** as the real floor (0.30 m below head height on one day, 0.41 m above it the next). Use a fixed offset from LOCAL instead.

## Controllers

- Controllers emulate Oculus Touch by default. The native profile is `/interaction_profiles/valve/frame_controller_valve` (`XR_VALVE_frame_controller_interaction`, not in the Khronos OpenXR-SDK 1.1.65 headers). Right: a/b/x/y, menu, system, bumper, squeeze value, trigger value/click, thumbstick, grip/aim pose, haptic. Left: dpad up/down/left/right, view, system, bumper, squeeze, trigger, thumbstick, poses, haptic.
- The native profile is accepted and becomes active on both hands at focus. Trigger, grip, thumbstick (including its click), haptics and bumpers reach the app. **View (left) and Menu (right) clicks never reach the app**; the system keeps them.
- Runtime controller models (`XR_EXT_render_model`, `XR_EXT_interaction_render_model`): `xrEnumerateInteractionRenderModelIdsEXT` returns `XR_ERROR_SESSION_NOT_RUNNING` before the session runs, and lists **no models while the controllers sleep**, so cache the last model each hand had. SteamVR's own copies are OBJ/MTL/PNG under `/opt/steamvr/drivers/frame_controller/resources/rendermodels/` (Valve's files; leave them on the headset).

## Eye gaze

- `XR_EXT_eye_gaze_interaction` works for a standalone app with **no prompt and no calibration step**. `supportsEyeGazeInteraction` is true, the gaze pose bound on `/interaction_profiles/ext/eye_gaze_interaction` is accepted, and `/user/eyes_ext` takes the profile at focus. `XrEyeGazeSampleTimeEXT` is filled.
- About **90 new samples a second** under a 120 Hz display (the located pose changes every frame, so the runtime interpolates). Lag from sample time to the frame's predicted display time: 45.5 ms p50, 49 ms p95, 66 ms worst, so the tracker runs about 30 to 37 ms behind the eye.
- Accuracy to dots at 1.5 m (one wearer, one run): raw mean 4.0°, p95 5.6°, almost all of it a steady offset (about 2.9° left, 2.2° up). After a five-dot fit (offset and gain per axis): mean 1.5°, p95 2.7°. The floor (each dot's own mean removed) is about p95 2.1°. Plan on a quick calibration and targets of at least 4°.
- Closed eyes are not reported as a tracking loss: the pose stays valid and tracked and holds its last spot. With input focus on the Steam menu the action stays active but no new sample times come.
- **Unworn over ssh the session never reaches FOCUSED**, so no action is ever active and gaze can't be tested without a wearer.

## Focus, presence and the system menu

- Opening the Steam menu over the app takes input focus (FOCUSED to VISIBLE); closing it gives it back.
- Over ssh, SteamVR sometimes withholds focus: a few seconds, or the whole run. With the headset unworn and the proximity sensor covered, no ssh run got focus at all on one night. Automated tests shouldn't depend on focus.
- `XR_EXT_user_presence` sends real `headset off` / `on` events. Started over ssh with the headset asleep or unworn, the first event is `off` and the session stays SYNCHRONIZED (never VISIBLE).

## Passthrough

- Passthrough is `XR_ENVIRONMENT_BLEND_MODE_ALPHA_BLEND` (Valve's SteamVR OpenXR feature list says so; `XR_FB_passthrough` is unsupported). A session in that mode with the layer's source-alpha flag shows the room around a flat screen. There is no camera-frame access for apps (OpenVR issue #1926).
- Switching between opaque and alpha-blend per frame in `xrEndFrame` works mid-session, with no restart and no missed frames.
