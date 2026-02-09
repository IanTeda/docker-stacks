# Nextcloud Docker Compose

Environmental variables need to be set in `.env`. An example config can be found at `env.example`.

## Redis

To use Redis `/config/www/nextcloud/config/config.php needs to be updated to include:

```php
'redis' => [
    'host' => 'redis',
    'port' => 6379,
],
'memcache.local' => '\OC\Memcache\Redis',
'memcache.locking' => '\OC\Memcache\Redis',
```

## Reverse Proxy

To use a reverse proxy your domain needs to be added as a trusted_domain in `/config/www/nextcloud/config/config.php

```php
'trusted_domains' =>
[
    'nas.local:3443',
    'nextcloud.domain.com'
],
```

## Increase PHP Memory Usage

.../nextcloud/config/php/php-local.ini

```properties
memory_limit          = 2048M
upload_max_filesize   = 2048M
post_max_size         = 2048M

max_execution_time = 3600
max_input_time = 3600
```

## SMTP Settings

* Send Mode: SMTP
* Encryption: SSL/TLS
* From: nextcloud@mydomain.com
* Authentication method: Login
* Authentication Required: Yes
* Server address: smtp.gmail.com:465
* Credentials: gmailusername/app-password-in-settings


## Cloudflare

Use internal docker container names for FQN in cloudflare.

### References

* [NextCloud with CloudFlare Tunnels](https://dbt3ch.com/books/nextcloud-with-cloudflare-tunnels/page/nextcloud-with-cloudflare-tunnels)