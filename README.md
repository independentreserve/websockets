# Independent Reserve - WebSockets

For more details on Independent Reserve's WebSockets API, please refer to [API Documentation](https://www.independentreserve.com/api#websockets)

## How to connect

WebSocket base url: `wss://websockets.independentreserve.com`
Message Encoding: `JSON`

## Available Channels

There are two channels available over WebSockets: **[orderbook](#orderbook-channel)** and **[ticker](#ticker-channel)**.
Each channel supports subscribing to currency pair events on the channel. See [Subscribing to Channels](#subscribing-to-channels) section for more details.

## Subscribing to Channels

Channels can be subscribed to using channel names in the format of `{channel type}-{cryptocurrency}-{fiatcurrency}`.

Where:

* **{channel type}** is `orderbook` or `ticker`
* **{cryptocurrency}** is a [primary currency code](https://www.independentreserve.com/API#GetValidPrimaryCurrencyCodes)
* **{fiatcurrency}** is a [secondary currency code](https://www.independentreserve.com/API#GetValidSecondaryCurrencyCodes)

E.g: `orderbook-xbt-aud`, `ticker-xbt-aud`

Subscribing to a channel can be done in two ways:

1. Supplying channel names with the `subscribe` query string when opening a connection. Eg: `wss://websockets.independentreserve.com?subscribe=orderbook-xbt-aud,ticker-xbt-aud` will subscribe to order book XBT/AUD events and ticker XBT/AUD
2. Sending a subscribe message once the connection is established. The subscribe message should set the **data** property to a string array with the channels to subscribe to.

### Subscribe message format

```json
{
    "Event":"Subscribe",
    "Data":["orderbook-xbt-aud", "ticker-xbt-aud"]
}
```

## Unsubscribing from Channels

Unsubscribing from a channel is done using an **Unsubscribe** message with the **data** property listing the channels to unsubscribe from.

### Unsubscribe message format

```json
{
    "Event":"Unsubscribe",
    "Data":["orderbook-xbt-aud", "ticker-xbt-aud"]
}
```

## Subscription Confirmation Event

Successful subscriptions and unsubscriptions are notified via the **Subscriptions** event. The **Data** property will list current subscriptions.

### Subscriptions event

```json
{
    "Event":"Subscriptions",
    "Data":["orderbook-xbt-aud", "ticker-xbt-aud"]
}
```

## Orderbook Channel

The order book channel provides real-time order book updates. Order created, canceled and modified events are published on the order book channel.

### NewOrder - Order Created Event

The NewOrder event is published when a limit order is placed on the order book.

```json
{
    "Event":"NewOrder",
    "Channel": "orderbook-xbt-aud",
    "Data":{
        "OrderGuid":"fa091562-4101-46de-8d66-aeddbeb8795b",
        "PrimaryCurrencyCode":"Xbt",
        "SecondaryCurrencyCode":"Aud",
        "Price":10270.31,
        "OrderType":"LimitBid",
        "Volume":1.0,
        "Nonce":1
        }
}
```

**OrderType** values can be: `LimitBid` or `LimitOffer`

### OrderChanged - Order modified Event

The OrderChanged event is published with updated volume when an order is filled or partially filled due to a trade. Fully filled orders will have `"Volume":0`

```json
{
    "Event":"OrderChanged",
    "Channel": "orderbook-xbt-aud",
    "Data":{
        "OrderGuid":"fa091562-4101-46de-8d66-aeddbeb8795b",
        "PrimaryCurrencyCode":"Xbt",
        "SecondaryCurrencyCode":"Aud",
        "OrderType":"LimitBid",
        "Volume":0.5,
        "Nonce":2
        }
}
```

The only value that can be updated is in a change event is `Volume`.

#### OrderCanceled - Order Cancelled Event

The OrderCanceled event is published when an order is canceled.

```json
{
    "Event":"OrderCanceled",
    "Channel": "orderbook-xbt-aud",
    "Data":{
        "OrderGuid":"fa091562-4101-46de-8d66-aeddbeb8795b",
        "PrimaryCurrencyCode":"Xbt",
        "SecondaryCurrencyCode":"Aud",
        "OrderType":"LimitBid",
        "Nonce":3
        }
}
```

## Ticker Channel

The ticker channel provides realtime trade updates. Trade events are published on the ticker channel.

### Trade Event

Trade events are published on every trade. Note: The SecondaryCurrencyCode of a trade event will be the currency the trade is executed in and not the {fiatcurrency} of the channel it is published on. Eg: Subscribing to ticker-xbt-aud will publish trade events with PrimaryCurrencyCode:"Xbt" and SecondaryCurrencyCode:"Aud", "Usd" or "Nzd" depending on the native secondary currency of the trade.

```json
{
    "Event":"Trade",
    "Channel": "orderbook-xbt-aud",
    "Data":{
        "TradeGuid":"c5bde544-d8ae-4e38-9e90-405a3f93b6d6",
        "TradeDate":"2009-01-03T18:15:05.9321664+00:00",
        "Volume":50.0,
        "Price":10270.0,
        "PrimaryCurrencyCode":"Xbt",
        "SecondaryCurrencyCode":"Aud",
        "BidGuid":"ebbeca4b-7148-4230-ad8f-833a3ccf35c2",
        "OfferGuid":"ad5ece89-083b-49fc-8bc1-bdb7482a9b9a",
        "Side":"Buy",
        "Nonce":1
        }
}
```

**Side** values can be: `Buy` or `Sell`. The **BidGuid** and **OfferGuid** fields match those published in the orderbook channel.

## Nonce

Each event on a channel will have a nonce. A nonce is an increasing integer value for each channel. A nonce will increase by exactly 1 with every published event on the channel. An increase of more than 1 from the previous event indicates a dropped event. A nonce less than the previous event indicates a reset channel. In both cases it's advised to have logic to ensure the subscriber is in the correct state.
Nonces are assigned per channel, per crypto currency and are duplicated for each of the fiat currency subscribtions. Eg: **orderbook-xbt-aud** will have the same nonce as the **orderbook-xbt-usd** and **orderbook-xbt-nzd**, but will have a different nonce to **ticker-xbt-aud** or **orderbook-eth-aud**

## Heartbeat

A heartbeat event is published every 60 seconds. Note: this interval may change in the future.

```json
{
    "Event":"Heartbeat",
}
```

## Troubleshooting

### 404 Status Code

* Make sure you are using the correct base URL. See [How to connect](#how-to-connect).
* Make sure that you are using a websockets protocol to open a connection.

If the status description on the response is "WebSockets disabled", the WebSockets server is temporarily unavailable.

### 400 Status Code

If you get status code of 400 when using a subscription query string, check that the query string is in the correct format. See [Subscribing to Channels](#subscribing-to-channels).

### Error event on subscribing

If you get an **Error** event when subscribing, check the error data field for details and check the subscribe format.

## Samples

### JavaScript

JavaScript native WebSocket support:

```javascript
var webSocket = new WebSocket('wss://websockets.independentreserve.com/?subscribe=orderbook-xbt-aud');
webSocket.onmessage = function (event) {
    console.log(event);
}
```

JavaScript example: [Source Code](https://github.com/independentreserve/websockets/tree/master/samples/JavaScript)

JavaScript example: [Demo - Live XBT-AUD Orderbook Events](https://independentreserve.github.io/websockets/samples/JavaScript/orderbook.html)
