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

## Praktická časť – Analýza SMTP komunikácie
### 1. Architektúra testovacieho prostredia
#### 1.1. Cieľ infraštruktúry
Cieľom testovacieho prostredia je simulovať realistický scenár SMTP komunikácie, kde:
- klient/útočník komunikuje so SMTP submission serverom
- SMTP server následne relaye správu do reálneho e-mailového systému
- demonštruje sa, že slabina v prvom kroku môže viesť k doručeniu e-mailu koncovému používateľovi
#### 1.2. Logická architektúra
```
┌──────────────────┐
│ Klient / Útočník │
│     (telnet)     │
└─────────┬────────┘
          │ SMTP (587)
          │ (bez TLS / bez AUTH / zlá politika)
          ▼
┌──────────────────┐
│ Postfix SMTP MTA │
│     (Docker)     │
└─────────┬────────┘
          │ SMTP + TLS
          │  (relay)
          ▼
┌──────────────────┐
│smtp.gmail.com:587│
│    (bezpečné)    │
└─────────┬────────┘
          ▼
┌──────────────────┐
│  Inbox príjemcu  │
│    (Gmail UI)    │
└──────────────────┘
```
### 2. Použité technológie a nástroje
#### 2.1. Docker Compose
Docker Compose bol použitý na:
- rýchle nasadenie SMTP servera
- izoláciu testovacieho prostredia
- jednoduché opakovanie experimentov

## Záver
Praktická časť práce ukazuje, že moderné e-mailové systémy sú vo väčšine prípadov správne zabezpečené a dokážu efektívne eliminovať známe slabiny SMTP protokolu. Zároveň však demonštruje, že tieto mechanizmy fungujú len v prípade ich korektného nasadenia a vynútenia na všetkých úrovniach komunikácie.