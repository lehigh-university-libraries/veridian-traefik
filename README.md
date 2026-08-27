# Veridian bot protection with traefik + captcha-protect + cloudflare turnstile

In August 2026 Lehigh University Libraries' self-hosted Veridian instance was being overwhelmed by bot web traffic.

To keep the service stable, we removed the default apache config and instead have web traffic served via [traefik](https://github.com/traefik/traefik) with the [captcha-protect](https://github.com/libops/captcha-protect) plugin enabled. Apache is still used to serve CGI-BIN, but moved to `127.0.0.1:8080` and traefik proxies to that port.

This was the first service where we installed traefik outside of docker. In our case, it was on a Debian Trixie VM hosted on-prem. This general pattern could potentially be used for other non-docker services.

## Clone this repo

Some config files might be easier to just copy into place from this repo. So check out this repo

```
cd ~/
git clone https://github.com/lehigh-university-libraries/veridian-traefik
```

## Install traefik on the VM

Put the traefik confs in place and edit the hostnames in `/etc/traefik/dynamic/*.yml` to match your hostnames

```
sudo cp ~/veridian-traefik/etc/traefik /etc/traefik
```

Create an unprivileged user to run traefik as

```
sudo useradd \
  --system \
  --home /var/lib/traefik \
  --shell /usr/sbin/nologin \
  traefik
sudo mkdir -p /var/lib/traefik /var/log/traefik
sudo chown -R traefik:traefik /var/lib/traefik /var/log/traefik
sudo chown -R root:traefik /etc/traefik
sudo chmod 750 /etc/traefik
sudo chmod 640 /etc/traefik/traefik.yml
sudo chmod 640 /etc/traefik/traefik.env
```

Put the binary and systemd unit in place (may need to change the arch from linux / amd64 depending on your CPU)
```
curl -sL \
   https://github.com/traefik/traefik/releases/download/v3.7.12/traefik_v3.7.12_linux_amd64.tar.gz \
    -o traefik_v3.7.12_linux_amd64.tar.gz
echo 'c3b2634fc5421626641e2f4256108c08078a9b2662f649d770af78e5e5d729a2  traefik_v3.7.12_linux_amd64.tar.gz' \
  | sha256sum -c -
tar -zxvf traefik_v3.7.12_linux_amd64.tar.gz
sudo mv traefik /usr/local/bin/
sudo chown root:traefik /usr/local/bin/traefik
rm traefik_v3.7.12_linux_amd64.tar.gz
sudo cp ~/veridian-traefik/etc/systemd/system/traefik.service /etc/systemd/system/traefik.service
```

## Install captcha-protect traefik plugin

Instead of using traefik's plugin downloader, just clone [the captcha-protect repo](https://github.com/libops/captcha-protect) on disk and reference it in the traefik config. This avoids needing to download the plugin every time traefik service starts.

```
sudo mkdir -p /var/lib/traefik/plugins-local/src/github.com/libops
sudo git clone https://github.com/libops/captcha-protect.git /var/lib/traefik/plugins-local/src/github.com/libops/captcha-protect
```

## Create a Cloudflare Turnstile widget

Go to https://dash.cloudflare.com, sign up or login, and create a turnstile widget for your Veridian site. Populate the site/secret keys in [/etc/traefik/traefik.env](./etc/traefik/traefik.env).

## Configure TLS

Create the fullchain traefik needs to serve the TLS cert (this may be an optional step for your setup)

```
sudo usermod -aG ssl-cert traefik
sudo sh -c '
  cat \
    /etc/ssl/certs/lib.lehigh.edu.crt \
    /etc/ssl/certs/gd_bundle-g2-g1.crt \
    > /etc/ssl/certs/lib.lehigh.edu-fullchain.crt
'
```

Make sure the traefik user can read the private key

```
sudo chgrp traefik /etc/ssl/private/lib.lehigh.edu.key
sudo -u traefik cat /etc/ssl/private/lib.lehigh.edu.key >/dev/null && echo OK
```

## Configure apache

Turn off the TLS conf for veridian.

```
sudo a2dissite veridian-ssl.conf
```

Put the apache conf in place (being sure to update `ServerName` and you have veridian site enabled in apache)
```
sudo cp /etc/apache2/sites-enabled/veridian.conf /etc/apache2/sites-enabled/veridian.bak
sudo cp ~/veridian-traefik/etc/apache2/sites-enabled/veridian.conf /etc/apache2/sites-enabled/veridian.conf
```

Ensure `/etc/apache/ports.conf` only listens on `127.0.0.1:8080`

```
sudo cp /etc/apache2/ports.conf /etc/apache2/ports.bak
sudo cp ~/veridian-traefik/etc/apache2/ports.conf /etc/apache2/ports.conf
```

## Style challenge page

When human visitors first visit your site, they will be challenged with a Cloudflare Turnstile widget. You may want to style the webpage that presents this challenge. To do so, you can edit the challenge go template HTML file. We created ours by copying the HTML from our Veridian site homepage and add the two HTML sections in [this repo's challenge.tmpl.html](./etc/traefik/challenge.tmpl.html)

```
    <!-- Start captcha-protect-->
    ...
    <!-- End captcha-protect-->
```

```
sudo vim.tiny /etc/traefik/challenge.tmpl.html
```

## Go live

Once you have everything configured, restart apache to have it stop binding port 80/443 and start traefik.

```
sudo systemctl restart apache2
sudo systemctl start traefik
sudo systemctl enable traefik
```
