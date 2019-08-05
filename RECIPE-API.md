# DNSSEC Key Signing Suite Documentation
## Key Ceremony Recipe API specification

Copyright (c) 2019 NLnet Labs  
Released under Creative Commons CC 4.0 BY-SA (see [`LICENSE`](LICENSE))

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
 * the cooked output (see below);
 * an optional digital signature.

If a recipe is digitally signed, then it MUST be signed using the JSON Web Signature format specified in RFC 7515 [[2]](https://tools.ietf.org/html/rfc7515).

The outer structure of a recipe MUST contain the following fields:

 * `recipeSpecVersion` - the version of this specification, currently this is limited to `v1.0`.
 * `recipe` - the actual recipe; this field MUST contain only a single recipe and MAY NOT contain an array.

An example of the outer structure of a recipe then looks like this:

```
{
	recipeSpecVersion: "v1.0",
	recipe: { ... }
}
```

## Recipe preamble

The preamble specifies common parameters that apply to the whole recipe. The following common parameters exist:

 * `timestamp` (mandatory) - UNIX epoch timestamp for when the recipe was created.
 * `cooked_timestamp` (optional) - UNIX epoch timestamp for when the recipe was executed (will be included by the software running in the "bunker").
 * `description` (optional) - A free-form human readable description of the recipe that can be displayed to operators and witnesses at the ceremony.

An example of a recipe with a preamble section is specified below:

```
{
	"recipeSpecVersion": "v1.0",
	"recipe":
	{
		"preamble":
		{
			"timestamp": 1562582413,
			"cooked_timestamp": 0,
			"description": "An example key ceremony recipe"
		}, ...
	}
}
```

## Recipe actions

A recipe specifies the actions that need to be performed inside the "bunker". Recipes can contain the following types of actions:

 * `haveKey` - to verify the existence of a specified key in the "bunker".
 * `deleteKey` - to irrevocably delete a key from the storage in the "bunker".
 * `generateKey` - to generate a new key in the "bunker" according to the included specifications.
 * `produceSignedKeyset` - to produce a signed `DNSKEY` RRset that includes certain specified keys and is signed using specified keys.
 * `importPublicKey` - to import a public key to be used for exporting keys from the secure storage in the "bunker".
 * `exportKeypair` - to export a key pair generated in the secure storage in the "bunker" as an encrypted PKCS #8 object using a specified public key that was previously imported using the `importPublicKey` action.

Irrespective of the order in which the different action types are specified in the file, the different action types SHALL always be performed in the same order, which is all `haveKey` actions, followed by all `deleteKey` actions, followed by all `generateKey` actions, followed by all `producedSignedKeyset` actions, followed by all `importPublicKey` actions and ending with all `exportKeyPair` actions. Always using this order ensures that the preconditions for each action type are typically automatically met, and it also allows operators to perform a backup of the key storage medium immediately after each ceremony ends, and to be sure that the state that is backed up can always be re-used to produce the same signed keysets. To facilitate human readability of recipes, action types SHOULD always be specified in the order in which they are executed.

If multiple actions of the same type are specified, then these MAY be executed in any order, and no particular assumptions must be made as to the order in which these actions are executed.

A generic syntax is used to specify each action. The `actionType` field specifies the action type, and the `actionParams` field specifies the parameters for that specific action. The `cooked` field will contain the results of the action and will be added by the software executed in the "bunker". The `cooked` field MAY be omitted from the input recipe.

An example of a recipe with the most common action types is given below:

```
{
	"recipeSpecVersion": "v1.0",
	"recipe":
	{
		"preamble": { ... },
		"actions":
		[
			{ "actionType": "haveKey", "actionParams": { ... }, "cooked": { ... } },
			{ "actionType": "haveKey", "actionParams": { ... }, "cooked": { ... } },
			{ "actionType": "deleteKey", "actionParams": { ... }, "cooked": { ... } },
			{ "actionType": "generateKey", "actionParams": { ... }, "cooked": { ... } },
			{ "actionType": "produceSignedKeyset", "actionParams": { ... }, "cooked": { ... } },
			{ "actionType": "produceSignedKeyset", "actionParams": { ... }, "cooked": { ... } },
		]	
	}
}
```

The parameters for each of the action types are specified further down.

## Specifying keys

The recipe specification is designed to be flexible in how keys are specified, in particular keys that are part of a keyset that needs to be signed in a `produceSignedKeyset` action. Two models are supported, a model in which all keys, including ZSKs, are generated in the "bunker" , and a model in which some keys are generated outside of the "bunker", but need to be included in the signed keyset. This means that there are also two ways to specify keys in action parameter sets: by reference, or direct. The two ways to specify keys are shown in detail below:

**Key by reference:**

This way of referring to keys may be used in `haveKey`, `deleteKey`, `generateKey` and `produceSignedKeyset` actions.

Keys are specified as follows:

```
{
	"keyType": "byRef",
	"keyAlgo": INTEGER,
	"keyFlags": INTEGER,
	"keyStore": STRING,
	"keyID": STRING,
	"keyLabel": STRING
}
```

The meaning of each of the fields is as follows:

|**Field**|**Optional?**|**Meaning**|
|-----|---------|-------|
|`keyType`|Mandatory|Key type, MUST be set to `"byRef"` for keys by reference.|
|`keyAlgo`|Mandatory|Key algorithm, MUST specify a valid DNSSEC key algorithm (e.g. 13 for ECDSAP256SHA256).|
|`keyFlags`|Mandatory|DNSSEC key flags (e.g. 256 for ZSK, 257 for KSK).|
|`keyStore`|Mandatory|Opaque reference to where the key is stored. This could, for example, be the name of the HSM partition label.|
|`keyID`|Optional*|Base64-encoded byte string for a PKCS #11 `CKA_ID` that should be used to create or find the key.|
|`keyLabel`|Optional*|Label string for a PKCS #11 `CKA_LABEL` that should be used to create or find the key.|

*When specifying keys by reference at least one of `keyID` or `keyLabel` MUST be specified.

**Direct key specification:**

This way of referring to keys may only be used in `produceSignedKeyset` and `importPublicKey` actions.

Keys are specified as follows:

```
{
	"keyType": "direct",
	"keyAlgo": INTEGER,
	"keyFlags": INTEGER,
	"keyData": STRING
}
```

The meaning of each of the fields is as follows:

|**Field**|**Optional?**|**Meaning**|
|-----|---------|-------|
|`keyType`|Mandatory|Key type, MUST be set to `"byRef"` for keys by reference.|
|`keyAlgo`|Mandatory|Key algorithm, MUST specify a valid DNSSEC key algorithm (e.g. 13 for ECDSAP256SHA256).|
|`keyFlags`|Mandatory|DNSSEC key flags (e.g. 256 for ZSK, 257 for KSK).|
|`keyData`|Mandatory|Base64-encoded data from the Public Key RDATA field of the `DNSKEY` resource record for this key.|

## haveKey action specification

The `haveKey` action can be used as a pre-condtion in a recipe to verify the existence of certain keys in the secure storage of the "bunker". A `haveKey` action is specified as follows:

```
{
	"actionType": "haveKey",
	"actionParams":
	{
		"key":
		{
			"keyType": "byRef",
			...
		},
		"relaxed": BOOLEAN
	},
	"cooked":
	{
		"exists": BOOLEAN
	}
}
```

The meaning of each of the input fields is as follows:

|**Field**|**Optional?**|**Meaning**|
|---|---|---|
|`key`|Mandatory|The key to check for. Each `haveKey` action works for a single key and SHALL only refer to keys by reference.|
|`relaxed`|Optional|Indicates whether the `haveKey` action should be evaluated in a relaxed manner. If set to `true`, then the action will not fail if the key is not present, if set to `false` then the whole recipe will fail if the key is not present. The default behaviour, if the field is not specified, is `false` (strict evaluation).|

The meaning of each of the output fields in the `cooked` section is as follows:

|**Field**|**Optional?**|**Meaning**|
|---|---|---|
|`exists`|Mandatory|Set to `true` if the specified key exists, `false` otherwise.|

## deleteKey action specification

The `deleteKey` action deletes a single key from the secure storage in the "bunker". A `haveKey` action is specified as shown below:

```
{
	"actionType": "deleteKey",
	"actionParams":
	{
		"key":
		{
			"keyType": "byRef",
			...
		},
		"mustExist": BOOLEAN
	},
	"cooked":
	{
		"deleteSuccess": BOOLEAN
	}
}
```

The meaning of each of the input fields is as follows:

|**Field**|**Optional?**|**Meaning**|
|---|---|---|
|`key`|Mandatory|The key to delete. Each `deleteKey` action is for a single key and SHALL only refer to keys by reference.|
|`mustExist`|Optional|Indicates whether the key specified in the `key` field must exist. If set to `true`, then the recipe will fail if the key does not exist, if set to `false` then the recipe will continue and a warning will be generated. The default behaviour, if the field is not specified, is `true` (strict evaluation).|

The meaning of each of the output fields in the `cooked` section is as follows:

|**Field**|**Optional?**|**Meaning**|
|---|---|---|
|`deleteSuccess`|Mandatory|Set to `true` if the specified key existed and was deleted successfully, `false` otherwise.|

## generateKey action specification

The `generateKey` action tells the system in the "bunker" to generate a single key. A `generateKey` action is specified as shown below:

```
{
	"actionType": "generateKey",
	"actionParams":
	{
		"key":
		{
			"keyType": "byRef",
			...,
			"keySize": INTEGER,
			"exportable": BOOLEAN
		}
	},
	"cooked":
	{
		"generateSuccess": BOOLEAN
	}
}
```

The meaning of each of the input fields is as follows:

|**Field**|**Optional?**|**Meaning**|
|---|---|---|
|`key`|Mandatory|The key to generate. Each `generateKey` action is for a single key and SHALL only refer to keys by reference.|
|`keySize`|Optional|The size of the key to generate. This parameters MUST be specified for algorithms 5, 7, 8 or 10 (RSA algorithms) and MUST NOT be specified for any other algorithm.|
|`exportable`|Optional|Specifies whether the key should be created with `CKA_EXPORTABLE` set to `CK_TRUE` (meaning that the key can be exported from the HSM). The default value, if not specified, is `false` (the key cannot be exported).|

The meaning of each of the output fields in the `cooked` section is as follows:

|**Field**|**Optional?**|**Meaning**|
|---|---|---|
|`generateSuccess`|Mandatory|Set to `true` if a key was successfully generated according to the specified parameters, `false` otherwise.|

## produceSignedKeyset action specification

The `produceSignedKeyset` action tells the system in the "bunker" to generate a signed `DNSKEY` RRset containing the specified keys and signed using the specified keys. A `produceSignedKeyset` action is specified as shown below:

```
{
	"actionType": "produceSignedKeyset",
	"actionParams":
	{
		"ownerName": STRING,
		"signerName": STRING,
		"inception": INTEGER,
		"expiration": INTEGER,
		"ttl": INTEGER,
		"keyset":
		[
			{
				"key":
				{
					...
				}
			},
			...
		],
		"signedBy":
		[
			{
				"key":
				{
					"keyType": "byRef",
					...
				}
			},
			...
		]
	},
	"cooked":
	{
		"signSuccess": BOOLEAN,
		"signedKeysetRRs": [ STRING, ... ]
	}
}
```

The meaning of each of the input fields is as follows:

|**Field**|**Optional?**|**Meaning**|
|---|---|---|
|`ownerName`|Mandatory|The value for the `ownerName` field in the `DNSKEY` and `RRSIG` resource records.|
|`signerName`|Optional|The value for the `signerName` field in the `RRSIG` resource record(s). If unspecified, the `ownerName` is used.|
|`inception`|Mandatory|The UNIX epoch timestamp to be used in the `inception` field in the `RRSIG` resource record(s).|
|`expiration`|Mandatory|The UNIX epoch timestamp to be used in the `expiration` field in the `RRSIG` resource record(s).|
|`ttl`|Mandatory|The Time-to-Live (TTL) to use in the `DNSKEY` and `RRSIG` resource records.|
|`keyset`|Mandatory|Specification of the keys to include in the signed `DNSKEY` record set; the key used to sign MUST be specified explicitly for it to be included in the `DNSKEY` RRset.|
|`signedBy`|Mandatory|Specifies the key(s) used to produce the `RRSIG` record(s) over the `DNSKEY` set. At least one key MUST be specified.|

The meaning of each of the output fields in the `cooked` section is as follows:

|**Field**|**Optional?**|**Meaning**|
|---|---|---|
|`signSuccess`|Mandatory|Set to `true` if a signed keyset was successfully produced according to the specified parameters, `false` otherwise.|
|`signedKeysetRRs`|Optional|Included if `signSuccess` is set to `true`; contains an array of strings with the individual `DNSKEY` and `RRSIG` resource records in text form that together comprise the signed keyset. Individual records SHALL not contain line breaks.|

## importPublicKey action specification

The `importPublicKey` action tells the software in the "bunker" to import a public key into the secure storage medium (HSM). This key may later be referred to for use in signed keysets and may be used as a wrapping key when exporting keys from secure storage.

```
{
	"actionType": "importPublicKey",
	"actionParams":
	{
		"key":
		{
			"keyType": "direct",
			"keyAlgo": INTEGER,
			"keyFlags": INTEGER,
			"keyData": STRING,
			"keyID": STRING,
			"keyLabel": STRING
		}
	},
	"cooked":
	{
		"importSuccess": BOOLEAN
	}
}
```

The meaning of each of the input fields is as follows:

|**Field**|**Optional?**|**Meaning**|
|---|---|---|
|key|Mandatory|Specifies the key to import; note: as specified further above, `keyData` must contain the Base64-encoded version of the RDATA field for a `DNSKEY` record containing this key. The software in the bunker MUST convert this into a suitable format for import into secure storage (HSM). Also note that `keyID` and or `keyLabel` MUST be specified so this key can be referred to in later commands (for the semantics of these fields, see above).|

The meaning of each of the output fields in the `cooked` section is as follows:

|**Field**|**Optional?**|**Meaning**|
|---|---|---|
|importSuccess|Mandatory|Set to `true` if the key was imported successfully, `false` otherwise.|

## exportKeypair action specification

The `exportKeypair` action tells the software in the "bunker" to export the specified key pair from secure storage (HSM) and to wrap it using the specified public key. In order for a key pair to be exportable, it must have been generated with the `exportable` flag set to `true` in the `generateKey` action that created the key. Alternatively, if the key was generated outside of the scope of the DNSSEC Key Ceremony Tool Suite, it must have the PKCS #11 `CKA_EXPORTABLE` flag set to `CK_TRUE`.

```
{
	"actionType": "exportKeypair",
	"actionParams":
	{
		"key":
		{
			"keyType": "byRef",
			...
		},
		"wrappingKey":
		{
			"keyType": "byRef",
			...
		}
	},
	"cooked":
	{
		"exportSuccess": BOOLEAN,
		
		"keyBlob": STRING
	}
}
```

The meaning of each of the input fields is as follows:

|**Field**|**Optional?**|**Meaning**|
|---|---|---|
|key|Mandatory|Specifies the key to export. Note: key must be *exportable*.|
|wrappingKey|Mandatory|Specifies the public key to use for wrapping. In most cases, this will be a key that has previously been imported using the `importPublicKey` action.|

The meaning of each of the output fields in the `cooked` section is as follows:

|**Field**|**Optional?**|**Meaning**|
|---|---|---|
|exportSuccess|Mandatory|Set to `true` if the key pair was exported successfully, `false` otherwise.|
|keyBlob|Optional|If the export was successful, this field contains a Base64-encoded PKCS #8 key blob that can be imported using the private key associated with the public key used for export.|

## Example recipes

### Example 1: producing a signed DNSKEY set for an external ZSK

The example recipe below verifies the presence of the KSK and then produces a signed DNSKEY set for an external ZSK. The inception date for the signature will be Aug 1, 2019, 00:00 UTC, and the expiration date Aug 1, 2020, 00:00 UTC. The TTL is one day. The KSK is included in the `DNSKEY` RRset.

```
{
	"recipeSpecVersion": "v1.0",
	"recipe":
	{
		"preamble":
		{
			"timestamp": 1564523557,
			"cooked_timestamp": 0,
			"description": "Example 1: producing a signed DNSKEY set for an external ZSK"
		},
		"actions":
		[
			{
				"actionType": "haveKey",
				"actionParams":
				{
					"key":
					{
						"keyType": "byRef",
						"keyAlgo": 8,
						"keyFlags": 257,
						"keyLabel": "example.com KSK"
					},
					"relaxed": false
				}
			},
			{
				"actionType": "produceSignedKeyset",
				"actionParams":
				{
					"ownerName": "example.com.",
					"signerName": "example.com.",
					"inception": 1564617600,
					"expiration": 1596240000,
					"ttl": 86400,
					"keyset":
					[
						{
							"key":
							{
								"keyType": "direct",
								"keyAlgo": 8,
								"keyFlags": 256,
								"keyData": "AwEAAZ8E9WYzWjBD++77FeN/rLCEEYeg2WX2T8X53yUGeCnvFXrVh0bMvPr6h706lktkDvjoT0uPrJzwgtPiF0NtkFfsKA3+pE3E1Ejnfg/bsrdq IrWuAGpanyFo2Z8UFMbvrhIkbGRkIIVRJO0g4L9JY/by5IcQHRp6Ns1LFTvC4Cu5"
							}
						},
						{
							"key":
							{
								"keyType": "byRef",
								"keyAlgo": 8,
								"keyFlags": 257,
								"keyLabel": "example.com KSK"
							}
						}
					],
					"signedBy":
					[
						{
							"key":
							{
								"keyType": "byRef",
								"keyAlgo": 8,
								"keyFlags": 257,
								"keyLabel": "example.com KSK"
							}
						}
					]
				}
			}
		]
	}
}
```

### Example 2: producing three signed DNSKEY sets for a generated ZSK

The example recipe below does four things. It checks if the KSK is present, generates a new ZSK and three signed keysets with different inception and expiration times and exports the newly generated ZSK.

```
{
	"recipeSpecVersion": "v1.0",
	"recipe":
	{
		"preamble":
		{
			"timestamp": 1564521229,
			"cooked_timestamp": 0,
			"description": "Example 2: producing three signed DNSKEY sets for a generated ZSK"
		},
		"actions":
		[
			{
				"actionType": "haveKey",
				"actionParams":
				{
					"key":
					{
						"keyType": "byRef",
						"keyAlgo": 8,
						"keyFlags": 257,
						"keyLabel": "example.com KSK"
					},
					"relaxed": false
				}
			},
			{
				"actionType": "generateKey",
				"actionParams":
				{
					"key":
					{
						"keyType": "byRef",
						"keyAlgo": 8,
						"keyFlags": 256,
						"keyLabel": "example.com Q3 ZSK",
						"keySize": 2048,
						"exportable": true
					}
				}
			},
			{
				"actionType": "produceSignedKeyset",
				"actionParams":
				{
					"ownerName": "example.com.",
					"signerName": "example.com.",
					"inception": 1569369600,
					"expiration": 1572912000,
					"ttl": 86400,
					"keyset":
					[
						{
							"key":
							{
								"keyType": "byRef",
								"keyAlgo": 8,
								"keyFlags": 256,
								"keyLabel": "example.com Q3 ZSK"
							}
						},
						{
							"key":
							{
								"keyType": "byRef",
								"keyAlgo": 8,
								"keyFlags": 257,
								"keyLabel": "example.com KSK"
							}
						}
					],
					"signedBy":
					[
						{
							"key":
							{
								"keyType": "byRef",
								"keyAlgo": 8,
								"keyFlags": 257,
								"keyLabel": "example.com KSK"
							}
						}
					]
				}
			},
			{
				"actionType": "produceSignedKeyset",
				"actionParams":
				{
					"ownerName": "example.com.",
					"signerName": "example.com.",
					"inception": 1572048000,
					"expiration": 1575504000,
					"ttl": 86400,
					"keyset":
					[
						{
							"key":
							{
								"keyType": "byRef",
								"keyAlgo": 8,
								"keyFlags": 256,
								"keyLabel": "example.com Q3 ZSK"
							}
						},
						{
							"key":
							{
								"keyType": "byRef",
								"keyAlgo": 8,
								"keyFlags": 257,
								"keyLabel": "example.com KSK"
							}
						}
					],
					"signedBy":
					[
						{
							"key":
							{
								"keyType": "byRef",
								"keyAlgo": 8,
								"keyFlags": 257,
								"keyLabel": "example.com KSK"
							}
						}
					]
				}
			},
			{
				"actionType": "produceSignedKeyset",
				"actionParams":
				{
					"ownerName": "example.com.",
					"signerName": "example.com.",
					"inception": 1574640000,
					"expiration": 1578182400,
					"ttl": 86400,
					"keyset":
					[
						{
							"key":
							{
								"keyType": "byRef",
								"keyAlgo": 8,
								"keyFlags": 256,
								"keyLabel": "example.com Q3 ZSK"
							}
						},
						{
							"key":
							{
								"keyType": "byRef",
								"keyAlgo": 8,
								"keyFlags": 257,
								"keyLabel": "example.com KSK"
							}
						}
					],
					"signedBy":
					[
						{
							"key":
							{
								"keyType": "byRef",
								"keyAlgo": 8,
								"keyFlags": 257,
								"keyLabel": "example.com KSK"
							}
						}
					]
				}
			},
			{
				"actionType": "exportKeypair",
				"actionParams":
				{
					"key":
					{
						"keyType": "byRef",
						"keyAlgo": 8,
						"keyFlags": 256,
						"keyLabel": "example.com Q3 ZSK"
					},
					"wrappingKey":
					{
						"keyType": "byRef",
						"keyAlgo": 8,
						"keyFlags": 256,
						"keyLabel": "ZSK exporting key"
					}
				}
			}
		]
	}
}
```

## References

[1] Bradner, S., "RFC 2119 - Key words for use in RFCs to Indicate Requirements Levels", March 1997, [https://tools.ietf.org/html/rfc2119](https://tools.ietf.org/html/rfc2119)  

[2] Jones, M., Bradley, J. and Sakimura, N., "RFC 7515 - JSON Web Signatures", May 2015, [https://tools.ietf.org/html/rfc7515](https://tools.ietf.org/html/rfc7515)  