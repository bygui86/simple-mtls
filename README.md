
# Simple mTLS example

## Prerequisites

- OpenSSL
- Node.js

## Instructions

### 1. Certificate Authority

1. Create Certificate Authority (CA) certificate

	```shell script
	openssl req \
		-new \
		-x509 \
		-nodes \
		-days 365 \
		-subj '/CN=my-ca' \
		-keyout ca.key \
		-out ca.crt
	```

	This outputs two files, `ca.key` and `ca.crt`, in the PEM format (base64 encoding of the private key and X.509 certificate respectively).

1. Inspect CA certificate

	```shell script
	openssl x509 \
		-in ca.crt \
		-text \
		-noout
	```

	Looking at the output, we can confirm:

	- Both the `Subject` and `Issuer` have the value `CN = my-ca`; this indicates that this certificate is `self-signed`
	- The `Validity` indicates that the certificate is valid for a year
	- The `X509v3 Basic Constraints` value `CA:TRUE` indicates that this certificate can be used as a CA, i.e., can be used to sign certificates

### 2. Server

1. Create the server's key

	```shell script
	openssl genrsa \
  		-out server.key 2048
	```

1. Create the server's Certificate Signing Request (CSR) for the Common Name (CN a.k.a. DNS name) `localhost` signed by our CA

	```shell script
	openssl req \
		-new \
		-key server.key \
		-subj '/CN=localhost' \
		-out server.csr
	```

1. Create the signed certificate using CSR and CA

	```shell script
	openssl x509 \
		-req \
		-in server.csr \
		-CA ca.crt \
		-CAkey ca.key \
		-CAcreateserial \
		-days 365 \
		-out server.crt
	```

	The output is the signed server certificate, `server.crt`, in the PEM format.

	`(i) INFO` CAcreateserial option manages a newly created file, `ca.srl`, which enables each certificate created by the CA to have a unique serial number.

1. Inspect server's certificate

	```shell script
	openssl x509 \
		-in server.crt \
		-text \
		-noout
	```

	Looking at the output, we can confirm:

	- The `Issuer` has the value `CN = my-ca`, this indicates that the certificate is signed by the `my-ca` CA
	- The `Validity` indicates that the certificate is valid for a year
	- The `Subject` has the value `CN = localhost`, this indicates that this certificate can be served to a client to validate that the server is trusted to serve up content for the DNS name `localhost`

### 3. Client

We essentially repeat the process to create the client's key and certificate using our CA.

1. Create client's key

	```shell script
	openssl genrsa \
		-out client.key 2048
	```

1. Create the CSR with the arbitrary `Common Name` of `my-client`

	```shell script
	openssl req \
		-new \
		-key client.key \
		-subj '/CN=my-client' \
		-out client.csr
	```

1. Create client's certificate

	```shell script
	openssl x509 \
		-req \
		-in client.csr \
		-CA ca.crt \
		-CAkey ca.key \
		-CAcreateserial \
		-days 365 \
		-out client.crt
	```

### Notes

> If you inspect client's certificate, you will observe that the `Serial Number` is indeed different than the server’s certificate.

> One observation is that both the server and client certificates are simpler `X.509 v1` certificates. The CA certificate however is a `X.509 v3` certificate. This is because OpenSSL automatically creates `X.509 v3` self-signed certificates (CA certificate) and we did not supply any v3 extensions when signing the server and client certificates (using the `extfile` and `extensions` options).

### 4. Test mTLS

`(i) INFO` The server is the basic Hello World example, provided by Node.js, enhanced to support mTLS.

`(i) INFO` The client is simply the cURL web browser with options.

1. Start server

	```shell script
	node index.js
	```

1. Make a request expecting it's successfull

	```shell script
	curl \
		--cacert ca.crt \
		--key client.key \
		--cert client.crt \
		https://localhost:3000
	```

	expected result

	```
	Hello World
	```

	Here the `cacert` option is used so that the client (cURL) can validate the server supplied certificate.
	
	The `key` and `cert` are used so the client sends the CA signed client certificate with the request.

1. Make a request leaving off the cacert option (fail)

	```shell script
	curl \
		--key client.key \
		--cert client.crt \
		https://localhost:3000
	```

	expected result

	```
	curl: (60) SSL certificate problem: self signed certificate in certificate chain
	More details here: https://curl.haxx.se/docs/sslcerts.html

	curl failed to verify the legitimacy of the server and therefore could not
	establish a secure connection to it. To learn more about this situation and
	how to fix it, please visit the web page mentioned above.
	```

1. Make a request leaving off key and cert options (fail with a different error)

	```shell script
	curl \
		--cacert ca.crt \
		https://localhost:3000
	```

	expected result

	```
	curl: (35) error:1401E410:SSL routines:CONNECT_CR_FINISHED:sslv3 alert handshake failure
	```

## Links

- https://codeburst.io/mutual-tls-authentication-mtls-de-mystified-11fa2a52e9cf
- https://nodejs.org/en/about/
