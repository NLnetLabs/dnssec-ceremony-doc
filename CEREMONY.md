# DNSSEC Key Signing Suite Documentation
## Key Signing Ceremonies: Requirements, Options and Implementation

Copyright (c) 2019-2020 NLnet Labs  
Released under Creative Commons CC 4.0 BY-SA (see [```LICENSE```](LICENSE))

### Funding Acknowledgement

Work on the DNSSEC Key Signing Suite was supported by a grant from the European Commission managed by the NLnet Foundation ([NGI0 PET project](https://nlnet.nl/PET/)).

## About this document

This document is intended for operators of high value domains that deploy, or intend to deploy DNSSEC, and that want to take security measures to protect their DNSSEC key material. In particular, this document is intended for setting up protection measures for so-called "offline Key Signing Keys", in which the root signing key in a domain is stored away from the DNSSEC signing system, or possibly even "air gapped".

We assume readers are familiar with DNSSEC terminology, in particular with the concepts of Key Signing Keys (KSKs), Zone Signing Keys (ZSKs) and key rollovers.

### Document organisation

This document consists of two parts. The first part discusses collecting requirements for your key ceremony. The second part discusses the building blocks of a key ceremony that can be combined to implement your own ceremony. It discusses human-centered processes and technical implementation steps separately. 

## Key Ceremony Requirements and Recommendations

### Introduction

Various operators of top-level domains and e.g. the DNS Root operator [[1]](https://www.iana.org/dnssec/procedures/ksk-operator/ksk-dps-20201104.html) have already deployed offline KSKs with accompanying ceremonies. In most cases, these ceremonies consist of a human process (often including public scrutiny by community members) and a technical process (in which the actual key operations take place).

While the human processes and technical processes typically depend on each other, the requirements for each process may differ from operator to operator. Additionally, we identify two high-level roles in the human process, that need to be clearly separated. The first role is that of a technical operator. People in this role operate equipment, run commands, optionally verify the technical soundness of the outcome of various steps and ensure timely execution of the process. The second role is that of a community or stakeholder witness. People in this role verify that the ceremony is carried out according to the pre-defined requirements and verify the output of various steps of the process.

In the remainder of this section we will discuss which requirements and recommendations you should consider for each part of the ceremony process and for every role. We label requirements according to the specific process and role they belong to, with the following labels:

 * *OP* - human technical operations;
 * *WS* - witness or stakeholder public scrutiny process;
 * *TG* - technical process (general);
 * *TZ* - technical process (ZSK keyset signing ceremony);
 * *TK* - technical process (KSK rollover ceremony).

These labels are used to refer back to the requirements and recommendations when we discuss the ceremony building blocks further down the document.

### Human processes

#### Technical operations

In order to carry out the technical operations process, you should take the following requirements into consideration:

***OP1*** *- technical staff must be trained to understand DNSSEC.*  
Explanation: this will help the staff troubleshoot any problems that may occur during the process, and also allows them to partially fulfil the role of community or stakeholder witnesses in case these are not part of the ceremony.

***OP2*** *- technical staff must be trained to operate the Hardware Security Module(s) used for storing and processing key material.*  
Explanation: we assume that in most cases where key material is stored offline that one or more HSMs will be used (see the requirements on the technical process further down). HSMs are devices that are not frequently encountered by system administrators and are typically difficult to operate correctly. Add to this that incorrect operation of an HSM may lead to it zeroing out key material, makes it important that all operators are properly trained.

***OP3*** *- staff must be available for up to a day for each ceremony.*  
Explanation: depending on where the ceremony physically takes place, staff may need to travel to/from this location. In addition to this, if public scrutiny is part of the process, time needs to be allowed for witnesses to arrive at the location and to prepare for the ceremony.

***OP4*** *- there needs to be a minimum of at least two trained technical staff members.*  
Explanation: this may seem obvious, but we point it out nevertheless. Given how important continuity of key management is for the operation of a high-value domain, having at least two trained staff at all times is the bare minimum.

***OP5*** *- you should consider security vetting and/or screening of technical staff.*  
Explanation: depending on the requirements of your constituency, your key management process may need to be audited. As part of this audit, it is likely that requirements are put on security qualifications of the staff that perform the key ceremony technical operations. Given that security vetting or screening typically takes up to several weeks, we include this requirement to remind you to start this process on time.

#### Public scrutiny process

For the public scrutiny process, we identify the following potential requirements:

***WS1*** *- you must decide if public scrutiny will be a part of your ceremony.*  
Explanation: it is not a given that public scrutiny needs to be part of your ceremony. While existing examples of key ceremonies (such as the one for the DNS Root) include public scrutiny, it is important to realise that there is no technical or security need to include public scrutiny. The most important function of public scrutiny during a key ceremony is to invest external trust in the key material and to be accountable to stakeholders or the community a domain serves. Representatives of stakeholders or the community follow the process and sign off on the generated key material. At any future point in time, anyone who wishes to check the authenticity of the key material can then rely on the trust invested by stakeholder or community representatives. If, however, you have no expectation or need for external trust to be invested in key material, you can completely leave out public scrutiny, as this will simplify the key ceremony process significantly. Optionally, you can decide to use a more lightweight approach by having internal scrutiny by stakeholders from within your organisation (e.g. a security office, or the management team).

***WS2*** *- you must decide who will perform the public scrutiny: stakeholders or community representatives.*  
Explanation: in this set of documents, we refer to stakeholders if these have a business relationship to you as a DNS operator. If, for example, you operate domains on behalf of customers, they are stakeholders and may wish to be involved in the key ceremony process. Or, for example, if you operate a TLD with which registrars have a membership relation, you may wish to have a representation from your members scrutinise the key ceremony. When we talk about community representatives, we typically mean people that are completely independent from your organisation, but who have a stake in the domain that you operate in some other way. For example, if you operate a country-code top-level domain citizens of the country may be community representatives. The biggest difference between stakeholders and community representatives is how these are appointed (see next requirement).

***WS3*** *- there must be a process for picking stakeholder or community representatives.*  
Explanation: for the continuity of your key ceremony process, it is important that it is clear who will take on public scrutiny duties, for what period, and how they are (re-)appointed. We also recommend that you define an exception process on what to do in case there is a problem in re-appointing or replacing people that perform public scrutiny duty.

***WS3.1*** *- define how stakeholders appoint people for public scrutiny duty.*  
Explanation: in the case of stakeholder representatives, we expect there to be a process in which these representatives are appointed by the respective stakeholder(s). We strongly recommend that this process is put on paper, or - if applicable - made part of the contract with stakeholders.

***WS3.2*** *- define how community representatives are selected for public scrutiny duty.*  
Explanation: in the case of community representatives we expect there to be some sort of election process in which representatives are chosen by a community. Selection criteria differ between communities so are considered out-of-scope for this document. We recommend establishing community bylaws that at a minimum establish: 

1. how potential representatives are nominated for the role;
2. how representatives are selected;
3. under what conditions they serve;
4. how long their term of service is;
5. how many subsequent terms they may serve. 

We also strongly recommend making clear that while serving as community representative is voluntary, it does come with responsibilities to attend ceremonies.

***WS4*** *- define a quorum for the number of witnesses and an escalation procedure.*  
Explanation: it is likely that not all of the selected witnesses can attend every ceremony. We therefore recommend setting a required minimum number of witnesses that is lower than the actual list of current witnesses. We also strongly recommend setting up a well-documented escalation procedure that can be followed in case it is not possible to gather a quorum, but a key ceremony must take place to guarantee the continuity of a domain. An escalation procedure could, for example, temporarily delegate the responsibility to witness a key ceremony to your board of directors.

***WS5*** *- establish training requirements for witnesses.*  
Explanation: your witnesses need to understand what they are witnessing. While it may be infeasible to explain DNSSEC and cryptography to them in detail, they should at least know what materials they are supposed to inspect and how they can check these materials. For more information on what materials should be output, see the requirements and recommendations for the technical process below.

***WS6*** *- publish information about your public scrutiny process and the people involved.*  
Explanation: an important reason to have public scrutiny is accountability. It is therefore important to show publicly what process you apply and who your witnesses are. In case your witnesses come from stakeholders, you need to discuss with them what information will be disclosed about witnesses. In case you work with community representatives, the trust is vested in the fact that these people are selected by and known to the community. Therefore, the names and contact details for community representatives must be accessible to the community they represent. A well-known location on your website is, for example, a good location to publish this information.

### Technical process

As we will discuss below for the design and implementation, we foresee two types of key ceremonies. The first type is a keyset signing ceremony, in which new Zone Signing Keys are generated and a set of signed DNSKEY sets is generated for the day-to-day operation of a domain. The second type is a Key Signing Key rollover ceremony, in which a new KSK is generated, DNSKEY sets are generated to perform the rollover and a new DS is generated for publication in the parent zone. In the remainder of this section, we first discuss generic technical requirements and recommendations, followed by a discussion of the requirements and recommendations for a keyset signing ceremony and ending with the KSK rollover ceremony.

#### General Technical Requirements and Recommendations

***TG1*** *- select Hardware Security Modules(s) to use.*  
Explanation: to protect key material, many operators of high-value domains use one or more Hardware Security Modules. While it is out-of-scope to provide an exhaustive set of criteria for HSM selection, we provide a list of common considerations to take into account:

 * Do you need a "real" HSM (hardware), or does a software solution, such as SoftHSM [[4]](https://github.com/opendnssec/SoftHSMv2), suffice?
 * If you use a physical HSM, will you use a single vendor, or will you use HSMs from multiple vendors? (*N.B.: using HSMs from multiple vendors potentially makes key management very difficult, so unless there is a good reason to do this, we strongly recommend against it*)
 * How many HSMs will you use and what provisions do they have to securely synchronise key material? (many HSM vendors provide a means to share sensitive material between different instances, consult the documentation of the HSM to find out how)
 * How will you retire and replace HSMs over time?
 * How will you deal with the catastrophic failure of an HSM?
 * Do you intend to back up key material stored in HSMs, and if so, what provisions does your selected HSM model provide for this purpose?
 * Unless you manage thousands of domains, your HSM probably does not have to be very fast (this may save cost and make selection easier).
 * Does your HSM support your current and future DNSSEC signing algorithms? (pay particular care in case you have plans to switch to newer elliptic curve algorithms such as `Ed25519` or `Ed448` in the future)

***TG2*** *- select your DNSSEC signing algorithm.*  
Explanation: the DNSSEC signing algorithm influences the size of DNS responses (```DNSKEY``` set and size of ```RRSIG```). Ideally, at any stage during operations DNS responses should fit within the Maximum Transmission Unit (MTU). This constraint is strongly influenced by the selected signing algorithm. For newly signed zones, we strongly recommend using an elliptic curve signing algorithm, for maximum compatibility and a good tradeoff between speed, size and security we currently recommend using ECDSA P-256 with SHA256 (algorithm 13). In case you are instituting a key ceremony for an existing zone that is signed using RSA keys, we strongly recommend paying extra attention to time schedules (see the Design and Implementation section below). We also recommend consulting current guidance on DNSSEC algorithms provided by the IETF [[5]](https://tools.ietf.org/html/rfc8624).

***TG3*** *- select your key rollover schedule.*  
Explanation: we will go into more detail for exact timing, but it is important to think beforehand about the approximate frequency at which you wish to change keys. For example: changing KSKs on an annual basis and changing ZSKs on a quarterly basis. The rough schedule is input for the design and implementation process.

***TG4*** *- select your preferred key ceremony days.*  
Explanation: we strongly recommend that you try to pick a fixed day of the week on which to have your key ceremony. The rationale behind this is that it creates a predictable pattern for system administrators and any witnesses to your ceremony. Also, we recommend picking a day of the week that allows for additional working time left in the week to fix any issues or to reschedule the ceremony if required. We recommend always leaving one day between your ceremony and the weekend or any public holidays. In practice, in many countries this means scheduling the ceremony on Tuesdays, Wednesdays or Thursdays.

***TG5*** *- select your timing parameters.*  
Explanation: it is important to think about timing beforehand. The most important parameter is signature validity and refresh intervals. Unless you have good reasons to do otherwise, we recommend making signatures valid for at least 14 days and to refresh them 5 days before they expire. The rationale behind this is what we call the "Easter holiday" scenario. In many countries, the Christian Easter Holiday includes Good Friday and Easter Monday, and is typically the longest uninterrupted holiday period composed solely of public holidays. In this scenario, if signatures are always refreshed 5 days before they expire, if there is some sort of failure by closing of business on Whit Thursday (the day before Good Friday), then there is no strict need for on-call engineers to intervene, and problems can be resolved on the Tuesday after Easter. Other timing parameters to consider include ```SOA``` parameters (see Design and Implementation below). The IETF also provides guidance on timing parameters [[6]](https://tools.ietf.org/html/rfc7583)

***TG6*** *- divide your time schedule into fixed slots.*  
Explanation: To ease planning your ceremonies, we strongly recommend using fixed time slots (e.g. calendar weeks, months, quarters, ...) as planning units. The Root DNS KSK operator, for example, uses a slot-based schedule.

***TG7*** *- plan for exceptions.*  
Explanation: There may be a plethora of reasons why a scheduled key ceremony may have to be moved in time. Plan for this in advance by ensuring that your key ceremonies take place well before the newly generated materials are needed in production, and always allow for a grace period of several slots.

***TG8*** *- plan for ZSK key compromise.*  
Explanation: While a compromise of an offline KSK is extremely unlikely, your ZSK will likely be online in operation, and as such may be more vulnerable to potential compromise. Even though you should take adequate security measures to limit the likelihood of a ZSK compromise, we nevertheless also recommend that you plan for it happening anyway. To migitate the impact, we recommend keeping any material that "blesses" the ZSK separate from your operational system. In practice, this means keeping the set of signed ```DNSKEY``` RRsets for the current planning period on a separate system, and to only feed in new ```DNSKEY``` RRsets to your signer when needed. A compromise of the signer system will then not automatically result in a compromise for the whole operating period of that ZSK. In essence, what this means is that you should also treat the set of pre-signed ```DNSKEY``` RRsets as sensitive information. We realise that current signer implementations may not support this, so this is an optional recommendation and a call to all signer implementers that (intend to) support offline KSK scenarios to support this type of functionality.

***TG9*** *- plan for an emergency ceremony.*  
Explanation: While you would typically not deviate from your planned rollover and accompanying ceremony schedule, there may be reasons for an emergency ceremony. A ZSK compromise is one reason for such an emergency ceremony, but, e.g., the catastrophic failure of a signer system can also mean new key material is needed urgently. We recommend that you plan for such an emergency ceremony, taking into consideration which actors need to be present, how the emergency materials can be taken into production and how such an emergency affects the regular ceremony and rollover schedule.

#### ZSK Keyset Signing Ceremony

In this subsection we discuss specific requirements and recommendations for the ZSK Keyset Signing Ceremony. In this ceremony, one or more new ZSKs are generated and a set of ```DNSKEY``` RRsets is pre-generated, signed with the KSK and reflecting the ZSK rollover schedules.

***TZ1*** *- decide where the ZSK is generated.*  
Explanation: The ZSKs that will be included in ```DNSKEY``` RRsets that are signed during this ceremony need to be generated. This can either happen on the DNSSEC signer system (out of the scope of the offline key management facility), or it can happen during the ceremony. In the first case, the public keys of the pre-generated ZSKs form the input to the ceremony, in the second case, generation of the ZSKs and exporting them such that the DNSSEC signer system can import them is part of the ceremony. Some considerations to take into account when choosing a model:

 * ZSKs generated by the signer system may make integration with existing signer software simpler (see, e.g., [[2]](https://indico.dns-oarc.net/event/31/contributions/689/attachments/670/1100/KNOT-offline.pdf)).
 * It may be more secure to have the ZSKs generated by the HSM during the signing ceremony, as a real HSM is expected to have a more secure random number generator.

***TZ2*** *- decide on a transport model for ZSKs.*  
Explanation: This applies in both the case where ZSKs are generated on the DNSSEC signers, and in the case where the ZSKs are generated during the ceremony.

***TZ2.1*** *- transport model for public keys for externally generated ZSKs.*  
Explanation: if ZSKs are generated externally (on the DNSSEC signer system), then the system used to execute the keyset signing ceremony needs to be able to verify the provenance of the provided ZSKs. The best approach to do this is to provide a file containing the ```DNSKEY``` RRsets for the period for which keysets must be signed. This file should then be signed, and the public key belonging to the signing key should be known to the system used to execute the keyset signing ceremony. For an example of this approach, see [[2]](https://indico.dns-oarc.net/event/31/contributions/689/attachments/670/1100/KNOT-offline.pdf).

***TZ2.2*** *- transport model for ZSKs generated during the ceremony.*  
Explanation: if ZSKs are generated during the ceremony, then they are most likely generated using the HSM(s). In this case, the ZSKs must be transported securely to the signer system(s). The most compatible way to do this is to set up a (software) HSM on the signer system, and to use a standards-based approach to export the ZSKs securely from the HSM and then to import them into the local (software) HSM on the signer system. HSMs generally use the PKCS #11 standard for access and can wrap keys in a PKCS #8 structure. The toolset provided with the DNSSEC Key Signing Suite project supports this model.

***TZ3*** *- choose a key rollover model.*  
Explanation: there are different key rollover approaches for ZSKs (for an exhaustive description see [[3]](https://tools.ietf.org/html/rfc6781#section-4.1.1)). Unless there are specific reasons to choose a different model, we recommend using the *ZSK pre-publish scheme* with gradual signature replacement. All of the examples in the DNSSEC Key Signing Suite documentation and all of the tools provided assume the *ZSK pre-publish scheme* is used.

#### KSK Generation, Signing and Rollover Ceremony

***TK1*** *- choose a key rollover model.*  
Explanation: there are different key rollover approaches for KSKs (for an exhaustive description see [[3]](https://tools.ietf.org/html/rfc6781#section-4.1.2)). Unless there are specific reasons to choose a different model, we recommend using the *KSK double DS scheme*, in which the ```DNSKEY``` set is signed with only one KSK and two ```DS``` records are published in the parent zone during the rollover. The advantage of this approach is that it reduces the size of the ```DNSKEY``` RRset during the rollover. All of the examples in the DNSSEC Key Signing Suite documentation and all of the tools provided assume the *KSK double DS scheme* is used.

***TK2*** *- decide whether KSK and ZSK rollovers may overlap.*  
Explanation: depending on what DNSSEC algorithm is used, allowing KSK and ZSK rollovers to overlap may lead to very large DNS responses to ```DNSKEY``` queries. This in turn may lead to response packet fragmentation, which is known to cause DNS resolution problems. The requirements on processes vary between environments, so it may not be possible to always separate rollovers, but if you do use RSA as signing algorithm, we recommend that you strongly consider not allowing KSK and ZSK rollovers to overlap.

## Key Ceremony Design and Implementation Building Blocks

### General guidance

When discussing the building blocks, the text will cleary indicate whether these:

 * **[W]** apply only to witnessed ceremonies
 * **[TW]** apply both to technical and witnessed ceremonies

### Step 1 [W] - Scheduling a ceremony and inviting witnesses

In case of witnessed ceremonies, the first step is to perform timely scheduling of a ceremony, and to invite witnesses to participate. Key elements to consider in this step are:

 * Impose a deadline for witnesses to RSVP to the invitation to participate so you know well in advance how many witnesses will attend;
 * Verify well in advance that you have a quorum of witnesses; (see also _WS4_)
 * Send witnesses an agenda for the ceremony that details each of the steps you plan to execute;
 * If your policy requires this, publicly announce your planned ceremony to your community.

### Step 2 [TW] - Technical preparation

In this step, all technical preparations for the ceremony will take place. The exact preparation depends on the technical choices you have made (such as where certain key material is generated, what rollover mechanism you use for ZSKs and KSKs, timing parameters). In general, however, it will likely involve some of the following elements:

 * Generating new ZSKs;
 * Preparing `DNSKEY` sets to be signed;
 * Putting together a recipe for the key ceremony to be executed in the secure environment (see also [`RECIPE-API.md`](./RECIPE-API.md) for more details);
 * Preparing your signer to receive newly signed keysets created in the secure environment.

### Step 3 [W] - Authentication of witnesses

For a witnessed ceremony, you may have a process for establishing the identity of witnesses. In addition to this, you may need to verify that the witnesses have everything they need in order for them to participate in the ceremony. In particular, you may need to check if, for example, they have the authentication tokens (smart cards or USB keys) to activate the Hardware Security Module, if you use such an authorisation model.

### Step 4 [W] - Verification and witnessing of ceremony recipe

In this part of a witnessed ceremony, witnesses will be shown the ceremony recipe that is to be executed. They should, for example, verify that the recipe matches the type of ceremony that was pre-announced (either ZSK or KSK ceremony), the number of new keys to be introduced, and, if keys are generated outside of the secure environment, should sign off on the exact keys that will be input for signing as part of the ceremony, such that this information can later be published as a public record.

### Step 5 [TW] - Execute the ceremony recipe

While the details of this step depend on the technical implementation, typically this step is largely automated, except for the situation where witnesses need to authenticate to the HSM to activate it. We envisage running an application that can execute a prepared recipe (that, e.g., follows the specification in [`RECIPE-API.md`](./RECIPE-API.md)).

A technical operator needs to verify if the recipe has been executed successfully and without errors.

### Step 6 [W] - Sign off on the output

While the output of the ceremony may partly depend on design choices (such as where keys are generated), typically, the output consists of signed `DNSKEY` sets which are valid for a certain period of time. Witnesses should sign off on these keysets, such that the output of the ceremony, together with the witness statements can be published as a public record.

### Step 7 [TW] - Import the output of the ceremony into the signer

In the final step, the output of the ceremony should be imported into the signer, such that the newly generated `DNSKEY` sets can be used in upcoming changes.

## References

[1] IANA, "DNSSEC Practice Statement for the Root Zone KSK Operator", [<https://www.iana.org/dnssec/procedures/ksk-operator/ksk-dps-20201104.html>](https://www.iana.org/dnssec/procedures/ksk-operator/ksk-dps-20201104.html)

[2] Jaromir Talomir, "Offline KSK with Knot DNS 2.8", DNS-OARC 30, Bangkok, Thailand, [<https://indico.dns-oarc.net/event/31/contributions/689/attachments/670/1100/KNOT-offline.pdf>](https://indico.dns-oarc.net/event/31/contributions/689/attachments/670/1100/KNOT-offline.pdf)

[3] Olaf Kolkman, Matthijs Mekking and Miek Gieben, "RFC 6781 - DNSSEC Operational Practices, Version 2", IETF, [<https://tools.ietf.org/html/rfc6781>](https://tools.ietf.org/html/rfc6781)

[4] SoftHSM v2, [<https://github.com/opendnssec/SoftHSMv2>](https://github.com/opendnssec/SoftHSMv2)

[5] Paul Wouters and Ondrej Sury, "RFC 8624 - Algorithm Implementation Requirements and Usage Guidance for DNSSEC", IETF, [<https://tools.ietf.org/html/rfc8624>](https://tools.ietf.org/html/rfc8624)

[6] Stephen Morris, Johan Ihren, John Dickinson and Matthijs Mekking, "RFC 7583 - DNSSEC Key Rollover Timing Considerations", IETF, [<https://tools.ietf.org/html/rfc7583>](https://tools.ietf.org/html/rfc7583)
