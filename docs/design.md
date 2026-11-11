# Marketplace plugin design

How the marketplace plugin connects TikTok Shop, eBay, Walmart and Amazon to Apache OFBiz trunk.
TikTok Shop comes first and is the template for the others.

## How an order flows

1. The marketplace sends a notice that an order was placed or changed.
2. The plugin checks the notice is genuine and saves it as a marketplace message.
3. The plugin fetches the full order from the marketplace and turns it into the common order.
4. If the shop has a hold time, the order waits. The marketplace's own hold also counts, such as TikTok's one-hour buyer window.
5. When the hold ends, the plugin calls the order handler named in data. A warehouse application, for example, names its order service.
6. A regular check lists orders changed since the last check. It finds any order whose notice was lost.

Example: a TikTok buyer places order 5761. The notice arrives and is saved. The plugin fetches the order and holds it until the buyer window ends. Then it calls the order handler, which records order 1090 with `externalId` 5761.

## Records

### From trunk, used as they are

| Record | Use |
| --- | --- |
| `OrderHeader.externalId`, `salesChannelEnumId` | The marketplace order id and the marketplace. Trunk already indexes `externalId`. |
| `OrderItem.externalId` | The marketplace line id, needed to ship and cancel single lines |
| `GoodIdentification` | A product's id on a marketplace, one identification type per marketplace |
| `Enumeration` | Marketplaces, workflows and message types |
| `StatusItem`, `StatusValidChange` | Shop and message statuses |
| `CustomMethod` | Names the handler service for each incoming message type, and each marketplace's services |
| `JobSandbox` | The regular check, release, stock and retry jobs |

### Added by the plugin

`MarketplaceShop`, one connected shop:

| Field | Meaning |
| --- | --- |
| `marketplaceShopId` | Id (key) |
| `marketplaceEnumId` | `MKT_TIKTOK`, `MKT_EBAY`, `MKT_WALMART`, `MKT_AMAZON` |
| `externalShopId` | The shop's id on the marketplace, such as TikTok's shop cipher or Amazon's seller id |
| `region` | Such as `US` or `CA` |
| `productStoreId` | The OFBiz store the shop feeds |
| `holdMinutes` | Extra wait before an order is released |
| `statusId` | Connected, needs approval again, stopped |
| `lastOrderCheckDate` | Where the regular check starts next time |

`MarketplaceShopWorkflow` turns one workflow on or off for a shop:

| Field | Meaning |
| --- | --- |
| `marketplaceShopId`, `workflowEnumId` | Key. Workflows: orders, cancellations, shipments, stock, returns |
| `enabled` | `Y` or `N` |

`MarketplaceMessage`, every notice in and call out:

| Field | Meaning |
| --- | --- |
| `marketplaceMessageId` | Id (key) |
| `marketplaceShopId` | The shop |
| `messageTypeEnumId` | Such as order placed, order cancelled, shipment sent, stock sent |
| `directionEnumId` | In or out |
| `externalMessageId` | The marketplace's notice id. Unique with the shop, so a notice seen twice is handled once |
| `externalOrderId` | The marketplace order id, when there is one |
| `body` | The notice or the call, without tokens |
| `statusId` | Received, held, handled, sent, failed |
| `releaseDate` | When a held order may go |
| `tryCount`, `nextTryDate`, `lastError` | Retries |

Each marketplace adds a login record keyed by the shop, such as `MarketplaceShopTikTok`. Tokens use `encrypt="true"`.

## Services

### Called by applications

| Service | In | Out |
| --- | --- | --- |
| `startMarketplaceAuthorization` | `marketplaceEnumId`, `productStoreId`, `region` | `marketplaceShopId`, `authorizationUrl` |
| `completeMarketplaceAuthorization` | `marketplaceEnumId`, the code and state the marketplace returns | `marketplaceShopId`, `statusId` |
| `checkMarketplaceShop` | `marketplaceShopId` | `passed`, the marketplace's reason |
| `cancelMarketplaceOrder` | `orderId`, the lines, a reason | `marketplaceMessageId` |
| `confirmMarketplaceShipment` | `shipmentId` | `marketplaceMessageId` |
| `pushMarketplaceStock` | `marketplaceShopId`, a list of product and quantity | the products that failed |
| `retryMarketplaceMessage` | `marketplaceMessageId` | `statusId` |

`confirmMarketplaceShipment` reads the shipment's order, lines, carrier and tracking. A shipment with only some lines is sent as a split shipment.

### Jobs and notice intake

| Service | What it does |
| --- | --- |
| `receiveMarketplaceNotice` | Takes a notice, checks it is genuine, saves it, and answers the marketplace at once |
| `processMarketplaceMessages` | Fetches orders for saved notices, holds them, and calls the handlers |
| `checkMarketplaceOrders` | The regular check. Lists orders changed since `lastOrderCheckDate` |
| `releaseHeldMarketplaceOrders` | Calls the order handler for orders whose hold has ended |
| `syncMarketplaceStock` | Sends stock in three tiers: after each change, every 15 minutes, and a full check every hour |

