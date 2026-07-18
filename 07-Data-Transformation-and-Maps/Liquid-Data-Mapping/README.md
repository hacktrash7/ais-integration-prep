# Liquid Data Mapping — The Complete Guide (Shopify Liquid → Azure Logic Apps)

**Purpose of this folder:** everything you need to become fluent in Liquid-based data mapping for Azure Logic Apps — every mapping scenario you'll meet in real integration work, with working templates you can copy.

**Files in this folder:**
| File | Use it for |
|---|---|
| `README.md` (this file) | Learn Liquid mapping scenario-by-scenario |
| [`MAPPING-SHEET.md`](./MAPPING-SHEET.md) | Quick-reference while writing templates (tags, filters, recipes) |
| [`DATAWEAVE-TO-LIQUID.md`](./DATAWEAVE-TO-LIQUID.md) | Translate your DataWeave thinking into Liquid, and know when Liquid can't do it |

---

## 1. Liquid in 5 minutes — the three building blocks

Liquid (created by Shopify) has exactly **three constructs**. That's the whole language:

| Construct | Syntax | Purpose | Example |
|---|---|---|---|
| **Objects** (output) | `{{ ... }}` | Print a value into the output | `{{ content.firstName }}` |
| **Tags** (logic) | `{% ... %}` | Control flow, loops, variables — produce *no* output themselves | `{% if x > 5 %} ... {% endif %}` |
| **Filters** (transform) | `\| filterName: args` | Transform a value, left-to-right pipeline | `{{ name \| Upcase \| Prepend: 'Mr. ' }}` |

A Liquid **template is the output document with holes in it** — the opposite mindset from DataWeave, where the script *is* the transformation and produces the document. You write the target JSON/text/XML shape literally, and inject/loop/branch inside it.

