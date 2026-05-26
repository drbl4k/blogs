+++
title = "Analysis on Partial Authentication"
date = "2026-05-26T01:35:14+05:30"
#dateFormat = "2006-01-02" # This value can be configured for per-post date formatting
author = "drbl4k"
authorTwitter = "" #do not include @
cover = ""
tags = ["Burpsuite", "Auth bypass", "2FA Bypass", "MFA Bypass"]
keywords = ["Burpsuite", ""]
description = "Detialed analysis on how 'Partial Authentication' leads to 2FA/MFA bypasses."
showFullContent = false
readingTime = true
hideComments = false
+++

In a web application penetration testing assessment, it is mandatory to understand the flow of the authentication, from entering credentials to logging in successfully, monitor them using a proxy tool such as Burpsuite to validate the auth mechanism. Lot of applications use 2FA (In the form of randomised security code or a well known security question) in order to increase the security of the authentication process. However apart from what I am covering in this blog, there are various other approaches to attain Auth / 2FA bypass such as 2FA security code brute force, API version downgrade *( i.e. v2 to v1 which doesn’t have rate-limiting )*, code leakage in HTTP response headers, use of stati OTP etc. In this blog I am going to cover how “Partial Authentication” allows 2FA/MFA bypass.

---

### What is it

Partial authentication is a state where a user has completed only the first (credentials) or some stages of identity verification but has not fully completed the authentication process. It commonly occurs in 2FA/MFA workflows. If an application treats a partially authenticated user as fully authenticated, it can create vulnerabilities. This is because, there is no session binding between the initial phrase (credentials) and the 2FA/MFA page which makes the verification process stateless.

---

### Proof Of Concept

The Issue arises when the application assigns a session token before completing the entire authentication process. Let’s say that you have your browser’s developer console open and trying to load the application which has 2FA enabled by default for all the users. Once you enter the right credentials, the application redirects you to the 2FA page. Staying in the 2FA page, you notice that either in the Cookies section or in the Session/Local Storage you have a new session identifier/bearer token. you can validate it by copying token’s value and appending it to any of the previously captured authenticated endpoint from the Burpsuite’s repeater. Once you send the request from the repeater tab, if the application responds back with 401, 403 or a redirect request to the login page, then the vulnerability does not exist. On the flip side if the application responds back with a successful response, it confirms that the application partially authenticates the user with full privlages and becomes vulnerable to 2FA/MFA bypass attacks.

---

### Exploitation

Once you have confirmed the vulnerability, we can go ahead and prove the impact. From an attacker’s perspective, let’s say that the attacker holds a personal account and the victim credentials, the attacker tries to log in as the victim, credentials are satisfied but the attacker does not know the security code. in that case few things could be done.

#### 1. Forcefully browse to an authenticated Endpoint

Log into the application using the credentials and reach the 2FA page. Once done open the Developer console and observe that the application as assigned a partial token.

![image.png](/images/partial-auth/image.png)

Staying on the OTP page, open a new tab and append the authenticated URL.

![image.png](/images/partial-auth/image1.png)

In the POC below the dashboard has loaded successfully. This proves that the token with full user rights has been issued before completing the entire auth flow.

![image.png](/images/partial-auth/image2.png)

#### 2. Response manipulation (200 OK)

Intercept the response of a false OTP request and tamper the received response by changing the status code to 200 OK and the status to true.

![image.png](/images/partial-auth/image3.png)

Once done forward the response back to the application and observe the dashboard being loaded.

![image.png](/images/partial-auth/image4.png)

This proves that the 2FA is working in a stateless way and the verification logic is present in the client side.

![image.png](/images/partial-auth/image5.png)

#### 3. Generate a GET request to an authenticated endpoint found in the client side code. (In case of black-box application)

Initiate the login process and reach the 2FA page. Copy the token issued by the application.

![image.png](/images/partial-auth/image6.png)

Open all the client side JavaScript files and find any authenticated endpoint URL. Copy the authenticated endpoint and append the path as well as the session token copied earlier to any GET request. 

![image.png](/images/partial-auth/image7.png)

Send the request and check for authenticated information in the response. This method can be used to prove impact on a black-box assessment.

![image.png](/images/partial-auth/image8.png)

#### 4. Using a valid (attacker’s account) code on the victim’s account (grey-box approach)

I have used this method in real world engagements and recorded a successful bypass. All you need is a valid account in the application to generate a security code. Once done copy the code and in a different browser reach the 2FA page as the victim and paste your code. The application should successfully log you in. This occurs because there is no session bind between the account that generated the request and the security code.

---

### Mitigation

- issue a temporary session token for the 2FA (or every stage of the MFA) phrase with no privileges assigned to it. *(i.e. the token should be valid only for the 2FA/MFA phrase)*
- the full session token should only be issued after all factors are verified
- OTP must be tied to the originating session which prevents valid OTP working on other users accounts.
- Every protected route should validate full auth process completion.

---

### References

- https://www.intigriti.com/researchers/blog/hacking-tools/broken-authentication-a-complete-guide-to-exploiting-advanced-authentication-vulnerabilities

- https://www.synack.com/exploits-explained/multi-factor-authentication-bypass-examples-via-response-tampering/

- https://www.webasha.com/blog/aditya-birla-capital-digital-gold-hack-195-crore-stolen-in-api-breach-services-restored#sec2

---

Happy Hunting !
