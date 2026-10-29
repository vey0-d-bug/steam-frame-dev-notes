# Graphics and video

## Vulkan

- GPU **Adreno 750** with Turnip (Mesa freedreno). The headset's driver reports `Mesa 26.3.0-devel (git-e07916b45f)`, Vulkan 1.4.362; host ICD `/usr/share/vulkan/icd.d/freedreno_icd.aarch64.json`, library `/usr/lib/libvulkan_freedreno.so`. The OpenXR runtime caps the app at Vulkan 1.2. Steam forces the instance layers `VK_LAYER_VALVE_rpo:VK_LAYER_VALVE_fdm_injection`. Valve's Mesa build is not public (not in upstream Mesa as of 2026-10-27). `mesa_version` on the headset prints it.
- Swapchain: 1728x1728, 3 images per eye. Formats offered, in order: `R8G8B8A8_SRGB`, `B8G8R8A8_SRGB`, `R32G32B32A32_SFLOAT`, `R32G32B32_SFLOAT`, `R16G16B16A16_SFLOAT`, `A2B10G10R10_UNORM_PACK32`.
- Device extensions offered: `VK_EXT_external_memory_dma_buf`, `VK_EXT_image_drm_format_modifier`, `VK_KHR_external_memory_fd`, `VK_KHR_external_semaphore_fd`, `VK_KHR_push_descriptor`, `VK_KHR_synchronization2`. **Not offered:** `VK_EXT_external_memory_host` (libplacebo assumes it and crashes without a patch) and **every Vulkan Video extension**, so decode goes through V4L2.
- With no host-pointer import, an upload from ordinary memory is a CPU copy into a staging buffer (about 5 ms per 8K NV12 frame). Copying decoded frames straight into host-mapped GPU buffers instead cut the render thread from 5.2 to 0.18 ms a frame.

## Hardware video decode (V4L2, Qualcomm iris)

- The decoder is `/dev/video22` (`/dev/video-dec0`), driver `qcom-iris-decoder`. Bitstream formats: H.264, HEVC, VP9, up to 8192x8192. **No AV1** decoder. Capture formats: `Q08C`, `NV12`, `NV21`, `AB24`, `QC24`, and **no P010**.
- FFmpeg's `h264_v4l2m2m` and `hevc_v4l2m2m` decode bit-exact against software. Headless speed: H.264 4K60 272 fps, HEVC 4K60 8-bit 235 fps, HEVC 7680x3840 30 about 62 fps, at a fraction of the CPU of software decode.
- FFmpeg n8.0.1's V4L2 decoders have no flush, so seeking or looping means closing and reopening (about 38 ms for H.264 4K, 80 to 126 ms for HEVC 8K). A second decoder opened while the first still holds buffers fails at large sizes (ENOMEM, EBUSY, POLLERR). Releasing FFmpeg's V4L2 context after its codec context is freed crashes in `ff_v4l2_context_release`. Seeking to a non-IDR HEVC keyframe logs one decode error and carries on.
- mpv's `v4l2m2m-copy` hung on every clip; FFmpeg's path works.
- **MV-HEVC** (Apple spatial video): `hevc_v4l2m2m` decodes the base layer only. Both views need software decode; with slice threads (the clips use WPP) 1080p both views ran at 2.76x real time. The 3D reference displays SEI decides which view is which eye: an iPhone file names view 0 the right eye, where x265 puts the left eye there.
- **After a sleep and wake, Steam's `steamwebhelper` holds `/dev/video22`** and 8K HEVC then fails (`Device or resource busy`, later `Cannot allocate memory`) while 4K still decodes. Waiting, opening Steam's menu, and restarting Steam don't free it; restarting the headset does.

## 10-bit on the hardware decoder

- 10-bit HEVC fed to iris with an `NV12` capture format decodes at full speed with **wrong colours** (magenta, blocky).
- After `SOURCE_CHANGE`, `G_FMT` offers 10-bit HEVC (Main10) and VP9 profile 2 only as **`Q10C`: UBWC-compressed TP10** (three 10-bit samples in 32 bits), single plane. 1920x1080 comes as 1920x1088 with a 2560-byte stride; 3840x2160 stride 5120; 7680x4320 stride 10240; 8192x4320 as 8256x4320 stride 11008 (width padded to 192-pixel tiles). `MIN_BUFFERS_FOR_CAPTURE` is 4. Twelve buffers at 8192x4320 are about 820 MB.
- **The headset's own Turnip cannot sample Q10C.** A separately built Turnip (upstream Mesa main `9fb60e5e`, 27.0.0-devel) with a small TP10 patch, after Stereonaut's `tools/mesa-tp10` (MIT, github.com/Erdnusschokolade/stereonaut), reads the decoder's dma-buf as TP10 Y and UV planes when imported with an explicit QCOM UBWC layout. Loaded for one process only through `VK_ICD_FILENAMES` / `VK_DRIVER_FILES`, nothing on the headset changes and no root is needed.
- Results with that driver: planes bit-exact against FFmpeg's software decode at 1080p, 4K, 7680x4320 and 8192x4320; in a player, within 2/255 of software for ramps, SMPTE bars, HDR10 (tone-mapped) and VP9 profile 2. Pacing at 120 Hz: HEVC Main10 4K60 0.0 % missed after startup over 3 minutes, 7680x4320 30p 0.03 %, 8192x4320 30p 0.22 %. A new decoder client per seek opened in 15 to 34 ms (8K 42 to 53 ms).
- Driver A/B at 8K30, missed frames after startup: the headset's own driver 0.90 %, Mesa main 9fb60e5e 0.90 %, 26.3-rc2 1.03 %, Stereonaut's pin 8c3c0b99 1.26 %.
- A new client's `VIDIOC_STREAMON` on the capture queue at 8192x4320 failed with ENOMEM while the previous client's buffers were still alive; free the old client's buffers (dma-bufs and GPU imports) at close.
- **iris drops HEVC's HDR10 SEI** (mastering display, content light level), and x265 MP4s have no container copy, so read it from the packets yourself or the picture is tone-mapped about 10 % too dark.
- Build the bundled driver with libdrm and zlib static and libstdc++ static and hidden, and it needs only glibc 2.31 or newer.

## Software decode, for comparison

On synthetic 40 to 80 Mb/s clips, real-time factor with FFmpeg's software decoders: H.264 1080p30 8.8x, H.264 4K60 2.5x, HEVC 4K60 8-bit 1.9x, 10-bit 1.5x, HEVC 5760x2880 60 0.98x, HEVC 7680x3840 30 0.99x, AV1 4K60 10-bit 1.8x, AV1 7680x3840 30 10-bit 1.45x (dav1d inside a VR app reached only about 60 fps at 4K). Software HEVC 4K60 10-bit in VR needs about 4 cores and runs the CPU at 93 to 96 °C.
