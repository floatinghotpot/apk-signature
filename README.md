
Command line tool to print MD5 / SHA1 / SHA256 fingerprints of Android APK package.

# Why we need it? #

keytool no longer print MD5 after java8.

But sometimes we do need the MD5 for example, in WeChat and AliPay developer console.

Some alternative tools do not work well:

1. Gen_Signature_Android.apk, need to be install to Android device, sometimes refused by Android device.

2. jadx, a powerful to decompile APK, we can view details of APK package, but it's too heavy.

To view the fingerprints, a very tiny command line tool will be enough.

# Installation #

Please make sure your Python is v3.7 or above.

```bash
python3 --version
python3 -m pip install apk-signature
```

Or, clone from GitHub:
```bash
git clone https://github.com/floatinghotpot/apk-signature.git
cd apk-signature
python3 -m pip install -e .
```

# How To Use #

```bash
apk-signature <path_to_apk_file>
```

Example:
```bash
apk-signature myapp.apk
```

# How It Works #

Here are the steps that the tool actualy runs:

```bash
# unzip the IPA file to tmp folder
mkdir ./tmp
unzip <path_to_apk_file> -d ./tmp

# run openssl to extract certificate file from CERT.RSA
openssl pkcs7 -inform DER -in ./tmp/META-INF/CERT.RSA -print_certs -out ./tmp/CERT.cert

# print the fingerprints in upper case
openssl x509 -in ./tmp/CERT.cert -fingerprint -noout -md5
openssl x509 -in ./tmp/CERT.cert -fingerprint -noout -sha1
openssl x509 -in ./tmp/CERT.cert -fingerprint -noout -sha256

# print the fingterprints in lower case
openssl x509 -in ./tmp/CERT.cert -outform DER | openssl dgst -md5
openssl x509 -in ./tmp/CERT.cert -outform DER | openssl dgst -sha1
openssl x509 -in ./tmp/CERT.cert -outform DER | openssl dgst -sha256

# clean up the tmp folder
rm -r ./tmp
```

# Dependency #

It will call openssl, so make sure it's installed first.

If not installed, install  it with Homebrew:
```bash
brew install openssl
```

# Credits #

A simple tool created by Raymond Xie, to print the MD5/SHA1/SHA256 signature of APK package with command line.

Any comments are welcome.
