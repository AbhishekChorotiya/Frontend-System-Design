# Pagination, Filtering, and Sorting

Almost every data-driven frontend application needs to display collections of items -- products in a catalog, messages in an inbox, rows in an admin dashboard. When those collections grow beyond a trivial size, delivering them all at once becomes impractical: responses balloon, render times spike, and users drown in information they never asked for. Pagination, filtering, and sorting are the three mechanisms that let a frontend request *just the right slice* of data, present it in a meaningful order, and let users narrow down what they see. Together they form the backbone of any list-based UI, and getting them right is one of the most common challenges in frontend system design.

From a system design perspective, these three concerns are deeply intertwined. A user filters a product list by category, sorts by price ascending, and pages through the results ten at a time. The frontend must coordinate filter state, sort parameters, and the current page into a single coherent API request, keep the URL in sync so the view is shareable and bookmarkable, and handle edge cases like empty pages, stale cursors, and mid-scroll data mutations. Understanding how each piece works -- and how they compose -- is essential for building performant, user-friendly interfaces.

> **Think of it like a library catalog.** Pagination is the card catalog splitting thousands of entries across numbered drawers so you only pull open one at a time. Filtering is telling the librarian "only show me science fiction from the 1970s." Sorting is deciding whether those filtered results should be arranged by title, author, or publication date. Each mechanism is useful on its own, but a real patron uses all three together -- and so do your users.

## Core Concepts

1. **Offset-Based Pagination:** The simplest pagination strategy. The client sends a `page` number and a `limit` (page size) to the server, which translates them into an SQL `OFFSET` and `LIMIT`. Easy to implement and gives users random access to any page, but performance degrades on large datasets because the database still scans past all skipped rows.

2. **Cursor-Based Pagination:** Instead of a page number, the client sends an opaque *cursor* (typically an encoded ID or timestamp) that marks where the last page ended. The server returns results after that cursor along with a new cursor for the next page. This approach is stable under insertions and deletions and performs well at any depth, but users cannot jump to an arbitrary page.

3. **Filtering (Client vs Server):** Filtering removes items that do not match a set of criteria. *Client-side filtering* works on data already loaded in the browser -- fast for small datasets, but impractical when the full dataset is too large to download. *Server-side filtering* sends filter parameters as query strings or request body fields, and the server applies them before returning results. Most production apps use server-side filtering with client-side filtering reserved for refining an already-fetched page.

4. **Sorting (Client vs Server):** Sorting arranges results in a given order. Like filtering, it can happen on the client for small, already-loaded datasets, or on the server where the database handles it efficiently. Server-side sorting is necessary when only a page of data is loaded at a time, because the client cannot sort globally when it only has a subset.

5. **Search and Debouncing:** Search is a specialized form of filtering where the user types a free-text query. Because every keystroke could trigger an API call, *debouncing* -- delaying the request until the user pauses typing -- is essential to avoid flooding the server and creating a janky experience.

## How It Works

The typical flow for a paginated, filtered, sortable list involves several coordinated steps between the client and server:

### Step-by-Step Flow

1. **User interacts with controls.** The user types into a search box, selects a filter dropdown, clicks a column header to sort, or navigates to a new page.

2. **Frontend updates local state.** The component updates its internal state (or a shared store) with the new filter values, sort field/direction, and page number or cursor.

3. **State is serialized into an API request.** The frontend builds a query string or request body from the current state. A typical REST request looks like:

```
GET /api/products?category=electronics&minPrice=50&sort=price&order=asc&page=2&limit=20
```

4. **Server processes the request.** The server parses parameters, applies filters to the database query, adds the `ORDER BY` clause for sorting, and applies `LIMIT`/`OFFSET` or cursor logic for pagination. It returns the matching page of results along with metadata (total count, next cursor, etc.).

5. **Frontend receives and renders.** The component receives the response, updates the displayed list, and adjusts pagination controls (page numbers, "Load More" button, or infinite scroll sentinel).

