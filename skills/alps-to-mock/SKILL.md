---
name: alps-to-mock
description: Generate website mock (HTML pages + JSON API responses) from ALPS profile. Converts ALPS semantic descriptors to static HTML pages and HAL-format JSON API mock files.
---

# ALPS to Website Mock Generator

Generate static HTML pages and JSON API mock responses from an ALPS profile.

## When to Use

- User asks to generate a website mock from ALPS
- User wants to create HTML prototypes from ALPS profile
- User says "alps2mock", "generate mock", "create mock site"

## Workflow

### Step 1: Interactive Setup (one question at a time)

Ask questions **one at a time**, waiting for the user's response before proceeding to the next. After all questions are answered, show a summary and ask for confirmation.

**Q1: ALPS profile path**

Ask: "What is the path to the ALPS profile?"

**Q2: Fidelity level**

Ask: "Choose the default CSS fidelity level. (All three CSS files are always generated; this selects the initial `<link>`.)"

1. **Bare HTML** — Minimal padding and margins for basic readability
2. **Wireframe** — Layout structure visible. X-box image placeholders, ALPS ID labels, no color or decoration
3. **Production quality** — Realistic design with fonts, colors, hover states, and responsive layout

**Q3: i18n label overrides**

Ask: "Generate an i18n label override file (labels.json)?"
- **Yes** — Generate `i18n/labels.json`
- **No** — Use ALPS `title`/`doc` values as-is

**Q4: Output directory**

Ask: "Specify the output directory. (Default: `mock/`)"

**Confirmation: Show summary and ask**

After all questions, show a settings summary and ask for confirmation:

```
Settings:
- ALPS profile: {path}
- Fidelity: {level}
- i18n file: {yes/no}
- Output: {directory}

Proceed with these settings?
```

Start generation only after user approval.

### Step 2: Read ALPS Profile

1. Read the ALPS profile (JSON or XML)
2. Identify:
   - **Semantic descriptors** — Fields with data (no `type` or `type="semantic"`)
   - **States** — Descriptors containing nested field references (Taxonomy)
   - **Transitions** — Descriptors with `type` (safe/unsafe/idempotent) and `rt` (Choreography)
3. Build a state transition map: which states link to which states via which transitions

### Step 3: Generate Files

Generate all HTML pages, CSS files (level1/2/3), and JSON API files according to the mapping rules below.

**ALPS profile link:**
- If the ALPS profile is specified as a URL: use `<link rel="profile" href="{URL}">` as-is
- If the ALPS profile is specified as a file path: copy to `profile/alps.json` and use `<link rel="profile" href="../profile/alps.json">` (convert XML to JSON if needed)

Also copy `mock-switch` utility from the skill directory to the output root:

```bash
cp /path/to/skills/alps-to-mock/mock-switch {output-dir}/
chmod +x {output-dir}/mock-switch
```

### Step 4: Self-Validation

After generation:
1. Verify all inter-page links resolve to existing files
2. Verify all HTML files are valid (proper `<!DOCTYPE html>`, closed tags, required attributes)
3. Verify all JSON files are valid JSON
4. Report any broken links or issues

## Mapping Rules

### States → HTML Pages

Each state descriptor becomes an HTML page:

| ALPS | Output |
|------|--------|
| State descriptor `id="Foo"` | `html/foo.html` (lowercase) |
| First state or `Home` state | `html/index.html` (also linked as entry point) |

### Semantic Fields → HTML Elements

Fields within a state descriptor map to HTML elements based on their `def` (schema.org definition):

