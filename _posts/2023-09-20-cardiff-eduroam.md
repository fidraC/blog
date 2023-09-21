---
title: "How to log into Cardiff University's eduroam Website"
categories:
  - blog
tags:
  - networking
  - wpa2
  - student life
  - fedora
---
Today, I moved into Talybont Court and had some issues connecting to the WIFI network.

Their instructions only support the common operating systems (Windows, iOS, Android, Ubuntu). While you might assume that the installer script for Ubuntu would work for other Linux distros, it fails due to missing dependencies and different file paths.Error messages were non-existant. It just hangs at 100% CPU.
Additionally, it's very insecure to run their script as it installs a root CA that could allow them to conduct MITM attacks.

Dependencies:
- `openssl`
- Gnome Settings

Steps:
1. Connect to `CU-Wireless`.
2. Go to https://onboard.cardiff.ac.uk/.
3. Select your operating system as "Other."
4. Enter your university email and password—the same one used for the intranet.

<img src="https://github.com/acheong08/blog/assets/36258159/f5b3a440-ab46-49aa-8632-78d3e7ca08e4" alt="login" width="400"/>

5. Download `OnboardCertificate.pkcs12` (automatic) and `CardiffUniversityRootCA.crt` (manual). Copy the certificate password. You can ignore `ClearPass_Onboard_Certificate_Authority.crt`

<img src="https://github.com/acheong08/blog/assets/36258159/031f3057-f78e-4bb6-bb88-83352637c231" alt="certificates" width="400"/>

6. Run `openssl pkcs12 -in OnboardCertificate.pkcs12` and enter the certificate password from the site three times for convenience.
7. Copy and save the certificate and private key in separate files. Place them somewhere you won't move/delete. I put them in `~/.config/certs`. `priv.pem` starts with `-----BEGIN ENCRYPTED PRIVATE KEY-----` and `user_cert.pem` starts with `-----BEGIN CERTIFICATE-----`.
8. Connect to the `eduroam` network with the following settings (Refer to the image below):

<img src="https://github.com/acheong08/blog/assets/36258159/4dd0866c-e467-4f89-844d-98d90b20d624" alt="login" width="400"/>

The "User key password" is the certificate password from earlier.

If you have any questions, feel free to contact me via email: `CheongYX@cardiff.ac.uk`
