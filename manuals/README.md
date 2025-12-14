# Návod na prihlásenie sa do Mozilla Thunderbird prostredníctvom Postfix účtu
Východiskový stav je nainštalovaná a otvorená aplikácia Mozilla Thunderbird.


```
REÁLNE PROSTREDIE (server ↔ server)           LAB SCENÁR (klient → Postfix → Gmail)

┌──────────────────┐                         ┌──────────────────┐
│  Mailový klient  │                         │ Klient / Útočník │
│ (Outlook, web)   │                         │     (telnet)     │
└─────────┬────────┘                         └─────────┬────────┘
          │ SMTP (587)                                 │ SMTP (587)
          │                                            │ 
          ▼                                            ▼
┌──────────────────┐                         ┌──────────────────┐
│ SMTP server      │                         │ Postfix SMTP MTA │
│ (organizácia)    │                         │    (Docker)      │
└─────────┬────────┘                         └─────────┬────────┘
          │ server ↔ server SMTP (port 25)             │ klient → server SMTP (port 587)
          │ bez loginu                                 │ AUTH (gmail účet)
          │                                            │
          │ Overenie:                                  │ Overenie:
          │ - verejná IP                               │ - SMTP AUTH (login/heslo)
          │ - PTR / SPF                                │ - STARTTLS
          │ - DKIM                                     │
          │ - reputácia                                │
          ▼                                            ▼
┌──────────────────┐                         ┌──────────────────┐
│ SMTP server      │                         │ smtp.gmail.com   │
│ (cudzí server)   │                         │      :587        │
└─────────┬────────┘                         └─────────┬────────┘
          ▼                                            ▼
┌──────────────────┐                         ┌──────────────────┐
│ Príjemca e-mailu │                         │ Inbox príjemcu   │
│                  │                         │   (Gmail UI)     │
└──────────────────┘                         └──────────────────┘
```