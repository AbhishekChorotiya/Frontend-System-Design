# API Retries, Timeouts, and Backoff Strategies

Network communication is inherently unreliable. Between the user's browser and the server, requests travel through DNS resolvers, load balancers, CDN edges, proxies, and cloud infrastructure -- any of which can introduce latency spikes, drop packets, or return transient errors. A 500 Internal Server Error does not always mean the server is broken; it might mean a single pod was restarting during a Kubernetes rollout. A `ECONNRESET` error does not mean the API is down; it might mean an intermediary proxy recycled a connection. Without retry and timeout mechanisms, frontend applications treat every failure as permanent, leaving users staring at error screens for problems that would resolve themselves in milliseconds.

Retry, timeout, and backoff strategies give frontend applications *resilience* -- the ability to absorb transient failures and recover gracefully without manual intervention. When implemented correctly, users never notice the blip. When implemented poorly, retries can amplify server load during outages (a *retry storm*), timeouts can be too aggressive or too lenient, and the application can feel sluggish or unreliable. Understanding how these mechanisms work together -- and when *not* to retry -- is a core frontend system design skill.

> **Think of it like calling a busy restaurant.** If you call and get a busy signal, you don't immediately give up and assume the restaurant has closed forever. You wait a moment and try again. If it's still busy, you wait a bit longer before the next attempt. But if you've called five times over ten minutes and it's still busy, you stop -- maybe they're closed for a private event. You also wouldn't keep calling every second without pause, because that just adds to the congestion. Retries with backoff work the same way: try again, wait progressively longer between attempts, and eventually give up if the problem persists.

## Core Concepts

1. **Transient vs Permanent Failures:** *Transient failures* are temporary problems that resolve on their own -- network timeouts, 503 Service Unavailable, DNS resolution hiccups, or rate limiting (429). *Permanent failures* indicate a real problem that retrying will not fix -- 400 Bad Request, 401 Unauthorized, 404 Not Found, or a malformed request body. The first step in any retry strategy is distinguishing between the two: retry transient failures, surface permanent failures immediately.

2. **Idempotency:** An operation is *idempotent* if performing it multiple times produces the same result as performing it once. GET requests are naturally idempotent. PUT and DELETE are typically idempotent. POST requests are often *not* idempotent -- retrying a payment POST could charge the user twice. Before retrying any request, you must consider whether it is safe to send the same request again.

3. **Retry Budget:** A *retry budget* is the maximum number of retry attempts (or the maximum total time) allowed for a single request. Without a budget, a failing request could retry indefinitely, consuming resources and degrading the user experience. Common budgets are 2-4 retries or a total elapsed time cap (e.g., 30 seconds).

4. **Timeout Types:** There are two primary timeout categories. A *connection timeout* limits how long the client waits to establish a TCP connection to the server. A *response timeout* (or read timeout) limits how long the client waits for the server to send back data after the connection is established. Both are essential: a missing connection timeout lets the request hang when the server is unreachable, while a missing response timeout lets it hang when the server accepts the connection but never responds.

5. **Exponential Backoff:** Instead of retrying immediately or at fixed intervals, *exponential backoff* increases the delay between retries geometrically -- typically doubling the wait time with each attempt (e.g., 1s, 2s, 4s, 8s). This gives the server time to recover and avoids overwhelming it with rapid-fire retries.

6. **Jitter:** When many clients retry with the same exponential backoff schedule, they all retry at the same times, creating *thundering herd* spikes. *Jitter* adds randomness to the delay so that retries are spread out over time. Full jitter randomizes the entire delay; equal jitter uses half the calculated delay plus a random portion of the other half.

7. **Circuit Breaker Pattern:** A *circuit breaker* monitors failure rates and temporarily stops sending requests to a failing service once a failure threshold is crossed. It has three states: *closed* (requests flow normally), *open* (requests are blocked), and *half-open* (a limited number of test requests are allowed through to check if the service has recovered). This prevents cascading failures and gives the server breathing room to recover.

## How It Works

The retry lifecycle follows a predictable flow from initial request through failure detection, retry decision, backoff delay, and eventual success or exhaustion:

1. **Send the initial request** with a configured timeout.
2. **Detect failure** -- the request either times out, throws a network error, or returns an HTTP status code indicating a server-side or transient problem.
3. **Evaluate retryability** -- check whether the error is transient (retryable) or permanent (non-retryable). Check whether the HTTP method is idempotent or safe to retry.
4. **Check the retry budget** -- if the maximum number of attempts has been reached, or the total elapsed time exceeds the cap, give up and surface the error to the user.
5. **Calculate backoff delay** -- determine how long to wait before the next attempt using the chosen strategy (fixed, linear, exponential, or exponential with jitter).
6. **Wait** for the calculated delay.
7. **Retry the request** -- go back to step 1.