📖 [Shopify Liquid docs](https://shopify.github.io/liquid/) · [Logic Apps Liquid transform](https://learn.microsoft.com/en-us/azure/logic-apps/logic-apps-enterprise-integration-liquid-transform)

---

## 2. Critical: Azure runs **DotLiquid**, not Shopify Liquid

Azure Logic Apps uses **DotLiquid 2.0.361** (a .NET port). Five differences that *will* bite you if you learn only from Shopify docs:

1. **Filter names are sentence-cased:** `Upcase`, `Replace`, `Split`, `Append` — **not** `upcase`, `replace`. (Tags stay lowercase: `for`, `if`, `assign`.)
2. **Your input lives under `content`:** the payload you pass into the Transform action is wrapped — access everything as `content.xxx`.
3. **`Replace` uses regex matching** (Shopify's is plain string match) — escape regex-reserved characters: `| Replace: '\\(', '['` to replace a literal `(`.
4. **No `json` filter** (Shopify extension) — escape reserved JSON characters manually with `Replace`.
5. **`Sort` behaves like Shopify's `sort_natural`** (case-insensitive, string-alphanumeric — so `10` sorts before `9`!).

Also: Liquid is **case-sensitive** for property names (`content.firstName` ≠ `content.FirstName`), and a missing property renders as **empty** (no error) — silent blanks are the #1 Liquid debugging pain.

> **Where templates live:** Consumption → upload `.liquid` file to the **Integration Account** (Maps, type = Liquid); Standard → upload under the Logic App resource's **Artifacts → Maps** (or a linked Integration Account). Then use the built-in actions: *Transform JSON to JSON / JSON to text / XML to JSON / XML to text*.

---

## 3. Scenario catalog — every mapping pattern

For all scenarios below, assume this input to the **Transform JSON to JSON** action:

```json
{
  "orderId": "PO-4512",
  "orderDate": "2026-07-14T10:15:00Z",
  "status": "OPEN",
  "customer": { "firstName": "dean", "lastName": "ledet", "type": "wholesale", "email": "DEAN@EXAMPLE.COM" },
  "devices": "Surface, Mobile, Desktop",
  "shipping": { "amount": 25.5, "country": "US" },
  "lines": [
    { "sku": "SKU-8871", "desc": "Damper actuator", "qty": 100, "unitPrice": 9.75, "category": "HVAC" },
    { "sku": "SKU-2231", "desc": "Valve",           "qty": 0,   "unitPrice": 4.20, "category": "HVAC" },
    { "sku": "SKU-9910", "desc": "Sensor",          "qty": 5,   "unitPrice": 55.0, "category": "CTRL" }
  ]
}
```

### Scenario 1 — Rename & restructure fields (the bread and butter)

```liquid
{
  "id": "{{ content.orderId }}",
  "buyer": {
    "name": "{{ content.customer.firstName | Capitalize }} {{ content.customer.lastName | Capitalize }}",
    "contact": "{{ content.customer.email | Downcase }}"
  },
  "shipTo": "{{ content.shipping.country }}"
}
```
Rules: navigate with dots, restructure by *writing the new structure*, transform inline with filters.

### Scenario 2 — String manipulation

```liquid
{
  "upper": "{{ content.customer.firstName | Upcase }}",
  "trimmedDesc": "{{ content.lines[0].desc | Strip }}",
  "prefixed": "{{ content.orderId | Prepend: 'ORD/' }}",
  "suffixed": "{{ content.orderId | Append: '/2026' }}",
  "numericPart": "{{ content.orderId | Remove: 'PO-' }}",
  "masked": "{{ content.customer.email | Slice: 0, 3 }}***",
  "swapped": "{{ content.status | Replace: 'OPEN', 'NEW' }}",
  "truncated": "{{ content.lines[0].desc | Truncate: 10 }}",
  "sized": {{ content.orderId | Size }}
}
```
`Slice: start, length` (0-based). `Truncate` includes the `...` in the length. `Size` works on strings *and* arrays.

### Scenario 3 — Numbers & math

```liquid
{%- assign line = content.lines[0] -%}
{
  "lineTotal": {{ line.qty | Times: line.unitPrice }},
  "withShipping": {{ line.qty | Times: line.unitPrice | Plus: content.shipping.amount }},
  "discounted": {{ line.unitPrice | Minus: 0.75 }},
  "half": {{ line.qty | DividedBy: 2 }},
  "remainder": {{ line.qty | Modulo: 3 }},
  "rounded": {{ content.shipping.amount | Round }},
  "rounded2dp": {{ line.unitPrice | Round: 1 }},
  "floorCeil": [ {{ 25.5 | Floor }}, {{ 25.5 | Ceil }} ],
  "atLeast": {{ line.qty | AtLeast: 1 }},
  "atMost": {{ line.qty | AtMost: 50 }}
}
```
⚠️ Note: math filters chain **left-to-right** — there is no operator precedence and no `( )` math inside `{{ }}`. For anything beyond chained arithmetic (percentages with conditions, running totals with decimals), consider a Function instead.

### Scenario 4 — Dates

```liquid
{
  "isoDate": "{{ content.orderDate | Date: 'yyyy-MM-dd' }}",
  "human": "{{ content.orderDate | Date: 'dd MMM yyyy HH:mm' }}",
  "yearOnly": "{{ content.orderDate | Date: 'yyyy' }}",
  "generatedAt": "{{ 'now' | Date: 'yyyy-MM-ddTHH:mm:ssZ' }}"
}
```
DotLiquid `Date:` accepts **.NET format strings** (`yyyy-MM-dd`) *and* Ruby strftime (`%Y-%m-%d`) — prefer .NET format in Azure. There is **no date arithmetic** (add days etc.) in Liquid — do that with `addDays()` in a workflow expression *before* the transform, or in a Function.

### Scenario 5 — Conditionals, defaults & null handling

```liquid
{
  "status": "{{ content.status | Default: 'UNKNOWN' }}",
  {%- if content.customer.type == 'wholesale' %}
  "priceBook": "TRADE",
  {%- else %}
  "priceBook": "RETAIL",
  {%- endif %}
  "highValue": {% if content.shipping.amount > 100 %}true{% else %}false{% endif %},
  "shipMode": {% case content.shipping.country %}
              {% when 'US' %}"GROUND"
              {% when 'DE' or 'FR' %}"EU-ROAD"
              {% else %}"AIR"
              {% endcase %},
  {%- unless content.lines == empty %}
  "hasLines": true,
  {%- endunless %}
  "notes": "{{ content.notes | Default: '' }}"
}
```
Operators: `==  !=  >  <  >=  <=  and  or  contains`. Special values: `empty`, `blank`, `nil` (`{% if content.notes == nil %}`). **No parentheses in conditions** and `and`/`or` evaluate **right-to-left** — split complex conditions into nested `if`s. `Default:` kicks in for nil/empty/false — beware: a legitimate `false` gets replaced too!

### Scenario 6 — Arrays: iterate & map (your DataWeave `map`)

```liquid
{
  "items": [
    {%- for line in content.lines %}
    {
      "sku": "{{ line.sku }}",
      "position": {{ forloop.index }},
      "qty": {{ line.qty }},
      "total": {{ line.qty | Times: line.unitPrice }}
    }{% unless forloop.last %},{% endunless %}
    {%- endfor %}
  ]
}
```
**The trailing-comma idiom** `{% unless forloop.last %},{% endunless %}` is mandatory JSON hygiene — memorize it. `forloop` gives you: `index` (1-based), `index0`, `rindex`, `first`, `last`, `length`.

Loop extras: `{% for line in content.lines limit: 2 offset: 1 reversed %}`, and `{% else %}` inside `for` runs when the array is empty:

```liquid
"items": [
  {%- for line in content.lines %} ... {% else %} {% endfor %}
]
```

### Scenario 7 — Arrays: filter (your DataWeave `filter`)

Two ways — the `Where` filter (property equality only) or loop + `if`:

```liquid
{%- assign hvacLines = content.lines | Where: 'category', 'HVAC' -%}
"hvacCount": {{ hvacLines | Size }},

"shippable": [
  {%- assign printed = 0 -%}
  {%- for line in content.lines -%}
    {%- if line.qty > 0 -%}
      {%- if printed > 0 %},{% endif -%}
      { "sku": "{{ line.sku }}", "qty": {{ line.qty }} }
      {%- assign printed = printed | Plus: 1 -%}
    {%- endif -%}
  {%- endfor %}
]
```
⚠️ Note the comma pattern changes when filtering inside a loop: `forloop.last` no longer works (the last item might be filtered out!), so count printed items instead — a classic Liquid trap.

### Scenario 8 — Arrays: aggregate (sum / min / max — your `reduce`)

No `reduce` in Liquid; accumulate with `assign` in a loop:

```liquid
{%- assign orderTotal = 0 -%}
{%- assign maxLine = 0 -%}
{%- for line in content.lines -%}
  {%- assign lineTotal = line.qty | Times: line.unitPrice -%}
  {%- assign orderTotal = orderTotal | Plus: lineTotal -%}
  {%- if lineTotal > maxLine %}{% assign maxLine = lineTotal %}{% endif -%}
{%- endfor -%}
{
  "orderTotal": {{ orderTotal }},
  "biggestLine": {{ maxLine }},
  "lineCount": {{ content.lines | Size }}
}
```
⚠️ Floating-point accumulation in Liquid can produce ugly precision artifacts — `Round:` the final result, or use a Function for money math.

### Scenario 9 — Arrays: extract, sort, join, first/last, dedupe

```liquid
{%- assign skus = content.lines | Map: 'sku' -%}
{
  "allSkus": "{{ skus | Join: ';' }}",
  "firstSku": "{{ skus | First }}",
  "lastSku": "{{ skus | Last }}",
  "sortedByPrice": {{ content.lines | Sort: 'unitPrice' | Map: 'sku' | Join: ', ' | Json }},
  "categories": "{{ content.lines | Map: 'category' | Uniq | Join: ',' }}"
}
```
`Map: 'prop'` plucks one property from every element (≈ `payload.lines.*sku`). `Uniq` dedupes. Remember DotLiquid `Sort` is case-insensitive string sort — numeric sort of `[9, 10]` gives `10, 9`!

*(If `Json` / `Uniq` misbehave in your runtime version, fall back to `Join` + manual formatting — always test filters in a live workflow, DotLiquid's filter set is narrower than Shopify's.)*

### Scenario 10 — Split a delimited string into an array

```liquid
{%- assign deviceList = content.devices | Split: ', ' -%}
{
  "devices": [
    {%- for d in deviceList %}
    "{{ d }}"{% unless forloop.last %},{% endunless %}
    {%- endfor %}
  ]
}
```
This is the exact example from the Microsoft docs — CSV-in-a-field is everywhere in legacy payloads.

### Scenario 11 — Lookup tables / value mapping (code → code)

The idiom: parallel `Split` arrays or a `case` block.

```liquid
{%- comment %} Map internal category codes to SAP division codes {% endcomment -%}
{%- case content.lines[0].category -%}
  {%- when 'HVAC' %}{% assign division = '10' %}
  {%- when 'CTRL' %}{% assign division = '20' %}
  {%- else %}{% assign division = '99' %}
{%- endcase -%}
{ "division": "{{ division }}" }
```
For big lookup tables (50+ codes): don't do it in Liquid — enrich in the workflow *before* the transform (SQL/Blob lookup) or map in a Function. The `case` idiom is for small stable code sets.

### Scenario 12 — Nested arrays (orders → lines → serials)

Nested `for` loops work fine; the comma idiom applies at *each* level:

```liquid
"orders": [
  {%- for order in content.orders %}
  {
    "id": "{{ order.id }}",
    "lines": [
      {%- for line in order.lines %}
      { "sku": "{{ line.sku }}" }{% unless forloop.last %},{% endunless %}
      {%- endfor %}
    ]
  }{% unless forloop.last %},{% endunless %}
  {%- endfor %}
]
```
Inside nested loops, `forloop` refers to the **innermost** loop; use `{% assign outerLast = forloop.last %}` before entering the inner loop if you need the outer state.

### Scenario 13 — Flatten (all lines from all orders into one array)

```liquid
"allLines": [
  {%- assign printed = 0 -%}
  {%- for order in content.orders -%}
    {%- for line in order.lines -%}
      {%- if printed > 0 %},{% endif -%}
      { "orderId": "{{ order.id }}", "sku": "{{ line.sku }}" }
      {%- assign printed = printed | Plus: 1 -%}
    {%- endfor -%}
  {%- endfor %}
]
```
(Printed-counter comma pattern again — `forloop.last` can't see across two loops.)

### Scenario 14 — Group-by ⚠️ (Liquid's hard limit)

Liquid has **no `groupBy`**. The workaround (nested loop over `Uniq` keys) is O(n²) and ugly:

```liquid
{%- assign cats = content.lines | Map: 'category' | Uniq -%}
"byCategory": [
  {%- for cat in cats %}
  {
    "category": "{{ cat }}",
    "skus": [
      {%- assign printed = 0 -%}
      {%- for line in content.lines -%}
        {%- if line.category == cat -%}
          {%- if printed > 0 %},{% endif %}"{{ line.sku }}"
          {%- assign printed = printed | Plus: 1 -%}
        {%- endif -%}
      {%- endfor %}
    ]
  }{% unless forloop.last %},{% endunless %}
  {%- endfor %}
]
```
**Rule of thumb:** one group-by = acceptable in Liquid; group-by + aggregation per group, or grouping large arrays → **Azure Function with LINQ `GroupBy`** (see `DATAWEAVE-TO-LIQUID.md` §3).

### Scenario 15 — JSON → text / CSV (Transform JSON to Text action)

```liquid
OrderId,Sku,Qty,LineTotal
{%- for line in content.lines %}
{{ content.orderId }},{{ line.sku }},{{ line.qty }},{{ line.qty | Times: line.unitPrice }}
{%- endfor %}
```
Output is plain text — perfect for CSV files, fixed-width (pad with `Slice`/`Append` tricks), email bodies, or EDI-adjacent text. For fixed-width at scale use a Function with `PadLeft/PadRight`.

### Scenario 16 — XML → JSON (the `JSONArrayFor` custom tag)

Azure adds a custom loop tag for XML input that **auto-handles trailing commas**:

```liquid
[{% JSONArrayFor item in content -%}
    { "name": "{{ item.name }}", "value": "{{ item.value }}" }
{% endJSONArrayFor -%}]
```
Notes: with XML input you navigate elements by name (`content.Order.Lines`); the `where` condition of `JSONArrayFor` compares **element names**, not values. Attributes are awkward in Liquid — if the XML is attribute-heavy, prefer XSLT (Module 07 §4).

### Scenario 17 — Conditional *sections* (omit a whole block)

JSON keys can't be "conditionally absent" via filters — wrap the whole fragment:

```liquid
{
  "id": "{{ content.orderId }}"
  {%- if content.shipping %},
  "shipping": { "amount": {{ content.shipping.amount }}, "country": "{{ content.shipping.country }}" }
  {%- endif %}
}
```
Put the comma *inside* the conditional block (leading-comma style) so the JSON stays valid in both branches.

### Scenario 18 — Reusable fragments: `capture`, `assign`, `increment`

```liquid
{%- capture fullName -%}{{ content.customer.firstName | Capitalize }} {{ content.customer.lastName | Capitalize }}{%- endcapture -%}
{
  "billTo": "{{ fullName }}",
  "shipTo": "{{ fullName }}",
  "seq1": {% increment counter %},
  "seq2": {% increment counter %}
}
```
`capture` = multi-line string assign (build a fragment once, reuse). `increment`/`decrement` = standalone counters (start at 0; independent from `assign` variables!).

### Scenario 19 — Whitespace control (`{%-` and `-%}`)

Every tag/output can trim adjacent whitespace: `{%- ... -%}` and `{{- ... -}}`. Without trimming, loops leave blank lines and stray spaces in your JSON/text output. Habit: **trim on all logic tags** (`{%- for -%}`, `{%- if -%}`, `{%- assign -%}`), keep output tags natural. For JSON targets the parser mostly forgives whitespace; for CSV/fixed-width it does not.

### Scenario 20 — Escaping & special characters

```liquid
{ "note": "{{ content.note | Replace: '\\\\', '\\\\' | Replace: '"', '\"' }}" }
```
Because there's no `json` filter in DotLiquid, escape backslashes and double quotes yourself when the source text may contain them (and remember `Replace` is regex-based — `\\` patterns, `.` `(` `)` `[` `]` `*` `+` `?` need escaping in the *match* argument). If a field can contain arbitrary free text, this is a strong signal to transform in a Function instead.

---

## 4. When Liquid is the WRONG tool (decision guardrail)

Bail out to an **Azure Function (LINQ)** or **XSLT** when you see:
1. **Group-by + per-group aggregation** (Scenario 14) — LINQ `GroupBy` is one line.
2. **Money math with precision requirements** — Liquid floats drift; C# `decimal` doesn't.
3. **Big lookup/cross-reference tables** — enrich before, or map in code.
4. **Date arithmetic** — pre-compute with workflow expressions or code.
5. **Free-text fields needing JSON escaping** — code serializers handle it safely.
6. **Attribute-heavy or namespace-heavy XML** — XSLT's home turf.
7. **Templates > ~150 lines or 3+ levels of nested loops** — unreadable, untestable; refactor.
8. **Anything needing unit tests** — Liquid templates can only be golden-file tested through a workflow run; C# maps get real xUnit tests.

Interview-ready sentence: *"Liquid is my template-shaped tool — ideal when the target document dominates and logic is light; the moment grouping, precision math, or heavy escaping appears, I move the map to a Function with LINQ, which is also unit-testable."*

---

## 5. Labs

1. **Kata set:** implement Scenarios 1, 5, 6, 7, 8 against the sample order (one Logic App, one Transform action, swap maps). Verify with Postman.
2. **The comma drills:** break Scenario 7 on purpose (use `forloop.last` while filtering) and watch invalid JSON appear; fix with the printed-counter idiom.
3. **CSV export:** Scenario 15 → write the output to Blob as `orders.csv`.
4. **XML round-trip:** take an XML order, do XML→JSON with `JSONArrayFor`, compare with the XSLT approach from Module 07 lab 2 — write down which felt better and why.
5. **The bail-out drill:** implement Scenario 14 (group-by) in Liquid, then re-implement in a Function with `GroupBy().Select()` — compare line count, readability, testability. This comparison is interview gold.
6. **Regression harness:** save input + expected output pairs for 3 of your templates; build a small Logic App that runs all 3 transforms and compares against expected (golden-file testing, Module 12).

---

## 6. Resources

- 📖 [Shopify Liquid reference](https://shopify.github.io/liquid/) — the language spec (remember: lowercase filters there, sentence-case in Azure!)
- 📖 [Logic Apps Liquid transforms + DotLiquid differences](https://learn.microsoft.com/en-us/azure/logic-apps/logic-apps-enterprise-integration-liquid-transform) ⭐ the canonical Azure page
- 📖 [DotLiquid GitHub](https://github.com/dotliquid/dotliquid) · [DotLiquid online tester](http://dotliquidmarkup.org/try-online) — test templates without deploying (use sentence-case filters!)
- 📖 [Shopify Cheat Sheet](https://www.shopify.com/partners/shopify-cheat-sheet) — visual filter/tag reference
- 🎥 Search **"Logic Apps Liquid transform tutorial"**, **"Liquid template JSON transformation Azure"**
- 📖 Sandro Pereira's Liquid posts: [blog.sandro-pereira.com](https://blog.sandro-pereira.com/) — search "Liquid" for dozens of real-world mapping posts

---

## 7. Interview questions on Liquid mapping

1. What are Liquid's three building blocks? How does a Liquid template differ philosophically from a DataWeave script? *(document-with-holes vs expression-that-produces-document)*
2. What implementation does Logic Apps use, and name three differences from Shopify Liquid. *(DotLiquid: sentence-case filters, regex Replace, no json filter, Sort = natural/case-insensitive)*
3. How do you access the input payload in a Logic Apps Liquid template? *(`content` root)*
4. How do you avoid trailing commas when generating JSON arrays? What changes when you filter inside the loop? *(unless forloop.last / printed counter)*
5. What is `JSONArrayFor` and when do you need it?
6. How do you handle missing/null fields? *(Default filter, nil checks, conditional sections — and the false-value gotcha with Default)*
7. Can Liquid do group-by? What's your threshold for moving a map to an Azure Function?
8. How do you do date formatting and why can't you do date arithmetic in Liquid?
9. Where are Liquid templates stored for Consumption vs Standard Logic Apps?
10. How do you test Liquid templates? *(DotLiquid online for drafts; golden-file regression through a workflow; keep templates small)*
