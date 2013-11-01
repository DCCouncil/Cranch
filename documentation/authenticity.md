# Authenticity

One of the three key requirements of [UELMA](UELMA.md) is authenticity which, in this context, means the ability to reliably determine the author of a file through technological means. The purpose of authenticity in UELMA is to be able to know whether a digital file containing legal materials was or was not issued by the office resposible for publishing those materials.

More on what this does and does not do:

* Digital authenticity does not prevent legal materials from being altered. In fact, altering legal materials is quite common and important. Legal publishers almost always modify legal materials by adding annotations and hyperlinks, improving citations, and so on. What authenticity does do is make it impossible (or at least very hard) for any altered materials to be passed off as originating from the government publisher.

* There are multiple methods for implementing authenticity. Most methods use a digital signature. A digital signature is an attachment to a document that reliably indicates who issued the document in a way that can be verified without the need to compare the document to any master copy. No connection to the Internet should be necessary to verify a digital signature.

* Any file can be digitally signed, but not every file is suitable for every purpose. For instance, a PDF is suitable for viewing but not suitable at all for legal publishers to add annotations.  Therefore it may be necessary to issue multiple digitally signed files to suit different purposes.

We are prototyping a method for digitally signing the DC Code using S/MIME, which is a standard envelope format for combining a file and its signature. We will likely also prototype XML Signature and PDF digital signatures at a later date.

## Preparation

Before creating a digital signature, the publishing office (us) will need to create a digital certificate. The certificate is created roughly once per year and is used for all digital signing during that year.

But before that, first we generate a "public/private key pair". The private key is a file which must be kept secret that allows us to generate digital signatures on our own behalf. We'll do this using the free `openssl` tool which is available on Linux, Mac, and any Unix-based operating systems, and we'll do it from the command line.

Type in at the command line: (the "$" at the start of the next line is a notational convention indicating you are at the command prompt)

	$ openssl genrsa -out dc_council_cranch.key 4096

This command creates a 4096-byte private key in the file `dc_council_cranch.key`. It is basically a file containing random, unguessable bytes. It must be kept private. We do not share it with anyone.

Next we create a certificate signing request (CSR) that associates our identity "Council of the District of Columbia" with those random bytes.

	$ openssl req -new -key dc_council_cranch.key -out dc_council_cranch.csr
	Country Name: US
	State or Province Name: District of Columbia
	Locality Name: Washington
	Organization Name: Council of the District of Columbia
	Organizational Unit Name: Office of the General Counsel
	Common Name: Council of the District of Columbia

The new CSR `dc_council_cranch.csr` is not trustable, however. So far anyone could have created this file.

What makes a CSR trustable is by getting it signed by a Certificate Authority (CA), such as VeriSign and Thawte. At this point we would buy a digital certificate from a trustworthy CA. That means we send them `dc_council_cranch.csr`and they check that we are who we say we are. They send back a certificate file for us which we will save as `dc_council_cranch.crt`, and their own public certificate which we'll save as `ca.crt`. It costs about $300 and the certificate file they send back is good for one year.

During testing, we will have our CSR signed by a fake CA called "Our Fake CA". To create the fake CA and then sign our real CSR, run the commands below. Again, this is just during testing.
	
	$ openssl genrsa -out ca.key 4096
	$ openssl req -new -x509 -days 1826 -key ca.key -out ca.crt
	Common Name: Fake CA
	$ openssl x509 -req -days 730 -in dc_council_cranch.csr -CA ca.crt -CAkey ca.key -set_serial 01 -out dc_council_cranch.crt

Now we have a file named `dc_council_cranch.crt` and the CA's public certificate `ca.crt`.

There are two things that make our certificate file `dc_council_cranch.crt` important:

* Because only we have the *private key* created at the beginning, we are the only ones who can create digital signatures using this particular certificate file.
* Because this certificate file was signed by a CA, users will believe (trust) that any file that contains a digital signature created by this certificate was in fact issued by us.

## Creating the Digital Signature

We will be signing the DC Code's XML format in this section using an S/MIME envelope. These steps apply equally well to any format of the DC Code whatsoever. We'll assume the DC Code is in a file named `dc_code.xml`.

To create the S/MIME envelope, we use `openssl` to combine the original file, our certificate, and the CA's certificate. We also provide our private key (but of course it is not included in the S/MIME envelope).

	$ openssl smime -sign -in dc_code.xml -text -signer dc_council_cranch.crt \
	  -inkey dc_council_cranch.key -certfile ca.crt -out dc_code.msg

This creates the a file `dc_code.msg`, which is the digitally signed DC Code.

## Verifying the Digital Signature

We assume the user now has `dc_code.msg`, which contains both the DC Code and the digital signature.

To verify the signature, the user will need to indepdently acquire the CA's public certificate. Because CAs sign millions of certificates, the user may already have it. We assume he has saved it in `ca.crt`.

If the user has a Mac, Linux, or other Unix-based operating system, he can extract the DC Code XML and verify the digital signature by running at the command line:

	$ openssl smime -verify -in dc_code.msg -CAfile ca.crt -signer signer.crt > dc_code.xml

The original XML file will be saved to `dc_code.xml` and the user will see:

	Verification successful

Additionally the certificate used to sign the file will be saved to `signer.crt`. It is important that the user now check that it was us who signed the file by checking the name in the certificate:

	$ openssl x509 -in received.crt -noout -text | grep Subject

The output will be the same as we provided for the CSR above:

	Subject: C=US, ST=District of Columbia, L=Washington,
	O=Council of the District of Columbia, OU=Office of the General Counsel,
	CN=Council of the District of Columbia

## Alternative Method

An alternative signing method is to generate a separate `.sig` file.  To generate the `.sig` file, we use our private key `dc_council_cranch.key` and the file that contains the DC Code:

	$ openssl dgst -sha256 -sign dc_council_cranch.key -out dc_code.xml.sig dc_code.xml

This created the file `dc_code.xml.sig`. The digital signature has been created.

When users download the DC Code, they should download both `dc_code.xml` and `dc_code.xml.sig`. They will also need our certificate file `dc_council_cranch.crt`, but bear in mind that they will only need to download this file once per year because it will not change.

If the user has a Mac, Linux, or other Unix-based operating system, he can verify the digital signature by running at the command line:

	$ openssl dgst -sha256 -verify  <(openssl x509 -in dc_council_cranch.crt -pubkey -noout) \
	  -signature dc_code.xml.sig dc_code.xml

The output is:

	Verified OK

This checks that the file `dc_code.xml` was issued by the owner of the digital certificate (namely, us). If any of the three files had been tampered with, the result would instead by:

	Verification Failure

The user may also want to check that our certificate was signed by a CA they trust. They will need to independently get the CA's certificate first (`ca.crt`), and then they can run:

	openssl verify -CAfile ca.crt dc_council_cranch.crt

Which outputs that the certificate is OK:

	dc_council_cranch.crt: OK