### Decision Tree for Retryable Errors

Not all errors should be retried. Here is a practical classification:

**Retryable (transient) errors:**
* Network errors (`ECONNRESET`, `ECONNREFUSED`, `ETIMEDOUT`, `fetch` throwing `TypeError: Failed to fetch`)
* HTTP 408 Request Timeout
* HTTP 429 Too Many Requests (respect the `Retry-After` header)
* HTTP 500 Internal Server Error
* HTTP 502 Bad Gateway
* HTTP 503 Service Unavailable
* HTTP 504 Gateway Timeout

**Non-retryable (permanent) errors:**
* HTTP 400 Bad Request
* HTTP 401 Unauthorized
* HTTP 403 Forbidden
* HTTP 404 Not Found
* HTTP 405 Method Not Allowed
* HTTP 409 Conflict
* HTTP 422 Unprocessable Entity

```typescript
// src/utils/retryable.ts
const RETRYABLE_STATUS_CODES = new Set([408, 429, 500, 502, 503, 504]);

function isRetryableError(error: unknown): boolean {
  // Network errors (fetch throws TypeError for network failures)
  if (error instanceof TypeError) {
    return true;
  }

  // HTTP errors with retryable status codes
  if (error instanceof Response) {
    return RETRYABLE_STATUS_CODES.has(error.status);
  }

  // AbortError means the request was intentionally cancelled -- do not retry
  if (error instanceof DOMException && error.name === 'AbortError') {
    return false;
  }

  return false;
}

function isIdempotent(method: string): boolean {
  const idempotentMethods = new Set(['GET', 'HEAD', 'PUT', 'DELETE', 'OPTIONS']);
  return idempotentMethods.has(method.toUpperCase());
}
```

## Timeout Strategies

Timeouts prevent requests from hanging indefinitely. Without them, a single unresponsive endpoint can lock up the UI, exhaust connection pools, or cause memory leaks from accumulated pending promises.

### Connection vs Response Timeouts

The browser's `fetch` API does not natively distinguish between connection and response timeouts -- it provides a single mechanism via `AbortController`. However, the conceptual distinction still matters for choosing timeout values:

* **Connection timeout** (typically 5-10 seconds): How long to wait for the TCP handshake and TLS negotiation. If a server is completely unreachable, you want to fail fast.
* **Response timeout** (typically 10-30 seconds): How long to wait for the server to process the request and begin sending data. This should be longer because the server may need time for database queries, computation, or upstream calls.

### AbortController Timeout Implementation

```typescript
// src/utils/fetchWithTimeout.ts
async function fetchWithTimeout(
  url: string,
  options: RequestInit = {},
  timeoutMs: number = 10000
): Promise<Response> {
  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), timeoutMs);

  try {
    const response = await fetch(url, {
      ...options,
      signal: controller.signal,
    });
    return response;
  } catch (error) {
    if (error instanceof DOMException && error.name === 'AbortError') {
      throw new Error(`Request to ${url} timed out after ${timeoutMs}ms`);
    }
    throw error;
  } finally {
    clearTimeout(timeoutId);
  }
}
```

### Promise.race Timeout Pattern

An alternative approach uses `Promise.race` to race the fetch against a timeout promise:

```typescript
// src/utils/timeoutRace.ts
function withTimeout<T>(promise: Promise<T>, timeoutMs: number): Promise<T> {
  const timeout = new Promise<never>((_, reject) => {
    setTimeout(() => reject(new Error(`Operation timed out after ${timeoutMs}ms`)), timeoutMs);
  });
  return Promise.race([promise, timeout]);
}

// Usage
const response = await withTimeout(
  fetch('https://api.example.com/data'),
  5000
);
```

Note that `Promise.race` does not cancel the underlying fetch -- the request continues in the background. Use `AbortController` when you need to actually abort the request and free up the connection.

### Per-Request vs Global Timeouts

```typescript
// src/api/client.ts

// ❌ Bad: Single global timeout for all requests
const API_TIMEOUT = 30000;

// ✅ Good: Different timeouts based on operation type
const TIMEOUT_CONFIG = {
  fast: 5000,     // Health checks, simple lookups
  normal: 10000,  // Standard CRUD operations
  slow: 30000,    // File uploads, report generation
  stream: 60000,  // Streaming responses, large data exports
} as const;

type TimeoutTier = keyof typeof TIMEOUT_CONFIG;

async function apiRequest(
  url: string,
  options: RequestInit & { timeoutTier?: TimeoutTier } = {}
): Promise<Response> {
  const { timeoutTier = 'normal', ...fetchOptions } = options;
  const timeoutMs = TIMEOUT_CONFIG[timeoutTier];
  return fetchWithTimeout(url, fetchOptions, timeoutMs);
}
```

