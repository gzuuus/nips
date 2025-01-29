NIP-XX
======

Marketplace Protocol
---------------------------

`draft` `optional`

// TODO: Mention the optionality of the different components used in this nip.

This NIP defines a comprehensive protocol for implementing decentralized marketplaces on Nostr, combining and enhancing the approaches from [NIP-15](15.md) and [NIP-99](99.md). It provides a complete e-commerce framework while maintaining the protocol's simplicity and interoperability.

The focus for NIP-99 is on the product listing but this NIP also provides a structure for other necessary flows for a complete checkout procedure:

1) Order Communication Flow 
2) Shipping Calculation Flow
   This will be it's own kind ( replaceable event ) for updates during the shipping process, pricing schema for shipping costs and general shipping settings: Examples are zoning restrictions: USA, EU, NA, and free shipping: 50 USD
3) Payment Flows

A general checkout implementation can be structured as followed:
1) Items are gathered in a Local Basket 
2) Shipping details are required to calculate a full payment amount
3) Order gets send once Payment is received ( Zap Receipt , Nut Receipt, Marketplace server )

In practice a shop can have a preferred checkout and shipping option defined at the store level and product level. A option at the store level will be used if nothing is defined at the product level, essentially making it that product level shipping and payment options always overrule store level settings.

## Events and Kinds

### Product Listing (Kind: 30402)
Following [NIP-99](99.md)'s schema for product representation:

```jsonc
{
  "kind": 30402,
  "created_at": <unix timestamp>,
  "content": "<product description in markdown>",
  "tags": [
    ["d", "<product identifier>"],
    ["title", "<product title>"],
    ["price", "<amount>", "<currency>", "<optional frequency>"],
    // Optional tags
    ["image", "<url>", "<dimensions>"],
    ["summary", "<short description>"],
    ["stock", "<integer>"], // Determines the amount of available stock
    ["shipping", "30406:<pubkey>:<d-tag>"], // References to shipping options
    ["shipping", "30405:<pubkey>:<d-tag>"], // References to a product collection, in this case, shipping is inherited from the collection
    ["type","<simple | variable | variation>", "<digital | physical>"], // Determines whether a product is a simple product, a variable product, or a variation of a variable product. The third element of the array determines whether the product is digital or physical. Default value (or if omitted): "simple and digital".
    ["visivilty","<hidden | on-sale | pre-order>"], // Determines how the product should be displayed, default value (or if omitted): "on-sale
    ["spec", "<spec-key>", "<spec-value>"], // E.g. spec-key: screen-size, spec-value: 21"
    ["weight","<weight-value>", "<weight-measure-unit>"], // Weight units should follow ISO 80000-1 standard
    ["dim","<dim-value>","<dim-measure-unit>"] // Dimension value should be expressed as "length x width x height", e.g. "3x3x3", using the "x" character as a delimiter. Units should be "mm", "cm", "m", following the ISO 80000-1 standard.
    ["location", "<location string>"],
    ["g", "<geo hash>"],
    ["t", "<category>"],
    ["a", "<30405>:<pubkey>:<d-tag>"] // Reference to product collection if applicable
  ]
}
```

#### Notes
- Products are the highest level item in a market place
- You can define the shipping option by referencing a shipping event, or a product collection, in this last ase the shipping options should be inherited from the collection and merge with the other shippings defined in the product if they exist

### Product Collection (Kind: 30405)
Using NIP-51 list format for grouping products:

```jsonc
{
  "kind": 30405,
  "created_at": <unix timestamp>,
  "content": "<optional collection description>",
  "tags": [
    ["d", "<collection identifier>"],
    ["name", "<collection name>"],
    ["a", "30402:<pubkey>:<d-tag>"], // Product references
    // Optional tags
    ["image", "<collection image>"],
    ["summary", "<collection description>"],
    ["location", "<location string>"],
    ["g", "<geo hash>"],
    ["shipping", "30406:<pubkey>:<d-tag>"], // References to shipping options
    ["currency", "<collection-currency>"] // Currency codes MUST follow the ISO 4217
  ]
}
```