Each job runs one at a time, with `semaphore="wait"`. Each shop runs in its own transaction.

Amazon sends its notices to an AWS SQS queue. A message queue channel reads it, and its message type handler is `receiveMarketplaceNotice`.

### Handlers named in data

| Incoming message | Handler, example for a warehouse application |
| --- | --- |
| Order ready | its order service |
| Order cancelled by the buyer | its cancel or pull-back service |
| Return started | its expected return service |

### Contract every marketplace follows

| Interface | Does |
| --- | --- |
| `marketplaceAuthorizeInterface` | Builds the approval link, and turns the returned code into tokens |
| `marketplaceRefreshTokenInterface` | Gets a new access token before the old one expires |
| `marketplaceVerifyNoticeInterface` | Checks a notice's signature over its raw body |
| `marketplaceFetchOrderInterface` | Fetches one order as the common order |
| `marketplaceListChangedOrdersInterface` | Lists orders changed since a time |
| `marketplaceCancelInterface` | Cancels an order or some lines |
| `marketplaceShipInterface` | Sends carrier, tracking and lines |
| `marketplaceSetStockInterface` | Sets sellable quantities, in groups |
| `marketplaceFetchReturnsInterface` | Lists returns started since a time |
| `marketplaceCheckShopInterface` | Tests the tokens and the shop |

### The common order

The common order is one JSON shape for all marketplaces: the order, its lines, the ship-to address, the shipping method and the marketplace references.
An application can give its order service the same input from every way an order arrives.

## Rules

- Save every notice before anything else happens to it.
- Answer a notice at once. Do the work after, so the marketplace does not send it again.
- Never release an order the marketplace says is still in the buyer's window.
- Never write a token to a log, a message or a test file.
- Echo guard: a cancellation OFBiz sent is not handled again when the marketplace sends it back.
- Back off when a marketplace says "too many requests", with growing waits.
- Refresh tokens before they expire. A token that cannot be refreshed sets the shop to "needs approval again".

## Security

- Services need the plugin's `MARKETPLACE` permission.
- Notice and callback addresses need no login, so each one checks the marketplace's signature or the state value it gave out.
- Nothing in the plugin knows who owns a shop. An application that serves many sellers, such as a warehouse for many merchants, adds that itself.

## Tests

- Each marketplace has a local copy for tests, built from its own test site and saved replies.
- Saved real responses, with made-up shops and buyers.

## Learn from

"Use" means the code may be used or adapted under its license. "Ideas only" means read it, but write our own code.

| Source | What to take | License | How |
| --- | --- | --- | --- |
| [amzn/selling-partner-api-models](https://github.com/amzn/selling-partner-api-models) | Amazon's exact API definitions | Apache 2.0 | Use |
| [amzn/selling-partner-api-sdk](https://github.com/amzn/selling-partner-api-sdk) | Amazon's official Java client, login and token handling | Apache 2.0 | Use |
| [amzn/selling-partner-api-samples](https://github.com/amzn/selling-partner-api-samples) | Order notices through SQS with a dead-letter queue; multi-channel fulfillment, AWD and label recipes | MIT-0 | Use |
| [saleweaver/python-amazon-sp-api](https://github.com/saleweaver/python-amazon-sp-api) | How a widely used client handles paging, rate limits and Restricted Data Tokens | MIT | Use |
| [EcomPHP/tiktokshop-php](https://github.com/EcomPHP/tiktokshop-php) | TikTok request signing, approval flow, notice signatures, and the order, fulfillment, stock and return calls | Apache 2.0 | Use |
| [whitebox-co/walmart-marketplace-api](https://github.com/whitebox-co/walmart-marketplace-api) | Walmart login, orders, shipping, stock, returns and notices | MIT | Use |
| [eBay/ebay-oauth-java-client](https://github.com/eBay/ebay-oauth-java-client) | eBay's official Java login client | Apache 2.0 | Use |
| [ledoent/connector-amazon](https://github.com/ledoent/connector-amazon) (Odoo) | One add-on per workflow: orders, stock, returns, labels, FBA, MCF, fees | AGPL | Ideas only |
| [frappe/ecommerce_integrations](https://github.com/frappe/ecommerce_integrations) (ERPNext) | Settings per shop, order and fee import; what breaks without Amazon address access | GPL | Ideas only |

Each marketplace's own developer documents are the first source for every call.

## Points for the design review

1. Notice address: trunk's REST reads the body as parsed JSON. A signature over the raw body may need a plain web request in the plugin's webapp.
2. Marketplace message: its own record, or the message queue plugin's message record.
3. Marketplace product ids: one `GoodIdentification` type per marketplace, or a separate listing record for marketplaces with several ids per product, such as Amazon's ASIN and seller SKU.
4. Stock push: an application may leave it off when another system already sends stock levels to the marketplace.
