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

#### 2.2. Postfix SMTP server
Postfix bol zvolený ako:
- reálne používaný SMTP server
- plne konfigurovateľný MTA
- vhodný na demonštráciu nesprávnych konfigurácií
- použitý image - **boky/postfix**

#### 2.3. Gmail SMTP Relay
Na odosielanie e-mailov do reálneho inboxu bol použitý Gmail SMTP relay.
Postup:
- vytvorenie účtu (alebo použitie existujúceho)
- generovanie App Password pre SMTP prístup
- konfigurácia Postfix relay hosta

#### 2.4. Telnet
Telnet bol použitý ako:
- jednoduchý TCP klient
- nástroj na manuálne odosielanie SMTP príkazov
- spôsob demonštrácie plaintext komunikácie

#### 2.5. Wireshark
Wireshark bol použitý na:
- zachytávanie SMTP komunikácie
- demonštráciu plaintext prenosu údajov
- porovnanie komunikácie pred a po zapnutí TLS

#### 2.6. Mozzila Thunderbird
Mozilla Thunderbird bol použitý ako plnohodnotný e-mailový klient na simuláciu
legitímneho používateľa, ktorý sa autentifikuje voči SMTP serveru pomocou mena
a hesla.

### 3. Scenáre útokov a testovania
#### 3.1. Scenár 1 - Open relay (nesprávna politika relaying-u)
Podmienky:
- trusted IP adresy nepotrebujú autentifikáciu

Náš Postfix server nakonfigurujeme nasledovne a naštartujeme Docker kontajner.

```
# docker-compose.yml

services:
  postfix:
    image: boky/postfix
    container_name: postfix-gmail
    restart: unless-stopped
    ports:
      - "25:25"
      - "587:587"
    environment:
      # Spam friendly
      ALLOW_EMPTY_SENDER_DOMAINS: "yes"

      # TRUST TOO MUCH → weakness (ask ChatGPT if weakness or vulnerability)
      POSTFIX_mynetworks: "192.168.0.0/16, 172.16.0.0/12, 172.18.0.0/16"

      # Classic open relay logic
      POSTFIX_smtpd_relay_restrictions: "permit_mynetworks,reject_unauth_destination"
      POSTFIX_smtpd_recipient_restrictions: "permit_mynetworks,reject"

      # Postfix -> Gmail relay settings
      RELAYHOST: "[smtp.gmail.com]:587"
      RELAYHOST_USERNAME: "${GMAIL_USER}"
      RELAYHOST_PASSWORD: "${GMAIL_PASSWORD}"
      RELAYHOST_TLS_LEVEL: "encrypt"
```

Otvoríme si terminál alebo iný príkazový riadok, ktorý podporuje telnet a spustíme nasledovnú sériu príkazov.

```
# terminál/príkazový riadok

telnet 192.168.0.52 587

EHLO bit.demo
MAIL FROM:<bitdemo25@gmail.com>
RCPT TO:<xvaliceks@stuba.sk>
DATA
To: xvaliceks@stuba.sk
From: bitdemo25@gmail.com
Subject: Step 1

Toto je krok cislo 1 v nasom deme
.
QUIT
```

Útok prebehol úspešne, výsledkom je nový e-mail v schránke koncového používateľa. Nepotrebovali sme sa pritom ani autentifikovať. Tento typ útoku môže teda pri súčasnej konfigurácii vykonať ktokoľvek s prístupom do siete.

![Open relay útok](images/screenshot1.png)
![Výsledok open relay útoku](images/screenshot2.png)

#### 3.2. Scenár 2 – Vynútenie autentifikácie (AUTH)
Zmena konfigurácie:
- vypnutie IP trust-u
- zapnutie SMTP AUTH

Vytvoríme docker-compose.yml
```
# docker-compose.yml

services:
  postfix:
    image: boky/postfix
    container_name: postfix-gmail
    restart: unless-stopped
    ports:
      - "25:25"
      - "587:587"
    environment:
      # Spam friendly
      ALLOW_EMPTY_SENDER_DOMAINS: "yes"

      POSTFIX_mynetworks: "127.0.0.0/8, 172.16.0.0/12"

      # Require SMTP authentication
      POSTFIX_smtpd_sasl_auth_enable: "yes"
      POSTFIX_smtpd_sasl_security_options: "noanonymous"
      POSTFIX_smtpd_sasl_local_domain: ""

      # Relay policy: auth required for non-local domains
      POSTFIX_smtpd_relay_restrictions: "permit_sasl_authenticated,reject_unauth_destination"
      POSTFIX_smtpd_recipient_restrictions: "permit_sasl_authenticated,reject"

      # Postfix -> Gmail relay settings
      RELAYHOST: "[smtp.gmail.com]:587"
      RELAYHOST_USERNAME: "${GMAIL_USER}"
      RELAYHOST_PASSWORD: "${GMAIL_PASSWORD}"
      RELAYHOST_TLS_LEVEL: "encrypt"
```

Následne musíme rozbehať databázu vo vnútri kontajnera, aby nám bolo umožnené vytvoriť usera, s ktorým sa budeme následne vedieť prihlásiť do Postfixu.

Na to nám slúžia nasledovné príkazy:
  
