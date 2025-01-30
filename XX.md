NIP-XX
======

Marketplace Protocol
---------------------------

`draft` `optional`

This NIP defines a comprehensive protocol for implementing decentralized marketplaces on Nostr. It provides a complete e-commerce framework while maintaining protocol simplicity and interoperability.

## Protocol Requirements

The protocol is structured into required core components and optional extensions:

### Required Components
Implementations MUST support these core features to be considered compatible:

- Product listing events (Kind: 30402)
- Basic product metadata (title, price, currency)
- Order communication via [NIP-17](17.md) encrypted messages
- Native Nostr payment processing

### Optional Components
These features MAY be implemented based on specific marketplace needs:

- Drafts following [NIP-37](37.md)
- Product collections (Kind: 30405) 
- Shipping options (Kind: 30406)
- Product reviews (Kind: 31555)
- Extended product metadata
- Server-assisted payment processing

### Core Flows
1. Order Communication Flow
   - Encrypted messaging between buyer and seller
   - Order status updates and confirmations
   
2. Shipping Flow
   - Shipping options and pricing (Kind: 30406)
   - Geographic restrictions and zones
   - Delivery status tracking
   
3. Payment Flow
   - Multiple payment method support
   - Payment verification
   - Receipt generation

A standard checkout process proceeds as:
1. Products added to cart
2. Shipping details collected and costs calculated
3. Payment request
4. Payment processed and verified
5. Order and shipping follow up using encrypted messages

## Events and Kinds

### Product Listing (Kind: 30402)
The core event type for representing products in the marketplace. Each product listing MUST include basic metadata and MAY include additional details. Products are the core element in a marketplace, their configuration is the source of truth, overriding other possible configurations of other market elements such as collections.

Content: Product description, markdown is allowed
Required tags:
- `d`: Unique product identifier for referencing the listing
- `title`: Product name/title for display
- `price`: Price information array `[<amount>, <currency>, <optional frequency>]`
  - amount: Decimal number (e.g., "10.99")
  - currency: ISO 4217 code (e.g., "USD", "EUR")
  - frequency: Optional subscription interval using ISO 8601 duration units (e.g. 'D' for daily, 'W' for weekly, 'Y' for yearly).

Optional tags:
- Product Details:
  - `type`: Product classification `[<type>, <format>]`
    - type: "simple", "variable", or "variation"
    - format: "digital" or "physical"
    - Default: type: "simple", format: "digital"
  - `visibility`: Display status ("hidden", "on-sale", "pre-order"). Default: "on-sale"
  - `stock`: Available quantity as integer
  - `summary`: Short product description
  - `spec`: Product specifications `[<key>, <value>]`

- Media:
  - `image`: Product images `[<url>, <dimensions>]`
    - url: Direct image URL
    - dimensions: Optional "<width>x<height>" format

- Physical Properties:
  - `weight`: Product weight `[<value>, <unit>]` using ISO 80000-1
  - `dim`: Dimensions `[<l>x<w>x<h>, <unit>]` using ISO 80000-1

- Location:
  - `location`: Human-readable location string
  - `g`: Geohash for precise location lookup

- Organization:
  - `t`: Product categories/tags
  - `a`: Collection reference "30405:<pubkey>:<d-tag>"
  - `shipping`: Shipping options
    - Format: "30406:<pubkey>:<d-tag>" for direct options
    - Format: "30405:<pubkey>:<d-tag>" for collection shipping

