# sellbrite product grouping + endpoint usage (June 5, 2026)

## purpose

this document explains the product grouping problem we found, why veeqo alone is not enough to detect it, and how sellbrite's api may help expose parent products, child variations, and inventory data.

## core problem

the issue is circular:

```text
goal:
detect products whose variations have mismatched sku bases

but:
to know whether variation skus belong to the same product, we first need product grouping

problem:
veeqo mostly exposes flat variants/sellables, so it may not tell us which variants belong together
```

example:

```text
CA10X15-BK-SM
CA10X16-BK-SM
CA10X15-RD-CL
```

if these are shown as flat veeqo variants, we cannot safely say whether this is:

```text
one product with mismatched bases
```

or:

```text
multiple products that happen to look similar
```

to detect a base mismatch, we need a known product boundary first.

## why woocommerce helped

woocommerce gave us:

```text
product
  -> variations
```

that allowed the validator to ask:

```text
within this known product group, do all variations share the same base?
```

that is why the base sku mismatch check worked in the woocommerce pipeline.

## why veeqo alone is not enough

if veeqo only gives flat variants, it can help detect:

- duplicate skus
- missing skus
- possible family bases
- inventory/channel presence

but it cannot confidently detect:

```text
this product has mismatched variation bases
```

unless it also exposes parent product grouping.

## sellbrite may solve the grouping problem

sellbrite appears to have explicit variation group concepts.

important endpoints:

```text
GET /products
GET /products/{sku}
GET /variation_groups
GET /inventory
```

## endpoint roles

### `GET /products`

use this to fetch the catalog.

this endpoint is paginated and may include:

- simple products
- variation parent products
- variation child products

a product's type is inferred from its fields.

### `GET /products/{sku}`

use this to fetch one specific sku.

the response can represent one of three cases:

#### simple product

```json
{
  "sku": "SHOE123",
  "name": "Shoe"
}
```

simple products do not have:

```text
parent_sku
variation_fields
variation_keys
variations
```

#### variation parent

```json
{
  "sku": "FM270X20TX",
  "variation_keys": ["color", "lens"],
  "variations": [
    {
      "sku": "FM270X20TX-BK-ECCL"
    },
    {
      "sku": "FM270X20TX-BK-CL"
    }
  ]
}
```

presence of:

```text
variations
variation_keys
```

means the sku is a parent / variable product.

#### variation child

```json
{
  "sku": "FM270X20TX-BK-ECCL",
  "parent_sku": "FM270X20TX",
  "variation_fields": {
    "color": "black",
    "lens": "transition"
  }
}
```

presence of:

```text
parent_sku
variation_fields
```

means the sku is a child variation.

### `GET /variation_groups`

use this to fetch known parent/variation group records.

this may be useful if `GET /products` does not fully expose the grouping structure.

### `GET /inventory`

use this to fetch stock and physical inventory fields by sku.

fields may include:

- sku
- warehouse_uuid
- on_hand
- available
- reserved
- product_name
- package_length
- package_width
- package_height
- package_weight
- cost
- upc
- asin
- fnsku
- bin_location

inventory should be joined to product/catalog records by sku.

## recommended sellbrite pull strategy

start with:

```text
1. GET /products across all pages
2. classify each product as simple, parent, or child
3. GET /inventory across all pages
4. join inventory rows by sku
5. use parent products + simple products as storefront products
6. use child variations as storefront variations
```

only use `GET /variation_groups` if `/products` does not expose enough grouping data.

## product type detection logic

```python
def classify_sellbrite_product(product):
    if "variations" in product or "variation_keys" in product:
        return "variable_parent"

    if "parent_sku" in product or "variation_fields" in product:
        return "variation_child"

    return "simple"
```

## base sku mismatch audit

once sellbrite gives a reliable product boundary:

```text
parent product
  -> child variation skus
```

then we can run the same style of check from the woocommerce pipeline:

```python
def base_sku(sku):
    return sku.split("-")[0]

def has_base_mismatch(variation_skus):
    bases = {base_sku(sku) for sku in variation_skus}
    return len(bases) > 1
```

example:

```text
FM270X20TX-BK-ECCL
FM270X20TX-BK-CL
FM270X20TX-RD-ECCL
```

base set:

```text
{FM270X20TX}
```

passes.

example:

```text
CA10X15-BK-SM
CA10X16-BK-SM
CA10X15-RD-CL
```

base set:

```text
{CA10X15, CA10X16}
```

flags.

## family sku principle

the ideal long-term catalog rule is:

