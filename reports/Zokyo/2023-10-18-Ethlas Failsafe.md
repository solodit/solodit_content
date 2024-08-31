**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## Low Risk

### Exposure of Sensitive Information 

**Severity**: Low 

**Status** : Resolved 

**Description**: 

In Interceptor the Logger.js having access key ID and secret access key are accessed directly from the configuration, potentially exposing sensitive AWS credentials. Mnemonics, certificates, and secrets are hardcoded in the interceptor/service/conf folder.
Impact: Potential breach or loss of funds if these keys are compromised.

**Recommendation**: 


Store sensitive information using secure environment variables or use AWS IAM Roles and Instance Profiles if running on AWS services like EC2 or Lambda. Use secure vaults or AWS Secrets Manager to store and retrieve sensitive information 
 
### Clickjacking Vulnerability in failsafe.eleoslabs.io

**Severity** : Low 

**Status**: Resolved 

**Description** :

 Clickjacking, also known as UI Redress Attack, is a malicious technique where an attacker tricks a user into clicking on something different from what the user perceives. In this case, a malicious actor can use an invisible or disguised iframe embedding https://failsafe.eleoslabs.io/ to mislead the users of the website into performing unintended actions without their knowledge.
an iframe is used to embed the failsafe.eleoslabs.io within another webpage, which can potentially allow an attacker to overlay other content or make the iframe transparent to deceive the user into interacting with the embedded page.
Impact: If successfully exploited, this vulnerability could lead to a range of impacts, including, but not limited to:
Unauthorized actions being performed on behalf of the user.
Loss of sensitive user data.
Damage to the reputation of the impacted site.

**Recommendation**: 

To mitigate this vulnerability, consider implementing the following measures:
Content Security Policy (CSP): Use the frame-ancestors directive to specify which websites are allowed to embed your site within iframes.


Content-Security-Policy: frame-ancestors 'none';
 This will prevent the page from being embedded within iframes, mitigating the Clickjacking risk.


X-Frame-Options Header: Employ the X-Frame-Options HTTP header on your website. This header can have values like DENY or SAMEORIGIN, which will block the page from being embedded in an iframe from other domains.




### The application supports the concurrent sessions  

**Severity** : Low

**Status** : Resolved  


**Description**: 

Failsafe  application currently permits authentication of a single account multiple times, leading to the possibility of multiple sessions. This also means users cannot invalidate potential attackers’ sessions when user’s credentials leaks. Moreover, the application does not display a list of active sessions, leaving users unaware if unauthorized access occurs using their credentials. 

**Recommendation**:

 The application should deactivate previously authenticated sessions when new ones are established.


### Lack of Secure HTTP Headers

**Severity** : Low

**Status** : Resolved 



**Description**: 

Content Security Policy. Content Security Policy (CSP) was developed to allow an application developer to define the type and location of resources allowed to be loaded within the context of a given website.
Strict-Transport-Security. HTTP headers are well known and its implementation can make your application more secure. HTTP Strict Transport Security (HSTS) is a web security policy mechanism which helps to protect websites against protocol downgrade attacks and cookie hijacking. It allows web servers to declare that web browsers (or other complying user agents) should only interact with it using secure HTTPS connections, and never via the insecure HTTP protocol. HSTS is an IETF standards track protocol and is specified in RFC 6797. A server implements an HSTS policy by supplying a header (Strict-Transport-Security) over an HTTPS connection (HSTS headers over HTTP are ignored).
Problem Details
The web application does not use a set of industry-standard HTTP headers to  enhance security. The attacker can exploit the absence of these headers to  introduce vulnerabilities like Cross-Site Scripting and clickjacking attacks.
Affected Areas:
 https://failsafe.eleoslabs.io/ 


**Proof of Concept**

The following is needed in order to reproduce this issue:

Step 1 - Run the following command from the curl configured terminal and observe that the secure http headers are missing.
curl -I -X GET https://failsafe.eleoslabs.io/  --insecure
Recommendation
Implement the recommended HTTP security response headers based on need and application requirements. Set attributes and values as securely as possible based on application requirements.
Content-Security-Policy: Content Security Policy is an effective measure to protect your site from XSS attacks. By whitelisting sources of approved content, you can prevent the browser from loading malicious assets.
Referrer-Policy: Referrer Policy is a new header that allows a site to control how much information the browser includes with navigations away from a document and should be set by all sites.

## Informational

