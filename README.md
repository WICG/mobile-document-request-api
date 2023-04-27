# Mobile Document Request API


## Authors:

- [@dcrousso](https://github.com/dcrousso)
- [@hlozi](https://github.com/hlozi)
- [@martijnharing](https://github.com/martijnharing)
- [@agl](https://github.com/agl)

## Participate

- [Issue tracker](https://github.com/WICG/mobile-document-request-api/issues)


## Introduction

This is a proposal for a new web API that allows websites to request a mobile document as defined in ISO/IEC 18013-5 using structures defined in the same ISO standard in order to achieve an online presentment.  The API protects the mobile document credential data, supports selective disclosure, and allows the user to control the release of credential data.

The ISO/IEC 18013-5:2021 standard specifies how a mobile driver's license (mDL) is represented on a mobile device and how it is presented to a physical reader.  Users who choose to have their mDL on device can present it to compatible readers instead of the physical driver's license.  The ISO standard defines the protocols and the exchanges between the mDL and the mDL reader for different physical uses case (e.g. TSA, hotels, stores, …).  The ISO standard provides building blocks to allow remote presentment of the the mobile driver’s license over the Web such as data structures and operational phases, however it does not define a web API to allow presentement via user agent.

The mechanisms defined by ISO/IEC 18013-5 are designed to support any type of mobile document, generically called “mdocs”. The API defined in this document can therefore be used to request and transmit any mobile document that implements the ISO mdoc specification.


## Goals

- Allow websites to request the presentment of a mobile document in mdoc format, such as driver’s license. 
- Enable the user to have control of what data is released to the website. 
- Protect the released data using encryption to ensure only the website the data is intended for can access the data.
- Don't preclude the addition of other credential types in the future, if needed.


## Non-goals

- Support requests spanning identity and authentication mechanisms.
- Define issuance and provisioning of an mdoc.
- Define the communication between the User Agent and the website server 
- Define the native communication between the User Agent and the application holding the mdoc.
- Define how the response to the request is used by the server. 
- Support requesting multiple mdocs in one API call.

## Proposed Solution

This API would be a new identity scheme within [IdentityCredential](https://github.com/agl/identity-credential) using the scheme identifier `mdoc`.

### Requests

```webidl
partial dictionary IdentityProviderRequestOptions  {
  optional MdocRequest mdoc;
}

dictionary MdocRequest  {
  required DOMString nonce;

  // The mdoc document type requested. See ISO 18013-5 section 7.1.
  required DOMString documentType;

  // The elements of the mdoc requested.
  required sequence<MdocElement> requestedElements;

  // The number of days that the requester will keep the data for.
  // Must be non-negative. A value of zero indicates that the data
  // will not be retained beyond the time to process it. A value of
  // positive infinity indicates unlimited retention.
  required double retentionDays;

  // A set of reader identities. Each value is a base64-encoded sequence of one
  // or more X.509 certificates in ASN.1 DER encoding (a “chain” of
  // certificates). At least one value must be provided. The first element of
  // each X.509 chain MUST be the end-entity certificate. Subsequent X.509
  // certificates in a chain are intermediate certificates as may help the wallet
  // verify the end-entity certificate.
  //
  // If the requester has no applicable X.509 chain then it may use a
  // single, self-signed certificate.
  //
  // The underlying wallet will evaluate these certificate chains
  // and pick one, at its discretion, to identify the requesting entity. The
  // SubjectPublicKeyInfo of the end-entity certificate of the selected chain
  // is used to encrypt the response.
  required sequence<DOMString> readerAuthentication;
};

dictionary MdocElement {
  required DOMString namespace;  // As defined in ISO 18013-5 section 7.
  required DOMString name;
};
```

An example request:

```js
let request = {
  identity: {
    mdoc: {
      nonce: "gf69kepV+m5tGxMyMF0cnn9NCnRHez/LUIsFtLi6pwg=",
      documentType: "org.iso.18013.5.1.mDL",
      retentionDays: 90,
      readerAuthentication: ["ftl+VEHPB17r2oi6it3ENaqhOOB0AZbAkb5f4VlCPakpdNioc9QZ7X/6w..."],
      mdocRequestedElements: [
        { namespace: "org.iso.18013.5.1", name: "document_number" },
        { namespace: "org.iso.18013.5.1", name: "portrait" },
        { namespace: "org.iso.18013.5.1", name: "driving_privileges" },
        { namespace: "org.iso.18013.5.1.aamva", name: "organ_donor" },
        { namespace: "org.iso.18013.5.1.aamva", name: "hazmat_endorsement_expiration_date" },
      ],
    },
  }
};
navigator.credentials.get(request).then(() => {
});
```

### Processing

If the user rejects the request, or if no applicable mdocs are available, or if the identity in `readerAuthentication` is not authorized to request the applicable mdoc, then a "[NotAllowedError](https://heycam.github.io/webidl/#notallowederror)" [DOMException](https://heycam.github.io/webidl/#idl-DOMException) is raised.

Otherwise the application holding the responsive mdoc builds a response according to ISO/IEC 18013-5, including the credential data itself, the authentication by the issuer, and the authentication by the device, encrypting everything with the key taken from the selected `readerAuthentication` value.

### Response

A positive result from an mdoc request will be an [IdentityCredential](https://github.com/agl/identity-credential) where the `scheme` will be `mdoc` and the `result` with be the base64-encoding of the `DeviceResponse` object as defined in ISO/IEC 18013-5 clause 8.3.2.1.2.2 encoded with CBOR according to ISO/IEC 18013-5, encrypted using HPKE with the following settings:

```
mode: mode_base
KEM: DHKEM(P-256, HKDF-SHA256)
KDF: HKDF-SHA256
AEAD: AES-128-GCM
```

Part of the signature generated for `DeviceAuth` is done using the `SessionTranscript` structure as defined in ISO/IEC 18103-5 section 9.1.5.1.  The `DeviceEngagementBytes` and the `EReaderKeyBytes` are both not present, and therefore have a value of `null`.  The `Handover` element shall be as defined below:

```cddl
BrowserHandover = [
    "BrowserHandoverv1",
    Nonce,
    OriginInfoBytes,
    RequesterIdentity,
    pkEm
]
```

where `Nonce` is the nonce provided by the request, `OriginInfoBytes` contains the origin of the request and is defined in ISO/IEC 18013-7, and `RequesterIdentity` is the requester identity object from the request. (**TODO**: tighen the definition of `RequesterIdentity`, should it be the SubjectPublicKeyInfo?)

This is all included inside a CBOR encoded `CredentialDocument` with the following structure:

```cddl
CredentialDocument = {
    "version": tstr,
    "encryptionParamaters": EncryptionParamaters,
    "data": bstr
}

EncryptionParamaters = {
    "version": tstr,
    "EDeviceKey": bstr,
    "originInfoBytes": OriginInfoBytes
}
```


## Examples

Requesting specific attributes from a driver’s license:

```js
let request = {
  identity: {
    mdoc: [{
      retentionDays: 90,
      nonce: "gf69kepV+m5tGxMyMF0cnn9NCnRHez/LUIsFtLi6pwg=",
      documentType: "org.iso.18013.5.1.mDL",
      readerAuthentication: ["ftl+VEHPB17r2oi6it3ENaqhOOB0AZbAkb5f4VlCPakpdNioc9QZ7X…"],
      mdocRequestedElements: [
        { namespace: "org.iso.18013.5.1", name: "document_number" },
        { namespace: "org.iso.18013.5.1", name: "portrait" },
        { namespace: "org.iso.18013.5.1", name: "driving_privileges" },
        { namespace: "org.iso.18013.5.1.aamva", name: "organ_donor" },
        { namespace: "org.iso.18013.5.1.aamva", name: "hazmat_endorsement_expiration_date" },
      ],
    },
  }
};
navigator.credentials.get(request).then(() => {
});
```

Requesting specific mDL attributes from a COVID card:

```js
let request = {
  identity: {
    mdoc: [{
      documentType: "org.micov.1",
      retentionDays: 90,
      nonce: "gf69kepV+m5tGxMyMF0cnn9NCnRHez/LUIsFtLi6pwg=",
      readerAuthentication: ["ftl+VEHPB17r2oi6it3ENaqhOOB0AZbAkb5f4VlCPakpdNioc9QZ7X…"],
      mdocRequestedElements: [
        { namespace: "org.micov.attestation.1", name: "PersonId_dl" },
        { namespace: "org.micov.attestation.1", name: "portrait" },
      ],
    },
  }
};
```


## Privacy & Security Considerations

This API provides the ability for websites to retrieve an ISO 18013-5 mDoc already on the device.  An important part of the security and privacy mechanisms are provided by ISO 18013-5:

- Protection against forgery with passive data authentication defined in ISO 18013-5 based on Mobile Security Object (MSO).
- Protection against cloning using active authentication leveraging a signature over session data.  The signature is produced by the device using the device private key as defined in ISO 18013-5.  The Device Public key is part of the MSO signed by the issuer.
- The ability for the user to see the requested data before disclosure, in addition to being able to choose what to disclose.
- The ability for the website to request a subset of data, rather than any sort of all-or-nothing.
- Protection against eavesdropping during transactions by using HPKE to encrypt the response containing the mDoc to the requesting server.
- Man in the middle attacks are detectable by the requesting server via validation of the device signature over session data.


## References & acknowledgements

Many thanks for valuable feedback and advice from:

- [@15characterlimi](https://github.com/15characterlimi)
- [@balfanz](https://github.com/balfanz)
- [@bslassey](https://github.com/bslassey)
- [@cwilso](https://github.com/cwilso)
- [@davidz25](https://github.com/davidz25)
- [@divegeek](https://github.com/divegeek)
- [@hober](https://github.com/hober)
- [@jyasskin](https://github.com/jyasskin)
- [@samuelgoto](https://github.com/samuelgoto)
- [@sethmoo](https://github.com/sethmoo)
