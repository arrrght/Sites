---
title: Адресная книга на OpenLDAP для Oulook'а
layout: post
created_at: 24 июня 2013
date: '2013-06-14'
tags: ['ldap', 'outlook', 'linux', 'openldap']
kdpv: '/images/outlook.png'
intro: Установка адресной книги OpenLDAP - совместимой с MS Outlook
---

## Вступление
На предприятии есть файл с контактами сотрудников, заполняемый руками менеджера. Необходимо сделать из него **синхронизируемую** адресную книгу с адресами и явками для почтовой программы - MS Outlook, The Bat, Mozilla Thunderbird, и.тд

Очень хочется, чтобы она была совместима с "показать весь список" в Oulook, без фильтра.

- Распилим задачу на 4 части:
	- Настроить OpenLDAP
	- Забрать XLS с сервера
	- Преобразовать в перевариваемый формат, распарсить
	- Удалить старые контакты из книги, положить туда модули

## OpenLDAP
Вследствие фривольного обращения со стандартами компанией Microsoft, мы имеем несовместимость показа всех адресов в адресной книге. Поскольку гора к Муххамеду не идет, придется немного подрихтовать OpenLDAP.

С патчем отлично разобрался Victor Sudakov [здесь](http://victor-sudakov.livejournal.com/124269.html), за что ему огромное админское спасибо.

В качестве операционки на сервере, которая легко может быть виртуальной, скажем, с 8Gb диска и 64Mb памяти, стоит CSS - Calculate Linux Scratch(Gentoo, одним словом), для которой можно сделать свой патч легко и непринужденно.

Пачим.

Отступление для тех, кто не хочет заниматься кровавым патчингом - просто подключите мою тестовую репу(не факт, что я буду ее обновлять):
Добавляем overlay в `/etc/layman/layman.cfg`:
```
overlays  : http://www.gentoo.org/proj/en/overlays/repositories.xml
            https://raw.github.com/arrrght/openldap-outlook/master/overlay.xml
```
Активируем и синхронизируем:
	# layman -S
	# layman -a openldap-outlook

Пишем ключики в `/etc/portage/package.use`:

	net-nds/openldap experimental icu slp perl overlays ms-sssvlv

Устанавливаем OpenLDAP

	emerge openldap

Для тех кто любит все делать руками, вот патч(взят [здесь](http://victor-sudakov.livejournal.com/124269.html)):
```
--- ./openldap-2.4.33/servers/slapd/schema_prep.c.orig	2012-12-07 09:54:56.000000000 +0700
+++ ./openldap-2.4.33/servers/slapd/schema_prep.c	2012-12-07 09:58:10.000000000 +0700
@@ -908,6 +908,7 @@
 			"DESC 'RFC4519: common supertype of name attributes' "
 			"EQUALITY caseIgnoreMatch "
 			"SUBSTR caseIgnoreSubstringsMatch "
+			"ORDERING caseIgnoreOrderingMatch "
 			"SYNTAX 1.3.6.1.4.1.1466.115.121.1.15{32768} )",
	 		NULL, SLAP_AT_ABSTRACT,
	 		NULL, NULL,
```

### Настройка
Наиболее правильно написана настройка вот [здесь](http://sysadminblog.ru/ldap/2009/10/26/pravilnaya-incializaciya-openldap-servera-s-dinamicheskoy-konfiguraciey.html) - это как раз для тех кто решит серъёзно заняться LDAP-ом. В моем случае все проще - мне просто нужна адресная книга. Почти все я оставил по умочанию в файле `/etc/openldap/slapd.conf`. Вот он, целиком, безо всяких индексов, и вообще, много чего лишнего:
```
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

pidfile         /var/run/openldap/slapd.pid
argsfile        /var/run/openldap/slapd.args

modulepath      /usr/lib64/openldap/openldap
moduleload      back_passwd.so
moduleload      back_ldap.so
moduleload      sssvlv.so

database        hdb
overlay         sssvlv
suffix          "dc=org,dc=com"
checkpoint      32      30
rootdn          "cn=Manager,dc=org,dc=com"
rootpw          secret
directory       /var/lib/openldap-data
index   objectClass     eq

```
Маленькая начальая конфигурация, для того, чтобы зайти на LDAP через клиент `entry.ldif`:

```
dn: dc=org,dc=com
objectClass: dcObject
objectClass: organization
o: org.com
```

Записываем этот ldif в базу:
```
slapadd < entry.ldif
```

Заходим через клиент [LdapAdmin](http://www.ldapadmin.org) и создаем адресную книгу `ou=ab,dc=org,dc=com`.
Вот, это точка, откуда будут забирать адресную книгу клиенты.

## XLS
Проще всего забирать простой утилиткой `smbget`, поставляемой вместе с samba, парсить утилитой [xls2csv](http://www.wagner.pp.ru/~vitus/software/catdoc/). В итоге у меня получился вот такой маленький скрипт `getfile.sh`:
```
#!/bin/sh
SMBGET=/usr/bin/smbget
LDAPSEARCH=/usr/bin/ldapsearch
LDAPDELETE=/usr/bin/ldapdelete

rm "temp.xls"
${SMBGET} "smb://domain;user:password@192.168.1.1/Share/Telephones.xls" -o temp.xls
xls2csv -c~ -b\' temp.xls > temp.csv

# delete all from ou=ab,dc=org,dc=com
${LDAPSEARCH} -b "ou=ab,dc=org,dc=com" "(uid=*)" | grep dn: | awk '{print $2}' | ${LDAPDELETE} -Dcn=Manager,dc=org,dc=com -wsecret

```

Дальше перловый скрипт, преобразующий csv в нужный ldif файл, примерно такой(написано на скорую руку):
```
#!/usr/bin/perl
use strict;
use encoding 'utf8';

open my $fh, "<temp.csv" or die 'Can\'t open file';
open my $fout, ">temp.ldif" or die 'Can\'t write to file';

my $org = '';
my $orgNum = 0;
my $div = '';
my $userNum = 1000;
my $ouDcDc = 'ou=ab,dc=org,dc=com';

while (my $line = <$fh>) {
        my $isNowOrg = 0;
        my $email = '';

        chomp $line;
        my @f = split /~/, $line;

        if (!$org || $f[0]=~/\'/){
                $org = clean($f[0]);
                $org =~s/^(.+)\"(.+)\"$/$2/;
                $org = firstUp($org);
                $isNowOrg = 1;
                $orgNum++;
        }

        $email = clean($f[5]) if $f[5]=~/@/;
        $div = firstUp(clean($f[0])) unless $f[1] || $email;

        my $name = clean($f[1]);
        my $post = clean($f[0]);

        my $telNum = clean($f[4]);
        $telNum =clean($f[2]) if $orgNum==3;
        $telNum =~s/\D/ /g;
        $telNum =~s/^\D*//g;
        #print "#$telNum#\n";
        #$telNum = ~s/\W//g;

        my %names = givenName($name);
        if ($name && $post && $names{'firstName'}){
                ++$userNum;
                print $fout "dn: uid=user${userNum},${ouDcDc}\n";
                print $fout "uid: user${userNum}\n";
                print $fout "objectClass: posixAccount\nobjectClass: top\nobjectClass: inetOrgPerson\ngidNumber: 1000\n";
                print $fout "uidNumber: ${userNum}\n";
                print $fout "givenName: ", $names{'firstName'}, "\n";
                print $fout "initials: ", $names{'middleName'}, "\n";
                print $fout "sn: ", $names{'lastName'}, "\n";
                #print $fout "username: user${userNum}\n";
                print $fout "homeDirectory: home_dir\ngecos: gecos\nloginShell: log_shell\n";
                print $fout "telephoneNumber: ${telNum}\n" if $telNum;
                print $fout "physicalDeliveryOfficeName: ${div}\n";
                print $fout "ou: ${div}\n";
                print $fout "o: $org\n";
                #print $fout "organizationName: ACME\n";
                print $fout "title: $post\n";
                print $fout "mail: $email\n";
                print $fout "cn: ", $names{'shortName'},"\n";
                print $fout "\n";
        #       print "ORG: $org, DIV: $div, POST: $post, NAME: $name, EMAIL: $email\n";
        }
}
close $fout;
close $fh;

sub givenName(){
        my $name = shift;
        my @s=[];
        my %names = {};
        @s = split ' ', $name;
        $names{'firstName'} = $s[1];
        $names{'lastName'} = $s[0];
        $names{'middleName'} = $s[2];
        $names{'shortName'} = $s[0] .' '. substr($s[1],0,1) .'.'. substr($s[2],0,1) .'.';
        return %names;
}

sub clean(){
        my $name = shift;
        $name =~s/^\'//;
        $name =~s/\"\"/\"/g;
        $name =~s/^\"//;
        $name =~s/\"$//;
        $name =~s/\ *$//g;
        $name =~s/\ +/\ /g;
        return $name;
}

sub firstUp(){
        my $name = shift;
        $name = lc($name);
        return ucfirst($name);
}
```

После чего а выходе должен получится файл `temp.ldif`, уже который мы скармиливаем команде

```
ldapadd -v -Dcn=Manager,dc=org,dc=com -wsecret < ./temp.ldif
```

Вот как-то так с серверной частью. На клиенте, в Outlook, делаем:

```
Настройка учетный записей -->
  Адресные книги -->
  Создать -->
    LDAP-->
    IP: наш IP
      Не требуется вход на сервер
      Другие настройки -->
        Поиск -->
          База поиска: ou=ab,dc=org,dc=com
          Просмотр: Включить просмотр (требуется серверная поддержка)
```

Все. Проверяем, прописываем в крон, оптимизируем.
PS: Для того, чтобы отображалась организация в списке адресов Oulook, надо отпатчить файл `/etc/openldap/schema/core.schema`:
Добавим 'company' - отдельное спасибо Microsoft, что она не обращает внимания на стандартную запись ```o```, строка 120
```
attributetype ( 2.5.4.10 NAME ( 'company' 'o' 'organizationName' )
        DESC 'RFC2256: organization this object belongs to'
        SUP name )
```