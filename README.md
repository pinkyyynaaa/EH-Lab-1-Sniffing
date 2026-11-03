# LAB01 – Network Sniffing: Capturing Plaintext HTTP Credentials with Wireshark

> **Disclaimer:** This walkthrough was performed in an isolated lab (two VMs I own) using dummy credentials (`test:test`). Do not sniff traffic on networks you do not own or have written permission to test.

## Table of Contents

1. [Objective](#1-objective)
2. [Concepts](#2-concepts)
3. [Lab Topology](#3-lab-topology)
4. [Step 1: Find the Server IP](#4-step-1-find-the-server-ip)
5. [Step 2: Run the Vulnerable Web App](#5-step-2-run-the-vulnerable-web-app)
6. [Step 3: Start Wireshark](#6-step-3-start-wireshark)
7. [Step 4: Simulate a Normal User Login](#7-step-4-simulate-a-normal-user-login)
8. [Step 5: Analyse the Capture](#8-step-5-analyse-the-capture)
9. [Step 6: Countermeasure Demo (HTTPS)](#9-step-6-countermeasure-demo-https)
10. [Findings](#10-findings)
11. [Lab Questions: Notes](#11-lab-questions-notes)
12. [Key Takeaways](#12-key-takeaways)

---

## 1. Objective

Show that **HTTP traffic is sent in plaintext**, so anyone who can observe the packets can read credentials. Then show how a **TLS/SSL wrapper (HTTPS)** protects **Confidentiality** in the CIA triad.

## 2. Concepts

| Term | Meaning |
|---|---|
| **Sniffing** | Passively observing network traffic. |
| **HTTP** | Application protocol with no built-in encryption. |
| **HTTPS** | HTTP wrapped in TLS. Content is encrypted in transit. |
| **Wireshark** | Free packet analyser used to capture and inspect traffic. |
| **CIA triad** | Confidentiality, Integrity, Availability. Sniffing attacks Confidentiality. |

> The "S" in HTTPS only secures the **network transport**. It does not fix application-layer flaws such as XSS, SQLi or SSTI. Those are out of scope here.

## 3. Lab Topology

```
 ┌──────────────────────┐        HTTP (port 5000)        ┌──────────────────────┐
 │  Windows 10 VM       │ ─────────────────────────────► │  Kali Linux VM       │
 │  "User" / browser    │                                │  Flask app (server)  │
 │  yyy.yyy.yyy.yyy     │ ◄───────────────────────────── │  xxx.xxx.xxx.xxx     │
 └──────────┬───────────┘                                └──────────────────────┘
            │
            └── Wireshark captures on Ethernet0
```

> IP addresses are masked for privacy: `xxx.xxx.xxx.xxx` = Kali server, `yyy.yyy.yyy.yyy` = Windows user. Use your own addresses.

## 4. Step 1: Find the Server IP

On Kali:

```bash
ip -br -4 a
```

`-br` gives brief output, `-4` shows IPv4 only, `a` is short for `addr`. Note the address of the main interface (e.g. `eth0`).

📸 *Screenshot: output of `ip -br -4 a`*

## 5. Step 2: Run the Vulnerable Web App

Save the lab's Flask app as `runme.py` on Kali (nano, vim or any editor). The important parts:

```python
@app.route("/login", methods=["POST"])
def login():
    user = request.form.get("username", "")
    pwd = request.form.get("password", "")
    return f"<h3>Authentication Processed</h3><p>User: <strong>{user}</strong> received over HTTP.</p>"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000, debug=False)
```

Why `host="0.0.0.0"`: `127.0.0.1` only accepts connections from the same machine. `0.0.0.0` listens on every interface, so the Windows VM can reach it.

Install Flask if missing, then run:

```bash
pip install flask
python runme.py
```

Expected output:

```
 * Serving Flask app 'runme'
 * Running on all addresses (0.0.0.0)
 * Running on http://xxx.xxx.xxx.xxx:5000
```

📸 *Screenshot: Flask running in the terminal*

## 6. Step 3: Start Wireshark

Install from <https://www.wireshark.org/download.html>.

- Choose the **correct interface**:
  - Server on Kali, capturing on Windows → **Ethernet0**
  - Capturing on Kali itself → **eth0** (use `lo` only if the browser is also on Kali)
- Loopback (`127.0.0.1`) only sees traffic from a machine to itself.
- Optional: set a **display filter** to cut noise:

```
tcp.port == 5000
```

(Display filters hide packets after capture. Capture filters, e.g. `port 5000`, drop packets before they are saved.)

📸 *Screenshot: Wireshark interface list*
📸 *Screenshot: live capture running*

## 7. Step 4: Simulate a Normal User Login

On the Windows VM, open a browser and go to:

```
http://xxx.xxx.xxx.xxx:5000
```

Enter dummy credentials:

```
Username: test
Password: test
```

Click **Sign In**. The server replies with *"Authentication Processed"*. Note the browser shows **"Not secure"**.

📸 *Screenshot: login page ("Not secure")*
📸 *Screenshot: "Authentication Processed" page*

## 8. Step 5: Analyse the Capture

Stop the capture and look for these packets (numbers will differ):

| Packet | What it is |
|---|---|
| `SYN`, `SYN-ACK`, `ACK` | TCP three-way handshake: the connection is established |
| `POST /login HTTP/1.1` | The browser submitting the login form |
| `HTTP/1.1 200 OK` | Server response |

**Read the credentials:**

1. Click the `POST /login` packet.
2. Expand **Hypertext Transfer Protocol → HTML Form URL Encoded**, or right-click → **Follow → TCP Stream**.
3. The body shows the credentials in clear text:

```
username=test&password=test
```

Useful details in the same stream:

- `Content-Type: application/x-www-form-urlencoded`: how HTML forms encode `key=value` pairs
- `User-Agent`: reveals the victim's browser and OS
- `Server: Werkzeug/... Python/...`: reveals the server stack

📸 *Screenshot: packet list with `POST /login` highlighted*
📸 *Screenshot: Follow TCP Stream showing `username=test&password=test`*

## 9. Step 6: Countermeasure Demo (HTTPS)

Generate a self-signed certificate on Kali (free, no purchase needed):

```bash
openssl req -x509 -newkey rsa:2048 -nodes -keyout key.pem -out cert.pem -days 30 -subj "/CN=lab.local"
```

Change the last line of `runme.py`:

```python
app.run(host="0.0.0.0", port=5000, debug=False, ssl_context=("cert.pem", "key.pem"))
```

Restart, then browse to `https://xxx.xxx.xxx.xxx:5000`. The browser warns about the self-signed certificate (expected). Accept it, log in again with `test:test`, and capture.

**Expected result:** Wireshark shows a TLS handshake followed by `Application Data` packets. The body is ciphertext, so `username=...&password=...` is **not** visible.

📸 *Screenshot: TLS handshake + `Application Data` packets*
📸 *Screenshot: Follow TCP Stream showing unreadable ciphertext*

### Going further: force HTTPS automatically

- **HSTS** header: `Strict-Transport-Security: max-age=31536000; includeSubDomains`
- **HTTP → HTTPS redirect** (301), for example with `Flask-Talisman`
- Browser policy: HTTPS-Only Mode (Firefox) or "Always use secure connections" (Chrome/Edge)

## 10. Findings

| | HTTP (port 5000) | HTTPS (TLS) |
|---|---|---|
| Browser indicator | Not secure | Padlock / certificate warning (self-signed) |
| Credentials in Wireshark | ✅ Readable | ❌ Encrypted |
| Metadata visible (IP, port, timing) | Yes | Yes |
| CIA property protected | None | Confidentiality (and integrity) |

## 11. Lab Questions: Notes

> Write your final answers in your own words. These are the key ideas to cover.

1. **Sniffing tool:** Wireshark.
2. **Sniff TLS-encrypted packets?** Capture yes, read contents no. Metadata (IPs, ports, sizes, handshake, SNI) remains visible.
3. **With the server's private RSA key?** Only decryptable if the session used static RSA key exchange. ECDHE (forward secrecy) and TLS 1.3 prevent it. `SSLKEYLOGFILE` is the alternative.
4. **Port 443 with no TLS configured:** the port number does not decide encryption, TLS does. A plain-HTTP server on 443 fails the TLS handshake. If a certificate exists (self-signed), TLS is active and Wireshark shows only ciphertext.
5. **Outside attacker on a switched network:** switches forward unicast frames only to the destination port, and routers do not forward other segments' frames. The attacker would need a MitM technique such as ARP spoofing (or MAC flooding / port mirroring).
6. **Forcing HTTPS automatically:** HSTS header, HTTP→HTTPS redirect (e.g. Flask-Talisman), browser HTTPS-Only policies, HSTS preload list.

## 12. Key Takeaways

- Plaintext protocols (HTTP, FTP, Telnet, LDAP) expose credentials to anyone on the path.
- Always use the TLS counterpart: **HTTP → HTTPS**, **LDAP → LDAPS**.
- Use strong cipher suites and modern TLS versions.
- Encryption in transit protects Confidentiality but does not replace secure application development.

## References

- Wireshark: <https://www.wireshark.org/>
- Cloudflare supported cipher suites: <https://developers.cloudflare.com/ssl/edge-certificates/additional-options/cipher-suites/supported-cipher-suites/>