6. **URL is kept in sync.** The frontend writes the current filter/sort/page state into the URL query string so the view is shareable, bookmarkable, and survives page refreshes.

```typescript
// src/services/productApi.ts
interface FetchProductsParams {
  page?: number;
  limit?: number;
  cursor?: string;
  category?: string;
  minPrice?: number;
  maxPrice?: number;
  sortBy?: string;
  sortOrder?: 'asc' | 'desc';
  search?: string;
}

interface PaginatedResponse<T> {
  data: T[];
  meta: {
    total: number;
    page: number;
    limit: number;
    totalPages: number;
    nextCursor?: string;
  };
}

async function fetchProducts(params: FetchProductsParams): Promise<PaginatedResponse<Product>> {
  const searchParams = new URLSearchParams();

  if (params.page) searchParams.set('page', String(params.page));
  if (params.limit) searchParams.set('limit', String(params.limit));
  if (params.cursor) searchParams.set('cursor', params.cursor);
  if (params.category) searchParams.set('category', params.category);
  if (params.minPrice) searchParams.set('minPrice', String(params.minPrice));
  if (params.maxPrice) searchParams.set('maxPrice', String(params.maxPrice));
  if (params.sortBy) searchParams.set('sortBy', params.sortBy);
  if (params.sortOrder) searchParams.set('sortOrder', params.sortOrder);
  if (params.search) searchParams.set('search', params.search);

  const response = await fetch(`/api/products?${searchParams.toString()}`);
  if (!response.ok) {
    throw new Error(`Failed to fetch products: ${response.status}`);
  }
  return response.json();
}
```

## Pagination Strategies

### Offset-Based Pagination

Offset-based pagination uses a `page` and `limit` (or `offset` and `limit`) to slice into the dataset. The server skips `offset` rows and returns the next `limit` rows.

```typescript
// src/hooks/useOffsetPagination.ts
import { useState, useEffect, useCallback } from 'react';

interface UseOffsetPaginationOptions {
  initialPage?: number;
  pageSize?: number;
  fetchFn: (page: number, limit: number) => Promise<{ data: any[]; total: number }>;
}

function useOffsetPagination({ initialPage = 1, pageSize = 20, fetchFn }: UseOffsetPaginationOptions) {
  const [page, setPage] = useState(initialPage);
  const [data, setData] = useState<any[]>([]);
  const [total, setTotal] = useState(0);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<Error | null>(null);

  const totalPages = Math.ceil(total / pageSize);

  const fetchPage = useCallback(async (targetPage: number) => {
    setLoading(true);
    setError(null);
    try {
      const result = await fetchFn(targetPage, pageSize);
      setData(result.data);
      setTotal(result.total);
      setPage(targetPage);
    } catch (err) {
      setError(err instanceof Error ? err : new Error('Fetch failed'));
    } finally {
      setLoading(false);
    }
  }, [fetchFn, pageSize]);

  useEffect(() => {
    fetchPage(page);
  }, []);

  const goToPage = (targetPage: number) => {
    if (targetPage >= 1 && targetPage <= totalPages) {
      fetchPage(targetPage);
    }
  };

  const nextPage = () => goToPage(page + 1);
  const prevPage = () => goToPage(page - 1);

  return {
    data,
    page,
    totalPages,
    total,
    loading,
    error,
    goToPage,
    nextPage,
    prevPage,
    hasNextPage: page < totalPages,
    hasPrevPage: page > 1,
  };
}
```

### Cursor-Based Pagination

Cursor-based pagination uses an opaque token (cursor) to mark the position in the dataset. The server returns a cursor alongside each page, and the client sends it back to fetch the next batch.

