---
title: "TLS in Nginx konfigurieren"
date: 2024-04-18T21:35:41+02:00
draft: false
tags: ["Nginx", "TLS"]
---

Basiskonfiguration, um HTTPS auf Nginx zu aktivieren. Alle Konfigurationen wurden in Nginx Version 1.14 getestet.

    server {
        listen $IP:443 ssl http2;
        root /var/www/example.com;
        index index.html index.php;
        server_name example.com www.example.com;
        ssl_certificate /etc/ssl/example_fullchain.pem;
        ssl_certificate_key /ssl/example_privatekey.pem;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_prefer_server_ciphers on;
        ssl_ciphers EECDH+AESGCM:EDH+AESGCM;
    }

Im listen Block setzt man den Port auf den gewünschten HTTPS-Port (hier der Standardport 443). Zusätzlich muss die Option ssl gesetzt werden. http2 (ehemals SPDY) ist optional. Notwendig sind zudem die Parameter ssl_certificate und ssl_certificate_key. Hier wird der Pfad zum Zertifikat und privatem Schlüssel angegeben. Eine Option, um Zwischenzertifikate (intermediate certs) anzugeben, fehlt. Diese müssen in der Datei für das Zertifikat hinterlegt werden.

Tipp: Erstellt man ein neues Zertifikat, sollte der Key mindestens eine Länge von 2048 Bit haben. Sicherer sind natürlich 4096 Bit. Keys mit nur 1024 Bit werden von den meisten modernen Webbrowsern nicht mehr akzeptiert.

Generell sollte man die SSL-Versionen (SSLv1 bis SSLv3) und TLSv1.0 deaktivieren. Auch TLSv1.1 ist schon etwas älter und nicht mehr sicher. TLSv1.1 sollte nur aktiviert werden, wenn man zwingend darauf angewiesen ist. Zusätzlich kann man noch eine TLS Cipher List definieren. Bei Cipher (Cipher Suite) handelt es sich um eine Sammlung von verschiedenen kryptographischen Verfahren für Verschlüsselung. Eine Cipher Suite besteht meist aus vier verschiedenen Algorithmen:

- Schlüsselaustausch (RSA, Diffie-Hellman, ...)
- Authentifizierung (DSA, RSA)
- Verschlüsselung (DES, 3DES, AES128, AES256, ...)
- Hashfunktionen (MD5, SHA1, SHA2, ...)

Durch das Definieren einer Cipher Suite kann man zusätzlich bestimmte Algorithmen verbieten. Als Vorlage für sichere Cipher empfehle ich folgende Webseite: cipherlist.eu.

Tipp: Achtet immer auf die Berechtigungen von (privaten) Schlüsseln. Diese sollten in der Regel 600 sein. Also nur für den Besitzer lesbar. So verhindert man, dass andere lokale Benutzer den Schlüssel auslesen können. Welchen Besitzer die Zertifikate haben, ist abhängig von der Anwendung. In den meisten Fällen ist dies aber root. Dienste wie Nginx oder Apache werden immer vom root Benutzer gestartet. Die (child) Prozesse laufen dann meist unter einem anderen Benutzer (z.B.: www oder www-Data).

Zusätzlich kann man noch DH (Diffie-Hellmann) Parameter definieren und diese in einer Datei festhalten. Die Datei kann über folgenden Befehl mittels OpenSSL erzeugt werden:

    openssl dhparam -out /etc/ssl/dhparam4096.pem 4096

Je nach Hardware kann das Erzeugen eines solchen Schlüssels einige Minuten dauern.

    ssl_dhparam /etc/nginx/dhparam.pem;

Weitere optionale Parameter:

    ssl_ecdh_curve secp384r1;
    ssl_session_timeout 10m;

Seine Nginx Konfiguration (bzw. Konfigurationen) kann man mit dem Befehl `nginx -t` testen. Dies kann unschöne Momente nach einem Reload oder Restart verhindern. Zum Testen von TLS-Konfigurationen empfehle ich folgende Tools:

    ssllabs.com/ssltest/
    github.com/drwetter/testssl.sh

Der Test von ssllabs.com ist relativ bekannt und eine gute Möglichkeit, seinen Webserver zu testen. Hier werden u.a. auch TLS-Handshakes simuliert. Man kann also sehen, welche Clients oder Browser keine TLS-Verbindung mehr aufbauen können. Wer mehr als nur Webseiten testen möchte, findet auf GitHub das testssl.sh Skript von Dirk Wetter. Mit dem Skript kann man TLS-Verbindungen auf allen Ports testen, zum Beispiel für E-Mail-Server oder Datenbanken.

Letztendlich ist eine sichere TLS-Konfiguration immer nur eine Momentaufnahme. TLS-Versionen oder Cipher können über Nacht als unsicher gelten. Dann muss schnell die Konfiguration angepasst oder Software aktualisiert werden. Ein Skript wie testssl.sh kann man auch super einmal monatlich ausführen und seine Umgebung auf Schwachstellen testen. Natürlich sollte man auch ab und zu das Skript selbst aktualisieren...
