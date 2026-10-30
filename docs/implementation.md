# HR Web API Deployment — DNS, NAT, IIS & Automated TLS Certificate Management
 
> Production identifiers, IP addresses, company-specific hostnames, account details, and other internal infrastructure information have been removed or replaced with placeholders.

## Overview

This mini infrastructure project involved securely exposing an internally hosted HR Web API to remote tablet clients over HTTPS.

The HR platform architecture changed so that employee card-scan events could be sent directly to an HR Web API hosted on the company's own Windows Server environment and then reach the internal ERP/backend, instead of following the previous intermediary flow through the vendor's infrastructure.

The initial API endpoint was available only internally:

```text
http://<IIS_PRIVATE_IP>

```

The target state was a publicly resolvable and encrypted endpoint:

```text
https://hr-api.example.com
```

The implementation covered:

```text
Public DNS
    ↓
Public IPv4
    ↓
Firewall / NAT
    ↓
Windows Server
    ↓
Microsoft IIS
    ↓
HR Web API
    ↓
ERP / Internal Database
```

---

# Step 1 – Network Identification & Port Forwarding

## 1.1 Identify the server network configuration

The Windows VPS hosting the HR Web API was inspected with:

```cmd
ipconfig
```

The relevant values showed that the server used a private IPv4 address and a private default gateway:

```text
IPv4 Address:     <IIS_PRIVATE_IP>
Subnet Mask:      <SUBNET_MASK>
Default Gateway:  <PRIVATE_GATEWAY>
```

This confirmed that the server was not directly addressed with a public IP and was located behind a gateway/NAT device.

The architecture was therefore:

```text
Internet
   │
   ▼
Public IPv4
   │
   ▼
Firewall / Router
   │
   ▼
Windows VPS
   │
   ▼
Microsoft IIS
   │
   ▼
HR Web API
```

## 1.2 Identify the public IPv4 address

From PowerShell, the public egress IP was checked with:

```powershell
(Invoke-WebRequest -Uri "https://api.ipify.org").Content
```

The returned address is represented in this public document as:

```text
<PUBLIC_IPV4>
```

## 1.3 Confirm the gateway / NAT device

The gateway was confirmed to be running MikroTik RouterOS.

The edge router/firewall was managed by the infrastructure provider, so the NAT rules were implemented by the responsible provider administrator rather than directly by me.

The requested TCP mappings were conceptually:

```text
<PUBLIC_IPV4>:80
        ↓
<IIS_PRIVATE_IP>:80
```

and:

```text
<PUBLIC_IPV4>:443
        ↓
<IIS_PRIVATE_IP>:443
```

TCP/80 was required for the HTTP-01 ACME validation flow used later by Let's Encrypt.

TCP/443 was required for the final HTTPS endpoint.

---

# Step 2 – DNS Configuration with Cloudflare

## 2.1 Create the public hostname

The existing company DNS zone was already managed through Cloudflare.

A dedicated hostname was created for the API. In this public version the production hostname has been replaced with:

```text
hr-api.example.com
```

The Cloudflare DNS record was configured conceptually as:

```text
Type:           A
Name:           hr-api
IPv4 address:   <PUBLIC_IPV4>
Proxy status:   DNS Only
TTL:            Auto
```

The resulting DNS flow was:

```text
hr-api.example.com
        ↓
A Record
        ↓
<PUBLIC_IPV4>
```

## 2.2 Why an A record was used

An A record was appropriate because the goal was to map a hostname directly to an IPv4 address:

```text
Hostname
   ↓
A Record
   ↓
IPv4 Address
```

## 2.3 Why Cloudflare was set to DNS Only

The record was intentionally configured as **DNS Only** rather than proxied.

This kept the initial network path simple and transparent:

```text
Client
   ↓
Cloudflare DNS lookup
   ↓
<PUBLIC_IPV4>
   ↓
Firewall / NAT
   ↓
Windows IIS
```

This was useful for troubleshooting and ensured that the client connected directly to the IIS-hosted service rather than terminating TLS at the Cloudflare edge.

## 2.4 Verify DNS resolution

DNS resolution was tested with:

```cmd
nslookup hr-api.example.com
```

Expected result:

```text
Name:    hr-api.example.com
Address: <PUBLIC_IPV4>
```