```typescript
// src/hooks/useCursorPagination.ts
import { useState, useCallback } from 'react';

interface CursorPage<T> {
  data: T[];
  nextCursor: string | null;
  hasMore: boolean;
}

interface UseCursorPaginationOptions<T> {
  pageSize?: number;
  fetchFn: (cursor: string | null, limit: number) => Promise<CursorPage<T>>;
}

function useCursorPagination<T>({ pageSize = 20, fetchFn }: UseCursorPaginationOptions<T>) {
  const [data, setData] = useState<T[]>([]);
  const [cursor, setCursor] = useState<string | null>(null);
  const [hasMore, setHasMore] = useState(true);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<Error | null>(null);

  const fetchNextPage = useCallback(async () => {
    if (loading || !hasMore) return;

    setLoading(true);
    setError(null);
    try {
      const result = await fetchFn(cursor, pageSize);
      setData(prev => [...prev, ...result.data]);
      setCursor(result.nextCursor);
      setHasMore(result.hasMore);
    } catch (err) {
      setError(err instanceof Error ? err : new Error('Fetch failed'));
    } finally {
      setLoading(false);
    }
  }, [cursor, fetchFn, hasMore, loading, pageSize]);

  const reset = useCallback(() => {
    setData([]);
    setCursor(null);
    setHasMore(true);
    setError(null);
  }, []);

  return { data, loading, error, hasMore, fetchNextPage, reset };
}
```

### Keyset Pagination

Keyset pagination is a variant of cursor-based pagination where the cursor is the value of the sort column from the last row. It avoids the `OFFSET` performance penalty and works well when results are sorted by a unique, indexed column.

```
-- Instead of:  SELECT * FROM products ORDER BY price OFFSET 1000 LIMIT 20
-- Keyset uses: SELECT * FROM products WHERE price > 29.99 ORDER BY price LIMIT 20
```

### Comparison Table

| Aspect | Offset-Based | Cursor-Based | Keyset |
|--------|-------------|-------------|--------|
| **Random page access** | Yes -- users can jump to page 5 | No -- must traverse sequentially | No -- must traverse sequentially |
| **Performance at depth** | Degrades -- DB scans skipped rows | Constant -- no offset scan | Constant -- uses indexed `WHERE` clause |
| **Stability under mutations** | Fragile -- inserts/deletes shift pages | Stable -- cursor anchors position | Stable -- anchored to column value |
| **Implementation complexity** | Low | Medium | Medium |
| **Total count available** | Yes (extra query) | Not inherently | Not inherently |
| **Best for** | Admin tables, small datasets | Social feeds, infinite scroll | Time-series, large sorted datasets |

## Filtering Patterns

### Client-Side Filtering

Client-side filtering works on data that has already been fetched. It is fast and responsive because no network request is needed, but it only works when the entire dataset (or a sufficiently large subset) is in memory.

```typescript
// src/hooks/useClientFilter.ts

// ✅ Good: Client-side filtering on an already-loaded dataset
function useClientFilter<T>(items: T[], filterFn: (item: T) => boolean) {
  return useMemo(() => items.filter(filterFn), [items, filterFn]);
}

// Usage
const products = useClientFilter(allProducts, (product) =>
  product.category === selectedCategory && product.price >= minPrice
);
```

```typescript
// ❌ Bad: Fetching ALL items just to filter on the client
async function getAllProductsThenFilter(category: string) {
  const response = await fetch('/api/products'); // Fetches thousands of items
  const allProducts = await response.json();
  return allProducts.filter((p: Product) => p.category === category);
}
```

### Server-Side Filtering

Server-side filtering sends filter parameters to the API and lets the database handle the work. This is the correct approach for large datasets.

```typescript
// src/services/api.ts
async function fetchFilteredProducts(filters: Record<string, string | number>) {
  const params = new URLSearchParams();
  Object.entries(filters).forEach(([key, value]) => {
    if (value !== undefined && value !== '') {
      params.set(key, String(value));
    }
  });

  const response = await fetch(`/api/products?${params.toString()}`);
  if (!response.ok) throw new Error('Failed to fetch');
  return response.json();
}
```

