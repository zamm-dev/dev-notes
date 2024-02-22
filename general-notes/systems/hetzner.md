# Hetzner storage box

When we first try to access our Hetzner storage box, we get the error

```bash
$ ssh u1234@u1234.your-storagebox.de   
The authenticity of host 'u1234.your-storagebox.de (2a01:4f9:4a:3853::2)' can't be established.
RSA key fingerprint is SHA256:EMlfI8GsRIfpVkoW1H2u0zYVpFGKkIMKHFZIRkf2ioI.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'u1234.your-storagebox.de' (RSA) to the list of known hosts.
u1234@u1234.your-storagebox.de's password: 
PTY allocation request failed on channel 0
shell request failed on channel 0
```

We find out from [this post](https://forum.cloudron.io/topic/5920/cannot-mount-hetzner-storage-box-for-backups-using-sshfs/2) that we should connect on port 23 instead:

```bash
$ ssh -p 23 u1234@u1234.your-storagebox.de
The authenticity of host '[u1234.your-storagebox.de]:23 ([2a01:4f9:4a:3853::2]:23)' can't be established.
ED25519 key fingerprint is SHA256:XqONwb1S0zuj5A1CDxpOSuD2hnAArV1A3wKY7Z3sdgM.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '[u1234.your-storagebox.de]:23' (ED25519) to the list of known hosts.
u1234@u1234.your-storagebox.de's password: 
+------------------------------------------------------------------+
| Welcome to your Storage Box.                                     |
|                                                                  |
| Please note that this is only a restricted shell environment and |
| therefore some shell features like pipes and redirects are not   |
| supported.                                                       |
+------------------------------------------------------------------+
u1234 /home >  
```

We open a new terminal to try to copy our SSH keys.

```bash
$ ssh-copy-id -p 23 u1234@u1234.your-storagebox.de
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/root/.ssh/id_rsa.pub"
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
u1234@u1234.your-storagebox.de's password: 

+----------------------------------------------------------------------------+
| ssh-copy-id is only supported with the "-s" argument.                      |
| Please try to rerun your command like this:                                |
|                                                                            |
|   ssh-copy-id -p 23 -s u1234@u1234.your-storagebox.de                  |
|                                                                            |
| Please note that the "-s" argument is only supported with OpenSSH 8.5+.    |
|                                                                            |
| You can find more details and a manual guide for the SSH key configuration |
| on this page:                                                              |
| https://docs.hetzner.com/robot/storage-box/backup-space-ssh-keys/          |
+----------------------------------------------------------------------------+
$ ssh-copy-id -s -p 23 u1234@u1234.your-storagebox.d
e
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/root/.ssh/id_rsa.pub"
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
u1234@u1234.your-storagebox.de's password: 
sftp> -get .ssh/authorized_keys /root/.ssh/ssh-copy-id.nnFfBQoKCH/authorized_keys
File "/home/.ssh/authorized_keys" not found.
sftp> -mkdir .ssh
remote mkdir "/home/.ssh": Failure
sftp> chmod 700 .ssh
sftp> put /root/.ssh/ssh-copy-id.nnFfBQoKCH/authorized_keys .ssh/authorized_keys
sftp> chmod 600 .ssh/authorized_keys

Number of key(s) added: 1

Now try logging into the machine, with:   "sftp -p '23' 'u1234@u1234.your-storagebox.de'"
and check to make sure that only the key(s) you wanted were added.
```
