# remark42

## Installation

If you can't start up on port 80, then follow [this answer](https://unix.stackexchange.com/a/244534) to kill what's on port 80:

```
$ sudo lsof -t -i tcp:80 -s tcp:listen | sudo xargs kill
```

### Let's Encrypt SSL

To get SSL set up, we try to follow the instructions [here](https://certbot.eff.org/instructions?ws=other&os=pip):

```
$ sudo apt update
$ sudo apt install python3 python3-venv libaugeas0
$ sudo python3 -m venv /opt/certbot/
$ sudo /opt/certbot/bin/pip install --upgrade pip
$ sudo /opt/certbot/bin/pip install certbot
$ sudo ln -s /opt/certbot/bin/certbot /usr/bin/certbot
$ sudo certbot certonly --standalone
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Enter email address (used for urgent renewal and security notices)
 (Enter 'c' to cancel): amos@zamm.dev

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Please read the Terms of Service at:
https://letsencrypt.org/documents/LE-SA-v1.4-April-3-2024.pdf
You must agree in order to register with the ACME server. Do you agree?
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es/(N)o: Y

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Would you be willing, once your first certificate is successfully issued, to
share your email address with the Electronic Frontier Foundation, a founding
partner of the Let's Encrypt project and the non-profit organization that
develops Certbot? We'd like to send you email about our work encrypting the web,
EFF news, campaigns, and ways to support digital freedom.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es/(N)o: N
Account registered.
Please enter the domain name(s) you would like on your certificate (comma and/or
space separated) (Enter 'c' to cancel): comments.zamm.dev
Requesting a certificate for comments.zamm.dev

Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/comments.zamm.dev/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/comments.zamm.dev/privkey.pem
This certificate expires on 2025-05-18.
These files will be updated when the certificate renews.

NEXT STEPS:
- The certificate will need to be renewed before it expires. Certbot can automatically renew the certificate in the background, but you may need to take steps to enable that functionality. See https://certbot.org/renewal-setup for instructions.

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
If you like Certbot, please consider supporting our work by:
 * Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
 * Donating to EFF:                    https://eff.org/donate-le
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -

```

It turns out there's [problems](https://forums.docker.com/t/docker-unable-to-mount-bind-letsencrypt-folder/138952) with mounting `/etc` from the host to the container due to the Docker VM's own `/etc` being used instead. So, we copy the files over to a directory that we can mount:

```bash
$ mkdir ssl
$ sudo cp -R /etc/letsencrypt/live/comments.zamm.dev/ ./ssl
```

We now edit the `docker-compose.yml` file to include the following:

```yaml
    environment:
      ...
      - AUTH_ANON=true
      - SSL_TYPE=static
      - SSL_CERT=/srv/ssl/fullchain.pem
      - SSL_KEY=/srv/ssl/privkey.pem
      ...
    volumes:
      ...
      - ./ssl:/srv/ssl:ro

```

#### Renewal

To renew the certificate, we can run the following command:

```bash
$ sudo certbot renew --force-renewal          
Saving debug log to /var/log/letsencrypt/letsencrypt.log

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Processing /etc/letsencrypt/renewal/comments.zamm.dev.conf
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Renewing an existing certificate for comments.zamm.dev
Failed to renew certificate comments.zamm.dev with error: Could not bind TCP port 80 because it is already in use by another process on this system (such as a web server). Please stop the program in question and then try again.

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
All renewals failed. The following certificates could not be renewed:
  /etc/letsencrypt/live/comments.zamm.dev/fullchain.pem (failure)
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
1 renew failure(s), 0 parse failure(s)
Ask for help or search for solutions at https://community.letsencrypt.org. See the logfile /var/log/letsencrypt/letsencrypt.log or re-run Certbot with -v for more details.
```

We try

```
$ sudo systemctl stop docker-remark42.service
$ sudo certbot renew --force-renewal         
Saving debug log to /var/log/letsencrypt/letsencrypt.log

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Processing /etc/letsencrypt/renewal/comments.zamm.dev.conf
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Renewing an existing certificate for comments.zamm.dev

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Congratulations, all renewals succeeded: 
  /etc/letsencrypt/live/comments.zamm.dev/fullchain.pem (success)
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
```

Note that the directory `/etc/letsencrypt/live/comments.zamm.dev/` now contains symlinks to the new keys:

```
$ ls -lh /etc/letsencrypt/live/comments.zamm.dev/    
total 4.0K
-rw-r--r-- 1 1001 1001 692 Feb 17 23:18 README
lrwxrwxrwx 1 root root  41 Apr 30 13:10 cert.pem -> ../../archive/comments.zamm.dev/cert2.pem
lrwxrwxrwx 1 root root  42 Apr 30 13:10 chain.pem -> ../../archive/comments.zamm.dev/chain2.pem
lrwxrwxrwx 1 root root  46 Apr 30 13:10 fullchain.pem -> ../../archive/comments.zamm.dev/fullchain2.pem
lrwxrwxrwx 1 root root  44 Apr 30 13:10 privkey.pem -> ../../archive/comments.zamm.dev/privkey2.pem
$ ls -lh /etc/letsencrypt/archive/comments.zamm.dev/
total 32K
-rw-r--r-- 1 root root 1.3K Feb 17 23:18 cert1.pem
-rw-r--r-- 1 root root 1.4K Apr 30 13:10 cert2.pem
-rw-r--r-- 1 root root 1.6K Feb 17 23:18 chain1.pem
-rw-r--r-- 1 root root 1.6K Apr 30 13:10 chain2.pem
-rw-r--r-- 1 root root 2.8K Feb 17 23:18 fullchain1.pem
-rw-r--r-- 1 root root 2.9K Apr 30 13:10 fullchain2.pem
-rw------- 1 root root  241 Feb 17 23:18 privkey1.pem
-rw------- 1 root root  241 Apr 30 13:10 privkey2.pem
```

As such, we must copy the symlinked files, and make `privkey.pem` readable by the user inside the Docker container:

```
$ rm -rf ssl
$ cp -Lr /etc/letsencrypt/live/comments.zamm.dev ./ssl
$ chmod a+r ssl/privkey.pem
$ sudo systemctl start docker-remark42.service
```

#### Upgrading `certbot`

If you get the error

```
$ sudo certbot renew --force-renewal
Saving debug log to /var/log/letsencrypt/letsencrypt.log

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Processing /etc/letsencrypt/renewal/comments.zamm.dev.conf
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Renewing an existing certificate for comments.zamm.dev
Failed to renew certificate comments.zamm.dev with error: urn:ietf:params:acme:error:malformed :: The request message was malformed :: No such authorization

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
All renewals failed. The following certificates could not be renewed:
  /etc/letsencrypt/live/comments.zamm.dev/fullchain.pem (failure)
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
1 renew failure(s), 0 parse failure(s)
Ask for help or search for solutions at https://community.letsencrypt.org. See the logfile /var/log/letsencrypt/letsencrypt.log or re-run Certbot with -v for more details.
```

then this may be because your [cerbot version is too old](https://community.letsencrypt.org/t/problem-with-renew-certificates-the-request-message-was-malformed-method-not-allowed/107889/2). Update it:

```bash
$ sudo snap install --classic certbot
certbot 4.1.1 from Certbot Project (certbot-eff✓) installed
```

### Telegram Auth and notification

We try following the Telegram instructions [here](https://remark42.com/docs/configuration/telegram/), and then edit `docker-compose.yml` to include the following:

```yaml
    environment:
      ...
      - TELEGRAM_TOKEN=...
      - AUTH_TELEGRAM=true
      - NOTIFY_TELEGRAM_CHAN=-100...
      - NOTIFY_ADMINS=telegram
      - NOTIFY_USERS=telegram
      ...
```

where you get the private channel ID by following [these instructions](https://neliosoftware.com/content/help/how-do-i-get-the-channel-id-in-telegram/) and going to [web.telegram.org](https://web.telegram.org/) and adding `-100` in front of the channel ID.

We get this error on the server when we try to login with Telegram:

```
remark42  | 2025/02/17 16:25:38.052 [DEBUG] {middleware/auth.go:75 middleware.(*Authenticator).Auth.(*Authenticator).auth.func1} auth failed, can't get token: token cookie was not presented: http: named cookie not present
```

It turns out this is a red herring; the real problem appears to not having set `NOTIFY_TELEGRAM_CHAN` at first.

### Facebook Auth

If you get an error such as "exchange failed", it is likely because you have [confused](https://stackoverflow.com/a/72679535) the app secret with the client token. The app secret looks like the last screenshot [here](https://help.keragon.com/hc/en-us/articles/21882995443218-How-to-find-your-OAuth-2-0-client-credentials-in-Facebook-for-conversions).

You will also need to add the valid callback URL in the settings for the login use case.

### Discord Auth

Discord Auth turns out to [not be available yet](https://github.com/umputun/remark42/issues/1880), despite what the documentation says.

### Admin

We make a comment with our logged-in user, and then do

```yaml
    environment:
      ...
      - ADMIN_SHARED_ID=telegram_...
      ...
```

The admin UI doesn't appear at first. It turns out you need to log out first and log back in.

### Running on server startup

We can try following [these instructions](https://stackoverflow.com/a/39493500) and editing `/etc/systemd/system/docker-remark42.service`:

```ini
[Unit]
Description=ZAMM Comments Service
Requires=docker.service
After=docker.service

[Service]
Restart=always
ExecStart=/usr/bin/docker compose -f /path/... up
ExecStop=/usr/bin/docker compose -f /path/... down

[Install]
WantedBy=default.target
```

and then

```bash
$ sudo systemctl enable docker-remark42.service
```

Initially, we get a problem when we try starting it manually:

```bash
$ sudo systemctl start docker-remark42.service
$ sudo systemctl status docker-remark42.service
× docker-remark42.service - ZAMM Comments Service
     Loaded: loaded (/etc/systemd/system/docker-remark42.service; enabled; vendor preset: enabled)
     Active: failed (Result: exit-code) since Tue 2025-02-18 06:40:08 +07; 12s ago
    Process: 1737 ExecStart=/usr/bin/docker-compose -f /root/Documents/zamm-dev-comments/docker-compose.yml up (code=exited, status=203/EXEC)
   Main PID: 1737 (code=exited, status=203/EXEC)
        CPU: 1ms

Feb 18 06:40:08 ubuntu-8gb-hil-1 systemd[1]: docker-remark42.service: Scheduled restart job, restart counter is at 5.
Feb 18 06:40:08 ubuntu-8gb-hil-1 systemd[1]: Stopped ZAMM Comments Service.
Feb 18 06:40:08 ubuntu-8gb-hil-1 systemd[1]: docker-remark42.service: Start request repeated too quickly.
Feb 18 06:40:08 ubuntu-8gb-hil-1 systemd[1]: docker-remark42.service: Failed with result 'exit-code'.
Feb 18 06:40:08 ubuntu-8gb-hil-1 systemd[1]: Failed to start ZAMM Comments Service.

```

Turns out the problem is `docker-compose` instead of `docker compose`. The problem is already fixed in the original example. As mentioned [here](https://serverfault.com/a/700956), we have to do

```bash
$ sudo systemctl daemon-reload
$ sudo systemctl start docker-remark42.service
```

to actually apply the fix.

Next, we disable the existing Apache server:

```bash
$ systemctl stop apache2.service
$ systemctl disable apache2.service
Synchronizing state of apache2.service with SysV service script with /lib/systemd/systemd-sysv-install.
Executing: /lib/systemd/systemd-sysv-install disable apache2
perl: warning: Setting locale failed.
perl: warning: Please check that your locale settings:
	LANGUAGE = (unset),
	LC_ALL = (unset),
	LC_CTYPE = "UTF-8",
	LANG = "en_US.UTF-8"
    are supported and installed on your system.
perl: warning: Falling back to a fallback locale ("en_US.UTF-8").
perl: warning: Setting locale failed.
perl: warning: Please check that your locale settings:
	LANGUAGE = (unset),
	LC_ALL = (unset),
	LC_CTYPE = "UTF-8",
	LANG = "en_US.UTF-8"
    are supported and installed on your system.
perl: warning: Falling back to a fallback locale ("en_US.UTF-8").
Removed /etc/systemd/system/multi-user.target.wants/apache2.service.
```

It still fails on startup because port 80 is blocked. We see what's up:

```
$ lsof -t -i:80                       
845
846
847
848
849
$ ps -p 845       
    PID TTY          TIME CMD
    845 ?        00:00:00 nginx
```

It finally runs on startup.
