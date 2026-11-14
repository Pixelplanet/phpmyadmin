# phpMyAdmin image of this fork

This folder builds a Docker image of this fork that behaves like the
[official phpMyAdmin image](https://github.com/phpmyadmin/docker) (Apache
variant): same PHP extensions and settings, same environment variables
(`PMA_HOST`, `PMA_HOSTS`, `PMA_ABSOLUTE_URI`, `UPLOAD_LIMIT`, ...), same
`/etc/phpmyadmin/config.user.inc.php` and `/etc/phpmyadmin/conf.d/` hooks, same
entrypoint. See the official README for the full list of options.

It lives on the `fork/docker` branch only, so it never becomes part of a pull
request to phpMyAdmin.

## Contents

- `Dockerfile`: packages the committed code with phpMyAdmin's own release
  script, then installs the result like the official image does.
- `config.inc.php`, `helpers.php`, `docker-entrypoint.sh`: unchanged copies
  from [phpmyadmin/docker](https://github.com/phpmyadmin/docker) at commit
  `4f7341b`, licensed under the GNU GPL version 3 (`LICENSE-phpmyadmin-docker`).

Differences to the official image:

- phpMyAdmin is built from this repository instead of being downloaded from
  files.phpmyadmin.net.
- phpMyAdmin 6 serves only its `public/` folder, so the code lives in
  `/var/www/phpmyadmin` and Apache's document root is
  `/var/www/phpmyadmin/public` (the official 5.2 image uses `/var/www/html`).
- The examples are kept in `/var/www/phpmyadmin/examples` (not served).

## Getting the image

Every push to `fork/docker` builds and publishes it with GitHub Actions
(`.github/workflows/fork-docker-image.yml`):

```bash
docker pull ghcr.io/pixelplanet/phpmyadmin:latest
```

Each build is also tagged with the short commit id (`sha-1234567`).

To build it yourself from the repository root (takes a while, the release
script runs the test suite):

```bash
docker build -f docker/Dockerfile -t phpmyadmin-fork .
```

Only committed files are used; uncommitted changes in the checkout are ignored.

## Single sign-on behind an authenticating proxy

To log users in from the identity header of a reverse proxy (see the
"Signon behind an authenticating reverse proxy" section of the
documentation), mount a copy of `examples/signon-proxy-header.php` with your
proxy address and user mapping, and switch the servers to signon:

```php
<?php
// /etc/phpmyadmin/config.user.inc.php
for ($n = 1; isset($cfg['Servers'][$n]); $n++) {
    $cfg['Servers'][$n]['auth_type'] = 'signon';
    $cfg['Servers'][$n]['SignonScript'] = '/etc/phpmyadmin/signon.php';
    $cfg['Servers'][$n]['LogoutURL'] = 'https://pma.example.com/oauth2/sign_out';
}
$cfg['ShowChgPassword'] = false;
```

```yaml
services:
  phpmyadmin:
    image: ghcr.io/pixelplanet/phpmyadmin:latest
    environment:
      PMA_HOSTS: db1,db2
      PMA_VERBOSES: Orders,Reporting
      PMA_ABSOLUTE_URI: https://pma.example.com/
    volumes:
      - ./config.user.inc.php:/etc/phpmyadmin/config.user.inc.php:ro
      - ./signon.php:/etc/phpmyadmin/signon.php:ro
```

Do not publish the port of this container: let only the proxy reach it, over
a shared Docker network, and list only the proxy's address in
`SIGNON_TRUSTED_PROXIES`. The image enables Apache's `remoteip` module but does
not configure it, so phpMyAdmin sees the proxy's address as the client.
