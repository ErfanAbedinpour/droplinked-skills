# Core Entity Models Reference

This reference documents the key data models in the Droplinked backend. Full schema at `prisma/schema.prisma`.

## Table of Contents
- [MerchantV2](#merchantv2)
- [ShopV2](#shopv2)
- [CustomerV2](#customerv2)
- [ProductV2](#productv2)
- [ProductSkuV2](#productskuv2)
- [CartV2](#cartv2)
- [OrderV2](#orderv2)
- [Key Enums](#key-enums)
- [Entity Relationships](#entity-relationships)

---

## MerchantV2

The shop owner/operator account.

```prisma
model MerchantV2 {
  id              String           @id @default(auto()) @map("_id") @db.ObjectId
  email           String?          @unique
  emailVerifiedAt DateTime?
  status          MerchantStatusV2 @default(PENDING)  // PENDING | ACTIVE | SUSPENDED | DISABLED
  tokenVersion    Int              @default(0)

  // Relations
  authCredentials    AuthCredentialsV2?
  oauthAccounts      OAuthAccountsV2[]
  sessions           SessionsV2[]
  shopsV2            ShopV2[]
  roles              Role[]
}
```

**Key fields:**
- `status`: Controls account state (PENDING, ACTIVE, SUSPENDED, DISABLED)
- `tokenVersion`: Incremented to invalidate all tokens

---

## ShopV2

The store entity owned by a merchant.

```prisma
model ShopV2 {
  id                String             @id @default(auto()) @map("_id") @db.ObjectId
  merchantId        String             @db.ObjectId
  name              String
  description       String?
  url               String?
  status            ShopStatus         @default(NEW)  // NEW | SHOP_INFO_COMPLETED | DISABLED

  // Configuration
  shopTemplate      ShopTemplate?
  shopDesign        ShopDesign?
  wallets           Wallet[]
  socialLinks       SocialLink[]
  settings          ShopSettings?
  loginMethods      LoginMethod[]
  deployedContracts DeployedContract[]
  paymentMethods    ShopPaymentMethodV2[]

  // Relations
  merchant           MerchantV2
  products           ProductV2[]
  productCollections ProductCollectionV2[]
  customersV2        CustomerV2[]
  orders             OrderV2[]
  carts              CartV2[]
  apiKeys            ApiKeyV2[]
  credits            Credit[]
}
```

**Key embedded types:**
- `ShopSettings`: Domain, currency, payment method, DNS data
- `PaymentMethod`: GATEWAY (Stripe/Coinbase) or WALLET_ROUTING (crypto)

---

## CustomerV2

A customer of a specific shop.

```prisma
model CustomerV2 {
  id            String  @id @default(auto()) @map("_id") @db.ObjectId
  email         String?
  walletAddress String?
  walletNetwork String?
  shopId        String  @db.ObjectId
  firstName     String?
  lastName      String?
  phoneNumber   String?

  // Relations
  shop   ShopV2
  orders OrderV2[]
  carts  CartV2[]
}
```

**Note:** Customers are shop-scoped (same email can be different customers in different shops).

---

## ProductV2

Product entity with support for digital, physical, and print-on-demand types.

```prisma
model ProductV2 {
  id                String                 @id @default(auto()) @map("_id") @db.ObjectId
  type              ProductTypeV2          // digital | physical | pod
  shopId            String                 @db.ObjectId
  collectionId      String?                @db.ObjectId
  title             String
  description       String?
  slug              String                 @unique
  status            ProductStatusV2        @default(draft)  // draft | published
  isVisible         Boolean                @default(true)
  isPurchasable     Boolean                @default(true)

  // Media
  images            ProductImageV2[]
  defaultImageIndex Int                    @default(0)

  // Features
  affiliate         ProductAffiliateV2?    // Commission settings
  delivery          ProductDeliveryV2?     // For digital products
  pod               ProductPodV2?          // Print-on-demand config
  nftRecording      ProductNftRecordingV2? // Blockchain recording

  // Relations
  skus       ProductSkuV2[]
  collection ProductCollectionV2?
  shop       ShopV2
}
```

**Product types:**
- `digital`: Downloadable content
- `physical`: Shipped goods
- `pod`: Print-on-demand (Printful integration)

---

## ProductSkuV2

Stock Keeping Unit - specific variant of a product.

```prisma
model ProductSkuV2 {
  id         String                  @id @default(auto()) @map("_id") @db.ObjectId
  productId  String                  @db.ObjectId
  type       ProductTypeV2
  price      Float
  rawPrice   Float?                  // Original price before markup
  royalty    Float?
  inventory  ProductSkuInventoryV2   // { policy: boolean, quantity?: number }
  externalId String?                 // Printful/external system ID
  variantKey String?                 // Unique key for variant combination
  image      String?

  // Variant attributes
  attributes ProductSkuAttributeV2[] // [{ key: "size", value: "L", caption: "Large" }]
  dimensions ProductSkuDimensionsV2? // width, length, height, weight

  // Blockchain
  record     ProductSkuRecordV2?     // NFT recording status

  // Relations
  product ProductV2
}
```

**Inventory policy:**
- `policy: true` = finite inventory (uses `quantity`)
- `policy: false` = infinite inventory

---

## CartV2

Shopping cart for a shop.

```prisma
model CartV2 {
  id                String               @id @default(auto()) @map("_id") @db.ObjectId
  shopId            String               @db.ObjectId
  customerId        String?              @db.ObjectId
  email             String?
  status            CartStatus           @default(ACTIVE)  // ACTIVE | EXPIRED | DELETED

  // Items
  items             CartItem[]           // Embedded array

  // Shipping
  shippingAddressId String?              @db.ObjectId
  shippingAddress   CartShippingAddress?
  availableShipping CartShipping[]

  // Pricing
  financialDetails  FinancialDetails?    // tax, shipping, discounts, amounts
  giftcard          CartGiftCard?
  additional        CartAdditional?      // note

  // Relations
  shop     ShopV2
  customer CustomerV2?
}
```

**CartItem embedded type:**
```typescript
type CartItem {
  productId     String
  skuId         String
  quantity      Int
  unitPrice     Float?
  totalPrice    Float?
  productType   CartProductType  // DIGITAL | EVENT | PHYSICAL | POD
  affiliateInfo CartAffiliateInfo?
  ruleset       CartRuleset?     // Applied discount
}
```

---

## OrderV2

Completed order with payment and shipping details.

```prisma
model OrderV2 {
  id                  String                     @id @default(auto()) @map("_id") @db.ObjectId
  orderNumber         String                     @unique
  shopId              String                     @db.ObjectId
  customerId          String?                    @db.ObjectId
  email               String
  status              OrderStatus                @default(PENDING)

  // Items & Shipping
  items               OrderItem[]
  shippingAddress     OrderShippingAddress?
  selectedShipping    OrderSelectedShipping[]

  // Payment & Finance
  payment             OrderPayment?
  financialDetails    OrderFinancialDetails
  paymentSplits       OrderPaymentSplits?
  revenueDistribution OrderRevenueDistribution[]

  // Dates
  orderDate           DateTime
  expiredAt           DateTime?

  // Relations
  shop     ShopV2?
  customer CustomerV2?
}
```

**Order statuses:**
- PENDING, CONFIRMED, PROCESSING, SHIPPED, DELIVERED, CANCELLED, REFUNDED, EXPIRED

---

## Key Enums

### Product Types
```prisma
enum ProductTypeV2 { digital, physical, pod }
enum ProductStatusV2 { draft, published }
```

### Order & Cart
```prisma
enum CartStatus { ACTIVE, EXPIRED, DELETED }
enum OrderStatus { PENDING, CONFIRMED, PROCESSING, SHIPPED, DELIVERED, CANCELLED, REFUNDED, EXPIRED }
enum CartProductType { DIGITAL, EVENT, PHYSICAL, POD }
```

### Payment
```prisma
enum PaymentMethodType { GATEWAY, WALLET_ROUTING }
enum OrderPaymentMethod { FIAT, CRYPTO }
enum OrderPaymentStatus { PENDING, COMPLETED, FAILED }
```

### Blockchain
```prisma
enum RecordNetwork { NONE, POLYGON, BINANCE, REDBELLY, BITLAYER, ETH, LINEA, NEAR, BASE, SKALE, XION }
enum NftRecordingStatusV2 { PENDING, RECORDED, FAILED }
```

---

## Entity Relationships

```
MerchantV2
  └── ShopV2 (1:N)
        ├── CustomerV2 (1:N)
        │     ├── OrderV2 (1:N)
        │     └── CartV2 (1:N)
        ├── ProductV2 (1:N)
        │     └── ProductSkuV2 (1:N)
        ├── ProductCollectionV2 (1:N)
        │     └── ProductV2 (1:N)
        ├── OrderV2 (1:N)
        ├── CartV2 (1:N)
        └── ApiKeyV2 (1:N)
```

**Key ownership rules:**
- Merchant owns Shops
- Shop owns Products, Collections, Customers, Orders, Carts
- Product owns SKUs
- Collection groups Products (optional)