### URL-Synced Filter State

Storing filter state in the URL makes views shareable, bookmarkable, and resilient to page refreshes. This pattern uses `URLSearchParams` and a router to keep the URL and component state synchronized.

```tsx
// src/components/ProductList.tsx
import { useSearchParams } from 'react-router-dom';
import { useCallback, useEffect, useState } from 'react';

function ProductList() {
  const [searchParams, setSearchParams] = useSearchParams();
  const [products, setProducts] = useState<Product[]>([]);
  const [loading, setLoading] = useState(false);

  // Read filter state from URL
  const filters = {
    category: searchParams.get('category') || '',
    minPrice: searchParams.get('minPrice') || '',
    maxPrice: searchParams.get('maxPrice') || '',
    sortBy: searchParams.get('sortBy') || 'createdAt',
    sortOrder: searchParams.get('sortOrder') || 'desc',
    page: Number(searchParams.get('page')) || 1,
  };

  // Update URL when filters change
  const updateFilters = useCallback(
    (newFilters: Partial<typeof filters>) => {
      const updated = { ...filters, ...newFilters, page: '1' }; // Reset to page 1 on filter change
      const params = new URLSearchParams();
      Object.entries(updated).forEach(([key, value]) => {
        if (value && value !== '' && value !== '0') {
          params.set(key, String(value));
        }
      });
      setSearchParams(params);
    },
    [filters, setSearchParams]
  );

  // Fetch data whenever URL params change
  useEffect(() => {
    setLoading(true);
    fetchProducts(filters)
      .then((res) => setProducts(res.data))
      .finally(() => setLoading(false));
  }, [searchParams]);

  return (
    <div>
      <FilterBar filters={filters} onChange={updateFilters} />
      {loading ? <Spinner /> : <ProductGrid products={products} />}
      <Pagination
        page={filters.page}
        onPageChange={(page) => updateFilters({ page })}
      />
    </div>
  );
}
```

## Sorting Patterns

### Single-Column Sorting

The most common pattern. Clicking a column header sorts by that column, clicking again toggles the direction.

```typescript
// src/hooks/useSort.ts
import { useState, useCallback } from 'react';

type SortDirection = 'asc' | 'desc';

interface SortState {
  field: string;
  direction: SortDirection;
}

function useSort(defaultField: string, defaultDirection: SortDirection = 'asc') {
  const [sort, setSort] = useState<SortState>({ field: defaultField, direction: defaultDirection });

  const toggleSort = useCallback((field: string) => {
    setSort((prev) => {
      if (prev.field === field) {
        return { field, direction: prev.direction === 'asc' ? 'desc' : 'asc' };
      }
      return { field, direction: 'asc' };
    });
  }, []);

  return { sort, toggleSort };
}
```

### Multi-Column Sorting

Some data-heavy UIs (admin dashboards, spreadsheets) support sorting by multiple columns in priority order. The user holds Shift and clicks additional column headers.

```typescript
// src/hooks/useMultiSort.ts
interface SortRule {
  field: string;
  direction: 'asc' | 'desc';
}

function useMultiSort() {
  const [sortRules, setSortRules] = useState<SortRule[]>([]);

  const toggleSort = useCallback((field: string, multi = false) => {
    setSortRules((prev) => {
      const existingIndex = prev.findIndex((r) => r.field === field);

      if (existingIndex >= 0) {
        const updated = [...prev];
        if (updated[existingIndex].direction === 'asc') {
          updated[existingIndex] = { field, direction: 'desc' };
        } else {
          updated.splice(existingIndex, 1); // Third click removes the sort
        }
        return updated;
      }

      const newRule: SortRule = { field, direction: 'asc' };
      return multi ? [...prev, newRule] : [newRule];
    });
  }, []);

  return { sortRules, toggleSort };
}
```

