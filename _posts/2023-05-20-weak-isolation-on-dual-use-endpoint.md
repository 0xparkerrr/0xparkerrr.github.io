---
layout: post
title: Weak Isolation on Dual-Use Endpoint
description: A walkthrough covering PortSwigger's Bussiness Logic Vulnerabilties lab on dual-use endpoints
summary: A walkthrough covering PortSwigger's Bussiness Logic Vulnerabilities lab on dual-use endpoints
tags: portswigger writeups web
minute: 10
---
Table of Contents
- [Overview](#overview)
    - [What Are Business Logic Vulnerabilities?](#what-are-business-logic-vulnerabilities)
    - [How Do Business Logic Vulnerabilities Arise?](#how-do-business-logic-vulnerabilities-arise)
    - [Users won't always supply mandatory input](#users-wont-always-supply-mandatory-input)
- [Lab: Weak Isolation of Dual-Use Endpoints](#lab-weak-isolation-of-dual-use-endpoints)
    - [Technical Analysis](#technical-analysis)
    - [Exploitation](#exploitation)

# Overview
I've been tackling PortSwigger's Academy learning path and wanted to cover a lab that I managed to seamlessly solve on my own (which rarely ever happens!).

The lab is apart of the Business Logic Vulnerabilities course. But..
## What Are Business Logic Vulnerabilities?
I like to think of them as the results of weak exception handling. When developers implement a function, they intend for it to work a particular way. And if there are certain rules they must follow in a 'business context', should it not cover all weaknesses, they are business logic flaws. Otherwise, they're simply just logic flaws.

A very simple example would be a web store that puts excessive trust in the client when adding products to a cart. If the request includes data that dictates what the price of a product is, then a malicious user can modify that value to be cheaper. This is quite different than simply just inspecting the element and changing the value.

## How Do Business Logic Vulnerabilities Arise?
The concept is very high-level, but it can severely impact the application and/or business when overlooked. 

When developers implement new functionalities or components to an application, they need to understand all the existing components and application in its entirety. If they fail to do so, they will miss out on the possibilities of what a user might accidently do during certain steps of the function.

In this particular lab, the developers make flawed assumptions about user behavior that lead to a serious logic flaw.

## Users won't always supply mandatory input
As I am no expert in APIs and endpoints, here's an excerpt from PortSwigger:

     One misconception is that users will always supply values for mandatory input fields. Browsers may prevent ordinary users from submitting a form without a required input, but as we know, attackers can tamper with parameters in transit. This even extends to removing parameters entirely.

    This is a particular issue in cases where multiple functions are implemented within the same server-side script. In this case, the presence or absence of a particular parameter may determine which code is executed. Removing parameter values may allow an attacker to access code paths that are supposed to be out of reach.

    When probing for logic flaws, you should try removing each parameter in turn and observing what effect this has on the response.

One key point is that `removing parameter values may allow an attacker to access code paths that are supposed to be out of reach`.

# Lab: Weak Isolation of Dual-Use Endpoints
    This lab makes a flawed assumption about the user's privilege level based on their input. As a result, you can exploit the logic of its account management features to gain access to arbitrary users' accounts. To solve the lab, access the administrator account and delete Carlos.

    You can log in to your own account using the following credentials: wiener:peter 

## Technical Analysis
![](/assets/portswigger/logic-flaw/dashboard.PNG)

After looking around, we see that we have the ability to change our password. But what's interesting is that there is an input field that is controllable with our username. What we'll do is make a couple of requests to gather some information.

Correct credentials with incorrect new passwords:
{% highlight http %}
POST /my-account/change-password HTTP/1.1
Host: 0a33009103d284b981da025600820095.web-security-academy.net
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:109.0) Gecko/20100101 Firefox/113.0
Connection: close

csrf=iyYbS34tyUP5JRAKgeA2O0y9vtn0cqHg&username=wiener&current-password=peter&new-password-1=123&new-password-2=1234

Response:
<p class=is-warning>New passwords do not match</p>
{% endhighlight %}

Having mismatched new passwords allow us to trigger different error messages that can help us identify the flow of this function. In this case, if we have the right credentials but the new passwords don't match, it might mean the credentials are important in order for us to change the password.

Incorrect credentials with incorrect new passwords:
{% highlight http %}
POST /my-account/change-password HTTP/1.1
Host: 0a33009103d284b981da025600820095.web-security-academy.net
Connection: close

csrf=iyYbS34tyUP5JRAKgeA2O0y9vtn0cqHg&username=carlos&current-password=peter&new-password-1=123&new-password-2=1234

Response: 
<p class=is-warning>Current password is incorrect</p>
{% endhighlight %}

If we change the `username` to `carlos` with mismatched new passwords, our response is different. This may have proved our theory that the change password function is dependant on the credentials. 

So as we can see, we might be capable of changing the password of other users. It is just that in this case, we don't have the necessary credentials to be able to do so. If we did, we would get the first response when we supplied the wiener:peter credentials. So what does this mean?

There's a possibility that there are two components working together on the same script rather than it being an all-in-one function. Those would be:
- Checking if the supplied username and password are correct
- Checking if the first new password and second new password match

And so we might be able to access the endpoint that changes the password if we don't provide the data that may not matter. Let's test it with our account first.

It wouldn't make sense to get rid of the `username` parameter and its value because it would not know which user to change the password for. So here I got rid of `current-password` and its value:
![](/assets/portswigger/logic-flaw/burp.PNG)
>Ridding of the current-password parameter

And we get a message unlike the others we found:
![](/assets/portswigger/logic-flaw/burp2.PNG)

Just to make sure that this did indeed change our password, I logged back in with our new credentials and get a redirect to our account page:
![](/assets/portswigger/logic-flaw/burp3.PNG)
>It works!

## Exploitation
Now that we've found a way to change our password without having to provide the right credentials, all we need to do is change the `administrator` password. 

We'll modify `username`'s value to `administrator` and get rid of the `current-password` parameter:
![](/assets/portswigger/logic-flaw/burp4.PNG)

Follow the redirect after logging in with our new credentials:
![](/assets/portswigger/logic-flaw/burp5.PNG)
![](/assets/portswigger/logic-flaw/admin.PNG)

All that's left is to access the admin panel and delete the user `carlos`:
![](/assets/portswigger/logic-flaw/solve.PNG)

# Conclusion
Secure coding is a very important practice when developing applications or features that are soon to be in production. That is why we are seeing more security practices and development models being enforced during different stages of development as one mishap can cause a big impact on the company.
