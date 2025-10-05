# Prosfores - A Smart Price Comparison App

Welcome to the showcase repository for Prosfores, a full-stack application designed to help users in Greece find the best deals on grocery products by scraping, comparing, and unifying items from multiple supermarkets.

**Note:** This repository is for demonstration purposes. It contains detailed documentation and selected code snippets but does not include the full source code.

---

## 🚀 Key Features

- **Multi-Supermarket Scraping:** Automatically scrapes product data from 8 major Greek supermarkets (AB, Sklavenitis, MyMarket, Masoutis, etc.).
- **Intelligent Product Search:** A powerful search engine that allows users to find products across all available stores, with filters for specific supermarkets.
- **Advanced Product Unification:** A custom Natural Language Processing (NLP) pipeline that intelligently identifies and groups the same product sold across different stores, even with variations in naming.
- **Personalized Watchlists:** Registered users can create multiple watchlists to track prices of their favorite items over time.
- **Price History:** Users can view the price history of a specific product to make informed purchasing decisions.
- **User Authentication:** Secure user registration and login system.
- **Responsive Frontend:** A mobile-first application built with React Native and Expo, ensuring a seamless experience on both web and mobile devices.

---

## 🖼️ Application Showcase

|                   Search & Filtering                    |                Watchlist Management                |
| :-----------------------------------------------------: | :------------------------------------------------: |
|       ![Search Demo](assets/prosfores_search.gif)       | ![Watchlist Demo](assets/prosfores_watchlists.gif) |
| A clean interface for searching and filtering products. |    Users can create and manage multiple lists.     |

---

## 🛠️ Technology Stack

| Area                 | Technologies                                                    |
| :------------------- | :-------------------------------------------------------------- |
| **Frontend**         | React Native, Expo, TypeScript, Axios, Formik, Yup              |
| **Backend**          | Python, FastAPI, SQLAlchemy, Pydantic, PostgreSQL               |
| **Web Scraping**     | Python, Playwright, BeautifulSoup, Requests, Concurrent Futures |
| **Product Matching** | Python, Sentence-Transformers, HDBSCAN, Aho-Corasick            |
| **Deployment**       | Render (for backend)                                            |

---

## 🏛️ Architecture & System Design

Prosfores is built on a robust, multi-component architecture designed for scalability, efficiency, and accuracy.

\*\*

### 1. Backend API

The core of the application is a RESTful API built with **FastAPI**. It handles user authentication, product searches, and watchlist management.

- **Performance-Optimized Queries:** The API uses `SQLAlchemy` Core to execute highly optimized raw SQL queries. A key example is the use of `LATERAL JOIN` to efficiently fetch the _latest_ price for each product without the performance overhead of traditional joins or subqueries on large tables.
- **Data Validation:** `Pydantic` models are used for robust request and response validation, ensuring data integrity.
- **Asynchronous by Nature:** Leverages FastAPI's async capabilities to handle concurrent requests efficiently.

### 2. Web Scraping System

A sophisticated, independent Python system is responsible for populating the database.

- **Concurrent Scraping:** The system uses `concurrent.futures.ThreadPoolExecutor` to run scrapers for multiple supermarkets simultaneously, significantly reducing the total time required to update the data.
- **Resilient & Robust:** Each scraper is wrapped in error handling and uses a custom `fetch_with_retry` helper with exponential backoff to handle network failures. A scraper is considered "failed" if it returns an insufficient number of products, preventing partial data from corrupting the database.
- **Hybrid Scraping Techniques:**
  - **API-based:** For modern sites like AB Vasilopoulos and Galaxias, the scraper reverse-engineers their GraphQL/REST APIs. It even includes a `Playwright`-based mechanism to automatically extract fresh API hash keys when they expire.
  - **HTML Parsing:** For traditional websites, `BeautifulSoup` and `requests` are used to parse HTML content.
- **Production-Ready Logging:** A centralized logging configuration creates rotating log files for each scraper, making it easy to debug failures and monitor activity.

### 3. Product Unification (NLP & Clustering)

This is the most complex and innovative part of the system. To compare "apples to apples," products from different stores must be unified. This is achieved through a multi-step NLP pipeline.

1.  **Brand Extraction:** A custom `BrandExtractor` class uses regex and an Aho-Corasick automaton built from a pre-compiled list of over 1000 brands to accurately identify the brand in a product title.
2.  **NER Component Parsing:** A `ProductExtractor` class then parses the remaining product title to extract key components (base product, size, volume, package type, variants like flavor or color) using a series of rule-based regex patterns.
3.  **Constraint-Based Pre-Grouping:** Products are first grouped by their hard constraints (Brand + Size + Package Type). This dramatically reduces the search space for the next step.
4.  **Semantic Clustering:** Within each constraint group, product embeddings are generated using the `paraphrase-multilingual-MiniLM-L12-v2` model from `sentence-transformers`. **HDBSCAN** is then used to cluster products based on the semantic similarity of their names, effectively grouping "Kellogg's Krave Choco Nut 410gr" with "KELLOGGS Δημητριακά Krave Πραλίνα Φουντουκιού 410g".

\*\*

### 4. Frontend Application

The user-facing application is built with **React Native & Expo**, allowing it to be deployed on the web, iOS, and Android from a single codebase.