### Server-Side vs Client-Side Sorting

```typescript
// ✅ Good: Server-side sorting when paginated (client only has one page of data)
const response = await fetch('/api/products?sortBy=price&sortOrder=asc&page=1&limit=20');

// ✅ Good: Client-side sorting when ALL data is loaded
const sorted = useMemo(() => {
  return [...products].sort((a, b) => {
    const modifier = sortDirection === 'asc' ? 1 : -1;
    if (a[sortField] < b[sortField]) return -1 * modifier;
    if (a[sortField] > b[sortField]) return 1 * modifier;
    return 0;
  });
}, [products, sortField, sortDirection]);

// ❌ Bad: Client-side sorting on paginated data (only sorts the visible page, not the full dataset)
const sorted = currentPageItems.sort((a, b) => a.price - b.price);
```

## Debounced Search

Search inputs should be debounced to avoid firing an API request on every keystroke. A typical debounce delay is 300-500ms.

```typescript
// src/hooks/useDebouncedSearch.ts
import { useState, useEffect, useRef } from 'react';

function useDebouncedValue<T>(value: T, delay: number = 300): T {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}

// Usage in a search component
function SearchInput({ onSearch }: { onSearch: (query: string) => void }) {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebouncedValue(query, 400);

  useEffect(() => {
    onSearch(debouncedQuery);
  }, [debouncedQuery, onSearch]);

  return (
    <input
      type="text"
      value={query}
      onChange={(e) => setQuery(e.target.value)}
      placeholder="Search products..."
    />
  );
}
```

## Infinite Scroll

Infinite scroll loads the next page of results automatically as the user scrolls near the bottom of the list. It uses the `IntersectionObserver` API to detect when a sentinel element enters the viewport.

```tsx
// src/components/InfiniteProductList.tsx
import { useRef, useEffect, useCallback } from 'react';

function InfiniteProductList() {
  const { data, loading, hasMore, fetchNextPage } = useCursorPagination({
    pageSize: 20,
    fetchFn: async (cursor, limit) => {
      const params = new URLSearchParams({ limit: String(limit) });
      if (cursor) params.set('cursor', cursor);
      const res = await fetch(`/api/products?${params}`);
      return res.json();
    },
  });

  const sentinelRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    if (!sentinelRef.current) return;

    const observer = new IntersectionObserver(
      (entries) => {
        if (entries[0].isIntersecting && hasMore && !loading) {
          fetchNextPage();
        }
      },
      { rootMargin: '200px' } // Start loading 200px before the sentinel is visible
    );

    observer.observe(sentinelRef.current);
    return () => observer.disconnect();
  }, [hasMore, loading, fetchNextPage]);

  return (
    <div>
      {data.map((product) => (
        <ProductCard key={product.id} product={product} />
      ))}
      {loading && <Spinner />}
      {hasMore && <div ref={sentinelRef} style={{ height: 1 }} />}
      {!hasMore && <p>No more products to load.</p>}
    </div>
  );
}
```

## Benefits

* **Reduced payload size:** Pagination ensures the server only serializes and transmits a small slice of the dataset per request, reducing bandwidth and parse time.
* **Faster initial render:** Loading 20 items instead of 10,000 means the browser paints the first meaningful frame sooner and uses less memory.
* **Better user experience:** Filtering and sorting let users find what they need without manually scanning a massive list, reducing cognitive load.
* **Server resource efficiency:** Server-side filtering and pagination push heavy work to the database where indexes make it fast, instead of transferring all data to the client.
* **Shareable views:** URL-synced filters and page state make it possible to share a link that reproduces the exact same view for another user.
* **Scalability:** These patterns allow the frontend to work with datasets of any size -- ten items or ten million -- without changing the UI architecture.

## Drawbacks and Challenges

