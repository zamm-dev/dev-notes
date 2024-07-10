# ffmpeg commands

## Converting MKV to MP4

If you just want to change the file format without caring about subtitles, we can use [this answer](https://askubuntu.com/a/396906):

```bash
$ ffmpeg -i input.mkv -codec copy output.mp4
```

If the MKV file has subtitles already, you can burn them into the output with [this answer](https://askubuntu.com/a/214351):

```bash
$ ffmpeg -i .\input.mkv  -filter_complex 'subtitles=input.mkv' -c:a copy output.mp4
```