| schema.org def | Display context | Input context |
|----------------|----------------|---------------|
| `schema.org/identifier` | `<span>` (display only) | `<input type="hidden">` |
| `schema.org/name` | `<span>` or `<h2>` | `<input type="text">` |
| `schema.org/email` | `<a href="mailto:">` | `<input type="email">` |
| `schema.org/URL` | `<a href="">` | `<input type="url">` |
| `schema.org/DateTime` | `<time>` | `<input type="datetime-local">` |
| `schema.org/Date` | `<time>` | `<input type="date">` |
| `schema.org/Boolean` | `<span>` | `<input type="checkbox">` |
| `schema.org/Text` | `<p>` | `<textarea>` |
| `schema.org/image` | `<img>` | `<input type="file" accept="image/*">` |
| `schema.org/Number` / `Integer` | `<span>` | `<input type="number">` |
| `schema.org/price` / `totalPrice` | `<span>` (formatted currency) | `<input type="number" step="0.01">` |
| `schema.org/isbn` | `<span>` | `<input type="text" pattern="[0-9-]+">` |
| `schema.org/address` | `<address>` | `<textarea>` |
| `schema.org/quantityValue` | `<span>` | `<input type="number" min="1">` |
| `schema.org/author` | `<span>` | `<input type="text">` |
| `schema.org/category` | `<span>` | `<select>` |
| (no def / unknown) | `<span>` | `<input type="text">` |

**Display vs Input context:**
- Fields referenced by a safe transition (read operation) or displayed within a state → Display context
- Fields referenced by an unsafe/idempotent transition (write operation) as nested descriptors → Input context (inside `<form>`)

### Transitions → Navigation / Forms

| ALPS type | HTML element | Details |
|-----------|-------------|---------|
| `safe` | `<a href="{target-state}.html" title="{doc}">` | Navigation link. ALPS `title` → link text, ALPS `doc` → HTML `title` attribute (tooltip). |
| `safe` with input fields | `<form method="get" action="{target-state}.html">` | Search/filter form with fields as `<input>`. |
| `unsafe` | `<form method="get" action="{target-state}.html">` | Create/action form. Nested descriptors become form fields. |
| `idempotent` | `<form method="get" action="{target-state}.html">` | Update/delete form. Nested descriptors become form fields. |

**Note:** Forms use `method="get"` for static hosting browsability. The transition semantics (safe/unsafe/idempotent) are carried by the ALPS class name, not the HTTP method.

**Transition target:** The `rt` attribute determines the target page. `rt="#Foo"` → links to `foo.html`.

### Transition Placement

Transitions are placed in two areas, mirroring HAL's separation of content and `_links`:

- **`<main>`** — Only inline forms (unsafe/idempotent transitions) directly related to the content. e.g., "Add to Cart" form placed right after the product info
- **`<nav>` (bottom of page, before `<footer>`)** — Safe transition links grouped together. Corresponds to HAL's `_links` section

This achieves:
1. Clear separation of content (semantic fields) and transitions (links)
2. Structural correspondence with HAL JSON's `_links` section
3. Reflects the ALPS Ontology/Taxonomy/Choreography separation

## HTML Template

### Base Structure (HTML Living Standard)

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>{State title} - {ALPS title}</title>
  <link rel="profile" href="{ALPS profile URL or ../profile/alps.json}">
  <link rel="stylesheet" href="../css/level{N}.css">
</head>
<body>
  <header>
    <h1>{State title}</h1>
  </header>
  <main>
    <!-- State content: semantic fields and inline forms (unsafe/idempotent) -->
  </main>
  <nav>
    <h2>Links</h2>
    <ul>
      <!-- Safe transitions as navigation links (corresponds to HAL _links) -->
      <li><a href="{target}.html">{transition title}</a></li>
    </ul>
  </nav>
  <footer>
    <p>Mock generated from ALPS profile: {ALPS title}</p>
  </footer>
