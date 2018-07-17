## Independent Reserve - WebSockets
For more details on Independent Reserve's WebSockets API, please refer to [API Documentation](https://www.independentreserve.com/api#websockets)

## How to connect
WebSocket base url: `wss://websockets.independentreserve.com`

Trade Events Channel: `/ticker-{cryptocurrency}-{fiatcurrency}`

Orderbook Events Channel: `/orderbook-{cryptocurrency}-{fiatcurrency}`

Where **{cryptocurrency}** is a [primary currency code](https://www.independentreserve.com/API#GetValidPrimaryCurrencyCodes) and **{fiatcurrency}** is a [secondary currency code](https://www.independentreserve.com/API#GetValidSecondaryCurrencyCodes).

## Nonce
Each event on a channel will have a nonce. A nonce is an increasing integer value for each channel. A nonce will increase by exactly 1 with every published event on the channel. An increase of more than 1 from the previous event indicates a dropped event. A nonce less than the previous event indicates a reset channel. In both cases it's advised to have logic to ensure the subscriber is in the correct state.

## orderbook-{cryptocurrency}-{fiatcurrency]
The orderbook channel provides realtime order book updates. Order created, cancelled and modified events are published on the orderbook channel.

`wss://websockets.independentreserve.com/orderbook-xbt-aud`

### NewOrder - Order Created Event
The NewOrder event is published when a limit order is placed on the order book. The PrimaryCurrencyCode will always match {cryptocurrency} of the channel and the SecondaryCurrencyCode will always match {fiatcurrency} of the channel.
```json
{
    "Event":"NewOrder",
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
The OrderChanged event is published with updated volume when an order is filled or partially filled due to a trade. Fully filled orders will have "Volume":0
```json
{
    "Event":"OrderChanged",
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
The only value that can be updated is `Volume`

### OrderCanceled - Order Cancelled Event
The OrderCanceled event is published when an order is cancelled.
```json
{
    "Event":"OrderCanceled",
    "Data":{
        "OrderGuid":"fa091562-4101-46de-8d66-aeddbeb8795b",
        "PrimaryCurrencyCode":"Xbt",
        "SecondaryCurrencyCode":"Aud",
        "OrderType":"LimitBid",
        "Nonce":3
        }
}
```

## ticker-{cryptocurrency}-{fiatcurrency}
The ticker channel provides realtime trade updates. Trade events are published on the ticker channel.

`wss://websockets.independentreserve.com/ticker-xbt-aud`

### Trade Event
Trade events are published on every trade. Note: The SecondaryCurrencyCode of a trade event will be the currency the trade is executed in and not the {fiatcurrency} of the channel it is published on. Eg: ticker-xbt-aud will publish trade events with PrimaryCurrencyCode:"Xbt" and SecondaryCurrencyCode:"Aud", "Usd" or "Nzd"
```json
{ 
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
```
**Side** values can be: `Buy` or `Sell`. The **BidGuid** and **OfferGuid** fields match those published in the orderbook channel.

## Troubleshooting
### Getting 404 Status Code
Make sure you are using the correct channel name. Eg for XBT (Bitcoin) AUD (Australian Dollar) orderbook events use `wss://websockets.independentreserve.com/orderbook-xbt-aud`. For ETH (Ethereum) USD (US Dollar) orderbook events use `wss://websockets.independentreserve.com/ordebook-eth-usd`

## Samples

### JavaScript
JavaScript native WebSocket support:

```javascript
var webSocket = new WebSocket('wss://websockets.independentreserve.com/orderbook-xbt-aud');
webSocket.onmessage = function (event) {
	console.log(event);
}
```

JavaScript example: [Source Code](https://github.com/independentreserve/websockets/tree/master/samples/JavaScript)

JavaScript example: [Demo - Live XBT-AUD Orderbook Events](https://independentreserve.github.io/websockets/samples/JavaScript/orderbook-xbt-aud.html)