This confirmed that the hostname resolved to the intended public IP.

## 2.5 Verify external HTTP connectivity

From an external client, TCP/80 was tested with:

```powershell
Test-NetConnection hr-api.example.com -Port 80
```

Expected result:

```text
TcpTestSucceeded : True
```

The HTTP endpoint was also opened in a browser:

```text
http://hr-api.example.com
```

The HR Web API returned its expected welcome/health response.

This verified the full path:

```text
DNS
 ↓
Public IP
 ↓
NAT / Port Forwarding
 ↓
Windows VPS
 ↓
IIS
 ↓
HR Web API
```

---

# Step 3 – SSL/TLS Certificate with Let's Encrypt & simple-acme

## 3.1 Check for an existing server certificate

Before issuing a new certificate, the IIS server was checked for an existing usable certificate.

In IIS Manager:

```text
Server
→ Server Certificates
```

No suitable certificate was installed.

The Windows certificate store was also checked from PowerShell:

```powershell
Get-ChildItem Cert:\LocalMachine\My |
Select-Object Subject, Issuer, NotAfter, HasPrivateKey, Thumbprint
```

No suitable certificate/private key pair was available for the API hostname.

The system was also checked for an existing ACME/certificate management client.

## 3.2 Distinguish Cloudflare edge certificates from IIS certificates

The existing Cloudflare zone already had a wildcard Universal Edge Certificate covering the company domain.

However, a Cloudflare **Edge Certificate** is presented by Cloudflare to clients when traffic is proxied through Cloudflare. It is not the same as a certificate installed on the IIS origin server with its private key.

Because the API record was configured as **DNS Only**, clients connected directly to the IIS server:

```text
Client
   ↓
hr-api.example.com
   ↓
<PUBLIC_IPV4>
   ↓
Firewall / NAT
   ↓
IIS
```

Therefore IIS required its own publicly trusted TLS certificate.

## 3.3 Choose Let's Encrypt and simple-acme

A dedicated Let's Encrypt certificate was issued for:

```text
hr-api.example.com
```

The ACME client used was:

```text
simple-acme
```

It was extracted to a permanent directory on the server:

```text
C:\Program Files\simple-acme
```

and executed with administrative privileges:

```text
wacs.exe
```

## 3.4 Create the certificate with full options

In simple-acme:

```text
M: Create certificate (full options)
```

Manual hostname input was selected:

```text
2: Manually enter host names
```

The production hostname was entered. In this public version:

```text
hr-api.example.com
```

A single certificate was requested because only one hostname was required:

```text
4: Single certificate
```

## 3.5 HTTP-01 domain validation

Let's Encrypt had to verify control of the requested hostname.

The selected validation method was:

```text
1: [http] Let simple-acme answer HTTP validation request
```

The validation path conceptually looked like:

```text
Let's Encrypt
      ↓
http://hr-api.example.com/.well-known/acme-challenge/...
      ↓
<PUBLIC_IPV4>:80
      ↓
Firewall / NAT
      ↓
<IIS_PRIVATE_IP>:80
      ↓
simple-acme
```

This worked because external TCP/80 connectivity had already been verified.

Successful validation returned:

```text
Authorization result: valid
```

## 3.6 Key type

RSA was selected:

```text
2: Generate an RSA public/private key pair
```

The goal was broad compatibility with Windows IIS and the expected client devices.

## 3.7 Store the certificate in Windows

The certificate was stored in the Windows Local Computer certificate store:

```text
2: Add to Windows Certificate Store (Local Computer)
```

and specifically in:

```text
1: [WebHosting] - Dedicated store for IIS
```

After issuance, the certificate was visible in IIS Server Certificates.

## 3.8 Export a password-protected PFX

A second storage step was configured:

```text
5: Create PFX/PKCS12 archive file
```

The PFX was stored in an administrator-controlled directory, for example:

```text
C:\Program Files\simple-acme\exports
```

The PFX was password-protected.

No password, secret value, private key, certificate thumbprint, ACME account identifier, or other credential is included in this public repository.

The PFX/PKCS#12 archive contains the certificate material required for protected export/import scenarios, including the private key.

## 3.9 Avoid automatic IIS changes during initial issuance

During the first certificate issuance, the IIS installation step was intentionally disabled:

```text
3: No (additional) installation steps
```

At that stage the application vendor/support team was responsible for completing the initial production HTTPS binding.

This allowed the certificate to be issued and stored without changing the existing IIS application configuration prematurely.

## 3.10 Certificate profile and ACME account

The Let's Encrypt `classic` profile was selected and an ACME account was created with an administrative contact email.

No account identifiers or contact details are included in this public version.

## 3.11 Initial renewal configuration

simple-acme automatically created a renewal configuration and a Windows Scheduled Task.

The high-level renewal flow was:

```text
Windows Scheduled Task
        ↓
simple-acme --renew
        ↓
Renewal eligibility check
        ↓
HTTP-01 validation
        ↓
New Let's Encrypt certificate
        ↓
Windows WebHosting Store
        +
Updated PFX
```

Because `wacs.exe` is referenced by the scheduled task, simple-acme was kept in a permanent installation directory rather than a temporary Downloads folder.

---

# Step 4 – IIS HTTPS Binding

After the certificate had been issued and installed, the application support team completed the initial HTTPS binding in IIS.

The target configuration was conceptually:

```text
Protocol:     HTTPS
Port:         443
Hostname:     hr-api.example.com
Certificate:  Let's Encrypt certificate for hr-api.example.com
```

The resulting end-to-end flow became:

```text
Remote Tablet / Client
        │
        │ HTTPS / TCP 443
        ▼
hr-api.example.com
        │
        ▼
Cloudflare DNS
        │
        ▼
<PUBLIC_IPV4>
        │
        ▼
Firewall / Router
        │
        │ NAT
        ▼
<IIS_PRIVATE_IP>:443
        │
        ▼
Microsoft IIS
        │
        │ TLS termination
        ▼
HR Web API
        │
        ▼
ERP / Internal Database
```

The HTTPS endpoint was tested externally and the expected API response was returned without a certificate warning.

---

# Step 5 – Automatic Certificate Renewal & IIS Binding Update

The initial certificate renewal task existed, but one operational issue remained: after a future certificate renewal, the IIS HTTPS binding also had to move automatically from the previous certificate to the newly issued certificate.

## 5.1 Edit the existing renewal

simple-acme was opened as Administrator:

```text
C:\Program Files\simple-acme\wacs.exe
```

Then:

```text
A: Manage renewals
```

followed by:

```text
E: Edit renewal
```

Only the installation stage was edited:

```text
7: Installation
```

The existing source, validation, key, store, PFX export, CA, and certificate profile settings were left unchanged.

## 5.2 Add the IIS installation plugin

The following installation step was enabled:

```text
1: Create or update bindings in IIS
```

When asked whether new HTTPS bindings should be created, the safe option was selected:

```text
<Enter>: Do not create new bindings
         (only update existing ones)
```

This was intentional because the correct production HTTPS binding already existed.

The renewal process therefore became responsible only for updating the existing binding to the new certificate.

## 5.3 Final automated renewal architecture

```text
Windows Scheduled Task
        │
        ▼
simple-acme renewal check
        │
        ▼
Certificate renewal required?
        │
        ▼
HTTP-01 Validation / TCP 80
        │
        ▼
Let's Encrypt
        │
        ▼
New SSL/TLS Certificate
        │
        ├──────────────────┐
        ▼                  ▼
Windows WebHosting     PFX Export
Certificate Store      Updated
        │
        ▼
simple-acme IIS Plugin
        │
        ▼
Find existing HTTPS binding
using the previous certificate
        │
        ▼
Update binding to the new certificate
        │
        ▼
HTTPS service remains operational
```

## 5.4 Forced renewal test

A forced renewal occurred after the installation stage was updated.

The relevant output confirmed:

```text
Force renewing [Manual] <API_HOSTNAME>
```

A new certificate was issued and installed in the Windows WebHosting store.

The PFX export was updated.

Most importantly, simple-acme detected and updated the existing IIS HTTPS binding:

```text
Installing with Manage IIS bindings...
Updating existing binding <EXISTING_HTTPS_BINDING>
Committing 1 https binding changes to IIS
```

The previous certificate was then removed from the Windows certificate store.

This demonstrated that the certificate lifecycle was automated end-to-end rather than merely configured theoretically.

## 5.5 Final verification

