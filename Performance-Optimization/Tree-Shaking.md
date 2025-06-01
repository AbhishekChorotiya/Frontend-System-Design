# Tree Shaking

## Table of Contents
- [Introduction](#introduction)
- [What is Tree Shaking?](#what-is-tree-shaking)
- [How Tree Shaking Works](#how-tree-shaking-works)
- [ES6 Modules and Tree Shaking](#es6-modules-and-tree-shaking)
- [Webpack Tree Shaking](#webpack-tree-shaking)
- [Rollup Tree Shaking](#rollup-tree-shaking)
- [Vite Tree Shaking](#vite-tree-shaking)
- [Tree Shaking in Practice](#tree-shaking-in-practice)
- [Library-Specific Tree Shaking](#library-specific-tree-shaking)
- [Side Effects and Tree Shaking](#side-effects-and-tree-shaking)
- [Optimization Techniques](#optimization-techniques)
- [Common Pitfalls](#common-pitfalls)
- [Measuring Tree Shaking Effectiveness](#measuring-tree-shaking-effectiveness)
- [Advanced Tree Shaking Strategies](#advanced-tree-shaking-strategies)
- [Common Beginner Doubts](#common-beginner-doubts)
- [Best Practices](#best-practices)
- [Conclusion](#conclusion)

## Introduction

Tree shaking is a critical optimization technique in modern frontend development that eliminates unused code from your final bundle. Named after the process of shaking a tree to remove dead leaves, tree shaking helps reduce bundle sizes, improve loading times, and enhance overall application performance. Understanding and implementing effective tree shaking strategies is essential for building performant web applications.

## What is Tree Shaking?

Tree shaking is a form of dead code elimination that removes unused exports from your JavaScript modules during the build process. It relies on the static structure of ES6 module syntax to determine which parts of your code are actually used and which can be safely removed.

### Key Concepts

1. **Dead Code Elimination**: Removing code that is never executed or referenced
2. **Static Analysis**: Analyzing code structure without executing it
3. **Module Dependencies**: Understanding which modules and exports are actually used
4. **Bundle Optimization**: Reducing the final bundle size by excluding unused code

### Before Tree Shaking

```javascript
// utils.js - A utility library
export function add(a, b) {
  return a + b;
}

export function subtract(a, b) {
  return a - b;
}

export function multiply(a, b) {
  return a * b;
}

export function divide(a, b) {
  return a / b;
}

export function complexCalculation() {
  // Large, complex function that's never used
  const result = [];
  for (let i = 0; i < 10000; i++) {
    result.push(Math.random() * Math.PI);
  }
  return result.reduce((acc, val) => acc + val, 0);
}

// main.js - Your application
import { add, subtract } from './utils.js';

console.log(add(5, 3)); // Only using add and subtract
console.log(subtract(10, 4));
```

**Without tree shaking**: All functions (add, subtract, multiply, divide, complexCalculation) are included in the final bundle.

**With tree shaking**: Only add and subtract functions are included in the final bundle.

## How Tree Shaking Works

Tree shaking works through static analysis of your code's import and export statements. Here's the process:

### 1. Dependency Graph Creation

```javascript
// The bundler creates a dependency graph
// Starting from entry points, it traces all imports

// entry.js (entry point)
import { userService } from './services/user.js';
import { formatDate } from './utils/date.js';

// services/user.js
import { apiClient } from './api.js';
export const userService = { /* ... */ };
export const adminService = { /* ... */ }; // Not imported anywhere

// utils/date.js
export function formatDate() { /* ... */ }
export function parseDate() { /* ... */ } // Not imported anywhere
```

### 2. Mark Phase

The bundler marks all reachable code starting from entry points:

```javascript
// Marked as used (reachable from entry point)
✓ userService from services/user.js
✓ formatDate from utils/date.js
✓ apiClient from services/api.js (used by userService)

// Marked as unused (not reachable)
✗ adminService from services/user.js
✗ parseDate from utils/date.js
```

### 3. Sweep Phase

Unused exports are removed from the final bundle:

```javascript
// Final bundle only includes:
// - userService and its dependencies
// - formatDate function
// - apiClient (because userService depends on it)
```

## ES6 Modules and Tree Shaking

Tree shaking relies heavily on ES6 module syntax because it provides static structure that can be analyzed at build time.

### ES6 Modules (Tree-shakable)

```javascript
// ✅ Good: ES6 named exports
export function calculateTax(amount, rate) {
  return amount * rate;
}

export function calculateDiscount(amount, percentage) {
  return amount * (percentage / 100);
}

export const TAX_RATES = {
  US: 0.08,
  EU: 0.20,
  UK: 0.20
};

// ✅ Good: ES6 default export
export default class Calculator {
  add(a, b) { return a + b; }
  subtract(a, b) { return a - b; }
}

// ✅ Good: Selective imports
import { calculateTax } from './tax-utils.js';
import Calculator from './Calculator.js';
```

### CommonJS (Not tree-shakable)

```javascript
// ❌ Bad: CommonJS exports (not tree-shakable)
function calculateTax(amount, rate) {
  return amount * rate;
}

function calculateDiscount(amount, percentage) {
  return amount * (percentage / 100);
}

// CommonJS export
module.exports = {
  calculateTax,
  calculateDiscount,
  TAX_RATES: {
    US: 0.08,
    EU: 0.20,
    UK: 0.20
  }
};

// ❌ Bad: CommonJS import (imports entire module)
const { calculateTax } = require('./tax-utils.js');
```

### Dynamic Imports (Limited tree-shaking)

```javascript
// ⚠️ Limited: Dynamic imports can't be fully tree-shaken
async function loadUtils() {
  const utils = await import('./utils.js');
  return utils.calculateTax;
}

// ✅ Better: Use dynamic imports with known exports
async function loadTaxCalculator() {
  const { calculateTax } = await import('./tax-utils.js');
  return calculateTax;
}
```

## Webpack Tree Shaking

Webpack has built-in tree shaking capabilities that work with ES6 modules.

### Basic Webpack Configuration

```javascript
// webpack.config.js
module.exports = {
  mode: 'production', // Enables tree shaking in production
  entry: './src/index.js',
  output: {
    filename: 'bundle.js',
    path: __dirname + '/dist'
  },
  optimization: {
    usedExports: true, // Mark unused exports
    sideEffects: false, // Indicate no side effects
    minimize: true // Remove unused code
  }
};
```

### Advanced Webpack Tree Shaking

```javascript
// webpack.config.js
const path = require('path');

module.exports = {
  mode: 'production',
  entry: './src/index.js',
  output: {
    filename: '[name].[contenthash].js',
    path: path.resolve(__dirname, 'dist'),
    clean: true
  },
  optimization: {
    usedExports: true,
    sideEffects: false,
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          chunks: 'all',
        },
      },
    },
  },
  resolve: {
    extensions: ['.js', '.jsx', '.ts', '.tsx']
  },
  module: {
    rules: [
      {
        test: /\.(js|jsx|ts|tsx)$/,
        exclude: /node_modules/,
        use: {
          loader: 'babel-loader',
          options: {
            presets: [
              ['@babel/preset-env', {
                modules: false // Preserve ES6 modules for tree shaking
              }],
              '@babel/preset-react'
            ]
          }
        }
      }
    ]
  }
};
```

### Package.json Configuration

```json
{
  "name": "my-app",
  "version": "1.0.0",
  "sideEffects": false,
  "main": "dist/index.js",
  "module": "src/index.js",
  "scripts": {
    "build": "webpack --mode=production",
    "analyze": "webpack-bundle-analyzer dist/bundle.js"
  },
  "devDependencies": {
    "webpack": "^5.0.0",
    "webpack-cli": "^4.0.0",
    "webpack-bundle-analyzer": "^4.0.0"
  }
}
```

## Rollup Tree Shaking

Rollup was one of the first bundlers to implement tree shaking and is known for its excellent tree shaking capabilities.

### Basic Rollup Configuration

```javascript
// rollup.config.js
import { terser } from 'rollup-plugin-terser';
import resolve from '@rollup/plugin-node-resolve';
import commonjs from '@rollup/plugin-commonjs';

export default {
  input: 'src/index.js',
  output: {
    file: 'dist/bundle.js',
    format: 'es',
    sourcemap: true
  },
  plugins: [
    resolve({
      browser: true,
      preferBuiltins: false
    }),
    commonjs(),
    terser() // Minification with dead code elimination
  ],
  external: ['react', 'react-dom'] // Don't bundle these dependencies
};
```

### Advanced Rollup Configuration

```javascript
// rollup.config.js
import { terser } from 'rollup-plugin-terser';
import resolve from '@rollup/plugin-node-resolve';
import commonjs from '@rollup/plugin-commonjs';
import babel from '@rollup/plugin-babel';
import { visualizer } from 'rollup-plugin-visualizer';

export default {
  input: 'src/index.js',
  output: [
    {
      file: 'dist/bundle.esm.js',
      format: 'es',
      sourcemap: true
    },
    {
      file: 'dist/bundle.cjs.js',
      format: 'cjs',
      sourcemap: true
    }
  ],
  plugins: [
    resolve({
      browser: true,
      preferBuiltins: false
    }),
    commonjs(),
    babel({
      babelHelpers: 'bundled',
      exclude: 'node_modules/**',
      presets: [
        ['@babel/preset-env', {
          modules: false // Preserve ES6 modules
        }]
      ]
    }),
    terser({
      compress: {
        drop_console: true, // Remove console.log statements
        drop_debugger: true, // Remove debugger statements
        pure_funcs: ['console.log'] // Remove specific function calls
      }
    }),
    visualizer({
      filename: 'dist/stats.html',
      open: true
    })
  ],
  external: id => /^react/.test(id) // External React dependencies
};
```

## Vite Tree Shaking

Vite uses Rollup under the hood for production builds, providing excellent tree shaking out of the box.

### Vite Configuration

```javascript
// vite.config.js
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import { visualizer } from 'rollup-plugin-visualizer';

export default defineConfig({
  plugins: [
    react(),
    visualizer({
      filename: 'dist/stats.html',
      open: true
    })
  ],
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['react', 'react-dom'],
          utils: ['lodash', 'date-fns']
        }
      }
    },
    minify: 'terser',
    terserOptions: {
      compress: {
        drop_console: true,
        drop_debugger: true
      }
    }
  },
  optimizeDeps: {
    include: ['react', 'react-dom']
  }
});
```

## Tree Shaking in Practice

### Creating Tree-Shakable Libraries

```javascript
// ✅ Good: Individual export files
// math/add.js
export function add(a, b) {
  return a + b;
}

// math/subtract.js
export function subtract(a, b) {
  return a - b;
}

// math/index.js - Barrel export
export { add } from './add.js';
export { subtract } from './subtract.js';
export { multiply } from './multiply.js';
export { divide } from './divide.js';

// Usage - only imports what's needed
import { add, subtract } from './math/index.js';
```

### Tree-Shakable Utility Functions

```javascript
// utils/array.js
export function chunk(array, size) {
  const chunks = [];
  for (let i = 0; i < array.length; i += size) {
    chunks.push(array.slice(i, i + size));
  }
  return chunks;
}

export function flatten(array) {
  return array.reduce((acc, val) => 
    Array.isArray(val) ? acc.concat(flatten(val)) : acc.concat(val), []);
}

export function unique(array) {
  return [...new Set(array)];
}

export function groupBy(array, key) {
  return array.reduce((groups, item) => {
    const group = item[key];
    groups[group] = groups[group] || [];
    groups[group].push(item);
    return groups;
  }, {});
}

// utils/string.js
export function capitalize(str) {
  return str.charAt(0).toUpperCase() + str.slice(1);
}

export function slugify(str) {
  return str
    .toLowerCase()
    .replace(/[^\w\s-]/g, '')
    .replace(/[\s_-]+/g, '-')
    .replace(/^-+|-+$/g, '');
}

export function truncate(str, length) {
  return str.length > length ? str.slice(0, length) + '...' : str;
}

// main.js - Only imports what's needed
import { chunk, unique } from './utils/array.js';
import { capitalize } from './utils/string.js';

// Only chunk, unique, and capitalize are included in the bundle
```

### React Component Tree Shaking

```javascript
// components/Button/Button.js
import React from 'react';
import './Button.css';

export function PrimaryButton({ children, onClick, disabled }) {
  return (
    <button 
      className="btn btn-primary" 
      onClick={onClick} 
      disabled={disabled}
    >
      {children}
    </button>
  );
}

export function SecondaryButton({ children, onClick, disabled }) {
  return (
    <button 
      className="btn btn-secondary" 
      onClick={onClick} 
      disabled={disabled}
    >
      {children}
    </button>
  );
}

export function DangerButton({ children, onClick, disabled }) {
  return (
    <button 
      className="btn btn-danger" 
      onClick={onClick} 
      disabled={disabled}
    >
      {children}
    </button>
  );
}

// components/Button/index.js
export { PrimaryButton } from './Button.js';
export { SecondaryButton } from './Button.js';
export { DangerButton } from './Button.js';

// App.js - Only imports needed components
import { PrimaryButton, SecondaryButton } from './components/Button';

function App() {
  return (
    <div>
      <PrimaryButton onClick={() => console.log('Primary')}>
        Primary Action
      </PrimaryButton>
      <SecondaryButton onClick={() => console.log('Secondary')}>
        Secondary Action
      </SecondaryButton>
      {/* DangerButton is not imported, so it won't be in the bundle */}
    </div>
  );
}
```

## Library-Specific Tree Shaking

### Lodash Tree Shaking

```javascript
// ❌ Bad: Imports entire lodash library
import _ from 'lodash';
const result = _.chunk([1, 2, 3, 4, 5], 2);

// ✅ Good: Import specific functions
import chunk from 'lodash/chunk';
import debounce from 'lodash/debounce';
const result = chunk([1, 2, 3, 4, 5], 2);

// ✅ Better: Use lodash-es for better tree shaking
import { chunk, debounce } from 'lodash-es';

// ✅ Best: Use babel-plugin-lodash for automatic optimization
// .babelrc
{
  "plugins": [
    ["lodash", { "id": ["lodash", "lodash-es"] }]
  ]
}

// This automatically transforms:
import { chunk, debounce } from 'lodash';
// Into:
import chunk from 'lodash/chunk';
import debounce from 'lodash/debounce';
```

### Material-UI Tree Shaking

```javascript
// ❌ Bad: Imports entire Material-UI library
import { Button, TextField, Dialog } from '@material-ui/core';

// ✅ Good: Import from specific paths
import Button from '@material-ui/core/Button';
import TextField from '@material-ui/core/TextField';
import Dialog from '@material-ui/core/Dialog';

// ✅ Better: Use babel-plugin-import
// .babelrc
{
  "plugins": [
    ["import", {
      "libraryName": "@material-ui/core",
      "libraryDirectory": "",
      "camel2DashComponentName": false
    }, "core"]
  ]
}
```

### React Icons Tree Shaking

```javascript
// ❌ Bad: Imports from main package
import { FaHome, FaUser } from 'react-icons/fa';

// ✅ Good: Import from specific icon sets
import { FaHome } from 'react-icons/fa/FaHome';
import { FaUser } from 'react-icons/fa/FaUser';

// ✅ Better: Use specific imports
import FaHome from 'react-icons/fa/FaHome';
import FaUser from 'react-icons/fa/FaUser';
```

### Date-fns Tree Shaking

```javascript
// ❌ Bad: Imports entire library
import * as dateFns from 'date-fns';

// ✅ Good: Import specific functions
import { format, addDays, isAfter } from 'date-fns';

// ✅ Better: Import from specific files
import format from 'date-fns/format';
import addDays from 'date-fns/addDays';
import isAfter from 'date-fns/isAfter';
```

## Side Effects and Tree Shaking

Side effects can prevent tree shaking from working effectively. Understanding and managing side effects is crucial.

### What are Side Effects?

```javascript
// ❌ Side effect: Modifying global state
window.myGlobalVar = 'some value';

// ❌ Side effect: Polyfills
import 'core-js/stable';

// ❌ Side effect: CSS imports
import './styles.css';

// ❌ Side effect: Executing code at module level
console.log('This module was loaded');

// ✅ No side effects: Pure functions
export function add(a, b) {
  return a + b;
}
```

### Managing Side Effects in package.json

```json
{
  "name": "my-library",
  "sideEffects": false,
  "main": "dist/index.js",
  "module": "src/index.js"
}

// Or specify files with side effects
{
  "name": "my-library",
  "sideEffects": [
    "*.css",
    "*.scss",
    "./src/polyfills.js"
  ]
}
```

### Webpack Side Effects Configuration

```javascript
// webpack.config.js
module.exports = {
  module: {
    rules: [
      {
        test: /\.js$/,
        sideEffects: false
      },
      {
        test: /\.css$/,
        sideEffects: true // CSS files have side effects
      }
    ]
  }
};
```

### Creating Side-Effect-Free Code

```javascript
// ❌ Bad: Side effects at module level
let cache = {};

export function memoize(fn) {
  return function(...args) {
    const key = JSON.stringify(args);
    if (cache[key]) {
      return cache[key];
    }
    const result = fn(...args);
    cache[key] = result;
    return result;
  };
}

// ✅ Good: No side effects
export function createMemoizer() {
  const cache = {};
  
  return function memoize(fn) {
    return function(...args) {
      const key = JSON.stringify(args);
      if (cache[key]) {
        return cache[key];
      }
      const result = fn(...args);
      cache[key] = result;
      return result;
    };
  };
}
```

## Optimization Techniques

### Barrel Exports Optimization

```javascript
// ❌ Bad: Re-exports everything
// index.js
export * from './utils';
export * from './components';
export * from './services';

// ✅ Good: Selective re-exports
// index.js
export { add, subtract } from './utils/math';
export { formatDate } from './utils/date';
export { Button } from './components/Button';
export { Modal } from './components/Modal';
```

### Dynamic Imports for Code Splitting

```javascript
// ✅ Good: Dynamic imports for large dependencies
async function loadChartLibrary() {
  const { Chart } = await import('chart.js');
  return Chart;
}

// ✅ Good: Conditional loading
async function loadPolyfills() {
  if (!window.fetch) {
    await import('whatwg-fetch');
  }
  
  if (!window.Promise) {
    await import('es6-promise/auto');
  }
}

// ✅ Good: Route-based code splitting
const LazyComponent = React.lazy(() => import('./LazyComponent'));

function App() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <LazyComponent />
    </Suspense>
  );
}
```

### Tree Shaking with TypeScript

```typescript
// tsconfig.json
{
  "compilerOptions": {
    "module": "ES2020", // Preserve ES modules
    "target": "ES2020",
    "moduleResolution": "node",
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true
  }
}

// ✅ Good: TypeScript with tree shaking
export interface User {
  id: string;
  name: string;
  email: string;
}

export function createUser(data: Partial<User>): User {
  return {
    id: generateId(),
    name: '',
    email: '',
    ...data
  };
}

export function validateUser(user: User): boolean {
  return !!(user.id && user.name && user.email);
}

// Only imported functions are included in bundle
import { createUser } from './user-utils';
```

## Common Pitfalls

### 1. CommonJS Modules

```javascript
// ❌ Problem: CommonJS prevents tree shaking
const utils = require('./utils');
module.exports = { processData: utils.processData };

// ✅ Solution: Use ES6 modules
import { processData } from './utils';
export { processData };
```

### 2. Default Exports with Objects

```javascript
// ❌ Problem: Default export of object
export default {
  add: (a, b) => a + b,
  subtract: (a, b) => a - b,
  multiply: (a, b) => a * b
};

// ✅ Solution: Named exports
export const add = (a, b) => a + b;
export const subtract = (a, b) => a - b;
export const multiply = (a, b) => a * b;
```

### 3. Babel Configuration Issues

```javascript
// ❌ Problem: Babel transforms ES6 modules to CommonJS
// .babelrc
{
  "presets": ["@babel/preset-env"]
}

// ✅ Solution: Preserve ES6 modules
// .babelrc
{
  "presets": [
    ["@babel/preset-env", {
      "modules": false
    }]
  ]
}
```

### 4. Side Effects in Imports

```javascript
// ❌ Problem: Import with side effects
import './polyfills'; // Executes code
import { myFunction } from './utils';

// ✅ Solution: Conditional side effects
if (needsPolyfill()) {
  import('./polyfills');
}
import { myFunction } from './utils';
```

## Measuring Tree Shaking Effectiveness

### Bundle Analysis Tools

```javascript
// webpack-bundle-analyzer
npm install --save-dev webpack-bundle-analyzer

// webpack.config.js
const BundleAnalyzerPlugin = require('webpack-bundle-analyzer').BundleAnalyzerPlugin;

module.exports = {
  plugins: [
    new BundleAnalyzerPlugin({
      analyzerMode: 'static',
      openAnalyzer: false,
      reportFilename: 'bundle-report.html'
    })
  ]
};

// Package.json script
{
  "scripts": {
    "analyze": "npm run build && npx webpack-bundle-analyzer dist/static/js/*.js"
  }
}
```

### Bundle Size Tracking

```javascript
// bundlesize configuration
// package.json
{
  "bundlesize": [
    {
      "path": "./dist/bundle.js",
      "maxSize": "100 kB"
    },
    {
      "path": "./dist/vendor.js",
      "maxSize": "50 kB"
    }
  ]
}

// CI/CD integration
npm install --save-dev bundlesize
npx bundlesize
```

### Source Map Explorer

```bash
# Install source-map-explorer
npm install --save-dev source-map-explorer

# Analyze bundle
npx source-map-explorer 'build/static/js/*.js'
```

### Custom Bundle Analysis

```javascript
// analyze-bundle.js
const fs = require('fs');
const path = require('path');

function analyzeBundleSize() {
  const bundlePath = path.join(__dirname, 'dist', 'bundle.js');
  const stats = fs.statSync(bundlePath);
  const fileSizeInBytes = stats.size;
  const fileSizeInKB = fileSizeInBytes / 1024;
  
  console.log(`Bundle size: ${fileSizeInKB.toFixed(2)} KB`);
  
  if (fileSizeInKB > 100) {
    console.warn('⚠️  Bundle size exceeds 100KB threshold');
    process.exit(1);
  } else {
    console.log('✅ Bundle size is within acceptable limits');
  }
}

analyzeBundleSize();
```

## Advanced Tree Shaking Strategies

### Micro-Frontend Tree Shaking

```javascript
// shared-utils/package.json
{
  "name": "@company/shared-utils",
  "sideEffects": false,
  "exports": {
    "./math": "./src/math/index.js",
    "./date": "./src/date/index.js",
    "./string": "./src/string/index.js"
  }
}

// app1/src/index.js
import { add, subtract } from '@company/shared-utils/math';

// app2/src/index.js
import { formatDate } from '@company/shared-utils/date';
```

### Plugin-Based Tree Shaking

```javascript
// plugin-system.js
const plugins = new Map();

export function registerPlugin(name, plugin) {
  plugins.set(name, plugin);
}

export function getPlugin(name) {
  return plugins.get(name);
}

// Individual plugins
// plugins/analytics.js
export default {
  name: 'analytics',
  track: (event) => console.log('Tracking:', event)
};

// plugins/logger.js
export default {
  name: 'logger',
  log: (message) => console.log('Log:', message)
};

// main.js - Only load needed plugins
import { registerPlugin } from './plugin-system';

// Conditionally load plugins
if (process.env.NODE_ENV === 'production') {
  const analytics = await import('./plugins/analytics');
  registerPlugin('analytics', analytics.default);
}

if (process.env.NODE_ENV === 'development') {
  const logger = await import('./plugins/logger');
  registerPlugin('logger', logger.default);
}
```

### Tree Shaking with Web Workers

```javascript
// worker-utils.js
export function createWorker(workerFunction) {
  const blob = new Blob([`(${workerFunction.toString()})()`], {
    type: 'application/javascript'
  });
  return new Worker(URL.createObjectURL(blob));
}

export function heavyCalculation(data) {
  // Heavy computation
  return data.map(x => x * x).reduce((a, b) => a + b, 0);
}

export function dataProcessing(data) {
  // Data processing logic
  return data.filter(x => x > 0).sort();
}

// main.js - Only import needed worker functions
import { createWorker, heavyCalculation } from './worker-utils';

// Only heavyCalculation is included in the bundle
const worker = createWorker(() => {
  self.onmessage = function(e) {
    // Worker code here
  };
});
```

## Common Beginner Doubts

### Q1: "Why isn't tree shaking working in my project?"

**Answer**: Tree shaking might not work due to several common issues:

1. **Using CommonJS instead of ES6 modules**:
```javascript
// ❌ This prevents tree shaking
const { myFunction } = require('./utils');

// ✅ Use ES6 imports instead
import { myFunction } from './utils';
```

2. **Babel transforming modules**:
```javascript
// Check your .babelrc - this breaks tree shaking:
{
  "presets": ["@babel/preset-env"]
}

// Fix by preserving modules:
{
  "presets": [
    ["@babel/preset-env", { "modules": false }]
  ]
}
```

3. **Side effects not properly configured**:
```json
// Add to package.json
{
  "sideEffects": false
}
```

### Q2: "How do I know if tree shaking is actually working?"

**Answer**: You can verify tree shaking effectiveness through several methods:

```javascript
// 1. Use bundle analyzer
npm install --save-dev webpack-bundle-analyzer
npx webpack-bundle-analyzer dist/bundle.js

// 2. Check bundle size before/after
// Create a test file that imports everything vs. selective imports

// 3. Use build tools' analysis
// Webpack: npm run build -- --analyze
// Vite: Use rollup-plugin-visualizer
```

### Q3: "Can I tree shake CSS and other assets?"

**Answer**: Yes, but it requires specific tools and configurations:

```javascript
// CSS tree shaking with PurgeCSS
// postcss.config.js
module.exports = {
  plugins: [
    require('@fullhuman/postcss-purgecss')({
      content: ['./src/**/*.html', './src/**/*.js', './src/**/*.jsx'],
      defaultExtractor: content => content.match(/[\w-/:]+(?<!:)/g) || []
    })
  ]
};

// Image tree shaking with webpack
module.exports = {
  module: {
    rules: [
      {
        test: /\.(png|jpe?g|gif|svg)$/,
        use: {
          loader: 'file-loader',
          options: {
            name: '[name].[hash].[ext]'
          }
        }
      }
    ]
  }
};
```

### Q4: "Should I always use named exports instead of default exports?"

**Answer**: For tree shaking, named exports are generally better:

```javascript
// ❌ Default export makes tree shaking harder
export default {
  function1,
  function2,
  function3
};

// ✅ Named exports enable better tree shaking
export { function1 };
export { function2 };
export { function3 };

// ✅ Also good for single-purpose modules
export default function MyComponent() {
  return <div>Component</div>;
}
```

### Q5: "How does tree shaking work with TypeScript?"

**Answer**: TypeScript works well with tree shaking when configured properly:

```typescript
// tsconfig.json
{
  "compilerOptions": {
    "module": "ES2020", // Preserve ES modules
    "target": "ES2020",
    "moduleResolution": "node",
    "declaration": true,
    "declarationMap": true
  }
}

// TypeScript interfaces are automatically tree-shaken
export interface User {
  id: string;
  name: string;
}

export interface Product {
  id: string;
  title: string;
}

// Only imported interfaces and functions are included
import { User } from './types'; // Product interface is not included
```

### Q6: "What's the difference between tree shaking and code splitting?"

**Answer**: They serve different purposes:

**Tree Shaking**: Removes unused code from modules
```javascript
// Before tree shaking: 100KB bundle with unused functions
// After tree shaking: 60KB bundle with only used functions
```

**Code Splitting**: Splits code into separate bundles
```javascript
// Before code splitting: 1 large bundle (200KB)
// After code splitting: Multiple smaller bundles (50KB + 50KB + 100KB)

// Dynamic import for code splitting
const LazyComponent = React.lazy(() => import('./LazyComponent'));
```

## Best Practices

### 1. Use ES6 Modules Consistently

```javascript
// ✅ Always use ES6 import/export
import { specificFunction } from './utils';
export { myFunction };

// ❌ Avoid CommonJS
const utils = require('./utils');
module.exports = myFunction;
```

### 2. Structure Your Code for Tree Shaking

```javascript
// ✅ Good: Separate files for different functionalities
// utils/math.js
export const add = (a, b) => a + b;
export const subtract = (a, b) => a - b;

// utils/string.js
export const capitalize = str => str.charAt(0).toUpperCase() + str.slice(1);
export const lowercase = str => str.toLowerCase();

// utils/index.js - Barrel exports
export * from './math';
export * from './string';
```

### 3. Configure Your Build Tools Properly

```javascript
// webpack.config.js
module.exports = {
  mode: 'production',
  optimization: {
    usedExports: true,
    sideEffects: false
  },
  module: {
    rules: [
      {
        test: /\.js$/,
        use: {
          loader: 'babel-loader',
          options: {
            presets: [['@babel/preset-env', { modules: false }]]
          }
        }
      }
    ]
  }
};
```

### 4. Mark Side Effects Appropriately

```json
// package.json
{
  "sideEffects": [
    "*.css",
    "*.scss",
    "./src/polyfills.js",
    "./src/global-setup.js"
  ]
}
```

### 5. Use Specific Imports

```javascript
// ✅ Good: Import only what you need
import { debounce } from 'lodash-es';
import { format } from 'date-fns';

// ❌ Bad: Import entire libraries
import _ from 'lodash';
import * as dateFns from 'date-fns';
```

### 6. Optimize Third-Party Libraries

```javascript
// Use babel plugins for automatic optimization
// .babelrc
{
  "plugins": [
    ["import", {
      "libraryName": "antd",
      "libraryDirectory": "es",
      "style": "css"
    }]
  ]
}

// This transforms:
import { Button, DatePicker } from 'antd';
// Into:
import Button from 'antd/es/button';
import DatePicker from 'antd/es/date-picker';
```

### 7. Monitor Bundle Size

```javascript
// Set up bundle size monitoring
// package.json
{
  "scripts": {
    "build": "webpack --mode=production",
    "analyze": "npm run build && webpack-bundle-analyzer dist/bundle.js",
    "size-check": "bundlesize"
  },
  "bundlesize": [
    {
      "path": "./dist/bundle.js",
      "maxSize": "100 kB"
    }
  ]
}
```

### 8. Use Dynamic Imports for Large Dependencies

```javascript
// ✅ Good: Load heavy libraries dynamically
async function loadChart() {
  const { Chart } = await import('chart.js');
  return new Chart(/* ... */);
}

// ✅ Good: Conditional loading
if (process.env.NODE_ENV === 'development') {
  import('./dev-tools').then(devTools => {
    devTools.setup();
  });
}
```

## Conclusion

Tree shaking is a powerful optimization technique that can significantly reduce your bundle size and improve application performance. By understanding how tree shaking works and following best practices, you can ensure that your applications only include the code they actually use.

**Key Takeaways:**

1. **Use ES6 Modules**: Tree shaking requires ES6 import/export syntax
2. **Configure Build Tools**: Ensure your bundler is set up for tree shaking
3. **Manage Side Effects**: Properly mark files with side effects
4. **Structure Code Appropriately**: Use named exports and avoid barrel exports when possible
5. **Monitor Results**: Use bundle analyzers to verify tree shaking effectiveness
6. **Optimize Dependencies**: Use tree-shakable versions of libraries when available

**Benefits of Effective Tree Shaking:**

- **Smaller Bundle Sizes**: Reduced download times and bandwidth usage
- **Faster Load Times**: Less code to parse and execute
- **Better Performance**: Improved application startup time
- **Reduced Memory Usage**: Less code loaded into memory
- **Better User Experience**: Faster, more responsive applications

Tree shaking is not a one-time setup but an ongoing optimization strategy. Regular monitoring and adjustment of your tree shaking configuration will help maintain optimal bundle sizes as your application grows and evolves.
