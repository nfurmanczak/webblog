---
title: "Sender Policy Framework (SFP)"
date: 2023-04-10T21:35:41+02:00
draft: false
tags: ["email", "DNS", "SPF"]
---

Bei SPF handelt es sich um einen Standard, um das Versenden von unautorisierten E-Mails zu verhindern. Hierbei wird ein TXT-Record (Text-Record) in die DNS-Zone eingetragen. Dieser TXT-Record enthält eine Liste von IP-Adressen oder anderen DNS-Namen, die für diese Domain E-Mails versenden dürfen.

Beispiel SPF-Record für die Domain example.com:

	"v=spf1 ip4:216.10.0.10/30 ip4:180.29.0.22 a:mail.example.com -all"

Jeder SPF-Record beginnt mit dem Visions Tag `v=spf1. Aktuell (Jan. 2022) gibt es noch keine Version 2. Neben SPF gibt es noch die SenderID, welche große Ähnlichkeit mit SPF hat. Teilweise wird die SenderID auch als spf2.0 angegeben. Die SenderID ist allerdings nicht so verbreitet wie SPF. In diesem SPF-Record dürfen die IPv4-Adressen aus dem Netz 216.10.0.10/30, die einzelne IPv4-Adresse 180.29.0.22/32 sowie die IPv4 Adresse hinter der Domain mail.example.com E-Mails versenden.

Der SPF-Record muss für die Domain im Envelope Header (RFC 5321) der E-Mail gesetzt werden. Diese Domain wird auch beim SMTP-Handshake verwendet. Die Envelope Domain sollte nicht mit der Domain im From-Header (Friendly-From, RFC 5322) verwechselt werden. Wer sich unsicher ist, kann sich gerne selbst eine Test-E-Mail senden und im Quelltext nach dem Return-Path Header suchen. In diesem Header steht die Domain aus dem Envelope Header.

In einem SPF-Record können über folgende Mechanismen IP-Adressen definiert werden:

|Mechanismus|Funktion|
|---|---|
|a, a:domain.org|Prüft den A und AAAA Record der angegebenen Domain nach dem Doppelpunkt. Wird keine Domain angegeben, wird der A und AAAA DNS-Eintrag der Domain vom SPF-Record geprüft.|
|mx, mx:mail.domain.org| Prüft den MX Record der angegebenen Domain nach dem Doppelpunkt. Wird keine Domain angegeben, wird der MX Eintrag der Domain vom SPF-Eintrag geprüft|
|ip4 oder ip6|	Hinter ipv4 und ip6 kann jeweils eine einzelne IP-Adresse oder ein Netz in CIDR Notation angegeben werden. Bei einzelnen Adressen muss kein /32 angegeben werden.|
|include:|Erweitert den SPF-Record um den SPF-Record von einer anderen Domain. Dieser SPF-Record darf auch wieder ein include enthalten. Dies kann sich bis zu 10mal wiederholen.|

Das `all` am Ende eines SPF-Records hat eine wichtige Bedeutung und wird fast immer auch in einem SPF-Record benötigt. Einzige Ausnahme ist ein `redirect`. Das `all` besagt, wie der Empfänger mit E-Mails von anderen IP-Adressen umgehen soll, welche nicht im SPF-Record hinterlegt sind. Ausschlaggebend hierfür ist das Zeichen vor dem all. SPF-Records werden immer von links nach rechts gelesen. Nach einem Treffer wird sofort aufgehört und der SPF-Record nicht weiter ausgewertet oder weitere DNS-Querys für includes abgesetzt. Optimalerweise sollte der SPF-Record so aufgebaut werden, dass die häufigsten IP-Adressen am Anfang stehen.

|Qualifikator|Funktion|
|---|---|
|+all (Pass)|Das +all ist theoretisch möglich, sollte aber in der Praxis nirgendwo vorkommen. Es bedeutet, dass alle anderen IP-Adressen ebenfalls für diese Domain E-Mails versenden dürfen. Der SPF-Record hat also keine Funktion. Diesen Fehler sollte man unbedingt vermeiden. Möglicherweise wird der SPF-Record von einigen Empfängern auch als ungültig angesehen. Das Pluszeichen ist gleichzeitig auch der Standard Qualifikator. Wird keiner angegeben, gilt automatisch das Pluszeichen. a:domain.de wird also automatisch zu +a:domain.de.|
|?all (Neutral)|Mit ?all überlässt man dem Empfänger, ob er die E-Mail annimmt oder nicht. Da diese Einstellung relativ nichtssagend ist, sollte man diese auch nicht verwenden.|
|~all (Softfail)|Ab dem Softfail wird es interessant. Der Empfänger soll E-Mails von anderen IP-Adressen noch annehmen. Meist werden diese E-Mails angenommen aber nicht in die Inbox, sondern in einen SPAM Ordner hinterlegt. Der Softfail eignet sich vor allem, wenn man den SPF-Record neu gesetzt hat.|
|-all (Hardfail)|Der Hardfail ist die härteste Regel bei einem SPF-Record. Der Empfänger wird alle E-Mails von nicht autorisierten IP-Adressen ablehnen.|

In der Praxis wird man meistens einen Soft- oder Hardfail vorfinden. Der Hardfail ist natürlich dem Softfail vorzuziehen. Allerdings sollte man bei einem SPF-Record mit Hardfail sicher sein, dass alle E-Mail Server im Record erfasst wurden. Ansonsten werden Mailbox Provider E-Mails nicht zustellen und auch in keinen SPAM (oder ähnlichen) Ordner zustellen.

Wer schon einen SPF-Record hat, kann auch via redirect auf einen anderen SPF-Record verweisen. Hier wird ein SPF-Record nicht wie bei einem include um einen anderen erweitert. Bei einem redirect gibt es allerdings ein paar wichtige Punkte zu beachten. Bei einem redirect darf kein all am Schluss gesetzt werden. Ansonsten wird der redirect Parameter ignoriert. Da es sich bei redirect um einen Modifier handelt, muss ein Gleichheitszeichen (=) und kein Doppelpunkt gesetzt werden. Der redirect besitzt auch die niedrigste Gewichtung und wird in einem SPF-Record immer zuletzt ausgewertet. Vorausgesetzt, dass es vorher nirgendwo anders einen Match gibt.

	"v=spf1 redirect=domain-example.org"

Tipp: Aufgrund der TTL einer DNS-Zone kann es etwas dauern, bis eine Änderung an einem DNS-Eintrag überall angekommen ist. Bei wichtigen Changes kann man 24 Stunden vorher die TTL auf 300 Sekunden ändern. Man sollte allerdings nicht vergessen, nach dem Change die TTL wieder auf einen höheren Wert zu setzen (default ist meist 86400). Ansonsten bekommt der eigene DNS-Server mehr Last als gewünscht.
Validieren von SPF-Records

Mit dem Tool dig kann man DNS-Einträge auslesen. Um die Ausgabe etwas übersichtlicher zu haben, kann man die Option +short angeben.

	dig -t txt example.com +short

Wer kein Zugriff auf ein Unix/Linux System mit dig hat, kann auch seinen SPF-Record mit verschiedenen Online Tools prüfen:

- mxtoolbox.com
- dmarcanalyzer.com/de/spf-de/checker
- dmarcian.com/spf-survey/

Für die meisten Programmiersprachen gibt es auch Plugins/Frameworks/Pakete, welche SPF-Records validieren können. Hier muss man selbst das Rad nicht neu erfinden.

## Nachteile von SPF

SPF kann bei Umleitungen von E-Mails u.a. problematisch werden. Bei einer Umleitung ändert sich der Absender einer E-Mail. Somit wird die SPF Prüfung scheitern. Je nach Bedingung (Hardfail, Softfail) wird die E-Mail nicht zugestellt. Ein Lösungsansatz ist ARC (Authenticated Received Chain). Hierbei prüft der Server, welche die E-Mail zuerst annimmt, ob SPF (und DKIM) korrekt sind. Das Ergebnis wird in einer Signatur dokumentiert und im Quelltext der E-Mail vermerkt. Die späteren E-Mail Server können die Signatur prüfen und werden fehlende IP-Adressen im SPF-Record ignorieren.
Brauche ich nun einen SPF-Record?

Um es kurz zu halten: SPF ist nicht zwingend nötig, wenn man einen eigenen E-Mail Server hat und E-Mails versenden möchte. Es kann aber helfen, Zustellprobleme bei Mailbox Providern wie Gmail, GMX oder Yahoo zu vermeiden. Ob eine E-Mail von einem Server angenommen wird und die Inbox des Benutzers erreicht, hängt von mehreren verschiedenen Faktoren ab. SPF ist nur einer davon. Nicht jeder (Mailbox) Provider prüft auch, ob ein SPF-Record vorhanden ist. Allerdings ist ein SPF-Record relativ einfach zu setzen. Wer also die Möglichkeit hat, sollte auch einen SPF-Record in seiner DNS-Zone setzen. Wer nebenbei über eine Domain keine E-Mails versenden möchte, kann folgenden SPF-Record setzen:

	"v=spf1 -all"

Abschließend möchte ich noch einen wichtigen Punkt erwähnen: SPF ist keine Technik, um (direkt) Spam zu verhindern. Jeder kann sich eine Domain mieten, einen Server besorgen und in dieser Domain einen entsprechenden SPF-Record eintragen. Mit diesem Setup kann jeder sofort einiges an Spam versenden. Zumindest so lange, bis seine IP-Adresse(n) oder Domain gesperrt wird. SPF wird eher genutzt, um die eigene Domain vor Missbrauch oder (begrenzt vor) Phishing zu schützen.