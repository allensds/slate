---
title: API Reference

language_tabs: # must be one of https://git.io/vQNgJ
  - ruby
  - python
  - javascript

# toc_footers:
#   - <a href='#'>Sign Up for a Developer Key</a>
#   - <a href='https://github.com/lord/slate'>Documentation Powered by Slate</a>

# includes:
#   - errors

search: true
---

# Introduction

Welcome to iSTOX trader and developer documentation. These documents outline exchange functionality, market details, and APIs.

APIs are separated into two categories: trading and feed. Trading APIs require authentication and provide access to placing orders and other account information. Feed APIs provide market data and are public.

# General

Market overview and general information.

# Matching Engine

iSTOX operates a continuous first-come, first-serve order book. Orders are executed in price-time priority as received by the matching engine.

# Data Centers

iSTOX data centers are in the Amazon Asia Pacific Singapore (ap-southeast-1) region.

# Sandbox

A public sandbox is available for testing API connectivity and web trading. While the sandbox only hosts a subset of the production order books, all of the exchange functionality is available. Additionally, in this environment you are allowed to add unlimited fake funds for testing.

Login sessions are separate from production.

##Sandbox URLs
When testing your API connectivity, make sure to use the following URLs.

###Website
**https://oauth.istox.com**

###REST API
**https://trading.istox.com/api.v2**

###Websocket Feed
**TODO**

###FIX API
**tcp+ssl://fix.istox.com:31337**

<a name="REST"></a>

# **REST API**

The REST API has endpoints for account and order management as well as public market data.

**REST API ENDPOINT URL** <br />
**https://trading.istox.com/api/v2**