## Retry Strategies

Different retry strategies make different tradeoffs between simplicity, server load, and recovery speed. Here are the four most common approaches:

### Fixed Delay

Wait the same amount of time between every retry attempt. Simple but can cause synchronized retries across clients.

**Formula:** `delay = fixedDelay`

### Linear Backoff

Increase the delay by a fixed amount with each attempt. A moderate step up from fixed delay.

**Formula:** `delay = initialDelay + (attempt * increment)`

### Exponential Backoff

Double (or multiply by a base factor) the delay with each attempt. The most widely recommended strategy because it gives the server exponentially more time to recover.

**Formula:** `delay = initialDelay * (base ^ attempt)`

### Exponential Backoff with Jitter

Add randomness to exponential backoff to prevent thundering herd effects when many clients retry simultaneously.

**Full jitter formula:** `delay = random(0, initialDelay * (base ^ attempt))`

**Equal jitter formula:** `delay = (initialDelay * (base ^ attempt)) / 2 + random(0, (initialDelay * (base ^ attempt)) / 2)`

### Comparison Table

| Strategy | Formula | Attempt 1 | Attempt 2 | Attempt 3 | Attempt 4 | Thundering Herd Risk |
|----------|---------|-----------|-----------|-----------|-----------|---------------------|
| Fixed (1s) | `1s` | 1s | 1s | 1s | 1s | High |
| Linear (1s + 1s) | `1s + n*1s` | 2s | 3s | 4s | 5s | Medium |
| Exponential (1s, base 2) | `1s * 2^n` | 2s | 4s | 8s | 16s | Medium |
| Exp + Full Jitter | `rand(0, 1s * 2^n)` | 0-2s | 0-4s | 0-8s | 0-16s | Low |

### Basic Retry Wrapper with Exponential Backoff

```typescript
// src/utils/retry.ts
interface RetryOptions {
  maxRetries: number;
  initialDelayMs: number;
  maxDelayMs: number;
  backoffMultiplier: number;
}

const DEFAULT_OPTIONS: RetryOptions = {
  maxRetries: 3,
  initialDelayMs: 1000,
  maxDelayMs: 30000,
  backoffMultiplier: 2,
};

async function retryWithBackoff<T>(
  fn: () => Promise<T>,
  options: Partial<RetryOptions> = {}
): Promise<T> {
  const config = { ...DEFAULT_OPTIONS, ...options };
  let lastError: unknown;

  for (let attempt = 0; attempt <= config.maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error;

      if (attempt === config.maxRetries) {
        break;
      }

      const delay = Math.min(
        config.initialDelayMs * Math.pow(config.backoffMultiplier, attempt),
        config.maxDelayMs
      );

      await new Promise((resolve) => setTimeout(resolve, delay));
    }
  }

  throw lastError;
}

// Usage
const data = await retryWithBackoff(
  () => fetch('/api/users').then((r) => {
    if (!r.ok) throw r;
    return r.json();
  }),
  { maxRetries: 3, initialDelayMs: 500 }
);
```

### Configurable Retry Utility with Jitter

```typescript
// src/utils/retryWithJitter.ts
type JitterStrategy = 'none' | 'full' | 'equal';

interface AdvancedRetryOptions {
  maxRetries: number;
  initialDelayMs: number;
  maxDelayMs: number;
  backoffMultiplier: number;
  jitter: JitterStrategy;
  retryCondition?: (error: unknown) => boolean;
  onRetry?: (attempt: number, delay: number, error: unknown) => void;
}

const DEFAULT_ADVANCED_OPTIONS: AdvancedRetryOptions = {
  maxRetries: 3,
  initialDelayMs: 1000,
  maxDelayMs: 30000,
  backoffMultiplier: 2,
  jitter: 'full',
};

function calculateDelay(
  attempt: number,
  initialDelayMs: number,
  backoffMultiplier: number,
  maxDelayMs: number,
  jitter: JitterStrategy
): number {
  const exponentialDelay = initialDelayMs * Math.pow(backoffMultiplier, attempt);
  const cappedDelay = Math.min(exponentialDelay, maxDelayMs);

  switch (jitter) {
    case 'none':
      return cappedDelay;
    case 'full':
      return Math.random() * cappedDelay;
    case 'equal':
      return cappedDelay / 2 + Math.random() * (cappedDelay / 2);
    default:
      return cappedDelay;
  }
}

async function retryWithJitter<T>(
  fn: () => Promise<T>,
  options: Partial<AdvancedRetryOptions> = {}
): Promise<T> {
  const config = { ...DEFAULT_ADVANCED_OPTIONS, ...options };
  let lastError: unknown;

  for (let attempt = 0; attempt <= config.maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error;

      // Check if error is retryable
      if (config.retryCondition && !config.retryCondition(error)) {
        throw error;
      }

      if (attempt === config.maxRetries) {
        break;
      }

      const delay = calculateDelay(
        attempt,
        config.initialDelayMs,
        config.backoffMultiplier,
        config.maxDelayMs,
        config.jitter
      );

      config.onRetry?.(attempt + 1, delay, error);

      await new Promise((resolve) => setTimeout(resolve, delay));
    }
  }

  throw lastError;
}

// Usage with retry condition and logging
const data = await retryWithJitter(
  () => fetch('/api/orders').then((r) => {
    if (!r.ok) throw new HttpError(r.status, r.statusText);
    return r.json();
  }),
  {
    maxRetries: 4,
    initialDelayMs: 500,
    jitter: 'full',
    retryCondition: (error) => {
      if (error instanceof HttpError) {
        return RETRYABLE_STATUS_CODES.has(error.status);
      }
      return error instanceof TypeError; // Network error
    },
    onRetry: (attempt, delay, error) => {
      console.warn(`Retry attempt ${attempt} after ${Math.round(delay)}ms`, error);
    },
  }
);
```

