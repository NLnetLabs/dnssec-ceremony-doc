# DNSSEC Key Signing Suite Documentation

Copyright (c) 2019 NLnet Labs  
Released under Creative Commons CC 4.0 BY-SA (see ```LICENSE```)

### Funding Acknowledgement

Work on the DNSSEC Key Signing Suite was supported by a grant from the European Commission's managed by the NLnet Foundation ([NGI0 PET project](https://nlnet.nl/PET/)).

## Preamble

The Domain Name System Security Extensions (DNSSEC) increase trust in the Domain Name System (DNS) by adding authenticity and integrity to the protocol. While originally designed to improve the security of the DNS alone, with the advent of DNS-based Authentication of Named Entitities (DANE) DNSSEC is increasingly used to improve trust in other Internet services (such as, e.g., e-mail). 

The root of the trust in DNSSEC is vested in the cryptographic keys that are used to sign DNS zones. For operators of high-value domains - such as, for example, top-level domains, governmental domains or high-value enterprise domains - it is important to handle this sensitive DNSSEC key material securely. While there exists a plethora of approaches to managing DNSSEC key material, often highly specific to the environment in which they are deployed, there is no generic approach, nor an overview of requirements or best practices. T

he goal of the DNSSEC Key Signing Suite project is to provide such a generic approach, and in particular, to describe an approach for so-called "offline KSKs", where the Key Signing Key for a domain is kept offline and only used during special key signing ceremonies to sign the ```DNSKEY``` record sets for a number of future Zone Signing Keys (ZSKs). We break this down into two parts: 1) an operational part in the form of a key signing ceremony that can be tailored to the specific needs of an environment and 2) a set of UNIX command-line tools that can support this ceremony at various stages.

## Audience and Scope

The audience for this project consists of managers and engineers involved in the management of high-value domains (such as, but not limited to, top-level domains, governmental domains, ...). Readers are assumed to be familiar with DNSSEC and its terminology.

The scope of this project's documentation is limited to DNSSEC key ceremonies and technical key management. DNSSEC signer operations are out of scope, although certain DNSSEC and DNS parameters are required as input to certain parts of the ceremony and may need to be specified to the technical key management tools.

## Reading Guide

This repository contains the following documents:

 - [```CEREMONY.md```](CEREMONY.md) - this document describes what to take into consideration when designing a ceremony and provides boiler plate approaches to the various stages. We recommend that you at least read the section on considerations before choosing an approach to your own key ceremony. You can then decide whether or not to use one of the example approaches (see below) or to create your own custom ceremony based on the building blocks provided in this document.
 - [```CEREMONY-SIMPLE.md```](CEREMONY-SIMPLE.md) - this document contains a working example of a key ceremony that can be used in environments where there is no immediate need for public scrutiny or witnessing of the ceremony. It only describes the technical steps to take when working with offline keys.
 - [```CEREMONY-PUBLIC.md```](CEREMONY-PUBLIC.md) - this document contains a working example of a key ceremony that includes public scrutiny or witnessing by community representatives. It describes how to involve the witnesses in the process as well as the technical implementation of the key ceremony.