# Substance Painter ACES View Transforms
Here is a collection of ACES view transforms for Substance Painter.

Included are 3 SDR view transforms: P3D65, Rec.709, and sRGB.

## Substance Painter Setup
1. In the Display Settings, enable Activate Post Effects and Tone Mapping. Restore to defaults and set function to Log and mapping factor to max (63.99). This will tonemap scene linear into a 0-1 range.
  
   ![Substance Painter Display Settings - Tone Mapping](/images/screenshots/substance_painter_aces_setup_01_tonemapping.png) 
  
2. Add the desired color lut image to the Substance Painter shelf. On Linux this is `${HOME}/Documents/Allegorithmic/Substance Painter/shelf/colorluts/`.
3. Enable activate color profile, and select the color lut image you added.
  
   ![Substance Painter Display Settings - Activate Color Profile](/images/screenshots/substance_painter_aces_setup_01_tonemapping.png)


## Notes
- These transforms assume an ACEScg working colorspace.
- Since the log curve used by Substance Painter is undocumented, I have reverse-engineered an approximation of it. It's not exact but it should be quite close. I basically created an environment map containing a ramp, screenshotted this with the log tone mapping applied, and the linear_to_linear.exr color lut activated, then did a match grade in Nuke to create a color match between the two. An spi1d of this log curve to linear is included here as well.
