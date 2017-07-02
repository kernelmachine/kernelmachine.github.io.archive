---
layout: post
title: Have I Been Pwned? with Python
comments: true
---

Massive data breaches are becoming more and more common these days. How can you tell whether you're safe?

The recently launched [Have I Been Pwned](https://haveibeenpwned.com) (HIBP) service can help. HIBP is a free resource to quickly assess if an account or domain has been compromised in a data breach. The project helps victims become aware of these security situations as fast as possible, and highlights the severity of Internet-wide attacks.

To make HIBP accessible to developers, I built a Python wrapper around its API: [hibp](http://github.com/kernelmachine/haveibeenpwned).

With this library you can access a number of HIBP services from Python, including:

* breaches on a particular domain (linkedin.com, ashleymadison.com)
* breaches on a user account or email (kernelmachine, example@gmail.com)
* all historical breaches of a particular name (adobe, linkedin)

Each service request object contains a response attribute that holds the raw data in JSON format. To perform a query, just setup a service request object, and then execute it:

```python
>> from hibp import HIBP
>> req = HIBP.get_account_breaches("kernelmachine")
>> req.execute()
>> req.response
```

You can see a full example script of how to use the API [here](https://github.com/kernelmachine/haveibeenpwned/blob/master/hibp/example.py).

Here's some example data we get back when we query for data breaches on `adobe.com`:

```python
>> import json
>> req = HIBP.get_domain_breaches("adobe.com")
>> req.execute()
>> print(json.dumps(req.response, indent=4, sort_keys=True))
[{
    "AddedDate": "2013-12-04T00:00:00Z",
    "BreachDate": "2013-10-04",
    "DataClasses": [
        "Email addresses",
        "Password hints",
        "Passwords",
        "Usernames"
    ],
    "Description": "In October 2013, 153 million Adobe accounts were breached...",
    "Domain": "adobe.com",
    "IsActive": true,
    "IsRetired": false,
    "IsSensitive": false,
    "IsVerified": true,
    "LogoType": "svg",
    "Name": "Adobe",
    "PwnCount": 152445165,
    "Title": "Adobe"
}]
```

Cool, huh? We can see when the breach occurred, the various types of data that were leaked,  a brief description of the breach, whether it's still active and verified, and how many accounts were affected. You can check [here](https://haveibeenpwned.com/API/v2#BreachModel) for a full description of all the data fields.

If you want to query on multiple accounts or domains at once, you can use the `AsyncHIBP` object, which can perform queries concurrently via [gevent](http://www.gevent.org/).

```python
>> from hibp import AsyncHIBP
>> names = ['adobe','ashleymadison', 'myspace']
>> breaches = [HIBP.get_breach(x) for x in names]
>> async_reqs = AsyncHIBP().map(breaches)
>> [async_req.response for async_req in async_reqs]
```

You can also perform lazy evaluations on multiple accounts or domains with the `imap` method. This will return a generator on request objects, which could be more memory efficient if you're working with a larger list of queries.

```python
>> domains = ['twitter.com','facebook.com', 'myspace.com']
>> breaches = [HIBP.get_domain_breaches(x) for x in domains]
>> async_reqs = AsyncHIBP().imap(breaches)
>> for req in async_reqs:
   ... print req
```

Concurrent queries are much faster than serial ones:

```python
# random set of query parameters
names = ['adobe','ashleymadison', 'linkedin', 'myspace']*20
accounts = ["ssgrn", "kernelmachine","barackobama"]*20
domains = ['twitter.com', 'facebook.com','github.com','adobe.com']*20

# setup HIBP objects for request executions
reqs = [HIBP.get_breach(x) for x in names] \
       + [HIBP.get_account_breaches(x) for x in accounts] \
       + [HIBP.get_domain_breaches(x) for x in domains]

### SERIAL
start_time = time.time()
for req in reqs:
    req.execute()
elapsed_time = time.time() - start_time ### 110.9 SECONDS
logging.info("serial impl took %.2f seconds" % elapsed_time)

### CONCURRENT
start_time = time.time()
async_reqs = AsyncHIBP().map(reqs)
elapsed_time = time.time() - start_time ### 10.01 SECONDS
logging.info("concurrent impl took %.2f seconds" % elapsed_time)
```

There's still work to do here. For example, I should add some functionality to:

* Convert the JSON data to a pandas dataframe or CSV
* Provide some simple helper functions to get users going (e.g. methods like `list_all_breaches()` or `breach.number_pwned()`).

I think this project will be useful for developers who want to build applications that programmatically assess whether accounts or sites are compromised. These types of products could drive faster remediation when large-scale security incidents occur. Also this project will be useful for applications that aim to analyze bulk, historical breach data for security research.

As always, I welcome contributors and feedback!
