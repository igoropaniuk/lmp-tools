This folder contains Firmware Management Protocol (FMP) keys for the UEFI
Capsule Generation process [1].

Details about keys generation can be found at [2]. Before running OpenSSL commands, edit
opensslroot.cfg and adjust the configuration accordingly (if needed).
To generate keys the cmds below were used:

Set env variables with proper configuration:
$ export FMP_ROOT_CERT_SUBJECT="/CN=OEM Root CA/O=FMP/OU=OEM Key/L=San Diego/ST=California/C=US"
$ export FMP_CA_CERT_SUBJECT="/CN=OEM Intermediate CA/O=FMP/OU=OEM Key/L=San Diego/ST=California/C=US"
$ export FMP_USER_CERT_SUBJECT="/CN=OEM User/O=FMP/OU=OEM Key/L=San Diego/ST=California/C=US"
$ export FMP_KEY_PASSWORD="foundriesio"

The demoCA directory should be initialized:
$ mkdir -p demoCA
$ mkdir -p demoCA/newcerts
$ touch demoCA/index.txt
$ echo 01 > demoCA/serial

Create rand file:
$ dd if=/dev/urandom of=randfile bs=256 count=1 > /dev/null 2>&1

Generate a Root Key/Certificate:
$ openssl genrsa -aes256 -passout "pass:${FMP_KEY_PASSWORD}" -out QcFMPRoot.key 2048
$ openssl req -new -x509 -config opensslroot.cfg -subj "${FMP_ROOT_CERT_SUBJECT}" -days 3650 \
  -passin "pass:${FMP_KEY_PASSWORD}" -key QcFMPRoot.key -out QcFMPRoot.crt
$ openssl x509 -in QcFMPRoot.crt -out QcFMPRoot.cer -outform DER
$ openssl x509 -inform DER -in QcFMPRoot.cer -outform PEM -out QcFMPRoot.pub.pem

Generate the Intermediate Key/Certificate:
$ openssl genrsa -aes256 -passout "pass:${FMP_KEY_PASSWORD}" -out QcFMPSub.key 2048
$ openssl req -new -config opensslroot.cfg -subj "${FMP_CA_CERT_SUBJECT}" \
    -passin "pass:${FMP_KEY_PASSWORD}" -key QcFMPSub.key -out QcFMPSub.csr
$ openssl ca -config opensslroot.cfg -extensions v3_ca -batch \
    -in QcFMPSub.csr -days 3650 -out QcFMPSub.crt -cert QcFMPRoot.crt \
    -passin "pass:${FMP_KEY_PASSWORD}" -keyfile QcFMPRoot.key
$ openssl x509 -in QcFMPSub.crt -out QcFMPSub.cer -outform DER
$ openssl x509 -inform DER -in QcFMPSub.cer -outform PEM -out QcFMPSub.pub.pem

Generate User Key Pair/Certificate for Data Signing:
$ openssl genrsa -aes256 -passout "pass:${FMP_KEY_PASSWORD}" -out QcFMPCert.key 2048
$ openssl req -new -config opensslroot.cfg -subj "${FMP_USER_CERT_SUBJECT}" \
    -passin "pass:${FMP_KEY_PASSWORD}" -key QcFMPCert.key -out QcFMPCert.csr
$ openssl ca -config opensslroot.cfg -batch -in QcFMPCert.csr -days 3650 \
    -out QcFMPCert.crt -cert QcFMPSub.crt -passin "pass:${FMP_KEY_PASSWORD}" -keyfile QcFMPSub.key
$ openssl x509 -in QcFMPCert.crt -out QcFMPCert.cer -outform DER
$ openssl x509 -inform DER -in QcFMPCert.cer -outform PEM -out QcFMPCert.pub.pem

Convert the User Key and Certificate:
$ openssl pkcs12 -export -passout "pass:${FMP_KEY_PASSWORD}" -out QcFMPCert.pfx \
    -passin "pass:${FMP_KEY_PASSWORD}" -inkey QcFMPCert.key -in QcFMPCert.crt
$ openssl pkcs12 -passin "pass:${FMP_KEY_PASSWORD}" -in QcFMPCert.pfx -nodes \
    -out QcFMPCert.pem

[1] https://github.com/quic/cbsp-boot-utilities/tree/main/uefi_capsule_generation
[2] https://github.com/tianocore/tianocore.github.io/wiki/Capsule-Based-System-Firmware-Update-Generate-Keys
