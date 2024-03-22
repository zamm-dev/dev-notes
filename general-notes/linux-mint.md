# Setting up Linux Mint

## Keyman

Follow the instructions [here](https://keyman.com/linux/) to install Keyman:

```bash
$ sudo add-apt-repository ppa:keymanapp/keyman
$ sudo apt install keyman onboard-keyman
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
Package onboard-keyman is not available, but is referred to by another package.
This may mean that the package is missing, has been obsoleted, or
is only available from another source

E: Package 'onboard-keyman' has no installation candidate
```

The first one doesn't work, so we try the second set of commands:

```bash
$ (wget -O- https://packages.sil.org/keys/pso-keyring-2016.gpg | \
  sudo tee /etc/apt/trusted.gpg.d/pso-keyring-2016.gpg)&>/dev/null
$ (. /etc/os-release && sudo tee /etc/apt/sources.list.d/packages-sil-org.list >/dev/null \
  <<< "deb http://packages.sil.org/ubuntu $UBUNTU_CODENAME main")
$ sudo apt update
$ sudo apt install keyman onboard-keyman
```

Now "Keyman Configuration" should show up.

Follow some of the steps [here](https://help.keyman.com/knowledge-base/101) to create `~/.xsessionrc` as such:

```bash
export GTK_IM_MODULE=ibus
export XMODIFIERS=@im=ibus
export QT_IM_MODULE=ibus
```

Install some keyboard in Keyman (for example, `khmer_angkor`). Start the ibus daemon, and then in ibus preferences, pick the language for the Keyman keyboard you just installed. It will then ask you to pick from different available keyboards for that language, and the one you just installed on Keyman should be visible.

We find that some applications don't recognize the new keyboard input until we restart the entire computer.

### Starting ibus automatically

To start ibus on account login, add this line to your `~/.profile``:

```bash
ibus-daemon --xim &
```

This will run the ibus daemon in the background during the login process.

## Increasing swap size

Follow the instructions [here](https://old.reddit.com/r/linuxmint/comments/uhjyir/how_to_increase_swap_size/i76gsi9/):

```bash
$ sudo swapoff /swapfile
$ sudo fallocate -l 4G /swapfile
$ sudo mkswap /swapfile
mkswap: /swapfile: warning: wiping old swap signature.
Setting up swapspace version 1, size = 4 GiB (4294963200 bytes)
no label, UUID=1d12c995-fbfc-4947-b8a5-067a0a9e3721
$ sudo swapon /swapfile
```

If you get a warning such as

```bash
$ sudo mkswap /swapfile
mkswap: /swapfile: insecure permissions 0644, fix with: chmod 0600 /swapfile
Setting up swapspace version 1, size = 4 GiB (4294963200 bytes)
no label, UUID=a13674ee-2f92-496d-8bbc-342a159d7501
```

then follow the instructions and do

```bash
$ chmod 0600 /swapfile
```

To verify the change:

```bash
$ swapon --show
NAME      TYPE SIZE USED PRIO
/swapfile file   4G   2M   -2
```

Note that this command only ever allocates space for the space, and never takes it away. You can verify this:

```bash
$ sudo mkswap /swapfile
mkswap: /swapfile: warning: wiping old swap signature.
Setting up swapspace version 1, size = 20 GiB (21474832384 bytes)
no label, UUID=4d26bbeb-27ae-4ac2-b79f-b8520282ea72
$ swapon --show
NAME      TYPE SIZE USED PRIO
/swapfile file  20G 9.3M   -2
```

To actually reduce the swap size, you'll have to remove the swap file altogether and create a new one:

```bash
$ sudo rm /swapfile
$ sudo fallocate -l 8G /swapfile
```

Note that if you activate the swap immediately, you'll get

```bash
$ sudo swapon /swapfile 
swapon: /swapfile: insecure permissions 0644, 0600 suggested.
```

So instead, change the permissions first before doing that:

```bash
$ sudo chmod 0600 /swapfile
$ sudo swapon /swapfile
$ swapon --show        
NAME      TYPE SIZE USED PRIO
/swapfile file   8G 3.8M   -2
```
