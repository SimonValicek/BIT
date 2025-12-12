# Bezpečnosť elektronickej pošty a SMTP komunikácie

Tento projekt sa zameriava na praktickú analýzu bezpečnostných slabín SMTP komunikácie
na úrovni prenosu, autentifikácie a politiky relaying-u.

Projekt **nerieši SPF, DKIM ani DMARC** a nezameriava sa na autentifikáciu identity
odosielateľa pomocou DNS mechanizmov. Cieľom je analyzovať bezpečnosť SMTP servera
z pohľadu komunikácie medzi klientom a MTA.