* **Increased complexity:** Coordinating pagination, filter, sort, and URL state adds significant complexity compared to simply rendering a flat list.
* **Consistency under mutation:** When items are added or removed between page fetches, offset-based pagination can show duplicates or skip items. Cursor-based pagination mitigates this but does not eliminate it entirely.
* **Total count overhead:** Computing the total number of matching items for pagination controls requires an additional `COUNT(*)` query, which can be expensive on large tables.
* **State synchronization:** Keeping component state, URL query params, and API request params in sync is error-prone. A filter change should reset the page to 1, but developers often forget this.
* **Deep linking edge cases:** A URL pointing to page 47 may no longer be valid if the dataset has shrunk. The frontend must handle out-of-range pages gracefully.
* **SEO limitations:** Infinite scroll hides content behind user interaction, making it harder for search engine crawlers to index all items compared to traditional paginated pages with unique URLs.

## Use Cases

* **E-commerce catalogs:** Products filtered by category, brand, price range, and rating, sorted by relevance or price, displayed 24 per page. Offset-based pagination is typical here because users expect numbered page controls and the ability to jump to a specific page.

* **Social media feeds:** Posts from followed accounts, sorted by most recent or algorithmic relevance. Cursor-based pagination with infinite scroll is the standard because feeds are append-heavy and users scroll continuously.

* **Admin dashboards and data tables:** Rows of users, orders, or logs with multi-column sorting and complex filters. Offset-based pagination with a page size selector (10/25/50/100) is common because admins need to jump to specific pages and see total counts.

* **Search results:** A search query filters results, which are sorted by relevance score. The search input is debounced, and results use either offset pagination (Google-style page numbers) or cursor-based "Load More" buttons.

* **Chat and messaging:** Messages within a conversation loaded in reverse chronological order. Cursor-based pagination loads older messages as the user scrolls up. No filtering or sorting is typically needed.

* **Real-time dashboards:** Logs or events streaming in, where the user filters by severity level or source. Keyset pagination on a timestamp column is ideal because the data is naturally ordered and continuously growing.

## Best Practices

1. **Reset to page 1 when filters or sort changes.** When a user changes a filter or sort parameter, the current page may no longer exist in the new result set. Always reset the page number (or cursor) to the beginning.

2. **Store filter/sort/page state in the URL.** Use `URLSearchParams` and your router's search params API to persist the current view state. This makes views shareable, bookmarkable, and resilient to page refresh.

3. **Debounce text-based filters and search inputs.** Avoid firing an API request on every keystroke. A 300-500ms debounce is a reasonable default.

4. **Use cursor-based pagination for infinite scroll and large, mutation-heavy datasets.** Offset-based pagination is simpler but breaks under concurrent insertions. Cursor-based pagination is more robust for feeds and timelines.

5. **Provide visual feedback during loading.** Show skeleton loaders or spinners when fetching a new page. Disable pagination controls while a request is in-flight to prevent double-fetches.

6. **Handle empty and error states explicitly.** When filters produce zero results, show a clear "No results" message with a suggestion to broaden filters. When the API returns an error, display an error state with a retry option.

7. **Avoid fetching total count if you do not need it.** `COUNT(*)` on large tables is expensive. If you only need a "Load More" button or infinite scroll, use `hasMore` from cursor-based responses instead.

8. **Use server-side filtering and sorting for paginated data.** Client-side filtering and sorting only make sense when the full dataset is loaded. If you are paginating, the server must handle filtering and sorting to produce correct results.

9. **Set reasonable default page sizes.** A default of 20-25 items works well for most UIs. Provide a page size selector for data tables where power users may want 50 or 100 rows.

10. **Prefetch the next page.** For offset-based pagination, prefetch page N+1 when the user lands on page N. For cursor-based pagination, start loading the next batch before the user reaches the bottom of the list (use a `rootMargin` on the `IntersectionObserver`).

## Common Beginner Doubts or Questions

### When should I use cursor-based pagination instead of offset-based?

