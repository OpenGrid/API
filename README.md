## API

### Test Server
Regular transactional API is located at https://api.inpay.pl
Testnet Server is available at https://api.test-inpay.pl

#### Testnet wallets
If you're willing to use Testnet coins with our Testnet server you should use appriopriate Wallet. We suggest using 
- [Android Bitcoin Wallet Testnet](https://play.google.com/store/apps/details?id=de.schildbach.wallet_test&hl=en)
- [Android Mycelium Testnet](https://play.google.com/store/apps/details?id=com.mycelium.testnetwallet&hl=en)
- http://testnetwallet.com/wallet

#### Testnet faucets
If you need to topup testnet coins use the following faucets or google one:
- http://tpfaucet.appspot.com/
- http://faucet.xeno-genesis.com/
- or just ask us for some testnet coins

#### Testnet explorers
You can monitor testnet transactions on the following sites:
- https://www.blocktrail.com/tBTC

#### Simple PHP app

Following PHP class allows for API interaction:  [phplib InPay](https://github.com/inpay/phplib-inpay)

To generate payment request just issue call to  `/invoice/create`. This method will generate payment URL which can be used to redirect customers to the payment gateway. It is also recommended to supply parameters with `callbackUrl`. This way you'll receive status POST form data notifications to your own service. Alternatively you can poll actively invoice status using `/invoice/status`

### Authentication
To be able to create payment requests you need to set up an account https://partner.inpay.pl and fetch your **API** keys

Attribute | Meaning
------------ | -------------
**apiKey** | Your public API key
**secretApiKey** | Your secret API key used for message signing and verification. Do not share with anyone

### /invoice/create
Method `/invoice/create` accepts following POST **form-data** arguments :

Attribute | Meaning
------------ | -------------
**apiKey** | `string` your public API key
**amount** | `float` amount in local currency `123.25`
**currency** | `string` local currency name. Accepted values: `PLN`, `EUR`, `USD`
**optData** | `string` optional urlencoded query string `key=value&key2=value2`, which will be added to a callback to your service
**orderCode** | `string` optional client order id
**description** | `string` optional description
**customerEmail** | `string` optional email
**callbackUrl** | url optional callback URL to your service which will receive payment status updates
**successUrl** | url optional URL to redirect your client after succesful payment
**failUrl** | url optional URL to redirect your client if payment fails for some reason

Returned **JSON** object

Attribute | Meaning
------------ | -------------
**invoiceCode** | invoice unique ID generated by InPay
**redirectUrl** | you should redirect to this URL

### Initiate payment
Redirect user to `redirectUrl`.

After succesful payment user will be redirected to `succcessUrl` or `failUrl`. 

A `callbackUrl` will receive (POST) status updates :

Attribute | Meaning
------------ | -------------
**orderCode** | your previosly supplied order code 
**amount** | amount in local currency
**currency** | local currency
**fee** | fee taken if applies
**invoiceCode** | unique InPay invoice ID
**status** | current payment status
**optData** | additional query parameters given at `/invoice/create`

`callbackUrl` has a header attribute called `API-Hash`. It is a **SHA512** hash of all above attirbutes combined with your `API private key`.

Below a simple script that checks the signature of transmitted data signed by header attribute `API-Hash`:

```php
  function checkApiHash() {
    $secretApiKey = 'yourPrivateAPIkeySecret';
    $apiHash = $_SERVER['HTTP_API_HASH'];
    $query = http_build_query($_POST);
    $hash = hash_hmac("sha512", $query, $secretApiKey);
    return $apiHash == $hash;
  }
```

### /invoice/status

Use this method if you're willing to poll invoice status yourself. Just POST (form-data) invoiceCode as a parameter. 

Returned JSON object:

Attribute | Meaning
------------ | -------------
**open_date** | invoice creation date (2014-04-15 16:01:01)
**close_date** | invoice full payment date (2014-04-15 16:01:01)
**expire_date** | invoice expiration date (2014-04-15 16:01:01)
**secondsFromOpenToExpire** | expiration date in seconds (1200)
**secondsToExpire** | expiration date in seconds left (1142)
**received_amount** | received amount until now (1234567 Satoshi = 0.00000001 BTC)
**received_currency** | cryptocurrency name (BTC)
**in_amount** | requested amount in local currency * 100 (np. 1000 dla 10 PLN)
**in_currency** | requested local currency name `PLN`, `EUR`, `USD`
**expected_amount** | `integer` requested amount in cryptocurrency
**expected_currency** | cryptocurrency denominator (satoshi)
**input_address** | unique cryptocurrency input (payment) address (mwTNRX6A1E9TNH9JcLqt9eVhJo6zeDsYs7)
**btcAmount** | amount in floating point format, 5 decimal points (0.12345)
**btcLink** | BIP 0021 url with intent, amount and description (bitcoin:mwTNRX6A1E9TNH9JcLqt9eVhJo6zeDsYs7?amount=0.01192708)
**invoice_code** | Unique invoice ID: (PURGAM4BQHN9J3L)
**status** | current status - see below
**confirmations** | current number of confirmations

### Payment QR codes
You can also fetch payment QR code `image/png` using `/qr/invoice/{invoice_code}` GET request
where `{invoice_code}` is a unique invoice code generated during `/invoice/create`

### Payment statuses

Attribute | Meaning
------------ | -------------
**new** | waiting for payment
**received** | payment registered and waiting for confirmation
**confirmed** | payment confirmed by InPay
**paid** | payment included in closed settlement and sent to your bank account
**expired** | payment expired
**invalid** | payment to low, waiting for missing amount
**overpaid** | payment overpaid
**refund** | payment refunded

If you're polling for /invoice/status, check until status  `confirmed`
