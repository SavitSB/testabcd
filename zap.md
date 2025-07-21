// === Configuration ===
// Change this to the History Reference ID of your original token request
var TOKEN_HISTORY_REF_ID = 42;

var TOKEN_HEADER    = 'X-Custom-Token';  // your custom auth header
var TOKEN_TTL       = 30;                // token lifetime in seconds
var REFRESH_BEFORE  = 5;                 // refresh this many seconds before expiry

// === Internal state ===
var lastToken      = null;
var lastFetchTime  = 0;

/**
 * Called before every request.
 * 1. Removes any old headers to avoid duplicates
 * 2. Refreshes the token if needed (uses History entry)
 * 3. Injects the fresh token into your custom header
 */
function sendingRequest(msg, initiator, helper) {
    var now = (new Date()).getTime() / 1000;

    // 1) Remove old headers
    msg.getRequestHeader().removeHeader(TOKEN_HEADER);
    msg.getRequestHeader().removeHeader('Authorization');

    // 2) Refresh token if we don't have one or it's about to expire
    if (lastToken === null || (now - lastFetchTime) >= (TOKEN_TTL - REFRESH_BEFORE)) {
        lastToken     = fetchTokenFromHistory();
        lastFetchTime = now;
    }

    // 3) Inject fresh token
    if (lastToken) {
        msg.getRequestHeader().setHeader(TOKEN_HEADER, lastToken);
    }
}

/**
 * Fetches a fresh token by re‑sending the original token request from History.
 * @returns {string} access_token or '' on error
 */
function fetchTokenFromHistory() {
    try {
        var model      = org.parosproxy.paros.model.Model.getSingleton();
        var session    = model.getSession();
        var historyRef = session.getHistoryReference(TOKEN_HISTORY_REF_ID);

        if (!historyRef) {
            throw 'No history entry found for ID ' + TOKEN_HISTORY_REF_ID;
        }

        // Clone the exact original message (headers + JSON body)
        var baseMsg  = historyRef.getHttpMessage();
        var tokenMsg = baseMsg.cloneRequest();

        // Send it
        var sender = new org.parosproxy.paros.network.HttpSender(
            org.parosproxy.paros.network.HttpSender.MANUAL_REQUEST_INITIATOR
        );
        sender.sendAndReceive(tokenMsg, true);

        // Parse out the token
        var resp = tokenMsg.getResponseBody().toString();
        var json = JSON.parse(resp);
        if (!json.access_token) {
            throw 'access_token missing in response';
        }

        print('[TokenRefresh] Retrieved token: ' + json.access_token.substring(0,8) + '…');
        return json.access_token;

    } catch (e) {
        print('[TokenRefresh] ERROR fetching token: ' + e);
        return '';
    }
}

// Required stub (no-op)
function responseReceived(msg, initiator, helper) {}