```jsonc
{
  "kind": 30402,
  "created_at": <unix timestamp>,
  "content": "<product description in markdown>",
  "tags": [
    // Required tags
    ["d", "<product identifier>"],
    ["title", "<product title>"],
    // TODO: How to inherit currency from a collection?
    ["price", "<amount>", "<currency>", "<optional frequency>"],

    // Product details
    ["type", "<simple|variable|variation>", "<digital|physical>"],  // Defaults: simple, digital
    ["visibility", "<hidden|on-sale|pre-order>"],  // Default: on-sale
    ["stock", "<integer>"],  // Available quantity
    ["summary", "<short description>"],
    
    // Media and specs
    ["image", "<url>", "<dimensions>"],
    ["spec", "<key>", "<value>"],  // Product specifications (e.g., "screen-size", "21 inch"). Cam be present multiple times
    
    // Physical properties (for shipping)
    ["weight", "<value>", "<unit>"],  // ISO 80000-1 units (g, kg, etc)
    ["dim", "<l>x<w>x<h>", "<unit>"], // ISO 80000-1 units (mm, cm, m)
    
    // Location
    ["location", "<address string>"],
    ["g", "<geohash>"],
    
    // Classifications
    ["t", "<category>"],
    
    // References
    ["shipping", "<30406|30405>:<pubkey>:<d-tag>"],  // Shipping options or collection
    ["a", "30405:<pubkey>:<d-tag>"]  // Product collection
  ]
}
```

#### Notes
1. Product Configuration:
   - Products can be simple, variable (with options), or variations of variable products
   - Digital products skip shipping requirements
   - Visibility controls product display status

2. Shipping Rules:
   - Shipping options can be defined directly or inherited from collections
   - If the product specifies product-specific shipping, and also from a collation, shipping options MUST be merged.

3. Collections and Categories:
   - Products can refer to one o multiple collections using `a` tags, whether or not they are part of it, for discoverability purposes.
   - Categories ("t" tags) aid in discovery and organization

4. Location Support:
   - Optional location data aids in local marketplace features
   - Geohash enables precise location-based searches

### Product Collection (Kind: 30405)
A specialized event type using [NIP-51](51.md) list format to organize related products into groups. Collections enable merchants to create meaningful product groupings and share common attributes that products can reference.

Required tags:
- `d`: Unique collection identifier
- `name`: Collection display name
- `a`: Product references `["a", "30402:<pubkey>:<d-tag>"]`
  - Multiple product references allowed
  - References must point to valid product listings

Optional tags:
- Display:
  - `image`: Collection banner/thumbnail URL
  - `summary`: Brief collection description

- Location:
  - `location`: Human-readable location string
  - `g`: Geohash for precise location lookup

- Reference Options:
  - `shipping`: Available shipping options `["shipping", "30406:<pubkey>:<d-tag>"]`
  - `currency`: ISO 4217 currency code for collection

```jsonc
{
  "kind": 30405,
  "created_at": <unix timestamp>,
  "content": "<optional collection description>",
  "tags": [
    // Required tags
    ["d", "<collection identifier>"],
    ["name", "<collection name>"],
    ["a", "30402:<pubkey>:<d-tag>"],  // Product reference
    
    // Optional tags
    ["image", "<collection image URL>"],
    ["summary", "<collection description>"],
    
    // Location
    ["location", "<location string>"],
    ["g", "<geohash>"],
    
    // Reference Options
    ["shipping", "30406:<pubkey>:<d-tag>"],  // Available shipping options
    ["currency", "<ISO 4217 currency code>"]  // Collection currency
  ]
}
```

#### Notes
1. Collection Management:
   - Collections can contain any number of products
   - Products can belong to multiple collections
   - Products must explicitly reference collection resources to inherit collection attributes (e.g. shipping, currency).

2. Reference Model:
   - Collection settings (shipping, currency) serve as references only
   - Products must explicitly reference collection shipping options
   - No automatic cascading of settings to products

3. Location Support:
   - Optional location data helps with marketplace organization
   - Enables geographic grouping of related products

