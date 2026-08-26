# Barnehage-agent – Bydel Nordstrand

Sjekker automatisk [Oslo kommunes side for ledige barnehageplasser](https://www.oslo.kommune.no/barnehage/ledige-barnehageplasser/)
hver morgen og sender deg e-post **når listen for Bydel Nordstrand endrer seg**
(Søndre Nordstrand holdes utenfor). Kjører gratis på GitHub Actions – ingen egen server.

## Slik virker det

1. GitHub Actions kjører skriptet daglig kl. 06:00 UTC (ca. 07–08 norsk tid).
2. Skriptet henter siden og plukker ut plassene under «Bydel Nordstrand».
3. Det sammenligner med forrige kjøring (lagret i `state/nordstrand.json`).
4. Er det en endring, får du e-post. Er alt likt, skjer ingenting.

Du får også en bekreftelses-e-post ved aller første kjøring.

## Oppsett (ca. 10 min)

### 1. Lag repoet
Opprett et **privat** GitHub-repo og last opp disse filene (behold mappestrukturen):
```
check_barnehage.py
requirements.txt
.github/workflows/check.yml
README.md
```

### 2. Lag en Gmail-apppassord (for utsending)
E-post sendes via Gmail. Du trenger et *apppassord* (ikke ditt vanlige passord):
1. Slå på 2-trinns bekreftelse på Google-kontoen.
2. Gå til **Google-konto → Sikkerhet → Apppassord** og lag ett nytt.
3. Kopier det 16-tegns passordet.

(Vil du bruke en annen e-postleverandør enn Gmail, endre `smtp.gmail.com`
og porten i `check_barnehage.py`.)

### 3. Legg inn hemmeligheter
I repoet: **Settings → Secrets and variables → Actions → New repository secret**.
Legg inn disse fire:

| Navn         | Verdi                                             |
|--------------|---------------------------------------------------|
| `SMTP_USER`  | Gmail-adressen din (f.eks. `dittnavn@gmail.com`)  |
| `SMTP_PASS`  | Det 16-tegns apppassordet                          |
| `EMAIL_FROM` | Samme Gmail-adresse                                |
| `EMAIL_TO`   | Adressen som skal motta varslene                   |

### 4. Test
Gå til **Actions**-fanen → velg workflowen → **Run workflow** for å kjøre
manuelt med én gang. Du bør få bekreftelses-e-posten. Etter det går den av seg selv.

## Nyttig å vite

- **Endre tidspunkt:** juster `cron` øverst i `.github/workflows/check.yml`.
  Formatet er UTC. Eksempel: `0 5 * * *` = 05:00 UTC (~06–07 norsk tid).
- **Vil du ha varsel også når kjøringen feiler?** Slå på e-postvarsling for
  mislykkede workflows under GitHub → Settings → Notifications.
- **Tilstanden** committes tilbake ved hver kjøring (med et tidsstempel), noe som
  også holder repoet aktivt så GitHub ikke pauser den planlagte jobben.
- Får du feilmeldingen om at «Bydel Nordstrand» ikke ble funnet, har enten siden
  endret struktur, eller så laster den innholdet med JavaScript. Si fra, så lager
  jeg en variant som bruker en headless nettleser (Playwright).
