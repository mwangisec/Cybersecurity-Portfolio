---
title: "vAPI: Full OWASP API Top 10 Security Assessment"
date: 2026-06-22
tags:
  - api-security
  - owasp
  - vapi
  - penetration-testing
  - jwt
  - ssrf
  - xss
  - sql-injection
categories:
  - Writeups
description: Complete security assessment of vAPI covering all OWASP API Top 10 (2019) vulnerabilities plus JWT manipulation, SSRF, and XSS Arena challenges. 13 confirmed findings from Critical to Low severity.
publish: "true"
toc: "true"
---
## Executive Summary
This document presents a technical security assessment of VAPI, a deliberately vulnerable API application designed to simulate the OWASP API Security Top 10 (2019). The assessment was conducted in a self-hosted lab environment with the objective of identifying, exploiting, and documenting each vulnerability class, as well as providing actionable remediation guidance for each.
Over the course of this assessment, all ten OWASP API vulnerability categories were successfully exploited, along with three additional Arena challenges covering JSON Web Token (JWT) manipulation, Server-Side Request Forgery (SSRF), and Cross-Site Scripting (XSS) injection. A total of thirteen distinct security findings were confirmed.
The findings range in severity from Critical to Low. The most severe vulnerabilities  Broken Object Level Authorization (API1), SQL Injection (API8), and Broken Authentication via credential stuffing (API2) represent attack vectors that, in a production environment, could result in full database compromise, complete account takeover, and mass unauthorized data disclosure. These three findings alone would constitute a critical risk posture for any organization operating an API with similar weaknesses.

