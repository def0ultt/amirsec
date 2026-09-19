# Google Dorking for Bug Hunters: Real-World Case Studies

In this writeup, I will **not teach you how to write Google Dorks**. There are already plenty of resources that explain Google search operators and how to build different dorks. Instead, I will focus on **how to use Google Dorking to find real security vulnerabilities**.

Most hackers start with something like:

```
site:example.com
```

Then they start filtering out subdomains:

```
site:example.com -www -status
```

This is useful for discovering new assets and expanding your attack surface. However, these are common techniques that almost every hacker knows.The problem is that this type of dorking usually helps you **find assets, not vulnerabilities**.

Finding a new subdomain is useful, but the real question is:

> _**What can you find on that asset that could lead to a vulnerability?**_

That is what this series will focus on: **real case studies where Google Dorking helped discover security bugs**, rather than simply collecting subdomains or exposed assets.

### Google Dorks from Real Reports <a href="#ba7c" id="ba7c"></a>

**Case Study 1**

Disclosure of DoD training PowerPoints leaking plaintext credentials, found via:

```
site:*.mil ext:ppt intext:password
```

Report URL: [https://hackerone.com/reports/672629](https://hackerone.com/reports/672629)

**Case Study 2**

Disclosure of a confidential PII PDF found via a semantic dork targeting classification keywords.

Google Dork used:

```
inurl:target.com "not for distribution" | confidential | "employee only" | proprietary | "top secret" | classified | "trade secret" | internal | private filetype:xls OR filetype:csv OR filetype:doc OR filetype:pdf OR filetype:txt
```

Report URL: [https://bugcrowd.com/disclosures/ab094307-76cf-4235-876f-ee34b306ed30/google-dork-leaded-information-confidential-in-pdf](https://bugcrowd.com/disclosures/ab094307-76cf-4235-876f-ee34b306ed30/google-dork-leaded-information-confidential-in-pdf)

**Case Study 3**

Disclosure of a confidential PII PDF found via a semantic dork targeting classification keywords.

Google Dork used:

```
site:target.org intext:"test_" + intext:"api key"
```

You can use this improved version:

```
site:target.org (intext:"api_key" OR intext:"apikey" OR intext:"api key" OR intext:"secret_key") intext:"test_"
```

Writeup link: [https://medium.com/@cybertechajju/how-i-earned-my-first-50-bug-bounty-with-a-google-dork-and-a-test-key-a3e6290db694](https://medium.com/@cybertechajju/how-i-earned-my-first-50-bug-bounty-with-a-google-dork-and-a-test-key-a3e6290db694)

**Case Study 4**

Mass enumeration of classroom invitation links, where an attacker could join a room without an invitation from the teacher.

Google Dork used:

```
site:khanacademy.org/join/*
```

Report link: [https://hackerone.com/reports/1210043](https://hackerone.com/reports/1210043)

**Case Study 5**

IDOR found via indexed API subdomains, exposing proprietary statistics without authentication.

The researcher found an API endpoint that did not require authentication when using an API key.



Google Dork used:

```
site:*.api.semrush.com
```

Report link: [https://hackerone.com/reports/284963](https://hackerone.com/reports/284963)

**Case Study 6**

A writeup about bypassing a 403 restriction using a basic Google Dork to find an endpoint that could not be found through brute-force enumeration.

Google Dork used:

```
site:nnnnnhelpdesk.redacted.com
```

Writeup link: [https://medium.com/@mrx\_w\_/how-i-discovered-23-000-leaked-records-through-google-dorking-7894df815109](https://medium.com/@mrx_w_/how-i-discovered-23-000-leaked-records-through-google-dorking-7894df815109)

**Case Study 7**

Account takeover via indexed email-confirmation tokens and a token-validation logic flaw.

This report was not accepted because only one user’s token was indexed, and that user was no longer active. However, we can still take the idea from the dork.

Google Dork used:

```
site:sorare.com inurl:token
```

Report link: [https://hackerone.com/reports/1817214](https://hackerone.com/reports/1817214)

**Case Study 8**

A hidden comment-form endpoint discovered through `inurl:blog/`, which led to further parameter testing.

Google Dork used:

```
site:www.starbucks.co.uk inurl:blog/
```

Writeup link: [https://hackerone.com/reports/218226](https://hackerone.com/reports/218226)

### Another Dorking I Use <a href="#id-7da3" id="id-7da3"></a>

Here are some other Google Dorks and tips that I use during reconnaissance.

**JHaddix’s Bug Bounty Tip**

One of the best tips from JHaddix is to search for a unique piece of text from a website’s footer:

```
intext:"Copyright © 2014 – 2026 Bugcrowd, Inc"
```

This can help identify pages and assets belonging to the target when the same copyright text is used across different parts of the website.

**Finding Sensitive Files**

A useful tip from LostSec for finding potentially sensitive files is to search for multiple document and configuration file types at once:

```
site:target.com (filetype:doc OR filetype:docx OR filetype:pdf OR filetype:rtf OR filetype:ppt OR filetype:pptx OR filetype:csv OR filetype:xls OR filetype:xlsx OR filetype:txt OR filetype:xml OR filetype:json OR filetype:zip OR filetype:rar OR filetype:log OR filetype:bak OR filetype:conf OR filetype:sql)
```

The idea is to search the target for different types of files that may contain useful or sensitive information.

<figure><img src=".gitbook/assets/image (21).png" alt=""><figcaption></figcaption></figure>

**Finding Contact Forms for Blind XSS**

Another useful dork is for finding support-team contact forms that can be tested for Blind XSS:

```
site:target.com intitle:"contact us" | intitle:"get in touch" | intitle:"contact form" | intitle:"reach us" | intitle:"reach out" | intitle:"talk to us" | intitle:"contact our team"
```

<figure><img src=".gitbook/assets/image (22).png" alt=""><figcaption></figcaption></figure>

\
This searches for different common titles used by contact and support pages, making it easier to find different forms across the target.

### Automation <a href="#id-6795" id="id-6795"></a>

If you are targeting a very large website and don’t want to go through hundreds of PDF and XLSX files manually, you can automate the process. You can use [webpaste](https://github.com/xnl-h4ck3r/webpaste) to save the Google search results and URLs. Then, you can pass the collected URLs to an LLM for analysis. This makes the process faster and more efficient, especially when you have a large number of results to review.

<figure><img src=".gitbook/assets/image (23).png" alt=""><figcaption></figcaption></figure>
