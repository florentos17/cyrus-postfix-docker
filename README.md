# yadd/cyrus-imapd-postfix

Docker image based on last Debian stable release. It embeds:
 * postfix
 * cyrus-imapd (backports version)

## HOW TO START
```
docker build -t cyrus-local .
docker compose up
docker exec -it cyrus /bin/bash
```
After remote into the docker, run below steps to add dummy data:
### Start saslthdb
```
/etc/init.d/saslauthd start
```
### Create Mailbox user/password in SASLAUTHDB
```
echo 'createmailbox user.bob@localhost.com' | cyradm -u cyrus -w cyrus localhost
echo 'createmailbox user.alice@localhost.com' | cyradm -u cyrus -w cyrus localhost
echo 'secret' | saslpasswd2 -p -c bob
echo 'secret' | saslpasswd2 -p -c alice
```
### List user in SASLAUTHDB and verify password
```
$ sasldblistusers2
> bob@5cc0d9eab3c9: userPassword
> alice@5cc0d9eab3c9: userPassword
> cyrus@5cc0d9eab3c9: userPassword
```
```
$ testsaslauthd -u bob -p secret -f /var/run/saslauthd/mux
> 0: OK "Success."
```
### Sample telnet
To play with Cyrus IMAP (143):
`telnet localhost 143` then `A1 LOGIN bob secret`, `A2 LIST "" "*"`, `A3 SELECT INBOX`, etc

To play with Postfix SMTP (25):
```
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
220 localhost ESMTP Postfix (Debian/GNU)
MAIL FROM:<alice@example.com>
250 2.1.0 Ok
RCPT TO:<bob@example.com>
250 2.1.5 Ok
DATA
354 End data with <CR><LF>.<CR><LF>
Subject:cool subject
cool content
.
250 2.0.0 Ok: queued as A7EB0B042F1
QUIT
221 2.0.0 Bye
Connection closed by foreign host.
```
-> logs are at `/var/log/syslogd/mail/current`:
```
mail.info: Jan  7 14:34:55 postfix/smtpd[301]: A7EB0B042F1: client=unknown[172.18.0.1]
mail.info: Jan  7 14:35:09 postfix/cleanup[305]: A7EB0B042F1: message-id=<>
mail.info: Jan  7 14:35:09 postfix/qmgr[251]: A7EB0B042F1: from=<alice@example.com>, size=198, nrcpt=1 (queue active)
mail.info: Jan  7 14:35:09 cyrus/lmtpunix[308]: Delivered: <cmu-lmtpd-308-1736260509-0@5cc0d9eab3c9> to mailbox: user.bob
mail.notice: Jan  7 14:35:09 cyrus/lmtpunix[308]: USAGE bob user: 0.006167 sys: 0.009251
mail.info: Jan  7 14:35:09 postfix/lmtp[307]: A7EB0B042F1: to=<bob@example.com>, relay=localhost[/var/lib/cyrus/socket/lmtp], delay=25, delays=25/0.01/0.02/0.01, dsn=2.1.5, status=sent (250 2.1.5 Success SESSIONID=<cyrus-1736260509-308-2-14725385898710560125>)
mail.info: Jan  7 14:35:09 postfix/qmgr[251]: A7EB0B042F1: removed
mail.info: Jan  7 14:35:11 postfix/smtpd[301]: disconnect from unknown[172.18.0.1] mail=1 rcpt=1 data=1 quit=1 commands=4
```

### Sample CURL
```
curl -X POST -H "Content-Type: application/json" -H "Accept: application/json" --user bob:secret \
-d '{
  "using": [ "urn:ietf:params:jmap:core", "urn:ietf:params:jmap:mail" ],
  "methodCalls": [[ "Mailbox/get", {}, "c1" ]]
}' http://localhost:8008/jmap/ | jq
```
## Environment variables

Changing these variables has no effect when /etc/postfix is populated
_(after the first run if volume is kept)_.

Environment variables for Postfix _(with default value)_:

* For Cyrus-Imapd:
  * `CYRUS_PWD` = `""` _(if not set, cyradm isn't usable)_
  * `SASL_PWCHECK_METHOD` = `saslauthd auxprop`
  * `SASLDB` = `sasldb`
* For Postfix
  * `MAILNAME` = `mail.example.com`
  * `OTHER_DESTINATIONS` = `example.com mail.example.com`
  * `RELAY_HOST` = `""`
  * `MYNETWORKS` = `127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128`
  * `ROOT_ADDRESS` = `""`: _(address to which mail from root and postmaster will be sent)_

## Exposed ports

* Postfix _(depends on configuration)_:
  * 25
  * 465 _(not configured by default)_
  * 587 _(not configured by default)_
* Cyrus _(depends on configuration)_:
  * 110
  * 143
  * 993
  * 995 _(not configured by default)_
  * 2000 _(not configured by default)_
  * 8008
  * 8443 _(not configured by default)_

## Repository and bug reports

* Repository: [github.com/guimard/cyrus-postfix-docker](https://github.com/guimard/cyrus-postfix-docker)
* [Dockerfile](https://github.com/guimard/cyrus-postfix-docker/blob/master/Dockerfile)
* [Issues database](https://github.com/guimard/cyrus-postfix-docker/issues)

## Copyright and license

Copyright: Xavier (Yadd) Guimard <yadd@debian.org>.

License: [GNU General Public License v2.0](https://github.com/guimard/cyrus-postfix-docker/blob/master/LICENSE)
