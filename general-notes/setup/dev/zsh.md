# Setting up ZSH

ZSH is a popular alternative shell to bash. To set it up, first install it via `apt`:

```bash
$ sudo apt install zsh
[sudo] password for amos:
Reading package lists... Done
Building dependency tree... Done
...
Setting up zsh-common (5.8.1-1) ...
Setting up zsh (5.8.1-1) ...
Processing triggers for man-db (2.10.2-1) ...
```

Then start up zsh:

```bash
$ zsh
This is the Z Shell configuration function for new users,
zsh-newuser-install.
You are seeing this message because you have no zsh startup files
(the files .zshenv, .zprofile, .zshrc, .zlogin in the directory
~).  This function can help you with a few settings that should
make your use of the shell easier.

You can:

(q)  Quit and do nothing.  The function will be run again next time.

(0)  Exit, creating the file ~/.zshrc containing just a comment.
     That will prevent this function being run again.

(1)  Continue to the main menu.

(2)  Populate your ~/.zshrc with the configuration recommended
     by the system administrator and exit (you will need to edit
     the file by hand, if so desired).

--- Type one of the keys in parentheses ---
```

Sometimes this screen doesn't show. If it does, type in 2.

```bash
/home/amos/.zshrc:15: scalar parameter HISTFILE created globally in function zsh-newuser-install
```

Then install Oh My ZSH:

```bash
$ sh -c "$(wget https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh -O -)"
--2023-07-17 23:21:10--  https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh
Resolving raw.github.com (raw.github.com)... 185.199.111.133, 185.199.110.133, 185.199.108.133, ...
Connecting to raw.github.com (raw.github.com)|185.199.111.133|:443... connected.
...

Time to change your default shell to zsh:
Do you want to change your default shell to zsh? [Y/n] y
Changing your shell to /usr/bin/zsh...
[sudo] password for amos:
Shell successfully changed to '/usr/bin/zsh'.

         __                                     __
  ____  / /_     ____ ___  __  __   ____  _____/ /_
 / __ \/ __ \   / __ `__ \/ / / /  /_  / / ___/ __ \
/ /_/ / / / /  / / / / / / /_/ /    / /_(__  ) / / /
\____/_/ /_/  /_/ /_/ /_/\__, /    /___/____/_/ /_/
                        /____/                       ....is now installed!


Before you scream Oh My Zsh! look over the `.zshrc` file to select plugins, themes, and options.

• Follow us on Twitter: https://twitter.com/ohmyzsh
• Join our Discord community: https://discord.gg/ohmyzsh
• Get stickers, t-shirts, coffee mugs and more: https://shop.planetargon.com/collections/oh-my-zsh
```

If `wget` doesn't exist yet (for example, because you're on a Mac), you can do

```bash
$ brew install wget
```

or use the curl version of the command instead:

```bash
$ sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

If the default shell prompt doesn't pop up, or you say no to it, you can change it manually later:

```bash
$ which zsh
/usr/bin/zsh
$ chsh
Changing the login shell for root
Enter the new value, or press ENTER for the default
        Login Shell [/usr/bin/zsh]: /usr/bin/zsh
```

Then, install the autocomplete plugin:

```bash
$ git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
Cloning into '/home/amos/.oh-my-zsh/custom/plugins/zsh-autosuggestions'...
remote: Enumerating objects: 2460, done.
remote: Counting objects: 100% (75/75), done.
remote: Compressing objects: 100% (52/52), done.
remote: Total 2460 (delta 36), reused 49 (delta 21), pack-reused 2385
Receiving objects: 100% (2460/2460), 571.83 KiB | 227.00 KiB/s, done.
Resolving deltas: 100% (1567/1567), done.
```

And edit your `~/.zshrc`, which should have lines like this:

```
# Which plugins would you like to load?
# Standard plugins can be found in $ZSH/plugins/
# Custom plugins may be added to $ZSH_CUSTOM/plugins/
# Example format: plugins=(rails git textmate ruby lighthouse)
# Add wisely, as too many plugins slow down shell startup.
plugins=(git)
```

into this:

```
# Which plugins would you like to load?
# Standard plugins can be found in $ZSH/plugins/
# Custom plugins may be added to $ZSH_CUSTOM/plugins/
# Example format: plugins=(rails git textmate ruby lighthouse)
# Add wisely, as too many plugins slow down shell startup.
plugins=(git zsh-autosuggestions)
```

Source the config file after editing:

```bash
$ . ~/.zshrc
```

Now you should see ZSH with autocomplete as the default shell! If instead you see

```
[oh-my-zsh] plugin 'zsh-autosuggestions' not found
```

then we can follow the instructions [here](https://github.com/zsh-users/zsh-autosuggestions/blob/master/INSTALL.md):

```bash
$ git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
```

## Corrupted history file

If you see

```
zsh: corrupt history file /home/amos/.zsh_history
```

then follow [these instructions](https://shapeshed.com/zsh-corrupt-history-file/) to fix it:

```bash
$ mv ~/.zsh_history ~/.zsh_history_bad
$ strings ~/.zsh_history_bad > ~/.zsh_history
$ fc -R ~/.zsh_history
$ rm ~/.zsh_history_bad
```

## Updates

Sometimes you'll see it say

```
[oh-my-zsh] Would you like to update? [Y/n] 
```

You can update it by saying `Y`. Then it will output something like

```
You can see the changelog with `omz changelog`
         __                                     __   
  ____  / /_     ____ ___  __  __   ____  _____/ /_  
 / __ \/ __ \   / __ `__ \/ / / /  /_  / / ___/ __ \ 
/ /_/ / / / /  / / / / / / /_/ /    / /_(__  ) / / / 
\____/_/ /_/  /_/ /_/ /_/\__, /    /___/____/_/ /_/  
                        /____/                       

Hooray! Oh My Zsh has been updated!

To keep up with the latest news and updates, follow us on X: @ohmyzsh
Want to get involved in the community? Join our Discord: Discord server
Get your Oh My Zsh swag at: Planet Argon Shop
```
