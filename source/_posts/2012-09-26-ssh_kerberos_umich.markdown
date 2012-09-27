---
layout: post
title: "Setting up a Mac for ssh+Kerberos at the University of Michigan"
date: 2012-09-26 19:36
comments: true
categories:
---

*Note*: this is probably irrelevant if you're not at the University of
Michigan. However, if you're a Mac user there and tired of having to
type in your password to ssh into the shell account servers, read on.

This is what I had to do to get passwordless ssh access working to the
U of M servers (e.g. `login.engin.umich.edu`) from my machine, running
Mac OS X 10.8.2. The university doesn't allow public-key-based ssh
access, so I had to set up Kerberos.

1. Get a copy of the `krb5.conf` file. You can find this at
`/etc/krb5.conf` on any of the Linux shell servers such as
`login.engin.umich.edu`. You'll need to get it to your machine with
scp/sftp/etc.

2. Install it as `/Library/Preferences/edu.mit.Kerberos`. You'll need
root privileges to write to this directory, so:

       $ sudo cp /path/to/krb5.conf /Library/Preferences/edu.mit.Kerberos

3. Open up /Applications/Utilities/Keychain Access.

4. Go to its application menu and select Ticket Viewer.

5. Acquire a Kerberos ticket by choosing Ticket > Add Identity, giving
`<uniqname>@UMICH.EDU` as your identity and your usual U of M password
as your password.

6. Set up ssh to use Kerberos authentication to U of M hosts, by
adding something like the following to your `~/.ssh/config`:

```
Host *.umich.edu
     User <your uniqname goes here>
     Compression yes
     GSSAPIAuthentication yes
     GSSAPIDelegateCredentials yes
     GSSAPITrustDNS yes
```

You should now be able to ssh to a university machine without being
prompted for a password.

