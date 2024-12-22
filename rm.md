# Documentation for Product Scraper (Jarir and Extra)

## Overview

This script is designed to scrape product data from the **Jarir** and **Extra** websites' APIs. It extracts product details such as name, specifications, pricing, stock status, and brand, and saves the data into a **PostgreSQL** database. It supports pagination and can be reused for multiple product categories by modifying the **API URL** and hardcoded **category name**.

The script uses the following core Python libraries and modules:

- `psycopg2` for interacting with the PostgreSQL database.
- `requests` for making HTTP API calls.
- `datetime` for timestamp formatting.
- `time` for handling delays in case of errors.

---

## Features

- **Pagination**: Fetches all products by iterating over API pages.
- **Dynamic Data Extraction**: Extracts key product details, including:
  - Product Name
  - Specifications
  - Prices (New and Old)
  - Product Link
  - Stock Status
  - Timestamp in GMT+3 timezone
- **PostgreSQL Integration**: Inserts product data into a database using batch inserts for optimal performance.
- **Error Handling**: Retries failed requests with a delay and logs issues.
- **Reusable for Multiple Categories**: By modifying the `API_URL` and `category` field, you can scrape data for any category.

---

## Setup

### Prerequisites

1. **Python**: Ensure Python 3.7+ is installed.
2. **PostgreSQL Database**: Set up a PostgreSQL database and table.
3. **Libraries**: Install the required libraries:
   ```bash
   pip install psycopg2 requests
   ```

### Database Configuration

Create a table in your PostgreSQL database using the following schema:

```sql
CREATE TABLE test (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    specs TEXT,
    new_price NUMERIC,
    old_price NUMERIC,
    link TEXT,
    brand TEXT,
    category TEXT,
    datetime TIMESTAMP,
    stock BOOLEAN,
    store TEXT
);
```

Update the `config.py` file with your database credentials:

```python
DB_CONFIG = {
    "host": "localhost",
    "port": 5432,
    "dbname": "your_database_name",
    "user": "your_username",
    "password": "your_password",
    "tablename": "test"  # Replace "test" with your table name if necessary
}
```

---

## Code Functions and Workflow

### 1. **Connection Pool Setup**

Creates a connection pool for managing database connections efficiently:

```python
connection_pool = psycopg2.pool.SimpleConnectionPool(
    minconn=1,
    maxconn=100,
    host=DB_CONFIG["host"],
    port=DB_CONFIG["port"],
    dbname=DB_CONFIG["dbname"],
    user=DB_CONFIG["user"],
    password=DB_CONFIG["password"]
)
```

### 2. **API Headers**

Defines custom headers to mimic a browser request:

#### Jarir

```python
HEADERS = {
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36',
    'Accept': 'application/json',
    'Accept-Language': 'en-US,en;q=0.9,ar-EG;q=0.8,ar;q=0.7',
    'Cache-Control': 'no-cache',
    'Content-Type': 'application/json',
    'Connection': 'keep-alive',
    'Host': 'www.jarir.com',
    'Referer': 'https://www.jarir.com/',
    'Pragma': 'no-cache',
}
```

#### Extra

```python
HEADERS = {
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36',
    'Accept': 'application/json',
    'Accept-Language': 'en-US,en;q=0.9,ar-EG;q=0.8,ar;q=0.7',
    'Cache-Control': 'no-cache',
    'Content-Type': 'application/json',
    'Connection': 'keep-alive',
    'Host': 'search.unbxd.io',
    'Referer': 'https://www.extra.com/en-sa/',
    'Pragma': 'no-cache',
}
```

### 3. **`extract_product_data`**

Extracts and processes data for each product:

#### Jarir

```python
def extract_product_data(product):
    """
    Extract relevant data from a product dictionary, including name, specs, prices, and stock status.
    """
    # Format the current timestamp (GMT+3)
    current_time = datetime.utcnow() + timedelta(hours=3)
    current_time = current_time.replace(minute=0, second=0, microsecond=0)
    current_time_str = current_time.strftime('%Y-%m-%d %H:%M:%S')

    # Extract basic product details
    product_name = product.get('name', 'No Name Available')
    product_link = f"https://www.jarir.com/{product.get('url_key', 'No Link Available')}.html"

    # Extract and clean specifications
    specs = 'No Specifications Available'
    if 'name' in product:
        parts = product_name.split(',', 1)
        if len(parts) > 1:
            specs = parts[1].strip()
            product_name = parts[0].strip()

    # Prices and brand
    new_price = product.get('jarir_final_price', product.get('price', 0))
    old_price = product.get('price', new_price)
    brand = product.get('GTM_brand', 'No Brand Available')

    # Add GTM_cofa to specs if available
    gtm_cofa = product.get('GTM_cofa', '')
    if gtm_cofa and gtm_cofa != 'n/a':
        specs = f"{specs}, {gtm_cofa}" if specs != "No Specifications Available" else gtm_cofa

    # Extract category, stock, and store
    category = product.get('GTM_category', 'No Category Available')
    stock = product.get('klevu_stock_flag', 0) == 1  # Stock status as boolean

    return {
        "name": product_name,
        "specs": specs,
        "new_price": new_price,
        "old_price": old_price,
        "brand": brand,
        "link": product_link,
        "category": "smartphone", 
        "datetime": current_time_str,
        "stock": stock,
        "store": "jarir"
    }
```

