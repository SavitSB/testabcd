#!/usr/bin/env python3
# -*- coding: utf-8 -*-
#
# SQLmap tamper: refresh 30s JWT before each request,
# using SQLmap’s own Request API (no urllib).

import time
import json

# tamper priority
try:
    from lib.core.enums import PRIORITY
    __priority__ = PRIORITY.NORMAL
except ImportError:
    __priority__ = 0

# ─── CONFIG ─────────────────────────────────────────────────────────────────────
# Your translate endpoint (no stray spaces!)
TOKEN_URL = (
    "https://cmb-ib2b-dsp-pprod-eu.systems.uk.hsbc:8443"
    "/dsp/rest-sts/DSP_iB2B/iB2B_tokenTranslator?_action=translate"
)

# Exactly the Basic header you saw working
AUTH_HEADER = "Basic VEVPRESWOnJhNzQzNzgw"

# JSON body creds
USERNAME = "CMBOBKYC-PCBIS-MCT"
PASSWORD = "gfA96qG48NyKLR"
# ────────────────────────────────────────────────────────────────────────────────

_cached_token = None
_next_refresh = 0.0

def _fetch_new_token():
    """
    POST via SQLmap’s Request.getPage, parse the JSON, cache JWT,
    schedule next refresh at now+25s (so we never send an expired 30s token).
    """
    global _cached_token, _next_refresh

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
    data = json.dumps(body)

    headers = {
        "Content-Type":  "application/json",
        "Accept":        "*/*",
        "Authorization": AUTH_HEADER
    }

    # use SQLmap's HTTP engine
    from lib.request import Request

    # Request.getPage returns (body, headers, status)
    page, resp_headers, status = Request.getPage(
        url=TOKEN_URL,
        httpMethod="POST",
        data=data,
        headers=headers
    )

    if status != 200:
        raise RuntimeError(f"[auth_token_refresh] token endpoint returned HTTP {status}")

    js = json.loads(page)
    token = js.get("issued_token")
    if not token:
        raise RuntimeError("[auth_token_refresh] no 'issued_token' in JSON response")

    _cached_token = token
    _next_refresh = time.time() + 25
    print(f"[auth_token_refresh] fetched new token, next refresh at {_next_refresh:.0f}")
    return token

def dependencies():
    """
    Called once at startup; prime the token cache.
    """
    try:
        _fetch_new_token()
    except Exception as e:
        # visible with -v 3
        print(f"[auth_token_refresh] initial token fetch failed: {e!r}")

def tamper(payload, **kwargs):
    """
    Called before each HTTP request.  Refresh if needed, then overwrite header.
    """
    global _cached_token, _next_refresh

    # if no token or past refresh time, grab a new one
    if not _cached_token or time.time() >= _next_refresh:
        try:
            _fetch_new_token()
        except Exception as e:
            print(f"[auth_token_refresh] token refresh error: {e!r}")

    # inject into outgoing headers
    hdrs = kwargs.get("headers")
    if isinstance(hdrs, dict) and _cached_token:
        hdrs["X-HSBC-E2E-Trust-Token"] = _cached_token

    # leave the SQL injection payload unchanged
    return payload
