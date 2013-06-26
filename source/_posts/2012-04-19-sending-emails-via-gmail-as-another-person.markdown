---
layout: post
title: "Sending emails via Gmail as \"another person\""
date: 2012-04-19 18:52
comments: true
categories:
---

Recently, I needed to find a way to send emails from a web-based application so
that it seemed, to the recipient, as if the user initiating the action was
doing the sending (as opposed to the application). Gmail, however, prevents
spoofing of emails: If you set the "from" field to an email address that
differs from the actual email address of the account you are using to send,
Gmail automatically resets the "from" field to the account email address.

What you can do, however, is tell Gmail what the name *should* be: "Bob
Smith" &lt;app-email-account@example.com&gt;. Then you can also set the
reply-to to the user's actual email address: bob.smith@example.com. Now when an
email sent with a from/reply-to as described above arrives in my inbox, I see
that I've received an email from Bob Smith (from the account
app-email-account@example.com, but only if I look at the details). When I hit
reply, the "to" field of my email is set to bob.smith@example.com.

To clarify, the from/reply-to fields should look as follows:

```
From:     "Bob Smith" <app-email-account@example.com>
Reply-To: bob.smith@example.com
```
