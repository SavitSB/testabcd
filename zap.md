// === Configuration ===
var TOKEN_URL      = 'https://api.example.com/auth/token';
var CLIENT_ID      = 'YOUR_CLIENT_ID';
var CLIENT_SECRET  = 'YOUR_CLIENT_SECRET';
var TOKEN_TTL      = 30;   // token lifetime in seconds
var REFRESH_BEFORE = 5;    // seconds before expiry to refresh

// === Internal state ===
var lastToken     = null;
var lastFetchTime = 0;

/**
 * Called by ZAP before sending each HTTP request.
 * Ensures the Authorization header has a fresh Bearer token.
 */
function sendingRequest(msg, initiator, helper) {
    var now = (new Date()).getTime() / 1000;

    // Refresh if we don’t have one, or it’s about to expire
    if (lastToken === null || (now - lastFetchTime) >= (TOKEN_TTL - REFRESH_BEFORE)) {
        lastToken     = fetchAccessToken();
        lastFetchTime = now;
    }

    // Inject into header
    if (lastToken) {
        msg.getRequestHeader().setHeader('Authorization', 'Bearer ' + lastToken);
    }
}

/**
 * Fetches a fresh access token via POST.
 * @returns {string} the new access_token (or empty on error)
 */
function fetchAccessToken() {
    try {
        // Build the request
        var tokenMsg = new org.parosproxy.paros.network.HttpMessage();
        var hdr      = tokenMsg.getRequestHeader();
        hdr.setMethod('POST');
        hdr.setURI(new org.apache.commons.httpclient.URI(TOKEN_URL, true));
        hdr.setHeader('Content-Type', 'application/x-www-form-urlencoded');

        // URL‑encoded body
        var body = 'grant_type=client_credentials'
                 + '&client_id='    + encodeURIComponent(CLIENT_ID)
                 + '&client_secret='+ encodeURIComponent(CLIENT_SECRET);
        tokenMsg.setRequestBody(body);

        // Send and receive
        var sender = new org.parosproxy.paros.network.HttpSender(
            org.parosproxy.paros.network.HttpSender.MANUAL_REQUEST_INITIATOR
        );
        sender.sendAndReceive(tokenMsg, true);

        // Parse JSON response
        var respBody = tokenMsg.getResponseBody().toString();
        var json     = JSON.parse(respBody);
        if (json.access_token) {
            print('[TokenRefresh] Obtained token: ' + json.access_token.substring(0,8) + '…');
            return json.access_token;
        } else {
            throw 'access_token missing';
        }
    } catch (e) {
        print('[TokenRefresh] ERROR fetching token: ' + e);
        return '';
    }
}

// Required no-op stub for HTTP Sender scripts
function responseReceived(msg, initiator, helper) {}
