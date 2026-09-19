# Regex for Hackers



In this blog, I won't teach you how to write regular expressions from scratch. There are already plenty of great tutorials and resources online for learning regex syntax.

Instead, this blog is about **how to use regex as a bug hunter and penetration tester**.

When you're hunting for vulnerabilities, you often have to deal with a huge amount of data: thousands of URLs, JavaScript files, API responses, parameters, headers, and configuration files. Manually searching through all of this data is slow and inefficient.

Regex can help you turn that huge amount of data into something much more useful.

This blog will focus on two main areas.

First, I'll talk about how you can use regex to find bugs. In this section, I'll focus on using regex to discover hidden endpoints and hidden assets.

Then, I'll talk about how to find bugs caused by developers incorrectly implementing regex.



### Use Regex to Find Bugs

#### github

As you know, GitHub supports regex searches. When I first discovered this, I deleted almost all of the GitHub cheat-sheet keywords I had saved 😂, because regex allows you to extract exactly what you need from a huge amount of data.

In this write-up, I will use **Facebook as the target**.

For example, if you want to find subdomains under `facebook.com`, instead of searching for `facebook` and then manually looking through thousands of results for subdomains, you can use a simple regex to extract exactly what you need.

This regex can help you find subdomains and endpoints under Facebook domains:

```bash
/https?:\/\/[a-z0-9\.-]+\.facebook\.com\/([a-z0-9-_]+\/)*v[0-9]+\/
```

When you combine a regex with a normal GitHub search operator, you can make the search much more powerful. For example:

```bash
/https?:\/\/[a-z0-9\.-]+\.facebook\.com\/([a-z0-9-_]+\/)*v[0-9]+\/ AND NOT "WWW"
```

<figure><img src=".gitbook/assets/image (14).png" alt=""><figcaption></figcaption></figure>

Another useful regex I use is:



```bash
# Find subdomains containing "api"
/https?:\/\/([a-z0-9-]{1,}[\.])*api\.([a-z0-9-]{1,}[\.])*facebook\.[a-z\.]+\//
# Find subdomains that have an API path
/https?:\/\/[a-z0-9\.-]+\.facebook\.[a-z]+\/([a-z0-9-_]+\/)*api\//
# Find Facebook URLs containing /v*/ patterns, such as /api/v1/users
/https?:\/\/[a-z0-9\.-]+\.facebook\.[a-z]+\/([a-z0-9-_]+\/)*v[0-9]+\//
# Find GraphQL endpoints
/https?:\/\/[a-z0-9\.-]+\.facebook\.[a-z]+\/([a-z0-9-_]+\/)*graphql\//
## find all email with password
/[a-zA-Z0-9._%+-]+@target\.com/ AND "password"
```

After understanding the concept, you can start creating your own regex-based GitHub searches for different use cases. With AI tools helping you write and modify regex, creating these searches becomes much easier.

**How I Found a Bug Using This Technique**

During recon, I found a hidden subdomain where I could log in as one of the target's team members.

I went to the **Forgot Password** section and entered my email address. The application responded:

> We sent an OTP to your email. Please enter the OTP.

Because my email was not registered, I didn't receive any OTP.

I tried entering an incorrect OTP. The application returned **"Wrong OTP"**, but when I looked at the response, I noticed:

```json
"status": false
```

I tried changing it to:

```json
"status": true
```

The error then changed to **"Email not found."**

This showed me that response manipulation was possible, but I still needed a valid email address.

I went back to GitHub and searched for email addresses belonging to the target using this regex:

```json
/[a-zA-Z0-9._%+-]+\@target\.com/
```

I extracted the emails I found and tested them using Burp Intruder with **Match and Replace**.

One of the email addresses worked, and I was redirected to the dashboard.

#### **regex with grep**​ <a href="#id-6cdb" id="id-6cdb"></a>

As you know, you can use regex with `grep` using the `-E` or `-P` options.​

Here is a simple example of using `grep` to extract endpoints from JavaScript files. First, download JavaScript files from the target. You can find them by crawling with a tool like `katana`, or through passive discovery with tools like `waymore`.​ Then:​

```bash
cat katana.txt | grep -E "\.js" | fff -S -c 20 ## Download all JS files​​
```

#### Grep endpoints <a href="#d03d" id="d03d"></a>



```bash
gf js-endpoints-without-line |gf blacklist |inscope | sed 's|"||g'​​
```

The `gf` patterns use grep internally. For example, this is the `js-endpoints-without-line` pattern:

```bash
cat ~/.gf/js-endpoints-without-line.json​​​
{
​
"flags": "-ERhoi",
​
"pattern": "(\"|'|`)(https?://[^\"`']+|/[a-zA-Z0–9_?&=/\\#~.:-]+)(\"|'|`)"
​
}​
```

And this is the `blacklist` pattern I use to remove common static files:

```bash
cat ~/.gf/blacklist.json​
{
​
"flags": "-IEiv",
​
"pattern": "\\.(jpeg|jpg|png|svg|gif|woff|woff2|svg|eot|tif|tiff|ttf|ico|icon|txt|pdf|css|js)$"
​
}​
```

<figure><img src=".gitbook/assets/image (15).png" alt=""><figcaption></figcaption></figure>

### Find Regex to Find Bugs​ <a href="#id-1007" id="id-1007"></a>

Some developers write incorrect regex because the syntax can be tricky. Sometimes, as hackers, we can take advantage of these mistakes to bypass certain security checks.​ Some people may think this type of bug will disappear soon because AI can now write regex. But I don’t think it will disappear yet.​ Sometimes, AI misunderstands what the developer actually needs and generates a regex for the wrong situation. If the developer blindly accepts the generated code, this can still lead to security issues.​

I found this image on one of the blogs I read. I think it really captures what I mean here 😅

<br>

<figure><img src=".gitbook/assets/image (16).png" alt=""><figcaption></figcaption></figure>

​Let’s look at a real-world example of how an incorrectly implemented regex can lead to a security vulnerability. For example, consider this code:​

```javascript
window.addEventListener("message", function(event) { 
    if (/https:\/\/www\.example\.com/.test(event.origin)) { 
        if (event.data.type === "error") { 
            div.innerHTML(event.data.error.message); 
        } 
    } 
});​​
```

The developer is trying to allow messages only from `https://www.example.com`, but the regex is not anchored.​ An attacker can register a domain such as:​

```bash
https://www.example.com.attacker.com​​
```

and host an HTML payload there.​ Because the regex only checks whether `https://www.example.com` appears in the origin, the attacker's domain can pass the check.

Since the application also places `event.data.error.message` directly into `innerHTML`, this can potentially lead to XSS.
