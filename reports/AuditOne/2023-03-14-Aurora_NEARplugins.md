**Auditors**

[AuditOne](https://twitter.com/auditone_team)

# Findings

## High Risk

### Full access key is not removed on ownership change

**Description:** The full access key is not removed on changing owner. This allow old owner to still have ownership using the access key set by him.

- Owner Asets a new full access key using attach\_full\_access\_key function
- Owner is changed to Owner B
- Owner A still has control over the contract using the full access key (since the public key from step 1 is not removed, owner A still can use the access key to perform critical operations)

**Recommendations:**

- Tie access key with the owner who has set it. In case owner has changed that access key becomes useless

- Remove the access key on changing owner

  

###  Mismatched Variable Names

**Description:** 

if users want to add except() arguments on function decrease\_1in examples/pausable-examples/pausable\_base/src/lib.rs line 49-52. However, the rust compiler cannot compile it due to the error:

**Recommendations:**

In the function get\_bypass\_condition, we use variable \_\_check\_paused. But in function if\_paused we use variable check\_paused.They should be the same variable. Change the variable check\_paused to \_\_check\_paused in function if\_paused



## Medium Risk

### Timelock behind up\_deploy\_code

**Description:**User should get considerable time to opt out of an upgraded code. Since up\_deploy\_code does not require any delay, a malicious owner can simply deploy malicious code instantly.

- An innocent contract Ais deployed with high staking APR
- User starts investing using the contract
- Owner stages and deploys a malicious code which adds a new function to drain user funds
- User funds get stolen

**Recommendations:** Place up\_deploy\_code function behind timelock which gives user suitable time to make decision



### Lack of State Migration and State Integrality Check

**Description:**

In the current implementation, there is no state migration interface. In this case, the contract cannot deserialize the account's state due to the newly added attribute counter2.

**Recommendations:**

To fix the problem, users need to implement a method that migrates the old state, add the counter2 to the Counter. In order to check if the migration is correct, it's highly recommended to invoke a view function to get the whole contract state after the migration is completed. Add a migration interface for user

###  Improper Full Access Key Fallback Design

**Description:** 

A Full Access Key could have total control over the account. It can transfer NEAR, deploy a smart contract and so on. Full access keys have more privileges than the owner and can do more things than the owner. In the current design, the owner of the contract can attach a new full-access key and can gain more privileges than expected.



### Pausable contract management should be handled by AccessControl plugin instead of the owner.

**Description:** 

In the current implementation, only the owner of the Pausable contract has the ability to pause or unpause it. This creates a potential security vulnerability, as the contract cannot be paused or unpaused if the owner's account is compromised or otherwise

unavailable. It also lacks accountability and transparency, as it may be dicult for other stakeholders to understand why certain actions were taken and whether they were justified. 

**Recommendations:**

- Update the Pausable contract to be managed by the AccessControl plugin instead of the owner.
- Implement additional security measures to protect the AccessControl plugin and ensure its availability.
- Ensure that all actions taken with the Pausable contract are properly documented and transparent to all stakeholders.



## Low Risk

### No way to remove access key

**Description:** 

Plugin has no way to remove an added access key

**Recommendations:** 

Add a new plugin method (owner only) which allows to remove an access key for given account



### Unchecked Removal of the Paused\_Key

**Description:** 

The function pa\_unpause\_feature() lacs a check on key's removal. The existence of the key is not checked. In this case, if the key does not exist, the contract will not panic, which may mislead the project manager and bring unexpected impacts.

**Recommendations:** 

Check the return value and panic if it is false.

**Status:** Resolved 



### Lack of Macro Attributes for Pausable Derivation

**Description:** 

For example, if users want to change the storage key of Pausable in examples/pausable-examples/pausable\_base/src/lib.rs line 7-12, he will add #[pausable(paused\_storage\_key="new\_storage\_key") However, the rust compiler cannot compile it due to the error.

**Recommendations:** 

Add attributes(pausable) to derive\_pausable.



## Informational

### Staging code is not removed

**Description:** 

upstage code is not removed on deployment

**Recommendations:** 

Remove upstage code on deployment



### Unnecessary Clone

**Description:** 

The functions return Vec<u8> so we need to copy the slice on the memory to construct a new Vec.

while 'static [u8]is a reference, there is no copy needed.

Thus, we can return 'static [u8]directly.

**Recommendations:** 

Change return type from Vec<u8> to 'static [u8]and remove to\_vec(). 



### Question about the design purpose of the Full Access Key Fallback

**Description:** 

The comment says Smart contracts can be considered trustless, when there is no Full Access Key (FAK) attached to it. Otherwise owner of the FAKcan redeploy or use the funds stored on the smart contract.

However, according to the doc[umentation, remov](https://docs.near.org/concepts/basics/accounts/access-keys#locked-accounts)ing the contracts' FAKis suggested to make the contract fully de-centralized.



### Missing documentation for admins being able to remove other admins in ACL **Severity:** Quality Assurance

**Description:** In current design, the admin of can revoke other admin in the same group. 



### Replace default panic with near-sdk panic

**Description:**

Currently, the codebase uses the default panic function to handle exceptional cases. This can lead to unexpected behavior and make it dicult to debug issues

**Recommendations:**

To improve the reliability and stability of the application, it is recommended to replace the default panic function with the panic function provided by the near-sdk. This panic function allows for more control over the handling of exceptional cases and can make it easier to debug and recover from issues.

Implementing this change will require updating all instances of the default panic function with the near\_sdk::panic function. It may also require updating any code that handles recovery from exceptional cases.



### revoke\_super\_admin\_unchecked not exposed through near bindgen

**Description:** 

It has been reported that the revoke\_super\_admin\_unchecked function is not exposed through the near bindgen, preventing it from being accessed from external interfaces. 

**Recommendations:**

To ensure that the revoke\_super\_admin\_unchecked function can be used as intended, it is recommended to expose it through the near bindgen. This will allow the function to be accessed from external interfaces and used to revoke super administrative privileges as needed. It may also be helpful to implement safeguards to prevent accidental or malicious revocation of super administrative privileges.



### Open to-dos

**Description:** 

Open To-dos can point to architecture or programming issues that still need to be resolved. Often these kinds of comments indicate areas of complexity or confusion for developers. This provides value and insight to an attacker who aims to cause damage to the protocol.

**Recommendations:** 

Consider resolving the To-Dos before deploying code to a production context. Use an independent issue tracker or other project management software to track development tasks.



### Missing documentation for the case of using Upgradable to patch a vulnerability

**Description:** 

Before deployment all code is kept in up\_stage\_code. If deployment is made for security issue then up\_stage\_code will contain fix for vulnerable code which could be seen by users to exploit current codebase.

- User reports a critical vulnerability in contract code
- Owner quickly make patch and calls up\_stage\_code to write the updated code
- Before owner could deploy, attacker checks this upstage code, figures out the vulnerability and exploits it

**Recommendations:** 

For critical upgrades, encrypted code could be upstaged which could be decrypted at deployment time using a provided key.
