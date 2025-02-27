---
author: msmbaldwin
ms.service: key-vault
ms.topic: include
ms.date: 03/21/2022
ms.author: mbaldwin
---

This article documents a simplified EBNF grammar for secure key release policy, which itself is modeled on [Azure Policy](/azure/governance/policy/).

```json
(* string and number from JSON *)
value =
  string |
  number |
  "true" |
  "false";

(* The operators supported for claim value comparison *)
operator =
  "equals:";

(* A JSON condition that evaluates the value of a claim *)
claim_condition =
  "{" "claim:", string "," operator, ":", value "}";

(* A JSON condition requiring any of the listed conditions to be true *)
anyof_condition =
  "{" "anyof:", condition_array "}";

(* A JSON condition requiring all of the listed conditions to be true *)
allof_condition =
  "{" "allof:", condition_array "}";

(* A condition is any of the allowed condition types *)
condition =
  claim_condition |
  anyof_condition |
  allof_condition;

(* A list of conditions, one is required *)
condition_list =
  condition { "," condition };

(* An JSON array of conditions *)
condition_array =
  "[" condition_list "]";

(* A JSON authority with its conditions *)
authority =
  "{" "authority:", string "," ( anyof_condition | allof_condition );

(* A list of authorities, one is required *)
authority_list =
  authority { "," authority_list };

(* A policy is an anyOf selector of authorities *)
policy = 
  "{" "version: \"1.0.0\"", "anyOf:", "[" authority_list "]" "}";
```

## Claim condition

A Claim Condition is a JSON object that identifies a claim name, a condition for matching, and a value, for example:

```json
{ 
  "claim": "<claim name>", 
  "equals": <value to match>
} 
```

In the first iteration, the only allowed condition is "equals" but future iterations may allow for other operators [similar to Azure Policy](/azure/governance/policy/concepts/definition-structure) (see the section on Conditions). If a specified claim isn't present, its condition is considered to haven't been met.

Claim names allow "dot notation" to enable JSON object navigation, for example:

```json
{ 
  "claim": "object.object.claim", 
  "equals": <value to match>
}
```

Array specifications aren't presently supported. Per the grammar, objects aren't allowed as values for matching.

## AnyOf, AllOf conditions

AnOf and AllOf condition objects allow for the modeling of OR and AND. For AnyOf, if any of the conditions provided are true, the condition is met. For AllOf, all of the conditions must be true. 

Examples are shown below. In the first, allOf requires all conditions to be met:

```json
{
  "allOf":
  [
    { 
      "claim": "<claim_1>", 
      "equals": <value_1>
    },
    { 
      "claim": "<claim_2>", 
      "equals": <value_2>
    }
  ]
}
```

Meaning (claim_1 == value_1) && (claim_2 == value_2).

In this example, anyOf requires that any condition match:

```json
{
  "anyOf":
  [
    { 
      "claim": "<claim_1>", 
      "equals": <value_1>
    },
    { 
      "claim": "<claim_2>", 
      "equals": <value_2>
    }
  ]
}
```

Meaning (claim_1 == value_2) || (claim_2 == value_2)

The anyOf and allOf condition objects may be nested:

```json
  "allOf":
  [
    { 
      "claim": "<claim_1>", 
      "equals": <value_1>
    },
    {
      "anyOf":
      [
        { 
          "claim": "<claim_2>", 
          "equals": <value_2>
        },
        { 
          "claim": "<claim_3>", 
          "equals": <value_3>
        }
      ]
    }
  ]
```

Or:

```json
{
  "allOf":
  [
    { 
      "claim": "<claim_1>", 
      "equals": <value_1>
    },
    {
      "anyOf":
      [
        { 
          "claim": "<claim_2>", 
          "equals": <value_2>
        },
        {
          "allOf":
          [
            { 
              "claim": "<claim_3>", 
              "equals": <value_3>
            },
            { 
              "claim": "<claim_4>", 
              "equals": <value_4>
            }
          ]
        }
      ]
    }
  ]
}
```

## Key release authority

Conditions are collected into Authority statements and combined:

```json
{
  "authority": "<issuer>",
  "allOf":
  [
    { 
      "claim": "<claim_1>", 
      "equals": <value_1>
    }
  ]
}
```

Where:

- **authority**: An identifier for the authority making the claims. This identifier functions in the same fashion as the iss claim in a JSON Web Token. It indirectly references a key that signs the Environment Assertion.
- **allOf**: One or more claim conditions that identify claims and values that must be satisfied in the environment assertion for the release policy to succeed. anyOf is also allowed. However, both are not allowed together. 

## Key Release Policy

Release policy is an anyOf condition containing an array of key authorities:

```json
{
  "anyOf":
  [
    {
      "authority": "my.attestation.com",
      "allOf":
      [
        { 
          "claim": "mr-signer", 
          "equals": "0123456789"
        }
      ]
    }
  ]
}
```

## Encoding key release policy

Since key release policy is a JSON document, it is encoded when carried in requests and response to AKV to avoid the need to describe the complete language in Swagger definitions. 

The encoding is as follows:

```json
{
  "contentType": "application/json; charset=utf-8",
  "data": "<BASE64URL(JSON serialization of policy)>"
}
```

## Environment Assertion

An Environment Assertion is a signed assertion, in JSON Web Token form, from a trusted authority that contains at least a key encryption key and one or more claims about the target environment (for example, TEE type, publisher, version) that are matched against the Key Release Policy. The KEK is a public RSA key owned by the target execution environment (and protected by it) that is used for key export, it must appear in one of, in preference order:

- The TEE keys claim (x-ms-runtime-claims/keys). This claim is a JSON object representing a JSON Web Key Set.
- The Enclave Held Data claim (maa-ehd) claim. The maa-ehd claim is expected to contain a string that is the Base64 URL encoding of an array of octets that contain a JSON document; within this document, AKV requires that there be a keys element containing a JSON Web Key Set.

Within the JWKS, one of the keys must meet the requirements for use as an encryption key (key_use is "enc", or key_ops contains "encrypt"). The first suitable key is chosen.