```
# terminál/príkazový riadok

docker exec -it postfix-gmail sh


rm -f /etc/sasldb2
rm -f /var/spool/postfix/etc/sasldb2
HOST=$(postconf -h myhostname)
echo $HOST

# vytvoríme usera mario
saslpasswd2 -c -u "$HOST" mario

# dostaneme výzvu na zadanie hesla 
# krokodil123 je naše heslo
krokodil123

# dostaneme výzvu na potvrdenie hesla
krokodil123

mkdir -p /var/spool/postfix/etc
cp /etc/sasldb2 /var/spool/postfix/etc/sasldb2
chown postfix:postfix /etc/sasldb2
chmod 600 /etc/sasldb2
chown postfix:postfix /var/spool/postfix/etc/sasldb2
chmod 600 /var/spool/postfix/etc/sasldb2

postfix stop
postfix start
```

Pokúsime sa spáchať útok ako predtým, čím si overíme, či sa nastavenia autentifikácie aplikovali správne.

```
# terminál/príkazový riadok

telnet 192.168.0.52 587

EHLO bit.demo
MAIL FROM:<bitdemo25@gmail.com>
RCPT TO:<xvaliceks@stuba.sk>
DATA
To: xvaliceks@stuba.sk
From: bitdemo25@gmail.com
Subject: Step 2

Toto je krok cislo 2 v nasom deme
.
QUIT
```

Vráti nás to s chybou, že sa musíme overiť.

![Pokus prihlásenia sa bez overenia po vynútení autentifikácie serverom](images/screenshot3.png)

Pokúsime sa teda, prihlásiť pomocou údajov, ktoré sme si vytvorili vyššie v databáze vo vnútri kontajnera.
V našom prípade, to budú údaje username:password → mario:krokodil123. Na prihlásenie sa do účtu použijeme klienta Mozilla Thunderbird. Návod, ako sa prihlásiť do klienta nájdeme [tu](manuals/README.md)

Po úspešnom prihlásení sa do Mozilly Thunderbird, pošleme skúšobný mail.

![Email z Mozzila Thunderbird → Postfix → smtp.gmail:587 → xvaliceks inbox](images/screenshot9.png)

Pokúsime sa teda overiť, musíme mať však na mysli, že prostredníctvom telnetu dostaneme výzvu na zadanie mena a hesla zakódovanú v base64, a rovnako tak zakódované musia byť meno a heslo aj keď ho do terminálu zadávame (to sú tie divné dva riadky po AUTH LOGIN):

```
# terminál/príkazový riadok

telnet 192.168.0.52 587

EHLO bit.demo
AUTH LOGIN
bWFyaW8=
a3Jva29kaWwxMjM=
MAIL FROM:<bitdemo25@gmail.com>
RCPT TO:<xvaliceks@stuba.sk>
DATA
To: xvaliceks@stuba.sk
From: bitdemo25@gmail.com
Subject: Step 2

Toto je krok cislo 2 v nasom deme
.
QUIT
```

Vďaka nezabezpečenej komunikácii vieme sniffnúť heslo z wiresharku:


#### 3.3. Scenár 3 – Chýbajúce TLS (cleartext credentials)
```
services:
  postfix:
    image: boky/postfix
    container_name: postfix-gmail
    restart: unless-stopped
    ports:
      - "25:25"
      - "587:587"
    environment:
      # Spam friendly
      ALLOW_EMPTY_SENDER_DOMAINS: "yes"

      POSTFIX_mynetworks: "127.0.0.0/8, 172.16.0.0/12"

      # Require SMTP authentication
      POSTFIX_smtpd_sasl_auth_enable: "yes"
      POSTFIX_smtpd_sasl_security_options: "noanonymous"
      POSTFIX_smtpd_sasl_local_domain: ""

      # Relay policy: auth required for non-local domains
      POSTFIX_smtpd_relay_restrictions: "permit_sasl_authenticated,reject_unauth_destination"
      POSTFIX_smtpd_recipient_restrictions: "permit_mynetworks,permit_sasl_authenticated,reject"

      # Postfix -> Gmail relay settings
      RELAYHOST: "[smtp.gmail.com]:587"
      RELAYHOST_USERNAME: "${GMAIL_USER}"
      RELAYHOST_PASSWORD: "${GMAIL_PASSWORD}"
      RELAYHOST_TLS_LEVEL: "encrypt"

      # TLS inbound enforcement
      SMTPD_USE_TLS: "yes"
      SMTPD_TLS_SECURITY_LEVEL: "encrypt"
      ALLOW_INSECURE_AUTH: "false"

      POSTFIX_smtpd_delay_reject: "no"


      # # Enable SASL authentication
      # POSTFIX_smtpd_sasl_auth_enable: "yes"
      # POSTFIX_smtpd_sasl_security_options: "noanonymous"
      # POSTFIX_smtpd_sasl_local_domain: ""

      # # Allow relay after authentication
      # POSTFIX_smtpd_relay_restrictions: "permit_sasl_authenticated,reject_unauth_destination"

      # POSTFIX_smtpd_recipient_restrictions: "permit_sasl_authenticated,reject"
      
      # ALLOW_INSECURE_AUTH: "true"
      # SMTP_AUTH_METHODS: "plain,login"

      # Disable TLS inbound
      # SMTPD_USE_TLS: "no"


      # AUTH METHODS (TLS not enforced yet)
      # ALLOW_INSECURE_AUTH: "true"
      # SMTP_AUTH_METHODS: "plain,login"
```   


## Záver
Praktická časť práce ukazuje, že moderné e-mailové systémy sú vo väčšine prípadov správne zabezpečené a dokážu efektívne eliminovať známe slabiny SMTP protokolu. Zároveň však demonštruje, že tieto mechanizmy fungujú len v prípade ich korektného nasadenia a vynútenia na všetkých úrovniach komunikácie.