```text
family_sku-variation_chunks
```

example:

```text
FM270X20TX-BK-ECCL-ZMD
```

means:

```text
family_sku = FM270X20TX
variation chunks = BK, ECCL, ZMD
```

the family sku should be globally unique.

that means:

```text
FM270X20TX
```

should refer to exactly one product family.

## architecture implication

sellbrite may be useful because it may already contain:

```text
variation group parent
  -> child variations
```

which is the structure veeqo appears to lack in its flat variant view.

if sellbrite exposes good grouping, it can be used as a reference to clean or validate veeqo.

## authentication

do not hardcode credentials in scripts.

store the credentials in `.env`:

```bash
SELLBRITE_ACCESS_TOKEN=your_access_token_here
SELLBRITE_SECRET_KEY=your_secret_key_here
SELLBRITE_BASE_URL=https://api.sellbrite.com/v1
```

## curl example

sellbrite authentication method may depend on the exact account/api version. if the credentials are accepted as basic auth, test like this:

```bash
curl -u "$SELLBRITE_ACCESS_TOKEN:$SELLBRITE_SECRET_KEY" \
  "$SELLBRITE_BASE_URL/products?limit=100&page=1"
```

inventory:

```bash
curl -u "$SELLBRITE_ACCESS_TOKEN:$SELLBRITE_SECRET_KEY" \
  "$SELLBRITE_BASE_URL/inventory?limit=100&page=1"
```

specific product:

```bash
curl -u "$SELLBRITE_ACCESS_TOKEN:$SELLBRITE_SECRET_KEY" \
  "$SELLBRITE_BASE_URL/products/FM270X20TX"
```

variation groups:

```bash
curl -u "$SELLBRITE_ACCESS_TOKEN:$SELLBRITE_SECRET_KEY" \
  "$SELLBRITE_BASE_URL/variation_groups?limit=100&page=1"
```

## node fetch example

```ts
const SELLBRITE_ACCESS_TOKEN = process.env.SELLBRITE_ACCESS_TOKEN;
const SELLBRITE_SECRET_KEY = process.env.SELLBRITE_SECRET_KEY;
const SELLBRITE_BASE_URL = process.env.SELLBRITE_BASE_URL ?? "https://api.sellbrite.com/v1";

if (!SELLBRITE_ACCESS_TOKEN || !SELLBRITE_SECRET_KEY) {
  throw new Error("Missing Sellbrite credentials");
}

const auth = Buffer.from(
  `${SELLBRITE_ACCESS_TOKEN}:${SELLBRITE_SECRET_KEY}`
).toString("base64");

async function sellbriteGet(path: string) {
  const res = await fetch(`${SELLBRITE_BASE_URL}${path}`, {
    headers: {
      Authorization: `Basic ${auth}`,
      Accept: "application/json",
    },
  });

  if (!res.ok) {
    const body = await res.text();
    throw new Error(`Sellbrite API error ${res.status}: ${body}`);
  }

  return res.json();
}

async function getProductsPage(page = 1, limit = 100) {
  return sellbriteGet(`/products?page=${page}&limit=${limit}`);
}

async function getInventoryPage(page = 1, limit = 100) {
  return sellbriteGet(`/inventory?page=${page}&limit=${limit}`);
}

async function getProductBySku(sku: string) {
  return sellbriteGet(`/products/${encodeURIComponent(sku)}`);
}
```

## pagination approach

```ts
async function getAllPages(path: string, limit = 100) {
  const all: unknown[] = [];
  let page = 1;

  while (true) {
    const data = await sellbriteGet(`${path}?page=${page}&limit=${limit}`);

    if (!Array.isArray(data) || data.length === 0) {
      break;
    }

    all.push(...data);

    if (data.length < limit) {
      break;
    }

    page += 1;
  }

  return all;
}
```

## final recommended audit

pull:

```text
/products
/inventory
/variation_groups
```

then answer:

1. does sellbrite expose parent products with embedded variations?
2. do child variations have `parent_sku`?
3. do parent products have `variations[]`?
4. does `/variation_groups` contain all variable parents?
5. are duplicate skus present?
6. do duplicate skus come from multiple channels?
7. can sellbrite grouping be used to validate veeqo grouping?

## key takeaway

to detect mismatched sku bases, we need known product grouping first.

veeqo flat variants are not enough.

sellbrite may provide the missing grouping layer through:

```text
variation groups
parent_sku
variation_keys
variation_fields
variations[]
```

if so, sellbrite can be used as a reference system for product grouping while veeqo remains the operational inventory/channel system.