## Circuit Breaker Pattern

The circuit breaker pattern prevents an application from repeatedly calling a service that is likely to fail. Instead of letting every request attempt (and fail) during an outage, the circuit breaker *trips* after a threshold of failures, immediately rejecting subsequent requests for a cooldown period. This protects both the client (no wasted time waiting for doomed requests) and the server (no additional load from hopeless retries).

### States

1. **Closed (normal operation):** Requests pass through normally. The circuit breaker monitors failure counts. If failures exceed the threshold within a time window, the circuit transitions to *open*.
2. **Open (tripped):** All requests are immediately rejected without being sent. After a configured cooldown period, the circuit transitions to *half-open*.
3. **Half-Open (testing recovery):** A limited number of test requests are allowed through. If they succeed, the circuit transitions back to *closed*. If they fail, it returns to *open*.

### Circuit Breaker Class Implementation

```typescript
// src/utils/circuitBreaker.ts
type CircuitState = 'CLOSED' | 'OPEN' | 'HALF_OPEN';

interface CircuitBreakerOptions {
  failureThreshold: number;
  resetTimeoutMs: number;
  halfOpenMaxAttempts: number;
  monitorWindowMs: number;
}

class CircuitBreaker {
  private state: CircuitState = 'CLOSED';
  private failureCount: number = 0;
  private lastFailureTime: number = 0;
  private halfOpenAttempts: number = 0;
  private successCount: number = 0;
  private readonly options: CircuitBreakerOptions;

  constructor(options: Partial<CircuitBreakerOptions> = {}) {
    this.options = {
      failureThreshold: 5,
      resetTimeoutMs: 30000,
      halfOpenMaxAttempts: 3,
      monitorWindowMs: 60000,
      ...options,
    };
  }

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === 'OPEN') {
      if (Date.now() - this.lastFailureTime >= this.options.resetTimeoutMs) {
        this.transitionTo('HALF_OPEN');
      } else {
        throw new CircuitBreakerError(
          'Circuit breaker is OPEN -- requests are blocked',
          this.options.resetTimeoutMs - (Date.now() - this.lastFailureTime)
        );
      }
    }

    if (this.state === 'HALF_OPEN' && this.halfOpenAttempts >= this.options.halfOpenMaxAttempts) {
      throw new CircuitBreakerError(
        'Circuit breaker is HALF_OPEN -- max test attempts reached',
        0
      );
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  private onSuccess(): void {
    if (this.state === 'HALF_OPEN') {
      this.successCount++;
      if (this.successCount >= this.options.halfOpenMaxAttempts) {
        this.transitionTo('CLOSED');
      }
    } else {
      this.failureCount = 0;
    }
  }

  private onFailure(): void {
    this.failureCount++;
    this.lastFailureTime = Date.now();

    if (this.state === 'HALF_OPEN') {
      this.transitionTo('OPEN');
    } else if (this.failureCount >= this.options.failureThreshold) {
      this.transitionTo('OPEN');
    }
  }

  private transitionTo(newState: CircuitState): void {
    const prevState = this.state;
    this.state = newState;

    if (newState === 'CLOSED') {
      this.failureCount = 0;
      this.successCount = 0;
      this.halfOpenAttempts = 0;
    } else if (newState === 'HALF_OPEN') {
      this.halfOpenAttempts = 0;
      this.successCount = 0;
    }

    console.info(`Circuit breaker: ${prevState} -> ${newState}`);
  }

  getState(): CircuitState {
    return this.state;
  }
}

class CircuitBreakerError extends Error {
  constructor(message: string, public readonly retryAfterMs: number) {
    super(message);
    this.name = 'CircuitBreakerError';
  }
}

// Usage
const apiCircuitBreaker = new CircuitBreaker({
  failureThreshold: 5,
  resetTimeoutMs: 30000,
});

async function fetchUsers(): Promise<User[]> {
  return apiCircuitBreaker.execute(async () => {
    const response = await fetch('/api/users');
    if (!response.ok) throw new Error(`HTTP ${response.status}`);
    return response.json();
  });
}
```

