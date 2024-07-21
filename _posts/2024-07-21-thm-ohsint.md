---
title: "TryHackMe: OhSINT"
date: 2024-07-21
categories:
  - blog
tags:
  - tryhackme
  - OSINT
---

OSINT, or Open-Source Intelligence, is the practice of collecting and analyzing information from publicly available sources to produce actionable intelligence. This can include data from:

- Social media (e.g., Facebook, Twitter, Instagram)
- Online publications and blogs
- Government reports and public records
- Academic papers and professional journals
- Commercial data and financial reports
- Grey literature (e.g., technical reports, patents, business documents)

OSINT is widely used in fields like national security, law enforcement, and cybersecurity to gather insights without needing covert methods.

The challenge description: 

Are you able to use open source intelligence to solve this challenge?
What information can you possible get with just one image file?

Our goal is to answer the following questions:

- What is this user's avatar of?
- What city is this person in?
- What is the SSID of the WAP he connected to?
- What is his personal email address?
- What site did you find his email address on?
- Where has he gone on holiday?
- What is the person's password?

Retrieve a copy of the original image file can be retrieved from the following location if you would like to follow along: https://tryhackme.com/r/room/ohsint

In addition to the original image, you would also need to have access to the "exiftool" application, a reliable search engine, and an account on WiGLE (https://www.wigle.net).

Below is a screenshot of the image we are presented with to begin our OSINT investigation.

We can see that there's not much information visible within the image (no significant landmarks, people, logos, or text). We'll have to rely on any metadata attached to the image.

![](/assets/img/thm/ohsint/WindowsXP_1551719014755.jpg)

We'll use the "exiftool" application to examine the metadata associated with the image.

```terminal
exiftool WindowsXP_1551719014755.jpg
```

![](/assets/img/thm/ohsint/OhSINT-exiftool.png)

We find some information that may be of use. Let's focus on the following information:

```
Copyright                       : OWoodflint
GPS Position                    : 54 deg 17' 41.27" N, 2 deg 15' 1.33" W
```

The "Copyright" information may be a clue as to who the photographer was. Let's fire up a search engine and search for "OWoodflint".

***NOTE: I have added "-TryHackMe" and "-CTF" to the search field to instruct the search engine to provide less of those results. Others have posted write-ups regarding this challenge and we don't want to see those links or references to TryHackMe in our results.***

The first result is for a twitter account with the exact same spelling as our copyright information. Let's have a look at the page.

![](/assets/img/thm/ohsint/OhSINT-search.png)

We can see a description that indicates some interests of this user: "I like taking photos and open source projects."

This profile also provides the answer to our first question: 

What is this user's avatar of?  

```
cat
```

![](/assets/img/thm/ohsint/OhSINT-twitter-1.png)

An additional clue regarding a possible BSSID used by our target is left as a tweet. They indicate the WiFi is accessible from their house. If we can determine where the WiFi is located we can get the location of where our target lives.  

Bssid: B4:5D:50:AA:86:41

We'll use the WiGLE website to see if any information on the BSSID is available. WiGLE is now requiring a user account to perform searches on the website, so if you are following along and don't already have an account, you will need to create one. 

![](/assets/img/thm/ohsint/OhSINT-wigle.png)

Once logged in, select "Advanced Search" from the "View" menu.

![](/assets/img/thm/ohsint/TryHackMe - OhSINT-wigle.png)

Enter the BSSID we retrieved from the tweet and click query.

![](/assets/img/thm/ohsint/TryHackMe - OhSINT-wigle-1.png)

We find one result for the BSSID. Let's click on the "map" link to see if we can determine the location of our target.

![](/assets/img/thm/ohsint/TryHackMe - OhSINT-wigle-2.png)

The BSSID is showing as being located in London. We have the answer to our next question:

What city is this person in?  

```
London
```

Heading back to our search result, we can also find the answer to the question regarding the SSID of the WAP:

![](/assets/img/thm/ohsint/TryHackMe - OhSINT-wigle-3.png)

What is the SSID of the WAP he connected to?  

```
UnileverWiFi
```

We've found our targets location as well as the SSID of the wireless access point they are connected on. Let's head back to the search results and review the GitHub page that was returned.

Here we find the answers to the next two questions:

What is his personal email address?  

```
OWoodflint@gmail.com
```

![](/assets/img/thm/ohsint/OhSINT-email.png)

What site did you find his email address on?

```
Github
```

The GitHub page links to a WordPress website that does not show in our initial search engine results. Let's move over to that site to see if there's any information to help us solve the last two questions.

The site hosts Oliver's blog. A blog entry appears on the main page stating "Im in New York right now, so I will update this site right away with new photos!"

Looks like we can now answer where Oliver went on vacation/holiday.

![](/assets/img/thm/ohsint/OhSINT-vacation.png)

Where has he gone on holiday?  

```
New York
```

Moving on to the final question, we haven't located any data that resembled a password on the twitter or GitHub pages. Taking a look at the Home and Contact pages of the WordPress site, there doesn't appear to be any password displaying on either page.

Let's take a peek at the code running on the WordPress site to see if a password may be hidden there in comments.

On the Home page we locate an entry where the text color has been set to white. The text will display on the page, but as the background of the page is white, the text appears to be hidden.

![](/assets/img/thm/ohsint/OhSINTpassword.png)

Heading back over to the rendered page, we select all text and see the password listed. We're now ready to answer the final question of the challenge.

![](/assets/img/thm/ohsint/OhSINT-password.png)

What is the person's password?

```
pennYDr0pper.!
```

This basic OSINT challenge highlights the surprising amount of information that can be uncovered from what seems like an innocuous image.

On to the next challenge!