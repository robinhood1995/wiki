---
title: Website Tools
description: 
published: true
date: 2025-10-23T01:58:59.382Z
tags: 
editor: markdown
dateCreated: 2025-10-23T01:58:56.941Z
---

# <img src="/images/windows-terminal.png" width="50" style="vertical-align:middle;margin-right:4px">Website Tools

This is a set of website testing tools for websites.

# Commands and Tools
# {.tabset}
## <img src="/images/linux.png" width="25" class="tab-icon"> Test SSL Certificates
To test an internal website's SSL/TLS configuration and certificate using OpenSSL, the s_client command is the primary tool.

### Basic SSL/TLS Connectivity Test:
The most common command to initiate a connection and retrieve certificate details is:
```bash
openssl s_client -connect <hostname_or_IP>:<port>
```
Replace <hostname_or_IP> with the internal website's hostname or IP address.
Replace [port] with the port the website is listening on (e.g., 443 for HTTPS).
This command will display various information, including the server's certificate, the certificate chain, and the cipher suite used for the connection.

### Verifying Certificate Details:
To specifically extract and display the certificate in a more readable format, you can pipe the output to openssl x509:
```bash
openssl s_client -connect <hostname_or_IP>:<port> | openssl x509 -noout -text
```
This will show details such as the issuer, subject, validity period, and extensions of the server's certificate.
### Testing Specific SSL/TLS Protocols and Ciphers:
You can test for support of specific protocols (e.g., TLSv1.2, TLSv1.3) or ciphers by adding options like -tls1_2, -tls1_3, or -cipher <cipher_string> to the s_client command.
Example for TLSv1.2:
```bash
openssl s_client -connect <hostname_or_IP>:<port> -tls1_2
```
### Verifying Certificate Chain and Trust:
If you have a custom certificate authority (CA) for your internal website, you might need to provide the CA certificate to OpenSSL for verification:
```bash
openssl s_client -connect <hostname_or_IP>:<port> -CAfile <path_to_CA_cert.pem> -verify_return_error
```
Replace <path_to_CA_cert.pem> with the path to your CA certificate file.
-verify_return_error ensures that the connection fails if there are certificate verification errors.
### Saving the Certificate:
To save the server's certificate to a file for later analysis:
```bash
openssl s_client -connect <hostname_or_IP>:<port> > server_cert.pem
```
This will redirect the output, including the certificate, to the server_cert.pem file. You can then use openssl x509 -in server_cert.pem -noout -text to inspect it.
