# TLS certificate setup

Only required for **TLS Pro** and **TLS Ultra** modes. Plain Residential mode is transparent — no CA install needed.

## Why a CA install is needed

When you use TLS Pro or TLS Ultra, the engine **MITMs your HTTPS traffic** to apply the chosen TLS fingerprint (JA3 / JA4 / extension order / cipher list) at the handshake layer. Your client's outbound TLS terminates at our engine, not at the destination — we then open a separate, fingerprint-signed TLS connection to the destination using the profile you picked.

For your client to trust that intermediate connection, our **root CA must be in your trust store**.

For Residential mode, the proxy passes the CONNECT through transparently — your client's TLS handshake reaches the destination directly. No MITM, no CA install.

## What the CA is authorised to do

`NikaRootCA.crt` can ONLY sign certificates for traffic that **passes through our MITM**. We can't sign certificates for arbitrary origins — the CA is loaded into the engine's signer module, which only mints certs for the specific (CONNECT-target) hostnames customers are actively proxying through us.

In practice this means:
- If you install our CA, we can MITM your traffic when you route through `proxy.nikaproxy.com`. Not when you connect to anything else.
- The CA cannot be used to sign certs for sites you visit directly (without our proxy). Your normal browsing is unaffected.
- If you uninstall the CA, our MITM stops working (you'd see TLS errors on TLS Pro / Ultra requests). Plain Residential still works without the CA.

If you'd rather not install the CA at all, use the [REST scrape API](scrape-api.md) instead — the engine MITMs server-side and returns the destination's response in clear text, no client-side certificate involvement.

## Download

[/dashboard/certificate](https://nikaproxy.com/dashboard/certificate) — download `NikaRootCA.crt` (PEM format, ~1.5 KB).

The file is the same for all customers (it's the public certificate of our signing CA). It has no secret in it — it just identifies us as the issuer.

## Install — per OS

### macOS

```bash
sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain ~/Downloads/NikaRootCA.crt
```

Or via the GUI: open Keychain Access → drag NikaRootCA.crt into the "System" keychain → double-click it → expand "Trust" → set "When using this certificate" to "Always Trust".

### Linux (Debian / Ubuntu)

```bash
sudo cp ~/Downloads/NikaRootCA.crt /usr/local/share/ca-certificates/NikaRootCA.crt
sudo update-ca-certificates
```

### Linux (Fedora / RHEL / CentOS)

```bash
sudo cp ~/Downloads/NikaRootCA.crt /etc/pki/ca-trust/source/anchors/
sudo update-ca-trust
```

### Windows

Double-click `NikaRootCA.crt` → "Install Certificate" → "Local Machine" → "Place all certificates in the following store" → Browse → "Trusted Root Certification Authorities" → Next → Finish.

Or via PowerShell as administrator:

```powershell
Import-Certificate -FilePath "$env:USERPROFILE\Downloads\NikaRootCA.crt" `
                   -CertStoreLocation Cert:\LocalMachine\Root
```

## Install — per language / tool

For one-off scripts where a system-wide CA install is overkill:

### curl

```bash
curl --cacert NikaRootCA.crt -x "http://USER:PASS@proxy.nikaproxy.com:8080" \
     "https://example.com"
```

### Python (requests, httpx)

```python
import requests
r = requests.get(
    "https://example.com",
    proxies={"http": proxy, "https": proxy},
    verify="NikaRootCA.crt",
)
```

```python
import httpx
client = httpx.Client(proxy=proxy, verify="NikaRootCA.crt")
```

### Node.js

Node respects the `NODE_EXTRA_CA_CERTS` env var:

```bash
NODE_EXTRA_CA_CERTS=./NikaRootCA.crt node my_script.js
```

Or in code with `https.Agent`:

```javascript
import https from 'https'
import fs from 'fs'

const agent = new https.Agent({
  ca: [...https.globalAgent.options.ca ?? [], fs.readFileSync('./NikaRootCA.crt')]
})
```

### Go

```go
caCert, _ := os.ReadFile("NikaRootCA.crt")
caCertPool := x509.NewCertPool()
caCertPool.AppendCertsFromPEM(caCert)

client := &http.Client{
    Transport: &http.Transport{
        Proxy: http.ProxyURL(proxyURL),
        TLSClientConfig: &tls.Config{RootCAs: caCertPool},
    },
}
```

### PHP (cURL)

```php
curl_setopt($ch, CURLOPT_CAINFO, '/path/to/NikaRootCA.crt');
curl_setopt($ch, CURLOPT_PROXY,  'proxy.nikaproxy.com:8080');
```

### Java (JVM truststore)

```bash
keytool -import -trustcacerts -alias NikaRootCA -file NikaRootCA.crt \
        -keystore $JAVA_HOME/lib/security/cacerts -storepass changeit
```

### Browsers

Each major browser uses its own trust store, separate from the OS in some cases:

- **Chrome / Edge** — uses the OS trust store on macOS / Windows / Linux. Install OS-level (above) and restart the browser.
- **Firefox** — has its own trust store. Settings → Privacy & Security → Certificates → View Certificates → Authorities → Import → select `NikaRootCA.crt` → check "Trust this CA to identify websites" → OK.
- **Safari** — uses the macOS Keychain (see macOS instructions).

## Verifying the install

```bash
# Should print Issuer = NikaProxy
openssl s_client -proxy proxy.nikaproxy.com:8080 \
                 -proxy_user "nka_KEY:nkp_PWD" \
                 -connect httpbin.org:443 < /dev/null 2>/dev/null \
  | openssl x509 -noout -issuer -subject
```

If your system trusts our CA, the connection succeeds. If you see `unable to get local issuer certificate`, the CA isn't installed in the right trust store for your shell's openssl.

## Removing the CA

You can uninstall the CA anytime. Reverse the install command:

- **macOS:** `sudo security delete-certificate -c NikaProxy`
- **Linux (Debian):** `sudo rm /usr/local/share/ca-certificates/NikaRootCA.crt && sudo update-ca-certificates --fresh`
- **Linux (Fedora):** `sudo rm /etc/pki/ca-trust/source/anchors/NikaRootCA.crt && sudo update-ca-trust`
- **Windows:** Manage user certificates → Trusted Root Certification Authorities → Certificates → find NikaProxy → right-click → Delete

After removal, TLS Pro / Ultra modes stop working (you'll see TLS handshake errors). Residential mode still works.
