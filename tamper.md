#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
SQLMap tamper script: always pull a fresh JWT and set it
in X‑HSBC‑E2E‑Trust‑Token for each request.
"""

from lib.core.enums import PRIORITY
import json
import urllib.request

__priority__ = PRIORITY.NORMAL

# ——— CONFIG ———
TOKEN_ENDPOINT = (
    "https://cmb-ib2b-dsp-pprod-eu.systems.uk.hsbc:8443"
    "/dsp/rest-sts/DSP_iB2B/iB2B_tokenTranslator?_action=translate"
)
BASIC_AUTH = "Basic VE…<your‑base64‑creds>…="
USERNAME   = "CMB0BKYC-PCBIS-MCT"
PASSWORD   = "gfA96q648NyKLR"


def get_token():
    """Fetch a fresh JWT from the translator service."""
    payload = json.dumps({
        "input_token_state": {
            "token_type": "CREDENTIAL",
            "username":   USERNAME,
            "password":   PASSWORD
        },
        "output_token_state": {
            "token_type": "JWT"
        }
    }).encode("utf-8")

    req = urllib.request.Request(TOKEN_ENDPOINT, data=payload, method="POST")
    req.add_header("Authorization", BASIC_AUTH)
    req.add_header("Content-Type", "application/json")

    with urllib.request.urlopen(req) as resp:
        data = json.loads(resp.read().decode("utf-8"))
        return data.get("issued_token")


def tamper(payload, **kwargs):
    """
    Called before each HTTP request. Grabs a fresh JWT and
    injects it into the headers dict that SQLMap will send.
    """
    token = get_token()
    headers = kwargs.get("headers")
    if headers is not None and token:
        headers["X-HSBC-E2E-Trust-Token"] = token
    return payload
