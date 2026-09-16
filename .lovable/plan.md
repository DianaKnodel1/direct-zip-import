# `/danke` zuverlässig nach erfolgreicher Bewerbung öffnen

## Bestätigter Befund

- Die aktuell live ausgelieferte `script.js` enthält bereits die Weiterleitung auf `/danke` und die Beschriftung **„Jetzt Termin vereinbaren →“**.
- Die Seite `/bewerben` bindet das Skript jedoch weiterhin nur als `script.js` ein.
- Dieses Skript darf fünf Minuten zwischengespeichert werden. Ein bereits geöffneter Browser kann deshalb weiterhin die alte Fassung ausführen, die das gezeigte Pop-up öffnet und auf `/bewerben` bleibt.
- Die vorgesehene Versionskennung ist in `server.ts` bereits vorhanden, fehlt aber in der tatsächlich ausgelieferten `server.js`.

## Umsetzung

1. Die Versionskennung aus `server.ts` auch in die produktiv verwendete `server.js` übernehmen.
2. Alle Landing Pages sollen `style.css` und `script.js` künftig mit einer eindeutigen Versionskennung laden, zum Beispiel `script.js?v=<Version>`.
3. Dadurch lädt der Browser nach jedem Neustart bzw. jeder Veröffentlichung sicher die neue Formularlogik statt einer alten gespeicherten Fassung.
4. `server.ts` und `server.js` auf denselben Stand bringen, damit die Abweichung nicht erneut entsteht.

## Prüfung

- Prüfen, dass `/bewerben` live einen Skript-Link mit Versionskennung ausliefert.
- Prüfen, dass dieses Skript `/danke` und **„Jetzt Termin vereinbaren →“** enthält.
- Eine Bewerbung im Browser absenden.
- Erwartung: Die Adresse wechselt auf `/danke`; das alte Pop-up erscheint nicht mehr.

## Veröffentlichung

Nach der Codeänderung muss das Portal erneut veröffentlicht werden, damit die korrigierte `server.js` auf den Landing-Server übertragen und der Dienst neu gestartet wird. Ein erneuter Theme-Abgleich ist dafür nicht erforderlich.
