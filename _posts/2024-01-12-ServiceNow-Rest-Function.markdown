---
layout: post
title:  "ServiceNow Rest Function"
date:   2024-01-11 19:49:21 -0600
tags: [scripting, python]
---
So I am tasked with randomly pulling data out of ServiceNow via the API and then doing small ETL tasks with it. Most of the SDKs are just way to much for a quick dirty need. So I cobbled this together so I have a small, lightweight function that I can easily adjust to random odd needs.

```python
import os
import http.client

def snc_rest(table,sysparm_query=False,sysparm_fields=False,sysparm_exclude_reference_link=True,sysparm_display_value=False):
    SN_AUTH = os.environ["SN_AUTH"]
    SN_INSTANCE = os.environ["SN_INSTANCE"]
    if not SN_AUTH:
        raise ValueError("Environment Variable: SN_AUTH must be set!")
    if not SN_INSTANCE:
        raise ValueError("Environment Variable: SN_INSTANCE must be set!")
    
    sncoffset = 0
    snclimit = 1000
    snctotalreturn = 0

    base_params = "?"
    if sysparm_query and sysparm_fields:
        base_params += "sysparm_query=" + sysparm_query + "&sysparm_fields=" + sysparm_fields
    elif sysparm_query and not sysparm_fields:
        base_params += "sysparm_query=" + sysparm_query
    elif not sysparm_query and sysparm_fields:
        base_params += "sysparm_fields=" + sysparm_fields

    if sysparm_exclude_reference_link:
        base_params += "&sysparm_exclude_reference_link=true"
    else:
        base_params += "&sysparm_exclude_reference_link=false"
    
    if sysparm_display_value:
        base_params += "&sysparm_display_value=true"
    else:
        base_params += "&sysparm_display_value=false"

    conn = http.client.HTTPSConnection(f"{SN_INSTANCE}.service-now.com")

    payload = ""

    headers = {
        'Authorization': SN_AUTH
    }
    returned_dict = []
    while True:
        servicenow_query_params = base_params + "&sysparm_limit=" + str(snclimit) + "&sysparm_offset=" + str(sncoffset)
        servicenow_query = table + servicenow_query_params
        conn.request("GET", servicenow_query, payload, headers)
        res = conn.getresponse()
        resstatus = res.status
        data = res.read()
        resources = json.loads(data.decode("utf-8"))
        for i in resources['result']:
            returned_dict.append(i)
        snctotalreturn = int(res.getheader("X-Total-Count"))

        sncoffset += snclimit
        remainingitems = snctotalreturn - sncoffset
        
        if remainingitems < 0:
            break

    return returned_dict
```

This will assume you have 2 envrionmental variables filled out. `SN_AUTH` which should be your basic http credential encoded already. The other environmental variable needed is `SN_INSTANCE`, which is your subdomain for the ServiceNow SAAS platform.

Example:
```bash
export SN_AUTH="Basic U2VydmljZUFjY291bnQxOldoeURpZFlvdURlY3J5cHRNZT8="
export SN_INSTANCE=foobar
```

And then to call function. We hit the incidents table, searching for all active incidents, returning their `number`, `sys_id` and `state`. Along with that, we exclude the reference link and get the distpay values for the returned fields.
```python
incidents = snc_rest(table='/api/now/table/incidents',sysparm_query='active=true',sysparm_fields='number,sys_id,state',sysparm_exclude_reference_link=True,sysparm_display_value=True)
for incident in incidents:
    print(f"Number: {incident['number']} | State: {incident['state']}")
```

This function isn't fancy, but it serves my need and allows me to be agile.