# Order Book Snapshot

## Overview

The `Order Book Snapshot` protocol is for advanced users who wish to maintain a verifiable, continuously updated, client side mirror of the Independent Reserve order book to facilitate trading operations.

## Core concepts

*Order book snapshot* of a given *market depth* `x` consists of two ordered sets of *price levels*:

- `Bids`: top `x` price levels of bids in descending order;
- `Offers`: top `x` price levels of offers in ascending order.

*Price level* entity combines all orders (bids and offers separately) with the same *price* into a single entity with the following attributes:

- `Price`: the common price of all combined orders;
- `Volume`: the sum of the volumes of all combined orders, i.e. the sum of volumes of all bids or offers of a given `Price`.

NB: Order book snapshot always consists of price levels with unique prices (no duplications) which are also sorted in accordance with their order type (bids or offers separately).

## Protocol

WebSocket Order Book API allows clients to easily maintain on their side any number of up to date order books of any market depth updated in real-time.  
The API uses the following protocol:

- Upon subscription the server first sends an `OrderBookSnapshot` message which contains an entire order book snapshot at the requested depth. A client uses this message to initialize their copy of order book snapshot;
- Afterward the server starts sending `OrderBookChange` messages for every order book update which affects the order book snapshot of the requested depth. This is a concise message which contains only the delta (difference) which, when applied to the client's copy of order book snapshot brings it up to date with the latest server order book. Each `OrderBookChange` message represents an atomic change and can contain one or several order book snapshot price level changes of two kinds:

    - *Price level added*: a new price level which should be inserted into client's order book snapshot;
    - *Price level deleted*: existing price level which should be deleted from client's order book snapshot.

The server sends `OrderBookChange` messages in real-time for every order book update but only if such update affects the client's order book subscription. For example if a client is subscribed to market depth `10` for `btc-aud` order book, it will only receive messages which affect top 10 price levels.

## Usage

### Initial subscription

URL format for initial subscription:

```console
wss://{server}/orderbook[/{depth}][?subscribe={subscriptions}][&heartbeat={heartbeat-interval}][&buffer={buffer-size}]
```

Where:

- `{depth}`: (optional) order book market depth. It can be any positive number, default is `10`;
- `{subscriptions}`: (optional) comma separated list of *subscription tokens*: currencies or currency pairs for initial subscription. The list can be empty (default). Each element of the list can be:
    - Currency pair in the format of `{primary}-{secondary}`: subscription to a single order book snapshot;  
    E.g.: `btc-aud`, `eth-usd`, `doge-nzd`
    - Primary currency ticker in the format of `{primary}`: subscription to all secondary currencies for a given primary currency;  
    E.g.: `ada`, `ltc`, `usdc`
    - `all` to subscribe to all trading currency pairs.  
    E.g.: `all`
