# Web Vitals

## Table of Contents
- [Introduction](#introduction)
- [Core Web Vitals](#core-web-vitals)
- [Largest Contentful Paint (LCP)](#largest-contentful-paint-lcp)
- [Interaction to Next Paint (INP)](#interaction-to-next-paint-inp)
- [Cumulative Layout Shift (CLS)](#cumulative-layout-shift-cls)
- [Other Important Web Vitals](#other-important-web-vitals)
- [Measuring Web Vitals](#measuring-web-vitals)
- [Optimization Strategies](#optimization-strategies)
- [Tools and Libraries](#tools-and-libraries)
- [Real-World Implementation](#real-world-implementation)
- [Common Beginner Doubts](#common-beginner-doubts)
- [Best Practices](#best-practices)

## Introduction

Web Vitals are a set of metrics introduced by Google to measure real-world user experience on web pages. These metrics focus on three key aspects of user experience: loading performance, interactivity, and visual stability. Web Vitals provide quantifiable measurements that help developers understand and optimize the user experience of their websites.

The initiative aims to simplify the landscape of performance metrics by providing unified guidance for quality signals that are essential to delivering a great user experience on the web.

> **Think of it like a restaurant review.** Web Vitals are the equivalent of rating a restaurant on three things: how quickly food arrives at your table (LCP — loading), how fast the waiter responds when you call them (INP — interactivity), and whether the table settings keep shifting around while you eat (CLS — visual stability). Just as a restaurant needs all three to get a great review, your website needs to score well on all three Core Web Vitals for a good user experience.

## Core Web Vitals

Core Web Vitals are a subset of Web Vitals that apply to all web pages and are considered the most important metrics for user experience. As of 2024, there are three Core Web Vitals:

### 1. Largest Contentful Paint (LCP) - Loading Performance
### 2. Interaction to Next Paint (INP) - Interactivity  
### 3. Cumulative Layout Shift (CLS) - Visual Stability

These metrics are also used as ranking factors in Google's search algorithm, making them crucial for both user experience and SEO.

## Largest Contentful Paint (LCP)

### What is LCP?

Largest Contentful Paint measures the loading performance of a page by tracking when the largest content element in the viewport becomes visible to the user. This metric provides insight into when the main content of the page has loaded.

### LCP Thresholds

- **Good**: 2.5 seconds or less
- **Needs Improvement**: 2.5 to 4.0 seconds
- **Poor**: More than 4.0 seconds

### Elements Considered for LCP

- `<img>` elements
- `<image>` elements inside `<svg>`
- `<video>` elements with poster images
- Elements with background images loaded via CSS
- Block-level text elements

### Code Example: Measuring LCP

```javascript
// Using the Web Vitals library
import { getLCP } from 'web-vitals';

getLCP((metric) => {
  console.log('LCP:', metric);
  // Send to analytics
  gtag('event', 'web_vitals', {
    event_category: 'Web Vitals',
    event_action: 'LCP',
    value: Math.round(metric.value),
    custom_parameter_1: metric.id,
  });
});

// Manual implementation using PerformanceObserver
function measureLCP() {
  const observer = new PerformanceObserver((list) => {
    const entries = list.getEntries();
    const lastEntry = entries[entries.length - 1];
    
    console.log('LCP candidate:', lastEntry.startTime);
    
    // Send to your analytics service
    sendToAnalytics('LCP', lastEntry.startTime);
  });
  
  observer.observe({ type: 'largest-contentful-paint', buffered: true });
}

// Call the function when the page loads
if (typeof PerformanceObserver !== 'undefined') {
  measureLCP();
}
```

### LCP Optimization Strategies

```html
<!-- Optimize images with proper sizing and formats -->
<img 
  src="hero-image.webp" 
  alt="Hero image"
  width="800" 
  height="600"
  loading="eager"
  fetchpriority="high"
>

<!-- Preload critical resources -->
<link rel="preload" href="hero-image.webp" as="image">
<link rel="preload" href="critical-font.woff2" as="font" type="font/woff2" crossorigin>

<!-- Use responsive images -->
<picture>
  <source media="(min-width: 800px)" srcset="hero-large.webp">
  <source media="(min-width: 400px)" srcset="hero-medium.webp">
  <img src="hero-small.webp" alt="Hero image">
</picture>
```

## Interaction to Next Paint (INP)

### What is INP?

Interaction to Next Paint measures the overall responsiveness of a page to user interactions. It observes the latency of **all** click, tap, and keyboard interactions throughout the entire lifespan of a page and reports the worst interaction (ignoring outliers). A low INP means the page is consistently responsive to user input.

> **Note:** INP replaced First Input Delay (FID) as a Core Web Vital in March 2024. While FID only measured the *first* interaction's input delay, INP captures the *full duration* of every interaction (from input to the next paint), giving a much more complete picture of a page's responsiveness.

### INP Thresholds

- **Good**: 200 milliseconds or less
- **Needs Improvement**: 200 to 500 milliseconds
- **Poor**: More than 500 milliseconds

### Code Example: Measuring INP

```javascript
// Using the web-vitals library (v3+)
import { onINP } from 'web-vitals';

onINP((metric) => {
  console.log('INP:', metric);

  // Send to analytics
  gtag('event', 'web_vitals', {
    event_category: 'Web Vitals',
    event_action: 'INP',
    value: Math.round(metric.value),
    custom_parameter_1: metric.id,
  });
});

// Manual INP measurement using PerformanceObserver
function measureINP() {
  let worstLatency = 0;

  const observer = new PerformanceObserver((list) => {
    for (const entry of list.getEntries()) {
      // INP considers the full duration: input delay + processing + presentation delay
      const duration = entry.duration;
      if (duration > worstLatency) {
        worstLatency = duration;
        console.log('New worst interaction:', {
          duration,
          type: entry.name,
          target: entry.target,
        });
      }
    }
  });

  observer.observe({ type: 'event', buffered: true, durationThreshold: 16 });
}
```

### INP Optimization Techniques

```javascript
// 1. Code splitting to reduce main thread blocking
const LazyComponent = React.lazy(() => import('./LazyComponent'));

// 2. Use requestIdleCallback for non-critical tasks
function performNonCriticalTask() {
  if ('requestIdleCallback' in window) {
    requestIdleCallback(() => {
      // Perform heavy computation during idle time
      processLargeDataSet();
    });
  } else {
    // Fallback for browsers without requestIdleCallback
    setTimeout(processLargeDataSet, 0);
  }
}

// 3. Break up long tasks using scheduler.yield() (modern) or setTimeout
async function processLargeArray(array) {
  const chunkSize = 100;
  
  for (let i = 0; i < array.length; i += chunkSize) {
    const chunk = array.slice(i, i + chunkSize);
    chunk.forEach(item => processItem(item));
    
    // Yield to the main thread between chunks so the browser
    // can respond to user input
    if (typeof scheduler !== 'undefined' && scheduler.yield) {
      await scheduler.yield();
    } else {
      await new Promise(resolve => setTimeout(resolve, 0));
    }
  }
}

// 4. Optimize event handlers — avoid layout thrashing
document.addEventListener('click', (event) => {
  // Read layout properties first (batched reads)
  const scrollTop = document.documentElement.scrollTop;
  const rect = event.target.getBoundingClientRect();
  
  // Then perform writes
  requestAnimationFrame(() => {
    updateUI(scrollTop, rect);
  });
});
```

## Cumulative Layout Shift (CLS)

### What is CLS?

Cumulative Layout Shift measures the visual stability of a page by quantifying how much visible content shifts during the page's lifetime. It helps identify unexpected layout shifts that can frustrate users.

### CLS Thresholds

- **Good**: 0.1 or less
- **Needs Improvement**: 0.1 to 0.25
- **Poor**: More than 0.25

### Code Example: Measuring CLS

```javascript
import { getCLS } from 'web-vitals';

getCLS((metric) => {
  console.log('CLS:', metric);
  
  // Log layout shift details
  metric.entries.forEach((entry) => {
    console.log('Layout shift:', {
      value: entry.value,
      sources: entry.sources,
      hadRecentInput: entry.hadRecentInput
    });
  });
  
  // Send to analytics
  gtag('event', 'web_vitals', {
    event_category: 'Web Vitals',
    event_action: 'CLS',
    value: Math.round(metric.value * 1000),
    custom_parameter_1: metric.id,
  });
});

// Manual CLS measurement
function measureCLS() {
  let clsValue = 0;
  let sessionValue = 0;
  let sessionEntries = [];
  
  const observer = new PerformanceObserver((list) => {
    for (const entry of list.getEntries()) {
      // Only count layout shifts without recent user input
      if (!entry.hadRecentInput) {
        const firstSessionEntry = sessionEntries[0];
        const lastSessionEntry = sessionEntries[sessionEntries.length - 1];
        
        // If the entry occurred less than 1 second after the previous entry
        // and less than 5 seconds after the first entry in the session,
        // include it in the current session. Otherwise, start a new session.
        if (sessionValue &&
            entry.startTime - lastSessionEntry.startTime < 1000 &&
            entry.startTime - firstSessionEntry.startTime < 5000) {
          sessionValue += entry.value;
          sessionEntries.push(entry);
        } else {
          sessionValue = entry.value;
          sessionEntries = [entry];
        }
        
        // If the current session value is larger than the current CLS value,
        // update CLS and the entries contributing to it.
        if (sessionValue > clsValue) {
          clsValue = sessionValue;
        }
      }
    }
    
    console.log('Current CLS:', clsValue);
  });
  
  observer.observe({ type: 'layout-shift', buffered: true });
}
```

### CLS Optimization Strategies

```css
/* 1. Always include size attributes for images and videos */
img, video {
  width: 100%;
  height: auto;
  aspect-ratio: 16 / 9; /* Modern approach */
}

/* 2. Reserve space for ads and embeds */
.ad-container {
  min-height: 250px; /* Reserve space for ad */
  background-color: #f0f0f0;
  display: flex;
  align-items: center;
  justify-content: center;
}

/* 3. Use CSS transforms for animations instead of changing layout properties */
.animated-element {
  transform: translateX(0);
  transition: transform 0.3s ease;
}

.animated-element.moved {
  transform: translateX(100px); /* Use transform instead of changing left/right */
}

/* 4. Avoid inserting content above existing content */
.dynamic-content {
  position: absolute; /* Or use fixed positioning */
  top: 0;
  left: 0;
}
```

```html
<!-- Reserve space for images -->
<div style="aspect-ratio: 16/9; background-color: #f0f0f0;">
  <img src="image.jpg" alt="Description" style="width: 100%; height: 100%; object-fit: cover;">
</div>

<!-- Use skeleton screens for loading content -->
<div class="skeleton-loader" style="height: 200px; background: linear-gradient(90deg, #f0f0f0 25%, #e0e0e0 50%, #f0f0f0 75%); background-size: 200% 100%; animation: loading 1.5s infinite;">
</div>
```

## Other Important Web Vitals

### First Contentful Paint (FCP)

Measures when the first text or image is painted on the screen.

```javascript
import { getFCP } from 'web-vitals';

getFCP((metric) => {
  console.log('FCP:', metric.value);
});
```

### Time to Interactive (TTI)

Measures when the page becomes fully interactive. TTI is a **lab-only metric** — it is measured via tools like Lighthouse and is **not** available through the `web-vitals` npm library or field data collection.

```javascript
// TTI is NOT available in the web-vitals library.
// Measure TTI using Lighthouse or the Chrome DevTools Performance panel.
//
// Programmatically via Lighthouse Node module:
// const lighthouse = require('lighthouse');
// const result = await lighthouse(url, options);
// console.log('TTI:', result.lhr.audits['interactive'].numericValue);
```

### Total Blocking Time (TBT)

Measures the total amount of time between FCP and TTI where the main thread was blocked for long enough to prevent input responsiveness. TBT is also a **lab-only metric** and is not available through the `web-vitals` library.

```javascript
// TBT is NOT available in the web-vitals library.
// Measure TBT using Lighthouse or WebPageTest.
//
// You can approximate long-task blocking in the browser using PerformanceObserver:
function measureLongTasks() {
  const observer = new PerformanceObserver((list) => {
    let totalBlockingTime = 0;
    for (const entry of list.getEntries()) {
      // Any task longer than 50ms contributes to blocking time
      if (entry.duration > 50) {
        totalBlockingTime += entry.duration - 50;
      }
    }
    console.log('Approximate blocking time:', totalBlockingTime, 'ms');
  });

  observer.observe({ type: 'longtask', buffered: true });
}
```

## Measuring Web Vitals

### Using the Web Vitals Library

```javascript
// Install: npm install web-vitals

import { onCLS, onINP, onFCP, onLCP, onTTFB } from 'web-vitals';

function sendToAnalytics(metric) {
  // Send to your analytics service
  gtag('event', 'web_vitals', {
    event_category: 'Web Vitals',
    event_action: metric.name,
    value: Math.round(metric.value),
    custom_parameter_1: metric.id,
    non_interaction: true,
  });
}

// Measure all Core Web Vitals (web-vitals v4+)
onCLS(sendToAnalytics);
onINP(sendToAnalytics);
onLCP(sendToAnalytics);

// Measure other Web Vitals
onFCP(sendToAnalytics);
onTTFB(sendToAnalytics);
```

### Custom Analytics Implementation

```javascript
class WebVitalsTracker {
  constructor(analyticsEndpoint) {
    this.analyticsEndpoint = analyticsEndpoint;
    this.metrics = {};
    this.initializeTracking();
  }
  
  initializeTracking() {
    // Track Core Web Vitals
    this.trackLCP();
    this.trackINP();
    this.trackCLS();
    
    // Send metrics when page is about to unload
    window.addEventListener('beforeunload', () => {
      this.sendMetrics();
    });
    
    // Send metrics when page becomes hidden
    document.addEventListener('visibilitychange', () => {
      if (document.visibilityState === 'hidden') {
        this.sendMetrics();
      }
    });
  }
  
  trackLCP() {
    const observer = new PerformanceObserver((list) => {
      const entries = list.getEntries();
      const lastEntry = entries[entries.length - 1];
      this.metrics.lcp = lastEntry.startTime;
    });
    
    observer.observe({ type: 'largest-contentful-paint', buffered: true });
  }
  
  trackINP() {
    const observer = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        const duration = entry.duration;
        if (!this.metrics.inp || duration > this.metrics.inp) {
          this.metrics.inp = duration;
        }
      }
    });
    
    observer.observe({ type: 'event', buffered: true, durationThreshold: 16 });
  }
  
  trackCLS() {
    let clsValue = 0;
    
    const observer = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        if (!entry.hadRecentInput) {
          clsValue += entry.value;
        }
      }
      this.metrics.cls = clsValue;
    });
    
    observer.observe({ type: 'layout-shift', buffered: true });
  }
  
  sendMetrics() {
    if (Object.keys(this.metrics).length > 0) {
      fetch(this.analyticsEndpoint, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({
          url: window.location.href,
          timestamp: Date.now(),
          metrics: this.metrics,
          userAgent: navigator.userAgent,
        }),
        keepalive: true, // Ensure request completes even if page unloads
      });
    }
  }
}

// Initialize tracking
const tracker = new WebVitalsTracker('/api/web-vitals');
```

## Optimization Strategies

### General Performance Optimization

```javascript
// 1. Resource prioritization
const criticalResources = [
  { href: '/critical.css', as: 'style' },
  { href: '/hero-image.webp', as: 'image' },
  { href: '/critical-font.woff2', as: 'font', type: 'font/woff2', crossorigin: true }
];

criticalResources.forEach(resource => {
  const link = document.createElement('link');
  link.rel = 'preload';
  Object.assign(link, resource);
  document.head.appendChild(link);
});

// 2. Lazy loading implementation
const imageObserver = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      const img = entry.target;
      img.src = img.dataset.src;
      img.classList.remove('lazy');
      imageObserver.unobserve(img);
    }
  });
});

document.querySelectorAll('img[data-src]').forEach(img => {
  imageObserver.observe(img);
});

// 3. Service Worker for caching
if ('serviceWorker' in navigator) {
  navigator.serviceWorker.register('/sw.js')
    .then(registration => {
      console.log('SW registered:', registration);
    })
    .catch(error => {
      console.log('SW registration failed:', error);
    });
}
```

### React-Specific Optimizations

```jsx
import React, { Suspense, lazy, memo, useMemo, useCallback } from 'react';

// 1. Code splitting with React.lazy
const LazyComponent = lazy(() => import('./LazyComponent'));

// 2. Memoization to prevent unnecessary re-renders
const OptimizedComponent = memo(({ data, onUpdate }) => {
  const processedData = useMemo(() => {
    return data.map(item => ({
      ...item,
      processed: true
    }));
  }, [data]);
  
  const handleUpdate = useCallback((id) => {
    onUpdate(id);
  }, [onUpdate]);
  
  return (
    <div>
      {processedData.map(item => (
        <div key={item.id} onClick={() => handleUpdate(item.id)}>
          {item.name}
        </div>
      ))}
    </div>
  );
});

// 3. Suspense for loading states
function App() {
  return (
    <div>
      <Suspense fallback={<div>Loading...</div>}>
        <LazyComponent />
      </Suspense>
    </div>
  );
}

// 4. Image optimization component
const OptimizedImage = ({ src, alt, width, height, ...props }) => {
  return (
    <div style={{ aspectRatio: `${width}/${height}` }}>
      <img
        src={src}
        alt={alt}
        width={width}
        height={height}
        loading="lazy"
        style={{ width: '100%', height: '100%', objectFit: 'cover' }}
        {...props}
      />
    </div>
  );
};
```

## Tools and Libraries

### Browser DevTools

```javascript
// Performance tab analysis
console.log('Performance analysis:');
console.log('- Check for long tasks (>50ms)');
console.log('- Identify layout thrashing');
console.log('- Monitor network waterfall');

// Lighthouse programmatic API
const lighthouse = require('lighthouse');
const chromeLauncher = require('chrome-launcher');

async function runLighthouse(url) {
  const chrome = await chromeLauncher.launch({ chromeFlags: ['--headless'] });
  const options = { logLevel: 'info', output: 'html', port: chrome.port };
  const runnerResult = await lighthouse(url, options);
  
  console.log('Report is done for', runnerResult.lhr.finalUrl);
  console.log('Performance score was', runnerResult.lhr.categories.performance.score * 100);
  
  await chrome.kill();
}
```

### Third-Party Tools

```javascript
// Google PageSpeed Insights API
async function getPageSpeedInsights(url) {
  const apiKey = 'YOUR_API_KEY';
  const apiUrl = `https://www.googleapis.com/pagespeedonline/v5/runPagespeed?url=${encodeURIComponent(url)}&key=${apiKey}&strategy=mobile`;
  
  try {
    const response = await fetch(apiUrl);
    const data = await response.json();
    
    const metrics = data.lighthouseResult.audits;
    console.log('Core Web Vitals:', {
      LCP: metrics['largest-contentful-paint'].displayValue,
      FID: metrics['max-potential-fid'].displayValue,
      CLS: metrics['cumulative-layout-shift'].displayValue
    });
  } catch (error) {
    console.error('Error fetching PageSpeed Insights:', error);
  }
}

// Real User Monitoring (RUM)
class RUMTracker {
  constructor() {
    this.startTime = performance.now();
    this.metrics = {};
    this.trackPageLoad();
    this.trackUserInteractions();
  }
  
  trackPageLoad() {
    window.addEventListener('load', () => {
      const loadTime = performance.now() - this.startTime;
      this.metrics.pageLoadTime = loadTime;
      this.sendMetrics();
    });
  }
  
  trackUserInteractions() {
    ['click', 'keydown', 'scroll'].forEach(eventType => {
      document.addEventListener(eventType, (event) => {
        const interactionTime = performance.now();
        this.metrics.lastInteraction = {
          type: eventType,
          timestamp: interactionTime,
          target: event.target.tagName
        };
      });
    });
  }
  
  sendMetrics() {
    // Send to your analytics service
    fetch('/api/rum-metrics', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(this.metrics)
    });
  }
}
```

## Real-World Implementation

### E-commerce Site Optimization

```javascript
// Product listing page optimization
class ProductListOptimizer {
  constructor() {
    this.initializeOptimizations();
  }
  
  initializeOptimizations() {
    this.optimizeImages();
    this.implementVirtualScrolling();
    this.preloadCriticalResources();
    this.trackWebVitals();
  }
  
  optimizeImages() {
    // Implement progressive image loading
    const images = document.querySelectorAll('.product-image');
    
    const imageObserver = new IntersectionObserver((entries) => {
      entries.forEach(entry => {
        if (entry.isIntersecting) {
          const img = entry.target;
          const highResUrl = img.dataset.highres;
          
          // Load high-res image
          const highResImg = new Image();
          highResImg.onload = () => {
            img.src = highResUrl;
            img.classList.add('loaded');
          };
          highResImg.src = highResUrl;
          
          imageObserver.unobserve(img);
        }
      });
    }, { rootMargin: '50px' });
    
    images.forEach(img => imageObserver.observe(img));
  }
  
  implementVirtualScrolling() {
    // Virtual scrolling for large product lists
    const container = document.querySelector('.product-list');
    const itemHeight = 200; // Height of each product item
    const visibleItems = Math.ceil(window.innerHeight / itemHeight) + 2;
    
    let startIndex = 0;
    
    const updateVisibleItems = () => {
      const scrollTop = container.scrollTop;
      const newStartIndex = Math.floor(scrollTop / itemHeight);
      
      if (newStartIndex !== startIndex) {
        startIndex = newStartIndex;
        this.renderVisibleItems(startIndex, visibleItems);
      }
    };
    
    container.addEventListener('scroll', updateVisibleItems);
  }
  
  preloadCriticalResources() {
    // Preload above-the-fold product images
    const aboveFoldProducts = document.querySelectorAll('.product-item:nth-child(-n+6)');
    
    aboveFoldProducts.forEach(product => {
      const img = product.querySelector('img');
      if (img && img.dataset.src) {
        const link = document.createElement('link');
        link.rel = 'preload';
        link.as = 'image';
        link.href = img.dataset.src;
        document.head.appendChild(link);
      }
    });
  }
  
  trackWebVitals() {
    import('web-vitals').then(({ getCLS, getFID, getLCP }) => {
      getCLS((metric) => this.sendMetric('CLS', metric));
      getFID((metric) => this.sendMetric('FID', metric));
      getLCP((metric) => this.sendMetric('LCP', metric));
    });
  }
  
  sendMetric(name, metric) {
    // Send to analytics with product page context
    gtag('event', 'web_vitals', {
      event_category: 'Web Vitals',
      event_action: name,
      value: Math.round(metric.value),
      custom_parameter_1: 'product-listing',
      custom_parameter_2: window.location.pathname,
    });
  }
}

// Initialize on product pages
if (document.querySelector('.product-list')) {
  new ProductListOptimizer();
}
```

### News Website Optimization

```javascript
// Article page optimization
class ArticlePageOptimizer {
  constructor() {
    this.optimizeArticleLoading();
    this.implementProgressiveEnhancement();
    this.trackReadingMetrics();
  }
  
  optimizeArticleLoading() {
    // Prioritize above-the-fold content
    const articleContent = document.querySelector('.article-content');
    const images = articleContent.querySelectorAll('img');
    
    // Mark first image as high priority
    if (images.length > 0) {
      images[0].loading = 'eager';
      images[0].fetchPriority = 'high';
    }
    
    // Lazy load remaining images
    images.forEach((img, index) => {
      if (index > 0) {
        img.loading = 'lazy';
      }
    });
  }
  
  implementProgressiveEnhancement() {
    // Load non-critical features progressively
    const loadNonCriticalFeatures = () => {
      // Load comments section
      import('./comments.js').then(module => {
        module.initializeComments();
      });
      
      // Load social sharing
      import('./social-sharing.js').then(module => {
        module.initializeSocialSharing();
      });
      
      // Load related articles
      import('./related-articles.js').then(module => {
        module.loadRelatedArticles();
      });
    };
    
    // Load after main content is ready
    if (document.readyState === 'complete') {
      loadNonCriticalFeatures();
    } else {
      window.addEventListener('load', loadNonCriticalFeatures);
    }
  }
  
  trackReadingMetrics() {
    let startTime = Date.now();
    let maxScroll = 0;
    
    const trackScroll = () => {
      const scrollPercent = (window.scrollY / (document.body.scrollHeight - window.innerHeight)) * 100;
      maxScroll = Math.max(maxScroll, scrollPercent);
    };
    
    window.addEventListener('scroll', trackScroll);
    
    // Track reading completion
    window.addEventListener('beforeunload', () => {
      const readingTime = Date.now() - startTime;
      
      gtag('event', 'article_engagement', {
        event_category: 'Reading',
        event_action: 'Article Completion',
        value: Math.round(maxScroll),
        custom_parameter_1: Math.round(readingTime / 1000), // seconds
      });
    });
  }
}
```

## Common Beginner Doubts

### Q1: Why are my Web Vitals scores different between tools?

**Answer:** Different tools measure Web Vitals at different times and under different conditions:

- **Lab data** (Lighthouse, PageSpeed Insights): Simulated environment with controlled conditions
- **Field data** (Chrome User Experience Report): Real user data from actual visitors
- **Real User Monitoring**: Your own users' actual experience

```javascript
// Understanding the difference
console.log('Lab vs Field Data:');
console.log('Lab Data: Controlled environment, consistent results, good for debugging');
console.log('Field Data: Real user conditions, varies by device/network, reflects actual UX');

// Use both for comprehensive analysis
const analyzePerformance = () => {
  // Lab data for development
  if (window.location.hostname === 'localhost') {
    console.log('Development environment - focus on lab data');
  }
  
  // Field data for production monitoring
  if (window.location.hostname === 'production-site.com') {
    // Implement RUM tracking
    trackRealUserMetrics();
  }
};
```

### Q2: How do I know which Web Vital to prioritize?

**Answer:** Prioritize based on your site's specific issues and user journey:

```javascript
// Prioritization strategy
const prioritizeOptimizations = (metrics) => {
  const priorities = [];
  
  if (metrics.lcp > 2500) {
    priorities.push({
      metric: 'LCP',
      impact: 'high',
      actions: ['Optimize images', 'Reduce server response time', 'Eliminate render-blocking resources']
    });
  }
  
  if (metrics.fid > 100) {
    priorities.push({
      metric: 'FID',
      impact: 'high',
      actions: ['Reduce JavaScript execution time', 'Code splitting', 'Remove unused JavaScript']
    });
  }
  
  if (metrics.cls > 0.1) {
    priorities.push({
      metric: 'CLS',
      impact: 'medium',
      actions: ['Add size attributes to images', 'Reserve space for ads', 'Avoid inserting content above existing content']
    });
  }
  
  return priorities.sort((a, b) => {
    const impactOrder = { high: 3, medium: 2, low: 1 };
    return impactOrder[b.impact] - impactOrder[a.impact];
  });
};
```

### Q3: Do Web Vitals affect SEO rankings?

**Answer:** Yes, Core Web Vitals are part of Google's Page Experience signals and can affect search rankings, but they're not the only factor:

```javascript
// SEO impact considerations
const seoFactors = {
  coreWebVitals: {
    weight: 'moderate',
    impact: 'Tie-breaker between similar content quality',
    note: 'Good content still ranks higher than fast but poor content'
  },
  contentQuality: {
    weight: 'high',
    impact: 'Primary ranking factor',
    note: 'Relevance and quality matter most'
  },
  mobileUsability: {
    weight: 'high',
    impact: 'Mobile-first indexing',
    note: 'Essential for mobile search rankings'
  }
};

console.log('SEO Strategy: Focus on content quality first, then optimize Web Vitals');
```

### Q4: Should I optimize for lab data or field data?

**Answer:** Both are important, but prioritize field data because it reflects what real users actually experience:

```javascript
// Understanding lab vs field data
const dataComparison = {
  labData: {
    source: 'Lighthouse, WebPageTest, Chrome DevTools',
    pros: ['Reproducible', 'Controlled environment', 'Good for debugging'],
    cons: ['Does not reflect real user conditions', 'Single device/network profile'],
    bestFor: 'Diagnosing issues and testing fixes before deployment'
  },
  fieldData: {
    source: 'Chrome User Experience Report (CrUX), web-vitals library, RUM tools',
    pros: ['Real user conditions', 'Diverse devices and networks', 'Reflects actual experience'],
    cons: ['Harder to reproduce issues', 'Requires traffic volume for data'],
    bestFor: 'Understanding real user experience and tracking improvements over time'
  }
};

// Strategy: Use lab data to debug, field data to prioritize
console.log('Start with field data to find problems, use lab data to fix them');
```

## Best Practices

### 1. Monitor Continuously
- Set up Real User Monitoring (RUM) to track Core Web Vitals from real users
- Use the `web-vitals` library to collect field data and send it to your analytics
- Set up alerts for when metrics cross "Needs Improvement" thresholds

### 2. Optimize for the 75th Percentile
- Google evaluates Core Web Vitals at the **75th percentile** — meaning 75% of page visits must meet the "Good" threshold
- Do not optimize for averages; focus on the long tail of slower experiences

### 3. Test on Real Devices
- Lab testing on a high-end laptop does not represent your users
- Use Chrome DevTools device emulation with CPU throttling (4x slowdown) and network throttling (Slow 3G)
- Test on physical mid-range Android devices when possible

### 4. Prioritize by Impact
- Fix the worst-performing metric first
- LCP issues often have the highest user-visible impact
- CLS fixes are usually the quickest wins (add dimensions to images, reserve ad space)

### 5. Avoid Regressions
- Include Web Vitals checks in your CI/CD pipeline using Lighthouse CI
- Monitor field data after every deployment
- Use performance budgets to catch regressions before they ship

```javascript
// Example: Lighthouse CI budget configuration (lighthouserc.json)
const lighthouseBudget = {
  ci: {
    assert: {
      assertions: {
        'largest-contentful-paint': ['error', { maxNumericValue: 2500 }],
        'interactive': ['error', { maxNumericValue: 3800 }],
        'cumulative-layout-shift': ['error', { maxNumericValue: 0.1 }],
        'total-blocking-time': ['warn', { maxNumericValue: 200 }]
      }
    }
  }
};
```
