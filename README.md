# Bezpečnosť elektronickej pošty a SMTP komunikácie

Tento projekt sa zameriava na praktickú analýzu bezpečnostných slabín SMTP komunikácie
na úrovni prenosu, autentifikácie a politiky relaying-u.

Projekt **nerieši SPF, DKIM ani DMARC** a nezameriava sa na autentifikáciu identity
odosielateľa pomocou DNS mechanizmov. Cieľom je analyzovať bezpečnosť SMTP servera
z pohľadu komunikácie medzi klientom a MTA.

## Úvod k SMTP protokolu
### Historický kontext
Protokol SMTP (Simple Mail Transfer Protocol) bol štandardizovaný začiatkom 80. rokov minulého storočia (RFC 821, neskôr RFC 5321) v období, keď počítačové siete fungovali prevažne v akademickom a výskumnom prostredí. Tieto siete boli malé, uzavreté a založené na vzájomnej dôvere medzi jednotlivými uzlami.

### Pôvodné návrhové predpoklady
SMTP bol navrhnutý ako jednoduchý prenosový protokol, ktorý predpokladá korektné správanie zúčastnených strán. Protokol neobsahuje natívne mechanizmy na autentifikáciu odosielateľa, šifrovanie komunikácie ani ochranu pred zneužitím.

### Vývoj bezpečnostných rozšírení
S rastúcim významom elektronickej pošty a zmenou bezpečnostného prostredia boli postupne zavedené viaceré bezpečnostné mechanizmy, ako napríklad SMTP AUTH, STARTTLS, SPF, DKIM a DMARC. Tieto mechanizmy výrazne zvyšujú úroveň bezpečnosti e-mailovej komunikácie.

### Hop-based model a jeho dôsledky
Napriek týmto rozšíreniam zostáva SMTP komunikácia založená na tzv. hop-based modeli. To znamená, že bezpečnostné vlastnosti sa uplatňujú vždy len medzi dvoma konkrétnymi účastníkmi komunikácie (napr. klient → server alebo server → server), nie naprieč celým reťazcom doručenia správy.