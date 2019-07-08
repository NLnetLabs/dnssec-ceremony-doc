# DNSSEC Key Signing Suite Documentation
## Key Ceremony Recipe API specification

Copyright (c) 2019 NLnet Labs  
Released under Creative Commons CC 4.0 BY-SA (see [```LICENSE```](LICENSE))

### Funding Acknowledgement

Work on the DNSSEC Key Signing Suite was supported by a grant from the European Commission managed by the NLnet Foundation ([NGI0 PET project](https://nlnet.nl/PET/)).

## About this document

This document specifies the file format for DNSSEC Key Signing Ceremony Recipes. A recipe is a single document that specifies all of the actions that must be performed in the secure environment in which offline key material is stored and in which the key signing ceremony takes place. The recipe file format is used by the tools included in the DNSSEC Key Signing Suite.

## Terminology

We foresee this document potentially evolving into an Internet draft; for this reason, this document uses the key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY" and "MAY NOT" in accordance with the guidelines set out in RFC 2119 [[1]](https://tools.ietf.org/html/rfc2119).

## Introduction

The goal of the DNSSEC Key Signing Suite is to support owners of high value domains (such as top-level domains) with key ceremonies that rely on offline key material for the KSK. The most important part of the key ceremony, of course, takes place in some sort of secure environment, in which the offline key material is stored. Such an environment is often colloquially referred to as a "bunker". In order to support automation of key ceremonies, integration with existing signer systems and to minimise the risk of human error, we specify a file format that we call a "Key Ceremony Recipe" (or "recipe" for short). The goal of this file format is to unambiguously communicate to systems in the "bunker" what actions they need to perform.

The basic design rationale behind the Key Ceremony Recipe is that we want to avoid (as much as possible) the need for control logic in the "bunker". That is: a recipe is a set of instructions that must be carried out verbatim, unconditional, and either the whole recipe succeeds, or the whole recipe fails in an atomic manner. A secondary goal that we follow when implementing the tooling related to processing of recipes is that we strive to minimise dependencies in the "bunker", so as to keep the software running in that environment as concise as possible.

## Recipe layout

From a high level perspective, each recipe contains the following elements:

 * a preamble, which specifies common parameters;
 * one or more actions (see below);
 * an optional digital signature.

If a recipe is digitally signed, then it MUST be signed using the JSON Web Signature format specified in RFC 7515 [[2]](https://tools.ietf.org/html/rfc7515).

The outer structure of a recipe MUST contain the following fields:

 * ```recipeSpecVersion``` - the version of this specification, currently this is limited to ```v1.0```.
 * ```recipe``` - the actual recipe; this field MUST contain only a single recipe and MAY NOT contain an array.

An example of the outer structure of a recipe then looks like this:

```
{
	recipeSpecVersion: "v1.0",
	recipe: { ... }
}
```

## Recipe preamble

The preamble specifies common parameters that apply to the whole recipe. The following common parameters exist:

 * ```timestamp``` (mandatory) - UNIX epoch timestamp for when the recipe was created.
 * ```description``` (optional) - A free-form human readable description of the recipe that can be displayed to operators and witnesses at the ceremony.

An example of a recipe with a preamble section is specified below:

```
{
	recipeSpecVersion: "v1.0",
	recipe:
	{
		preamble:
		{
			timestamp: 1562582413,
			description: "An example key ceremony recipe"
		}, ...
	}
}
```

## Recipe actions

A recipe specifies the actions that need to be performed inside the "bunker". Recipes can contain the following types of actions:

 * ```haveKey``` - to verify the existence of a specified key in the "bunker".
 * ```deleteKey``` - to irrevocably delete a key from the storage in the "bunker".
 * ```generateKey``` - to generate a new key in the "bunker" according to the included specifications.
 * ```produceSignedKeyset``` - to produce a signed ```DNSKEY``` RRset that includes certain specified keys and is signed using specified keys.
 * ```importPublicKey``` - to import a public key to be used for exporting keys from the secure storage in the "bunker".
 * ```exportKeypair``` - to export a key pair generated in the secure storage in the "bunker" as an encrypted PKCS #8 object using a specified public key that was previously imported using the ```importPublicKey``` action.

Irrespective of the order in which the different action types are specified in the file, the different action types SHALL always be performed in the same order, which is all ```haveKey``` actions, followed by all ```deleteKey``` actions, followed by all ```generateKey``` actions, and ending with all ```producedSignedKeyset``` actions. Always using this order ensures that the preconditions for each action type are typically automatically met, and it also allows operators to perform a backup of the key storage medium immediately after each ceremony ends, and to be sure that the state that is backed up can always be re-used to produce the same signed keysets. To facilitate human readability of recipes, action types SHOULD always be specified in the order in which they are executed.

If multiple actions of the same type are specified, then these MUST be carried out in the order in which they are specified in the array of actions in the ```actions``` clause.

A generic syntax is used to specify each action. The ```actionType``` field specifies the action type, and the ```actionParams``` field specifies the parameters for that specific action. 

An example of a recipe with the most common action types is given below:

```
{
	recipeSpecVersion: "v1.0",
	recipe:
	{
		preamble: { ... },
		actions:
		[
			{ actionType: "haveKey", actionParams: { ... } },
			{ actionType: "haveKey", actionParams: { ... } },
			{ actionType: "deleteKey", actionParams: { ... } },
			{ actionType: "generateKey", actionParams: { ... } },
			{ actionType: "produceSignedKeyset", actionParams: { ... } },
			{ actionType: "produceSignedKeyset", actionParams: { ... } },
		]	
	}
}
```

The parameters for each of the action types are specified further down.

## Specifying keys

The recipe specification is designed to be flexible in how keys are specified, in particular keys that are part of a keyset that needs to be signed in a ```produceSignedKeyset``` action. Two models are supported, a model in which all keys, including ZSKs, are generated in the "bunker" , and a model in which some keys are generated outside of the "bunker", but need to be included in the signed keyset. This means that there are also two ways to specify keys in action parameter sets: by reference, or direct. The two ways to specify keys are shown in detail below:

**Key by reference:**

This way of referring to keys may be used in ```haveKey```, ```deleteKey```, ```generateKey``` and ```produceSignedKeyset``` actions.

Keys are specified as follows:

```
{
	keyType: "byRef",
	keyAlgo: INTEGER,
	keyFlags: INTEGER,
	keyStore: STRING,
	keyID: STRING,
	keyLabel: STRING
}
```

The meaning of each of the fields is as follows:

|**Field**|**Optional?**|**Meaning**|
|-----|---------|-------|
|```keyType```|Mandatory|Key type, MUST be set to ```"byRef"``` for keys by reference.|
|```keyAlgo```|Mandatory|Key algorithm, MUST specify a valid DNSSEC key algorithm (e.g. 13 for ECDSAP256SHA256).|
|```keyFlags```|Mandatory|DNSSEC key flags (e.g. 256 for ZSK, 257 for KSK).|
|```keyStore```|Mandatory|Opaque reference to where the key is stored. This could, for example, be the name of the HSM partition label.|
|```keyID```|Optional*|Base64-encoded byte string for a PKCS #11 ```CKA_ID``` that should be used to create or find the key.|
|```keyLabel```|Optional*|Label string for a PKCS #11 ```CKA_LABEL``` that should be used to create or find the key.|

*When specifying keys by reference at least one of ```keyID``` or ```keyLabel``` MUST be specified.

**Direct key specification:**

This way of referring to keys may only be used in ```produceSignedKeyset``` actions, in the ```keysetKeys``` clause.

Keys are specified as follows:

```
{
	keyType: "direct",
	keyAlgo: INTEGER,
	keyFlags: INTEGER,
	keyData: STRING
}
```

The meaning of each of the fields is as follows:

|**Field**|**Optional?**|**Meaning**|
|-----|---------|-------|
|```keyType```|Mandatory|Key type, MUST be set to ```"byRef"``` for keys by reference.|
|```keyAlgo```|Mandatory|Key algorithm, MUST specify a valid DNSSEC key algorithm (e.g. 13 for ECDSAP256SHA256).|
|```keyFlags```|Mandatory|DNSSEC key flags (e.g. 256 for ZSK, 257 for KSK).|
|```keyData```|Mandatory|Base64-encoded data from the Public Key RDATA field of the ```DNSKEY``` resource record for this key.|

## haveKey action specification

The ```haveKey``` action can be used as a pre-condtion in a recipe to verify the existence of certain keys in the secure storage of the "bunker". A ```haveKey``` action is specified as follows:

```
{
	actionType: "haveKey",
	actionParams:
	{
		key:
		{
			keyType: "byRef",
			...
		},
		relaxed: BOOLEAN
	}
}
```

The meaning of each of the fields is as follows:

|**Field**|**Optional?**|**Meaning**|
|---|---|---|
|```key```|Mandatory|The key to check for. Each ```haveKey``` action works for a single key and SHALL only refer to keys by reference.|
|```relaxed```|Optional|Indicates whether the ```haveKey``` action should be evaluated in a relaxed manner. If set to ```true```, then the action will not fail if the key is not present, if set to ```false``` then the whole recipe will fail if the key is not present. The default behaviour, if the field is not specified, is ```false``` (strict evaluation).|

## deleteKey action specification

The ```deleteKey``` action deletes a single key from the secure storage in the "bunker". A ```haveKey``` action is specified as shown below:

```
{
	actionType: "deleteKey",
	actionParams:
	{
		key:
		{
			keyType: "byRef",
			...
		},
		mustExist: BOOLEAN
	}
}
```

The meaning of each of the fields is as follows:

|**Field**|**Optional?**|**Meaning**|
|---|---|---|
|```key```|Mandatory|The key to delete. Each ```deleteKey``` action is for a single key and SHALL only refer to keys by reference.|
|```mustExist```|Optional|Indicates whether the key specified in the ```key``` field must exist. If set to ```true```, then the recipe will fail if the key does not exist, if set to ```false``` then the recipe will continue and a warning will be generated. The default behaviour, if the field is not specified, is ```true``` (strict evaluation).|

## generateKey action specification

The ```generateKey``` action tells the system in the "bunker" to generate a single key. A ```generateKey``` action is specified as shown below:

```
{
	actionType: "generateKey",
	actionParams:
	{
		key:
		{
			keyType: "byRef",
			...,
			keySize: INTEGER,
			exportable: BOOLEAN
		}
	}
}
```

The meaning of each of the fields is as follows:

|**Field**|**Optional?**|**Meaning**|
|---|---|---|
|```key```|Mandatory|The key to generate. Each ```generateKey``` action is for a single key and SHALL only refer to keys by reference.|
|```keySize```|Optional|The size of the key to generate. This parameters MUST be specified for algorithms 5, 7, 8 or 10 (RSA algorithms) and MUST NOT be specified for any other algorithm.|
|```exportable```|Optional|Specifies whether the key should be created with ```CKA_EXPORTABLE``` set to ```CK_TRUE``` (meaning that the key can be exported from the HSM). The default value, if not specified, is ```false``` (the key cannot be exported).|

## produceSignedKeyset action specification

The ```produceSignedKeyset``` action tells the system in the "bunker" to generate a signed ```DNSKEY``` RRset containing the specified keys and signed using the specified keys. A ```produceSignedKeyset``` action is specified as shown below:

```
{
	actionType: "produceSignedKeyset",
	actionParams:
	{
		ownerName: STRING,
		signerName: STRING,
		inception: INTEGER,
		expiration: INTEGER,
		ttl: INTEGER,
		keyset:
		[
			key:
			{
				...
			},
			...
		]
		signedBy:
		[
			key:
			{
				keyType: "byRef",
				...
			},
			...
		]
	}
}
```

## Example recipes

### Example 1: producing a signed DNSKEY set for an external ZSK

### Example 2: producing three signed DNSKEY sets for a generated ZSK

### Example 3: mixing internal and external ZSKs

### Example 4: a KSK rollover

## References

[1] Bradner, S., "RFC 2119 - Key words for use in RFCs to Indicate Requirements Levels", March 1997, [https://tools.ietf.org/html/rfc2119](https://tools.ietf.org/html/rfc2119)  

[2] Jones, M., Bradley, J. and Sakimura, N., "RFC 7515 - JSON Web Signatures", May 2015, [https://tools.ietf.org/html/rfc7515](https://tools.ietf.org/html/rfc7515)  