# Mac-specific notes

## Writing to NTFS drives

You can try the instructions for [Mounty](https://mounty.app/#installation):

```bash
$ brew install --cask macfuse
$ brew install gromgit/fuse/ntfs-3g-mac
$ brew install --cask mounty
```

On first load, you'll see a message like "System Extension Blocked"

> The system extension required for mounting macFUSE volumes could not be loaded.
>
> Please open the Privacy & Security System Settings and allow loading system software from developer "Benjamin Fleischer". A restart might be required before the system extension can be loaded.
>
> Then try mounting the volume again.

Once you try to do that, you get the additional message

> **To enable system extensions, you need to modify your security settings in the Recovery environment.**
>
> To do this, shut down your system. Then press and hold the Touch ID or power button to launch Startup Security Utility. In Startup Security Utility, enable kernel extensions from the Security Policy button.

You'll have to restart twice: once to allow kernel extensions, then another time to actually enable that specific kernel extension. Once you do so, the external volume will no longer be mounted under `/Volumes/`, but instead under your home directory at (for example) `/Users/amos/.mounty/external`.
