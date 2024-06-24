# python-Paybox

Simple Python class to help you with the integration of the Paybox payment system (Paybox system, not Paybox direct)

## Requirements

Only the standard library if you intend not to verify the authenticity of the Paybox response via its public key

pycryptodome otherwise (recommended)

   `sudo pip install pycryptodome`

## Usage

Complete the **settings_example.py** file and rename it **settings.py**

Calling Paybox from a Django view
```python
    def paybox_call(request, order_reference):
     from Paybox import Transaction
     
     # Your order object
     order = get_object_or_404(Order, reference=order_reference)
	
     transaction = Transaction(production=False,
                              firstname=order.firstname,
                              lastname=order.lastname,
                              address1=order.address,
                              city=order.city,
                              zipcode=order.zipcode,
                              countrycode=order.countrycode, # 250 for France
                              PBX_TOTAL=order.total, 	# total of the transaction, in cents (10€ == 1000) (int)
                              PBX_PORTEUR=order.email,  # customer's email address
                              PBX_TIME=order.date.isoformat(),	 # datetime object
                              PBX_CMD=order.reference),  # order_reference (str),
                              PBX_DEVISE=978,
                              PBX_REFUSE='http://127.0.0.1:8000/polls/up2pay',
                              PBX_REPONDRE_A='http://127.0.0.1:8000/polls/up2pay',
                              PBX_EFFECTUE='http://127.0.0.1:8000/polls/up2pay',
                              PBX_ANNULE='http://127.0.0.1:8000/polls/up2pay',
                              PBX_ATTENTE='http://127.0.0.1:8000/polls/up2pay',
                              )

    form = transaction.construct_html_form()
    return render(request, 'polls/payment.html',
                  {'action': form['action'],
                   'html': form['html'],
                   })
```
How to organise the variables in a Django template
```html
<form action="{{action}}" id="payboxform" method="POST">
    {{html|safe}}
    <input type="submit" value="Send payment"/>
</form>
```
For using in with a dynamic form building in javascript  
```python
    def paybox_call(request, order_reference):
     from Paybox import Transaction
     
     # Your order object
     order = get_object_or_404(Order, reference=order_reference)
	
     transaction = Transaction(production=False,
                              firstname=order.firstname,
                              lastname=order.lastname,
                              address1=order.address,
                              city=order.city,
                              zipcode=order.zipcode,
                              countrycode=order.countrycode, # 250 for France
                              PBX_TOTAL=order.total, 	# total of the transaction, in cents (10€ == 1000) (int)
                              PBX_PORTEUR=order.email,  # customer's email address
                              PBX_TIME=order.date.isoformat(),	 # datetime object
                              PBX_CMD=order.reference),  # order_reference (str),
                              PBX_DEVISE=978,
                              PBX_REFUSE='http://127.0.0.1:8000/polls/up2pay',
                              PBX_REPONDRE_A='http://127.0.0.1:8000/polls/up2pay',
                              PBX_EFFECTUE='http://127.0.0.1:8000/polls/up2pay',
                              PBX_ANNULE='http://127.0.0.1:8000/polls/up2pay',
                              PBX_ATTENTE='http://127.0.0.1:8000/polls/up2pay',
                              )

    return render(request, 'polls/payment.html',
                  {'action': transaction.get_action(),
                   'values': transaction.get_html_elements(),
                   })
```
and the corresponding template:
```html
<h1>Javascript rendering: </h1>
<form action="{{action}}" id="payboxform" method="POST">
</form>
<button type="submit" onclick="document.getElementById('payboxform').submit()">PAYER</button>

<script>
    const formElements = {{ values|safe }};
    const payboxForm = document.getElementById('payboxform');
    console.log(formElements);
    function populateForm(parentFormElement, payboxFormElements) {
        payboxFormElements.map((e) => {
            const _inputElement = document.createElement('input');
            _inputElement.type = "hidden";
            _inputElement.name = e.name;
            _inputElement.value = e.value;
            parentFormElement.appendChild(_inputElement)
        })
    }
    populateForm(payboxForm,formElements);
</script>
```
Receiving an Instant Payment Notification in a Django view

    def ipn(request):
     from Paybox import Transaction
     
     # Your order object
     order = get_object_or_404(Order, reference=request.GET.get('RE'))
	
     transaction = Transaction()
     notification = transaction.verify_notification(response_url=request.get_full_path(), order_total=order.total)
	
     order.payment = notification['success']	  	 # Boolean
     order.payment_status = notification['status']   	 # Paybox Status Message
     order.payment_auth_code = notification['auth_code'] # Authorization Code returned by Payment Center
     order.save()
    
     # Paybox Requires a blank 200 response
     return HttpResponse('')

## Understanding the Paybox Flow

1. The customer comes to your website and add an item to his basket

2. He creates an account and validates the purchase (or not but you may validate his email address before sending it to Paybox)

3. The purchase is stored in your database, payment set to False (or whatever column you prefer)

4. Your server constructs an html page which redirects the customer to the Paybox website via an hidden form (variables are sent in POST, and you may trigger the redirection automatically via javascript)
	
> post_to_paybox()

> construct_html_form()

5. Your customer pays, and Paybox send a confirmation to your server. You verify the authenticity of the Paybox callback request, set payment to True and send your customer a thank you email.

To set your IPN url you have to contact customer support. Your ipn url MUST NOT trigger any sort of redirection. Beware of 301 between http:// and http://www.

> verify_notification(response, order_reference, order_total, verify_certificate=True)

> verify_certificate(message, signature)

6. The customer is redirected by the Paybox server on a confirmation page on your server. You may configure this in the Paybox admin. Paybox returns a few variables. Don't do any sort of validation on those urls.

## Paybox installation

### Mandatory variables to be sent by the customer's browser with the payment request :

	PBX_SITE (given by Paybox)
	PBX_RANG (Paybox gives you a three digit number, take the last two)
	PBX_IDENTIFIANT (given by Paybox)
	PBX_TOTAL (in cents. 10$ = 1000)
	PBX_DEVISE (number, 978 for €)
	PBX_PORTEUR (email address of the customer)
	PBX_RETOUR (list of variables that paybox will return with the payment confirmation)
	PBX_HASH (How you've keyed-hashed your message. SHA512 in this app)
	PBX_TIME (ISO 8601 Format)
    PBX_BILLING
    PBX_SHOPPINGCART
	PBX_HMAC 

### How to send a valid payment call to Paybox

1. Connect to http://preprod-admin.paybox.com to generate a valid secret key. Copy it in the settings of your app or any place SAFE. When in prod, generate a new secret key by going to http://admin.paybox.com

2. In the admin interface, define your redirection urls, and contact the support to set your ipn url

3. Connect everything via your router or anything you're using.

4. In case of trouble, verify that all your mandatory variables are valid. Especially SITE, RANG, IDENTIFIANT. The app constructs a valid redirection form if all your variables are ok.

5. With corrects values for all the variables, the method **post_to_paybox()** constructs a HMAC that ensure the variables are transmitted unaltered to the Paybox server.

## Methods

### post_to_paybox()

returns two dicts :

- mandatory, which contains all the mandatory variables unordered
- accessory, which contains all the other variables you may send to Paybox

### construct_html_form()

returns a string, which is a valid html form ready to be put inside a template for example

### verify_notification(response, order_reference, order_total, verify_certificate=True)

returns three strings in a dict :

- success, True if the payment is successful
- status, Paybox status message
- auth_code, The Authorization Number sent by the Payment Center

### verify_certificate(message, signature)

return True if everything is ok, and horrible errors if the verification fails. This is automatically called by the *verify_notification* method, unless you decide otherwise.
