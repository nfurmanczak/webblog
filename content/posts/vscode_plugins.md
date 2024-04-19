---
title: "Nützliche VScode Plugins"
date: 2023-04-19T21:35:41+02:00
draft: false
tags: ["vscode"]
---

Eine kurze Liste mit aus meiner Sicht empfehlenswerten Plugins für VScode.

## Polacode

Polacode ist kein Plugin, das einem direkt im Alltag oder bei seinem Workflow hilft, ausgenommen, ihr wollt Teile eures Quellcodes als Bild auf Twitter posten. Dann braucht ihr Polacode unbedingt. Natürlich kann man Bilder seines Quellcodes in den eigenen Dokumentationen unterbringen. Nur werden einem dann die Kollegen hassen, die eure Codeschnipsel abtippen müssen.

[Polacode Plugin](https://marketplace.visualstudio.com/items?itemName=oderwat.indent-rainbow)

## indent-rainbow

Wer viel mit YAML oder Python arbeitet, kann durch das Einrücken manchmal den Überblick verlieren. Das Plugin indent-rainbow färbt die Einrückungen farbig ein. Besonders empfehlenswert, wenn man gerade neu mit YAML (z. B.: Ansible) oder Python anfängt.

[indent-rainbow Plugin]()

## bracket-pair-colorizer

Für Python-Entwickler vermutlich überflüssig. Aber mit diesem Plugin könnt ihr alle Klammern einfärben und optisch besser hervorheben. Klingt trivial, kann aber in einigen Situationen durchaus helfen.

[bracket-pair-colorizer Plugin](https://marketplace.visualstudio.com/items?itemName=CoenraadS.bracket-pair-colorizer)

## Rainbow CSV

Wir bleiben bei farbenfrohen Helferlein: Mit Rainbow CSV werden CSV-Dateien visuell deutlich besser dargestellt. Auch wenn ich nicht täglich mit CSV-Dateien arbeite, hat dieses Plugin einen Platz auf dieser Liste verdient.

[Rainbow CSV Plugin](https://marketplace.visualstudio.com/items?itemName=mechatroner.rainbow-csv)

## Docker

Auf meinem Notebook hat Docker fast vollständig virtuelle Maschinen ersetzt. Mit dem Plugin kann ich direkt Dateien im Docker Container via VSCode bearbeiten. Ein nützliches Feature, wenn man aktuell Software debuggt.

[Docker Plugin](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-docker)

## file-icon

Kleine passende Symbole in der Dateileiste können VSCode optisch etwas hübscher, aber auch übersichtlicher machen. Gerade bei Verzeichnissen mit vielen Dateien kann man sich so schneller orientieren. Hierfür gibt es viele verschiedene Plugins. Für jeden Geschmack sollte das passende Icon-Pack dabei sein. Persönlich würde ich die beiden folgenden empfehlen:

[file-icons Plugin](https://marketplace.visualstudio.com/items?itemName=file-icons.file-icons)
[vscode-icons Plugin](https://marketplace.visualstudio.com/items?itemName=vscode-icons-team.vscode-icons)

## Prettier

Was soll ich sagen. Entweder ist man sehr diszipliniert bei seinem Quellcode und formatiert alles sauber, oder man nutzt Plugins wie Prettier, die das für einen automatisch erledigen. Und das richtig gut. Am meisten Spaß macht mir das Plugin, wenn ich mit HTML5 oder CSS-Dateien arbeite.

[Prettier Plugin](https://marketplace.visualstudio.com/items?itemName=esbenp.prettier-vscode)

## Remote SSH

Wer viel remote via SSH auf anderen Servern Dateien anpasst, sollte sich mal RemoteSSH ansehen. Der Name ist Programm. Das Plugin richtet zwar auf dem Remote-Ziel einen „VSCode Server“ ein, der in seltenen Fällen auch mal abstürzt oder 100% der CPU frisst, aber dafür habe ich im Quellcode eine Autovervollständigung für Dateipfade. Das genügte schon, um mich zu überzeugen. (Wer das ebenfalls interessant findet, aber das SSH Plugin nicht braucht, sollte sich mal „Path Intellisense“ ansehen)

[Remote SSH Plugin](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-ssh)
[Path Intellisense](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-ssh)

## Quokka.js

Diese Erweiterung habe ich mir persönlich noch nicht so intensiv genutzt. Die Idee und Umsetzung finde ich aber trotzdem gelungen. Quokka kann euch bei der Entwicklung und dem Debugging von JavaScript und TypeScript helfen. Das Plugin zeigt euch eine Live „Codevorschau“. Euer Quellcode wird also direkt ausgeführt während der Entwicklung. Hilft vor allem, kleine logische Fehler schnell zu entdecken. Wer nur ab und zu mal JavaScript/TypeScript macht, sollte sich das Plugin einmal ansehen und testen.

[Quokka.js Plugin](https://marketplace.visualstudio.com/items?itemName=WallabyJs.quokka-vscode)
