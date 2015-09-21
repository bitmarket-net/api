Summary
=======

This document provides information about the BitMarket.net API. 
For a quick start, you can jump to the [PHP sample code](#info_sample).

Table of contents
-----------------

 * API basics
   * [How to access the API?](#info_access)
   * [API keys](#info_keys)
   * [Message signing](#info_signing)
   * [The tonce parameter](#info_tonce)
   * [Call limits](#info_limits)
   * [Server responses](#info_responses)
   * [Sample code](#info_sample)
   * [Sample return values](#info_return)
   * [Error codes](#info_codes)
 * Method list
   * [info](#api_info) - account information
   * [trade](#api_trade) - submit an order
   * [cancel](#api_cancel) - order cancel
   * [orders](#api_orders) - list of user orders
   * [trades](#api_trades) - list of user trades
   * [history](#api_history) - account operation history
   * [withdraw](#api_withdraw) - withdraw cryptocurrency
   * [deposit](#api_deposit) - deposit cryptocurrency
   * [marginList](#api_marginList) - list of open positions
   * [marginOpen](#api_marginOpen) - open a long or short position
   * [marginClose](#api_marginClose) - close a position
   * [marginCancel](#api_marginCancel) - cancel opening a position
   * [marginModify](#api_marginModify) - modify position parameters
   * [swapList](#api_swapList) - swap contract list
   * [swapOpen](#api_swapOpen) - open swap contract
   * [swapClose](#api_swapClose) - close swap contract

How to access the API?
----------------------

<a name="info_access"></a>
To make an API call, you need to send a POST request to the following address:
https://www.bitmarket.pl/api2/

The request must include appropriate parameters describing the API method to be called. 
There are two parameters that must be sent woth every request: `method` which selects the API method to be used,
and `tonce` which contains the time parameter and is used to avoid replay attacks.
All the other parameters dopend on the API method used.

<a name="info_keys"></a>
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

<a name="info_signing"></a>
### Message signing

The parameters of every API call must be signed to create a HMAC signature. 
The secret key component is used as th einput to the SHA512 method. 
This ensures that even if someone learns the public key component, they will still not be able to send API calls.

The HMAC signature must be passed in the `API-Hash` HTTP header with every API call.

<a name="info_tonce"></a>
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

<a name="info_limits"></a>
### Call limits

The number of API calls that can be made on the user account is limited to 600 calls within 10 minutes. 
This limit is global for the user account, regardless of how many API keys the user is using.

<a name="info_responses"></a>
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

<a name="info_sample"></a>
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

<a name="info_return"></a>
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

<a name="info_codes"></a>
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
507 | Invalid nonce value (only if an old request is sent again - replay attack)
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
410 | Invalid value of the `address` parameter.
411 | Invalid value of the `id` parameter.
412 | Invalid value of the `type` parameter.
413 | Invalid value of the `rateLoss` parameter.
414 | Invalid value of the `rateProfit` parameter.
415 | Cannot close margin because the position is not fully open
416 | Cannot cancel margin because the position is fully open
417 | Order cannot be fully satisfied and all or nothing was requested (no longer in use)
300 | Internal application error

API method list
---------------

The list of currently supported API methods is presented below. 
The method name must always be passed in the `method` parameter.

<a name="api_info"></a>
### `info` - account information

Input parameters:

*none*

Output parameters:

 * `balances` - object describing the funds available on the user account, with the following properties:
   * `available` - object describing free funds available in each currency.
   * `blocked` - object describing funds in active trade offers, in each currency.
 * `account` - object describing the state of the user account, with the following properties:
   * `turnover` - account turnover value.
   * `commissionMaker` - commission value as market maker.
   * `commissionTaker` - commission value as market taker.
 * `bank_deposit_fiat` - object describing information to deposit fiat currency, one for each. Currently only PLN and EUR:
   * `PLN`
     * `bank_name` - name of the bank where the deposit is to be made.
     * `pay_to` - name to whom deposit is to be made.
     * `acc_num` - the account number where deposit is to be made.
     * `transfer_title` - text to identify the  deposit by the user.
   * `EUR` - same as PLN with one additional data
     * `swift_code` - the switch code to be used to transfer EURO

<a name="api_trade"></a>
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

 1. The trade is executed immediately, because a matched order is already present on the orderbook. 
 In such case, the fields `id` and `order` will be empty, because no order is submitted in the orderbook.
 The `balances` field indicates the new account balances after the trade.
 2. The trade is partially executed. In such case the `order` field will describe the partial order submitted in the orderbook.
 3. The trade is not immediately exdecuted, and the order is submitted in the orderbook in full. 
 In such case the `order` field will contain the copy of the input parameters.
 
<a name="api_cancel"></a>
### `cancel` - order cancel

Input parameters:

 * `id` - order identifier.
 
Output parameters:

 * `balances` - account balances after canceling the order.
 
<a name="api_orders"></a>
### `orders` - list of user orders

Input parameters:

 * `market` - the market from which the orderbook must be returned.
 
The `market` parameter can be empty, in which case the operation will return user orders from all markets
(as an object with fields identical to available markets).
Otherwise an object is returned with the following fields:

 * `buy` - list of buy orders (in the format identical to that returned from the `order` method).
 * `sell` - list of sell orders.

<a name="api_trades"></a>
### `trades` - list of user trades

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

<a name="api_history"></a>
### `history` - account operation history

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

<a name="api_withdraw"></a>
### `withdraw` - withdraw cryptocurrency

Input parameters:

 * `amount` - amount to withdraw.
 * `currency` - cryptocurrency code (like *"BTC"*).
 * `address` - wallet address where the funds must be withdrawn.

Output value: the withdrawal transaction ID.

<a name="api_deposit"></a>
### `deposit` - deposit cryptocurrency

Input parameters:

 * `currency` - cryptocurrency code (like "*BTC*").

Output value: the address for cryptocurrency deposits to the account.

<a name="api_marginList"></a>
### `marginList` - list of open positions

Input parameters:

 * `market` - the market from which the position list must be returned.

Return value:

 * `long` - list of open long positions, each object has the following parameters:
   *  `id` - position identifier.
   *  `type` - position type (*"long"* or *"short"*).
   *  `time` - time when the position was submitted.
   *  `leverage` - leverage value.
   *  `fiatTotal` - total position value in fiat currency.
   *  `fiatOpened` - opened amount.
   *  `fiatClosed` - closed amount.
   *  `security` - blocked secutory deposit (in cryptocurrency).
   *  `rate` - requested position open rate.
   *  `rateOpen` - real position open rate.
   *  `rateClose` - position closing rate (if partially closed).
   *  `rateLoss` - Stop Loss rate.
   *  `rateProfit` - Take Profit rate.
   *  `rateCurrent` - current position rate (according to the market orderbook).
   *  `fees` - sum of charged fees.
   *  `profit` - current position profit (in cryptocurrency).
   *  `profitPercentage` - current position profit (in percent).

* `short` - list of open short positions (same as above).
* `performance` - object describing total account performance:
   * `balance` - amount of deposited security guarantee.
   * `blocked` - amount of security guarantee blocked for open positions.
   * `available` - amount of available security guarantee.
   * `profit` - current profit from open positions.
   * `profitPercentage` - current profit (in percent).
   * `value` - final value of the account.

<a name="api_marginOpen"></a>
### `marginOpen` - open a position

Input parameters:

 * `market` - the market on which the position must be opened.
 * `type` - position type (*"long"* or *"short"*).
 * `leverage` - levarage value (from 1.5 to 10).
 * `amount` - position value (in fiat currency).
 * `rate` - position open rate (0 means "at current market rate").
 * `rateLoss` - Stop Loss rate (0 means "do not use Stop Loss").
 * `rateProfit` - Take Profit rate (0 means "do not use Take Profit").

Return value:
 * `id` - identifier of a newly opened position.
 * `long`, `short`, `value` - account status after opening the position (same as with the `marginList` function).

<a name="api_marginClose"></a>
### `marginClose` - close a position

Input parameters:

 * `market` - the market on which the position must be opened.
 * `id` - position identifier.
 * `amount` - amount to close (up to position value).

Return value:
 * `long`, `short`, `value` - account status after opening the position (same as with the `marginList` function).

<a name="api_marginCancel"></a>
### `marginCancel` - cancel opening a position (by lowering its value)

Input parameters:

 * `market` - the market on which the position must be opened.
 * `id` - position identifier.
 * `amount` - new position value.

Return value:
 * `long`, `short`, `value` - account status after opening the position (same as with the `marginList` function).

<a name="api_marginModify"></a>
### `marginModify` - open a position

Input parameters:

 * `market` - the market on which the position must be opened.
 * `id` - position identifier.
 * `rate` - new position open rate (meaningful only if the position is not open).
 * `rateLoss` - new Stop Loss rate (0 means "do not use Stop Loss").
 * `rateProfit` - new Take Profit rate (0 means "do not use Take Profit").

Return value:
 * `long`, `short`, `value` - account status after opening the position (same as with the `marginList` function).

<a name="api_swapList"></a>
### `swapList` - list swap contracts

Input parameters:

 * `currency` - cryptocurrency code (like "*BTC*").

Output value: array of objects describing active contracts on user account, with the following propertied for each object:

 * `id` - contract identifier.
 * `amount` - contract base amount.
 * `rate` - contract interest rate.
 * `earnings` - earnings accrued on contract.

<a name="api_swapOpen"></a>
### `swapOpen` - open swap contract

Input parameters:

 * `currency` - cryptocurrency code (like "*BTC*").
 * `amount` - contract amount.
 * `rate` - contract percentage rate.

Output parameters:

 * `id` - contract identifier.
 * `balances` - balances on the account.

<a name="api_swapClose"></a>
### `swapClose` - close swap contract

Input parameters:

 * `id` - contract identifier.
 * `currency` - cryptocurrency code (like "*BTC*").

Output parameters:

 * `balances` - balances on the account.
