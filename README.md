# Documentation for Jarir Product Scraper

## Overview

This script is designed to scrape product data from the **Jarir** website's API. It extracts product details such as name, specifications, pricing, stock status, and brand, and saves the data into a **PostgreSQL** database. It supports pagination and can be reused for multiple product categories by modifying the **API URL** and hardcoded **category name**.

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
```python
HEADERS = {
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36',
    'Accept': 'application/json',
    'Accept-Language': 'en-US,en;q=0.9,ar-EG;q=0.8,ar;q=0.7',
    'Cache-Control': 'no-cache',
    'Content-Type': 'application/json',
    'Connection': 'keep-alive',
    'Host': 'www.jarir.com', //replace with the store host 
    'Referer': 'https://www.jarir.com/', //replace with the store referer
    'Pragma': 'no-cache',
}
```

### 3. **`extract_product_data`**
Extracts and processes data for each product:       
```python

// based on the store 

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
        "category": "smartphone",  # Replace with your desired category
        "datetime": current_time_str,
        "stock": stock,
        "store": "jarir"
    }
```

### 4. **`fetch_products`**
Fetches product data using pagination and saves it to the database:
```python
def fetch_products():
    start_index = 0
    all_products = []

    with requests.Session() as session:
        session.headers.update(HEADERS)

        while True:
            try:
                # Construct the API URL with pagination
                url = f"{API_URL}{start_index}"
                response = session.get(url)
                if response.status_code != 200:
                    print(f"Failed to fetch data at index {start_index}, status code: {response.status_code}")
                    break

                data = response.json()
                hits = data.get('hits', {}).get('hits', [])
                total_hits = data.get('hits', {}).get('total', 0)

                if not hits:
                    print(f"No more products found at index {start_index}. Stopping.")
                    break

                for product_data in hits:
                    product = product_data.get('_source', {})
                    all_products.append(extract_product_data(product))

                if len(all_products) >= total_hits:
                    print(f"Fetched all {total_hits} products.")
                    break

                start_index += 12

            except Exception as e:
                print(f"An error occurred at index {start_index}: {e}")
                time.sleep(5)  # Retry after delay

    save_to_postgresql(all_products)
```

### 5. **`save_to_postgresql`**
Inserts product data into the PostgreSQL database using batch inserts:
```python
def save_to_postgresql(products):
    try:
        conn = connection_pool.getconn()
        cursor = conn.cursor()

        insert_query = f"""
        INSERT INTO {tablename} (name, specs, new_price, old_price, link, brand, category, datetime, stock, store)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        """

        product_data = [
            (
                product['name'],
                product['specs'],
                product['new_price'],
                product['old_price'],
                product['link'],
                product['brand'],
                product['category'],
                product['datetime'],
                product['stock'],
                product["store"]
            )
            for product in products
        ]

        execute_batch(cursor, insert_query, product_data)
        conn.commit()
        connection_pool.putconn(conn)

        print(f"Successfully saved {len(products)} products to the PostgreSQL database.")
    except Exception as e:
        print(f"An error occurred while saving to the database: {e}")
```

---

## Reusing for Other Categories

1. Update the **API URL**:
   ```python
   API_URL = "https://www.jarir.com/api/catalogv1/product/store/sa-en/category_ids/<CATEGORY_ID>/aggregation/true/size/12/from/"
   ```
   Replace `<CATEGORY_ID>` with the desired category's ID.

2. Update the **category** field in `extract_product_data`:
   ```python
   "category": "new_category"  # Replace "new_category" with the desired category name
   ```

3. Run the script

---
