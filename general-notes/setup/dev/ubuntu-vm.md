# Ubuntu VM on Apple Silicon

Since VirtualBox doesn't support Apple Silicon, we can instead try to use `multipass` as mentioned [here](https://www.xda-developers.com/how-to-run-ubuntu-vm-apple-silicon-free/).

```bash
$ brew install multipass
$ multipass launch 24.04 -n dev -c 4 -m 4G -d 50G
```

If you get

```
launch failed: Downloaded image hash does not match
```