</body>
</html>
```

### Fidelity-Specific CSS

Common to all levels:
- **Only ALPS semantic IDs in HTML `class` attributes** — never mix in presentation classes
- **Always generate all three CSS files** — only the `<link>` tag in HTML differs
- **Fidelity switching = string-replace `levelN.css` in the `<link>` href** — no HTML structure changes needed

```html
<!-- In HTML <head> — according to selected level -->
<link rel="stylesheet" href="../css/level1.css">
<!-- or level2.css, level3.css -->
```

#### Level 1: Bare HTML (`level1.css`)

Minimal padding and margins for readability. Similar to GitHub Pages default rendering.

```css
/* level1.css — Bare readability */
body {
  max-width: 64rem;
  margin: 0 auto;
  padding: 2rem;
  font-family: system-ui, sans-serif;
  line-height: 1.6;
  color: #333;
}
a { color: #0366d6; }
img { max-width: 100%; }
table { border-collapse: collapse; width: 100%; }
th, td { padding: 0.5rem; border-bottom: 1px solid #ddd; text-align: left; }
button, [type="submit"] { padding: 0.5rem 1rem; cursor: pointer; }
input, textarea, select { padding: 0.25rem 0.5rem; }
```

#### Level 2: Wireframe (`level2.css`)

The information skeleton before visual design begins. Similar to a planning document layout.

```css
/* level2.css — Wireframe */

/* Base layout: grayscale, no decoration */
* { font-family: system-ui, sans-serif !important; }
body { max-width: 72rem; margin: 0 auto; padding: 2rem; color: #333; line-height: 1.5; }

/* Show ALPS ID label on each element */
[class]::before {
  content: "." attr(class);
  display: block;
  font-size: 0.625rem;
  font-family: monospace !important;
  color: #999;
  letter-spacing: 0;
  margin-bottom: 0.125rem;
}

/* Images: X-box + gray background */
img {
  background: #eee;
  background-image:
    linear-gradient(to top right, transparent calc(50% - 1px), #ccc, transparent calc(50% + 1px)),
    linear-gradient(to top left, transparent calc(50% - 1px), #ccc, transparent calc(50% + 1px));
  min-height: 120px;
  min-width: 120px;
}

/* Links and buttons: border only */
a { color: #555; }
button, [type="submit"] {
  border: 1px solid #999;
  background: white;
  padding: 0.5rem 1.5rem;
  color: #333;
  cursor: pointer;
}

/* Form elements */
input, textarea, select {
  border: 1px solid #ccc;
  padding: 0.375rem 0.5rem;
  background: white;
}

/* Visualize block structure with borders */
section, article, aside, form {
  border: 1px dashed #ddd;
  padding: 1rem;
  margin: 0.75rem 0;
}

/* Tables */
table { border-collapse: collapse; width: 100%; }
th, td { padding: 0.5rem; border: 1px solid #ccc; text-align: left; }
```

This wireframe shows:
- Each element displays its ALPS ID label (`.Book`, `.title`, `.doAddToCart`)
- Image positions and sizes shown as X-box placeholders
- `section`, `article`, `form` separated by dashed borders to reveal block structure
- No color or font decoration — **what data goes where** is immediately visible

#### Level 3: Production Quality (`level3.css`)

Plain CSS + CSS Custom Properties in an external file. No CDN dependencies, no build step.

```css
/* level3.css — Production quality */
@import url('https://fonts.googleapis.com/css2?family=...&display=swap');

:root {
  --brand-50: #faf8f5;
  --brand-900: #2c2518;
  --font-display: 'Playfair Display', Georgia, serif;
  --font-sans: 'Inter', system-ui, sans-serif;
  --transition: 300ms ease;
}

/* Apply design using ALPS classes as selectors */
.Book .title {
  font-family: var(--font-display);
  font-size: 1.125rem;
  font-weight: 500;
  transition: color var(--transition);
}
.Book:hover .title { color: var(--brand-600); }
.Book .price { color: var(--brand-700); font-weight: 500; }
.doAddToCart button {
  border: 1px solid var(--brand-900);
  color: var(--brand-900);
  padding: 1rem 2.5rem;
  text-transform: uppercase;
  letter-spacing: 0.15em;
}
.doAddToCart button:hover {
  background: var(--brand-900);
  color: var(--brand-50);
}
```

#### Design Principles

- **Only ALPS semantic classes in HTML** — no presentation classes mixed in
- **Design fully separated into external CSS** — ALPS classes used as CSS selectors
- **No CDN dependencies** — works by opening the file (Google Fonts is the only exception)
- **Fidelity switching = swapping CSS filename** — HTML remains unchanged
- **CSS Custom Properties** for centralized design tokens (colors, fonts, spacing)

#### Design Reference (Level 3)

Infer design taste from the ALPS profile's domain (`title`, `doc`, `tag`, `def`). Apply a clean, modern aesthetic with:

- Design tokens in `:root` (colors, fonts, spacing)
- Google Fonts for display typography
- Responsive breakpoints
- Hover/focus states with transitions
- All selectors reference ALPS classes — no `.btn-primary`, `.card`, `.grid` etc.

### ALPS Class Attributes (§2.3.1)

ALPS descriptor IDs are **always** assigned as HTML element `class` attributes (no need to ask). Applied to states, semantic fields, and transitions.

```html
<section class="Book">
  <h2 class="title">Sample Book Title</h2>
  <span class="author">Author Name</span>
  <span class="isbn">978-4-xxxx-xxxx-x</span>
  <span class="price">¥1,980</span>
</section>
```

- State descriptor → container element's `class` (e.g., `<section class="Book">`)
- Semantic field → data element's `class` (e.g., `<span class="price">`)
- Transition → `<a>`, `<form>`, `<button>` `class` (e.g., `<form class="doAddToCart">`)

### Sample Data Generation

Generate realistic placeholder data appropriate to the domain:
- Infer from `title`, `doc`, and `def` attributes
- Use schema.org type to determine data format
- Examples: book titles for bookstores, task names for todo apps
- Use consistent data across pages (same book appears in catalog and detail)

#### Sample Value Inference

| schema.org def | Sample value |
|---------------|-------------|
| `identifier` | `"BK-001"`, `"USR-042"` |
| `name` | Contextual: book title, user name, etc. |
| `email` | `"user@example.com"` |
| `price` / `totalPrice` | `"1,980"`, `"5,940"` |
| `isbn` | `"978-4-1234-5678-9"` |
| `DateTime` | `"2025-01-15T10:30:00"` |
| `Boolean` | `true` / `false` |
| `quantityValue` | `1`, `2`, `3` |
| `author` | `"Author Name"` |
| `address` | `"123 Main St, City, Country"` |

## JSON API Mock

### Structure

For each state, generate a HAL-format JSON file:

```json
{
  "field1": "sample value",
  "field2": "sample value",
  "_links": {
    "self": { "href": "/api/state-name" },
    "transitionId": {
      "href": "/api/target-state",
      "title": "Transition title"
    }
  }
}
```

### Rules

- Field names match ALPS semantic descriptor IDs
- `_links` keys match ALPS transition descriptor IDs
- Do NOT include `"method"` in links — HTTP method assignment belongs to the OpenAPI level, not HAL links (RFC 8288)

### Collection States

If a state contains a nested state reference (e.g., Catalog contains Book), generate as a collection:

```json
{
  "_embedded": {
    "book": [
      { "id": "BK-001", "title": "Book 1", "price": "1,980" },
      { "id": "BK-002", "title": "Book 2", "price": "2,480" }
    ]
  },
  "_links": {
    "self": { "href": "/api/catalog" },
    "goToBookDetails": { "href": "/api/book/{id}", "title": "Go to Book Details Screen", "templated": true }
  }
}
```

### Sample Data

Use the same sample values as HTML pages for consistency. Infer types and formats from schema.org `def` attributes.

## i18n Label Overrides

When enabled, generate `mock/i18n/labels.json`:

```json
{
  "_comment": "Override ALPS title/doc labels. Keys are descriptor IDs.",
  "title": "書籍タイトル",
  "author": "著者",
  "price": "価格",
  "Home": "ホーム",
  "Catalog": "カタログ",
  "goToCatalog": "カタログへ"
}
```

When this file exists, use its values instead of ALPS `title` for:
- HTML headings, labels, button text, link text
- JSON API `title` fields in `_links`

## Output Directory Structure

```text
mock/
├── mock-switch              # CLI: ./mock-switch <1|2|3> to switch fidelity
├── profile/
│   └── alps.json            # ALPS profile (copied when file input; omitted when URL input)
├── html/
│   ├── index.html          # Entry point (Home state or first state)
│   ├── catalog.html        # Example: Catalog state
│   ├── book.html           # Example: Book state
│   └── ...
├── css/
│   ├── level1.css          # Bare HTML: minimal readability
│   ├── level2.css          # Wireframe: ALPS ID labels, X-box images, structure visualization
│   └── level3.css          # Production quality: full design
├── api/
│   ├── home.json           # HAL JSON for Home state
│   ├── catalog.json        # HAL JSON for Catalog state
│   ├── book.json           # HAL JSON for Book state
│   └── ...
└── i18n/                   # Optional
    └── labels.json
```

## Example

### Input: Bookstore ALPS (partial)

```xml
<alps>
  <title>ALPS Book Store</title>
  <descriptor id="title" def="https://schema.org/name" title="Title"/>
  <descriptor id="price" def="https://schema.org/price" title="Price"/>
  <descriptor id="author" def="https://schema.org/author" title="Author"/>

  <descriptor id="Catalog" title="Book Catalog">
    <descriptor href="#goToBookDetails"/>
    <descriptor href="#goToCart"/>
    <descriptor href="#Book"/>
  </descriptor>

  <descriptor id="Book" title="Book">
    <descriptor href="#title"/>
    <descriptor href="#author"/>
    <descriptor href="#price"/>
    <descriptor href="#doAddToCart"/>
    <descriptor href="#goToCatalog"/>
  </descriptor>

  <descriptor id="goToBookDetails" type="safe" rt="#Book" title="Go to Book Details">
    <descriptor href="#id"/>
  </descriptor>
  <descriptor id="doAddToCart" type="unsafe" rt="#ShoppingCart" title="Add to Cart">
    <descriptor href="#id"/>
    <descriptor href="#quantity"/>
  </descriptor>
</alps>
```

### Output: `html/book.html`

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Book - ALPS Book Store</title>
  <link rel="stylesheet" href="../css/level3.css">
</head>
<body>
  <header>
    <h1 class="Book">Book</h1>
  </header>
  <main>
    <article class="Book">
      <h2 class="title">The Art of Programming</h2>
      <dl>
        <dt>Author</dt>
        <dd class="author">Jane Smith</dd>
        <dt>Price</dt>
        <dd class="price">¥2,480</dd>
      </dl>

      <form method="get" action="shoppingcart.html" class="doAddToCart">
        <input type="hidden" name="id" value="BK-001" class="id">
        <label for="quantity">Quantity</label>
        <input type="number" id="quantity" name="quantity" min="1" value="1" class="quantity">
        <button type="submit">Add to Cart</button>
      </form>
    </article>
  </main>
  <nav>
    <h2>Links</h2>
    <ul>
      <li><a href="catalog.html" class="goToCatalog" title="Navigate to the book catalog list screen. All books are displayed.">Go to Catalog Screen</a></li>
    </ul>
  </nav>
  <footer>
    <p>Mock generated from ALPS profile: ALPS Book Store</p>
  </footer>
</body>
</html>
```

### Output: `api/book.json`

```json
{
  "id": "BK-001",
  "title": "The Art of Programming",
  "author": "Jane Smith",
  "isbn": "978-4-1234-5678-9",
  "price": 2480,
  "category": "Programming",
  "_links": {
    "self": { "href": "/api/book/BK-001" },
    "doAddToCart": {
      "href": "/api/shoppingcart",
      "title": "Add to Cart",
      "title": "Add to Cart"
    },
    "goToCatalog": {
      "href": "/api/catalog",
      "title": "Go to Catalog Screen"
    }
  }
}
```