### Drafts
Users may want to save products or collections as private drafts before they are publicly visible, or while they are working on the details. To achieve this, clients MUST follow the [nip-37](https://github.com/nostr-protocol/nips/blob/master/37.md)

### Shipping Option (Kind: 30406)

This event type defines shipping methods, costs, and constraints. To ensure reliable tag association, each physical pickup location should be defined in a separate event. These events can be published by the merchant or a third-party provider, and can be subscribed to by the merchant. This approach allows merchants to easily define their shipping options manually, or reference shipping options published by a third-party provider, such as a delivery company, a DVM, etc.

```jsonc
{
  "kind": 30406,
  "created_at": <unix timestamp>,
  "content": "<optional shipping description>",
  "tags": [
    ["d", "<shipping identifier>"],
    ["name", "<shipping method name>"],
    ["price", "<base_cost>", "<currency>"],
    ["country", "<ISO 3166-1 alpha-2 country code>"],  // Can be repeated for multiple countries
    ["region", "<ISO 3166-2 region code>"],         // Optional subdivision within country
    ["service", "<service-type>"],                  // e.g., "standard", "express", "overnight", "pickup"
    ["duration", "<min-hours>", "<max-hours>"],     // Estimated delivery window
    
    // Optional tags    
    ["location", "<pickup location description>"],   // Physical address
    ["g", "<geohash>"],                            // Precise location
    
    // Weight constraints
    ["weight-min", "<number>", "<unit>"],          // unit: g, kg, oz, lb. Following ISO 80000-1
    ["weight-max", "<number>", "<unit>"],
    
    // Dimensional constraints
    ["dim-max", "<dim-value>", "<unit>"],  // unit: cm, in. Following ISO 80000-1. Dimension value should be expressed as "length x width x height"
    ["dim-min", "<dim-value>", "<unit>"],
    
    // Price calculations
    ["price-weight", "<price-per-unit>", "<currency>", "<weight-unit>"],
    ["price-volume", "<price-per-unit>", "<currency>", "<volume-unit>"],
    ["price-distance", "<price-per-unit>", "<currency>", "<distance-unit>"]
  ]
}
```

#### Example Events

Single pickup location:
```jsonc
{
  "kind": 30406,
  "created_at": 1703187600,
  "content": "Downtown Miami Store Pickup",
  "tags": [
    ["d", "miami-downtown-pickup"],
    ["name", "Downtown Miami Pickup"],
    ["price", "0", "USD"],
    ["zone", "US"],
    ["region", "US-FL"],
    ["service", "pickup"],
    ["location", "789 Brickell Ave, Miami, FL 33131"],
    ["g", "dhwm9c4ws"],
  ]
}
```

Separate pickup location (same merchant):
```jsonc
{
  "kind": 30406,
  "created_at": 1703187600,
  "content": "Miami Beach Store Pickup",
  "tags": [
    ["d", "miami-beach-pickup"],
    ["name", "Miami Beach Pickup"],
    ["price", "0", "USD"],
    ["zone", "US"],
    ["region", "US-FL"],
    ["service", "pickup"],
    ["location", "456 Ocean Drive, Miami Beach, FL 33139"],
    ["g", "dhwv1zp8k"],
  ]
}
```

Standard shipping option:
```jsonc
{
  "kind": 30406,
  "created_at": 1703187600,
  "content": "Standard domestic shipping within Florida",
  "tags": [
    ["d", "fl-standard"],
    ["name", "Florida Standard Shipping"],
    ["price", "5.99", "USD"],
    ["zone", "US"],
    ["region", "US-FL"],
    ["service", "standard"],
    ["duration", "24", "72"],
    ["weight-max", "30", "kg"],
    ["dim-max", "120", "60", "60", "cm"],
    ["price-weight", "0.75", "USD", "kg"]
  ]
}
```

#### Shipping Notes

1. For merchants with multiple pickup locations:
   - Create separate shipping option events for each physical location
   - Each location should have its own unique `d` tag identifier
   - Product listings can reference multiple pickup options

2. Clients should:
   - Group pickup locations by merchant when displaying options
   - Use geohash data to show pickup locations on a map
   - Sort pickup locations by distance from user when possible

3. Location identification:
   - Each pickup location must have both `location` and `g` tags
   - The `location` tag contains human-readable address
   - The `g` tag contains geohash for precise positioning

## Order Communication Flow

- Order processing and communication uses [NIP-17](17.md) encrypted direct messages.
  - Kind `14` is used for regular communication, enabling users to maintain a conversation. The subject can be an order ID or left blank, depending on the context.
  - Kind `16` is used for order processing and business logic.
  - Kind `17` is used for order receipts
- Message direction is determined by the `p` tag - when sent from buyer to merchant, `p` contains the merchant's pubkey, and when sent from merchant to buyer, `p` contains the buyer's pubkey. 
- The payment request message can be initiated in two ways, depending on whether the merchant has a server handling payments

### Message Types
1. Order Creation (buyer → merchant) (subject "order-info")
```jsonc
{
  "kind": 16,
  "tags": [
    ["p", "<merchant-pubkey>"],
    ["subject", "order-info"],
    ["type", 1],
    ["order", "<order-id>"],
    ["item", "30402:<pubkey>:<d-tag>", "<quantity>"],
    ["item", "30402:<pubkey>:<d-tag>", "<quantity>"], // Multiple items possible
    ["shipping", "30406:<pubkey>:<d-tag>"],
    ["amount", "<total-amount>"],
    ["address", "<shipping-address>"],
    ["email", "<customer-email>"],
    ["phone", "<customer-phone>"],
    // Other order related fields
  ],
  "content": "Additional notes: Please gift wrap the items."
}
```

2. Payment Request (merchant doesnt have a payment server) (merchant → buyer ) (subject "order-payment")
```jsonc
{
  "kind": 16,
  "tags": [
    ["p", "<buyer-pubkey>"],
    ["subject", "order-payment"],
    ["type", 2],
    ["order", "<order-id>"],
    ["amount", "<total-amount>", "<currency>"],
    ["payment", "lightning", "<bolt11-invoice | ln-address(LUD16)>"],
    ["payment", "bitcoin", "<btc-address>"] // Multiple payment options possible,
    ["expiry", "<unix-timestamp>"]
  ],
  "content": "Payment is due within 24 hours."
}
```

Payment Request (merchant have a payment server) ( buyer → merchant ) (subject "order-payment")
```jsonc
{
  "kind": 16,
  "tags": [
    ["p", "<merchant-pubkey>"],
    ["subject", "order-payment"],
    ["type", 2],
    ["order", "<order-id>"],
    ["amount", "<total-amount>"],
    ["payment", "lightning", "<bolt11-invoice | bolt12-offer>"],
    ["payment", "bitcoin", "<btc-address>"],
    ["expiry", "<unix-timestamp>"]
  ],
  "content": "Payment details provided by merchant's payment server."
}
```

3. Order Status Updates (merchant → buyer) (subject "order-info")
```jsonc
{
  "kind": 16,
  "tags": [
    ["p", "<buyer-pubkey>"],
    ["subject", "order-info"],
    ["type", 3],
    ["order", "<order-id>"],
    ["status", "<order-status>"], // e.g., "confirmed", "processing", "completed"
    ["date", "<unix-timestamp>"]
  ],
  "content": "Your order is being prepared for shipping."
}
```

4. Shipping Updates (merchant → buyer) (subject "shipping-info")
```jsonc
{
  "kind": 16,
  "tags": [
    ["p", "<buyer-pubkey>"],
    ["subject", "shipping-info"],
    ["type", 4],
    ["order", "<order-id>"],
    ["status", "<shipping-status>"], // e.g., "processing", "shipped", "delivered"
    ["tracking", "<tracking-number>"],
    ["carrier", "<carrier-name>"],
    ["eta", "<unix-timestamp>"]
  ],
  "content": "Your order has been picked up by UPS and is on its way!"
}
```

Regular communication between users (subject "<order-id>")
```jsonc
{
  "kind": 14,
  "tags": [
    ["p", "<buyer-pubkey | buyer-pubkey>"],
    ["subject", "<order-id | empty-string>"],
  ],
  "content": "Some extra communication"
}
```

Payment Receipt (buyer → merchant) (subject "order-receipt")
```jsonc
{
  "kind": 17,
  "tags": [
    ["p", "<merchant-pubkey>"],
    ["subject", "order-receipt"],
    ["order", "<order-id>"],
    ["payment", "lightning", "<bolt11-invoice | bolt12-offer>", "<preimage>"],
    // or
    ["payment", "bitcoin", "<btc-address>", "<txid>"],
    ["date", "<unix-timestamp>"]
  ],
  "content": "Payment completed via Lightning Network"
}
```

### Product Reviews (Kind: 31555)

Following [NIP-85](https://github.com/nostr-protocol/nips/blob/b1432b705f553bde6c4eb5fcfde8525d2913b477/85.md) and [QTS](https://habla.news/u/arkinox@arkinox.tech/DLAfzJJpQDS4vj3wSleum) for the review schema:

```jsonc
{
  "kind": 31555,
  "tags": [
    ["d", "a:<product-listing-kind>:<merchant-pubkey>:<product-listing-d-tag>"],
    ["rating", "<0-or-1>", "thumb"],
    ["rating", "<0-or-1>", "<rating-label-1>"],
    ["rating", "<0-or-1>", "<rating-label-2>"],
    ...
  ],
  "content": "<comment-on-product>"
}
```

The `thumb` rating label MUST represent 50% of the score weight and be set as "good" (1) or "bad" (0), indicating the overall sentiment. Additional arbitrary rating labels can be added and would also be scored as "good" (1) or "bad" (0), but with equal weight across the remaining 50% of the rating. More granular scores between 0-1 can also be used without breaking compatibility.

Rating calculation:

Total Score = (Thumb × 0.5) + (0.5 × (∑(Additional Ratings) ÷ Number of Additional Ratings))

Review Example:

```jsonc
{
  "kind": 31555,
  "tags": [
    ["d", "a:<listing kind>:<merchant pubkey>:<listing d-tag>"],
    ["rating", "1", "thumb"],
    ["rating", "1", "value"], 
    ["rating", "1", "quality"], 
    ["rating", "0", "delivery"], 
    ["rating", "1", "communication"],
  ],
  "content": "Great product!"
}
```

### Payment Flow Notes

A payment preference can be added to the Kind0 event in the following structure payment-preference = ecash | lud16 | bolt12 | manual

ORDER OF SUGGESTED PAYMENT ACCEPTANCE:

1) eCash

2) Lightning (bolt11 / bolt12)
   
