---

copyright:
  years: 2020, 2022
lastupdated: "2022-04-25"

keywords: create certificate authority, create root CA, create intermediate CA, set up PKI, set up private certificates, private certificates engine

subcollection: secrets-manager

---

{:codeblock: .codeblock}
{:screen: .screen}
{:download: .download}
{:external: target="_blank" .external}
{:faq: data-hd-content-type='faq'}
{:gif: data-image-type='gif'}
{:important: .important}
{:note: .note}
{:pre: .pre}
{:tip: .tip}
{:preview: .preview}
{:deprecated: .deprecated}
{:beta: .beta}
{:term: .term}
{:shortdesc: .shortdesc}
{:script: data-hd-video='script'}
{:support: data-reuse='support'}
{:table: .aria-labeledby="caption"}
{:troubleshoot: data-hd-content-type='troubleshoot'}
{:help: data-hd-content-type='help'}
{:tsCauses: .tsCauses}
{:tsResolve: .tsResolve}
{:tsSymptoms: .tsSymptoms}
{:video: .video}
{:step: data-tutorial-type='step'}
{:tutorial: data-hd-content-type='tutorial'}
{:api: .ph data-hd-interface='api'}
{:cli: .ph data-hd-interface='cli'}
{:ui: .ph data-hd-interface='ui'}
{:curl: .ph data-hd-programlang='curl'}
{:java: .ph data-hd-programlang='java'}
{:ruby: .ph data-hd-programlang='ruby'}
{:c#: .ph data-hd-programlang='c#'}
{:objectc: .ph data-hd-programlang='Objective C'}
{:python: .ph data-hd-programlang='python'}
{:javascript: .ph data-hd-programlang='javascript'}
{:php: .ph data-hd-programlang='PHP'}
{:swift: .ph data-hd-programlang='swift'}
{:curl: .ph data-hd-programlang='curl'}
{:dotnet-standard: .ph data-hd-programlang='dotnet-standard'}
{:go: .ph data-hd-programlang='go'}
{:unity: .ph data-hd-programlang='unity'}
{:release-note: data-hd-content-type='release-note'}

# Preparing to create private certificates
{: #prepare-create-certificates}

You can enable your {{site.data.keyword.secrets-manager_full}} service instance to generate private certificates by configuring the private certificates engine.
{: shortdesc}

In {{site.data.keyword.secrets-manager_short}}, the private certificates engine serves as the back end for the `private_cert` secret type. Private certificates are SSL/TLS certificates that you can sign, issue, and manage in the service. Before you can create a private certificate, you must enable your service instance by creating [certificate authorities (CA)](#x2016383){: term} and a valid chain of trust for your certificates.


## Learning about certificate hierarchies
{: #what-is-ca-hierarchy}

With {{site.data.keyword.secrets-manager_short}}, you can build your own public-key infrastructure (PKI) system by creating certificate authorities (CA) that can sign and issue SSL/TLS certificates to your applications. With a certificates chain in place, you can use your {{site.data.keyword.secrets-manager_short}} instance to create [private certificates](/docs/secrets-manager?topic=secrets-manager-certificates#create-certificates) for your client and server apps.

A valid chain of certificates begins at a trusted root CA, passes through one or more subordinate CAs, and ends with a leaf certificate that is issued to your end-entity application. For example, check out the following simple CA hierarchy:


![The diagram shows a three-level hierarchy of certificates that begins with a root certificate authority in the first level.](images/certificate-hierarchy.svg){: caption="Figure 1. Example three-level CA hierarchy" caption-side="bottom"}

1. The root CA serves as the trust anchor for your entire chain of certificates. 
2. The subordinate CAs in level 2 are signed and issued by the root CA. These subordinate CAs sign other subordinate CA certificates.
3. Finally, the subordinate CA certificates in level 3 sign and issue leaf certificates to your end-entity applications.

   In {{site.data.keyword.secrets-manager_short}}, leaf certificates are the [private certificates](/docs/secrets-manager?topic=secrets-manager-certificates#create-certificates) that you create and deploy to your applications.

## Designing your CA hierarchy
{: #design-ca-hierarchy}

With {{site.data.keyword.secrets-manager_short}}, you can create up to 10 root CAs and 10 intermediate CAs in your service instance with multiple branches and hierarchies.

| Authority type | Description |
| --- | --- |
| [Root certificate authority](/docs/secrets-manager?topic=secrets-manager-root-certificate-authorities) | A trust anchor for your certificates chain. In a hierarchy of certificates, a root CA resides at the top of a certificates chain, and is used to sign the certificates of CAs that are subordinate to them, for example intermediate CAs.  |
| [Intermediate certificate authority](/docs/secrets-manager?topic=secrets-manager-intermediate-certificate-authorities) | A subordinate or lower-level certificate authority that signs and issues other intermediate CA certificates. An intermediate CA is also used to issue leaf certificates to end-entities, for example a client or server application. In {{site.data.keyword.secrets-manager_short}}, you can create intermediate CAs that are [signed internally or externally](/docs/secrets-manager?topic=secrets-manager-intermediate-certificate-authorities#intermediate-ca-signing).|
{: caption="Table 1. Certificate authority options" caption-side="top"}


### Planning the structure of a CA hierarchy
{: #plan-ca-structure}

As a best practice, plan a hierachy of certificate authorities that corresponds with the structure of your organization. In most cases, you can implement one of the following common CA structures.

#### Two levels: Root CA and subordinate CA
{: #two-level-ca}

Use this option if your workload requires the simplest CA structure. In this scenario, you create a single trusted root CA. Then, you create a subordinate CA, for example an [intermediate CA](/docs/secrets-manager?topic=secrets-manager-intermediate-certificate-authorities), to issue leaf certificates to your apps and services.

![The diagram shows a two-level hierarchy of certificates that begins with a root certificate authority in the first level.](images/certificate-hierarchy-2.svg){: caption="Figure 2. Two-level certificate authority hierarchy" caption-side="bottom"}

#### Three levels: Root CA and two subordinate CAs
{: #three-level-ca}

Use this option if your workload requires an additional layer between the root CA and lower-level CA operations. In this scenario, the middle subordinate CA is used only to sign subordinate CAs that issue leaf certificates to your apps.

![The diagram shows a three-level hierarchy of certificates that begins with a root certificate authority in the first level.](images/certificate-hierarchy-3.svg){: caption="Figure 3. Three-level certificate authority hierarchy" caption-side="bottom"}

  
### Setting the depth of your CA hierarchy
{: #set-max-path-length}

When you're creating a certificate authority in {{site.data.keyword.secrets-manager_short}}, you can set the **Maximum path length** parameter to determine how many CA certificates can exist in its authority chain. This value enforces how many CA certificates can exist in the certification path of your CA.

Generally, you can configure a root CA that doesn't limit the number of subordinate CAs that it can create in its certification path. However, defining a maximum path length on subordinate CAs is an important security step to avoid misconfigured CAs. Depending on the placement of a subordinate CA in your hierarchy, make sure to specify a maximum path length so that its signing authority power is limited to only the intended depth.

![The diagram shows a four-level hierarchy of certificates that begins with a root certificate authority in the first level.](images/max-path-length.svg){: caption="Figure 3. Three-level certificate authority hierarchy" caption-side="bottom"}

The maximum path length that you define does not include leaf certificates. In the example above, the subordinate CA in level 4 can issue leaf certificates but is unable to have more CA certificates in its path. 
{: note}

### Choosing a validity period for your certificates
{: #choose-validity-period}

The validity period of an X.509 certificate is a required field that determines how long the certificate is trusted and remains valid. When you plan your CA hierarchy, work backwards from your preferred lifespan for the leaf certificates that you want to issue to your applications. Then, determine the validity period of the CA certificates.

A certificate must have a validity period that is shorter than or equal to the validity period fo the CA that issued it. For example, if you create a root CA with a time-to-live (TTL) of 10 years, any intermediate CAs that are subordinate to it must have a TTL that is equal to or less than 10 years. Likewise, if an intermediate CA has TTL of 3 years, any leaf certificates must have a TTL that is equal to or less than 3 years.
{: important}

1. Choose a validity period for your leaf certificates that is appropriate for your use case.

   The private certificates that you can create with {{site.data.keyword.secrets-manager_short}} are considered leaf certificates that can be issued to an end-entity, such as a client or server app. With {{site.data.keyword.secrets-manager_short}}, you can create certificates with a maximum validity period of three years or 36 months. After you determine a TTL or validity period for leaf certificates, you set the value using a [certificate template](/docs/secrets-manager?topic=secrets-manager-certificate-templates) so that your preferred TTL is applied each time a new leaf certificate is generated.

   The shorter the TTL of your leaf certificates, the more protected you are against inadventent exposure or compromise of your certificate and its private key. A shorter validity period for your certificates means that you reduce the likelihood of compromise, but it also requires that you rotate the certificate more frequently to ensure that it stays valid. To avoid an inadvertent outage, you can schedule [automatic rotation](/docs/secrets-manager?topic=secrets-manager-automatic-rotation) of your private certificates.
   {: note}

2. Choose a validity period for the subordinate CA.

   As a best practice, set a validity period for a subordinate CA certificate that is signitificantly longer than the validity periods of the certificates that they issue. Define a validity period of a parent CA that is two to five times the period of any child CA certificate or leaf certificate that it issues. For example, if you have a [two-level CA hierarchy](#two-level-ca) and you want to issue leaf certificates with a TTL of one year, configure the subordinate issuing CA with a TTL of three years. You can always update subordinate CA certificates without replacing the root CA certificate.

3. Choose a validity period for your root CA.

   When you update a root CA certificate, the change impacts your entire public-key infrastructure. To minimize impact, it is recommended that you set a long validity period for your root CA certificate. In {{site.data.keyword.secrets-manager_short}}, the default TTL for root certificates is 10 years.


### Choosing an algorithm for generating keys
{: #choose-key-algorithm}

Before you create a certificate authority in {{site.data.keyword.secrets-manager_short}}, you must choose a key algorithm for generating the public and private keys for your CA. The public and private key-pair is used to authenticate an SSL/TLS connection. If you're not sure where to start, you can use the following suggested guide for selecting a key algorithm.

1. Choose an algorithm family.

    The key algorithm that you select determines the encryption algorithm and key size to use to generate keys and sign certificates. As a best practice, use the same algorithm family for all certificates that belong to a certificate chain. {{site.data.keyword.secrets-manager_short}} supports the following families of algorithms.

    | Algorithm family | Description | Supported key sizes |
    | --- | --- | --- | 
    | RSA | Widely used and compatible with most browsers and servers, RSA is the industry standard for public-key cryptography. | 2048 bits  \n4096 bits|
    | Elliptic curve (EC) | Generates stronger keys and smaller certificates. For example, a 256-bit EC key is equilavent in encryption strength to a 3072-bit RSA key. |  224 bits  \n256 bits  \n384 bits  \n521 bits|
    {: caption="Table 2. Supported algorithm families and key sizes" caption-side="top"}

2. Choose a key size.

   The key size or length that you select determines the encryption strength. The larger the key size for an algorithm family, the more difficult it is to break. Keep in mind that longer key lengths results in more data to store and transmit, which can impact the performance of your certificate. As a best practice, choose a key size that is appropriate for the TTL or validity period of your certificate.
   
   For longer living certificates it is recommended to use longer key lengths to provide more encryption protection.
   {: tip}

## Next steps
{: #prepare-create-certificates-next-steps}

Now you're ready to create certificate authorities for your instance.

- [Create a root certificate authority](/docs/secrets-manager?topic=secrets-manager-root-certificate-authorities)
- [Create an intermediate certificate authority](/docs/secrets-manager?topic=secrets-manager-intermediate-certificate-authorities)
- [Add a certificate template](/docs/secrets-manager?topic=secrets-manager-certificate-templates)

