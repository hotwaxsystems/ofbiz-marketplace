# OFBiz Marketplace

A plugin for Apache OFBiz that connects marketplace shops to OFBiz: TikTok Shop, eBay, Walmart and Amazon.
TikTok Shop comes first and is the template for the others.

## How it works

- A shop is connected when its seller approves the app on the marketplace. Tokens are stored encrypted.
- Every notice from a marketplace and every call to it is saved first, with its state, tries and errors. A notice seen twice is handled once.
- Orders wait while the marketplace still lets the buyer change them, then go to the order service named in data.
- Cancellations go both ways. Shipments send carrier and tracking back. Stock levels and returns follow.
- Every marketplace follows one shared contract: authorize, check notices, fetch and list orders, cancel, ship, set stock, fetch returns.
- Each workflow can be turned on or off per shop.
- It does not change OFBiz core code.

## Install

Clone this repository into the `plugins` folder of an OFBiz trunk checkout:

```
cd ofbiz-framework/plugins
git clone https://github.com/hotwaxsystems/ofbiz-marketplace.git marketplace
```

## Contributing

Every change goes through a pull request.

## License

Apache License 2.0. See `LICENSE` and `NOTICE`.
