# Перенос Otrs 6.0 с Centos на AlmaLinux 9.9

## 1. Установка базовых компонентов на новом серере OTRS
На новом сервере:
```
sudo dnf install -y epel-release
sudo dnf config-manager --set-enabled crb
sudo dnf install -y httpd mod_ssl mod_http2 mod_perl
sudo dnf install -y perl-Archive-Tar perl-Archive-Zip perl-Authen-SASL perl-Crypt-Eksblowfish perl-Crypt-SSLeay perl-DBD-MySQL perl-DBD-Pg perl-Digest-SHA perl-LDAP perl-libwww-perl perl-LWP-Protocol-https perl-Mail-IMAPClient perl-Net-DNS perl-Template-Toolkit perl-Text-CSV_XS perl-XML-LibXML perl-XML-LibXSLT perl-XML-Parser perl-App-cpanminus perl-YAML-LibYAML perl-JSON-XS perl-TimeDate perl-DateTime perl-Encode-HanExtra
sudo dnf install postfix postfix-pcre -y
sudo dnf install -y mariadb-server
```

## 2. Настройка правил Firewall на новом сервере OTRS
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

## 4. Создание пользователя OTRS
```
sudo mkdir /opt/otrs
sudo useradd -u 500 -g 48 -c 'OTRS System User' -d /opt/otrs -s /bin/bash -M otrs
```

## 5. Остановка OTRS на старом сервере
```
sudo systemctl stop httpd
sudo crontab -u otrs -l >> ~/crontab_backup_otrs
sudo -i -u otrs
/opt/otrs/bin/Cron.sh stop
/opt/otrs/bin/otrs.Daemon.pl stop
```

## 6. Убедись что OTRS остановлен
```
/opt/otrs/bin/otrs.Daemon.pl status
sudo ps -u otrs
sudo lsof +D /opt/otrs 2>/dev/null
```

## 7. Установка и настройка MariaDB
sudo vim /etc/my.cnf.d/ignore-lost-found.cnf
```diff
[mysqld]
ignore-db-dir=lost+found
```
sudo vim /etc/my.cnf.d/zotrs.cnf
```diff
[mysql]
max_allowed_packet=256M
[mysqldump]
max_allowed_packet=256M
[mysqld]
innodb_log_file_size = 256M
max_allowed_packet=256M
```

```
sudo systemctl enable --now mariadb
sudo systemctl restart mariadb
```

```
sudo mysql_secure_installation
Enter current password for root: Enter
Switch to unix_socket authentication: Y
Change the root password?: N
Всё остальное: Y
```
```
sudo mysql -u root
CREATE DATABASE otrs CHARACTER SET utf8 COLLATE utf8_general_ci;
CREATE USER 'otrs'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON *.* TO 'otrs'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

## 8. Дамп Базы данных на старом сервере
```
Убедись что свободно минимум 180GB
df -h | awk 'NR==1 || /backup/' 
sudo mysqldump -u root otrs -r /backup/otrs_backup.sql
```

## 9. Перенос дампа Базы данных
```
scp root@helpdesk-test.example.com:/backup/otrs_backup.sql root@helpdesk-test1.example.com:/var/lib/mysql/
tmux new -s import (tmux attach -t import)
sudo mysql -u root otrs < /var/lib/mysql/otrs_backup.sql
```

## 10. Архивирование папки OTRS на старом сервере
```
sudo tar -czpf /tmp/otrs_backup.tar.gz -C /opt otrs
```

## 11. Перенос папки OTRS на новый сервер
```
scp root@helpdesk-test.example.com:/tmp/otrs_backup.tar.gz root@helpdesk-test1.example.com:/opt/otrs_backup.tar.gz
sudo tar -xzpf /tmp/otrs_backup.tar.gz -C /opt
```

## 12. Настройка HTTPD
```
sudo cp /opt/otrs/scripts/apache2-httpd.include.conf /etc/httpd/conf.d/zzz_otrs.conf
sudo chmod 644 /etc/httpd/conf.d/zzz_otrs.conf
```
sudo vim /etc/httpd/conf.d/security.conf
```
ServerTokens Prod
ServerSignature Off
```
sudo vim /etc/httpd/conf.d/helpdesk.conf
```
<VirtualHost *:80>
    ServerName helpdesk-test.example.com
    Redirect permanent /  https://helpdesk-test.example.com/
</VirtualHost>

<VirtualHost *:443>
    ServerName helpdesk-test.ascon.ru
    Redirect permanent /index.html https://helpdesk-test.example.com/otrs/customer.pl
    SSLEngine On
    SSLCertificateKeyFile   /etc/pki/tls/private/helpdesk-test.example.com.key
    SSLCertificateFile      /etc/pki/tls/certs/helpdesk-test.example.com.crt
    RemoteIPHeader X-Real-IP
</VirtualHost>
```
sudo vim /etc/httpd/conf.d/ssl.conf
```diff
- # <VirtualHost _default_:443>
- # Закоментировать весь блок
- # </VirtualHost>
```
### 12.2 Перенести SSL Сертификат
### 12.3 Проверить конфигурацию HTTPD
```
httpd -M
httpd -S
```

## 13 Проверка готовности запуска OTRS
```
sudo -i -u otrs
/opt/otrs/bin/otrs.CheckModules.pl
/opt/otrs/bin/otrs.Console.pl Admin::Package::List
/opt/otrs/bin/otrs.Console.pl Maint::Database::Check
```
### 13.1 Проверка доступности LDAP
```
telnet ldap.example.com 389
telnet ldap.example.com 636
```
### 13.2 Проверка доступности IMAP и SMTP
```
telnet imap.example.com 993

telnet smtp.example.com 25
telnet smtp.example.com 465
telnet smtp.example.com 587

```
### 13.3 Нужно ли ? (Чистим старый кэш и пересобираем конфиг)
```
sudo -i -u otrs
/opt/otrs/bin/otrs.Console.pl Maint::Cache::Delete
/opt/otrs/bin/otrs.Console.pl Maint::Config::Rebuild
```

## 14 Запуск OTRS
```
sudo -i -u otrs
/opt/otrs/bin/Cron.sh start
/opt/otrs/bin/otrs.Daemon.pl start
/opt/otrs/bin/otrs.Daemon.pl status
```

## 15 Финальная проверка работы OTRS
```
Работает вход по LDAP
Приходят новые заявки
Приходит автоответ о создании заявки
Агент может ответить на заявку 
Работают эскалации
```


