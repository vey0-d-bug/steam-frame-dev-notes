# Compositor layers

The runtime offers 16 layers but no cylinder, equirect or cube layers, so the useful ones are projection and quad layers.

- **Quad layers (`XrCompositionLayerQuad`) are shown.** Six variants, 20 s each at 1.5 m (a 1600x160 region of a 1600x800 swapchain, its own 1600x160 swapchain, with SAMPLED and TRANSFER_DST usage, opaque, and a 1280x720 one at 1.0 x 0.5625 m) were all created, submitted for about 2400 frames each at 120 Hz, and visible worn.
- **Opaque quads look best.** Hard-edged panels with no mip chain looked heavily aliased on blended quads; the one opaque quad (no blend flag) looked really good. A UI on an opaque quad looked sharper than the same UI drawn into each eye's projection, and the eye-drawn one sharper than a blended quad.
- **Projection layers are composited under quads, whatever the submit order.** A premultiplied projection layer submitted after a quad (for controllers and a laser pointer) did not show over the quad: the laser tip appeared behind the panel.
- **A later quad draws over an earlier one**, and `eyeVisibility` LEFT/RIGHT shows a quad to one eye only. Per-eye premultiplied quads after an opaque menu quad put a laser and a controller over the menu correctly, never doubled, with 0 of 13058 frames missed at 120 Hz.
- **Cost:** a 1600 px-wide UI on its own opaque quad cost nothing visible in pacing (0 of 2403 frames missed over playing video); the app's CPU for the UI dropped from 1.1 to 1.3 ms a frame (drawn into the eyes) to 0.1 to 0.2 ms.
- Reading a layer back for test captures: use the driver's host-visible, coherent and **cached** memory type. Through uncached write-combined memory a capture took 0.3 to 1 s and stalled the frame loop; cached took 15 to 50 ms.
- At the recommended 1728x1728 eye resolution, a 1600 px UI at 1.5 m lands at about 0.40 eye pixels per texture pixel, so fine detail (2 px line pairs) is lost. Render scale 1.25 / 1.5 / 2.0 gives 0.50 / 0.60 / 0.80. Valve's foveation layer (`VK_LAYER_VALVE_fdm_injection`) made no measurable difference to UI sharpness.
