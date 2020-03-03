# OCIO Configs

# [Modular ACES OCIO Config](/ocio-configs/config_modular-aces.ocio)
![Modular ACES OCIO Config](/images/modular-aces-ocio-config_screenshot.png)
An OCIO config based on the [ACES 1.2 OCIO config](https://github.com/colour-science/OpenColorIO-Configs/tree/feature/aces-1.2-config/aces_1.2) from the [colour-science](https://github.com/colour-science) [OpenColorIO-Configs git repo](https://github.com/colour-science/OpenColorIO-Configs). 

This description is going to be technical and wordy, so apologies in advance. Here's a pretty picture to keep you interested.

![Nuke OCIO Screenshot - DRT](/images/modular-aces_screenshot_nuke-drt.jpg)


### Terms
A very important thing when talking about color science topics is to define your terms and make sure they are clear. To whit, here is a list of terms I'll use below!

- **DRT** - Display Rendering Transform : This is not an ACES term, but I'm borrowing it [from Filmlight](https://vimeo.com/119143638) to describe the color transformation from scene linear wide gamut to display device. This transform generally contains some type of look, a tonemapping transform to map from scene linear to display linear, gamut transformations to map from working colorspace to display colorspace, and an EOTF for the display device.
- **RRT** - The ACES Reference Rendering Transform : Transforms from ACES to OCES. Contains the segmented_spline_c5 tonescale algorithm.
- **OCES** - The ACES Output Color Encoding Specification : A display-referred colorspace targeted for an idealized wide gamut hdr display device with AP1 primaries, and a 10000 nit dynamic range, in floating point candelas per meter squared encoding. (Some more information in [these](https://acescentral.com/t/odt-tonescale/387/11) [threads](https://acescentral.com/t/color-appearance-models-interpretation-options-and-odts/1257/4) on [acescentral.com](https://acescentral.com)).
- **ODT** - The ACES Output Device Transform : Transforms from OCES to a specific display device. Contains the segmented_spline_c9 tonescale algorithm.
- **SSTS** - The ACES Single Stage Tonescale : This is a newer tonescale algorithm which combines the segmented_spline_c5 and segmented_spline_c9 tonescales into a single parameterized tonescale that is simpler for HDR display transforms. In ACES 1.2, this tonescale is used for all HDR display transforms. The SDR display transforms still use the older RRT+ODT tonescale algorithms, but this [might change](https://acescentral.com/t/nuke-aces-output-transform-and-a-question/2554/3) with the ACES 2.0 Output Transforms.
- **LMT** - [Look Modification Transform](https://github.com/ampas/aces-dev/tree/master/transforms/ctl/lmt) : A color transform to apply a creative look to an image. This is an ACES -> ACES' transform.
- **CTL** - Color Transformation Language : The programmiing language that the source ACES code is written in.
- **EOTF** - Electrical to Optical Transfer Function : Transfer function to transform from electrical domain to optical domain, for example from pixels to a monitor.
- **Shaper** - A shaper space is a transfer function that transforms the input data into a more convenient representation for performing certain operations. Commonly, it is a linear to log transfer function which remaps scene linear into a 0 to 1 range, so that the available precision of a color transform like a 3d lut might be better utilized.
- **Code Values** - A term used to refer to the integer encoding common in display-referred images. For example an 8 bit jpeg image from the internet has 256 code values or discreet steps between 0 (black) and 1 (white).


![Nuke OCIO Screenshot - DRT](/images/modular-aces_screenshot_nuke-idt.jpg)

Here's another screenshot of the config in action in Nuke.

With the increasing amount of display rendering transforms (DRT), ACES OCIO configs have gotten increasingly large in file size. The [luts](https://github.com/colour-science/OpenColorIO-Configs/tree/feature/aces-1.2-config/aces_1.2/luts) folder of the latest config is 429.5MiB. This filesize is mostly from the fact that each DRT is baked out into a 3d lut. This lut describes the color transformation from the shaper log encoding of scene linear to the final code values for display on a monitor. The ACES config uses 65x65x65 luts, so each lut is about 9-10MiB. 

When building the [Nuke ACES Output Transforms](https://github.com/jedypod/nuke-colortools), I started wondering if it had to be this way. The ACES Output transform has different stages, many of which are the same for each display render transform. Here's a breakdown of the basic stages of a typical ACES display rendering transform:
1. All have the default ACES LMT, described as the `rrt_sweeteners` in the [AMPAS CTL](https://github.com/ampas/aces-dev/blob/master/transforms/ctl/lib/ACESlib.RRT_Common.ctl). Technically, the rrt_sweeteners are included in the Reference Rendering Transform (RRT), but in my opinion, the function of this component of the RRT is to modify the look of the image to render the colors in a more visually appealing way, and thus is should be thought of as an LMT. It's also useful to conceptualize it this way because the `rrt_sweeteners` is actually the only component of the typical ACES display render transform that actually needs a 3d lut. (!)
2. A tonescale or tonemapping operation to compress the high dynamic range scene linear working space into low dynamic range display linear code values. This operation can be described by a single channel 1d lut since it is only a curve.
3. A gamut transformation to convert from the working colorspace gamut to the display gamut (with optional gamut limiting and whitepoint compensation). This operation can be described by a matrix transform.
4. A device transfer function, or EOTF, which compensates for the transfer function of the display.

So instead of doing this in the OCIO config:
```
                |-------|
                |       |
      ACES ---->|3D LUT |--> code values
                |       |
                |-------| 
```

We could do something like this:

```

          |--------|      |-------|      |-------|      |-------|      |-------|
          |        |      |       |      |       |      |       |      |       |
ACES ---->| SHAPER |----->|  LMT  |----->| SSTS  |----->|  PRI  |----->| EOTF  |--> code values
          |        |      |       |      |       |      |       |      |       |
          |--------|      |-------|      |-------|      |-------|      |-------| 

```

It's a bit more verbose to write out, but the advantage is that we only need a single 3D LUT to describe all display rendering transforms.

That takes our filesize from 429.5MiB to around 30MiB. Interesting! 

Of course filesize in an OCIO config is not the primary concern. Easily understanding what is going on is probably the most important thing, and I don't claim this approach is better. It does make it more clear what is going on in the ACES output transform though, and I think this is useful for learning. 

Another thing I don't like about the standard ACES OCIO configs is how much duplication of information there is. For example the color matrix to transform from ACES to ACEScg is duplicated many many times. To try to avoid this, I've taken the approach of building each transform as a modular colorspace. Each gamut transformation is defined as a colorspace. Each transfer function for each log space is defined as a colorspace. Then each Input Device Transform (IDT) is defined as a combination of the other colorspaces. 

To keep things simple I'm using a single shaper log space for all output transforms: ACEScct. I'm sure there is a good reason that there are multiple shaper spaces being used in the official ACES config, probably optimization, but I haven't found a visual difference strong enough to warrant the increased complexity.


![Blender OCIO Screenshot](/images/modular-aces_screenshot_blender.png)
And here's a picture of it in Blender.


### Show Config
I've also been doing some thinking about color pipelines for balance grades (see [CalibrateMacbeth](https://gist.github.com/jedypod/798b365ea64e8121999e7036ae7e0217) and [BalanceGrade](https://gist.github.com/jedypod/d13595f856976869fe4cacd265a2b15e)). I've included a sketch of an idea for how a VFX company might set up a show. Generally in VFX you receive plates in camera colorspcae. These might be scene-linear exr files with Alexa Wide Gamut primaries, or DPX files encoded in RedLog3G10 / RedWideGamutRGB, or sometimes even weirder formats.

It is an advantage for streamlining and simplifying a color pipeline to have a single common working colorspace company wide that all images are transformed into on ingest. The most common emerging practice seems to be ACEScg. 

To that end, this config includes a few colorspace definitions to describe those image states:
- work_lin: The working scene-linear colorspace.
- work_log: The working log colorspace.
- work_dmp: uint16 display-referred hdr wide gamut working colorspace for digital matte painting.
- src_lin: The external source image scene-linear colorspace.
- src_log: The external source image log colorspace.
- balance: The per-shot internal balance grade. Applied in work_lin.
- shot_grade: The external source DI shot grade.
- shot_grade_internal: The internal shot grade.
- view_show: The show view transform.
- view_log: The show log transform.
- view_shotgrade: The shot di grade view transform. Balance grade inverted if there is one.
- view_balance: The balance view transform: work_lin -> shot_grade_internal -> view_show.





# [Minimal ACEScg OCIO Config](/ocio-configs/config_minimal-acescg.ocio)
This is a minimal ACES OCIO config that uses ACEScg as the reference colorspace instead of ACES 2065-1 / AP0. It includes only the less obscure IDT transforms, and includes only a subset of the display render transforms.


# [Truelight ACES OCIO Config](/ocio-configs/config_truelight-acescg.ocio)
This config utilizes the Truelight CAM display rendering transforms instead of the ACES output transforms. It also includes the Filmlight scene look transforms (basically LMTs) as 3D LUTs. For more information, check out the [trulight lut creation folder](/lut-creation/truelight/).