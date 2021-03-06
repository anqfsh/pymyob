# PyMYOB
A Python API around [MYOB's AccountRight API](http://developer.myob.com/api/accountright/api-overview/).

This code is based off [PyXero](https://github.com/freakboy3742/pyxero) and [PyWorkflowMax](https://github.com/ABASystems/pyworkflowmax), providing pythonic access to the MYOB api in a similar fashion.

It's not yet a fully fleshed out ORM, but rather a collection of namespaced functions that link up to MYOB's endpoints, but the plan is to eventually move it in that direction.

## Pre-getting started

Register for API Keys with MYOB. You'll find a detailed instructions [here](http://developer.myob.com/api/accountright/api-overview/getting-started/).

## Getting started

Install:
```
pip install pymyob
```

Create a `PartnerCredentials` instance and provide the Key, Secret and Redirect Uri as you've set up in MYOB:
```
from myob.credentials import PartnerCredentials

cred = PartnerCredentials(
    consumer_key=<Key>,
    consumer_secret=<Secret>,
    callback_uri=<Redirect Uri>,
)
```

Cache `cred.state` somewhere. You'll use this to rebuild the `PartnerCredentials` instance later.
Redirect the user to `cred.url`. There, they will need to log in to MYOB and authorise partnership with your app. Once they do, they'll be redirected to the Redirect Uri you supplied.

At the url they're redirected to, rebuild the `PartnerCredentials` then pick the verifier out of the request and use it to verify the credentials.
```
from myob.credentials import PartnerCredentials

def myob_authorisation_complete_view(request):
    verifier = request.GET.get('code', None)
    if verifier:
        state = <cached_state_from_earlier>
        if state:
            cred = PartnerCredentials(**state)
            cred.verify(verifier)
            if cred.verified:
                messages.success(request, 'OAuth verification successful.')
            else:
                messages.error(request, 'OAuth verification failed: verifier invalid.')
        else:
            messages.error(request, 'OAuth verification failed: nothing to verify.')
    else:
        messages.error(request, 'OAuth verification failed: no verifier received.')
```

Save `cred.state` once more, but this time you want it in persistent storage. So plonk it somewhere in your database.

With your application partnered with MYOB, you're now good to go. Create a `Myob` instance, supplying the verified credentials:
```
from myob import Myob
from myob.credentials import PartnerCredentials

cred = PartnerCredentials(**<persistently_saved_state_from_verified_credentials)
myob = Myob(cred)
```

Access stuff:
```
# Obtain list of company files. Here you will also find their IDs, which are required for other endpoints
company_files = myob.companyfiles.all()

# Obtain a list of customers (two ways to go about this).
customers = myob.contacts.all(company_id=<company_id>, Type='Customer')
customers = myob.contacts.customer(company_id=<company_id>)

# Obtain a list of sale invoices (two ways to go about this).
invoices = myob.invoices.all(company_id=<company_id>, InvoiceType='Item', orderby='Number desc')
invoices = myob.invoices.item(company_id=<company_id>, orderby='Number desc')

# Create an invoice.
myob.invoices.post_item(company_id=<company_id>, data=data)

# Obtain a specific invoice.
invoice = myob.invoices.get_item(company_id=<company_id>, uid=<invoice_uid>)

# Download PDF for a specific invoice.
invoice_pdf = myob.invoices.get_item(company_id=<company_id>, uid=<invoice_uid>, headers={'Accept': 'application/pdf'})

# Obtain a list of tax codes.
taxcodes = myob.general_ledger.taxcode(company_id=<company_id>)

# Obtain a list of inventory items.
inventory = myob.inventory.item(company_id=<company_id>)
```

If you don't know what you're looking for:

- the repr of a `Myob` instance will yield a list of available managers (i.e. `invoices` in the above example).
- the repr of a `Manager` instance will yield a list of available methods on that manager. Each method corresponds to an API call in MYOB.


## Development

This project is still a baby. It has no tests, no documentation aside from this README, limited endpoint coverage, and supports Python 3 only.

The plan is to ORM-ify it more, so object representations can be returned rather than raw JSON data. Would also be nice not having to pass `company_id` in everywhere.

Contributions are welcome. ;)
