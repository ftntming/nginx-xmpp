# IMPORTANT NOTICE

Since [FastMail no longer have a XMPP service](https://blog.fastmail.com/2015/11/16/shutting-down-our-xmpp-chat-service/), I have no reason to continue updating this to the latest nginx. You are welcome to take this on yourself, or ask me to help out and I'll see what I can do.

# nginx-xmpp

This is [nginx](https://nginx.org/en/) with [XMPP](https://tools.ietf.org/html/rfc6120) proxy support. It adds XMPP to the list of protocols supported by the mail module, allowing nginx to do TLS and auth termination for XMPP servers.

This is a *fork* of nginx to add XMPP to its [mail module](https://nginx.org/en/docs/mail/ngx_mail_core_module.html). It is _not_ part of the more common nginx http module. If you're unfamiliar with the mail module I've written a quick intro below.

## Building

Compile nginx for mail as you normally would. XMPP support will be included alongside IMAP, POP3 and SMTP as normal. You probably also need TLS support. Something like this should get you started:

```bash
$ ./auto/configure --with-mail --with-mail_ssl_module
$ make -j9
$ make install
```

See the [nginx building docs](https://nginx.org/en/docs/configure.html) for more useful switches.

## Configuring

nginx-xmpp adds support for the `xmpp` protocol to nginx-mail. Apart from that, configuration should be the same as any other mail config. This should be enough to get you started:

```nginx
worker_processes 1;

error_log /var/log/nginx/error.log info;

events {
    worker_connections  1024;
}

mail {
    server_name       xmpp.example.com;
    auth_http         localhost:9000/cgi-bin/nginxauth.cgi;

    ssl_prefer_server_ciphers on;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers EECDH+ECDSA+AESGCM:EECDH+aRSA+AESGCM:EECDH+ECDSA+SHA256:EECDH+aRSA+SHA256:EECDH:EDH+aRSA:HIGH:DES-CBC3-SHA:!aNULL:!eNULL:!LOW:!DES:!MD5:!EXP:!PSK:!SRP:!DSS:!RC4:!SEED;

    ssl_certificate      /etc/ssl/ssl.crt/xmpp.example.com.crt;
    ssl_certificate_key  /etc/ssl/ssl.key/xmpp.example.com.key;

    server {
      listen   5222;
      protocol xmpp;
      starttls on;
      proxy on;
    }

    server {
      listen   5223;
      protocol xmpp;
      ssl on;
      proxy on;
    }
}
```

The `xmpp_auth` and `xmpp_client_buffer` directives also exist. These operation analogously to the [`imap_auth`](https://nginx.org/en/docs/mail/ngx_mail_imap_module.html#imap_auth) and [`imap_client_buffer`](https://nginx.org/en/docs/mail/ngx_mail_imap_module.html#imap_client_buffer) directives.

See [the mail module docs](https://nginx.org/en/docs/mail/ngx_mail_core_module.html) for more configuration options.

## The nginx mail module: a primer

If you haven't seen the nginx mail module before, here's a quick intro.

It's job is to act as an IMAP, POP3 and SMTP server, accepting connections and proxying them through to "real" IMAP/POP3/SMTP servers on the backend.

nginx itself speaks just enough of these protocols to do the initial greeting and auth handshake. Once auth is completed, nginx calls out to an [external auth service](https://nginx.org/en/docs/mail/ngx_mail_auth_http_module.html) to validate the credentials. The auth service returns a yes/no response and, if the auth succeeds, an IP, username, password, etc for the backend server.

nginx then connects to the backend service, authenticates on behalf of the user using the credentials supplied by the auth service and, once completed, returns an "auth success" message to the client. All data in both directions is then proxied by nginx's normal connection proxying machinery.

(incidentally, nginx also has as [stream module](https://nginx.org/en/docs/stream/ngx_stream_core_module.html), which is like the mail module but without the application protocol and auth handshake support. You'd use that to let nginx act as a "dumb" load balancer and TLS terminator.)

See [this howto](https://www.nginx.com/resources/admin-guide/mail-proxy/) for more information on the mail module and in particular, how to construct your auth service.

## TODO

- [ ] federation/S2S
- [ ] multiple certificate support (like SNI but using domain from stream header)
- [ ] XEP-0198 session resumption?
- [ ] CRAM-MD5?
- [ ] ...

## Further reading

https://robn.io/nginx-xmpp/ has the history and rationale for this project.

## Shameless advertising

[FastMail](https://www.fastmail.com/) employs me to do crazy things like this :)
