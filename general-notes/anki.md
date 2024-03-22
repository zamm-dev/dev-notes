# Installing Anki

Download and install from [this page](https://apps.ankiweb.net/). Extract the folder, which should be named something such as `anki-23.12.1-linux-qt6`, cd to it, and then run

```bash
$ sudo ./install.sh
```

Unfortunately, the application fails to start up. When running from the commandline, we see this error:

```bash
$ Anki starting...
Initial setup...
Preparing to run...
  File "<string>", line 1, in <module>
  File "aqt", line 509, in run
  File "aqt", line 583, in _run
  File "aqt.profiles", line 139, in setupMeta
  File "aqt.profiles", line 421, in _loadMeta
resetting corrupt _global
Qt warning: QGuiApplication::setDesktopFileName: the specified desktop file name ends with .desktop. For compatibility reasons, the .desktop suffix will be removed. Please specify a desktop file name without .desktop suffix 
Qt warning: From 6.5.0, xcb-cursor0 or libxcb-cursor0 is needed to load the Qt xcb platform plugin. 
Qt info: Could not load the Qt platform plugin "xcb" in "" even though it was found. 
Qt fatal: This application failed to start because no Qt platform plugin could be initialized. Reinstalling the application may fix this problem.

Available platform plugins are: wayland-egl, offscreen, wayland, eglfs, xcb, minimal, vnc, linuxfb, vkkhrdisplay, minimalegl.
 
[1]    9646 IOT instruction (core dumped)  anki

```

We try installing it:

```bash
$ sudo apt install libxcb-cursor0
```

and find that it now works.