3) On-chain

4) Marketplace Server 

4.1. When a merchant has a payment server:
    - The buyer can immediately send a payment request message containing payment details obtained from the merchant's server
    - This eliminates the need to wait for the merchant to come online
    - The payment server is responsible for generating valid payment details and monitoring for completion
4.2 When no payment server is available:
    - The traditional flow is used where the merchant sends the payment request
    - The buyer waits for the merchant's response before proceeding with payment
43 In both cases:
    - The payment receipt is sent by the buyer after completing payment
    - All payment details should be verified against the original order
    - The message direction is clearly indicated by the `p` tag

5) Fiat Gateway ( Example Robosats )

### Marketplace Server Role

Marketplace servers can optionally facilitate the payment process by:

1. Generating payment requests based on merchant preferences when buyers initiate orders
2. Verifying payments and generating receipts automatically
3. Managing inventory and order status updates
4. Coordinating shipping information
5. Price calculations

This provides a smoother user experience while maintaining the ability for direct merchant-buyer communication as a fallback mechanism.

## Notes and Considerations

1. Tags are used for all structured, machine-readable data to facilitate easier parsing and filtering.

2. The content field is reserved for human-readable messages and additional information that doesn't require machine parsing.

3. All timestamps should be in Unix format (seconds since epoch).

4. Order IDs should be used consistently across all related messages.

5. Message threading should follow [NIP-10](10.md) conventions when replies are needed.