Use cursor-based pagination when working with large datasets that change frequently (e.g., social feeds, real-time logs), when performance at deep pages matters, or when implementing infinite scroll. Offset-based pagination suffers from two problems at scale: the database must scan past all skipped rows (making deep pages slow), and insertions or deletions between requests can cause items to appear twice or be skipped entirely.

Offset-based pagination is the better choice when users need to jump to a specific page number (e.g., "go to page 15 of 42"), when you need to display a total page count, or when the dataset is small and relatively static. Admin dashboards and e-commerce catalogs often use offset-based pagination because the random-access UX is more familiar.

### Should filtering happen on the client or the server?

It depends on how much data you have. If the entire dataset fits comfortably in memory (a few hundred items at most), client-side filtering gives instant results without network latency. But if you are paginating -- which implies the dataset is too large to load at once -- filtering *must* happen on the server. Otherwise you are only filtering within the current page, which gives the user incorrect results.

A common hybrid approach is to use server-side filtering for the primary query and then apply lightweight client-side filtering for UI-only refinements (e.g., highlighting or hiding items that match a local toggle) on the already-fetched page of results.

### How do I sync filters with the URL?

Use your router's search params API (e.g., `useSearchParams` in React Router) to read filter state from the URL on mount and write it back whenever filters change. The URL becomes the single source of truth: component state is derived from it, not the other way around. When the user changes a filter, update the URL search params, which triggers a re-render and a new API fetch. This makes the view automatically shareable and bookmarkable.

Watch out for a common pitfall: updating the URL replaces the browser history entry by default, which means the user cannot press "Back" to undo a filter change. Use `navigate` with `{ replace: true }` for frequent incremental changes (like typing in a search box) and a regular push for discrete actions (like selecting a category).

### What is infinite scroll and how does it compare to traditional pagination?

Infinite scroll automatically loads and appends the next page of results as the user scrolls near the bottom of the list. It uses cursor-based pagination under the hood with an `IntersectionObserver` watching a sentinel element. Traditional pagination shows explicit page controls (Previous/Next buttons or numbered page links) and replaces the entire list when navigating.

Infinite scroll feels seamless for browse-heavy experiences like social feeds and image galleries. But it has drawbacks: users cannot jump to a specific point in the dataset, the scroll position is hard to restore on navigation, the page consumes more memory as items accumulate, and search engines cannot easily crawl content behind the scroll. Traditional pagination is better for task-oriented interfaces (e-commerce, admin tables) where users want to know their position, share a specific page, or jump ahead.

### How do I handle sorting with pagination?

Sorting must happen on the server when the data is paginated. If you sort client-side on a paginated dataset, you are only reordering the items on the *current page*, not the full dataset -- the user would see a different result than if the sort were applied globally. Always send the sort field and direction as query parameters (e.g., `?sortBy=price&sortOrder=asc`) alongside the page number, and let the database handle the `ORDER BY`.

When the user changes the sort, reset the page back to 1 (or reset the cursor). The existing page number is meaningless under a different sort order. Also keep in mind that sort stability matters: if two items have the same value for the sorted column, their relative order might change between requests. Adding a secondary sort on a unique column (like `id`) ensures deterministic ordering, which is especially important for cursor-based pagination where the cursor depends on a stable sort.

### Can I combine filtering, sorting, and pagination in a single custom hook?

Yes, and doing so is a recommended pattern. Wrapping all three concerns into a single hook centralizes state management, ensures that filter and sort changes reset pagination correctly, and provides a clean API for components. The hook should accept configuration (default page size, available filters), manage internal state, sync with the URL, and expose the current data along with control functions.

The key challenge is keeping the URL, the hook's internal state, and the API request params in sync. A common approach is to use the URL as the single source of truth: the hook reads its initial state from `URLSearchParams`, and any state update writes back to the URL, which triggers a re-fetch via a `useEffect` dependency on the search params string.