- **Modern React Practices:** The app is built with functional components and extensively uses custom hooks for state management (`useSearchState`, `useShopSelection`, `useWatchlistData`) to create clean, decoupled, and reusable logic.
- **Optimized Performance:** Techniques like `React.memo`, `useCallback`, `useMemo`, and debouncing with `lodash` are used throughout the application to prevent unnecessary re-renders and API calls, ensuring a smooth user experience.
- **State Management:** A combination of `React Context` for global state (like authentication) and component-level state (`useState`) is used for a predictable and manageable state flow.
- **Responsive & Adaptive UI:** The UI is designed to adapt to different screen sizes, with responsive grid layouts and platform-aware components.

---

## ✨ Code Snippets Showcase

Here are a few snippets that highlight the quality and complexity of the code.

### Backend: Optimized Product Search Query

This SQL query, executed via SQLAlchemy, uses a `LATERAL JOIN` to efficiently fetch the single latest price for each product. This is significantly faster than other methods on large datasets.

```python
def search_for_product(
    db: Session, user_input: str | None, shop_ids: List[int] | None = [], limit: int = 20, offset: int = 0
):
    from sqlalchemy.sql import text

    if not user_input:
        return [], False

    search_terms = normalize_name(user_input).split()
    sql = """SELECT
            p.id, p.name, p.link, p.shop_id, s.name AS shop_name,
            lp.regular_price, lp.discounted_price, lp.discount_percentage, lp.created_at AS price_created_at
        FROM products p
        JOIN shops s ON p.shop_id = s.id
        LEFT JOIN LATERAL (
            SELECT ph.*
            FROM price_history ph
            WHERE ph.product_id = p.id
            ORDER BY ph.created_at DESC
            LIMIT 1
        ) lp ON true
        WHERE
            s.id = ANY(:shop_ids)"""

    params = { "shop_ids": shop_ids or None, "limit": limit + 1, "offset": offset }
    for i, term in enumerate(search_terms):
        sql += f" AND p.name_normalized ILIKE :term{i} "
        params[f"term{i}"] = f"%%{term}%%"
    sql += " ORDER BY lp.created_at DESC NULLS LAST LIMIT :limit OFFSET :offset"

    rows = db.execute(text(sql), params).fetchall()
    # ... (code to process rows) ...
```

### Frontend: Custom Hook for API Search Logic

This custom hook encapsulates all logic for handling product searches, including debouncing user input, managing loading/pagination state, and fetching results from the API. This makes the main component cleaner and more readable.

```typescript
export const useSearchAPI = (
  selectedShopIds: number[],
  setSuggestions: (suggestions: Product[]) => void
  // ... other setters
) => {
  const debouncedFetchSuggestions = useMemo(
    () =>
      debounce((user_input: string) => {
        // ... API call logic for suggestions
      }, 300),
    [selectedShopIds, setSuggestions]
  );

  const fetchResults = useCallback(
    async (search: string, reset = false, currentOffset = 0) => {
      if (!search || search.length < MIN_CHAR || selectedShopIds.length === 0)
        return;

      setLoadingMore(true);
      try {
        const res = await api.get("/products/search", {
          /* params */
        });
        const fetched = res.data.products || [];

        if (reset) {
          setFullResults(fetched);
          setOffset(fetched.length);
        } else {
          setFullResults((prev) => [...prev, ...fetched]);
          setOffset((prev) => prev + fetched.length);
        }
        setHasMore(res.data.has_more);
      } catch (err: any) {
        // ... error handling
      } finally {
        setLoadingMore(false);
      }
    },
    [selectedShopIds /* ... other dependencies */]
  );

  return { debouncedFetchSuggestions, fetchResults };
};
```

### Product Matching: Core Clustering Logic

This snippet shows the core logic for the product unification system. It groups products by brand and size, then uses `sentence-transformers` to create embeddings and `HDBSCAN` to cluster them based on semantic similarity.

```python
def cluster_by_constraints(
    products: List[ProductComponents], model: SentenceTransformer, count_shops: int = 6
) -> Dict[int, List[ProductComponents]]:
    # Step 1: Group by brand + size + package
    constraint_groups = defaultdict(list)
    for product in products:
        brand = product.brand.lower().strip() if product.brand else "unknown"
        size = f"{product.size_value}{product.size_unit}" if product.size_value and product.size_unit else "no_size"
        package = product.package_type.lower().strip() if product.package_type else "no_package"
        constraint_key = f"{brand}|{size}|{package}"
        constraint_groups[constraint_key].append(product)

    # Step 2: Cluster within each group using embeddings
    final_clusters = {}
    cluster_counter = 0
    for constraint_key, group_products in constraint_groups.items():
        if len(group_products) < 2:
            final_clusters[cluster_counter] = group_products
            cluster_counter += 1
            continue

        embedding_texts = [p.base_product + " " + " ".join(p.variants) for p in group_products]
        embeddings = model.encode(embedding_texts, convert_to_numpy=True)

        clusterer = hdbscan.HDBSCAN(min_cluster_size=2, max_cluster_size=count_shops)
        labels = clusterer.fit_predict(embeddings)

        # ... (code to assign products to final clusters) ...

    return final_clusters
```

---

Thank you for viewing my project!
