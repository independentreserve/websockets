# Independent Reserve - Web Sockets

Independent Reserve supports several web socket protocols for streaming order book and trade events.

## Order Book Ticker

The `Order Book Ticker` channels are for receiving a stream of `order` and/or `trade` events from the Independent Reserve server.

See [orderbook-ticker.md](orderbook-snapshot.md) for details

## Order Book Snapshot

The `Order Book Snapshot` protocol is for advanced users who wish to maintain a continuously updated, client side mirror of the Independent Reserve order book to facilitate trading operations.

See [orderbook-snapshot.md](orderbook-snapshot.md) for details

## Web API

For more details on Independent Reserve's HTTP (non web socket) API, please refer to our [API Documentation](https://www.independentreserve.com/api)
