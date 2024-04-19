---
title: "TLS in MySQL oder MariaDB"
date: 2023-04-10T21:35:41+02:00
draft: false
tags: ["mysql", "TLS"]
---

Jede sensible Information in einer Datenbank wurde vermutlich vorher über das Netzwerk übertragen. Kommunizieren Anwendungen über ein Netzwerk mit einer Datenbank, sollte geprüft werden, ob es sinnvoll ist, diesen Traffic via TLS abzusichern. Dies kann auch bei internen Netzen durchaus sinnvoll sein. Denn unverschlüsselter Datenverkehr kann nicht nur einfach ausgelesen, sondern auch manipuliert werden.

Zunächst prüfen wir, ob die installierte MariaDB oder MySQL Version auch TLS unterstützt. In den meisten Fällen wurde das Paket vermutlich mit einer entsprechenden TLS-Bibliothek gebaut. Häufig ist das OpenSSL oder YaSSL. Wer seine Datenbank ohne TLS-Unterstützung kompiliert hat, muss diese leider nochmal erneut kompilieren.

```
MariaDB [(none)]> show global variables like '%ssl%';

+---------------------+-----------------------------+
| Variable_name | Value |
+---------------------+-----------------------------+
| have_openssl | YES |
| have_ssl | YES |
| version_ssl_library | OpenSSL 1.1.1k 25 Mar 2021 |
+---------------------+-----------------------------+
```

In diesem Beispiel wurde MariaDB unter Debian 11 mit OpenSSL kompiliert.

Je nach genutzter Version hat MySQL vielleicht schon während der Installation passende TLS-Zertifikate im Datenverzeichnis (datadir, default ist `/var/lib/mysql`) hinterlegt. Allerdings sollte man auch bei Datenbanken auf einen privaten Schlüssel mit mindestens 2048 Bit achten. Wer schon einen Schlüssel im Datenverzeichnis vorfindet, kann mit folgendem OpenSSL-Befehl die Länge des Schlüssels prüfen:

`openssl rsa -in secret.key -text -noout | grep "Private-Key"`

Wer keine Zertifikate vorfindet, kann diese mittels OpenSSL erstellen.

```
openssl genrsa 2048 > ca-key.pem
openssl req -new -x509 -nodes -days 3000 -key ca-key.pem > ca-cert.pem

openssl req -newkey rsa:2048 -days 3000 -nodes -keyout server-key-pkcs8.pem > server-req.pem
openssl rsa -in server-key-pkcs8.pem -out server-key.pem

openssl x509 -req -in server-req.pem -days 3000 -CA ca-cert.pem -CAkey ca-key.pem -set_serial 01 > server-cert.pem
```

Nach diesen Befehlen hat man alles Nötige, damit die Datenbank eine TLS-Verbindung anbieten kann. Die nachfolgenden Befehle sind nur notwendig, wenn sich die (MySQL) Benutzer mit einem Client am Server authentifizieren sollen. Achten Sie darauf, bei den Server- und Client-Zertifikaten einen unterschiedlichen CN (Common Name) zu verwenden. Ansonsten wird sich ein Benutzer mit Zertifikat nicht anmelden können.

```
openssl req -newkey rsa:2048 -days 1095 -nodes -keyout client-key-pkcs8.pem > client-req.pem
openssl rsa -in client-key-pkcs8.pem -out client-key.pem

openssl x509 -req -in client-req.pem -days 1095 -CA ca-cert.pem -CAkey ca-key.pem -set_serial 01 > client-cert.pem
```

Ab bestimmten Versionen hat MySQL ein Auto-Discover für TLS-Zertifikate im Datenverzeichnis. Wenn MySQL während des Starts die \*.pem Zertifikate, z. B. server-cert.pem und server-key.pem, vorfindet, bietet MySQL direkt TLS-Verbindungen an. Natürlich kann man für die Zertifikate auch andere Namen verwenden. Allerdings muss dann in der Konfigurationsdatei der Pfad bzw. Name angegeben werden.

Bis MySQL Version 8.0.16 muss der MySQL-Dienst neugestartet werden, wenn Zertifikate hinzugefügt wurden. Ab 8.0.16 werden hierfür dynamische Variablen verwendet. TLS kann also während des Betriebs ohne Neustart aktiviert werden. In der MySQL-Konfiguration sollten folgende Parameter gesetzt werden:

```
[mysqld]
ssl_ca=ca.pem
ssl_cert=server-cert.pem
ssl_key=server-key.pem
tls_version=TLSv1.1,TLSv1.2
#require_secure_transport=ON
```

Da kein Pfad angegeben wurde, erwartet der MySQL-Dienst die Zertifikate im Datenverzeichnis. Achten Sie bitte besonders bei dem privaten Schlüssel auf die korrekten Dateirechte (`chmod 400 _.pem; chown root _.pem`). Dieser sollte auf keinen Fall für alle lokalen Benutzer lesbar sein. Mit dieser minimalistischen Konfiguration kann der MySQL-Dienst nach einem Neustart TLS-Verbindungen anbieten. Allerdings werden auch weiterhin unverschlüsselte Verbindungen zugelassen.

Wie bei vielen anderen Anwendungen, kann man bei MySQL auch nur bestimmte TLS-Versionen aktivieren. Empfehlenswert sind alle TLS-Versionen ab v1.1 aufwärts. Die TLS-Versionen müssen in der MySQL-Konfiguration aufsteigend angegeben werden. Es sollte keine Version ausgelassen werden. Man sollte also nicht nur TLSv1.0 und TLSv1.2 anbieten.

Über `require_secure_transport=ON` werden non-TLS Verbindungen nicht mehr akzeptiert. Diese Option sollte erst nach ausgiebigen Tests aktiviert werden, um Probleme zu vermeiden.

Die gesetzten SSL-Parameter kann man mit dem folgenden Befehl überprüfen:

```
show variables like '%ssl%';
```

Eine TLS-Verbindung kann man mittels MySQL-Client und dem Parameter -ssl testen.

```
mysql —ssl -h mysql_host -u mysql_user -p
```

Je nach MySQL-Client-Version ist der `—ssl` Parameter auch veraltet. Dann sollte `—ssl-mode` genutzt werden. Der Client versucht dann zuerst via TLS eine Verbindung zur Datenbank aufzubauen. Ist dies nicht erfolgreich, wird danach als Fallback eine unverschlüsselte Verbindung aufgebaut. Man kann `—ssl-mode` auch noch einen Wert zuweisen. Beispielsweise versucht der Client dann keine unverschlüsselte Verbindung, falls die TLS-Verbindung fehlschlägt.

```
mysql —ssl-mode -h mysql_host -u mysql_user -p

mysql Ver 15.1 Distrib 10.5.12-MariaDB, for debian-linux-gnu (x86_64) using EditLine wrapper

Connection id: 32
Current database:
Current user: root@localhost
SSL: Cipher in use is TLS_AES_256_GCM_SHA384
```

Man kann eine TLS auch nur für einzelne Benutzer erzwingen:

```
CREATE USER 'test'@'localhost' IDENTIFIED BY 'Password123' REQUIRE SSL;
````

Mögliche Optionen sind:

- `REQUIRE X509`: Der Benutzer kann ein beliebiges passendes Zertifikat verwenden.
- `REQUIRE ISSUER`: Der Benutzer muss ein TLS-Zertifikat, das von einer CA mit dem angegebenen ISSUER ausgestellt wurde, verwenden.
- `REQUIRE SUBJECT`: Wie ISSUER, nur mit einem SUBJECT
- `REQUIRE SSL`: Der Benutzer muss sich via TLS verbinden. Dies kann mit einem Passwort oder Zertifikat geschehen.

Wir haben nun eine TLS-Verbindung auf dem MySQL-Server eingerichtet. Der Server bietet einen privaten Schlüssel inkl. einem Zertifikat an. Diese Methode heißt One-Way, da sich nur der Server mit einem Zertifikat gegenüber dem Client authentifiziert. Der Client wiederum kann optional ein Zertifikat nutzen, um sich beim Server zu authentifizieren. Das Zertifikat kann das Passwort ersetzen oder als Ergänzung genutzt werden.

```
CREATE USER 'user2'@'localhost' REQUIRE X509;
mysql --ssl -h localhost -u user2 --ssl-cert=client-cert.pem --ssl-key=client-key.pem
```

Links:

- [MySQL SSL-RSA Setup](https://dev.mysql.com/doc/refman/5.7/en/mysql-ssl-rsa-setup.html)
