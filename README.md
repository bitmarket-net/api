Summary
=======

This document provides information about the BitMarket.net API. 

How to access the API?
----------------------

To make an API call, you need to send a POST request to the following address:
https://www.bitmarket.pl/api2/

The request must include appropriate parameters describing the API method to be called. 
There are two parameters that must be sent woth every request: `method` which selects the API method to be used,
and `tonce` which contains the time parameter and is used to avoid replay attacks.
All the other parameters dopend on the API method used.

### API keys

To call the API functions, it is necessary first to obtain an API key. 
To generate this key, the user must visit our website:
https://www.bitmarket.pl/apikeys.php

Each key contains the following components: 

 * **Public component**  - It is a code that is visible on the key list, and is passed along every API call.
 * **Secret component**  - It is a code that is used to digitally sign the API call parameters and to authenticate the call. 
 It is visible only during key creation.
 * **Permissions** - It is a list of API methods that can be called using this key. 
 It is possible to limit this list, so that he person who possesses this API key can only use specific API methods.
 
The public key component must be passed in the `API-Key` HTTP header with every API call.

### Message signing

The parameters of every API call must be signed to create a HMAC signature. 
The secret key component is used as th einput to the SHA512 method. 
This ensures that even if someone learns the public key component, they will still not be able to send API calls.

The HMAC signature must be passed in the `API-Hash` HTTP header with every API call.

### The tonce parameter

The `tonce` parameter is required with every API call to secure the API from replay attacks.

This parameter must contain the current Unix timestamp when the API is called.
Upon receiving the API call, our server compares it to its own local time (thich is synchronized to Internet time servers using the NTP proocol).
If the difference is greater than 5 seconds, the operation will be rejected. 
This mechanism ensures that even if an attacker was able to snoop the headers of an API call, 
they would not be able to resubmit the command again because the timestamp would not match.

It is important that the time on the computer used to send API calls is also properly synchronized,
to avoid mismatches in the timestamps. 
Proper NTP software, such as the **ntpd** Unix daemon, can be used to ensure this.

### API call limit

The number of API calls that can be made on the user account is limited to 600 calls within 10 minutes. 
This limit is global for the user account, regardless of how many API keys the user is using.

### Server responses

The API server sends responses to API calls in the JSON format. 
If the API call succeeds, the response contains a field named `success` with the value of *true*,
as well as a field named `data` which contains the result of the call.
Another field named `limit` contains information about the current API call limits on the user account, 
with the following fields:

 * `used` - the number of API calls used in the current time interval.
 * `allowed` - the total number of API calls allowed in the current time interval.
 * `expires` - the Unix timestamp indicating when the current time interval elapses.
 
If the API call does not succeed, the returned object contains a field `error` with the error number,
as well as an `errorMsg` field with the textual error description.

### Sample code

This sample PHP code can be used to send API calls to the server:

```php
function bitmarket_api($method, $params = array())
{
    $key = "klucz_jawny";
    $secret = "klucz_tajny";

    $params["method"] = $method;
    $params["tonce"] = time();

    $post = http_build_query($params, "", "&");
    $sign = hash_hmac("sha512", $post, $secret);
    $headers = array(
        "API-Key: " . $key,
        "API-Hash: " . $sign,
    );

    $curl = curl_init();
    curl_setopt($curl, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($curl, CURLOPT_URL, "https://www.bitmarket.pl/api2/");
    curl_setopt($curl, CURLOPT_POST, true);
    curl_setopt($curl, CURLOPT_POSTFIELDS, $post);
    curl_setopt($curl, CURLOPT_HTTPHEADER, $headers);
    $ret = curl_exec($curl);

    return json_decode($ret);
}
```

### Sample return values

Here is a smaple JSON object returned afther a successful `info` method call:

```javascript
{
    "success":true,
    "data":
    {
        "balances":
        {
            "available":
            {
                "PLN":4.166000000000,
                "BTC":0.029140000000,
                "LTC":10.301000000000
            },
            "blocked":
            {
                "PLN":59.4,
                "BTC":0,
                "LTC":0.089
            }
        }
    },
    "limit":
    {
        "used" : 1,
        "allowed" : 200,
        "expires" : 1395750600,
    }
}
```

Here is an example of an error message:

```javascript
{
    "error":502,
    "errorMsg":"Invalid message hash"
}
```

### Error codes

The following error codes can be returned from the application:

Error code | Error description
-----------|------------------------
500 | Invalid HTTP method (other than POST)
501 | Invalid public key component
502 | Invalid message hash
503 | Invalid value of the `tonce` parameter
504 | The key is not authoriced to use this API method
505 | Invalid value of the `method` parameter
506 | Too many commands in a given time interval
400 | Invalid value of the `market` parameter
401 | Invalid value of the `type` parameter
402 | Invalid value of the `amount` parameter
403 | Invalid value of the `rate` parameter
404 | The operation amount is invalid
405 | Insufficient account balance to perform the operation
406 | The user has no access to specified market offer
407 | Invalid value of the `currency` parameter
408 | Invalid value of the `count` parameter
409 | Invalid value of the `start` parameter
300 | Internal application error

