#!/usr/bin/env python3
"""Sjekker Oslo kommunes side for ledige barnehageplasser i Bydel Nordstrand
og sender e-postvarsel når listen endrer seg."""

import json
import os
import re
import ssl
import smtplib
import sys
from datetime import datetime, timezone
from email.message import EmailMessage
from pathlib import Path

import requests
from bs4 import BeautifulSoup

URL = "https://www.oslo.kommune.no/barnehage/ledige-barnehageplasser/"
BYDEL = "Bydel Nordstrand"          # NB: matcher IKKE "Bydel Søndre Nordstrand"
STATE_FILE = Path("state/nordstrand.json")
HEADERS = {"User-Agent": "barnehage-agent/1.0 (personlig varsling)"}


def hent_side() -> str:
    r = requests.get(URL, headers=HEADERS, timeout=30)
    r.raise_for_status()
    return r.text


def parse_nordstrand(html: str) -> dict:
    """Trekker ut plassene som er listet under 'Bydel Nordstrand'."""
    soup = BeautifulSoup(html, "html.parser")

    heading = None
    for h in soup.find_all(re.compile(r"^h[1-6]$")):
        # startswith ekskluderer 'Bydel Søndre Nordstrand' automatisk
        if h.get_text(" ", strip=True).startswith(BYDEL):
            heading = h
            break

    if heading is None:
        raise RuntimeError(
            "Fant ikke overskriften 'Bydel Nordstrand'. Siden kan ha endret "
            "struktur, eller innholdet lastes med JavaScript (se README)."
        )

    # Oppdateringsdato står gjerne i parentes i overskriften
    m = re.search(r"oppdatert\s+([^)]+)", heading.get_text(" ", strip=True), re.I)
    dato = m.group(1).strip() if m else "ukjent dato"

    level = int(heading.name[1])
    plasser = []
    for el in heading.find_all_next():
        navn = el.name or ""
        # stopp ved neste overskrift på samme eller høyere nivå
        if re.fullmatch(r"h[1-6]", navn) and int(navn[1]) <= level:
            break
        if navn == "li":
            plasser.append(el.get_text(" ", strip=True))

    return {"dato": dato, "plasser": plasser}


def last_state() -> dict | None:
    if STATE_FILE.exists():
        return json.loads(STATE_FILE.read_text(encoding="utf-8"))
    return None


def lagre_state(data: dict) -> None:
    STATE_FILE.parent.mkdir(parents=True, exist_ok=True)
    STATE_FILE.write_text(
        json.dumps(data, ensure_ascii=False, indent=2), encoding="utf-8"
    )


def send_epost(emne: str, tekst: str) -> None:
    msg = EmailMessage()
    msg["From"] = os.environ["EMAIL_FROM"]
    msg["To"] = os.environ["EMAIL_TO"]
    msg["Subject"] = emne
    msg.set_content(tekst)
    ctx = ssl.create_default_context()
    with smtplib.SMTP_SSL("smtp.gmail.com", 465, context=ctx) as s:
        s.login(os.environ["SMTP_USER"], os.environ["SMTP_PASS"])
        s.send_message(msg)


def main() -> None:
    data = parse_nordstrand(hent_side())
    forrige = last_state()

    if data["plasser"]:
        status = "\n".join(f"• {p}" for p in data["plasser"])
    else:
        status = "Ingen ledige plasser."

    if forrige is None:
        # Første kjøring: bekreft at overvåkingen er i gang
        send_epost(
            "Barnehage-varsling er i gang – Bydel Nordstrand",
            f"Overvåkingen er satt opp og kjører nå daglig.\n\n"
            f"Status akkurat nå (oppdatert {data['dato']}):\n{status}\n\n{URL}",
        )
        print("Første kjøring – bekreftelse sendt.")
    elif forrige.get("plasser") != data["plasser"]:
        # Listen har endret seg siden sist
        send_epost(
            "Endring i ledige barnehageplasser – Bydel Nordstrand",
            f"Listen for Bydel Nordstrand har endret seg "
            f"(oppdatert {data['dato']}):\n\n{status}\n\n{URL}",
        )
        print("Endring oppdaget – e-post sendt.")
    else:
        print("Ingen endring siden sist.")

    data["sist_sjekket"] = datetime.now(timezone.utc).isoformat(timespec="seconds")
    lagre_state(data)


if __name__ == "__main__":
    try:
        main()
    except Exception as e:
        print(f"FEIL: {e}", file=sys.stderr)
        sys.exit(1)