### Insecure Transmission and Handling of Sensitive Information in Interceptor 

**Severity** : Informational

**Status** : Resolved 

**Description** :  

The httpUtils.js from Interceptor is vulnerable due to the insecure transmission of sensitive information, lack of input validation, and a disregard for secure practices in SSL/TLS certificate validation. These can potentially lead to exposure of sensitive data and Man-In-The-Middle (MITM) attacks.
The application is set to transmit sensitive information such as wallet addresses, encrypted private keys, and API keys without proper SSL/TLS certificate validation (rejectUnauthorized: false). This can expose the communication to interception and manipulation by malicious actors.
There is no validation or sanitation of the inputs provided to the functions. This may leave the application vulnerable to a variety of injection attacks if user-controlled data is passed to these functions.
There is no evident usage of secure headers or proper Content-Type specifications for the requests, which might lead to potential security misconfigurations.

**Recommendation**: 

Enable Strict SSL/TLS Certificate Validation: Do not disable SSL/TLS certificate validation. Always validate certificates properly to ensure the security of the data in transit.
Input Validation and Sanitization: Always validate and sanitize inputs to the functions. Use a proper validation framework or library to ensure that only expected and valid inputs are processed.
#Note : This is used by the interceptor to submit a request to the signing server which is the same Trust boundary as the interceptor (inside an externally restricted VPC).  So above attacks do not apply (and we are doing this for perf reasons since for interception shaving off milliseconds matter).


### Unsecured WebSocket Reconnection Logic

**Severity** : Informational 

**Status** : Acknowledged 

**Description**:

In Interceptor the Writer.js the logic to reconnect the WebSocket can potentially be exploited by forcing socket closure, leading to a Denial-of-Service (DoS) scenario.
An attacker may exploit this by repeatedly causing the WebSocket to close or error (could be through a network attack or by exploiting another vulnerability that can crash the socket), knowing that the system will keep trying to reconnect without backoff or limit. This could lead to resource exhaustion or Denial-of-Service (DoS) as the system is constantly trying to reconnect.

**Recommendation**:

Implement a secure reconnection strategy, including a back-off strategy and limiting reconnection attempts to mitigate the risk of DoS attacks


 
### Outdated or Untrusted Dependencies

**Severity**: Informational

**Status**: Resolved  

**Description**:

 The Interceptor Implementation  uses several third-party packages. Outdated or untrusted dependencies can introduce security vulnerabilities to the application.
https://snyk.io/advisor/check/npm/7d1112e5-9221-4e69-ad23-dcedd8de1850/needReview
Affected area

**Recommendation**: 

Regularly update all dependencies to their latest versions and ensure they come from reputable sources. Perform periodic security audits on these packages to reduce the risk of potential vulnerabilities.

### Unrestricted Asset Interaction and Lack of Allowlist

**Severity** : Informational 

**Status**: Acknowledged  

**Description**:

The service does not impose restrictions or perform validations/sanitizations on the assets it interacts with. Consequently, it creates an exploitable security vulnerability, potentially allowing attackers to exploit the service by using malicious assets, such as a malicious ERC20 token with an overwritten transfer() method. For example, an attacker could mint and drop such a token into a protected account. Since the service does not have an allowlist or validation mechanism, it would perceive such an asset as a legitimate one to protect and potentially interact with it, causing unintended and malicious transactions, leading to account damage.

The impact of this vulnerability is critical as it can lead to unauthorized asset interaction, leading to unintended financial losses, and could potentially compromise the integrity and reputation of the service. It could allow attackers to execute unintended actions like performing maximum approval or even directly drawing funds from a protected account.
Attack Vector:
The attack vector is primarily phishing tokens—tokens specifically crafted with malicious intent, which can perform unauthorized actions when interacted with.
Proof of Concept:
Attacker creates a malicious ERC20 token with a modified transfer() method, which, when called, could trigger unauthorized actions.
Attacker mints and drops this token into a protected account.
The interceptor service, without any allowlist or asset validation, perceives this asset as legitimate and interacts with it.
The malicious transfer() method is triggered, causing unauthorized transactions or actions.

**Recommendation**:

Implement Asset Allowlist: Deploy a strictly managed allowlist of assets that the service is permitted to interact with, preventing interaction with any unauthorized or unverified assets.
Enhance Asset Validation: Develop robust asset validation mechanisms to verify and sanitize the assets before any interaction occurs, reducing the risk of interacting with malicious assets.

