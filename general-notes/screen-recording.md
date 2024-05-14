# Screen recording

## OBS

A FOSS screen recording program would be the [Open Broadcaster Software project](https://obsproject.com/). While the Window capture doesn't work right for our Tauri app, we can capture the entire display and then trim the video with [Adobe Express](https://www.adobe.com/express/feature/video/crop).

To check the framerate, you can do

```bash
$ ffmpeg -i input.mp4 -r 30 dashboard.mp4
...
Input #0, mov,mp4,m4a,3gp,3g2,mj2, from 'dashboard.mp4':
  Metadata:
    major_brand     : isom
    minor_version   : 512
    compatible_brands: isomiso2avc1mp41
    encoder         : Lavf59.16.100
  Duration: 00:00:02.90, start: 0.000000, bitrate: 2883 kb/s
  Stream #0:0(und): Audio: aac (LC) (mp4a / 0x6134706D), 48000 Hz, stereo, fltp, 2 kb/s (default)
    Metadata:
      handler_name    : SoundHandler
      vendor_id       : [0][0][0][0]
  Stream #0:1(und): Video: h264 (High) (avc1 / 0x31637661), yuv420p(tv, bt709), 1230x1006 [SAR 1:1 DAR 615:503], 2901 kb/s, 60 fps, 60 tbr, 15360 tbn, 120 tbc (default)
    Metadata:
      handler_name    : VideoHandler
      vendor_id       : [0][0][0][0]
At least one output file must be specified

```

We see that the stream is at 60 fps. To reduce the framerate, you can do

```bash
$ ffmpeg -i dashboard.mp4 -r 30 output.mp4
```

Now the file size drops from nearly 1 MB to just 212 KB.

### Cropping video

To crop the video, we can either use the GUI Adobe Express tool, which inexplicably drives up file size sometimes, or we could use ffmpeg. For example, to remove the bottom row of pixels:

```bash
$ ffmpeg -i dashboard.mp4 -filter:v "crop=1230:1005:0:0" output.mp4
```

where the first two numbers are the width and height of the cropped video, and the last two numbers are the x and y coordinates of the top-left corner of the crop. We can see the original dimensions via

```bash
$ ffmpeg -i dashboard.mp4
...
  Stream #0:0(und): Video: h264 (High) (avc1 / 0x31637661), yuv420p(tv, bt709), 1230x1006 [SAR 1:1 DAR 615:503], 578 kb/s, 30 fps, 30 tbr, 15360 tbn, 60 tbc (default)
    Metadata:
      handler_name    : VideoHandler
      vendor_id       : [0][0][0][0]
...
```

which tells us that the original dimensions are 1230 pixels wide by 1006 pixels tall.
