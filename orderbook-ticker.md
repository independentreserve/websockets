# Order Book Ticker

## Overview

The `Order Book Ticker` channels are for receiving a stream of `order` and/or `trade` events from the Independent Reserve server.

## Available Channels

There are two channels available over WebSockets: **[orderbook](#orderbook-channel)** and **[ticker](#ticker-channel)**.

Each channel supports subscribing to currency events.

## Subscribing to Channels

To subscribe to a specific channel, construct a string in the following format:

* For order book: `orderbook-{cryptocurrency}`
* For ticker: `ticker-{cryptocurrency}`

Where:

* **{cryptocurrency}** is a [primary currency code](https://www.independentreserve.com/API#GetValidPrimaryCurrencyCodes)

E.g: `orderbook-xbt`, `ticker-xbt`

Subscribing to a channel can be done in two ways:

1. Supplying channel names with the `subscribe` query string when opening a connection. Eg: `wss://websockets.independentreserve.com?subscribe=orderbook-xbt,ticker-xbt` will subscribe to order book XBT and ticker XBT events
2. Sending a subscribe message once the connection is established. The subscribe message should set the **data** property to a string array with the channels to subscribe to.

### Subscribe message format

```json
{
    "Event":"Subscribe",
    "Data":["orderbook-xbt", "ticker-xbt"]
}
```

## Unsubscribing from Channels

Unsubscribing from a channel is done using an **Unsubscribe** message with the **data** property listing the channels to unsubscribe from.

### Unsubscribe message format

```json
{
    "Event":"Unsubscribe",
    "Data":["orderbook-xbt", "ticker-xbt"]
}
```

## Subscription Confirmation Event

Successful subscriptions and unsubscriptions are notified via the **Subscriptions** event. The **Data** property will list current subscriptions.

### Subscriptions event

```json
{
    "Event":"Subscriptions",
    "Data":["orderbook-xbt", "ticker-xbt"],
    "Time":1689718230428
}
```

## Orderbook Channel

The order book channel provides real-time order book updates. Order created, modified, and canceled events are published on the order book channel.

### NewOrder - Order Created Event

The NewOrder event is published when a limit order is placed on the order book.

```json
{
    "Channel":"orderbook-eth",
    "Nonce":28,
    "Data":{
        "OrderType":"LimitBid",
        "OrderGuid":"dbe7b832-b9b7-4eac-84ce-9f49c2a93b87",
        "Price":{
            "aud":2500,
            "usd":1816.5,
            "nzd":2587.5,
            "sgd":2453
        },
        "Volume":1
    },
    "Time":1689718320139,
    "Event":"NewOrder"
}
```

### OrderChanged - Order modified Event

The OrderChanged event is published with updated volume when an order is filled or partially filled due to a trade.

```json
{
    "Channel":"orderbook-eth",
    "Nonce":30,
    "Data":{
        "OrderType":"LimitBid",
        "OrderGuid":"e64fb6e2-a9f8-4f52-95e7-5a3c7a9f8f53",
        "Volume":0.09646808
    },
    "Time":1689718320138,
    "Event":"OrderChanged"
}
```

The only value that can be updated is in a change event is `Volume` which represents the unfilled amount remaining. Fully filled orders will have `"Volume":0`

### OrderCanceled - Order Cancelled Event

The OrderCanceled event is published when an order is canceled.

```json
{
    "Channel":"orderbook-eth",
    "Nonce":29,
    "Data":{
        "OrderType":"LimitBid",
        "OrderGuid":"dbe7b832-b9b7-4eac-84ce-9f49c2a93b87"
    },
    "Time":1689719167720,
    "Event":"OrderCanceled"
}
```

## Ticker Channel

The ticker channel provides realtime trade updates. Trade events are published on the ticker channel.

### Trade Event

Trade events are published on every trade.

```json
{
    "Event":"Trade",
    "Channel":"ticker-xbt",
    "Nonce":1,
    "Time":1689718320138,
    "Data":{
        "TradeGuid":"c5bde544-d8ae-4e38-9e90-405a3f93b6d6",
        "TradeDate":"2009-01-03T18:15:05.9321664+00:00",
        "Volume":50.0,
        "Price":{
            "Aud":3313.59,
            "Sgd":4798.92
        },        
        "BidGuid":"ebbeca4b-7148-4230-ad8f-833a3ccf35c2",
        "OfferGuid":"ad5ece89-083b-49fc-8bc1-bdb7482a9b9a",
        "Side":"Buy"
    }
}
```

**Side** values can be: `Buy` or `Sell`. The **BidGuid** and **OfferGuid** fields match those published in the orderbook channel.

## Nonce

Each event on a channel will have a nonce. A nonce is an increasing integer value for each channel. A nonce will increase by exactly 1 with every published event on the channel. An increase of more than 1 from the previous event indicates a dropped event. A nonce less than the previous event indicates a reset channel. In both cases it's advised to have logic to ensure the subscriber is in the correct state.
Nonces are assigned per channel, per crypto currency.

##Time
Each event contains a `Time` property. It is the unix time of the event in milliseconds.

## Heartbeat

A heartbeat event is published every 60 seconds. This interval may change in the future.

```json
{
    "Event":"Heartbeat",
    "Time":1689719100201
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

```javascript
// Basic example
var webSocket = new WebSocket('wss://websockets.independentreserve.com/?subscribe=orderbook-xbt');
webSocket.onmessage = function (event) {
    console.log(event);
}
```

Also see [/samples/JavaScript](https://github.com/independentreserve/websockets/tree/master/samples/JavaScript) including the
[live demo of XBT Orderbook Events](https://independentreserve.github.io/websockets/samples/JavaScript/orderbook.html).