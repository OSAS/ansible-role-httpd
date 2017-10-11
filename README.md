Ansible module used by OSAS to manage Apache httpd.

[![Build Status](https://travis-ci.org/OSAS/ansible-role-httpd.svg?branch=master)](https://travis-ci.org/OSAS/ansible-role-httpd)

# Role Logic

The role is quite flexible and support all kind of setup. Since it serve as the basis for others roles
who may be different, the default setting do almost nothing, and in turn require almost nothing.

A role can depend on this one and ensure the webserver is properly installed. There are no settings to
provide. A defaut vhost using the canonical name of the machine (as defined by `ansible_fqdn`) is installed
by default if not already defined (see _Adding a Vhost_ chapter).

In order to allow defining settings  various levels, this role provides extra entrypoints which are
intended to be used with the `include_role` module. Main entrypoints are:
* server: setup server-wide settings; there is no settings in the core but additional features may add some
* vhost: define a vhost (see _Adding a Vhost_ chapter)
Some additional features may add extra entrypoints which would be described in their own chapter.

If default server-wide settings are fine in your project, then you don't need to call the `server` entrypoint.
If you call it multiple times, previous settings would be overriden.

# Adding a Vhost

This feature use the `vhost` entrypoint.

The domain used for the vhost come from `website_domain`. If not given, it will take `ansible_fqdn` by default.
If you call it multiple times with the same `website_domain`, previous settings would be overriden.

One of the most basic usage is serving static web pages. For that, the variable `document_root` will need to
be set. If none is provided, no document root will be set. You can use the `document_root_group` parameter
if you need this directory to be owner by a specific group (or it defaults to 'root').

```
- hosts: web
  tasks:
    - include_role:
        name: httpd
        tasks_from: vhost
      vars:
        website_domain: www.example.com
        document_root: /var/www/www.example.com
```

# TLS support

This feature use the `vhost` entrypoint.

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

# Others vhost settings

These features use the `vhost` entrypoint.

## ModSecurity

A administrator can decide to enable ModSecurity by setting `use_mod_sec: True`. This will enable a regular
non tweaked ModSecurity install, with full protection. In order to test that it break nothing, the module can be
turned in detection mode only with `mod_sec_detection_only: True`.

## Tor Hidden service

The role can enable a tor hidden service for http and https with the option `use_hidden_service: True`. It will
automatically enable the hidden service and add the .onion alias to the httpd configuration. No specific
hardening for anonymisation purpose is done, the goal is mostly to enable regular website to be served over
tor to let the choice to people.

This feature requires a `tor` role in charge of the Tor base installation. OSAS made [a role](https://gitlab.com/osas/ansible-role-tor) you can use alongside this one.

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

Removing the password protection requires to remove the 2 variables and remove the htpasswd file created
(`/etc/httpd/$DOMAIN.htpasswd` on RedHat systems, `/etc/apache2/$DOMAIN.htpasswd` on Debian).

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

# Python support

Running Python scripts is supported using WSGI.

## Script instance

This feature use the `wsgi` entrypoint.

The vhost defined in `website_domain` would have this script associated to its root. The vhost then needs
to be defined (see _Adding a Vhost_ chapter).

`wsgi_instance` needs to define a short and unique service name (allowed characters whould match the `name`
setting of the WSGIDaemonProcess directive). `wsgi_script` needs to be set to the script absolute path.
You also need to set the user and group the script process will run as, using `wsgi_user`; the group must
have the same name, and both must already exist. Last you need to set the home of the application (working
directory) using `wsgi_home`.

The number of threads allocated to the Python instance is defined using `wsgi_threads` and defaults to 5.

```
- hosts: web
  tasks:
    - include_role:
        name: httpd
        tasks_from: wsgi
      vars:
        website_domain: www.example.com
        use_wsgi: True
        wsgi_instance: myservice
        wsgi_script: "/usr/share/myservice/www/myservice.wsgi"
        wsgi_user: "myuser"
        wsgi_home: /var/www/www.example.com/app
```

## Server configuration

This feature use the `server` entrypoint.

Usually applications do not need to register their own signal handlers, but some may need it, like Mailman 3;
In this case you can set `wsgi_allow_app_signals` to True, which will ensure WSGIRestrictSignal is off.

```
- hosts: web
  tasks:
    - include_role:
        name: httpd
        tasks_from: server
        use_wsgi: True
        wsgi_allow_app_signals: True
```

# Extend the role

In order to compose more complex roles by combining (and using depends), the installed configuration also
support to include part of the configuration. Nevertheless the easiest way to extend the logic nowadays is
to create a higher level role and use `include_role` in its tasks. You can find a real life example in the
[mailing-lists-server role](https://gitlab.com/osas/ansible-role-mailing-lists-server).

In order to let a role extend the httpd configuration, a role can drop files ending in .conf in `/etc/httpd/conf.d/$DOMAIN.conf.d/` on RedHat systems or `/etc/apache2/site-enabled/$DOMAIN.conf.d/` on Debian.
The file will be included for TLS and non TLS vhost for now, which might cause some issues. This is planned to be fixed later
to be able to support WSGI cleanly.

To ease extension without recalculating path and other variables logic, a role extending the vhost configuration can use the `` entrypoint like this:
```
- name: Define httpd Variables
  include_role:
    name: httpd
    tasks_from: vhost_vars
```
It can then, for example, install a specific configuration fragment like this:
```
- name: Install Mailman 2 Web UI Configuration
  copy:
    src: mailman2.conf
    dest: "{{ _vhost_confdir }}/"
  notify: verify config and restart httpd
```

