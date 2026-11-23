# Перенос Otrs 6.0 с Centos на AlmaLinux

## 1. Подготовка новой системы
На новом сервере:
```
sudo dnf install -y epel-release
sudo dnf config-manager --set-enabled crb
sudo dnf install -y httpd mod_ssl mod_http2 mod_perl
sudo dnf install -y perl-Archive-Tar perl-Archive-Zip perl-Authen-SASL perl-Crypt-Eksblowfish perl-Crypt-SSLeay perl-DBD-MySQL perl-DBD-Pg perl-Digest-SHA perl-LDAP perl-libwww-perl perl-LWP-Protocol-https perl-Mail-IMAPClient perl-Net-DNS perl-Template-Toolkit perl-Text-CSV_XS perl-XML-LibXML perl-XML-LibXSLT perl-XML-Parser perl-App-cpanminus perl-YAML-LibYAML perl-JSON-XS perl-TimeDate perl-DateTime perl-Encode-HanExtra
sudo dnf install postfix postfix-pcre -y
```

## 2. Правила FireWall
На новом сервере:
```
iptables -A INPUT -p tcp -m tcp --dport 22 -j ACCEPT
iptables -A INPUT -p tcp -m tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp -m tcp --dport 443 -j ACCEPT
```

## 3. Перенос Postfix MTA
На новом сервере:
### Файл /etc/postfix/generic со старого сервера
### Файл /etc/postfix/relocated со старого сервера
### Файл /etc/postfix/canonical_sender со старого сервера
### Файл /etc/postfix/sasl_passwd со старого сервера
### Файл /etc/postfix/main.cf самое важное
Добавить
```
smtp_tls_security_level = may
relayhost = [smtp.example.com]:587
smtp_sasl_auth_enable = yes
smtp_sasl_security_options = noanonymous
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
sender_canonical_maps = hash:/etc/postfix/canonical_sender
mailbox_size_limit = 0
message_size_limit = 0
```
Изменить
```diff
- inet_protocols = all
+ inet_protocols = ipv4
```
### Перечитать проверить перезапустить Postfix
```
sudo postmap /etc/postfix/relocated /etc/postfix/generic /etc/postfix/canonical_sender /etc/postfix/sasl_passwd
sudo postfix check
sudo systemctl enable --now postfix
sudo systemctl restart postfix
```