There is also a [FIX API](#FIX) for order management.

# Requests

All requests and responses are application/json content type and follow typical HTTP response status codes for success and failure.

## Errors

Unless otherwise stated, errors to bad requests will respond with HTTP 4xx or status codes. The body will also contain a message parameter indicating the cause. Your language’s http library should be configured to provide message bodies for non-2xx requests so that you can read the message field from the body.

###Common error codes

Status Code | Reason
-------------- | -------------- 
400	| Bad Request – Invalid request format
401	| Unauthorized – Invalid API Key
403	| Forbidden – You do not have access to the requested resource
404	| Not Found
500	| Internal Server Error – We had a problem with our server

## Success

A successful response is indicated by HTTP status code 200 and may contain an optional body. If the response has a body it will be documented under each resource below.

<a name="aunthentication"></a>
# Authentication

```python
{
import istox

client = istox.Client(endpoint="", application_id="")
# pass in email and password for your account,
# and a publicKey and privateKey pair generated from your side
# istox will only receive your public key, the private key is
# used by the client internal calls which will not be sent to istox
jwt = client.get_api_jwt(email, password, publicKey, privateKey)
}
```

```javascript
{
const client = require("istox/client")
const base64url = require('base64url')

var opts = {};

//add your own account and keys
var email = "";
var password = "";
var privateKeyBase64 = "";
var publicKeyBase64 = "";
var privateKey = base64url.decode(privateKeyBase64);
var publicKey = base64url.decode(publicKeyBase64);

var myclient = client(opts)

myclient.get_api_jwt(email, password, publicKey, privateKey).then((jwt) => {
    console.log(JSON.stringify(jwt, null, 2));
})
}
```

Clients are required to pass json web token (JWT) in the request header in order to call private APIs. To get JWT, clients need to do a few REST API calls to iSTOX's authentication server. iSTOX created a client library for Python and NodeJs to ease the steps.

To install Python package, run **pip install istox**

To install NodeJs npm package, run **npm install istox**


# Orders

## Place a New Order

```json
{
    "size": "0.01",
    "price": "0.100",
    "side": "buy",
    "product_id": "BTC-USD"
}

Response

{
    "id": "d0c5340b-6d6c-49d9-b567-48c4bfca13d2",
    "price": "0.10000000",
    "size": "0.01000000",
    "product_id": "BTC-USD",
    "side": "buy",
    "stp": "dc",
    "type": "limit",
    "time_in_force": "GTC",
    "post_only": false,
    "created_at": "2016-12-08T20:02:28.53864Z",
    "fill_fees": "0.0000000000000000",
    "filled_size": "0.00000000",
    "executed_value": "0.0000000000000000",
    "status": "pending",
    "settled": false
}
```

You can place two types of orders: **limit** and **market**. Orders can only be placed if your account has sufficient funds. Once an order is placed, your account funds will be put on hold for the duration of the order. How much and which funds are put on hold depends on the order type and parameters specified. See the **Holds** details below.

###HTTP REQUEST
**POST /market/orders**

###PARAMETERS
These parameters are common to all order types. Depending on the order type, additional parameters will be required (see below).

Param	| Description
-------------- | -------------- 
ord_type	| [optional] **limit** or **market** (default is **limit**)
side | **buy** or **sell**
market | A valid market (e.g. btcusd)
volume | Volume of the token

###LIMIT ORDER PARAMETERS

Param	| Description
-------------- | -------------- 
price	| Price of the token


###MARKET
The market must match a valid market. The market list is available via the [public/markets](#markets) endpoint.

###ORDER TYPE
When placing an order, you can specify the order type. The order type you specify will influence which other order parameters are required as well as how your order will be executed by the matching engine. If **ord_type** is not specified, the order will default to a **limit** order.

**Limit** orders are both the default and basic order type. A limit order requires specifying a **price** and **volume**. The **volume** is the number of bitcoin to buy or sell, and the price is the price per bitcoin. The **limit** order will be filled at the price specified or better. A sell order can be filled at the specified price or a higher price and a buy order can be filled at the specified price or a lower price depending on market conditions. If market conditions cannot fill the limit order immediately, then the limit order will become part of the open order book until filled by another incoming order or canceled by the user.

**market** orders differ from **limit** orders in that they provide no pricing guarantees. They however do provide a way to buy or sell specific amounts of security tokens without having to specify the price. **Market** orders execute immediately and no part of the **market** order will go on the open order book. 

<!-- ###PRICE
The price must be specified in quote_increment product units. The quote increment is the smallest unit of price. For the BTC-USD product, the quote increment is 0.01 or 1 penny. Prices less than 1 penny will not be accepted, and no fractional penny prices will be accepted. Not required for market orders.

###VOLUME
The volume must be greater than the min_ask_amount for the market and no larger than the base_max_size. The size can be in any increment of the base currency (BTC for the BTC-USD product), which includes satoshi units. size indicates the amount of BTC (or base currency) to buy or sell. -->

###HOLDS

For **limit buy** orders, we will hold price x size x (1 + fee-percent) USD. For **sell** orders, we will hold the number of tokens you wish to sell. Actual fees are assessed at time of trade. If you cancel a partially filled or unfilled order, any remaining funds will be released from hold.

For market buy orders, all of your account balance (in the quote account) will be put on hold for the duration of the market order (usually a trivially short time). For a sell order, the volume in tokens will be put on hold. 

###ORDER LIFECYCLE

The HTTP Request will respond when an order is either rejected (insufficient funds, invalid parameters, etc) or received (accepted by the matching engine). A 200 response indicates that the order was received and is active. Active orders may execute immediately (depending on price and market conditions) either partially or fully. A partial execution will put the remaining size of the order in the open state. An order that is filled completely, will go into the done state.

###RESPONSE

A successful order will be assigned an order id. A successful order is defined as one that has been accepted by the matching engine.

## Cancel an Order

Cancel a previously placed order.

If the order had no matches during its lifetime its record may be purged. This means the order details will not be available with GET /market/orders/<order-id>.

###HTTP REQUEST
**POST /market/orders/<order-id>/cancel**

<aside class="notice">
The order id is the server-assigned order id when a new order is placed.
</aside>

###CANCEL REJECT
If the order could not be canceled (already filled or previously canceled, etc), then an error response will indicate the reason in the message field.

## Cancel all

With best effort, cancel all open orders. The response is a list of ids of the canceled orders.

###HTTP REQUEST
**POST /market/orders/cancel**

###QUERY PARAMETERS

Param	| Description
-------------- | -------------- 
side	| [optional] **buy** or **sell** If present, only sell orders or buy orders will be canncelled.

## List Orders

List all your orders. 

###HTTP REQUEST
**GET /market/orders**

###QUERY PARAMETERS

Param	| Description
-------------- | -------------- 
market | [optional]	Only list orders for a specific market
state |	[optional, wait, done, cancel]	Limit list of orders to these states
limit	| [optional] Limit the number of returned orders, default to 100
page | [optional] Specify the page of paginated results
order_by | [optional, asc or desc] If set, returned orders will be sorted in specific order, default to "desc"

<aside class="notice">
This request is paginated.
</aside>

###POLLING

For high-volume trading it is strongly recommended that you maintain your own list of open orders and use one of the streaming market data feeds to keep it updated. You should poll the open orders endpoint once when you start trading to obtain the current state of any open orders.

<aside class="notice">
Open orders may change state between the request and the response depending on market conditions.
</aside>

## Get an Order

Get a single order by order id.

###HTTP REQUEST
**GET market/orders/<order-id>**

<aside class="notice">
Open orders may change state between the request and the response depending on market conditions.
</aside>

## Get Trades

Get your executed trades. Trades are sorted in reverse creation order.

###HTTP REQUEST
**GET /market/trades**

###PARAMETERS

Name |	Description
-------------- | -------------- 
market |	market symbol
limit	| [optional] Limit the number of returned orders, default to 100
page | [optional] Specify the page of paginated results, default to 1
timestamp | [optional] An integer represents the seconds elapsed since Unix epoch. If set, only trades executed before the time will be returned
order_by | [optional, asc or desc] If set, returned orders will be sorted in specific order, default to "desc"

# Deposites

TODO?

# Fees

TODO?

# Withdrawals

TODO?

# Markets

<a name="markets"></a>

## Get Markets

```json
{
  [
    {
        "id": "usdbtc",
        "name": "USD/BTC",
        "ask_unit": "usd",
        "bid_unit": "btc",
        "ask_fee": "0.0015",
        "bid_fee": "0.0015",
        "min_ask_price": "0.0",
        "max_bid_price": "0.0",
        "min_ask_amount": "0.0",
        "min_bid_amount": "0.0",
        "ask_precision": 4,
        "bid_precision": 4
    },
    ...
  ]
}
```

Get a list of available currency pairs for trading.

###HTTP REQUEST
**GET /public/markets**

<aside class="notice">
Market ID will not change once assigned to a market but the min/max price/amount can be updated in the future.
</aside>

## Get Market Order Book

Get a list of open orders for a market. 

###HTTP REQUEST
**GET /public/markets/{market}/order-book**

###PARAMETERS

Name |	Description
-------------- | -------------- 
market |	market symbol
asks_limit | Limit the number of returned sell orders. Default to 20
bids_limit | Limit the number of returned buy orders. Default to 20

## Get Market Ticker

Snapshot information about the last trade (tick), best bid/ask and 24h volume.

###HTTP REQUEST
**GET /public/markets/{market}/tickers**

To get tickers of all markets: <br />
**GET /public/markets/tickers**

###REAL-TIME UPDATES
Polling is discouraged in favor of connecting via the websocket stream and listening for match messages.

###PARAMETERS

Name |	Description
-------------- | -------------- 
market |	market symbol

## Get Trades

Get recent trades on market, each trade is included only once. Trades are sorted in reverse creation order.

###HTTP REQUEST
**GET /public/markets/{market}/trades**

###REAL-TIME UPDATES
Polling is discouraged in favor of connecting via the websocket stream and listening for match messages.

###PARAMETERS

Name |	Description
-------------- | -------------- 
market |	market symbol
limit	| [optional] Limit the number of returned orders, default to 100
page | [optional] Specify the page of paginated results, default to 1
timestamp | [optional] An integer represents the seconds elapsed since Unix epoch. If set, only trades executed before the time will be returned
order_by | [optional, asc or desc] If set, returned orders will be sorted in specific order, default to "desc"

## Get Market Depth

Get depth or specified market. Both asks and bids are sorted from highest price to lowest.

###HTTP REQUEST
**GET /public/markets/{market}/depth**

###PARAMETERS

Name |	Description
-------------- | -------------- 
market |	market symbol
limit	| [optional] Limit the number of returned orders, default to 300

# Currencies

## Get Currencies

Get list of currencies.

###HTTP REQUEST
**GET /public/currencies**

**PARAMETERS**

Name |	Description
-------------- | -------------- 
type |	[optional, fiat or coin] 

## Get a Currency

Get list of currencies.

###HTTP REQUEST
**GET /public/currencies/{id}**

###PARAMETERS

Name |	Description
-------------- | -------------- 
id |	symbol, e.g. USD 

# Time

Get API server current time, in seconds since [Unix Epoch](https://en.wikipedia.org/wiki/Unix_time)

###HTTP REQUEST
**GET /public/timestamp**

# Websocket Feed

The websocket feed provides real-time market data updates for orders and trades.

**TODO**

# Overview

Real-time market data updates provide the fastest insight into order flow and trades. This however means that you are responsible for reading the message stream and using the message relevant for your needs which can include building real-time order books or tracking real-time trades.

The websocket feed is publicly available, but connections to it are rate-limited to 1 per 4 seconds per IP.

## Protocol Overview

The websocket feed uses a bidirectional protocol, which encodes all messages as JSON objects. All messages have a type attribute that can be used to handle the message appropriately.

Please note that new message types can be added at any point in time. Clients are expected to ignore messages they do not support.

Error messages: Most failure cases will cause an error message (a message with the type "error") to be emitted. This can be helpful for implementing a client or debugging issues.

## Subscribe

```json
// Request
// Subscribe to ETH-USD and ETH-EUR with the level2, heartbeat and ticker channels,
// plus receive the ticker entries for ETH-BTC and ETH-USD
{
    "type": "subscribe",
    "product_ids": [
        "ETH-USD",
        "ETH-EUR"
    ],
    "channels": [
        "level2",
        "heartbeat",
        {
            "name": "ticker",
            "product_ids": [
                "ETH-BTC",
                "ETH-USD"
            ]
        }
    ]
}
```
```json
// Response
{
    "type": "subscriptions",
    "channels": [
        {
            "name": "level2",
            "product_ids": [
                "ETH-USD",
                "ETH-EUR"
            ],
        },
        {
            "name": "heartbeat",
            "product_ids": [
                "ETH-USD",
                "ETH-EUR"
            ],
        },
        {
            "name": "ticker",
            "product_ids": [
                "ETH-USD",
                "ETH-EUR",
                "ETH-BTC"
            ]
        }
    ]
}
```
```json
// Request
{
    "type": "unsubscribe",
    "product_ids": [
        "ETH-USD",
        "ETH-EUR"
    ],
    "channels": ["ticker"]
}
```
```json
// Request
{
    "type": "unsubscribe",
    "channels": ["heartbeat"]
}
```
```json
// Authenticated feed messages add user_id and
// profile_id for messages related to your user
{
    "type": "open", // "received" | "open" | "done" | "match" | "change" | "activate"
    "user_id": "5844eceecf7e803e259d0365",
    "profile_id": "765d1549-9660-4be2-97d4-fa2d65fa3352",
    /* ... */
}
```
```json
// Request
{
    "type": "subscribe",
    "product_ids": [
        "BTC-USD"
    ],
    "channels": ["full"],
    "signature": "...",
    "key": "...",
    "passphrase": "...",
    "timestamp": "..."
}
```

To begin receiving feed messages, you must first send a subscribe message to the server indicating which channels and products to receive. This message is mandatory — you will be disconnected if no subscribe has been received within 5 seconds.

There are two ways to specify products ids to listen for within each channel: First, you can specify the product ids for an individual channel. Also, as a shorthand, you can define products ids at the root of the object, which will add them to all the channels you subscribe to.

Once a subscribe message is received the server will respond with a subscriptions message that lists all channels you are subscribed to.

Subsequent subscribe messages will add to the list of subscriptions. In case you already subscribed to a channel without being authenticated you will remain in the unauthenticated channel.

If you want to unsubscribe from channel/product pairs, send an unsubscribe message. The structure is equivalent to subscribe messages. As a shorthand you can also provide no product ids for a channel, which will unsubscribe you from the channel entirely.

As a response to an unsubscribe message you will receive a subscriptions message.

###AUTHENTICATION

It is possible to authenticate yourself when subscribing to the websocket feed.

Authentication will result in a couple of benefits:

1. Messages where you’re one of the parties are expanded and have more useful fields
1. You will receive private messages, such as lifecycle information about stop orders you placed

Here’s an example of an authenticated subscribe request:

To authenticate, you send a subscribe message as usual, but you also pass in fields just as if you were signing a request to GET /users/self/verify. To get the necessary parameters, you would go through the same process as you do to make authenticated calls to the API.

<aside class="notice">
Authenticated feed messages do not increment the sequence number. It is currently not possible to detect if an authenticated feed message was dropped.
</aside>

## Sequence Number

Most feed messages contain a sequence number. Sequence numbers are increasing integer values for each product with every new message being exactly 1 sequence number than the one before it.

If you see a sequence number that is more than one value from the previous, it means a message has been dropped. A sequence number less than one you have seen can be ignored or has arrived out-of-order. In both situations you may need to perform logic to make sure your system is in the correct state.

<aside class="notice">
While a websocket connection is over TCP, the websocket servers receive market data in a manner which can result in dropped messages. Your feed consumer should either be designed to expect and handle sequence gaps and out-of-order messages, or use channels that guarantee delivery of messages.
<br />
<br />
If you want to keep an order book in sync, consider using the level 2 channel, which provides such guarantees.
</aside>

# Channels


<a name="FIX"></a>


# FIX API

FIX ([Financial Information eXchange](https://en.wikipedia.org/wiki/Financial_Information_eXchange)) is a standard protocol which can be used to enter orders, submit cancel requests, and receive fills. Users of the FIX API will typically have existing software using FIX for order management. Users who are not familiar with FIX should first consider using the [REST API](#REST).

**FIX API ENDPOINT URL** <br />
**tcp+ssl://fix.istox.com:31337**

# Connectivity

Before logging onto a FIX session, clients must establish a secure connection to the FIX gateway (**fix.istox.com:31337**). If your FIX implementation does not support establishing a TCP SSL connection natively, you will need to setup a local proxy such as [stunnel](https://www.stunnel.org/) to establish a secure connection to the FIX gateway. See the [SSL Tunnels](#ssl stunnel) section for more details and examples.

<aside class="notice">
iSTOX does not support static IP addresses. If your firewall rules require a static IP address, you will need to create a TCP proxy server with a static IP address which is capable of resolving an IP address using DNS.
If connecting from servers outside of AWS which require firewall rules, please use the AWS provided resources to determine how best to whitelist AWS IP ranges.
</aside>

# Messages

The baseline specification for this API is FIX 4.2. There are additional tags from later versions of FIX, and custom tags in the high number range as allowed by the standard.

A standard header must be present at the start of every message in both directions.

Tag	| Name	| Description
-------------- | -------------- | --------------
8	| BeginString	| e.g. FIX.4.2
49 | SenderCompID	| Client ID provided by iSTOX (on messages from the client)
56 | TargetCompID	| Must be **iSTOX** (on messages from the client)


## Logon (A)

Sent by the client to initiate a session, and by the server as an acknowledgement. Only one session may exist per connection; sending a Logon message within an established session is an error.

Tag	| Name	| Description
-------------- | -------------- | --------------
98	| EncryptMethod	| Must be **0** (None)
108	| HeartBtInt	| Must be **30** (seconds)
96	| RawData	| JWT for authentication (see below)
95	| RawDataLength	| length of RawData

<!-- 8013	| CancelOrdersOnDisconnect	| Y: Cancel all open orders for the current profile; S: Cancel open orders placed during session
9406	| DropCopyFlag	| If set to Y, execution reports will be generated for all user orders (defaults to Y) -->

The Logon message sent by the client must contain a json web token (JWT) for authentication, and the JWT is passed with the RawData field. To get the JWT, please see [Authentication](#aunthentication)

## Logout (5)

Sent by either side to initiate session termination. The side which receives this message first should reply with the same message type to confirm session termination. Closing a connection without logging out of the session first is an error.

## New Order Single( D)

Sent by the client to enter an order.

Tag	| Name	| Description
-------------- | -------------- | --------------
21	| HandlInst	| Must be **1** (Automated)
11	| ClOrdID	| UUID selected by client to identify the order
55	| Symbol	| E.g. BTC-USD
54	| Side	| Must be **1** to buy or **2** to sell
44	| Price	| Limit price (e.g. in SGD) (Limit order only)
38	| OrderQty	| Order size in base units (e.g. BTC)
152	| CashOrderQty	| Order size in quote units (e.g. SGD) (Market order only)
40	| OrdType	| Must be **1** for Market or **2** for Limit
59	| TimeInForce	| Must be a valid TimeInForce value. See the table below (Limit order only)
7928	| SelfTradePrevention	| Optional, see the table below


###SELFTRADEPREVENTION VALUES

Value	| Description
-------------- | -------------- | --------------
**D**	| Decrement and cancel (the default)
**O**	| Cancel resting order
**N**	| Cancel incoming order
**B**	| Cancel both orders
If an order is decremented due to self-trade prevention, an Execution Report will be sent to the client with ExecType=D indicating unsolicited OrderQty reduction (i.e. partial cancel).

See the self-trade prevention documentation for more details about this field.


###TIMEINFORCE VALUES

Value	| Description
-------------- | -------------- | --------------
**1**	| Good Till Cancel
**3**	| Immediate or Cancel
**4**	| Fill or Kill
**P**	| Post-Only
The post-only flag (P) indicates that the order should only make liquidity. If any part of the order results in taking liquidity, the order will be rejected and no part of it will execute. Open Post-Only orders will be treated as Good Till Cancel.

See the time in force documentation for more details about these values.

###ERRORS

If a trading error occurs (e.g. user has insufficient funds), an ExecutionReport with **ExecType=8** is sent back, signifying that the order was rejected.

## Order Cancel Request (F)

Sent by the client to cancel an order.

Tag	| Name | Description
-------------- | -------------- | --------------
11	| ClOrdID	| UUID selected by client for the order
37	| OrderID	| OrderID from the ExecutionReport with OrdStatus=New (39=0)
41	| OrigClOrdID	| ClOrdID of the order to cancel (originally assigned by the client)
55	| Symbol	| Symbol of the order to cancel (must match Symbol of the Order)

###CLORDID

Use of the ClOrdID is not available after reconnecting or starting a new session. You should use the OrderID obtained via the ExecutionReport once available.

## Order Status Request (H)

Sent by the client to obtain information about pending orders.

Tag	| Name	| Description
-------------- | -------------- | --------------
37	| OrderID	| OrderID of order(s) to be sent back. Can be equal to * (wildcard) to send back all pending orders


###RESPONSE

The response to an Order Status Request is a series of ExecutionReports with **ExecType=I**, each representing one open order belonging to the user. If the user has no open orders, a single ExecutionReport is sent back with **OrderID=0**.

## Execution Report (8)

Sent by the server when an order is accepted, rejected, filled, or canceled. Also sent when the user sends an **OrderStatusRequest**.

Tag	| Name	| Description
-------------- | -------------- | --------------
11	| ClOrdID	| Only present on order acknowledgements, ExecType=New (150=0)
37	| OrderID	| OrderID from the ExecutionReport with ExecType=New (150=0)
55	| Symbol	| Symbol of the original order
54	| Side	| Must be **1** to buy or **2** to sell
32	| LastShares	| Amount filled (if ExecType=1). Also called LastQty as of FIX 4.3
44	| Price	| Price of the fill if ExecType indicates a fill, otherwise the order price
38	| OrderQty	| OrderQty as accepted (may be less than requested upon self-trade prevention)
152	| CashOrderQty	| Order size in quote units (e.g. USD) (Market order only)
60	| TransactTime	| Time the event occurred
150	| ExecType	| May be 1 (Partial fill) for fills, D for self-trade prevention, etc.
39	| OrdStatus	| Order status as of the current message
103	| OrdRejReason	| Insufficient funds=3, Post-only=8, Unknown error=0
136	| NoMiscFees	| 1 (Order Status Request response only)
137	| MiscFeeAmt	| Fee (Order Status Request response only)
139	| MiscFeeType	| 4 (Exchange fees) (Order Status Request response only)
1003 | TradeID	| Product unique trade id
1057 | AggressorIndicator	| Y for taker orders, N for maker orders

###EXECTYPE VALUES

ExecType	| Description
-------------- | --------------
0	| New Order
1	| Fill
3	| Done
4	| Canceled
7	| Stopped
8	| Rejected
D	| Order Changed
I	| Order Status

## Order Cancel Reject (9)

Sent by the server when an Order Cancel Request cannot be satisfied, e.g. because the order is already canceled or completely filled.

Tag	| Name	| Description
-------------- | -------------- | --------------
11	| ClOrdID	| As on the cancel request
37	| OrderID	| As on the cancel request
41	| OrigClOrdID	| As on the cancel request
39	| OrdStatus	| 4 if too late to cancel
102	| CxlRejReason	| 1 if the order is unknown
434	| CxlRejResponseTo	| 1 (Order Cancel Request)

## Reject (3)

Sent by either side upon receipt of a message which cannot be processed, e.g. due to missing fields or an unsupported message type.

Tag	| Name	| Description
-------------- | -------------- | --------------
45	| RefSeqNum	| MsgSeqNum of the rejected incoming message
371	| RefTagID	| Tag number of the field which caused the reject (optional)
372	| RefMsgType	| MsgType of the rejected incoming message
58	| Text	| Human-readable description of the error (optional)
373	| SessionRejectReason	| Code to identify reason for reject


###SESSIONREJECTREASON VALUES

The following values can be sent by the server.

Value	| Description
-------------- | --------------
1	| Required tag missing
5	| Value is incorrect (out of range) for this tag
6	| Incorrect data format for value
11	| Invalid MsgType (35)

## Heartbeat (0)

Sent by both sides if no messages have been sent for HeartBtInt seconds as agreed during logon. May also be sent in response to a Test Request.

Tag	Name	| Description
-------------- | --------------
112	| TestReqID	Copied from the Test Request, if any

## Test Request (1)

May be sent at any time by either side.

Tag	Name	| Description
-------------- | --------------
112	| TestReqID	Free text

<a name="ssl stunnel"></a>

# SSL Tunnel

fix.istox.com:31337 only accepts TCP connections secured by SSL. If your FIX client library cannot establish an SSL connection natively, you will need to run a local proxy that will establish a secure connection and allow unencrypted local connections.

## Stunnel Configuration

This is an example configuration file for stunnel to listen on a port locally and proxy unencrypted TCP connections to the encrypted SSL connection. The service name (iSTOX) and the accept port (31336) may be changed to any suitable values.

```
[iSTOX]
client = yes
accept = 31336
connect = fix.istox.com:31337
verify = 4
CAfile = /example/path/to/fix.istox.com.pem
```

When stunnel is started with the above configuration file, it will run in the background. On Unix-like systems the option foreground = yes may be specified at the top of the file to avoid running in the background. For testing it may be easier to use foreground mode, or to specify the top-level output option as a file path where stunnel will write log messages

<aside class="notice">
The stunnel configuration must include either **verify=3** or **verify=4** to enable client certificate pinning. The exchange certificate is available via **istox.com** and must be installed in a secure (not openly writable) directory on the client system which is specified in the stunnel configuration file as **CAfile**.
</aside>

If your system has OpenSSL installed, you can run this command to download the certificate:

**openssl s_client -showcerts -connect fix.istox.com:31337 | openssl x509 -outform PEM > fix.istox.pem**

<aside class="notice">
To connect to sandbox, change the url to fix.istox.com to download the appropriate certificate.
</aside>
