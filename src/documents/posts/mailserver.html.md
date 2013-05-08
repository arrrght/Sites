---
title: Linux, RAID, LVM, почта и LDAP (5-в-одном)
layout: post
created_at: 21 мая 2011
tags: ['post']
intro: Быстро-быстро делаем почтовый сервер на 4 винтах и управляем им c LDAP-клиента виндовой машины
---
## Вступление
- Имеем:
	- полудохлое железо на CeleronD
	- 4 полудохлых винта на 80Гб


- Что надо:
	- почтовый сервер
	- pop3 и imap
	- управляемый, не входя (лишний раз) в консоль, почтой с Win*
	- общая адресная книга на предприятии
	- в случае выхода из строя одного из винчестеров система не должна падать
	- и все это надо быстро
	- и чтоб не тормозило


- Делаем:
	- программный райд на 4 винчестера
	- LVM (почта + барахло)
	- posfix/dovecot
	- вся аутентификация через ldap
	- на Win* ставим [LdapAdmin](http://www.ldapadmin.org) (или любой [другой](http://jxplorer.org) клиент)


## Подготовка
Грузимся с CSS - [Calculate Linux Scratch](http://www.calculate-linux.ru/main/ru/download) - тот же 100% Gentoo, вид сбоку. Устанавливается быстрее, большая часть пакетов - бинарные

Имея 4 винчестера на 80Гб, бьем их на 4 части:
```
4Мб -  [EF02] Bios boot partition
20Гб - [FD00] root RAID1 (4)
2Гб -  [FD00] swap RAID1 (4)
52Гб - [FD00] LVM RAID10 (2+2)
```

Делаем все по-модному, с GPT - разбиваем утилитой gdisk.
Первый раздел делаем, чтобы grub не ругался на отсутствие "bios partition". В теории более чем достаточно 1Мб, но я решил не жадничать
([GdiskBooting](http://www.rodsbooks.com/gdisk/booting.html) | [BiosBootPartition](http://en.wikipedia.org/wiki/BIOS_Boot_partition))

Чистим первые несколько секторов: ```dd if=/dev/zero of=/dev/sda bs=1M count=1```

Разбиваем один винчестер через gdisk, дальше копируем разбивку на остальные:
```
sgdisk -R=/dev/sdb /dev/sda
sgdisk -R=/dev/sdc /dev/sda
sgdisk -R=/dev/sdd /dev/sda
```
Диск разбили, делаем 2 софтовых райда (```md0 - root, md1 - swap```):
```
mdadm --create --verbose /dev/md0 --level=1 --raid-devices=4 /dev/sda2 /dev/sdb2 /dev/sdc2 /dev/sdd2
mdadm --create --verbose /dev/md1 --level=1 --raid-devices=4 /dev/sda3 /dev/sdb3 /dev/sdc3 /dev/sdd3
```
Наблюдаем за созданием (минут 10):```watch cat /proc/mdstat```

```/dev/md2 (LVM)``` сделаем потом, чтобы сейчас не ждать синхронизации - скажем, после установки и обновления системы

## Система
Устанавливаем Calculate Linux Scratch:

```cl-install -d /dev/md0 -d /dev/md1:swap```

Вводим парроль рута и ждем около 5 минут чтобы система установилась.
После перезагрузки ставим grub2 на все винты (или проверям чтобы он установился):
```
grub-install /dev/sda
grub-install /dev/sdb
grub-install /dev/sdc
grub-install /dev/sdd
```
Обновляем систему:
```
eix-sync
emerge -uND @world
dispach-conf
```

Ставим необходимые пакеты:
```
emerge vixie-cron logrotate syslog-ng
dispach-conf
rc-update add lvm boot
```

Явно вышло новое ядро - **перезагружаемся**, смотрим, что все хорошо и ставим 10 раид на остаток(потом попилим lvm).
Это надолго, около часа, не меньше.
```
mdadm --create --verbose /dev/md1 --level=10 --raid-devices=4 /dev/sda3 /dev/sdb3 /dev/sdc3 /dev/sdd3
watch cat /proc/mdstat
```

Итак, наша система готова. И вот здесь надо уже приступать к тестам. Лучше сделать это сейчас, чтобы понять/вспомнить что делать в случае аварии. **Очень** рекомендую потратить на это время.

Выключам комп, отключаем один из винтов, включаемся-загружаемся, наблюдаем что система работает, ```cat /proc/mdstat``` говорит что у него 3 винта, ему плохо, но он работает.

Выключаем машину, подключаем обратно винт, загружаемся, смотрим что происходит, принимает меры, гуглим.

**PS**: Можно сказать что я часто перезагружаюсь. Но очень часто лучше заметить ошибку сразу, после перезагрузки (не включили службу, не добили диск, забыли установить grub или не обратили внимания на его ошибки, итд). А в итоге, когда сервер работает уже полгода, и все давно забыли что это за железка и что она вообще делает, - пропадает питание, и сервер останавливается на буте, ожидая что кто-нибудь загрузит наконец-то ядро... Сервер - автономная железка, которая должна уметь включаться/выключаться без вмешательства админа.

## Почта

Ставим зависимости в ```/etc/portage/package.use/custom```
```
mail-mta/postfix dovecot-sasl ldap
net-mail/dovecot ldap
net-nds/openldap ssl
```

Устанавливаем
```
emerge postfix dovecot openldap
dispatch-conf
```


### LDAP
Настраиваем LDAP ```/etc/openldap/slapd.conf```
```
## Включаем схемы:
include         /etc/openldap/schema/core.schema
include         /etc/openldap/schema/cosine.schema

include         /etc/openldap/schema/inetorgperson.schema
include         /etc/openldap/schema/java.schema

include         /etc/openldap/schema/openldap.schema
include         /etc/openldap/schema/nis.schema
include         /etc/openldap/schema/corba.schema
include         /etc/openldap/schema/dyngroup.schema
include         /etc/openldap/schema/misc.schema
include         /etc/openldap/schema/pmi.schema

## Включаем аутентификацию
moduleload      back_passwd.so
moduleload      back_ldap.so

## Пишем корень и супергероя
database        hdb
suffix          "dc=company, dc=local"
rootdn          "cn=Manager, dc=company, dc=local"
rootpw          super-hero-passwd

## Расписываем индексы (чтобы быстрее искало и в логах не ругалось)
index   objectClass     eq
index   uid             eq
index   cn              eq,sub
index   mail            eq,sub
index   sn              eq,sub
index   givenName       eq,sub
index   displayName     eq,sub
```

Прописываем в автозагрузку и запускаем
```
/etc/init.d/slapd restart
rc-update add slapd default
```

Сразу проверяем, можем ли мы цепанутся к LDAP с винды: прописываем ip хоста, base ```dc=company,dc=local```,
юзера ```cn=Manager, dc=company, dc=local``` и пароль ```super-hero-passwd```.

Создаем раздел Users ```objectClass=organizationUnit, ou=Users```
и там -  пару учеток - test1 и test2 ```objectClass=posixAccount,intOrgPerson,top uid=test1@company.local```
ставим им пароль (правый клик, setPassword[CRYPT] в ldapAdmin)

Сразу делаем пользвателя который будет отвечать за нашу почту:
```
useradd vmail
```

### Dovecot

Для начала создаем самоподписанный сертификат - переписваем из ```/usr/share/doc/dovecot-2.1.15/``` себе в ```/root``` пару файлов - ```mkcert.sh``` и ```dovecot-openssl.cnf```

Правим ```dovecot-openssl.cnf``` запускаем ```sh ./mkcert.sh``` - устаналиваем самоподписанные сертификаты

Правим ```/etc/dovecot/conf.d/10-auth.conf```, раскоментируем ```!include auth-ldap.conf.ext```, остальные инклуды ремарим

Правим ```/etc/dovecot/conf.d/auth-ldap.conf.ext```:
```
passdb {
  driver = ldap
  args = /etc/dovecot/dovecot-ldap.conf.ext
}
userdb {
  driver = static
  args = uid=vmail gid=vmail home=/var/vmail/%d/%n
}
```

Правим последний файл - ```/etc/dovecot/dovecot-ldap.conf.ext```:
```
base = ou=Users,dc=company,dc=local
user_attrs = %n,%Dd=user,home=/var/vmail/%d/%n/.maildir
user_filter = (&(objectClass=posixAccount)(uid=%u))
pass_attrs = mail=user,userPassword=password,userdb_home=/var/vmail/%d/%n/.maildir
pass_filter = (&(objectClass=posixAccount)(uid=%u))
```

Все файлы просты и понятны. Конфиг сразу настраивается как мультидоменный, на всякий случай, т.е. юзер будет аутентифицироваться как test1@company.local, почта будет хранится в ```/var/vmail/company.local/test1```

С Dovecot закончили. Отрезаем кусок, скажем в 30Гб, у LVM, и монтируем его в /var/vmail
```
vgcreate vg /dev/md2
lvcreate -nvmail -L30G vg
mkfs.ext4 /dev/vg/vmail
mkdir /dev/vmail
mount /dev/vg/vmail /var/vmail
```

Не забываем прописать в ```/etc/fstab```:
```/dev/vg/vmail /var/vmail ext4 noatime 0 1```

Запускаем, прописываем в автозагрузку и проверяем
```
rc-update add dovecot default
/etc/init.d/dovecot restart
```

Заходим в почтовый клиент, проверям, что подключилось нормально - писем нет, аутентификация прошла, в логах ```/var/log/messages``` ошибок нет.

### Postfix

```/etc/postfix/master.cf```: добавляем одну строку
```
dovecot   unix  -       n       n       -       -       pipe
        flags=DRhu user=vmail:vmail argv=/usr/libexec/dovecot/deliver -f ${sender} -d ${recipient}
```

и в файл ```/etc/postfix/main.cf```:
```
message_size_limit=99999999
mailbox_size_limit=99999999
smtpd_use_tls=yes
smtpd_tls_cert_file=/etc/ssl/certs/dovecot.pem
smtpd_tls_key_file=/etc/ssl/private/dovecot.pem

virtual_mailbox_maps     = ldap:/etc/postfix/ldap_virtual_mailbox_maps.cf
virtual_alias_maps       = hash:/etc/mail/aliases
dovecot_destination_recipient_limit = 1

local_transport = dovecot
virtual_gid_maps = static:1000
virtual_uid_maps = static:1000
virtual_mailbox_base = /var/vmail
virtual_mailbox_limit = 1000000000
virtual_transport = dovecot

myhostname = mailbox.company.local
myorigin = $mydomain
mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain
home_mailbox = .maildir/
```

На будущее: сделать алиазы тоже через LDAP, сейчас надо править ```/etc/mail/aliases``` и делать ```newaliases```.
Это происходит достаточно редко, поэтому я особо не заморачивался.

Опять же, запускаем, прописываем в автозагрузку:
```
rc-update add postfix default
/etc/init.d/postfix restart
```

Ставим пакет для простоты проверки: ```emerge mailx```, и проверям работу:
```
date | mail test1
```

В это время смотрим в логи ```tail -f /var/log/messages```, проверям что создался ```/var/vmail/company.local/test1/.maildir```, разбух на пару кило. Идем в почтовый клиент и проверяем почту от test1@company.local на test2@company.local.

Все хорошо.

Проверям как сервер справится с нагрузкой - гуглим [multimail](http://download.cnet.com/MultiMail/3000-2382_4-75325323.htm). Ставим для начала пару потоков простым письмом, если все хорошо - ставим 10 потоков по 999 писем с большими вложениями - смотрим в это время на очередь к почте ```postqueue -p | tail -1```, загрузку системы ```atop,iotop,htop```, и одновременно смотрим на любимый почтовый клиент.

Все живы, все работает. Удаляем все каталоги в ```/var/vmail/company.local/*``` (чтобы не перегружать почтовый клиент спамом)

**Перезагружаемся**, не заходя в консоль убеждаемся что почта ходит, создаем третьего пользователя test3 через ldap, проверям что все у него хорошо. Выставляем сервер в инет, отправляем самому себе почту с личного почтового ящика и обратно, проверям на открытый релей.

Ну и поскольку это все ldap  - настраиваем общую адресную книгу в почтовых клиентах, а это уже задание на дом ::))