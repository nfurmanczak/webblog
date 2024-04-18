---
title: "DKIM mit Postfix und openDKIM"
date: 2023-04-10T17:21:55+02:00
draft: false
tags: ["email", "crypto", "dkim"]
---

DKIM (DomainKeys Identify Mails) ermöglicht es, ausgehende E-Mails (Header und Body) mithilfe eines privaten Schlüssels zu signieren. Diese Signatur kann der Empfänger mit dem öffentlichen Schlüssel aus der DNS-Zone des Versenders überprüfen. Passt die Signatur, so wurde die E-Mail auf dem Weg vom Sender zum Empfänger (höchstwahrscheinlich) nicht durch Dritte verändert. Zudem sollte nur der vorgesehene und vertrauenswürdige Versandserver im Besitz des privaten Schlüssels sein. Die Signatur sollte allerdings nicht mit einer Verschlüsselung verwechselt werden. Bei einer Signatur wird lediglich der Hashwert einer Information verschlüsselt, nicht aber der Inhalt der E-Mail. Möchten Sie sensible Informationen per E-Mail versenden, sollten Sie auf eine Transportverschlüsselung ([TLS-Verbindung](https://de.wikipedia.org/wiki/Transport_Layer_Security) zwischen beiden E-Mail-Servern) und/oder Ende-zu-Ende-Verschlüsselung (z.B.: [GPG](https://de.wikipedia.org/wiki/GNU_Privacy_Guard) oder [S/MIME](https://de.wikipedia.org/wiki/S/MIME)) setzen.


Vielleicht ergibt sich nicht direkt der Nutzen von DKIM. Ein Ersatz für eine Verschlüsselung ist DKIM ja nicht. Vielmehr kann man DKIM als Erweiterung des Sender Policy Framework (SPF) ansehen. Bei SPF werden die IP-Adressen der E-Mail-Server in einem DNS-Eintrag verzeichnet. DKIM erweitert dies um einen öffentlichen Schlüssel. Damit kann der Domaininhaber festlegen, welche Server für den E-Mail-Versand dieser Domain autorisiert sind. Zudem ist DKIM de facto Voraussetzung, um später auch erfolgreich [DMARC](https://de.wikipedia.org/wiki/DMARC) einzusetzen.


Mit DKIM haben wir nun eine weitere wichtige Information (den öffentlichen Schlüssel) in unserer DNS-Zone. Natürlich kann ein öffentlicher Schlüssel gefahrlos im Internet verteilt werden. Allerdings können DNS-Abfragen an verschiedenen Stellen manipuliert werden. Kurz gesagt: Ohne [DNSSEC](https://de.wikipedia.org/wiki/Domain_Name_System_Security_Extensions) gilt eine Antwort von einem DNS-Server als nicht vertrauenswürdig. Die DNS-Zone entwickelt sich also Stück für Stück zu einer sensiblen Informationsquelle, die geschützt werden soll. DNSSEC ist keine zwingende Voraussetzung für DKIM (oder SPF oder DMARC), allerdings ist DNSSEC immer eine sinnvolle Erweiterung, um auch Antworten vom DNS-Server zu signieren und vor Manipulation zu schützen.

### Erstellen eines Schlüsselpaares

Ein Schlüsselpaar für DKIM kann man über verschiedene Wege erstellen. Ich empfehle aber das Tool `opendkim-genkey` aus dem Paket `opendkim-tools`. Dies erzeugt den öffentlichen Schlüssel direkt im passenden Format für die DNS-Zone. Der folgende Befehl erstellt ein Schlüsselpaar für die Domain `example.com` mit 2048 Bit (Standardwert) und dem Selektor (`-s`) `key1`:

```
opendkim-keygen -s key1 -d example.com
```

Der Selektor (`-s`) kann frei gewählt werden. Er dient dazu, den Schlüssel in der DNS-Zone zu identifizieren. Durch den unterschiedlichen Selektor können in einer Domain mehrere DKIM-Schlüssel hinterlegt werden. Als Schlüssellänge wird aktuell (ich schreibe diesen Text im Jahr 2022) noch 2048 Bit empfohlen. Längere Schlüssel sind möglich, können aber bei DNS-Abfragen wegen der größeren Datenmenge problematisch werden.

Der Befehl erzeugt zwei Dateien: Den privaten Schlüssel (`key1.private`) und eine Textdatei mit dem öffentlichen Schlüssel (`key1.txt`) für den DNS-Eintrag. Alles zwischen den Klammern muss in den DNS-Eintrag übertragen werden.

### DKIM-DNS-Eintrag

Der öffentliche DKIM-Schlüssel wird als TXT-Eintrag in der DNS-Zone hinterlegt. Der Name setzt sich aus dem gewählten Selektor, der Domain und dem Zusatz `_domainkey` zusammen. Unser öffentlicher Schlüssel aus dem Beispiel wäre also unter folgender Adresse aufrufbar: `key1._domainkey.example.com`.

Der Zusatz `_domainkey` darf nicht verändert werden und muss unbedingt vorhanden sein. In der DKIM-Signatur der E-Mail steht die Domain und der genutzte Selektor. Mit diesen Informationen kann der Empfänger die Signatur validieren.

### openDKIM konfigurieren

Nach der Installation sollte ein Verzeichnis für den (oder die) openDKIM-Schlüssel eingerichtet werden. Der Verzeichnisname und Speicherort kann frei gewählt werden. Allerdings muss der Benutzer `opendkim` die entsprechenden Rechte haben, um den Schlüssel lesen zu dürfen.


	mkdir -p /etc/opendkim/keys
	cp key1.private /etc/opendkim/keys
	chown -R opendkim:opendkim /etc/opendkim
	chmod 600 /etc/opendkim/keys


Die zentrale Konfigurationsdatei von openDKIM liegt unter `/etc/opendkim.conf`. Bei Debian und Ubuntu wird jedoch auch die Datei `/etc/default/opendkim` geladen. Diese besitzt zudem eine höhere Priorität gegenüber `/etc/opendkim.conf`. Man sollte sich also entscheiden, welche der beiden Dateien man nutzen möchte. Falls man die Datei im `default`-Verzeichnis nicht nutzen möchte, kann man dort alles bis auf den `RUNDIR`-Parameter löschen oder auskommentieren.

Zusätzlich empfehle ich, noch zwei weitere Dateien im `/etc/opendkim`-Verzeichnis anzulegen: `signing.table` und `key.table`. Diese Dateien enthalten Informationen, welche privaten Schlüssel für welche Domain zum Signieren genutzt werden.

Inhalt der Datei `key.table`:

```
example example.com:key1:/etc/opendkim/keys/key1.private
```

Inhalt der Datei `signing.table`:

```
*@example.com example
```

In der `signing.table` wird festgelegt, welcher Schlüssel für die angegebene (Sub-)Domain genutzt wird. In diesem Beispiel soll der Eintrag mit dem Namen `example` in der `key.table` für die Domain `@example.com` verwendet werden.

Zuletzt widmen wir uns dem Herzstück von openDKIM, der `opendkim.conf`:

```
RUNDIR=/var/run/opendkim
SOCKET=inet:12345@localhost
USER=opendkim
GROUP=opendkim
PIDFILE=$RUNDIR/$NAME.pid
EXTRAAFTER=
MODE=sv
Canonicalization=relaxed/relaxed
SigningTable=refile:/etc/opendkim/signing.table
KeyTable=/etc/opendkim/key.table
SignatureAlgorithm=rsa-sha256
OversignHeaders=FROM
```

Erwähnenswert sind hier die folgenden Parameter:

| Parameter           | Funktion                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
|---------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Socket              | Der Socket, über den Postfix auf openDKIM zugreift.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| Canonicalization    | E-Mail-Server können manchmal eingehende E-Mails verändern. Diese Veränderungen können dazu führen, dass eine DKIM-Signatur ungültig wird. Um diesem Problem etwas vorzubeugen, definiert DKIM unterschiedliche Canonicalization-Verfahren. Die beiden Verfahren heißen "simple" und "relaxed". Je nachdem, ob diese auf einen E-Mail-Header oder Body angewendet werden, verhalten sie sich etwas unterschiedlich. Meine Empfehlung wäre hier unbedingt "relaxed/relaxed" zu verwenden. Das erste "relaxed" steht für den Header, das zweite für den Body.                                                                                                                |
| SignatureAlgorithm  | Legt den verwendeten Hashalgorithmus und das kryptografische Verfahren für die Signatur fest. "rsa-sha256" sind der Standard und gute Werte. Andere Verfahren werden meist nicht unterstützt. Mittels `opendkim -V` kann man die unterstützten Algorithmen einsehen.                                                                                                                                                                                                                                                                                                                                                                                               |
| OversignHeaders| Beim Oversigning wird ein Header mehrfach signiert, auch wenn ein Header nur einmal vorkommt. Die Signatur wird also einmal über den vorhandenen Header (und dessen Inhalt) gebildet und dann über einen nicht vorhandenen Header (also nichts). Wird die E-Mail nachträglich verändert und ein vorhandener Header noch einmal hinzugefügt, in der Hoffnung, Fehler in E-Mail-Clients auszunutzen, wird die DKIM-Signatur ungültig.|


### Postfix konfigurieren

Der openDKIM-Dienst wird in der Postfix-Konfiguration (`/etc/postfix/main.cf`) als Mail-Filter (Milter) eingetragen. Falls noch kein Milter in Postfix konfiguriert ist, kann folgende Konfiguration genutzt werden:

```
milter_default_action = accept
milter_protocol = 6
smtpd_milters = inet:localhost:12345
non_smtpd_milters = inet:localhost:12345
```

Mit diesen Einstellungen kann der Postfix- und openDKIM-Dienst neu gestartet (bzw. neu geladen) werden. Mit dem Programm `opendkim-testkey` (aus `opendkim-tools`) kann geprüft werden, ob der öffentliche Schlüssel in einem gültigen Format vorliegt:

```
opendkim-testkey -d example.com -s key1 [-vvv]
```

Der Befehl kann optional noch um den Schalter `-v` ergänzt werden. Wie bei Linux-Tools üblich, gibt es keine Rückmeldung, wenn alles in Ordnung ist.

Jetzt können wir eine Testmail versenden und unsere DKIM-Signatur in der E-Mail überprüfen. Wir können natürlich auch eine E-Mail an uns selbst senden, um die DKIM-Signatur zu prüfen, es sei denn, wir haben die DKIM-Signatur für unsere eigene Domain deaktiviert. Ich würde aber eher eine Testmail an ein Gmail-, AOL-, Yahoo- oder Outlook/Microsoft-Konto senden. Viele Mailbox-Anbieter vermerken im Quelltext der E-Mail auch den Status der DKIM-Signatur.

### DKIM-Signatur

Eine DKIM-Signatur setzt sich aus verschiedenen Tags zusammen. Anhand des folgenden Beispiels werden wir uns die wichtigsten Tags genauer ansehen:

```
DKIM-Signature: v=1; a=rsa-sha256; c=relaxed/relaxed; d=example.com; s=key1;
    bh=9STWz4nrhKjxqSxQWYoki+h8Gct3PtzEmgm+Pp9CAl4=; h=From:FromSubject:Date:To;
    b=OVNK91I5VuNlYiwc5BHmCgqWWKzji1VotcJiIGSw0Ydfm4jqdD5oIgt7B++FWPoES
    +SvSuThinNd8zDBwiF95y+kAvOj46uP6vo5kdUvkF9a5iBWhYX67b0BaaDCJZEvauw
    aluZVU8F+GbYYFlj6/HNzwRkn5q1fEUW4Bg4+k/tpMmuPYUZVJiGgKXCDjAlDvDJME
    uPqiaPsKDcl2cmK1QV5O2rCAyrpLx0pLZJxeDTPyj5kAFg8NYNWWCL8XCfV3f06D5n
    c04ba/qW056fvyKgVLOzQYlun6uC3iv637lIEwb8OY4bVWVnasfZAVx+CwMhtSnjZv
    XmwmvM3oUfQvQ==
```

| Tag| Funktion|
|---|---|
| `v=1;`| Die genutzte DKIM-Version. Aktuell (2022) gibt es keine neue Version.|
| `a=rsa-sha256;`| Der verwendete Algorithmus.|
| `c=relaxed/relaxed;`|Das genutzte Canonicalization-Verfahren für den Header und Body.|
| `d=example.com;`|Die DKIM-Domain, in der sich auch der öffentliche Schlüssel befindet.|
| `s=key1;`| Der Selektor (Name) des Schlüssels.|
| `h=From:FromSubject:Date:To;` | Enthält die signierten Header. Oversigned Header sind doppelt vorhanden in der Liste.|
| `bh= ...;`| Hash des E-Mail-Bodys.|
| `b= ...;`| Hash des E-Mail-Headers, einschließlich aller wichtigen Metadaten aus der DKIM-Signatur.|


### DKIM-Alignment

Wie bei SPF gibt es auch bei DKIM ein Alignment. Ein Alignment ist erreicht, wenn die Domain im `From:` und der Domain-Tag der DKIM-Signatur (`d=...`) identisch sind. Hierzu ein kurzes Beispiel:

Die E-Mail wurde von der E-Mail-Adresse `From: bob@example.de` gesendet. Der öffentliche DKIM-Schlüssel befindet sich in der Domain `d=example.de`. Beide Domains sind identisch, es gibt also ein DKIM-Alignment. Dies würde auch bestehen, wenn die E-Mail von `From: bob@web.example.de` gesendet worden wäre. Allerdings nur, wenn im DMARC-Record das Alignment für DKIM als relaxed gesetzt ist.

### ARC (Authenticated Received Chain)

Wer sich öfter mal den Quelltext von E-Mails angesehen hat, dem könnte vielleicht eine ARC-Signatur oder ein ARC-Seal aufgefallen sein. Vor allem die größeren Mailbox-Provider wie Gmail, AOL oder Microsoft setzen ARC schon seit einigen Jahren ein.

Bei Mailinglisten, also dem Umleiten von E-Mails, kann es zu Problemen mit SPF und DKIM kommen. Deswegen prüft der erste Empfänger die DKIM-Signatur und schreibt das Ergebnis in den Header der E-Mail. Diese Information wird wieder mittels einer zusätzlichen DKIM-Signatur signiert. Weitere Empfänger können so diese neue ARC-Signatur prüfen. Technisch handelt es sich bei ARC also eigentlich um eine (zusätzliche) DKIM-Signatur.

Beispiel einer ARC-Signatur (bestehend aus ARC-Seal, ARC-Message-Signature und ARC-Authentication-Results):


 
	ARC-Seal: i=1; a=rsa-sha256; t=1653370592; cv=none;
	d=google.com; s=arc-20160816;
	b=L7DGTN1C6tZYZ6J7x3WPntsi5vZsiw66J5UudCz6uyacYGlsfVG22narurUKoQRxZZDgKijBNaOxaRTIwi3oys2nqBAzFcbOY7bEWMGJh5fHqAPWDdUsI/6OpqtSXHkfrnFlycnaTWWM83R1gKZMuPdkcbLaaTX4Qv+syzW/vxsiBZlvpXwXW0Fn7Br5iFuyuHKwJWH4NbBl6ptCcNBnVovSffH0E0K3eL/KPUsLLXl4tdj8tpTmtyjBQ0cjMOqAdpfDQFOy9P/kJSGxtXAQNdrN6QoumqKIAxDnCSbzoODtSpMMpkzWgEMICOczNBNQ3WCsshvSeHQSKaH033I8ew==
    ARC-Message-Signature: i=1; a=rsa-sha256; c=relaxed/relaxed; d=google.com; s=arc-20160816; h=to:references:message-id:subject:date:mime-version:from:content-transfer-encoding:dkim-signature; bh=IbKKW+FuGfbd6G5AXON591T9sxFMyXkAMVqgA64URJM=;b=eJSRFRPxcn1hdLCidRGn/Y/RPTorPlB1tFwx8PIKywEMEkJNgRuEdOdPpu6z+/aqSC9cVy7mAeb64BP+tfFWaMJywh0DOYzAjUTrHALR3GgX54N427iRzV5/EdZ9TrtSc1TouMHWgO79wlq5jAfxOe4qt+SySq6nytrQREy1TkSNahg0kBkX5qqyJm3gIfP3NjQJJ2mfo86FP2S9pa9WlROiuBULZ88/J1Jc0yNGIQvH54lqMkmu3/KMrltn1IosVwa2m46IDATle9bXRY/OEnMf2atA+RjgVXjt4ObTPLDe+d0ZmFpd/8G7AgVN9vTvBuastIc20HDJtjk4KjjTNg==
    ARC-Authentication-Results: i=1; mx.google.com;
    dkim=pass header.i=@gmail.com header.s=20210112 header.b=qbAPGVPb; spf=pass (google.com: domain of versender@gmail.com designates 209.85.220.41 as permitted sender) smtp.mailfrom=versender@gmail.com; dmarc=pass (p=NONE sp=QUARANTINE dis=NONE) header.from=gmail.com
        

### Monitoring für DKIM

Es ist durchaus sinnvoll, die DKIM-Infrastruktur zu überwachen. Das Prüfen des öffentlichen Schlüssels in der DNS-Zone ist relativ simpel. Man sollte aber nicht nur prüfen, ob der DNS-Record vorhanden ist, sondern ob auch der korrekte Schlüssel inkl. weiterer Parameter in der DNS-Zone eingetragen ist.

Das nachfolgende Bash-Skript erwartet drei Übergabeparameter und prüft mittels `dig`, ob der Schlüssel korrekt ist. Ein solches Skript könnte z.B. in Icinga eingebunden werden. Natürlich kann man anstelle der Übergabeparameter diese auch direkt statisch ins Skript schreiben. Alternativ bieten aber auch Sprachen wie Python oder Perl entsprechende Module, um DNS-Abfragen durchzuführen.


	#!/bin/bash
    if [ "$#" -ne 3 ]; then
    	exit 255
    fi
    
    selector=$1
    domain=$2
    expected_public_key=$3
    #expected_public_key='"v=DKIM1; h=sha256; k=rsa;
    p=MIGfMA0GCSxruhUzopyrLWvJUnSe7zbfWl2m+GSf5ieMpDTESxkpj6iun6qEwsVUfc3ylw7C+Cy7L/ZpcWQWCLRaeMLxTZb7j2PY0nQB7/4DJUdWV5y0zPMwIDAQAB"'
    
    public_key_in_dnszone=$(/usr/bin/dig -t TXT +short "$selector._domainkey.$domain")
    if [ "$expected_public_key" = "$public_key_in_dnszone" ]; then
    	exit 0
    else
    	exit 1
    fi
    

Für bestimmte E-Mail-Clients gibt es auch Plugins, um eine DKIM-Signatur zu überprüfen. Für Thunderbird kann ich das Tool DKIM-Verifier empfehlen. Ansonsten gibt es wenig weitere Tools, um automatisiert DKIM-Signaturen zu überwachen. Zudem muss ja auch eine signierte E-Mail verwendet werden. Man bräuchte also ein Test-E-Mail-Konto, welches signierte E-Mails empfängt. Diese E-Mails müsste man dann mit einer kleinen selbst geschriebenen Software überprüfen.

Allerdings gibt es auch noch DMARC. Empfänger, die DMARC unterstützen, verschicken Reports über den Zustand von SPF und DKIM. Mittels den DMARC-Reports bekommt man also einen Überblick, ob es bei den Empfängern zu Problemen mit SPF und DKIM gibt.
