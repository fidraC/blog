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
Today, I moved into Talybont Court. They only support the common operating systems (Windows, iOS, Android, Ubuntu). You might assume that the installer script for Ubuntu would work for other Linux distros, but it doesn't. Additionally, it's very insecure to run their script as it installs a root CA that allows them to conduct MITM attacks. Here's how I connected to the network on Fedora Silverblue without trusting their root CA for websites:

1. Connect to `CU-Wireless`.
2. Go to https://onboard.cardiff.ac.uk/.
3. Select your operating system as "Other."
4. Enter your university email and password—the same one used for the intranet.
5. Download all the certificates and files. Don't close the tab. Copy the certificate password.
6. Run `openssl pkcs12 -in OnboardCertificate.pkcs12` and enter the certificate password from the site three times for convenience.
7. Copy and save the certificate and private key in separate files.
8. Connect to the `eduroam` network with the following settings (Refer to the image below):
![image](https://github.com/acheong08/blog/assets/36258159/4dd0866c-e467-4f89-844d-98d90b20d624)

The "User key password" is the certificate password from earlier.
