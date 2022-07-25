---
title: "Secure the Supply Chain"
date: 2022-07-10T20:11:49-06:00
draft: true
---

If you examine cloud services being built today, you are likely to find a mix of proprietary and open source software. In order to maintain secure systems, it is important to know what software we are using and to make sure that we are actually using what we think we are using! Each dependency added to a system complicates this picture and introduces some amount of risk. In recent years, there have been a number of high profile security incidents that targeted companies and governments through their software supply chains that have unfortunately converted these risks into real issues. 

The Cloud Native Computing Foundation (CNCF) Security Technical Advisory Group (TAG) has produced a set of [supply chain security paper](https://github.com/cncf/tag-security/blob/main/supply-chain-security/supply-chain-security-paper/CNCF_SSCP_v1.pdf) in response to the increasing frequency of these incidents. The supply chain security paper contains in-depth recommendations to help organizations mitage these increasing risks. These recommendations focus on many aspects of producing and consuming software, from securing source code and the artifacts produced to deploying your build enviroments and deployments. This document has a lot of information to digest and it can be difficult to know where to begin. To help you get started, the CNCF Security TAG has also produced a reference [supply chain assessmet](https://github.com/cncf/tag-security/blob/main/supply-chain-security/supply-chain-security-paper/secure-supply-chain-assessment.md) that can help identify software supply chain security gaps. 

One important take away from the CNCF Security TAG supply chain security paper is that that when using open source software, it is recommended that an "analysis must be conducted to ensure the level of
assurance provided by an open source producer matches the required level of the consumer." Within Microsoft Azure, we have implemented a number of the CNCF Security TAG's recommendations in order to ensure that open source projects used within Azure services are consumed in a way that complies with the Microsoft [security development lifecycle](https://www.microsoft.com/en-us/securityengineering/sdl/about). 

# Securing Materials

* Building from source
* Vulnerability scanning and patching
* Review build pipelines and require review for all changes produced

# Signing

In our open source build pipelines, we sign container images and binaries, like kubectl, with the same code signing service used to sign device drivers and other things for Windows. This service produces container signatures using the Concise Binary Object Representation (CBOR) Object Signing and Encryption (COSE) protocol, that can be verified using the [Notary](https://notaryproject.dev) project's [notation](https://github.com/notaryproject/notation) client and the [notation cose](https://github.com/microsoft/notation-cose) plugin.

Notary can produce signatures for any [OCI artifact](https://github.com/opencontainers/artifacts) and signatures can be stored along side images in an OCI conformant registry. Once the signatures are published, they can be viewed in the [Microsoft Artifact Registry](https://mcr.microsoft.com) using the [OCI Registry as Store Client (ORAS)](https://oras.land/cli/6_reference_types/). For example, let's take a look at a Kubernetes API Server image for the 1.24 release.

```
$ oras discover -o tree mcr.microsoft.com/oss/kubernetes/kube-apiserver:v1.24.2
mcr.microsoft.com/oss/kubernetes/kube-apiserver:v1.24.2
└── application/vnd.cncf.notary.v2.signature
    └── sha256:5367b984d7ec0e6f7281eab551eff957e5349f1252a59b03e3c8e71670aea689
```

This signature can be validated using the public key for Microsoft's supply chain certificate:

```
notary verify here
````

# SBOM

In May of 2021, the White House released an [executive order](https://www.whitehouse.gov/briefing-room/presidential-actions/2021/05/12/executive-order-on-improving-the-nations-cybersecurity/) focused on improving cybersecurity within the United States. One focus of this executive order was the production of a Software Bill of Materials (SBOM) for all software products. The CNCF Security Tag also recommends that it is a best practice to generate an SBOM for any open source projects that you rebuild. 

Our open source software pipelines now produce an SBOM for each container image we produce. Using [Syft](https://github.com/anchore/syft), we produce an SBOM using the Software Package Data Exchange (SPDX) format for all containers produced by our build process. These SBOMs contain information about the software packages used during the build, as well as information about the OS-level pacakges used within the containers. These SBOMs are stored as OCI artifacts in the Microsoft Artifact Registry along side the container images they reference. Storing the SBOM along side the container image makes them very easy to find. The ORAS cli can be used to find the SBOM for each container image:

```
oras discover -o tree mcr.microsoft.com/oss/kubernetes/kube-apiserver:v1.25.0-alpha.3-linux-amd64
mcr.microsoft.com/oss/kubernetes/kube-apiserver:v1.25.0-alpha.3-linux-amd64
└── application/spdx+json
    └── sha256:cb0050db894b7c9f94807b094bc7e45a5e44c6268a2cb16d75aeb5085e3674c3
```

Like our container images, these are also signed using the Microsoft code signing service. This means that services consuming the these images can verify the authenticity of not only the image, but of the SBOM as well. 

