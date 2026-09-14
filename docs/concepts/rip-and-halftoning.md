# RIP and halftoning

The RIP (raster image processor) converts continuous-tone images into 1-bit
fire/skip patterns. Colour conversion, ink coverage and dithering at this
stage determine much of the final print quality.

## The processing chain

1. **Resample.** The image is scaled with a high-quality filter so its
   pixels line up with the print grid, target size in mm times DPI.
2. **Colour separation.** RGB becomes the head's ink channels, which come
   from the head profile rather than being fixed: six
   on `c6n90`, four on `c4n180`. Neutral content is drawn from the black
   channel, grey-component replacement, instead of stacking C+M+Y, which
   wastes ink and shifts colour. Supply an ICC profile if you have one;
   without one a UCR/GCR conversion is used.
3. **Tone compensation and ink limiting.** Printed density is not linear in
   requested coverage, because dots overlap and spread. Midtones are pulled
   down before halftoning, per-channel scales balance the inks, and a
   total-ink cap keeps the substrate from flooding.
4. **Halftoning.** Each channel is dithered to 1 bit per pixel:
   Floyd–Steinberg error diffusion by default, or blue-noise masks, or
   ordered dithering.
5. **Pass slicing.** The bitmaps are cut into interleaved passes (see
   [Swaths and passes](swaths-and-passes.md)) and written out as the RIP
   payload the encoder reads.

## Why resolution changes tone

Drop size stays the same when DPI changes, but grid spacing gets smaller. At
360 dpi, the pitch is about 70 µm; at 630 dpi, it is about 40 µm. With the
reference drop size, a drop covers roughly three grid cells at 630 dpi. The
same requested coverage therefore deposits more ink per area, and coverage
above roughly 30% approaches solid.

Calibrate tone separately for each resolution. Reduce coverage as DPI
increases, using measured per-channel curves where possible.

## Calibration philosophy

To calibrate a print, print a target, measure the result and feed the
measurements back into the RIP. Calibration parameters stay in the RIP; the
firmware fires the patterns it receives.

The full workflow is in the rip-encoder module's `docs/RIP_USAGE_GUIDE.md`.

For the packing step that follows the RIP, see
[Printhead data protocol](printhead-protocol.md). For the CLI, see the
[RIP & Encoder component page](../components/rip-encoder.md).
