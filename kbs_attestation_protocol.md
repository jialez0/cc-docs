# KBS Attestation Protocol

As a trusted service, KBS's client (Key Broker Client, KBC for short) runs in a HW-TEE platform. 
When the KBC sends a request to KBS, HW-TEE attestation is required. Universal-KBS performs a 
simple, universal and extensible "Request-Challenge-Attestation-Response" (we call it **RCAR** 
for short) protocol to complete the attestation, and encrypt the response payload content 
returned by KBS to KBC with the key which protected by HW-TEE.

# Introduction

The purpose of the attestation between KBS and KBC is actually to confirm whether 
**the platform where the KBC is located is in the expected security state**, that is, it runs in 
a real and harmless HW-TEE. Therefore, naturally, HW-TEE authentication can be regarded as an 
extension/enhancement of the existing security authentication protocol stack. At present, some 
projects have made such attempts, such as [HTTPA](https://arxiv.org/abs/2110.07954), [RATS-TLS](https://github.com/inclavare-containers/rats-tls) and so on. 
However, Universal-KBS do not intend to apply these projects directly, nor will we design a new 
similar implementation, because it is inevitable to make some deep-seated adaptation to the 
existing transmission module at the server and client. We hope that the intrusion of 
Universal-KBS into existing transmission modes will be as few as possible, so that Universal-KBS 
can be applied in more and broader scenarios.

Therefore, in Universal-KBS, HW-TEE attestation process is the semantics of the application 
layer, which is defined as a simple, universal and extensible 
"Request-Challenge-Attestation-Response" (RACR) protocol. The temporary asymmetric key generated 
by HW-TEE is used to encrypt the response payload, and the token mechanism is used to avoid the 
performance problems caused by multiple attestation.

In order to ensure the ease of use and security completeness of Universal-KBS, we will use HTTPS 
as the default transmission method to carry the application layer semantics designed in this 
document. This is because HTTPS provides KBC with a means to authenticate KBS identity, which 
effectively avoids malicious attackers from hijacking KBS address to impersonate KBS and deceive KBC.

# RCAR semantics

The semantics of attestation defined by Universal-KBS is a simple and extensible four-step model, 
which uses JSON structure to organize information. As follows:

1. **Request**: The KBC actively sends to KBS, calls the service API provided by KBS to request resources or services.
2. **Challenge**: After receiving the request, KBS returns a challenge to KBC and asks KBC to 
send evidence to prove that its environment (HW-TEE) is safe and reliable.
3. **Attestation**: KBC sends the evidence and the newly generated public key in HW-TEE to KBS.
4. **Response**: After the KBS verified the evidence, it returns the API output of the requested 
service and a token to KBC, where the API output payload is protected by the public key in step 3. 
Within the valid time of the token, KBC can directly request resources or services of KBS by 
virtue of the token without step 2 and step 3.

## Request

The payload format of the request is as follows:

(Note that the `/*...*/` comments are not valid in JSON, and must not be used in real message.)

```json
{
    "request": {
	    /* Protocol version number used by KBC */
	    "protocol-version": "0.1.0",
	    /* Types of HW-TEE platforms where KBC is located, such as "TDX", "SEV-SNP", etc. */
	    "tee": "",
	    /* Access token to avoid multiple attempts triggered by consecutive requests. It can be set to blank. */
	    "token": "",
	    /* The service API requested to be called. The supported API types are defined and published by KBS. */
	    "api": "",
	    /* API input parameters. */
	    "api-input": {},
	    /* Reserved fields are used to support some special requests sent by HW-TEE. */
	    "extra-params": {}
	}
}
```

- `protocol-version`

The protocol version number supported by KBC. KBS needs to judge whether this KBC can communicate 
normally according to this field. If the protocol version used by this KBC is not supported by 
KBS, KBS will directly disconnect the link and ignore this request.

- `tee`

Used to declare the type of HW-TEE platform where KBC is located.

- `token`

If other requests have been made before this request, in the previous response KBS would return a 
token to KBC. The `token` field in request provides the token. If the token is within the 
validity period, the communication can skip the Challenge and Attestation stage. If no other 
request has been made before this request, the 'token' field can be left blank.

- `api`

The name of the KBS service API requested by KBC (such as obtaining key, decrypting payload or 
downloading specified file, etc.). The allowed APIs is specified by [KBS service API document]() 
(TODO). If KBS cannot support the API specified in this field, it will directly disconnect the 
link and ignore the request.

- `api-input`

Input parameters for the service API. For example, for the Get Key service API, KBC need to 
provide KBS with key ID and other information. For the input parameters required by each service 
API, please refer to the [service API document ]() (TODO) of KBS. If the format of the input 
parameters does not meet the requirements of he corresponding API, the parameter error 
information of the API will be returned in the response information in step 4.

## Challenge

After KBS receives the request, if the token is found to be empty or expired, KBS will return an 
attestation challenge to KBC. The payload format is as follows:

```json
{
    "challenge": {
	    /* To ensure the freshness of evidence, Base64 coding. */
	    "nonce": "",
	    /* Extra parameters to support some special HW-TEE attestation. */
	    "extra-params": {}
	}
}
```

- `nonce`

The fresh number passed to KBC. KBC needs to place it in the evidence sent to KBS in the next 
step to prevent replay attacks.

- `extra-params`

The reserved extra parameter field which is used to pass the additional information provided by 
the KBS when some specific HW-TEE needs to be attested.

## Attestation

After receiving the invitation challenge, KBC collects the attestation evidence of HW-TEE 
platform and organizes it into the following payload format:

```json
{
    "evidence": {
	    /* The nonce included in challenge to prove the freshness of this message.
	       Its hash needs to be included in HW-TEE evidence and signed by HW-TEE hardware. */
	    "nonce": "",
	    /* The public key generated by KBC in HW-TEE, it is valid until the next time an attestation is required.
	       Its hash needs to be included in HW-TEE evidence and signed by HW-TEE hardware. */
	    "tee-pubkey": {
		    "algorithm": "",
		    "pubkey-length": "",
		    "pubkey": ""
		},
	    /* The content of evidence. Its format is specified by Attestation-Service. */
	    "tee-evidence": {}
	}
}
```

- `nonce`

Nonce contained in challenge. It is used to prove the freshness of this message and prevent 
replay attacks. Its hash needs to be signed by HW-TEE hardware as part of the custom field of 
HW-TEE evidence (commonly referred to as `report data`).

- `tee-pubkey`

After KBC receives the attestation challenge, a pair of temporary asymmetric keys are generated 
in HW-TEE. The private key is stored in HW-TEE. The public key and its description information 
are exported and placed in the `tee-pubkey` field and sent to KBS together with evidence. The 
hash of the `tee pubkey` field needs to be included in the custom field of HW-TEE evidence and 
signed by HW-TEE hardware. This public key is valid until the next time KBC receives this KBS's 
attestation challenge.

- `tee-evidence`

The specific content of evidence related to the HW-TEE platform software and hardware in KBC's 
environment, different `tee-evidence` formats will be defined according to the type of TEE. This 
format is defined by the universal Attestation-Service. 
**KBS will not analyze the content of this structure, but will directly forward it to the Attestation-Service for verification**.

## Response

If Attestation-Service fails to verify the evidence, the link will be disconnected by KBS.

If KBS verified the token or the attestation evidence successfully, it will return a response to KBC in the following format:

```json
{
    "response": {
	    /* The result status of calling KBS service API, "OK" or "ERR". */
	    "api-status": "OK",
	    /* The output of KBS service API, needs to be encrypted by a symmetric key randomly generated by KBS. */
	    "api-output": "",
	    /* Symmetric key and algorithm information for encrypting `api-output`. */
	    "crypto-annotation": {
            "algorithm": "",
            "symkey-length": "",
            /* The symmetric key used to encrypt `api-output`, which is encrypted by HW-TEE's public key */
            "enc-symkey": ""
	    },
	    /* The token issued by KBS to KBC. If there is a token in the request, this field will be left blank. */
	    "token": "",
	}
}
```

- `api-status`

The result status of calling KBS service API, `"OK"` or `"ERR"`. If it is `"ERR"`, KBS need to 
give specific error information output in `api-output`. Please refer to [KBS service API document]() (TODO) 
for the definition of error information of each service API.

- `api-output`

The output of KBS service API, it needs to be encrypted by a symmetric key randomly generated by KBS. 
Different service APIs define different output parameters.

- `crypto-annotation`

The symmetric key and algorithm used to encrypt `api-output`. Because the output of KBS service 
API may contain large data, it is inefficient to directly use HW-TEE's public key for encryption. 
Therefore, digital envelope technology is used here, KBS generate a random symmetric key to 
encrypt `api-output`, and then use HW-TEE's public key to encrypt the random symmetric key.

- `token`

If the attestation evidence sent by KBC is verified before returning the response, KBS will issue 
a token to KBC in this field and sign it with KBS's root key. KBC can use this token to request 
resources or services from KBS without Challenge and Attestation within the validity period of 
the token. If there is already a token in the KBC's request, no new token will be issued and this 
field will be left blank.

The [JSON web token](https://jwt.io/) standard is adopted for the token. The format is as follows:

(Note that the `/*...*/` comments are not valid in JSON web token, and must not be used in real token.)

```json
{
    /* JWT header */
    "alg": "HS256",
    "typ": "JWT"
}.
{
    /* JWT Payload official claim set */
  
    /* Token's expiration time */
    "exp": 1568187398,
    /* Token's issuing time */
    "iat": 1568158598,
    /* Token's issuer, which is the URL address of KBS */
    "iss": "https://xxxxxx",
    
    /* JWT payload custom claim set */
  
    /* The HW-TEE's public key sent by KBC in the attestation evidence, 
       which is valid within the validity period of the token. */
    "tee-pubkey": {
	    "algo": "",
	    "pubkey-length": "",
	    "pubkey": ""
	}
}.
[Signature]
```

The header part of the token declares the format standard of the token and the signature algorithm used.

The payload part is divided into official claim set and custom claim set. In the official claim 
set, we select three fields, `exp`, `iat`' and `iss`, which respectively declare the expiration 
time, issuing time and issuer (KBS URL address) of the token. The custom field in the payload 
contains the HW-TEE's public key sent by KBC along with the attestation evidence, which is valid 
within the validity period of the token. When KBC uses the token to request resources or services 
of KBS service API, KBS uses this public key recorded in the token to protect `api-output` with 
digital envelope technology.

At the end of the token, KBS signs the header and payload with its root key to confirm that the 
token is indeed issued by KBS itself.

Within the validity period of the token, KBC can provide the token encoded by Base64 as the 
content of the `token` field in the request to KBS, so as to quickly obtain the required 
resources or service result.