API method list
---------------

The list of currently supported API methods is presented below. 
The method name must always be passed in the `method` parameter.

### `info` - account information

Input parameters:

*none*

Output parameters:

 * `balances` - object describing the funds available on the user account, with the following properties:
   * `available` - object describing free funds available in each currency.
   * `blocked` - object describing funds in active trade offers, in each currency.

### `trade` - submit an order

Input parameters:

 * `market` - market where the trade must be made (for example, *"BTCEUR"*).
 * `type` - order type: buy (*"bid"* or *"buy"*) or sell (*"ask"* or *"sell"*).
 * `amount` - order amount (in cryptocurrency).
 * `rate` - exchange rate.

Output parameters:

 * `id` - market order identifier.
 * `order` - object describing the newly made order:
   * `id` - market order identifier.
   * `market` - market where the order has been made.
   * `amount` - cryptocurrency amount.
   * `rate` - exchange rate.
   * `fiat` - fiat amount after exchange.
   * `type` - order type (*"buy"* or *"sell"*).
   * `time` - order creation time.
 * `balances` - account balances after the operation (identical to those returned by the `info` command).

Please note that there are three possible scenarios when a trade is submitted:

 1. The trade is executed immediately, because a matched order os already present on the orderbook. 
 In such case, the fields `id` and `order` will be empty, because no order is submitted in the orderbook.
 The `balances` field indicates the new account balances after the trade.
 2. The trade is partially executed. In such case the `order` field will describe the partial order submitted in the orderbook.
 3. The trade is not immediately exdecuted, and the order is submitted in the orderbook in full. 
 In such case the `order` field will contain the copy of the input parameters.
 
### `cancel` - order cancel

Input parameters:

 * `id` - order identifier.
 
Output parameters:

 * `balances` - account balances after canceling the order.
 
### `order` - user order list

Input parameters:

 * `market` - the market from which the orderbook must be returned.
 
The `market` parameter can be empty, in which case the operation will return user orders from all markets
(as an object with fields identical to available markets).
Otherwise an object is returned with the following fields:

 * `buy` - list of buy orders (in the format identical to that returned from the `order` method).
 * `sell` - list of sell orders.

### `history` - account history

Input parameters:

 * `currency` - currency of the operations to be fetched.
 * `count` - number of list elements, possible values: from 1 to 1000 (1000 is the default).
 * `start` - number of the first element, zero based (0 is the default).
 
Output parameters:

 * `total` - total number of elements.
 * `start` - number of the first list element.
 * `count` - number of returned list elements.
 * `results` - the list of history entries, each object has the following parameters:
   * `id` - operation identifier.
   * `amount` - operation amount.
   * `currency` - operation currency.
   * `rate` - exchange rate (if the operation describes a trade on the market).
   * `commission` - operation commission.
   * `time` - operation time.
   * `type` - operation type, such as:
     * *"deposit"* - deposit of funds.
     * *"withdraw"* - withdrawal of funds.
     * *"withdrawCancel"* - withdrawal cancellation.
     * *"order"* - order submission.
     * *"trade"* - market trade.
     * *"cancel"* - order cancellation.

### `trades` - list of market trades

Input parameters:

 * `market` - market where the trades took place, for example *"BTCEUR"*.
 * `count` - number of list elements, possible values: from 1 to 1000 (1000 is the default).
 * `start` - number of the first element, zero based (0 is the default).
 
Output parameters:

 * `total` - total number of elements.
 * `start` - number of the first list element.
 * `count` - number of returned list elements.
 * `results` - the list of trades, each object has the following parameters:
   * `id` - trade identifier.
   * `type` - trade type (*"buy"* or *"sell"*).
   * `amountCrypto` - cryptocurrency amount.
   * `currencyCrypto` - crptocurrency code (for example *"BTC"*).
   * `amountFiat` - fiat amount.
   * `currencyFiat` - fiat currency code (for example *"EUR"*).
   * `rate` - exchange rate.
   * `time` - trade time.

### `withdraw` - withdraw cryptocurrency

Input parameters:

 * `amount` - amount to withdraw.
 * `currency` - cryptocurrency code (like *"BTC"*).
 * `address` - wallet address where the funds must be withdrawn.

Output value: the withdrawal transaction ID.

### `deposit` - deposit cryptocurrency

Input parameters:

 * `currency` - cryptocurrency code (like "*BTC*").

Output value: the address for cryptocurrency deposits to the account.

