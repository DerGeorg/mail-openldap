# sealeo/mail-openldap

Main features:
* use postfix, dovecot
* manage multidomain emails
* connects to an existing LDAP (see: [OpenLDAP](https://hub.docker.com/r/sealeo/openldap/))
* RoundCube Webmail

Mail: postfix, dovecot - [Docker Hub](https://hub.docker.com/r/sealeo/mail-openldap/) 

# Installation
## Docker

### Mountpoints
- /vmail: all mailboxes
- /ssl/...: the SSL certificates for SMTP and IMAP (see SSL section)
- /etc/opendkim: storage for opendkim keys

### Ports
- 25 and 587 are for SMTP
- 993 is for IMAP
- 80 (optional) is for the Webmail

### Environment variables
- TZ: the timezone
- MAIL_DOMAIN: the domain of the mail server
- LDAP_DOMAIN_BASE: the URL to access the LDAP
- LDAP_PASSWORD: the password for LDAP
- DKIM_KEY_SIZE: DKIM key size in bits

#### For RoundCube Webmail
- RC_ENABLED: any non-empty value to enable the webmail
- RC_NAME: the display name of the webmail
- RC_SUPPORT_URL: the URL displayed when support is required
- RC_DB_HOST: the URL of the database server
- RC_DATABASE: the database name
- RC_DB_USER: the roundcube user (must have permissions on the database)
- RC_DB_PASSWORD: the roundcube password corresponding to the user

### Examples
#### Docker CLI
```bash
docker run -d --name mail \
 -v /home/mail/mailboxes:/vmail \
 -v /home/mail/ssl/smtp.mydomain.com:/ssl/smtp.mydomain.com:ro \
 -v /home/mail/ssl/imap.mydomain.com:/ssl/imap.mydomain.com:ro \
 -v /home/mail/dkim:/etc/opendkim \
 -p 25:25 -p 80:80 -p 587:587 -p 993:993 \
 --link ldap --link db
 -e TZ=Etc/UTC -e MAIL_DOMAIN=mydomain.com \
 -e LDAP_DOMAIN_BASE=ldapdomain.com -e LDAP_PASSWORD=password \
 -e DKIM_KEY_SIZE=2048 \
 -e RC_ENABLED=1 RC_DB_HOST=db -e RC_DB_PASSWORD=password \
 sealeo/mail-openldap
```

#### docker-compose.yml
```yaml
version: '3'
services:
  mail:
    image: sealeo/mail-openldap
    volumes:
    - /home/mail/mailboxes:/vmail
    - /home/mail/ssl/smtp.mydomain.com:/ssl/smtp.mydomain.com:ro
    - /home/mail/ssl/imap.mydomain.com:/ssl/imap.mydomain.com:ro
		- /home/mail/dkim:/etc/opendkim
    ports:
    - "25:25"
		- 80:80
    - 587:587
    - 993:993
    external_links:
    - ldap
		- db
    environment:
		- TZ=Etc/UTC
		- MAIL_DOMAIN=mydomain.com
    - LDAP_DOMAIN_BASE=mydomain.com
    - LDAP_PASSWORD=password
		- DKIM_KEY_SIZE=2048
		- RC_ENABLED=1
		- RC_DB_HOST=db
		- RC_DB_PASSWORD=password
```

## DNS
### Minimal configuration
Minimal DNS zone configuration for the `MAIL_DOMAIN` (e.g. *mydomain.com* as above)
```
smtp 300 IN A x.x.x.x
imap 300 IN A x.x.x.x
mail 10800 IN A x.x.x.x
@ 10800 IN MX 10 mail.mydomain.com.
```
x.x.x.x is the IP address of your mail server.

### More advanced configuration
#### SPF
See: [SPF on Wikipedia](https://en.wikipedia.org/wiki/Sender_Policy_Framework)
See: [SPF Wizard](https://www.spfwizard.net/)

Example of possible configuration:
```
mydomain.com. IN TXT "v=spf1 mx a ptr ip4:x.x.x.x ~all"
```
x.x.x.x is the IP address of your mail server.

#### DKIM
See: [DKIM on Wikipedia](https://en.wikipedia.org/wiki/DomainKeys_Identified_Mail)
See the **Add domain** section that explains what is required in DNS for DKIM

#### DMARC
See: [DMARC on Wikipedia](https://en.wikipedia.org/wiki/DMARC)

Example of possible configuration:
```
_dmarc IN TXT "v=DMARC1; p=none"
```

## SSL
You need to provide SSL certificates for SMTPS and IMAPS in `/ssl`, in directories `smtp.mydomain.com` and `imap.mydomain.com` respectively.
Minimal files are `fullchain.pem` and `privkey.pem`. Any other file will be ignored.
You must ensure that these certificates are up to date.

Examples below are with Let's Encrypt but they must be easy to adapt to anything.
Note: this configuration is to be done on the **host**

### First installation
if you have any service listening on port 80: (example: nginx)
```bash
certbot certonly --pre-hook 'systemctl stop nginx' --post-hook 'systemctl start nginx' --standalone --agree-tos --rsa-key-size 4096 -d smtp.mydomain.com
certbot certonly --pre-hook 'systemctl stop nginx' --post-hook 'systemctl start nginx' --standalone --agree-tos --rsa-key-size 4096 -d imap.mydomain.com
```
**else**
```bash
certbot certonly --standalone --agree-tos --rsa-key-size 4096 -d smtp.mydomain.com
certbot certonly --standalone --agree-tos --rsa-key-size 4096 -d imap.mydomain.com
```

You now have certificates in `/etc/letsencrypt/live/`
To copy these for the container, you can run the two lines below.
Note: `/container/ssl` must be replaced by the mountpoint corresponding to inner `/ssl`
```bash
cp -TLrf /etc/letsencrypt/live/smtp.mydomain.com /container/ssl/smtp.mydomain.com
cp -TLrf /etc/letsencrypt/live/imap.mydomain.com /container/ssl/imap.mydomain.com
```
### Keep certificates up to date
This section will describe a way to keep easily up to date the certificates

#### Requirements
```bash
apt update
apt install -y incron
```

#### Setup
```bash
cat>/etc/incron.d/certs.mail.mydomain.com<<EOF
/etc/letsencrypt/live/smtp.mydomain.com/fullchain.pem IN_CLOSE_WRITE cp -LTrf /etc/letsencrypt/live/smtp.mydomain.com /data/containers/email/ssl/smtp.mydomain.com
/etc/letsencrypt/live/imap.mydomain.com/fullchain.pem IN_CLOSE_WRITE cp -LTrf /etc/letsencrypt/live/imap.mydomain.com /data/containers/email/ssl/imap.mydomain.com
EOF
```

# Usage
Scripts are available in the container to add a new domain, email address or alias

## Add domain
```bash
docker exec -it mail add_domain
Domain? mydomain2.com
```

It will eventually show the required additional DNS zone configuration to enable DKIM for the new domain.
You will find it in you `/home/mail/dkim/keys/mydomain2.com/mail.txt` also, if you mounted like described above.

## Add alias
```bash
docker exec -it mail add_alias
Domain? mydomain.com
User? admins
Name? Administrator list

Recipient list? CTRL+D to finish
alice@mydomain.com
bob@mydomain.com
```

## Add email
```bash
docker exec -it mail add_email
Domain? mydomain.com
User? bob
Password?
```

## (Re)generate DKIM for a domain
```bash
docker exec -it mail gen_dkim
Domain? mydomain.com
```
See **Add domain**