External TCP/443 connectivity was tested with:

```powershell
Test-NetConnection hr-api.example.com -Port 443
```

Expected result:

```text
TcpTestSucceeded : True
```

The HTTPS API endpoint was also verified in a browser:

```text
https://hr-api.example.com
```

Validation criteria:

```text
HTTPS connection succeeds
No certificate warning is displayed
The current certificate is presented by IIS
The HR Web API returns its expected response
```

---

# Operational Requirements

For the automated certificate lifecycle to keep working:

```text
The API hostname must continue to resolve to the correct public IP.

TCP/80 must remain reachable for HTTP-01 validation.

TCP/443 must remain reachable for the production HTTPS service.

The simple-acme installation directory must remain available.

The Windows Scheduled Task created by simple-acme must remain enabled.

The simple-acme renewal configuration must not be deleted.

The existing IIS HTTPS binding must continue to use the certificate managed by simple-acme.
```

---

# Troubleshooting

## DNS

```cmd
nslookup hr-api.example.com
```

Expected: the hostname resolves to the configured public IPv4 address.

## HTTP / ACME validation path

```powershell
Test-NetConnection hr-api.example.com -Port 80
```

Expected:

```text
TcpTestSucceeded : True
```

## HTTPS

```powershell
Test-NetConnection hr-api.example.com -Port 443
```

Expected:

```text
TcpTestSucceeded : True
```

## API availability

Open:

```text
https://hr-api.example.com
```

Expected: the API responds successfully without a TLS/certificate warning.

## Renewal status

Run:

```text
wacs.exe
→ A: Manage renewals
```

Review:

```text
Renewal status
Next renewal date
Renewal configuration
```

---

# Final Architecture

```text
Remote HR Client
        │
        │ HTTPS / TCP 443
        ▼
hr-api.example.com
        │
        ▼
Cloudflare DNS
        │
        ▼
<PUBLIC_IPV4>
        │
        ▼
MikroTik RouterOS / Firewall
        │
        │ NAT / Port Forwarding
        ▼
<IIS_PRIVATE_IP>
        │
        ▼
Windows Server / Microsoft IIS
        │
        │ SSL/TLS
        ▼
HR Web API
        │
        ▼
ERP / Internal Database

        ▲
        │
simple-acme + Let's Encrypt
Automated Certificate Lifecycle
```

---

# Final Status

```text
DNS configuration                         COMPLETE
Public hostname                           COMPLETE
NAT / Port forwarding                     COMPLETE
HTTP connectivity                         VERIFIED
HTTPS connectivity                        VERIFIED
Let's Encrypt SSL/TLS                     COMPLETE
ACME HTTP-01 validation                   VERIFIED
Windows Certificate Store                 CONFIGURED
Password-protected PFX export             CONFIGURED
IIS HTTPS binding                         COMPLETE
Scheduled certificate renewal             CONFIGURED
Automatic IIS certificate rebinding       VERIFIED
Automatic PFX update                      VERIFIED
Forced renewal test                       SUCCESSFUL
End-to-end HTTPS verification             SUCCESSFUL
```

---

# Skills / Technologies Demonstrated

- Windows Server Administration
- Microsoft IIS
- TCP/IP Networking
- Public vs. Private IP Addressing
- DNS Management
- Cloudflare DNS
- A Records and FQDNs
- NAT / Port Forwarding
- MikroTik RouterOS
- HTTP / HTTPS
- SSL/TLS
- PKI & Certificate Management
- Let's Encrypt
- ACME Protocol
- HTTP-01 Domain Validation
- RSA Public Key Cryptography
- Windows Certificate Store
- PFX / PKCS#12 Management
- simple-acme
- Automated Certificate Renewal
- IIS SSL Binding Automation
- Windows Task Scheduler
- PowerShell Network Troubleshooting
- Secure API Publishing
- End-to-End Infrastructure Troubleshooting

---

# Public-Safety Notes

This repository intentionally does **not** include production IP addresses, internal subnet details, production hostnames, certificate private keys, PFX files, passwords, ACME account identifiers, certificate thumbprints, employee data, screenshots containing sensitive infrastructure details, or provider credentials.

The hostname `hr-api.example.com` and all values written as `<PLACEHOLDER>` are documentation-only examples.