#### Extra

```python
def extract_product_data(product):
    """
    Extract relevant data from a product dictionary, including the category and stock status.
    """
    # Get current time in GMT+3
    current_time = datetime.utcnow() + timedelta(hours=3)
    current_time = current_time.replace(minute=0, second=0, microsecond=0)
    current_time_str = current_time.strftime('%Y-%m-%d %H:%M:%S')

    # Extract product details
    product_name = product.get('nameEn', 'No Name Available')
    product_parts = product_name.split(',', 1)

    name = product_parts[0]
    additional_name = product_parts[1] if len(product_parts) > 1 else ''

    processor_core = product.get('featureEnProcessorCore', 'No Processor Info Available')

    specs = f"{additional_name}, {processor_core}" if additional_name else processor_core
    
    product_link = product.get('productUrl', 'No URL Available')

    # Extract price details
    new_price = product.get('sellingPrice', None)
    old_price = product.get('wasPrice', new_price)

    in_stock_flag = product.get('inStockFlag', False)
    stock = in_stock_flag

    raw_brand = product.get('brand', ['No Brand Available'])[0]
    brand = format_brand(raw_brand)

    return {
        "name": name,
        "specs": specs,
        "new_price": new_price,
        "old_price": old_price,
        "brand": brand,
        "link": product_link,
        "category": 'laptop',
        "datetime": current_time_str,
        "stock": stock,
        "store": "extra"
    }
```

### 4. **`fetch_products`**

Fetches product data using pagination and saves it to the database. Both versions for Jarir and Extra are similar, but the API URLs and data structure differ.

---

## Reusing for Other Categories

### API URLs and Categories Table

The following table lists the API URLs and their corresponding categories for both Jarir and Extra:

| **Store** | **Category**     | **API URL** |
|-----------|------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Extra     | Smartphone       | [Link](https://search.unbxd.io/21705619e273429e5767eea44ccb1ad5/ss-unbxd-auk-extra-saudi-en-prod11541714990488/category?stats=price&selectedfacet=true&facet.multiselect=true&page=0&rows=96&bfrule=inStockCities%3A%22SA-riyadh%22+OR+sellingOutFastCities%3A%22SA-riyadh%22+OR+restockableCities%3ASA-riyadh&boost=if%28eq%28query%28%24bfrule%29%2Cfalse%29%2C0%2C1%29&filter=type%3APRODUCT&p=categories_uFilter%3A%223-303%22&pagetype=boolean&facet=true&version=V2&uid=uid-1726931238891-10989) |
| Extra     | Laptop           | [Link](https://search.unbxd.io/21705619e273429e5767eea44ccb1ad5/ss-unbxd-auk-extra-saudi-en-prod11541714990488/category?stats=price&selectedfacet=true&facet.multiselect=true&page=1&rows=96&bfrule=inStockCities%3A%22SA-riyadh%22+OR+sellingOutFastCities%3A%22SA-riyadh%22+OR+restockableCities%3ASA-riyadh&boost=if%28eq%28query%28%24bfrule%29%2Cfalse%29%2C0%2C1%29&filter=familyEn_uFilter%3AMobiles&filter=type%3APRODUCT&p=categories_uFilter%3A%222%22&pagetype=boolean&facet=true&version=V2&uid=uid-1726931238891-10989) |
| Extra     | Tablet           | [Link](https://search.unbxd.io/21705619e273429e5767eea44ccb1ad5/ss-unbxd-auk-extra-saudi-en-prod11541714990488/category?stats=price&selectedfacet=true&facet.multiselect=true&page=2&rows=96&bfrule=inStockCities%3A%22SA-riyadh%22+OR+sellingOutFastCities%3A%22SA-riyadh%22+OR+restockableCities%3ASA-riyadh&boost=if%28eq%28query%28%24bfrule%29%2Cfalse%29%2C0%2C1%29&filter=familyEn_uFilter%3ATablets&filter=type%3APRODUCT&p=categories_uFilter%3A%222%22&pagetype=boolean&facet=true&version=V2&uid=uid-1726931238891-10989) |
| Jarir     | Smartphone       | [Link](https://www.jarir.com/api/catalogv1/product/store/sa-en/category_ids/1008/aggregation/true/size/12/from/) |
| Jarir     | Laptop           | [Link](https://www.jarir.com/api/catalogv1/product/store/sa-en/category_ids/1331/aggregation/true/size/12/from/) |
| Jarir     | Tablet           | [Link](https://www.jarir.com/api/catalogv1/product/store/sa-en/category_ids/1329/aggregation/true/size/12/from/) |

To scrape data for a specific category, update the **API URL** in the script and ensure the correct category name is used.

---

