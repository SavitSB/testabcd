# -*- coding: utf-8 -*-
"""
ZAP HTTP Sender (Python) script to transparently refresh a short‐lived JWT
during an Active Scan against the audit‐log endpoint.
"""

from org.parosproxy.paros.network import HttpSender
from org.apache.commons.httpclient import URI
import json
import time

# === GLOBAL TOKEN CACHE ===
token = None
expiryTime = 0

# === CONFIGURATION ===
# Token‐translator endpoint
tokenEndpoint = (
    "https://cmb-ib2b-dsp-pprod-eu.systems.uk.hsbc:8443"
    "/dsp/rest-sts/DSP_iB2B/iB2B_tokenTranslator?_action=translate"
)
# Basic Auth header for the translator
basicAuth = "Basic VE…<your‑base64‑creds>…="
# Credentials payload
username = "CMB0BKYC-PCBIS-MCT"
password = "gfA96q648NyKLR"


def sendingRequest(msg, initiator, helper):
    """
    Called for every outgoing request. We only act on Active Scanner requests
    to the audit‑log endpoint.
    """
    global token, expiryTime

    # Filter: only from the Active Scanner
    if initiator != HttpSender.ACTIVE_SCANNER_INITIATOR:
        return

    uri = msg.getRequestHeader().getURI().toString()
    if "/audit-log-rps-pib-gb-hbeu-internal-proxy/v1/audit-log" not in uri:
        return

    now = int(time.time() * 1000)
    # Refresh if no token or expired
    if token is None or now >= expiryTime:
        # Build a fresh token request
        tokenMsg = msg.cloneRequest()
        th = tokenMsg.getRequestHeader()
        th.setURI(URI(tokenEndpoint, False))
        th.setMethod("POST")
        th.setHeader("Authorization", basicAuth)
        th.setHeader("Content-Type", "application/json")

        payload = {
            "input_token_state": {
                "token_type": "CREDENTIAL",
                "username":   username,
                "password":   password
            },
            "output_token_state": {
                "token_type": "JWT"
            }
        }
        body = json.dumps(payload)
        tokenMsg.setRequestBody(body)
        th.setContentLength(len(body))

        # Send the token request (no redirects)
        helper.sendAndReceive(tokenMsg, False)

        try:
            resp = json.loads(tokenMsg.getResponseBody().toString())
            token = resp["issued_token"]
            # TTL 30s; refresh 3s early
            expiryTime = now + (30 * 1000) - (3 * 1000)
            print("[Refresh JWT] new token fetched, expires at %d" % expiryTime)
        except Exception as e:
            print("[Refresh JWT] error parsing token response:", e)
            return

    # Inject the (fresh) token
    msg.getRequestHeader().setHeader("X-HSBC-E2E-Trust-Token", token)


def responseReceived(msg, initiator, helper):
    """
    Called for incoming responses. We don’t need to do anything here.
    """
    pass
