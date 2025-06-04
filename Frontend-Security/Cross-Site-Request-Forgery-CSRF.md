# Cross-Site Request Forgery (CSRF)

## Table of Contents
1. [Introduction](#introduction)
2. [What is CSRF?](#what-is-csrf)
3. [How CSRF Attacks Work](#how-csrf-attacks-work)
4. [Types of CSRF Attacks](#types-of-csrf-attacks)
5. [CSRF Prevention Techniques](#csrf-prevention-techniques)
6. [Implementation Examples](#implementation-examples)
7. [Frontend-Specific CSRF Protection](#frontend-specific-csrf-protection)
8. [Testing for CSRF Vulnerabilities](#testing-for-csrf-vulnerabilities)
9. [Best Practices](#best-practices)
10. [Common Beginner Doubts](#common-beginner-doubts)

## Introduction

Cross-Site Request Forgery (CSRF) is a web security vulnerability that allows an attacker to induce users to perform actions that they do not intend to perform. It's one of the most common and dangerous web application security risks, ranking in the OWASP Top 10.

CSRF attacks exploit the trust that a web application has in a user's browser, leveraging the fact that browsers automatically include credentials (cookies, authentication tokens) with requests to a domain.

## What is CSRF?

CSRF is an attack that forces an end user to execute unwanted actions on a web application in which they're currently authenticated. With a little help of social engineering (such as sending a link via email or chat), an attacker may trick the users of a web application into executing actions of the attacker's choosing.

### Key Characteristics:
- **Exploits user's authenticated session**: The attack leverages an existing authenticated session
- **Performed without user's knowledge**: The user is unaware that the malicious action is being performed
- **Uses legitimate credentials**: The attack uses the user's own authentication credentials
- **Cross-origin nature**: The malicious request originates from a different domain than the target application

## How CSRF Attacks Work

### Basic Attack Flow:

1. **User Authentication**: User logs into a legitimate website (e.g., bank.com)
2. **Session Establishment**: The website sets authentication cookies in the user's browser
3. **Malicious Site Visit**: User visits a malicious website or clicks a malicious link
4. **Forged Request**: The malicious site sends a request to the legitimate website
5. **Automatic Credential Inclusion**: Browser automatically includes authentication cookies
6. **Unauthorized Action**: The legitimate website processes the request as if it came from the user

### Visual Representation:

```
[User] ←→ [Legitimate Site] (Authenticated Session)
   ↓
[Malicious Site] → [Forged Request] → [Legitimate Site]
                                           ↓
                                    [Unwanted Action Executed]
```

## Types of CSRF Attacks

### 1. GET-based CSRF

The simplest form where malicious requests are made using GET methods:

```html
<!-- Malicious image tag that transfers money -->
<img src="https://bank.com/transfer?to=attacker&amount=1000" 
     style="display:none;">

<!-- Malicious link -->
<a href="https://bank.com/delete-account">Click here for free gift!</a>
```

### 2. POST-based CSRF

More sophisticated attacks using POST requests:

```html
<!-- Auto-submitting form -->
<form action="https://bank.com/transfer" method="POST" id="maliciousForm">
    <input type="hidden" name="to" value="attacker">
    <input type="hidden" name="amount" value="1000">
</form>

<script>
    document.getElementById('maliciousForm').submit();
</script>
```

### 3. JSON-based CSRF

Attacks targeting APIs that accept JSON payloads:

```html
<script>
fetch('https://api.bank.com/transfer', {
    method: 'POST',
    credentials: 'include', // Include cookies
    headers: {
        'Content-Type': 'application/json',
    },
    body: JSON.stringify({
        to: 'attacker',
        amount: 1000
    })
});
</script>
```

### 4. CSRF with File Upload

Exploiting file upload functionality:

```html
<form action="https://app.com/upload" method="POST" enctype="multipart/form-data">
    <input type="hidden" name="file" value="malicious-content">
    <input type="hidden" name="filename" value="../../../etc/passwd">
</form>
```

## CSRF Prevention Techniques

### 1. CSRF Tokens (Synchronizer Token Pattern)

The most common and effective defense mechanism:

#### Server-side Implementation:
```javascript
// Express.js with csurf middleware
const csrf = require('csurf');
const csrfProtection = csrf({ cookie: true });

app.use(csrfProtection);

app.get('/form', (req, res) => {
    res.render('form', { csrfToken: req.csrfToken() });
});

app.post('/transfer', (req, res) => {
    // CSRF token is automatically validated
    // Process the transfer
});
```

#### Frontend Implementation:
```html
<!-- Include CSRF token in forms -->
<form action="/transfer" method="POST">
    <input type="hidden" name="_csrf" value="{{csrfToken}}">
    <input type="text" name="to" placeholder="Recipient">
    <input type="number" name="amount" placeholder="Amount">
    <button type="submit">Transfer</button>
</form>
```

#### AJAX Implementation:
```javascript
// Get CSRF token from meta tag
const csrfToken = document.querySelector('meta[name="csrf-token"]').getAttribute('content');

// Include in AJAX requests
fetch('/api/transfer', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
        'X-CSRF-Token': csrfToken
    },
    body: JSON.stringify({
        to: 'recipient',
        amount: 100
    })
});
```

### 2. SameSite Cookie Attribute

Modern browsers support the SameSite attribute to prevent CSRF:

```javascript
// Set SameSite attribute on cookies
app.use(session({
    cookie: {
        sameSite: 'strict', // or 'lax'
        secure: true, // HTTPS only
        httpOnly: true
    }
}));
```

#### SameSite Values:
- **Strict**: Cookie never sent in cross-site requests
- **Lax**: Cookie sent with top-level navigation but not with embedded requests
- **None**: Cookie sent with all cross-site requests (requires Secure flag)

### 3. Double Submit Cookie Pattern

Alternative to CSRF tokens:

```javascript
// Server sets a random value in both cookie and form field
app.get('/form', (req, res) => {
    const randomValue = generateRandomString();
    res.cookie('csrf-token', randomValue);
    res.render('form', { csrfToken: randomValue });
});

// Verify both values match
app.post('/transfer', (req, res) => {
    const cookieToken = req.cookies['csrf-token'];
    const formToken = req.body._csrf;
    
    if (cookieToken !== formToken) {
        return res.status(403).send('CSRF token mismatch');
    }
    
    // Process request
});
```

### 4. Origin and Referer Header Validation

```javascript
app.use((req, res, next) => {
    const origin = req.get('Origin') || req.get('Referer');
    const allowedOrigins = ['https://myapp.com', 'https://www.myapp.com'];
    
    if (req.method !== 'GET' && !allowedOrigins.includes(origin)) {
        return res.status(403).send('Invalid origin');
    }
    
    next();
});
```

### 5. Custom Headers

Require custom headers that cannot be set by simple forms:

```javascript
// Frontend
fetch('/api/transfer', {
    method: 'POST',
    headers: {
        'X-Requested-With': 'XMLHttpRequest',
        'Content-Type': 'application/json'
    },
    body: JSON.stringify(data)
});

// Backend validation
app.use((req, res, next) => {
    if (req.method !== 'GET' && !req.get('X-Requested-With')) {
        return res.status(403).send('Missing required header');
    }
    next();
});
```

## Implementation Examples

### React Application with CSRF Protection

```jsx
// CSRFToken component
import React, { useState, useEffect } from 'react';

const CSRFToken = () => {
    const [csrfToken, setCsrfToken] = useState('');

    useEffect(() => {
        fetch('/api/csrf-token')
            .then(response => response.json())
            .then(data => setCsrfToken(data.token));
    }, []);

    return (
        <input type="hidden" name="_csrf" value={csrfToken} />
    );
};

// Transfer form component
const TransferForm = () => {
    const [formData, setFormData] = useState({
        to: '',
        amount: ''
    });

    const handleSubmit = async (e) => {
        e.preventDefault();
        
        const csrfToken = document.querySelector('input[name="_csrf"]').value;
        
        try {
            const response = await fetch('/api/transfer', {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json',
                    'X-CSRF-Token': csrfToken
                },
                body: JSON.stringify(formData)
            });
            
            if (response.ok) {
                alert('Transfer successful!');
            } else {
                alert('Transfer failed!');
            }
        } catch (error) {
            console.error('Error:', error);
        }
    };

    return (
        <form onSubmit={handleSubmit}>
            <CSRFToken />
            <input
                type="text"
                placeholder="Recipient"
                value={formData.to}
                onChange={(e) => setFormData({...formData, to: e.target.value})}
            />
            <input
                type="number"
                placeholder="Amount"
                value={formData.amount}
                onChange={(e) => setFormData({...formData, amount: e.target.value})}
            />
            <button type="submit">Transfer</button>
        </form>
    );
};
```

### Vue.js with CSRF Protection

```vue
<template>
    <form @submit.prevent="submitForm">
        <input type="hidden" name="_csrf" :value="csrfToken">
        <input v-model="recipient" type="text" placeholder="Recipient">
        <input v-model="amount" type="number" placeholder="Amount">
        <button type="submit">Transfer</button>
    </form>
</template>

<script>
export default {
    data() {
        return {
            csrfToken: '',
            recipient: '',
            amount: ''
        };
    },
    
    async mounted() {
        try {
            const response = await fetch('/api/csrf-token');
            const data = await response.json();
            this.csrfToken = data.token;
        } catch (error) {
            console.error('Failed to fetch CSRF token:', error);
        }
    },
    
    methods: {
        async submitForm() {
            try {
                const response = await fetch('/api/transfer', {
                    method: 'POST',
                    headers: {
                        'Content-Type': 'application/json',
                        'X-CSRF-Token': this.csrfToken
                    },
                    body: JSON.stringify({
                        to: this.recipient,
                        amount: this.amount
                    })
                });
                
                if (response.ok) {
                    this.$emit('transfer-success');
                } else {
                    this.$emit('transfer-error');
                }
            } catch (error) {
                console.error('Transfer failed:', error);
            }
        }
    }
};
</script>
```

### Angular Service for CSRF Protection

```typescript
// csrf.service.ts
import { Injectable } from '@angular/core';
import { HttpClient, HttpHeaders } from '@angular/common/http';
import { Observable } from 'rxjs';

@Injectable({
    providedIn: 'root'
})
export class CsrfService {
    private csrfToken: string = '';

    constructor(private http: HttpClient) {
        this.initializeCsrfToken();
    }

    private async initializeCsrfToken(): Promise<void> {
        try {
            const response = await this.http.get<{token: string}>('/api/csrf-token').toPromise();
            this.csrfToken = response.token;
        } catch (error) {
            console.error('Failed to initialize CSRF token:', error);
        }
    }

    public getHeaders(): HttpHeaders {
        return new HttpHeaders({
            'Content-Type': 'application/json',
            'X-CSRF-Token': this.csrfToken
        });
    }

    public makeSecureRequest(url: string, data: any): Observable<any> {
        return this.http.post(url, data, { headers: this.getHeaders() });
    }
}

// transfer.component.ts
import { Component } from '@angular/core';
import { CsrfService } from './csrf.service';

@Component({
    selector: 'app-transfer',
    template: `
        <form (ngSubmit)="onSubmit()">
            <input [(ngModel)]="recipient" placeholder="Recipient" name="recipient">
            <input [(ngModel)]="amount" type="number" placeholder="Amount" name="amount">
            <button type="submit">Transfer</button>
        </form>
    `
})
export class TransferComponent {
    recipient: string = '';
    amount: number = 0;

    constructor(private csrfService: CsrfService) {}

    onSubmit(): void {
        const transferData = {
            to: this.recipient,
            amount: this.amount
        };

        this.csrfService.makeSecureRequest('/api/transfer', transferData)
            .subscribe(
                response => console.log('Transfer successful:', response),
                error => console.error('Transfer failed:', error)
            );
    }
}
```

## Frontend-Specific CSRF Protection

### 1. Axios Interceptors for CSRF

```javascript
// Set up Axios interceptor for automatic CSRF token inclusion
import axios from 'axios';

// Get CSRF token from meta tag or cookie
const getCSRFToken = () => {
    return document.querySelector('meta[name="csrf-token"]')?.getAttribute('content') ||
           document.cookie.split('; ').find(row => row.startsWith('csrf-token='))?.split('=')[1];
};

// Request interceptor
axios.interceptors.request.use(
    config => {
        const token = getCSRFToken();
        if (token) {
            config.headers['X-CSRF-Token'] = token;
        }
        return config;
    },
    error => Promise.reject(error)
);

// Response interceptor to handle CSRF token refresh
axios.interceptors.response.use(
    response => response,
    error => {
        if (error.response?.status === 403 && error.response?.data?.message === 'CSRF token mismatch') {
            // Refresh CSRF token and retry request
            return refreshCSRFTokenAndRetry(error.config);
        }
        return Promise.reject(error);
    }
);

const refreshCSRFTokenAndRetry = async (originalRequest) => {
    try {
        const response = await axios.get('/api/csrf-token');
        const newToken = response.data.token;
        
        // Update meta tag
        document.querySelector('meta[name="csrf-token"]').setAttribute('content', newToken);
        
        // Retry original request with new token
        originalRequest.headers['X-CSRF-Token'] = newToken;
        return axios(originalRequest);
    } catch (error) {
        console.error('Failed to refresh CSRF token:', error);
        throw error;
    }
};
```

### 2. Custom Hook for CSRF in React

```jsx
// useCSRF hook
import { useState, useEffect } from 'react';

const useCSRF = () => {
    const [csrfToken, setCsrfToken] = useState('');
    const [loading, setLoading] = useState(true);
    const [error, setError] = useState(null);

    const fetchCSRFToken = async () => {
        try {
            setLoading(true);
            const response = await fetch('/api/csrf-token');
            if (!response.ok) {
                throw new Error('Failed to fetch CSRF token');
            }
            const data = await response.json();
            setCsrfToken(data.token);
            setError(null);
        } catch (err) {
            setError(err.message);
        } finally {
            setLoading(false);
        }
    };

    useEffect(() => {
        fetchCSRFToken();
    }, []);

    const makeSecureRequest = async (url, options = {}) => {
        const defaultOptions = {
            headers: {
                'Content-Type': 'application/json',
                'X-CSRF-Token': csrfToken,
                ...options.headers
            }
        };

        try {
            const response = await fetch(url, { ...options, ...defaultOptions });
            
            if (response.status === 403) {
                // Token might be expired, refresh and retry
                await fetchCSRFToken();
                defaultOptions.headers['X-CSRF-Token'] = csrfToken;
                return fetch(url, { ...options, ...defaultOptions });
            }
            
            return response;
        } catch (error) {
            throw error;
        }
    };

    return {
        csrfToken,
        loading,
        error,
        makeSecureRequest,
        refreshToken: fetchCSRFToken
    };
};

// Usage in component
const TransferComponent = () => {
    const { csrfToken, makeSecureRequest, loading } = useCSRF();
    const [formData, setFormData] = useState({ to: '', amount: '' });

    const handleSubmit = async (e) => {
        e.preventDefault();
        
        try {
            const response = await makeSecureRequest('/api/transfer', {
                method: 'POST',
                body: JSON.stringify(formData)
            });
            
            if (response.ok) {
                alert('Transfer successful!');
            }
        } catch (error) {
            console.error('Transfer failed:', error);
        }
    };

    if (loading) return <div>Loading...</div>;

    return (
        <form onSubmit={handleSubmit}>
            {/* Form fields */}
        </form>
    );
};
```

### 3. CSRF Protection Middleware for Express

```javascript
// csrf-middleware.js
const crypto = require('crypto');

class CSRFProtection {
    constructor(options = {}) {
        this.secret = options.secret || process.env.CSRF_SECRET || 'default-secret';
        this.tokenLength = options.tokenLength || 32;
        this.cookieName = options.cookieName || 'csrf-token';
        this.headerName = options.headerName || 'x-csrf-token';
    }

    generateToken() {
        return crypto.randomBytes(this.tokenLength).toString('hex');
    }

    createHash(token, secret) {
        return crypto.createHmac('sha256', secret).update(token).digest('hex');
    }

    verifyToken(token, hash) {
        const expectedHash = this.createHash(token, this.secret);
        return crypto.timingSafeEqual(Buffer.from(hash), Buffer.from(expectedHash));
    }

    middleware() {
        return (req, res, next) => {
            // Generate token for GET requests
            if (req.method === 'GET') {
                const token = this.generateToken();
                const hash = this.createHash(token, this.secret);
                
                res.cookie(this.cookieName, hash, {
                    httpOnly: true,
                    secure: process.env.NODE_ENV === 'production',
                    sameSite: 'strict'
                });
                
                req.csrfToken = () => token;
                return next();
            }

            // Verify token for state-changing requests
            const token = req.get(this.headerName) || req.body._csrf;
            const hash = req.cookies[this.cookieName];

            if (!token || !hash) {
                return res.status(403).json({ error: 'CSRF token missing' });
            }

            if (!this.verifyToken(token, hash)) {
                return res.status(403).json({ error: 'CSRF token invalid' });
            }

            next();
        };
    }
}

// Usage
const csrfProtection = new CSRFProtection({
    secret: process.env.CSRF_SECRET
});

app.use(csrfProtection.middleware());

// Endpoint to get CSRF token
app.get('/api/csrf-token', (req, res) => {
    res.json({ token: req.csrfToken() });
});
```

## Testing for CSRF Vulnerabilities

### 1. Manual Testing

```html
<!-- Create a test page to check for CSRF vulnerabilities -->
<!DOCTYPE html>
<html>
<head>
    <title>CSRF Test</title>
</head>
<body>
    <h1>CSRF Vulnerability Test</h1>
    
    <!-- Test GET-based CSRF -->
    <img src="https://target-app.com/delete-account" style="display:none;">
    
    <!-- Test POST-based CSRF -->
    <form action="https://target-app.com/transfer" method="POST" id="csrfTest">
        <input type="hidden" name="to" value="attacker">
        <input type="hidden" name="amount" value="1000">
    </form>
    
    <script>
        // Auto-submit form
        document.getElementById('csrfTest').submit();
        
        // Test JSON-based CSRF
        fetch('https://target-app.com/api/transfer', {
            method: 'POST',
            credentials: 'include',
            headers: {
                'Content-Type': 'application/json'
            },
            body: JSON.stringify({
                to: 'attacker',
                amount: 1000
            })
        }).then(response => {
            console.log('CSRF test result:', response.status);
        });
    </script>
</body>
</html>
```

### 2. Automated Testing with Jest

```javascript
// csrf.test.js
const request = require('supertest');
const app = require('../app');

describe('CSRF Protection', () => {
    let csrfToken;
    let cookies;

    beforeEach(async () => {
        // Get CSRF token
        const response = await request(app)
            .get('/api/csrf-token')
            .expect(200);
        
        csrfToken = response.body.token;
        cookies = response.headers['set-cookie'];
    });

    test('should reject requests without CSRF token', async () => {
        await request(app)
            .post('/api/transfer')
            .send({ to: 'test', amount: 100 })
            .expect(403);
    });

    test('should reject requests with invalid CSRF token', async () => {
        await request(app)
            .post('/api/transfer')
            .set('X-CSRF-Token', 'invalid-token')
            .set('Cookie', cookies)
            .send({ to: 'test', amount: 100 })
            .expect(403);
    });

    test('should accept requests with valid CSRF token', async () => {
        await request(app)
            .post('/api/transfer')
            .set('X-CSRF-Token', csrfToken)
            .set('Cookie', cookies)
            .send({ to: 'test', amount: 100 })
            .expect(200);
    });

    test('should reject cross-origin requests', async () => {
        await request(app)
            .post('/api/transfer')
            .set('Origin', 'https://malicious-site.com')
            .set('X-CSRF-Token', csrfToken)
            .set('Cookie', cookies)
            .send({ to: 'test', amount: 100 })
            .expect(403);
    });
});
```

### 3. Browser Extension for CSRF Testing

```javascript
// content-script.js for browser extension
(function() {
    'use strict';

    const testCSRF = () => {
        const forms = document.querySelectorAll('form[method="POST"], form[method="post"]');
        const results = [];

        forms.forEach((form, index) => {
            const hasCSRFToken = form.querySelector('input[name*="csrf"], input[name*="token"]');
            const action = form.getAttribute('action');
            
            results.push({
                formIndex: index,
                action: action,
                hasCSRFProtection: !!hasCSRFToken,
                vulnerability: !hasCSRFToken ? 'Potential CSRF vulnerability' : 'Protected'
            });
        });

        console.table(results);
        
        // Test AJAX endpoints
        const testAjaxCSRF = () => {
            fetch('/api/test-endpoint', {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json'
                },
                body: JSON.stringify({ test: 'data' })
            }).then(response => {
                console.log('AJAX CSRF test:', response.status === 403 ? 'Protected' : 'Vulnerable');
            }).catch(error => {
                console.log('AJAX CSRF test error:', error);
            });
        };

        testAjaxCSRF();
    };

    // Run test when page loads
    if (document.readyState === 'loading') {
        document.addEventListener('DOMContentLoaded', testCSRF);
    } else {
        testCSRF();
    }
})();
```

## Best Practices

### 1. Defense in Depth

```javascript
// Implement multiple layers of CSRF protection
app.use((req, res, next) => {
    // 1. Check SameSite cookies
    if (req.method !== 'GET' && !req.cookies.sessionId) {
        return res.status(403).json({ error: 'No session cookie' });
    }

    // 2. Validate Origin/Referer
    const origin = req.get('Origin') || req.get('Referer');
    if (req.method !== 'GET' && !isValidOrigin(origin)) {
        return res.status(403).json({ error: 'Invalid origin' });
    }

    // 3. Check CSRF token
    const token = req.get('X-CSRF-Token') || req.body._csrf;
    if (req.method !== 'GET' && !isValidCSRFToken(token, req)) {
        return res.status(403).json({ error: 'Invalid CSRF token' });
    }

    next();
});
```

### 2. Secure Token Generation

```javascript
const crypto = require('crypto');

// Use cryptographically secure random number generation
const generateSecureToken = () => {
    return crypto.randomBytes(32).toString('base64url');
};

// Use timing-safe comparison
const compareTokens = (token1, token2) => {
    if (token1.length !== token2.length) {
        return false;
    }
    return crypto.timingSafeEqual(
        Buffer.from(token1),
        Buffer.from(token2)
    );
};
```

### 3. Token Rotation

```javascript
// Rotate CSRF tokens periodically
class CSRFTokenManager {
    constructor() {
        this.tokens = new Map();
        this.tokenLifetime = 30 * 60 * 1000; // 30 minutes
    }

    generateToken(sessionId) {
        const token = crypto.randomBytes(32).toString('hex');
        const expiry = Date.now() + this.tokenLifetime;
        
        this.tokens.set(sessionId, { token, expiry });
        
        // Clean up expired tokens
        this.cleanupExpiredTokens();
        
        return token;
    }

    validateToken(sessionId, providedToken) {
        const tokenData = this.tokens.get(sessionId);
        
        if (!tokenData || Date.now() > tokenData.expiry) {
            return false;
        }
        
        return crypto.timingSafeEqual(
            Buffer.from(tokenData.token),
            Buffer.from(providedToken)
        );
    }

    cleanupExpiredTokens() {
        const now = Date.now();
        for (const [sessionId, tokenData] of this.tokens.entries()) {
            if (now > tokenData.expiry) {
                this.tokens.delete(sessionId);
            }
        }
    }
}
```

### 4. Content Security Policy (CSP)

```javascript
// Add CSP headers to prevent malicious script injection
app.use((req, res, next) => {
    res.setHeader('Content-Security-Policy', 
        "default-src 'self'; " +
        "script-src 'self' 'unsafe-inline'; " +
        "style-src 'self' 'unsafe-inline'; " +
        "form-action 'self';"
    );
    next();
});
```

### 5. Logging and Monitoring

```javascript
// Log CSRF attempts for monitoring
const logCSRFAttempt = (req, reason) => {
    console.warn('CSRF attempt detected:', {
        ip: req.ip,
        userAgent: req.get('User-Agent'),
        origin: req.get('Origin'),
        referer: req.get('Referer'),
        reason: reason,
        timestamp: new Date().toISOString()
    });
};

// Enhanced CSRF middleware with logging
app.use((req, res, next) => {
    if (req.method === 'GET') return next();

    const token = req.get('X-CSRF-Token');
    if (!token) {
        logCSRFAttempt(req, 'Missing CSRF token');
        return res.status(403).json({ error: 'CSRF token required' });
    }

    if (!validateCSRFToken(token, req)) {
        logCSRFAttempt(req, 'Invalid CSRF token');
        return res.status(403).json({ error: 'Invalid CSRF token' });
    }

    next();
});
```

## Common Beginner Doubts

### Q1: What's the difference between CSRF and XSS?

**Answer**: While both are web security vulnerabilities, they work differently:

- **CSRF**: Exploits the trust a website has in a user's browser. The attacker tricks the user into performing unwanted actions on a site where they're authenticated.
- **XSS**: Exploits the trust a user has in a website. The attacker injects malicious scripts into the website that execute in the user's browser.

**CSRF Example**: Attacker tricks you into transferring money from your bank account.
**XSS Example**: Attacker injects script into a website that steals your session cookies.

### Q2: Why can't I just use HTTPS to prevent CSRF?

**Answer**: HTTPS encrypts data in transit but doesn't prevent CSRF attacks because:
- CSRF attacks don't rely on intercepting or modifying data
- The malicious request is sent directly from the user's browser to the legitimate site
- HTTPS protects against man-in-the-middle attacks, not against requests originating from malicious sites

### Q3: Is checking the Referer header enough for CSRF protection?

**Answer**: No, relying solely on Referer headers is insufficient because:
- Referer headers can be spoofed or omitted
- Some browsers or privacy tools strip Referer headers
- Corporate firewalls might remove Referer headers
- It's better used as an additional layer of defense, not the primary protection

```javascript
// Don't rely only on this
app.use((req, res, next) => {
    const referer = req.get('Referer');
    if (req.method !== 'GET' && !referer?.startsWith('https://myapp.com')) {
        return res.status(403).send('Invalid referer');
    }
    next();
});
```

### Q4: Do I need CSRF protection for APIs that use JWT tokens?

**Answer**: It depends on how the JWT is stored and transmitted:

- **If JWT is stored in localStorage/sessionStorage**: CSRF protection is not needed because these storages are not automatically sent with requests
- **If JWT is stored in cookies**: CSRF protection is still needed because cookies are automatically sent with requests
- **If JWT is sent in Authorization header**: CSRF protection is not needed for API calls, but you still need it for traditional form submissions

```javascript
// JWT in localStorage - No CSRF needed for API calls
const token = localStorage.getItem('jwt');
fetch('/api/data', {
    headers: {
        'Authorization': `Bearer ${token}`
    }
});

// JWT in cookies - CSRF protection needed
fetch('/api/data', {
    credentials: 'include', // Sends cookies automatically
    headers: {
        'X-CSRF-Token': csrfToken // Still need CSRF token
    }
});
```

### Q5: Can SameSite cookies completely replace CSRF tokens?

**Answer**: SameSite cookies provide excellent CSRF protection but have limitations:

**Advantages:**
- Simple to implement
- No need for token management
- Works automatically

**Limitations:**
- Not supported by all browsers (especially older ones)
- `SameSite=Lax` allows some cross-site requests (top-level navigation)
- `SameSite=Strict` might break legitimate functionality
- Doesn't protect against subdomain attacks

**Best Practice**: Use SameSite cookies as a first line of defense, but combine with CSRF tokens for comprehensive protection.

### Q6: Why do some sites use double-submit cookies instead of synchronizer tokens?

**Answer**: Double-submit cookies are used when:

- **Stateless applications**: No server-side session storage available
- **Microservices**: Multiple services need to validate tokens without shared state
- **Performance**: Reduces server-side storage requirements
- **Scalability**: Easier to scale horizontally

```javascript
// Double-submit pattern
// 1. Server sets random value in cookie
res.cookie('csrf-token', randomValue);

// 2. Client includes same value in request header/body
fetch('/api/endpoint', {
    headers: {
        'X-CSRF-Token': getCookieValue('csrf-token')
    }
});

// 3. Server compares cookie value with header value
const cookieToken = req.cookies['csrf-token'];
const headerToken = req.get('X-CSRF-Token');
if (cookieToken !== headerToken) {
    return res.status(403).send('CSRF token mismatch');
}
```

### Q7: How do I handle CSRF protection in Single Page Applications (SPAs)?

**Answer**: SPAs require special consideration for CSRF protection:

```javascript
// 1. Get CSRF token on app initialization
class CSRFManager {
    constructor() {
        this.token = null;
        this.initialize();
    }

    async initialize() {
        try {
            const response = await fetch('/api/csrf-token');
            const data = await response.json();
            this.token = data.token;
        } catch (error) {
            console.error('Failed to initialize CSRF token:', error);
        }
    }

    async refreshToken() {
        await this.initialize();
    }

    getToken() {
        return this.token;
    }
}

// 2. Use with HTTP client
const csrfManager = new CSRFManager();

// 3. Include in all state-changing requests
const apiCall = async (url, data) => {
    const response = await fetch(url, {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
            'X-CSRF-Token': csrfManager.getToken()
        },
        body: JSON.stringify(data)
    });

    // Handle token expiration
    if (response.status === 403) {
        await csrfManager.refreshToken();
        // Retry request with new token
        return fetch(url, {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
                'X-CSRF-Token': csrfManager.getToken()
            },
            body: JSON.stringify(data)
        });
    }

    return response;
};
```

### Q8: What happens if a user has multiple tabs open?

**Answer**: Multiple tabs can cause CSRF token synchronization issues:

**Problem**: Each tab might have a different CSRF token, causing conflicts.

**Solutions:**

1. **Shared token storage**:
```javascript
// Use sessionStorage to share tokens across tabs
const getCSRFToken = () => {
    return sessionStorage.getItem('csrf-token');
};

const setCSRFToken = (token) => {
    sessionStorage.setItem('csrf-token', token);
};
```

2. **Token refresh on focus**:
```javascript
// Refresh token when tab gains focus
window.addEventListener('focus', async () => {
    await refreshCSRFToken();
});
```

3. **Longer token lifetime**:
```javascript
// Use longer-lived tokens to reduce conflicts
const tokenLifetime = 60 * 60 * 1000; // 1 hour instead of 30 minutes
```

### Q9: How do I test if my CSRF protection is working?

**Answer**: Here's a comprehensive testing approach:

```javascript
// 1. Unit tests for CSRF middleware
describe('CSRF Protection', () => {
    test('should reject requests without token', async () => {
        const response = await request(app)
            .post('/api/transfer')
            .send({ amount: 100 });
        expect(response.status).toBe(403);
    });

    test('should accept requests with valid token', async () => {
        const tokenResponse = await request(app).get('/api/csrf-token');
        const token = tokenResponse.body.token;
        
        const response = await request(app)
            .post('/api/transfer')
            .set('X-CSRF-Token', token)
            .send({ amount: 100 });
        expect(response.status).toBe(200);
    });
});

// 2. Manual testing with browser console
// Open browser console on your site and run:
fetch('/api/sensitive-endpoint', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ data: 'test' })
}).then(r => console.log('Status:', r.status));
// Should return 403 if CSRF protection is working

// 3. Cross-origin test
// Create a simple HTML file and open it:
/*
<!DOCTYPE html>
<html>
<body>
<script>
fetch('https://your-app.com/api/sensitive-endpoint', {
    method: 'POST',
    credentials: 'include',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ data: 'test' })
}).then(r => console.log('Cross-origin test status:', r.status));
</script>
</body>
</html>
*/
```

### Q10: Are there any performance implications of CSRF protection?

**Answer**: CSRF protection has minimal performance impact, but consider:

**Token Generation**: 
- Cryptographic random generation is fast
- Consider caching tokens for short periods

**Token Validation**:
- String comparison is very fast
- Use timing-safe comparison to prevent timing attacks

**Storage**:
- In-memory storage is fastest
- Database storage adds latency
- Consider Redis for distributed applications

```javascript
// Optimized CSRF implementation
class OptimizedCSRF {
    constructor() {
        this.tokenCache = new Map();
        this.cacheTimeout = 5 * 60 * 1000; // 5 minutes
    }

    generateToken(sessionId) {
        // Check cache first
        const cached = this.tokenCache.get(sessionId);
        if (cached && Date.now() < cached.expiry) {
            return cached.token;
        }

        // Generate new token
        const token = crypto.randomBytes(32).toString('hex');
        const expiry = Date.now() + this.cacheTimeout;
        
        this.tokenCache.set(sessionId, { token, expiry });
        return token;
    }

    // Cleanup expired tokens periodically
    startCleanup() {
        setInterval(() => {
            const now = Date.now();
            for (const [sessionId, data] of this.tokenCache.entries()) {
                if (now > data.expiry) {
                    this.tokenCache.delete(sessionId);
                }
            }
        }, 60000); // Clean every minute
    }
}
```

---

## Summary

Cross-Site Request Forgery (CSRF) is a serious security vulnerability that can be effectively prevented through proper implementation of defense mechanisms. The key takeaways are:

1. **Use CSRF tokens** as the primary defense mechanism
2. **Implement SameSite cookies** for additional protection
3. **Validate Origin/Referer headers** as a secondary check
4. **Use defense in depth** - combine multiple protection methods
5. **Test thoroughly** to ensure protection is working correctly
6. **Monitor and log** CSRF attempts for security analysis

Remember that CSRF protection is just one part of a comprehensive security strategy. Always combine it with other security measures like proper authentication, authorization, input validation, and secure coding practices.
