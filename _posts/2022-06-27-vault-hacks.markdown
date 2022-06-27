---
layout: post
title:  "Simulating --dry-run and --patch in Hashicorp Vault"
date:   2022-06-27 15:32:31 +0100
categories: [vault]
---
Hashicorp [Vault](https://www.hashicorp.com/products/vault) is a great resource for keeping secrets secret. However version 1 of the [KV secrets engine](https://www.vaultproject.io/api-docs/secret/kv/kv-v1) lacked the ability to patch a particular endpoint which made modifying existing values fraught with danger. We mitigate this by utilising the ever wonderful [jq](https://stedolan.github.io/jq/) alongside a little used Vault flag to perform inline modifcations before similuating writes.

Let's write some test data and read it back again in sequentially more useful ways:

{% highlight bash %}

vault write /secret/test/smchm testkey=testvalue
Success! Data written to: secret/test/smchm

vault read /secret/test/smchm
Key                 Value
---                 -----
refresh_interval    768h
testkey             testvalue

vault read --format=json /secret/test/smchm | jq .
{
  "request_id": "087490c9-ccad-e225-cd17-9f88df12abd5",
  "lease_id": "",
  "lease_duration": 2764800,
  "renewable": false,
  "data": {
    "testkey": "testvalue"
  },
  "warnings": null
}

vault read --format=json /secret/test/smchm | jq .data
{
  "testkey": "testvalue"
}

{% endhighlight %}

Now lets *patch* the data with `jq`, the resultant patched data is not yet stored in Vault:

{% highlight bash %}

vault read --format=json /secret/test/smchm | jq '.data += { "testkey": "newvalue" }' | jq .data
{
  "testkey": "newvalue"
}

{% endhighlight %}

We can also add values:

{% highlight bash %}

vault read --format=json /secret/test/smchm | jq '.data += { "testkey": "newvalue", "testkey2": "testvalue2" }' | jq .data
{
  "testkey": "newvalue",
  "testkey2": "testvalue2"
}

{% endhighlight %}

Now we have the values we want in our JSON object we can write them back the data to the Vault endpoint. However first we can *simulate* the write by performing a dry-run using the Vault flag *--output-curl-string* to make sure everything is as we expect:

{% highlight bash %}

vault read --format=json /secret/test/smchm | jq '.data += { "testkey": "newvalue", "testkey2": "testvalue2" }' | jq .data | vault write -output-curl-string /secret/test/smchm - 
curl -X PUT -H "X-Vault-Token: $(vault print token)" -d '{"testkey":"newvalue","testkey2":"testvalue2"}' https://ourvaulthost:8200/v1/secret/test/smchm

{% endhighlight %}

which can be a little difficult to grok for endpoints with many values, so lets parse the output with *sed* and jq:

{% highlight bash %}

vault read --format=json /secret/test/smchm | jq '.data += { "testkey": "newvalue", "testkey2": "testvalue2" }' | jq .data | vault write -output-curl-string /secret/test/smchm - | sed 's/^.*{/{/' | sed 's/}.*/}/' | jq .
{
  "testkey": "newvalue",
  "testkey2": "testvalue2"
}

{% endhighlight %}

looks good, so finally lets stream our values back to Vault to patch our endpoint and validate the results:

{% highlight bash %}

vault read --format=json /secret/test/smchm | jq '.data += { "testkey": "newvalue", "testkey2": "testvalue2" }' | jq .data | vault write /secret/test/smchm -
Success! Data written to: secret/test/smchm

vault read --format=json /secret/test/smchm | jq .data
{
  "testkey": "newvalue",
  "testkey2": "testvalue2"
}

{% endhighlight %}

Of course this doesn't prevent us writing to the wrong endpoint!