## Axios Interceptor for Automatic Retries

Axios interceptors provide a centralized way to add retry logic to every outgoing request without modifying individual API calls:

```typescript
// src/api/axiosRetry.ts
import axios, { AxiosError, AxiosInstance, InternalAxiosRequestConfig } from 'axios';

interface RetryConfig {
  maxRetries: number;
  initialDelayMs: number;
  maxDelayMs: number;
  retryableStatuses: Set<number>;
}

const DEFAULT_RETRY_CONFIG: RetryConfig = {
  maxRetries: 3,
  initialDelayMs: 1000,
  maxDelayMs: 10000,
  retryableStatuses: new Set([408, 429, 500, 502, 503, 504]),
};

interface ExtendedConfig extends InternalAxiosRequestConfig {
  _retryCount?: number;
  _retryConfig?: Partial<RetryConfig>;
}

function setupRetryInterceptor(
  instance: AxiosInstance,
  defaultConfig: Partial<RetryConfig> = {}
): void {
  const config = { ...DEFAULT_RETRY_CONFIG, ...defaultConfig };

  instance.interceptors.response.use(
    (response) => response,
    async (error: AxiosError) => {
      const axiosConfig = error.config as ExtendedConfig | undefined;

      if (!axiosConfig) {
        return Promise.reject(error);
      }

      const retryConfig = { ...config, ...axiosConfig._retryConfig };
      const retryCount = axiosConfig._retryCount ?? 0;

      // Check if we should retry
      const status = error.response?.status;
      const isRetryable = status
        ? retryConfig.retryableStatuses.has(status)
        : !error.response; // Network errors have no response

      if (!isRetryable || retryCount >= retryConfig.maxRetries) {
        return Promise.reject(error);
      }

      // Calculate delay with full jitter
      const exponentialDelay = retryConfig.initialDelayMs * Math.pow(2, retryCount);
      const cappedDelay = Math.min(exponentialDelay, retryConfig.maxDelayMs);
      const delay = Math.random() * cappedDelay;

      // Respect Retry-After header for 429 responses
      if (status === 429 && error.response?.headers['retry-after']) {
        const retryAfter = parseInt(error.response.headers['retry-after'], 10);
        if (!isNaN(retryAfter)) {
          await new Promise((resolve) => setTimeout(resolve, retryAfter * 1000));
        }
      } else {
        await new Promise((resolve) => setTimeout(resolve, delay));
      }

      // Increment retry count and retry
      axiosConfig._retryCount = retryCount + 1;
      console.warn(`Retrying request (${axiosConfig._retryCount}/${retryConfig.maxRetries}): ${axiosConfig.url}`);

      return instance.request(axiosConfig);
    }
  );
}

// Setup
const api = axios.create({ baseURL: '/api', timeout: 10000 });
setupRetryInterceptor(api, { maxRetries: 3 });

export default api;
```

## React Hook: useFetchWithRetry

A reusable React hook that combines fetching, retries, timeouts, and loading/error state management:

