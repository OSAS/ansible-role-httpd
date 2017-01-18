Ansible module used by OSAS to manage Apache httpd.

[![Build Status](https://travis-ci.org/OSAS/ansible-role-httpd.svg?branch=master)](https://travis-ci.org/OSAS/ansible-role-httpd)

# Adding a Vhost

The role is quite flexible and support all kind of setup. Since it serve as the basis for others roles
who may be different, the default setting do almost nothing, and in turn require almost nothing.

The domain used for the vhost come from `website_domain`. If not given, it will take `ansible_fqdn` by default.

One of the most basic usage is serving static web pages. For that, the variable `document_root` will need to
be set. If none is provided, no document root will be set. You can use the `document_root_group` parameter
if you need this directory to be owner by a specific group (or it defaults to 'root').

# TLS support

This role can be used to setup and enable TLS support in differents ways, using
either letsencrypt, freeipa or a direct ssl certificate drop. The configuration
by default will set HSTS, and use the set of cipher from https://wiki.mozilla.org/Security/Server_Side_TLS
SSL v3 and V2 are disabled

A option to force the TLS version of the website exist, just set `force_tls: True` to enable it.

## Letsencrypt

To enable the letsencypt support, just set `use_letsencrypt: True` with your role.
It will take care of installing needed packages, and set the config for the HTTP challenge.

It will use a automatically constructed email address based on domain name. If the server domain
cannot receive email, please see the `mail_domain` variable to override it.

## FreeIPA / dogtag

[FreeIPA](http://freeipa.org) come with [DogTag](http://pki.fedoraproject.org/wiki/PKI_Main_Page),
a certificate management system. You can request certificates signed by the internal CA and have them
renewed automatically using the variable `use_freeipa`. This is mostly used for internal services.

## Self managed certificates

For people who prefer to use certificates signed manually by a CA, the option `use_tls` permit to
enable SSL/TLS without managing the certificate. The key should be placed on /etc/pki/tls/private/$DOMAIN.key
the certificate in /etc/pki/tls/certs/$DOMAIN.crt and the CA certificate in /etc/pki/tls/certs/$DOMAIN.ca.crt.

# Others settings
## ModSecurity

A administrator can decide to enable ModSecurity by setting `use_mod_sec: True`. This will enable a regular
non tweaked ModSecurity install, with full protection. In order to test that it break nothing, the module can be
turned in detection mode only with `mod_sec_detection_only: True`.

## Tor Hidden service

The role can enable a tor hidden service for http and https with the option `use_hidden_service: True`. It will
automatically enable the hidden service and add the .onion alias to the httpd configuration. No specific
hardening for anonymisation purpose is done, the goal is mostly to enable regular website to be served over
tor to let the choice to people.

## Log retention period

Logs are rotated on a regular basis, and kept for some amount on time on the server. The default is 8 weeks, but it
can be tweaked with the `log_retention_week` variable

## Redirection

If the variable `redirect` is set, the vhost will redirect to the new domain name. This is mostly done for
supporting compatibility domain after moving a project, or to support multiple domain names. This argument
is aimed for the simpler case, handling various things like letsencrypt automatically.

One can also use the variable `redirects`, as a array of url and redirection. This is
much more powerful, but requires to becareful as there is fewer verifications built-in.
It can also use the Redirectmatch directive by using `match: True`.

```
$ cat deploy_web.yml
- hosts: web
  roles:
  - role: httpd
    website_domain: foo.example.org
    redirects:
    - src: "/blog/about"
      target: "/about"
    - src: "^/feed/(.*)"
      target: "http://blog.example.org/feed/$1"
      match: True
```

## Aliases

The support of the Aliases directive is enabled with `aliases`, which is a list of url and path. The
role take care of adding `Require all granted` for the path.

```
$ cat deploy_web.yml
- hosts: web
  roles:
  - role: httpd
    website_domain: alamut.example.org
    aliases:
    - url: "/blog/"
      path: "/var/www/blog/"
```


## Password protection

In order to deploy website before launch, we traditionnaly protect them with a simple user/password
using a .htpasswd file. The 2 variables `website_user` and `website_password` can be set and will automatically
set the password protection.

Removing the password protection requires to remove the 2 variables and remove the /etc/httpd/$DOMAIN.htpasswd
file created.

## Server aliases

Some vhosts can correspond to multiple vhosts. While most of the time, this can be achieved with a proper redirect,
a alias is sometime a prefered way to handle that. So one can use the `server_aliases` variable for that like this:

```
$ cat deploy_web.yml
- hosts: web
  roles:
  - role: httpd
    website_domain: foo.example.org
    server_aliases:
    - www.example.org
    - web.example.org
```

But usually, for cleaner URL, a redirect is preferred.

## Enable mod_speling

Administrators wishing to use mod_speling can juse use `use_mod_speling: True` in the definition
of the vhost.

# Extend the role

In order to compose more complex roles by combining (and using depends), the installed configuration also
support to include part of the configuration.

Due to the way ansible work, variable set on the first role will be inherited by the httpd role if set on the role
level. Thus, this permit to keep the same variable name, like this:

```
$ cat roles/piwik/meta/main.yml
---
dependencies:
- { role: mariadb }
- { role: httpd, document_root: /var/www/piwik/, use_freeipa: True }

$ cat deploy_piwik.yml
- hosts: piwik
  roles:
  - role: piwik
    website_domain: piwik.example.org
```

However, some variable name are a bit generic (like use_tls), so beware of this as it can produes weird side effects.

In order to let a role extend the httpd configuration, a role can drop files ending in .conf in /etc/httpd/conf.d/$DOMAIN.conf.d/.
The file will be included for TLS and non TLS vhost for now, which might cause some issues. This is planned to be fixed later
to be able to support WSGI cleanly.