| API Reference | Link                        | API vulnerability                   | Severity |
| ------------- | --------------------------- | ----------------------------------- | -------- |
| API1          | [API1](Vapi.md#API1)                   | Broken Object Level Authorization   | Critical |
| API2          | [API2](Vapi.md#API2)                   | Broken Authentication               | Critical |
| API3          | [API3](Vapi.md#API3)                   | Excessive Data Exposure             | High     |
| API4          | [API4](Vapi.md#API4)                   | Lack of Resources and Rate Limiting | High     |
| API5          | [API5](Vapi.md#API5)                   | Broken Function Level Authorization | High     |
| API6          | [API6](Vapi.md#API6)                   | Mass Assignment                     | High     |
| API7<br>      | [API7](Vapi.md#API7)                   | Security Misconfiguration           | High     |
| API8<br>      | [API18](Vapi.md#API18)                  | Injection                           | Critical |
| API9<br>      | [API9](Vapi.md#API9)                   | Improper Assets Management          | High     |
| API10<br>     | [API10](Vapi.md#API10)                  | Insufficient Logging and Monitoring | Medium   |
| Arena- JWT    | [Just weak tokens (JWT)](Vapi.md#Just%20weak%20tokens%20(JWT)) | Just- Weak Tokens                   | High     |
| Arena - SSRF  | [ServerSurfer](Vapi.md#ServerSurfer)           | Server-Side Forgery                 | High     |
| Arena - XSS   | [Sticky Notes](Vapi.md#Sticky%20Notes)           | Cross-Site Scripting                | High     |

The assessment demonstrates that API security cannot be treated as an afterthought. Each vulnerability in this report represents a category of weakness consistently found in real-world production APIs. The remediation guidance provided for each finding reflects current industry best practice as defined by OWASP, NIST and established secure development standards.
## Introduction and Installation
This write-up focuses on  VAPI. It is a vulnerable adversely Programmed Interface which is a self-hostable API mimicking OWASP API Top 10 the 2019 version. It can  be found in  https://github.com/roottusk/vapi and the installation process since it will be self hosted:
```
git clone https://github.com/roottusk/vapi.git
cd <your-hosting-directory>
docker-compose up -d 
```
![Setup.png](/images/Setup.png)When you open localhost:9000 on a browser you'll get to see the VAPI documentation, showing a successful installation.
![Vapi documentation.png](/images/Vapi%20documentation.png)

### Setup
#### Postman
- Since we will be working with APIs, we will use Postman since it is a good tool for building, testing, designing and managing APIs.
- Note that a collection and an environment has been copied to your device.
- Launch your postman and login/ sign up if you hadn't and you'll land on the postman dashboard.
- On the left there are 3 dots click on it and select the import button![importing.png](/images/importing.png)![import2.png](/images/import2.png) Follow the steps as numbered in the image to import the collection and environment that are stored in postman. Note vapi is the name of my directory where vapi is hosted, so just change the name to the name of your directory![Pasted image 20260522185213.png](/images/Pasted%20image%2020260522185213.png) Import the 2.
- ![import3.png](/images/import3.png) Note a new collection is added and when you click on the collection part vapi (2) you'll get to see a whole list of API and an environment. Go to the top right and set the environment to the new environment as shown in (4). The Vapi_env is important as it allows you to set variables on the whole collection as well as save authorization token allowing as to replay request
- ![Pasted image 20260522190843.png](/images/Pasted%20image%2020260522190843.png) Currently only the host is set, however, I'd like you to change it so that we specify the port where the application is running on so it be **localhost:9000**
#### Postman and Burp
- This allows to traffic our request from postman to burp so we can see how the request and response looks like as it's being sent and received.
- If you don't need to proxy the traffic through burp you can skip this stage and go directly to Burp.[API1](Vapi.md#API1)
- We will have to proxy it so go to the settings then app settings and follow the following:![postman settings.png](/images/postman%20settings.png) On the settings page go to the proxy (2) and select use custom proxy (3) by default its off and set the following under proxy server as shown on (4,5,6). Note after you've finished working with postman you can now come and disable it
- The next step is to place burp certificate so that postman can trust the request:![Burp certificate.png](/images/Burp%20certificate.png) First we will need to export burp certificate, so open burp, go to settings(1), then to proxy (2) and follow the rest steps(3,4,5) ensure to choose the der format and name the certificate and save to a directory of your choice
- Now we will go to the postman and import the file we just exported it:![Postman certificate.png](/images/Postman%20certificate.png)Go back to postman settings and go to the certificates (2) and enable the CA certificates and choose the certificate we just exported and select it.
-  Now we can confirm and see if the setup has succeded:![burp-postman.png](/images/burp-postman.png)Click on any request under the collections and just send the request just to see if the request goes and it is being channeled to burp and as we can see from above mine has succeeded.
We have now done all the configurations let's now start working on them.


### API1 
![API1.png](/images/API1.png)Broken Object Level Authorization (BOLA) occurs where an Object (a user) is able to view other unauthorized user information by manipulating object identifiers such as id , Uid, name.
1. We will go to the first request and create a user ![api1 POST.png](/images/api1%20POST.png)In Postman, Click the Post request and go to the body and fill in the details as shown as in (3) and send the request. Note that an ID parameter is generated together with the details we submitted with.
2. We can then see that the ID parameter is being used to get the information about the user specified in the second request as shown below:![API1 GET.png](/images/API1%20GET.png) Note the {{api_id}} is variable name and is set to the id parameter that was created, as can be seen on the burp side on the left side. The results we get is information about our user John.
3. Since we see that the ID is being used to identify a user we can then try to manipulate by trying a different number to see if there is any measures to stop one seeing information about other users.
4. Since we know the ID parameter is a number we can try and see if it auto increments or the id are generated by trying a number next to it that is 28 or 30 or any number![BOLA1.png](/images/BOLA1.png) As you can see by changing the ID parameter and placing other ID I can see other users information such as their name and course they are doing. This has serious implications as users information aren't then confidential.
5. We can use the capability of burp intruder to brute-force the ID parameter rather than manually changing the ID value. So send any of the requests that we have proxied and follow the following instructions:![API1 intruder.png](/images/API1%20intruder.png) Go to the intruder tab in burp and select attack type as in (1), Then Highlight where you want to bruteforce for our case it is the ID parameter so we highlight the value and after highlighting click on add so it looks something like (2,3). On the right side set the payload type to numbers as in (4) since we are dealing with numbers then set to sequential to iterate and set the range (5,6) and then start the attack.
6. Once the attack is done you can go through the results and see other users.  One specific ID has a flag find it and note it. ![API1 Flagredacted.png](/images/API1%20Flagredacted.png)
7. We can now go to the next Request which updates a user details we'll try update ourselves first to get to understand how it works.![Pasted image 20260522212212.png](/images/Pasted%20image%2020260522212212.png)This works as expected since as a user i should be able to update my details with no issues.
8. Then we will try update another user for this we will use ID 25![API1 ID25.png](/images/API1%20ID25.png) The above shows the original account without any changes. ![Pasted image 20260522213039.png](/images/Pasted%20image%2020260522213039.png) After updating the body with different details one is able to update another user details information without their authorization. This is dangerous since the PUT method is used to update details and with so it can't be reversed, this makes a legitimate user not access their account. This affects the Integrity and Availability of the account. ![Pasted image 20260522213640.png](/images/Pasted%20image%2020260522213640.png) We can see the details of the user has completely changed.

**Severity: <mark style="background: #FF5582A6;">Critical</mark> 
#### Impact
- Confidentiality is broken because any authenticated user can read any other user's data. 
- Integrity is broken because user records can be overwritten without authorization, and this action is irreversible given the destructive nature of PUT operations.
- Availability is broken because an attacker can lock legitimate users out of their own accounts by modifying their credentials.
- It has resulted to most significant data breaches as it requires no elevated privilege, no sophisticated tooling and no deep technical knowledge to exploit.
#### Remediation
- Implement server-side authorization checks on every object-level request, verifying that the authenticated user owns or has explicit permission to access the requested resource.
- Hide the ID parameter or obfuscate it to reduce attack surface
- Replace sequential integer IDs with non-enumerable identifiers such as UUIDs.
- Apply the principle of least privilege to all API endpoints by default.
#### Summary 
Broken Object Level Authorization is consistently ranked as the most prevalent and most damaging API vulnerability. It occurs when an API uses a predictable, user-controlled identifier such as a numeric ID  to retrieve or modify objects, without verifying that the requesting user has permission to access that specific object. In this assessment, the API1 endpoint accepted a numeric user ID parameter in both its GET and PUT requests. Upon creating a test account, the application returned an auto-incrementing integer ID. By systematically substituting this ID with adjacent integers using Burp Suite Intruder, it was possible to retrieve the full account details of every other user registered in the system. More critically, the PUT method accepted updates to any user record regardless of the authenticated user's identity, allowing complete modification  and therefore takeover  of any account.

### API2
![API2.png](/images/API2.png)
Authentication is the process of verifying the identity of a user. Broken authentication is where authentication controls implemented in a system can be bypassed through various ways. There are 2 ways for this, 1. lack of authentication mechanism or 2. Misimplementation of the authentication mechanisms.
There is a hint that has been given to us showing that there is a folder named Resources, we'll navigate and look for it and see what it contains.
1. In the terminal we will look at the contents of things in the Resources folder![API2 resources.png](/images/API2%20resources.png) We note it seems to be a list of credentials probably usernames and passwords.
2. Let's see what it entails. It seems that one is required to enter an email and password as shown below, however to be authenticated one has to know the correct credentials.![API2 Post.png](/images/API2%20Post.png) The 401 shows we have gotten the wrong credentials, we will have to guess the passwords. There are 2 ways to this either manually inputting each credential or brute-forcing it.
3. Manually filling it will take time while brute-forcing also won't work if there are any rate limiting or any other security measures are in place. We'll try bruteforcing to rule out if its applicable or not.
4. First we will need to separate the credentials.csv file to match email and password so that we can input it as a file.
5. To separate the 2 we will use a cut command as shown below: `cut -d',' -f1 creds.csv > ~/projects/writeups/Vapi/username.txt` 
![API2 username.png](/images/API2%20username.png) 
6. And the same is done for the password part  `cut -d',' -f2 creds.csv > ~/projects/writeups/Vapi/password.txt` ![password.txt.png](/images/password.txt.png)
7. We will use burp to bruteforce the credentials for this you can also use ffuf  for this also
8. Using burp, send the request to intruder ![Pasted image 20260522232248.png](/images/Pasted%20image%2020260522232248.png) Highlight the email and then click (2), do the same for password, then select attack to pitchfork (3) then go to payload position and select payload type and load username.txt file and repeat for password (4,5,6) ensure payload encoding is disabled(7) then start attack(8).
9.  Once the attack is done, click the status code column and click it to see any difference with the codes. You'll note there are 3 different status codes, showing we have a success. ![API2 Redacted intruder.png](/images/API2%20Redacted%20intruder.png) Note i have hidden the payload2 Column so you can get the password.
10. Another alternative is to use ffuf which is a web fuzzing tool if using burpsuite community which is heavily throttled. To use ffuf use this command on the terminal:  `ffuf -w username.txt:W1 -w password.txt:W2 -X POST -d '{"email":"W1", "password":"W2"}' -u http://localhost:9000/vapi/api2/user/login -mode pitchfork -mc 200
`
- the -w specifies the wordlist where W1 is for username.txt and w2 for password.txt. The username and passwords file are just the files we separated above in (5) and (6) respectively.
-  The -X is used to specify the HTTP method to use for us its POST
- The -d is used to define the payload
- The -u is used to define the target URL
- The mode is used to specify the fuzzing mode  which for us is Pitchfork.
- The -mc Is used to only display responses that have status code 200 which we have specified.
The response is: 
![API2 Ffuf redacted.png](/images/API2%20Ffuf%20redacted.png) Similar to response from Burp I get 3 credentials, we can use all to try and see what each entails.
When you login you will get different token and one has a flag get it and store it / save it

**Severity: <mark style="background: #FF5582A6;">Critical</mark> 
#### Impact 
- Account takeover- An attacker is able to gain unauthorized access to an account of their choice since they have access to the credentials
- Resource overhaul and denial of service attack- A system can be brought down by excessive requests which the server can't withstand making it not be accessible to legitimate  users.
#### Remediation
- Implement rate limiting to all access endpoints. This will reduce the ability to bruteforce an account.
- Implement multi-factor authentication mechanism where not only one authentication is accepted but 2 or more.
- Implement account lockout preventing brute force on accounts after a defined number of failed attempts.
- Monitor for unusual authentication patterns using SIEM or other equivalent logging systems.
#### Summary
Authentication is the mechanism by which an API verifies that a user is who they claim to be. When this mechanism is absent, poorly implemented or unprotected against automated attacks, it becomes trivially bypassable. API2 covers both the complete absence of authentication and its misimplementation  in this case, the failure to implement any anti-brute-force protection on the login endpoint. The Resources folder provided with the lab contained a credential list in CSV format of leaked credentials from potentially a prior breach. Using the `cut` command to separate usernames and passwords into individual wordlists, a credential stuffing attack was executed against the login endpoint using both Burp Suite Intruder (pitchfork mode) and ffuf. The absence of rate limiting, account lockout, or CAPTCHA meant the attack completed without interruption, successfully identifying three valid credential pairs from a list of one thousand entries in approximately one minute. Upon successful authentication, each valid session returned a unique token one of which contained a flag confirming privilege escalation. In a production context, this attack would grant an attacker the full session permissions of the compromised user, including access to all data and actions available to that account.
### API3
![API3.png](/images/API3.png) 
Excessive Data Exposure is where sensitive information is being returned to the user in the response and is visible on the web traffic by design. Sniffing the traffic an attacker can see thr sensitive data.
We are informed that there is an android app in the resource folder and it exposes excessive data.
```
Preriquisite for this is to have an android emulator such as android studio or genymotion where we will use virtual devices. There is also need to have knowledge of setting it up.
- For this I will be using genymotion as my choice of emulator
- Ensure the virtual android to be used is android 8 and above.
- Having setup the emulator and powering it up we can continue.
```

When we go to the resource folder we get an APK file: ![API3 Resources.png](/images/API3%20Resources.png)
1. Having known the path of the APK there are different ways to install the app,we can drag and drop the apk to install the app ![AP13 DRAG APK.png](/images/AP13%20DRAG%20APK.png) Also we can use ADB to install it by using `adb install TheCommentApp.apk` from where the APK is from as  ![API3 ADB APK.png](/images/API3%20ADB%20APK.png)                This both will lead to the same end of installing the APK as shown here by swiping up the homepage ![AP13 APPS.png](/images/AP13%20APPS.png) 
2. We can now open the application where we are required to pick a base URL ![Pasted image 20260601193522.png](/images/Pasted%20image%2020260601193522.png) However in the resources folder there is a readme file that tells us how to name the base url as shown: ![Pasted image 20260601193831.png](/images/Pasted%20image%2020260601193831.png) Therefore at the end of the baseurl add /vapi  ![API3 BASE url 1.png](/images/API3%20BASE%20url%201.png)
3. Once we click the save button we are redirected to a login page, since we haven't created an account we will register one by clicking the Create account: ![API3 Login.png](/images/API3%20Login.png) After successfully registering and logging in, we are directed to some comments. I had created 2 accounts for testing.
4. When you open the get request containing the comments we get to see some information that can't be seen on the app: ![API3 Comments exposure.png](/images/API3%20Comments%20exposure.png) When you look at the right hand side the only information being displayed is the comment and username however on the left we get to see deviceid, longitude and latitude. we also get to see a flag.
5. We can add a comment to see how the app works: ![API3 Comments](/images/API3%20Comments.png) Ensure to allow the app permission to access location, write a comment and send it where a pop up message will be generated and in the emulator ensure that GPS is allowed as shown in (2)
6. Once we allow the permissions, and send the comment we can look at the GET request where we see our comment is sent plus our latitude and longitude ![API3 Comments 2.png](/images/API3%20Comments%202.png)

**Severity:** High
#### Impact
- Latitude and longitude data attached to user comments enables an attacker to map the physical locations from which users regularly post, constructing a detailed picture of a target's daily movements and frequented locations.
- Device ID exposure enables device fingerprinting and tracking.
- Lawsuit since exposure of Personal identifiable information can lead to fines and lawsuits.
#### Remediation
- Apply a server-side response filter that returns only the fields explicitly required by the consuming application.
- Collect only the necessary information required by the application.
- Conduct regular API response audits to identify and remove unnecessary data exposure. 
- Define a strict response schema for each endpoint and enforce it at the API layer.
#### Summary
Excessive Data Exposure occurs when an API returns more information in its response than is required by the client application, relying on the client to filter out sensitive fields before displaying them to the user. This is a fundamentally flawed design pattern because it assumes that the only consumer of the API is the intended front-end application  an assumption that any attacker with a proxy can immediately invalidate. This finding was demonstrated using a companion Android application (TheCommentApp.apk) deployed in  Genymotion, an android emulator and proxied through Burp Suite. The application's User Interface displayed only two fields per comment: the comment text and the posting username. However, inspection of the underlying GET request response revealed that the API was returning four additional fields: a device ID, latitude, longitude, and a flag value. None of these fields were visible in the application interface, yet all were present in every API response.

### API4

![API4.png](/images/API4.png) 
This is where multiple concurrent requests can be performed from a single device in a certain time-frame.
Based on the information there is no rate limiting so this is a chance for bruteforcing.
1. Initiate the OTP process by sending the number the OTP to be sent, I've gone for the default number but you can just change it. ![API4 POST OTP.png](/images/API4%20POST%20OTP.png) After sending the request an OTP is generated and we get an information such as it is a 4 digit.
2. When we send a random OTP number to verify it we get to see how it responds: ![API4 Verify OTP.png](/images/API4%20Verify%20OTP.png) We get a status 403, and an invalid OTP message. We need to get the valid OTP to get a different status code and message.
3. We can now bruteforce the OTP by sending the request to intruder and setting up the payloads in burp: ![API4 OTP intruder.png](/images/API4%20OTP%20intruder.png) Setting the OTP payload position, set the attack type(2), Setting the number range (3), placing for 4 digits since we know what OTP (4) and start the attack (5).
4. After a successful bruteforce attack is completed we get a successful OTP and a status 200 ok ![669](/images/API4%20OTP%20Results.png) We get an OTP result. This is as seen in POSTMAN. ![669](/images/Pasted%20image%2020260524152114.png)
5. When we go to the next request we get information about the user as well as a flag: ![API4 redacted flag.png](/images/API4%20redacted%20flag.png) 
**Severity:** High
#### Impact
- It can lead to a denial of service (DOS) since it is overwhelmed by the requests.
- Legitimate and authenticated users can't access the services hindering confidentiality. 
#### Remediation
- Implement rate limiting on all authentication and verification endpoints.
- Limit a number of request a device can make in a specific duration of time.
- Implement a load balancer that distributes incoming traffic across multiple servers to avoid a single server to be overloaded.
- Implement account lockout or progressive delay after repeated failures.
#### Summary
Rate limiting governs how many requests a client can make to an API endpoint within a defined time window. Without it, an API is fully exposed to automated abuse including brute force attacks against authentication mechanisms, enumeration of identifiers, and resource exhaustion attacks that can deny service to legitimate users.The API4 endpoint presented a two-step OTP authentication flow. Initiating the process revealed that the OTP was a four-digit numeric code a search space of exactly 10,000 possible values. When a random incorrect OTP was submitted, the server responded with a 403 Forbidden status. Critically, there was no observable rate limiting, no lockout mechanism, and no expiry enforcement on the OTP. Using Burp Suite Intruder in Sniper mode with a sequential numeric payload from 0000 to 9999, the correct OTP was identified in under five minutes as the sole response returning a 200 OK status code.The follow-up GET request to retrieve user details, available once authenticated, returned both the user's information and a flag, confirming complete authentication bypass.

### API5
![API5.png](/images/API5.png)  Broken Function Level Authorization is where an unauthorized user is able to perform functions that they are not allowed to don't have the permissions.
1. For the first request we are going to create a user ![API5 Post.png](/images/API5%20Post.png) So a user has been created successfully.
2. The second request is used to get information about the user that we have created as shown: ![API5 GET.png](/images/API5%20GET.png) We can see that as a user the id is being used to identify the user.
3. Look at the HTTP methods we can look at the OPTIONS to see which methods are allowed: ![Pasted image 20260524233414.png](/images/Pasted%20image%2020260524233414.png) Nothing much since only GET and HEAD are allowed so we can rule out using POST or PUT.
4. We can try and work around the ID parameter by changing the values and see if there is a BOLA that we can work with. ![Pasted image 20260524234731.png](/images/Pasted%20image%2020260524234731.png) As we can see only the authorized ID give us a status 200. So with this we can't rely on BOLA.
5. We can try the other endpoint /user and try see if any success can be found by changing user to admin: ![Pasted image 20260524235925.png](/images/Pasted%20image%2020260524235925.png) No success since we get a status code 500 meaning the endpoint admin is non existent.
6. we can try back endpoint user and try out various endpoints: ![Pasted image 20260525000437.png](/images/Pasted%20image%2020260525000437.png) By changing the endpoint to users, we get information about all users and get information on a flag as shown above.
7. This however looks more of a BOLA since no function has been done.
#### Impact
- Attacker can gain higher privilege access that can increase the severity of the attack.
#### Remediation
- Implementing least privilege access to all users by default. This will reduce the attack vector where a user is given the least amount permissions that are needed to do their work.
- Implement role-based access controls to resources where users are only allowed to interact with endpoints, resources that are needed to perform their hob functions.
- Deny all access by default and requiring explicit grants to specific roles to access every function.
#### Summary
We were able to perform admin functions by listing all users in the system, however this feels more of BOLA since no specific function can be done after listing out the users.Broken Function Level Authorization differs from BOLA (API1) in its scope. While BOLA concerns unauthorized access to specific objects , Broken Function Level Authorization concerns unauthorized access to entire API functions particularly administrative functions that should be restricted to privileged roles. The API5 endpoint presented a standard user registration and retrieval flow. After creating a test user and confirming normal GET access using the assigned ID, attention was directed toward the endpoint structure itself. By modifying the endpoint path from `/user/{id}` to `/users`  a small but significant alteration  the API returned a complete listing of all registered users in the system, including an administrative account with a flag embedded in its address field. This exposure was possible because the application failed to apply any authorization check to the `/users` endpoint, allowing any authenticated user to invoke what was effectively an administrative function. 
### API6
![API6.png](/images/API6.png)  

Mass assignment is where a user can manipulate the object parameters by giving out values of the properties from user input.
We get a hint on the credit management so we can target it.
1. When we create a user we note that a new parameter id is generated no much on the credit parameter as seen: ![API6 POST.png](/images/API6%20POST.png) Nothing much here
2. on the second request we get to see the credit parameter: ![API6 GET.png](/images/API6%20GET.png) We get to see the Credit is set to 0 
3. So in the previous POST request we can add the credits parameter in our body and set an amount for ourselves:  ![API6 POST2.png](/images/API6%20POST2.png) Add a new parameter credit and set an amount of your choice, note to place a comma before the new entry also place all into a double quotes ""
4. We can now go and see if it was a success on the Get request: ![API6 GET2.png](/images/API6%20GET2.png) As we can see it was a success as the credit amount we had set is a success.
5. We were able to manipulate the credit property by setting an amount for our self in the credit parameter.
**Severity** High 
#### Impact
- It can lead to privilege escalation by changing functions from a normal user to an admin user.
- It tampers with the data as it is easy to change by giving user input values.
- Fraudulent purchases in an e-commerce context with no financial cost. 
#### Remediation
- Whitelist only parameters that should be updated by client and blacklist the rest 
- Use Data Transfer Objects (DTOs) or equivalent schema validation to define the exact shape of acceptable input for each endpoint.
#### Summary
Mass Assignment occurs when an API automatically binds user-supplied input to internal object properties without restricting which properties the user is permitted to modify. This vulnerability is particularly insidious because it often exists not due to an explicit error in the code, but due to the convenience of frameworks that automatically map request body parameters to data model attributes. The API6 endpoint simulated a retail store credit system. Upon creating a user account, a GET request to retrieve user details revealed a `credit` field set to zero  a field that had not appeared in the original registration response and was clearly intended to be managed server-side. By re-submitting the POST registration request with an additional `credit` parameter manually inserted into the request body and set to an arbitrary value, the application accepted the input without validation and persisted the attacker defined credit balance. A subsequent GET request confirmed the manipulation had succeeded.

### API7
![API7.png](/images/API7.png)
It is commonly when the configurations in place are done poorly leaving room to expose vulnerabilities. An API is vulnerable to security misconfiguration if a cross-origin resource sharing (CORS) policy is missing or misconfigured.
Cross origin requests is a rule that blocks front end from making requests to a different origin unless that origin is explicitly allowed.
1. We will create a user using the first Post request. ![API7 Post.png](/images/API7%20Post.png) A successful user is created.
2. In the second request, there is a user login request ![API7 Login.png](/images/API7%20Login.png) It doesn't take any parameters but logs you in.
3. In the next request it fetches an authentication key: ![API7 Key.png](/images/API7%20Key.png) However we get a valuable information on the HTTP headers where it allows any origin to access this request as well as credentialed cross origin requests.
4. With that information we can make a request and add an origin header by sending the request to repeater on burp as: ![API7 Key Flag.png](/images/API7%20Key%20Flag.png) When we add the origin header, we get a flag as the origin is accepted. note the value can be anything since it is accepting all, due to the * sign which means all.
5. The last request just logs out a user: ![API7 Logout.png](/images/API7%20Logout.png)
#### Impact
- It enables malicious websites to make authenticated API calls on behalf of any logged-in user who visits the attacker's page, effectively turning the browser into an unwitting attack proxy
- It can expose sensitive user data
- It can lead to a full server  compromise
#### Remediation
-  Proper configuration of cross origin requests by properly specifying the origin header
- Should only allow trusted sites
- Avoid using wildcards as it can be accessed by external domains.
- Apply CORS policy validation as part of the deployment pipeline to prevent misconfiguration from reaching production.
#### Summary
Security misconfiguration represents a broad category of vulnerabilities arising not from flaws in application logic but from incorrect or missing security controls at the configuration level. One of the most common and consequential manifestations in APIs is the misconfiguration of Cross-Origin Resource Sharing (CORS) policy. CORS is a browser enforced mechanism that controls which external origins are permitted to make credentialed requests to an API. When properly configured, it restricts API access to trusted domains. When misconfigured particularly when the `Access-Control-Allow-Origin` header is set to a wildcard value of `*` in combination with `Access-Control-Allow-Credentials: true` the protection is entirely nullified. Inspection of the API7 response headers after fetching the authentication key endpoint revealed exactly this misconfiguration. The server was accepting requests from any origin and allowing credentialed cross-origin requests. By forwarding the request to Burp Repeater and adding an arbitrary `Origin: http://attacker.com` header, the server reflected the attacker supplied origin in its response and returned a flag embedded in the authentication key response confirming that any external domain could make credentialed requests to this API.
### API8 
![697](/images/API8.png) The API seem to be vulnerable to an injection. An injection attack is where malicious user input is used by the system and is taken as a command, query, interpreter.
Injection include: SQLI, NOSQL, XSS, OS Commands.
1. In the first request we will enter any credential and see how the API behaves when the credentials are wrong.  ![API8 POST1.png](/images/API8%20POST1.png) As we can see it requires to know the correct credentials.
2. We will now try out some injection payloads to try and see how the API behaves: ![API8 Injection1.png](/images/API8%20Injection1.png) By adding ' on username we get a verbose error message that tells us this is a SQL database so we can test with SQLI payloads.
3. Trying out some manual payloads we get some errors and status code 500, as shown: ![Pasted image 20260525140039.png](/images/Pasted%20image%2020260525140039.png) 
4. We will have to use a tool called SQLMAP which is great for SQL injection ![Pasted image 20260525141115.png](/images/Pasted%20image%2020260525141115.png) You can use the -h to familiarize yourself with the flags being used.
5. We will have to specify the target URL ,  the data and the parameter we are planning to target as shown below: ![Pasted image 20260525142947.png](/images/Pasted%20image%2020260525142947.png) `sqlmap -u "http://localhost:9000/vapi/api8/user/login" --data="username=john&password=Test" -p username --dbs --proxy="http://127.0.0.1:8080" `  The above command gives a list of database present by the use of the -dbs flag. The proxy is used to proxy this through Burp to help understand what is being sent
6. Since we know the name of an interesting table we will then view its contents. Looking at the SQLMAP help we can see that the flag for listing columns in a table is **--columns** and specify the database using this command sqlmap -u "http://localhost:9000/vapi/api8/user/login" --data="username=john&password=Test" -p username --dbs --proxy="http://127.0.0.1:8080" -D vapi --columns : ![Pasted image 20260525144107.png](/images/Pasted%20image%2020260525144107.png) We can see there are various columns  and one that interest us is the a_p_i8_users as it matches with the current API we are working on.
7. We can see that column and what it contains by just scrolling to that column: ![Pasted image 20260525145528.png](/images/Pasted%20image%2020260525145528.png) we can see there are columns like username and password that are needed to login.
8. We can try to output the contents of the column by specifying the columcs we want to enumerate and the flag for this is  -C and dump the contents using --dump `sqlmap -u "http://localhost:9000/vapi/api8/user/login" --data="username=john&password=Test" -p username --dbs --proxy="http://127.0.0.1:8080" -D vapi -T a_p_i8_users -C password,secret,username --dump`   :![Pasted image 20260525150510.png](/images/Pasted%20image%2020260525150510.png) We can see we get a username and password that we can now use in the POST request as our credentials. 
9. Using the credentials we found we get a successful login: ![API8 POST 200.png](/images/API8%20POST%20200.png)
10. The next request gives us the same flag that we got in the sqlmap results.![API8 GET.png](/images/API8%20GET.png)
11. We have been able to use SQLI to be able to retrieve information from the database, the username and password.

**Severity:** Critical
#### Impact
- Data manipulation, deletion or retrieval from database. This leads to loss of confidentiality, Integrity and Availability of the data.
- It may also lead to Denial of Service(DoS), or complete host takeover.
- Information disclosure by getting information stored in the database.

#### Remediation
- Use parameterized queries or prepared statements for all database interactions
- Special characters should be escaped using specific syntax.
- Suppress verbose error message by returning generic error responses rather than specific error messages.
- Apply  the principle of least privilege to the database user account used by application.
#### Summary
We were able to perform a successful SQL injection using SQLMAP to get the flag that was stored on the database. We could list all the tables and the content in the database and view it. This included sensitive information such as password that were stored by the application. This would be safeguarded by implementing input sanitization where input is validated before it is implemented.

Injection vulnerabilities occur when user supplied input is incorporated into a query, command, or interpreter without adequate sanitization allowing the input to be interpreted as code rather than data. SQL injection remains one of the most documented and consequential vulnerability classes in application security, and its presence in an API is no less severe than in a traditional web application. The API8 login endpoint provided no credentials and challenged the tester to authenticate. Submitting a single apostrophe character in the username field produced a verbose MySQL error message, confirming that user input was being directly interpolated into a SQL query and that error output was being returned to the client a compounding issue that substantially assists an attacker in crafting effective payloads. SQLMap was used to automate exploitation. An initial scan enumerated all available databases, identifying the `vapi` database as the target of interest. A subsequent column enumeration query against the `vapi` database identified the `a_p_i8_users` table, which contained `username`, `password`, and `secret` columns. Dumping the table contents returned a single administrative credential set in plaintext, along with a flag value stored in the `secret` column. Using these credentials to authenticate via the standard login endpoint produced a successful session.

### API9
![API9.png](/images/API9.png) This involves looking for old API that aren't retired or unpatched. The current version may have robust security mechanisms however, previous versions and are online act as an easy gateway since security measures aren't updated when others are being implemented
1. When we run the first request we get to see the version being used is a v2 also some information in the header: ![API9 POST V2.png](/images/API9%20POST%20V2.png) Their is a X-RateLimit which shows their is a rate limit and after sending a request we get a reduction of the one's remaining.
2. However, when we change the version to v1 the ratelimit isn't implemented as shown here: ![API9 POST2.png](/images/API9%20POST2.png) We saw that their is no rate limit so we can bruteforce it.
3. We can send the request to intruder and set up the payloads. ![Pasted image 20260525224252.png](/images/Pasted%20image%2020260525224252.png) Change the version to v1 and set the payload as in (2), and place the payload in numbers and number range as in (3,4,5) and start the attack.
4. After a successful attack, the results we get a different length showing there is a difference showing a successful attack: ![API9 Results.png](/images/API9%20Results.png) 
**Severity:** High
#### Impact
- Attackers may gain access to sensitive information]
- Easy attack vector for attackers to takeover the server through use of old or unpatched versions.
#### Remediation
- Having an up-to date inventory of all assets that are owned or used by the organization. 
- Well documentation of all assets/ api being used and the requests and responses by each
- Updating all versions when doing updates and upgrades.
- Retiring old or unused assets by taking them offline and refusing any connections made to them.
- Establish a formal API lifecycle policy that defines when versions are deprecated and retired.
#### Summary
Improper Assets Management describes the risk that arises when organizations fail to maintain an accurate, current inventory of their API surface. As APIs evolve, new versions are released and old versions are deprecated  but deprecation in policy does not always mean retirement in practice. Legacy API versions that remain accessible often lack the security controls implemented in their successors, creating a lower resistance attack path that bypasses the protections an organization believes are in place. The API9 endpoint operated on version v2, which correctly implemented an `X-RateLimit` header limiting authentication attempts per session. By inspecting the response headers and noting the version path in the URL, the assessment tested whether the previous version v1 remained accessible. It did. More significantly, the v1 endpoint returned responses with no rate limit headers at all, confirming that the rate limiting control had been implemented in v2 but never retrofitted to the older version. Using Burp Suite Intruder against the v1 endpoint with a sequential four-digit numeric payload targeting the PIN field, the correct PIN was identified within the full 0000–9999 search space. The authenticated response returned the user's account balance along with a flag data that the v2 rate limit was specifically designed to protect.
### API10
![API10.png](/images/API10.png)   
This is where attacker take advantage of lack of logging and monitoring to abuse systems and go unnoticed.
1. This is much easier since we get sent the flag by just sending the request: ![Pasted image 20260525232035.png](/images/Pasted%20image%2020260525232035.png)
**Severity:** Medium
#### Impact
- With no visibility an attacker has unlimited time to fully compromise a system
- Won't know the point of compromise, the extent of access and duration of the compromise the attacker uses so hard to fix it.
#### Remediation
- Log all failed authentication mechanism, denied access.
- Logs should include enough details that can be used to identify an attacker such as timestamps, source IP, endpoint accessed, request size.
- Configure a monitoring system to continuously monitor the systems
- Implement a Security Information and Event Management system(SIEM) to manage all logs being generated and can raise alerts based on the rules set.
#### Summary
 Insufficient logging and monitoring is unique among the OWASP API Top 10 in that it is not a vulnerability that enables direct exploitation  rather, it is the absence of the defensive capability that would detect and limit the impact of every other vulnerability on this list. An attacker who has exploited any of the preceding findings in a system with adequate logging would trigger alerts, generate audit trails, and potentially initiate an incident response. In a system without logging, that same attacker operates in complete darkness, with unlimited time and no detection risk. The API10 endpoint demonstrated this condition directly: sending a simple GET request to the flag endpoint returned a response that explicitly acknowledged the absence of logging the flag was delivered freely, with a message confirming that no requests had been recorded.
### Arena
### Just weak tokens (JWT)
1. In the post request when we create a user as shown below a token is generated for us: ![JWT POST request.png](/images/JWT%20POST%20request.png)
2. In the second request when we send it we are authenticated as that user we created showing the token is being used to authenticate: ![JWT GET user.png](/images/JWT%20GET%20user.png)
3.  A good thing to note the token is a jwt token as it can be seen by the first 3 letters the eyj also by the heading of the folder.
4. We will use a tool called jwt_tool which is good for jwt tokens testing. We will need to know what the token consists of by: `jwt_tool "authorization token"` ![JWT token.png](/images/JWT%20token.png) Few things to note here: the algorithm being used here is HS256, the role assigned to us is a user and the token expires by 30 minutes.
5. We will now try and remove the algorithm by `jwt_tool "token" -X a` : ![Pasted image 20260527120406.png](/images/Pasted%20image%2020260527120406.png) The above removes the algorithm , so copy any of the generated JWT tokens but ensure to copy till the . 
6. The above can be proved by:  ![Pasted image 20260527121245.png](/images/Pasted%20image%2020260527121245.png)
7. We will now try to edit the contents / tamper with the JWT  by changing the role from a user to an admin, note we will be using the JWT we generated above that has no algorithm by using `jwt_tool "JWT" -T`: ![JWT Tamper.png](/images/JWT%20Tamper.png) Ensure to use the JWT with the . at the end, also change the role to admin as shown in (2) and copy the token that is generated as shown in (3)
8. Go back to burp and send the request with the JWT that is provided to us from the beginning without changing anything: ![JWT Burp.png](/images/JWT%20Burp.png) Note how the API responds as shown above.
9. However, going back and sending this request to repeater and change the authorization token to what we generated in 7 we get to see this:  ![669](/images/JWT%20Burp%20tamper.png) This can also be achieved in postman by changing the value of the Authorization-Token to the one was  tampered with ![JWT POSTMAN.png](/images/JWT%20POSTMAN.png) Go to headers as in (1), go to the authorization-token header and change the value to the one we generated (3) and send the request , where we will get the flag (5).
**Severity:** High
#### Impact
- Privilege escalation by creating valid tokens
- Impersonation of other users
#### Remediation
- Performing and implementing a robust signature verification of any JWTs 
- Enforce the signing algorithm on the server side.
- Never allowing JWT that are have no algorithm.
#### Summary
JSON Web Tokens are a widely adopted mechanism for stateless API authentication. A JWT consists of three base64-encoded segments  a header declaring the algorithm, a payload containing claims, and a signature verifying integrity. The security of the entire mechanism depends on the server validating the signature using the expected algorithm and a secret key. If either the algorithm check or the signature verification is bypassed, the token can be forged. The Arena JWT challenge exposed two weaknesses simultaneously. First, the token's role claim was set to `user` a field that controls authorization level and should be server-authoritative. Second, the server accepted tokens using the `none` algorithm  a value that explicitly signals the absence of a cryptographic signature. This is a known vulnerability in JWT implementations that trust the algorithm specified in the token's own header rather than enforcing a server-configured algorithm.
Using jwt_tool, the original token was first analyzed to confirm the algorithm (HS256), role (user), and expiry (30 minutes). The `-X a` flag was then used to generate algorithm-stripped variants with `alg: none`. The tamper flag (`-T`) was used to modify the role claim from `user` to `admin`. When the modified token was submitted in the Authorization-Token header, the server accepted it and returned a flag  confirming that it had trusted the token's self-declared algorithm and role without independent verification.

### ServerSurfer
1. In the GET request a specific URL is being requested to and we get an encoded data as shown here:  ![Pasted image 20260527144948.png](/images/Pasted%20image%2020260527144948.png) We get a random string as the data results output and when I highlight it we see that it has been encoded and when you click the Inspector you can actually see the context of the content.
2. We will send a request to a server that we can control and see the contents, so we will use webhook.site for this: ![Webhook.site.png](/images/Webhook.site.png) We will also add content of what is to be displayed when someone interacts with the server. ![webhook.site content.png](/images/webhook.site%20content.png) You can copy the unique url and replace it to our url and send the request 
3. When you send the request we get to see the message that we initially set above when decoded ![Pasted image 20260527151819.png](/images/Pasted%20image%2020260527151819.png) After we decode we get to see information about the content we added
4. Looking at the homepage of the webhooksite, we get to see that after every target to the server it logs the details of the request as shown below:  ![Pasted image 20260527153702.png](/images/Pasted%20image%2020260527153702.png)
**Severity:** High
#### Remediation
- Validate and sanitize all user supplied URLs before use.
- Implementing an allow list of permitted domains and reject the rest.
#### Summary
Server-Side Request Forgery is a vulnerability that allows an attacker to induce the server to make HTTP requests to an attacker specified destination. This is particularly dangerous in cloud environments, where the server's internal network may include metadata services (such as AWS  at 169.254.169.254) that expose instance credentials and configuration data to any process that can reach them from within the cloud network. The ServerSurfer endpoint accepted a `url` parameter in its GET request, which the server used to fetch and return the content of the specified URL. Substituting the default roottusk.com value with a webhook.site URL  a service that captures and logs incoming HTTP requests  the server dutifully fetched the attacker-controlled page and returned its content. The webhook.site dashboard logged the server's IP address, request headers, and request metadata, confirming the SSRF was active and that the server was making outbound requests using its own network identity. By configuring the webhook response to include custom content, the server reflected that content in its API response  demonstrating that the vulnerability enables both outbound SSRF (the server fetching attacker content) and response reflection. 
### Sticky Notes
1. In the sticky notes the POST request; ![Pasted image 20260601151938.png](/images/Pasted%20image%2020260601151938.png)
2. On the second request we get the information we just posted in the previous request as seen here: ![Pasted image 20260601153821.png](/images/Pasted%20image%2020260601153821.png) We can see that one is able to change the format of how the details can be displayed as by changing as seen in the highlighted part.
3. We will try and see if there is a stored cross site scripting(xss) by placing an alert message that is reflected once a user interacts with the page. ![Pasted image 20260601154357.png](/images/Pasted%20image%2020260601154357.png) Note that there is need for this \ since it bypasses the safeguard in place since when you don't place them we get a 500 status error code. ![Pasted image 20260601154642.png](/images/Pasted%20image%2020260601154642.png)
4. When you get that request, we get to have a flag as well as an alert message: ![Pasted image 20260601160319.png](/images/Pasted%20image%2020260601160319.png)
5. We can confirm if the XSS was successful by opening our response in our browser: On the request right click and you'll see an option of open response in browser and click it which will lead you to this ![Pasted image 20260601161033.png](/images/Pasted%20image%2020260601161033.png) Click it and copy the URL and open it in any choice of your browser, preferably where the proxy is set up in.
6. Once you open the response we get, ![Pasted image 20260601161355.png](/images/Pasted%20image%2020260601161355.png) This shows that our XSS was successful. Click the OK button which will lead us to the flag message.![Pasted image 20260601161549.png](/images/Pasted%20image%2020260601161549.png) Once we reload the page the alert message pops up again:![Pasted image 20260601161657.png](/images/Pasted%20image%2020260601161657.png) This shows the script is being run every time a user interacts with the page.
**Severity:** High
#### Impact 
- This would enable session hijacking, credential theft via phishing overlays, keylogging, and forced redirection to malicious domains.
#### Remediation
- Sanitize all user supplied input before storage. 
- Implement a strong Content Security Policy header that restricts script execution to trusted sources.
#### Summary
Cross-Site Scripting is an injection vulnerability in which an attacker causes a victim's browser to execute attacker supplied JavaScript. In its stored variant  the most severe form the malicious payload is persisted in the application's database and executed for every user who subsequently views the affected content, without any interaction beyond loading the page. The Sticky Notes endpoint accepted notes via POST and offered multiple output formats via a `format` query parameter (JSON, XML, HTML). When the HTML format was requested, the API returned raw HTML to the browser without sanitizing the stored content. By submitting a note containing a JavaScript alert payload using escape characters to bypass a naive input filter that rejected unescaped script tags the payload was accepted and stored. A subsequent GET request with `format=html` returned the stored XSS payload in an HTML response. Opening this response in a proxied browser triggered an alert dialog, confirming successful JavaScript execution. The XML format response additionally revealed a flag in a `<flag>` element, confirming the intended finding.
The persistence of the payload was confirmed by reloading the page the alert fired on every page load, demonstrating that the script would execute for every user who viewed the notes, not only the attacker. 
### Conclusion

This assessment demonstrates in practical terms what OWASP has documented as statistical reality: API security vulnerabilities are widespread, consistently exploited, and entirely preventable. Every finding in this report was confirmed through systematic, methodical testing using tools and techniques that are freely available and well documented. None of the exploits required novel research or advanced capability they required only an understanding of how APIs work and what can go wrong when they are built without security as a design principle.
Several observations stand out from this assessment as lessons beyond the technical findings themselves.
- **Defense in depth is non-negotiable.** Multiple findings in this report were exploitable precisely because a single missing control  rate limiting, output encoding, authorization checking was the only thing standing between an attacker and a successful compromise. Real-world APIs require layered controls, where the failure of any one mechanism is caught by another.
- **Authentication is not authorization.** The distinction between API1, API2, and API5 makes this concrete. Knowing who a user is (authentication) does not determine what they are permitted to do (authorization). Many of the most impactful findings in this report involved authenticated users accessing data and functions they had no business accessing.
- **Old assets are attack surface.** API9 demonstrated that an organization's effective security posture is determined not by its best-secured endpoint but by its least-secured one. Legacy versions, forgotten endpoints, and undocumented routes are consistently among the most productive targets in real engagements.
- **Logging is a security control, not an operational nicety.** API10 is often underweighted in severity assessments because it does not enable direct exploitation. This is the wrong frame. In every confirmed breach in this report, adequate logging would have generated an alert. The absence of logging converts every other vulnerability from a recoverable incident into a silent, sustained compromise.
- **The OWASP API Top 10 is a baseline, not a ceiling.** The vulnerabilities documented in this report represent the most common, well-understood categories of API weakness. Real-world APIs present combinations of these vulnerabilities alongside application-specific logic flaws, infrastructure misconfigurations, and third-party integration risks that extend well beyond any standardized list.
The discipline required to produce secure APIs threat modeling, secure design, input validation, access control, logging, and ongoing testing  is not separate from the discipline required to build good software. It is the same discipline, applied with security as an explicit and non-negotiable requirement from the first line of code to the final deployment.

This report represents a complete traversal of the OWASP API Top 10 (2019) attack surface. The skills and methodologies demonstrated here  proxy-based traffic analysis, credential stuffing, brute force via Intruder and ffuf, SQL injection automation, JWT manipulation, SSRF exploitation, and XSS injection form the core technical vocabulary of API penetration testing. 