- `{heartbeat-interval}`: (optional) [heartbeat](#heartbeat-message) interval in seconds. Use `0` to disable the heartbeat. By omitting this parameter, a server configured value will be used;
- `{buffer-size}`: (optional) internal message buffer size used to queue messages for slow clients (See _reasons why a CRC32 mismatch may happen_ below).

Example:

```console
wss://websockets.independentreserve.com/orderbook/100?subscribe=btc,eth-aud
```

The server will continue sending WebSocket messages until a client either closes the connection or unsubscribes without closing it using `Unsubscribe` WebSockets message.

### WebSockets subscription updates

After initial subscription a client can modify subscriptions without closing the existing connection using WebSocket messages sent to the server:  

- `Subscribe`: use this message to subscribe to a new currency or currency pair using the format described [above](#initial-subscription). The server will only process subscription tokens that  a client is not currently subscribed to. Any token which a client is already subscribed will be ignored. Similar to the initial subscription, the API will first send `OrderBookSnapshot` message and then continue sending `OrderBookChange` messages;
- `Unsubscribe`: use this message to unsubscribe from any currently subscribed currency. Upon receiving this message API will stop sending `OrderBookChange` messages for the affected currencies. The server will only process unsubscriptions for which a client is currently subscribed to, any token which a client is not currently subscribed to will be ignored.

## Message formats

### OrderBookSnapshot message

This is the first message which the server sends to a client after subscription and contains the entire order book snapshot at requested marked depth.  

The message format:

```json
{
  "Channel": "orderbook/{depth}/{primary}/{secondary}",
  "Data": {
    "Bids": [
      ... // array of bids of length <= n
    ],
    "Offers": [
      ... // array of offers of length <= n
    ],
    "Crc32": n  // CRC32 value of top 10 bids and offers (positive number)
  },
  "Time": t, // unix time of the event in ms (positive number)
  "Event": "OrderBookSnapshot"
}  
```

where:

- `{depth}` is market depth (positive number);
- `{primary}` is primary currency ticker;
- `{secondary}` is secondary currency ticker;
- `Bids` and `Offers` both contain arrays of sorted unique price levels in the following format:

  ```json
  {
    "Price": p, // price level price (positive fractional number)
    "Volume": v // price level volume (positive fractional number)
  },
  ```

Example of `OrderBookSnapshot` message for market depth `5` and subscription `btc-aud`:

```json
{
  "Channel": "orderbook/5/btc/aud",
  "Data": {
    "Bids": [
      {
        "Price": 31802.46,
        "Volume": 0.25
      },
      {
        "Price": 31802.45,
        "Volume": 0.32464684
      },
      {
        "Price": 31802.42,
        "Volume": 0.34465528
      },
      {
        "Price": 31785.01,
        "Volume": 2.733
      },
      {
        "Price": 31785,
        "Volume": 1.5
      }
    ],
    "Offers": [
      {
        "Price": 31844.99,
        "Volume": 0.30740328
      },
      {
        "Price": 31845,
        "Volume": 1.5
      },
      {
        "Price": 31865.3,
        "Volume": 0.2
      },
      {
        "Price": 31875,
        "Volume": 1.5
      },
      {
        "Price": 31875.9,
        "Volume": 0.788
      }
    ],
    "Crc32": 2893776693
  },
  "Time": 1660895883834,
  "Event": "OrderBookSnapshot"
}  
```

### OrderBookChange message

This type of message is sent to a client after initial `OrderBookSnapshot` message and each message of this type contains incremental atomic order book snapshot update.  

The message format:

```json
{
  "Channel": "orderbook/{depth}/{primary}/{secondary}",
  "Data": {
    "Bids": [
      ... // array of bids changes
    ],
    "Offers": [
      ... // array of offers changes
    ],
    "Crc32": n  // CRC32 value of top 10 bids and offers (positive number)
  },
  "Time": t, // unix time of the event in ms (positive number)
  "Event": "OrderBookChange"
}
```

where either `Bids` or `Offers` will contain a set of changes constituting a single atomic update. Such an update consists of one or more individual price level changes in the following format:

- Price level added:
  
  ```json
  {
    "Price": p, // new price level price (positive fractional number)
    "Volume": v // new price level volume (positive fractional number)
  }
  ```

- Price level deleted (marked by `"Volume": 0`):

  ```json
  {
    "Price": p, // existing price level price (positive fractional number)
    "Volume": 0 // 0 volume indicates that the price level with price "p" was deleted
  }
  ```

Examples of `OrderBookChange` messages:

- New price level `31844.98` was added to offers of order book snapshot `btc-aud` of marked depth `5`:
  
  ```json
  {
    "Channel": "orderbook/5/btc/aud",
    "Data": {
      "Bids": [],
      "Offers": [
        {
          "Price": 31844.98,
          "Volume": 0.02396605
        }
      ],
      "Crc32": 263206970
    },
    "Time": 1660895884514,
    "Event": "OrderBookChange"
  }
  ```

- Price level `1718.78` was deleted and new price level `1714.62` was added to bids of order book snapshot `eth-usd` of marked depth `10`:

  ```json
  {
    "Channel": "orderbook/10/eth/usd",
    "Data": {
      "Bids": [
        {
          "Price": 1718.78,
          "Volume": 0
        },
        {
          "Price": 1714.62,
          "Volume": 2.2688
        }
      ],
      "Offers": [],
      "Crc32": 2276313494
    },
    "Time": 1660897884613,
    "Event": "OrderBookChange"
  }
  ```

- A more involved example: Price level `2334.26` volume was adjusted to `18.40609027` (by removing it and immediately adding an adjusted one) and price level `2146.32`  added. Such an update could happen if one of the orders which constitutes such a price level is partially traded or modified. This was demonstrated in an order book snapshot subscription to `eth-aud` and marked depth `50`:

  ```json
  {
    "Channel": "orderbook/50/eth/aud",
    "Data": {
      "Bids": [
        {
          "Price": 2334.26,
          "Volume": 0
        },
        {
          "Price": 2146.32,
          "Volume": 0.01
        },
        {
          "Price": 2334.26,
          "Volume": 18.40609027
        }
      ],
      "Offers": [],
      "Crc32": 1814645097
    },
    "Time": 1661138525620,
    "Event": "OrderBookChange"
  }
  ```

### Heartbeat message

To maintain a WebSocket connection in absence of order book updates, the server will periodically send `Heartbeat` messages in the following format:

```json
{
  "Time": t, // unix time of the event in ms (positive number)
  "Event": "Heartbeat"
}
```

E.g.:

```json
{
  "Time": 1661145843028,
  "Event": "Heartbeat"
}
```

### WebSocket subscription update messages

It is possible to modify a set of current subscriptions without closing existing the WebSocket connection. To do that a client should send a WebSocket message to the server using one of the following formats:

- To subscribe/re-subscribe a client should send the following WebSocket message using the existing WebSocket connection:

  ```json
  {
    "Event": "Subscribe",
    "Data": [
      ... // array of subscription tokens to subscribe to / re-subscribe to
    ]
  }  
  ```  

  where array of subscription is an array of subscription string tokens in the format used for initial subscription.    
  E.g.:

  ```json
  {
    "Event": "Subscribe",
    "Data": ["btc-nzd","eth"]
  }  
  ```  

- To unsubscribe from a currently subscribed order book use:

  ```json
  {
    "Event": "Unsubscribe",
    "Data": [
      ... // array of subscription tokens to unsubscribe from
    ]
  }  
  ```

  E.g.:

  ```json
  {
    "Event": "Unsubscribe",
    "Data": ["btc-aud","eth-sgd"]
  }  
  ```  

### CRC32 checksum

Both `OrderBookSnapshot` and `OrderBookChange` messages include a `Crc32` field which contains the CRC32 checksum value calculated by API for the top `10` bids and offers of its version of order book snapshot at the moment when the message was sent. Clients can use the value of this field to check if their version of order book snapshot matches the server version. To do this a client needs to calculate the CRC32 of their version of the order book snapshot (after changes from `OrderBookChange` have been applied) and compare the result with the `Crc32` field value of the applied message.  

1. To calculate checksum, take the top 10 bids and offers. If there is less than 10 bids or offers use all bids or offers that are present. The ordering of orders is important, the orders must be processed in the order they appear in order book snapshot, i.e. with bids in descending order and offers in ascending;
1. Both `Price` and `Volume` are used for the calculation and to do it correctly both of them have to be converted to a string with precision of exactly `8` digits after the decimal point, e.g. using `toFixed(8)` in JavaScript or `ToString("F8")` in C#;
1. For each order:
    1. Remove the decimal character, '.', from the price, i.g. `0.05000000` -> `005000000`;
    1. Remove all leading zero characters from the price. i.g. `005000000` -> `5000000`;
    1. Add the formatted price string to the concatenation;
    1. Repeat above steps but for the volume.
1. Feed the concatenated string as input to a CRC32 checksum function, storing the result;
1. Cast the result (comprising 32 bits) as an unsigned 32-bit integer. This value can now be compared to the checksum received to ensure your local book is accurate.

## How to maintain a client side order book snapshot

In order to maintain a correct, up to date order book snapshot on the client side after subscription do the following:

1. When the `OrderBookSnapshot` message is received, replace the entire order book snapshot using the content of this message;  
   NB: The `OrderBookSnapshot` message is always the first message sent by API after a successful subscription (one message per subscribed currency pair) but it can also be sent by API from time to time later (for example in the event of an FX rate change on the server) so a client must handle subsequent `OrderBookSnapshot` messages exactly like the initial `OrderBookSnapshot` message: by replacing their order book copy from the content of the received `OrderBookSnapshot` message;
1. When an `OrderBookChange` message is received, all bid or offer changes (there will be at least one but it might contain several) must be applied to the client copy of order book snapshot atomically in the following way:
   1. Price level deleted change, i.e. the change which has `"Volume": 0` is handled by finding corresponding price level in client's order book using unique `Price` and removing this price level from the client's copy;
   1. Price level added change, i.e. the change which has non-zero `Volume` and `Price` is handled by inserting this change as a new price level into client order book copy. Since both bids and offers are sorted and always have unique prices, a new price level must be inserted:
      1. In `Bids` array, which is sorted in descending order, immediately before the first bid which has lower `Price` or at the end of the array if no such bid was found;
      1. In `Offers` array, which is sorted in ascending order, immediately before the first offer which has higher `Price` or at the end of the array if no such offer was found.
1. After all changes are applied if the size of the `Bids` or `Offers` arrays is larger than the marked depth the client subscribed to, excess bids or offers (i.e. below subscribed marked depth) must be removed from the end of arrays so the sizes of both arrays are always equal or less than the subscribed marked depth;  
     NB: Both bids and offers arrays can have less orders than the subscribed marked depths if the actual server order book contains less total bids or offers than the subscribed marked depth.
1. As a last step, the CRC32 must be recalculated using the updated copy of the client order book snapshot using the algorithm described above and the resulting CRC32 value should be compared with the `Crc32` field value from the current `OrderBookChange` message being processed:
     1. If the values match we are done with this `OrderBookChange` and are ready to process the next message;
     1. If the values do not match it means that the client's version must be different from the server version. To fix this situation a client can unsubscribe and immediately subscribe to this particular order book currency pair by first sending a WebSocket `Unsubscribe` message with the affected token and then send a `Subscribe` message with the same token. In response the server will send an `OrderBookSnapshot` message containing the latest order book snapshot with subsequent `OrderBookChange` messages which should now have matching CRC32 values if applied correctly.  
     NB: There are several reasons why a CRC32 mismatch may happen. Apart from incorrect client implementations those are:
        - An `OrderBookChange` message has been lost in transmission between the server and a client (unlikely);
        - Slow client: if a client cannot keep up with the number of `OrderBookChange` messages generated by the server, the latter queues pending messages up to a certain point (e.g. `1000` messages per WebSocket connection). When the queue hits its limit the server has no other option but to drop new `OrderBookChange` messages which will result in a CRC32 mismatch due to missing `OrderBookChange` updates.
