rgetting any thing Documentary: Jarir Products Scraper
This script is designed to scrape product data from Jarir’s API and save it to a PostgreSQL database. The code is modular, allowing it to handle multiple categories by simply changing the base API URL.

Code Overview
1. Database Connection Setup
The script utilizes psycopg2 to establish a connection pool with the PostgreSQL database. This ensures efficient management of multiple database connections.
Code:
connection_pool = psycopg2.pool.SimpleConnectionPool(
    minconn=1,
    maxconn=100,
    host=DB_CONFIG["host"],
    port=DB_CONFIG["port"],
    dbname=DB_CONFIG["dbname"],
    user=DB_CONFIG["user"],
    password=DB_CONFIG["password"]
)
Why:
* A connection pool allows reuse of connections, reducing the overhead of repeatedly opening and closing them.
* Configuration parameters (e.g., host, port, dbname) are stored in a separate config file for better maintainability.

2. API Configuration
The script fetches data from Jarir’s API. A base URL is defined, and pagination is handled by incrementing the start_index parameter.
Code:
API_URL = "https://www.jarir.com/api/catalogv1/product/store/sa-en/category_ids/1008/aggregation/true/size/12/from/"
Headers:
Custom headers are included to mimic a real browser request and avoid being blocked.
HEADERS = {
    'User-Agent': 'Mozilla/5.0 ... Safari/537.36',
    'Accept': 'application/json',
    ...
}

3. Product Data Extraction
The extract_product_data function parses each product’s JSON data, extracting relevant fields such as:
* Name
* Specifications
* Prices (new and old)
* Stock status
* Category
Code:
def extract_product_data(product):
    current_time = datetime.utcnow() + timedelta(hours=3)
    product_name = product.get('name', 'No Name Available')
    product_link = f"https://www.jarir.com/{product.get('url_key', 'No Link Available')}.html"
    ...
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
Why: This modular approach ensures that the data is consistent and ready for insertion into the database.

4. Fetching Products
The fetch_products function handles:
* Pagination
* HTTP requests with retries in case of failures
* Parsing the API response
Code:
def fetch_products():
    start_index = 0
    all_products = []
    with requests.Session() as session:
        session.headers.update(HEADERS)
        while True:
            url = f"{API_URL}{start_index}"
            response = session.get(url)
            if response.status_code != 200:
                break
            data = response.json()
            hits = data.get('hits', {}).get('hits', [])
            if not hits:
                break
            for product_data in hits:
                product = product_data.get('_source', {})
                all_products.append(extract_product_data(product))
            start_index += 12
Why:
* Uses a while loop for continuous pagination until all products are fetched.
* Sessions improve performance by reusing TCP connections.
* Error handling ensures the script doesn’t crash on temporary API issues.

5. Saving to PostgreSQL
The save_to_postgresql function batches product inserts into the database for efficiency.
Code:
def save_to_postgresql(products):
    conn = connection_pool.getconn()
    cursor = conn.cursor()
    insert_query = f"""
    INSERT INTO {tablename} (name, specs, new_price, old_price, link, brand, category, datetime, stock, store)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    """
    execute_batch(cursor, insert_query, product_data)
    conn.commit()
    connection_pool.putconn(conn)
Why:
* Batching reduces the number of database round-trips.
* Error handling ensures the database connection is always returned to the pool.

6. Running the Script
The main entry point is the fetch_products function, which orchestrates data extraction and storage.
Code:
fetch_products()

Adapting for Other Categories
The script can be reused for different categories by simply changing the API_URL to point to a different category_ids parameter.
Example:
To fetch laptops instead of smartphones:
API_URL = "https://www.jarir.com/api/catalogv1/product/store/sa-en/category_ids/2008/aggregation/true/size/12/from/"

Key Features
1. Scalable: Handles large datasets with pagination and batching.
2. Reusable: Easily adaptable for different product categories.
3. Efficient: Uses connection pooling and batch inserts for optimal performance.
4. Resilient: Includes error handling for API and database interactions.


This script provides a robust foundation for scraping and storing product data efficiently, while its modular design ensures easy maintenance and adaptability.

