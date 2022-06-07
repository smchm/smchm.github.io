---
layout: post
title:  "All your passwords are Troy's"
date:   2022-06-07 08:57:23 +0100
categories: [java, spring, junit5]
---
Our institution is re-opening the age old debate about password hygiene. One of the topics under discussion is whether to introduce expiry conditions to limit password longevity. A [bad idea](https://www.ncsc.gov.uk/collection/passwords/updating-your-approach). Instead store the hashes and periodically check Troy Hunt's excellent database of [compromised credentials](https://haveibeenpwned.com/Passwords) via the [API](https://haveibeenpwned.com/API/v3#PwnedPasswords). The example below generates the hash from the passed argument so not something which can be used in production:

{% highlight bash %}

#!/bin/bash

PASSWD="${1:-password}"
SHAPREFIX="$(echo -n $PASSWD | sha1sum | cut -c1-5)"
SHASUFFIX="$(echo -n $PASSWD | sha1sum | cut -c 6- | cut -f1 -d" ")"
PWNED=$( \
  curl -Ss https://api.pwnedpasswords.com/range/$SHAPREFIX | grep -i $SHASUFFIX \
)
if (test $PWNED)
  then echo "pwned"
  else echo "not pwned"  
fi

{% endhighlight %}

and to use:

{% highlight bash %}

[smchm@home ~]$ have-i-been-pwned.sh 'passwd'
pwned
[smchm@home ~]$ have-i-been-pwned.sh 'not-in-the-database-as-it-is-too-long-but-reasonably-easy-to-remember'
not pwned
[smchm@home ~]$ have-i-been-pwned.sh 'password1234567890'
pwned

{% endhighlight %}

or perhaps more interestingly:

{% highlight bash %}

[smchm@home ~]$ have-i-been-pwned.sh 'password'
pwned
[smchm@home ~]$ have-i-been-pwned.sh 'password123'
pwned
[smchm@home ~]$ have-i-been-pwned.sh 'password123123'
pwned
[smchm@home ~]$ have-i-been-pwned.sh 'password123123123'
pwned
[smchm@home ~]$ have-i-been-pwned.sh 'password123123123123'
pwned
[smchm@home ~]$ have-i-been-pwned.sh 'password123123123123123'
not pwned

{% endhighlight %}