### Drafts
Products and collections can be saved as private drafts while being prepared for publication. This allows merchants to work on listings before making them publicly visible. Implementation MUST follow [NIP-37](https://github.com/nostr-protocol/nips/blob/master/37.md) for draft management.

### Shipping Option (Kind: 30406)
A specialized event type for defining shipping methods, costs, and constraints. Shipping options can be published by merchants or third-party providers (delivery companies, DVMs, etc.) and referenced by product listings or collections.

Required tags:
- `d`: Unique shipping option identifier
- `name`: Display name for the shipping method
- `price`: Base cost array `[<base_cost>, <currency>]`
- `country`: Array of ISO 3166-1 alpha-2 country codes `[<code1>, <code2>, ...]`
- `service`: Service type ("standard", "express", "overnight", "pickup")

Optional tags:
- Extra details:
  - `carrier`: The name of the carrier that will be used for the delivery
- Time and Location:
  - `region`: Array of ISO 3166-2 region codes `[<code1>, <code2>, ...]`
  - `duration`: Delivery window `[<min>, <max>, <unit>]` using ISO 8601 duration units
    - min: Minimum delivery time
    - max: Maximum delivery time
    - unit: "H" (hours), "D" (days), "W" (weeks)
  - `location`: Physical address for pickup
  - `g`: Geohash for precise location

- Constraints:
  - `weight-min`: Minimum weight `[<value>, <unit>]` (ISO 80000-1)
  - `weight-max`: Maximum weight `[<value>, <unit>]`
  - `dim-min`: Minimum dimensions `[<l>x<w>x<h>, <unit>]`
  - `dim-max`: Maximum dimensions `[<l>x<w>x<h>, <unit>]`

- Price Calculations:
  - `price-weight`: Per weight pricing `[<price>, <currency>, <unit>]`
  - `price-volume`: Per volume pricing `[<price>, <currency>, <unit>]`
  - `price-distance`: Per distance pricing `[<price>, <currency>, <unit>]`

```jsonc
{
  "kind": 30406,
  "created_at": <unix timestamp>,
  "content": "<optional shipping description>",
  "tags": [
    // Required tags
    ["d", "<shipping identifier>"],
    ["name", "<shipping method name>"],
    ["price", "<base_cost>", "<currency>"],
    ["country", "<ISO 3166-1 alpha-2>", "...", "..."],  // Array of country codes
    ["service", "<service-type>"],

    // Extra details
    ["carrier","<name of the carrier>"]
    
    // Time and Location
    ["region", "<ISO 3166-2 code>", "...", "..."],  // Array of region codes
    ["duration", "<min>", "<max>", "<unit>"],  // ISO 8601 duration units (H/D/W)
    ["location", "<address string>"],
    ["g", "<geohash>"],
    
    // Constraints
    ["weight-min", "<value>", "<unit>"],
    ["weight-max", "<value>", "<unit>"],
    ["dim-min", "<l>x<w>x<h>", "<unit>"],
    ["dim-max", "<l>x<w>x<h>", "<unit>"],
    
    // Price Calculations
    ["price-weight", "<price>", "<currency>", "<unit>"],
    ["price-volume", "<price>", "<currency>", "<unit>"],
    ["price-distance", "<price>", "<currency>", "<unit>"]
  ]
}
```

#### Implementation Examples

Local Pickup:
```jsonc
{
  "kind": 30406,
  "created_at": 1703187600,
  "content": "Downtown Store Pickup",
  "tags": [
    ["d", "downtown-pickup"],
    ["name", "Downtown Store Pickup"],
    ["price", "0", "USD"],
    ["country", "US"],
    ["region", "US-FL"],
    ["service", "pickup"],
    ["location", "123 Main St, Downtown, FL"],
    ["g", "dhwm9c4ws"]
  ]
}
```

Standard Shipping:
```jsonc
{
  "kind": 30406,
  "created_at": 1703187600,
  "content": "Standard Regional Shipping",
  "tags": [
    ["d", "standard-regional"],
    ["name", "Standard Shipping"],
    ["price", "5.99", "USD"],
    ["country", "US"],
    ["region", "US-FL"],
    ["service", "standard"],
    ["duration", "24", "72", "H"],  // 24-72 hours delivery window
    ["weight-max", "30", "kg"],
    ["dim-max", "120x60x60", "cm"],
    ["price-weight", "0.75", "USD", "kg"]
  ]
}
```

#### Notes
1. Event Management:
   - Create separate events for each distinct shipping option
   - Each option needs a unique `d` tag identifier
   - Merchants can reference third-party shipping options

2. Shipping Rules:
   - Physical pickup requires location and/or geohash
   - Weight/dimension constraints use ISO 80000-1 units
   - Price calculations can combine multiple factors

3. Client Behavior:
   - Group options by service type and location
   - Use geohash for distance-based sorting
   - Validate package constraints before offering options

## Order Communication Flow
Order processing and status updates use [NIP-17](17.md) encrypted direct messages, with three event kinds serving different purposes:

- Kind `14`: Regular communication between parties
  - General inquiries and responses
  - Order clarifications
  - Subject can be order ID or empty
  
- Kind `16`: Order processing and status. These messages include a `type` field that indicates the specific kind of message
  - Order creation and details. `type`: 1
  - Payment requests. `type`: 2
  - Status updates. `type`: 3
  - Shipping information. `type`: 4
  
- Kind `17`: Payment receipts and verification

Message direction is determined by the `p` tag:
- Buyer → Merchant: event author is the buyer, `p` tag contains merchant's pubkey
- Merchant → Buyer: event author is the merchant, `p` tag contains buyer's pubkey

The payment request flow can operate in two modes:
1. Direct: Merchant processes requests manually
2. Service-assisted: Merchant's payment service handles requests

### Message Types

#### 1. Order Creation
Sent by buyer to initiate order

```jsonc
{
  "kind": 16,
  "tags": [
    // Required tags
    ["p", "<merchant-pubkey>"],
    ["subject", "<order-info subject>"],
    ["type", "1"],  // Order creation
    ["order", "<order-id>"],  // Unique order identifier
    ["amount", "<total-amount>", "<currency>"],
    
    // Order items (can repeat)
    ["item", "30402:<pubkey>:<d-tag>", "<quantity>"],
    
    // Shipping details
    ["shipping", "30406:<pubkey>:<d-tag>"],
    ["address", "<shipping-address>"],
    
    // Customer contact
    ["email", "<customer-email>"],  // Optional
    ["phone", "<customer-phone>"],  // Optional
  ],
  "content": "Order notes or special requests"
}
```

#### 2. Payment Request
Two variants depending on payment processing mode:

##### Direct Processing (merchant → buyer)
In this mode the merchant have to manually send the payment request to the buyer.

```jsonc
{
  "kind": 16,
  "tags": [
    // Required tags
    ["p", "<buyer-pubkey>"],
    ["subject", "order-payment"],
    ["type", "2"],  // Payment request
    ["order", "<order-id>"],
    ["amount", "<total-amount>", "<currency>"],
    
    // Payment options (can include multiple)
    ["payment", "lightning", "<bolt11-invoice|lud16>"],
    ["payment", "bitcoin", "<btc-address>"],
    ["payment", "ecash", "<mint-url>"],
  ],
  "content": "Payment instructions and notes"
}
```

##### Service Processing (buyer → merchant)
In this mode, the merchant uses a service to process payments automatically without manual interaction. The key difference is that the buyer initiates the transaction using information provided by the merchant's chosen service. To enhance security and verifiability, the merchant SHOULD use [NIP-89](89.md) application handlers to define their preferred payment processing service and prevent fake services from issuing fraudulent payment requests.

```jsonc
{
  "kind": 16,
  "tags": [
    // Required tags
    ["p", "<merchant-pubkey>"],
    ["subject", "order-payment"],
    ["type", "2"],  // Payment request
    ["order", "<order-id>"],
    ["amount", "<total-amount>", "<currency>"],
    
    // Payment details from service
    ["payment", "lightning", "<bolt11-invoice|bolt12-offer>"],
    ["payment", "bitcoin", "<btc-address>"],
    ["payment", "ecash", "<mint-url>"],
  ],
  "content": "Service-generated payment details"
}
```

#### 3. Order Status Updates
Sent by merchant to update order status (Kind 16)

```jsonc
{
  "kind": 16,
  "tags": [
    // Required tags
    ["p", "<buyer-pubkey>"],
    ["subject", "order-info"],
    ["type", "3"],  // Status update
    ["order", "<order-id>"],
    
    // Status information
    ["status", "<order-status>"],  // pending|confirmed|processing|completed|cancelled
    ["date", "<unix-timestamp>"],
  ],
  "content": "Human readable status update"
}
```

#### 4. Shipping Updates
Sent by merchant with delivery information (Kind 16)

```jsonc
{
  "kind": 16,
  "tags": [
    // Required tags
    ["p", "<buyer-pubkey>"],
    ["subject", "shipping-info"],
    ["type", "4"],  // Shipping update
    ["order", "<order-id>"],
    
    // Shipping details
    ["status", "<shipping-status>"],  // processing|shipped|delivered|exception
    ["tracking", "<tracking-number>"],
    ["carrier", "<carrier-name>"],
    ["eta", "<unix-timestamp>"],
  ],
  "content": "Shipping status and tracking information"
}
```

#### 5. General Communication
Used for any order-related messages (Kind 14)

```jsonc
{
  "kind": 14,
  "tags": [
    // Required tags
    ["p", "<recipient-pubkey>"],
    ["subject", "<order-id>"],  // Optional, can be empty
  ],
  "content": "General communication message"
}
```

#### 6. Payment Receipt
Sent by buyer to confirm payment (Kind 17)

```jsonc
{
  "kind": 17,
  "tags": [
    // Required tags
    ["p", "<merchant-pubkey>"],
    ["subject", "order-receipt"],
    ["order", "<order-id>"],
    
    // Payment proof (one required)
    ["payment", "lightning", "<invoice>", "<preimage>"],
    ["payment", "bitcoin", "<address>", "<txid>"],
    ["payment", "ecash", "<mint-url>", "<proof>"],
    
    // Metadata
    ["date", "<unix-timestamp>"],
    ["amount", "<amount>", "<currency>"]
  ],
  "content": "Payment confirmation details"
}
```

#### Notes
1. Message Flow:
   - All sensitive data must be encrypted
   - Status updates should be prompt and clear
   - Receipts should include verifiable proofs

2. Payment Processing:
   - Service mode enables faster processing
   - Direct mode provides more flexibility
   - Multiple payment options can be offered

3. Status Tracking:
   - Use consistent status codes
   - Include timestamps for all updates
   - Provide clear user messages

### Product Reviews (Kind: 31555)
Product reviews follow [NIP-85](https://github.com/nostr-protocol/nips/blob/b1432b705f553bde6c4eb5fcfde8525d2913b477/85.md) and [QTS](https://habla.news/u/arkinox@arkinox.tech/DLAfzJJpQDS4vj3wSleum) guidelines with additional marketplace-specific rating criteria. Reviews provide structured feedback about products, merchants, and the overall purchase experience.

Required tags:
- `d`: Reference to product `["d", "a:30402:<merchant-pubkey>:<product-d-tag>"]`
- `rating`: Primary rating `["rating", "<score>", "thumb"]`
  - score: 0 (negative) to 1 (positive)
  - "thumb" label MUST be present as primary rating

Optional tags:
- Additional Ratings:
  - `rating`: Category scores `["rating", "<score>", "<category>"]`
    - score: 0 to 1 (supports fractional values)
    - category: These are optional, some standard categories may include:
      - "value": Price vs quality
      - "quality": Product quality
      - "delivery": Shipping experience
      - "communication": Merchant responsiveness

```jsonc
{
  "kind": 31555,
  "created_at": <unix timestamp>,
  "tags": [
    // Required tags
    ["d", "a:30402:<merchant-pubkey>:<product-d-tag>"],
    ["rating", "1", "thumb"],  // Primary rating
    
    // Optional rating categories
    ["rating", "0.8", "value"],
    ["rating", "1.0", "quality"],
    ["rating", "0.6", "delivery"],
    ["rating", "0.9", "communication"]
  ],
  "content": "Detailed review text"
}
```

#### Rating Calculation
The final score combines the primary "thumb" rating (50% weight) with additional category ratings (50% combined weight):

```
Total Score = (Thumb × 0.5) + (0.5 × (∑(Category Ratings) ÷ Number of Categories))
```

#### Notes
1. Rating System:
   - Primary thumb rating is required
   - Additional categories are optional
   - Scores support fractional values (0-1)
   - Custom categories can be added

// TODO: continue here
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