```tsx
// src/hooks/useFetchWithRetry.ts
import { useState, useEffect, useCallback, useRef } from 'react';

interface UseFetchWithRetryOptions {
  maxRetries?: number;
  initialDelayMs?: number;
  timeoutMs?: number;
  jitter?: boolean;
  enabled?: boolean;
}

interface UseFetchWithRetryResult<T> {
  data: T | null;
  error: Error | null;
  isLoading: boolean;
  isRetrying: boolean;
  retryCount: number;
  refetch: () => void;
}

function useFetchWithRetry<T>(
  url: string,
  options: UseFetchWithRetryOptions = {}
): UseFetchWithRetryResult<T> {
  const {
    maxRetries = 3,
    initialDelayMs = 1000,
    timeoutMs = 10000,
    jitter = true,
    enabled = true,
  } = options;

  const [data, setData] = useState<T | null>(null);
  const [error, setError] = useState<Error | null>(null);
  const [isLoading, setIsLoading] = useState(false);
  const [isRetrying, setIsRetrying] = useState(false);
  const [retryCount, setRetryCount] = useState(0);
  const [fetchKey, setFetchKey] = useState(0);
  const abortControllerRef = useRef<AbortController | null>(null);

  const refetch = useCallback(() => {
    setFetchKey((prev) => prev + 1);
  }, []);

  useEffect(() => {
    if (!enabled) return;

    let cancelled = false;
    const controller = new AbortController();
    abortControllerRef.current = controller;

    const executeFetch = async () => {
      setIsLoading(true);
      setError(null);
      setRetryCount(0);
      setIsRetrying(false);

      for (let attempt = 0; attempt <= maxRetries; attempt++) {
        if (cancelled) return;

        try {
          const timeoutId = setTimeout(() => controller.abort(), timeoutMs);

          const response = await fetch(url, { signal: controller.signal });
          clearTimeout(timeoutId);

          if (!response.ok) {
            const isRetryable = [408, 429, 500, 502, 503, 504].includes(response.status);
            if (!isRetryable || attempt === maxRetries) {
              throw new Error(`HTTP ${response.status}: ${response.statusText}`);
            }
            throw new Error(`Retryable HTTP ${response.status}`);
          }

          const result: T = await response.json();
          if (!cancelled) {
            setData(result);
            setError(null);
            setIsLoading(false);
            setIsRetrying(false);
          }
          return;
        } catch (err) {
          if (cancelled) return;

          if (err instanceof DOMException && err.name === 'AbortError') {
            setError(new Error(`Request timed out after ${timeoutMs}ms`));
            setIsLoading(false);
            return;
          }

          if (attempt === maxRetries) {
            setError(err instanceof Error ? err : new Error(String(err)));
            setIsLoading(false);
            setIsRetrying(false);
            return;
          }

          // Calculate backoff delay
          const exponentialDelay = initialDelayMs * Math.pow(2, attempt);
          const delay = jitter ? Math.random() * exponentialDelay : exponentialDelay;

          setRetryCount(attempt + 1);
          setIsRetrying(true);

          await new Promise((resolve) => setTimeout(resolve, delay));
        }
      }
    };

    executeFetch();

    return () => {
      cancelled = true;
      controller.abort();
    };
  }, [url, maxRetries, initialDelayMs, timeoutMs, jitter, enabled, fetchKey]);

  return { data, error, isLoading, isRetrying, retryCount, refetch };
}

// Usage in a component
function UserProfile({ userId }: { userId: string }) {
  const { data, error, isLoading, isRetrying, retryCount, refetch } = useFetchWithRetry<User>(
    `/api/users/${userId}`,
    { maxRetries: 3, timeoutMs: 8000 }
  );

  if (isLoading) {
    return (
      <div>
        {isRetrying
          ? <p>Request failed, retrying... (attempt {retryCount})</p>
          : <p>Loading user profile...</p>
        }
      </div>
    );
  }

  if (error) {
    return (
      <div>
        <p>Failed to load profile: {error.message}</p>
        <button onClick={refetch}>Try Again</button>
      </div>
    );
  }

  return <div><h2>{data?.name}</h2><p>{data?.email}</p></div>;
}
```

## Integration with React Query and SWR

Both React Query (TanStack Query) and SWR provide built-in retry mechanisms. Understanding their configuration allows you to leverage battle-tested retry logic instead of building your own:

```typescript
// src/api/reactQueryRetry.ts
import { QueryClient } from '@tanstack/react-query';

// ✅ Good: Configurable retry with retryable error detection
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      retry: (failureCount, error) => {
        // Don't retry on 4xx client errors (except 408 and 429)
        if (error instanceof Response) {
          if (error.status >= 400 && error.status < 500) {
            return error.status === 408 || error.status === 429;
          }
        }
        // Retry up to 3 times for other errors
        return failureCount < 3;
      },
      retryDelay: (attemptIndex) => {
        // Exponential backoff with jitter, capped at 30 seconds
        const delay = Math.min(1000 * Math.pow(2, attemptIndex), 30000);
        return delay + Math.random() * delay * 0.1; // Add 10% jitter
      },
      staleTime: 5 * 60 * 1000,
    },
    mutations: {
      retry: false, // Mutations should not auto-retry by default
    },
  },
});

// ❌ Bad: Blindly retrying everything including mutations
const badQueryClient = new QueryClient({
  defaultOptions: {
    queries: {
      retry: 5, // Retries all errors 5 times, even 401/403
    },
    mutations: {
      retry: 3, // Dangerous -- could duplicate non-idempotent operations
    },
  },
});
```

```typescript
// src/api/swrRetry.ts
import useSWR from 'swr';

// SWR retry configuration
function useUserData(userId: string) {
  return useSWR(
    `/api/users/${userId}`,
    async (url) => {
      const response = await fetch(url);
      if (!response.ok) throw new Error(`HTTP ${response.status}`);
      return response.json();
    },
    {
      errorRetryCount: 3,
      errorRetryInterval: 2000,
      // SWR uses exponential backoff by default
      onErrorRetry: (error, key, config, revalidate, { retryCount }) => {
        // Don't retry on 404 or 401
        if (error.message.includes('404') || error.message.includes('401')) {
          return;
        }

        // Retry after exponential backoff with jitter
        const delay = Math.min(1000 * Math.pow(2, retryCount), 30000);
        const jitteredDelay = delay + Math.random() * 1000;
        setTimeout(() => revalidate({ retryCount }), jitteredDelay);
      },
    }
  );
}
```

