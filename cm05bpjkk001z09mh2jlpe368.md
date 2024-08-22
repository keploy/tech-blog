---
title: "How RB2B is exploiting LiveIntent to website visitor Personal Identify"
datePublished: Thu Aug 22 2024 13:33:28 GMT+0000 (Coordinated Universal Time)
cuid: cm05bpjkk001z09mh2jlpe368
slug: how-rb2b-is-exploiting-liveintent-to-website-visitor-personal-identify

---

I recently came across surging popularity of rb2b.com due to their claim of providing personal information (like name, email, Linkedin profile url, etc) of visitors to website owners.

![](https://cdn.prod.website-files.com/65a6cc745ee423cf5c0fb003/65c0faa21c6a88ae232d1cb2_slack_full_mockup.png align="left")

# How?

It caught my attention and so I decided to try it and also read through [the SDK](https://s3-us-west-2.amazonaws.com/b2bjsstore/b/LNKLDHM9D0OJ/reb2b.js.gz). I noticed this method:  

```javascript
B2BRetention.prototype.trigger_li = function () {
    window.liQ = window.liQ || [];
    window.liQ.push({config: {sync: false, identityResolutionConfig: {publisherId: 72731}}})
    let scriptTag = document.createElement('script');
    
    // LiveIntent SDK to fetch user hash
    scriptTag.src = 'https://b-code.liadm.com/lc2.js';
    scriptTag['onload'] = function () {_reb2b.fetch_li_id();}
    document.head.appendChild(scriptTag);
}
```

This seems to using [LiveIntent](https://support.liveintent.com/hc/en-us/articles/212078486-Dynamic-Product-Ads-Content-View-Implementation) JS SDK to get a hash that uniquely identifies the user. LiveIntent supposedly helps identify users via [MD5 or SHA1 hashes](https://support.liveintent.com/hc/en-us/articles/207560146-What-is-an-Email-Hash). These can then be mapped back to the email address if you have table of hash -&gt; email addresses or other unique identifiers.

# Is this a vulnerability?

I think it is!

I think this method of user tracking by LiveIntent can be easily exploited by other actors by a simple dictionary attack. I think the rb2b team is sending these hashes back and then trying to compare these hashes to table of email hashes or other unique identifiers like linkedin ids.

So If anyone ever used LiveIntent, your email hashes could be the same across different companies and you could be easily tracked via a simple dictionary lookup.

# Where do we go from here?

I'm not fully aware of how LiveIntent works, but it would be great if they can share some more insights about personal data security and potential fix these issues. Else it could be used for malicious purposes (other than just trying to spam website visitors 😛)