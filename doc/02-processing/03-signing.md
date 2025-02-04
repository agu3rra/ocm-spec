# Signing

Signing of a component version consists of several steps:
1. digests for all reference component versions are determined and put
   into the dedicated reference element of the component descriptor.
   This is done by recursively following this procedure, but without the signing step.
2. digests for all described artifacts are determined and put into the dedicated
   element entry in the component descriptor (sources, resources)
3. the resulting component descriptor is normalized
   and a digest is calculated according to the selected digest algorithm. The resulting
   digest is stored in the component descriptor
4. the digest is signed with the selected signing algorithm and stored in the
   component descriptor.
5. the final component descriptor is updated in the OCM repository.

The digest fields *MUST* be calculated during the signing process and already existing
digest fields *CAN NOT* be trusted. Resources have to be accessed and
digests has to be determined according to the combination of artifact
normalization and digest algorithm and verified against existing digests.
Component version references have to be followed recursively, calculating the
digest for the referenced component descriptor. If digests fields for resources or
component references already exist, they have to be compared against
the calculated digest and rejected if different.

The algorithms used for signing are listed in the [extensions](../04-extensions/04-algorithms/README.md#signature-algorithms).

## Verification Procedure

Verifying a component descriptor consist of three steps. Any failing step
**MUST** fail the validation.

1. Verify the digest of all resources and component references. Recursively follow component references and create an in-memory representation of the referenced component-descriptor by accessing and digesting all resources and references. Do not trust any digest data in child component-descriptors. The digest of the normalised in-memory representation of a component-reference **MUST** match the digest in the root component-descriptor (that contains a signature we verify in the next step).

```go
func digestForComponentDescriptor(cd) -> digest:
  for reference in cd.component.componentReferences:
    referencedCd = loadCdForReference(reference)
    reference.Digest = digestForComponentDescriptor(referencedCd)

  for resource in cd.component.Resource:
    resource.Digest = loadAndDigestResource(resource)

  normalisedCd = normaliseComponentDescriptor(cd)
  digest = createDigestForNormalisedCd
  return digest
```

2. check if calculated digest of the normalized component descriptor matches the
   digest in signatures.digest with hashAlgorithm, NormalisationAlgorithm and Value
3. verify the signature for the digest of the component descriptor.

### Verify with RSA

Signature verification with RSASSA-PKCS1-V1_5 requires a Public Key. This is used in step 3 of the
verification algorithm.

### Verify with X509

Signature verification with X509 certificates require a validation of the
*signing* certificate and the signature itself. First, the validity of the
signing certificate is checked with a root CA and a chain of intermediate
certificates. The certificate requires the usage types `x509.KeyUsageDigitalSignature` and
`x509.ExtKeyUsageCodeSigning`. Afterwards, the CD signature is verified with the public key in
the signing certificate. If the signature entry of the component version describes an issuer it is
validated against the common name of the certificate.