## Benefits

* **Improved Reliability:** Transient failures are absorbed transparently, and users see successful results without knowing a retry occurred.
* **Better User Experience:** Instead of showing error screens for temporary glitches, the application self-heals, resulting in fewer interruptions and higher user trust.
* **Reduced Support Burden:** Fewer user-visible errors means fewer support tickets about "the app is broken" when the real issue was a momentary network blip.
* **Server Protection:** Backoff and circuit breaker patterns prevent retry storms from overwhelming a struggling server, giving it time to recover.
* **Graceful Degradation:** Circuit breakers allow the application to fail fast and show fallback UI when a service is down, rather than hanging on hopeless requests.
* **Observability:** Retry counters and circuit breaker state transitions provide valuable signals for monitoring dashboards and alerting on elevated failure rates.

## Drawbacks and Challenges

* **Increased Latency on Failure:** Retries with backoff delay add time before the user sees a final error. A request that retries 3 times with exponential backoff could take 15+ seconds to exhaust all attempts.
* **Duplicate Side Effects:** Retrying non-idempotent operations (like payment charges or email sends) can cause real harm. Without careful idempotency checks, retries can produce duplicated data.
* **Retry Storms:** If many clients retry simultaneously using the same backoff schedule without jitter, they create synchronized spikes that can worsen an outage instead of helping.
* **Complexity:** Adding retry logic, circuit breakers, timeout tiers, and jitter increases code complexity. Each layer needs testing, monitoring, and configuration management.
* **Masking Real Failures:** Aggressive retries can hide persistent problems. If a service is fundamentally broken but returns intermittent successes, retries may mask the issue while degrading performance.
* **Resource Consumption:** Each retry attempt consumes browser connections, memory, and bandwidth. On mobile devices with limited resources, excessive retries can impact battery life and data usage.

## Use Cases

* **Payment Processing:** Payment APIs are the canonical example of needing *idempotency keys* combined with retries. If a payment request times out, the client must retry with the same idempotency key to avoid double-charging. The server uses the key to detect duplicate submissions and return the original result.
* **Form Submissions:** Long forms with user-entered data should retry on transient server errors rather than losing the user's input. Combine retries with local state persistence (localStorage or sessionStorage) to ensure data survives even browser crashes.
* **Data Fetching and Dashboards:** Dashboard panels fetching metrics, analytics, or list data benefit from automatic retries. Users expect dashboards to load successfully; a single failed request out of a dozen panels should not break the entire view.
* **File Uploads:** Large file uploads over unreliable connections should use resumable upload protocols (like the tus protocol) combined with retries for individual chunks. Retrying a full multi-megabyte upload from scratch is wasteful; retrying the failed chunk is efficient.
* **API Gateways and BFF Layers:** Backend-for-frontend layers that aggregate multiple microservice calls should implement circuit breakers per upstream service. If the recommendations service is down, the product page should still load with the product data -- the recommendations section can show a fallback.

## Best Practices

1. **Only retry idempotent or safe operations by default.** GET, HEAD, PUT, and DELETE are generally safe to retry. POST requests require idempotency keys or explicit opt-in before retrying.

2. **Use exponential backoff with jitter.** Never use fixed-interval retries in production. Full jitter provides the best distribution of retry attempts across clients and minimizes thundering herd effects.

3. **Set both per-request timeouts and a total retry time budget.** A single request should timeout after 10 seconds, and the total retry cycle should not exceed 30-60 seconds. Users will not wait longer than that.

4. **Respect the `Retry-After` header.** When a server returns 429 Too Many Requests with a `Retry-After` header, use that value instead of your own backoff calculation. The server is telling you exactly when it can handle your request again.

5. **Implement circuit breakers for critical dependencies.** If your app calls 5 microservices, a circuit breaker on each prevents a single failing service from degrading the entire application.

6. **Provide user feedback during retries.** Show a subtle indicator like "Reconnecting..." or "Retrying..." so users know the app is working on the problem rather than frozen.

7. **Log and monitor retry metrics.** Track retry rates, circuit breaker state changes, and timeout occurrences. A sudden spike in retries is an early warning signal for infrastructure problems.

8. **Cap retries with a maximum attempt count and maximum delay.** Without caps, exponential backoff can produce absurdly long delays (e.g., 2^10 seconds = 17 minutes). Cap the delay at 30-60 seconds and attempts at 3-5.

9. **Never retry on authentication or authorization failures.** A 401 or 403 will not resolve by retrying -- the user needs to re-authenticate or the permissions need to change.

10. **Test retry behavior explicitly.** Write unit tests that simulate network failures, verify retry counts, confirm backoff timing, and ensure non-retryable errors are surfaced immediately. Use mock timers to avoid slow tests.

## Common Beginner Doubts or Questions

### Should I retry all failed requests?

No. You should only retry requests that failed due to *transient* errors -- problems that are likely to resolve on their own within seconds. Network timeouts, DNS failures, HTTP 500/502/503/504 responses, and rate limiting (429) are all candidates for retries. However, client errors like 400 Bad Request, 401 Unauthorized, 403 Forbidden, and 404 Not Found indicate problems with the request itself, not the server's availability. Retrying these will produce the same error every time. Additionally, you should be cautious about retrying POST requests unless you have an idempotency mechanism in place, because sending the same POST twice could create duplicate records, duplicate charges, or duplicate emails.

### What is idempotency and why does it matter for retries?

An operation is *idempotent* if executing it multiple times produces the same outcome as executing it once. For example, setting a user's name to "Alice" is idempotent -- doing it three times still results in the name being "Alice." Creating a new order is *not* idempotent -- doing it three times creates three orders. Idempotency matters for retries because when a request times out, the client does not know whether the server processed it before the timeout. If the server did process it, retrying a non-idempotent request will cause duplication. The solution is *idempotency keys*: the client generates a unique key for each logical operation and sends it with the request. The server checks whether it has already processed a request with that key and returns the cached result instead of re-executing the operation. Payment APIs like Stripe require idempotency keys for exactly this reason.

### How do I choose timeout values?

Start by measuring your API's actual response times under normal load. Use the 95th or 99th percentile latency as a baseline, and set your timeout to 2-3x that value. For example, if your API's p99 latency is 2 seconds, a timeout of 5-6 seconds gives enough headroom for slow responses without hanging too long on failures. For different operations, use different timeouts: health checks and lookups should be fast (3-5 seconds), standard CRUD operations should be moderate (8-15 seconds), and file uploads or report generation should be generous (30-60 seconds). Avoid setting timeouts too low (you'll get false timeouts on legitimate slow responses) or too high (users will wait too long before seeing an error). Also consider the total time budget: if your timeout is 10 seconds and you retry 3 times, the user could wait up to 40+ seconds including backoff delays.

### What is the difference between exponential backoff and jitter?

Exponential backoff and jitter solve different problems and are best used together. *Exponential backoff* solves the problem of overwhelming a recovering server by spacing out retries with increasingly longer delays (1s, 2s, 4s, 8s). However, if 1,000 clients all start retrying at the same time with the same backoff schedule, they will all retry at second 1, then all at second 3, then all at second 7 -- creating synchronized spikes. *Jitter* solves this thundering herd problem by adding randomness to the delay. With full jitter, instead of all clients waiting exactly 4 seconds for their third retry, each client waits a random time between 0 and 4 seconds. This spreads the retry load evenly over time, giving the server a smooth stream of requests instead of periodic bursts.

### When should I use a circuit breaker on the frontend?

Use a circuit breaker when your frontend depends on a backend service that can experience sustained outages (not just individual request failures). The circuit breaker is most valuable when: (1) the failing service is one of several your app depends on, so you can show partial content while the broken service recovers; (2) the failure is causing noticeable latency because each request waits for a timeout before failing; (3) you have a meaningful fallback to show, like cached data, a placeholder, or a "service temporarily unavailable" message. On the other hand, a circuit breaker adds complexity and is unnecessary if your app only talks to one API (the entire app is broken anyway), if failures are rare and brief, or if you are already using a library like React Query that handles retries and caching for you. In practice, circuit breakers are more common in BFF (backend-for-frontend) layers than in browser code, but they are a valid pattern for frontend apps that aggregate data from multiple independent services.

### How do libraries like React Query handle retries?

React Query (TanStack Query) has built-in retry support with sensible defaults. By default, failed queries retry 3 times with exponential backoff. You can customize this globally via `QueryClient` options or per-query. The `retry` option accepts a number (max attempts) or a function `(failureCount, error) => boolean` for conditional retries. The `retryDelay` option accepts a number or a function `(attemptIndex) => number` for custom backoff logic. Importantly, React Query only auto-retries *queries* (GET-like operations), not mutations, because mutations are often non-idempotent. For mutations, retry is disabled by default, and you should enable it explicitly only when you have confirmed the operation is idempotent. SWR has similar built-in retry with the `errorRetryCount` and `onErrorRetry` options, using exponential backoff by default.
