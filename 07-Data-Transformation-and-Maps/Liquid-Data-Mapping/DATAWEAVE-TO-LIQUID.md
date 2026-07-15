# DataWeave → Liquid Correlation Sheet

Your 7 years of DataWeave thinking, translated construct-by-construct. Three sections: **mindset shift**, the **side-by-side dictionary**, and **what DataWeave does that Liquid can't** (with the Azure escape hatch for each).

---

## 1. The mindset shift (read this first)

| | DataWeave | Liquid |
|---|---|---|
| Philosophy | **Expression language** — the script *is* a transformation that *evaluates to* the output document | **Template language** — you *write the output document* and punch holes in it |
| Direction of thought | "Given input, derive output" (functional) | "Here's the output, splice input in" (presentational) |
| Types | Rich type system, type coercion operators (`as Number`) | Everything is strings/numbers/bools loosely; quote-or-not decides JSON type |
| Null safety | `?`, `default`, pattern matching | Missing values silently render empty; `Default:` filter, explicit `nil` checks |
| Output validity | Guaranteed well-formed (DW builds the structure) | **Your job** — commas, quotes, escaping are manual (the #1 source of bugs) |
| Power ceiling | Recursion, custom functions, modules — Turing-complete transformation | Deliberately limited; heavy logic belongs elsewhere (Functions/XSLT) |
| Where it runs | Inside every Mule event processor | A dedicated Transform action with an uploaded `.liquid` map |

**The one habit to unlearn:** in DW you build *data* and serialization is automatic. In Liquid you build *text* that must happen to parse as JSON. Every recipe idiom (comma guards, quote discipline, escaping) exists because of this.

**The one habit to keep:** mapping-spec thinking. Source→target field tables, null rules, and edge-case samples matter exactly as much as before — only the syntax changed.

---

## 2. Side-by-side dictionary

Input for all examples (same as the folder README):
`{ "orderId": "PO-4512", "customer": {"firstName": "dean"}, "lines": [ {"sku":"A","qty":100,"unitPrice":9.75,"category":"HVAC"}, ... ] }`

### 2.1 Field access & rename

```dataweave
%dw 2.0
output application/json
---
{
  id: payload.orderId,
  buyer: upper(payload.customer.firstName)
}
```

```liquid
{
  "id": "{{ content.orderId }}",
  "buyer": "{{ content.customer.firstName | Upcase }}"
}
```
`payload` → `content`. Navigation is identical dot-style. Quotes on target keys are yours to write.

### 2.2 Null safety & defaults

```dataweave
status: payload.status default "UNKNOWN",
city: payload.address.city  // null-safe: DW returns null
```

```liquid
"status": "{{ content.status | Default: 'UNKNOWN' }}",
"city": "{{ content.address.city }}"   {%- comment %} missing → empty string, NOT null {% endcomment %}
```
⚠️ Differences: DW propagates `null` (and can omit the key with `if`); Liquid renders **empty string** — `"city": ""` is what downstream sees. To omit the key entirely, wrap in `{% if content.address.city %}...{% endif %}` (Scenario 17 in the README). And `Default:` also replaces `false` — DW's `default` doesn't.

### 2.3 map (array projection)

```dataweave
items: payload.lines map {
  sku: $.sku,
  pos: $$ + 1,
  total: $.qty * $.unitPrice
}
```

```liquid
"items": [
  {%- for line in content.lines %}
  {
    "sku": "{{ line.sku }}",
    "pos": {{ forloop.index }},
    "total": {{ line.qty | Times: line.unitPrice }}
  }{% unless forloop.last %},{% endunless %}
  {%- endfor %}
]
```
`$` → named loop variable · `$$` (index) → `forloop.index0` / `forloop.index` · the comma guard has no DW equivalent because DW never needed one.

### 2.4 filter

```dataweave
shippable: payload.lines filter ($.qty > 0)
```

```liquid
{%- comment %} simple property-equality only: {% endcomment -%}
{%- assign hvac = content.lines | Where: 'category', 'HVAC' -%}

{%- comment %} any other predicate → loop + if + printed counter: {% endcomment -%}
"shippable": [
  {%- assign n = 0 -%}
  {%- for line in content.lines -%}
    {%- if line.qty > 0 -%}
      {%- if n > 0 %},{% endif %}{ "sku": "{{ line.sku }}" }
      {%- assign n = n | Plus: 1 -%}
    {%- endif -%}
  {%- endfor %}
]
```
DW's arbitrary-predicate `filter` maps to `Where:` **only for equality**; everything else is the loop idiom — and `forloop.last` breaks (filtered-out last element), hence the counter.

### 2.5 map + filter chain

```dataweave
payload.lines filter ($.qty > 0) map $.sku
```

```liquid
{%- comment %} chains of Where/Map work when predicates are equality: {% endcomment -%}
{{ content.lines | Where: 'category', 'HVAC' | Map: 'sku' | Join: ',' }}
```
Filter pipelines (`|`) are Liquid's version of DW's function chaining — same left-to-right reading.

### 2.6 reduce / sum

```dataweave
total: payload.lines reduce ((line, acc = 0) -> acc + line.qty * line.unitPrice)
// or: sum(payload.lines map ($.qty * $.unitPrice))
```

```liquid
{%- assign total = 0 -%}
{%- for line in content.lines -%}
  {%- assign lt = line.qty | Times: line.unitPrice -%}
  {%- assign total = total | Plus: lt -%}
{%- endfor -%}
"total": {{ total | Round: 2 }}
```
No `reduce`, no `sum` filter in DotLiquid — accumulate manually. `Round:` the result (float drift). Money-critical? → Function with `decimal`.

### 2.7 groupBy ⚠️

```dataweave
byCat: payload.lines groupBy $.category
```

```liquid
{%- comment %} No groupBy. O(n²) workaround: {% endcomment -%}
{%- assign cats = content.lines | Map: 'category' | Uniq -%}
[ {%- for cat in cats %}
  { "category": "{{ cat }}",
    "lines": [ {%- assign n = 0 -%}
      {%- for l in content.lines -%}{%- if l.category == cat -%}
        {%- if n > 0 %},{% endif %}"{{ l.sku }}"{%- assign n = n | Plus: 1 -%}
      {%- endif -%}{%- endfor %} ]
  }{% unless forloop.last %},{% endunless %}
{%- endfor %} ]
```
One-liner in DW, pyramid in Liquid. **This is the canonical "move it to a Function" trigger** — see §3.

### 2.8 orderBy / distinctBy / flatten / joinBy / splitBy

| DataWeave | Liquid | Caveat |
|---|---|---|
| `orderBy $.price` | `\| Sort: 'price'` | ⚠️ string-natural sort — numbers missort (`10` < `9`) |
| `distinctBy $` | `\| Uniq` | values only; `distinctBy $.prop` on objects → Function |
| `flatten` | nested `for` loops (README Scenario 13) | manual comma counter |
| `payload.lines joinBy ","` | `\| Map: 'sku' \| Join: ','` | pluck first, then join |
| `splitBy ","` | `\| Split: ','` | ~identical |
| `payload.lines[0]` | `content.lines[0]` or `\| First` | same |
| `sizeOf(...)` | `\| Size` | same |
| `payload.lines[-1]` | `\| Last` | no negative indexing |

### 2.9 Conditionals & pattern matching

```dataweave
band: if (payload.qty > 100) "BULK" else "STD",
mode: payload.country match {
  case "US" -> "GROUND"
  case "DE" -> "EU-ROAD"
  else -> "AIR"
}
```

```liquid
"band": {% if content.qty > 100 %}"BULK"{% else %}"STD"{% endif %},
"mode": {% case content.country %}
        {% when 'US' %}"GROUND"
        {% when 'DE' %}"EU-ROAD"
        {% else %}"AIR"
        {% endcase %}
```
`if/else` expression → `{% if %}` block · `match/case` → `{% case %}{% when %}` · no pattern-matching on *types/structures* (DW's `matches`, regex cases) — regex lives only inside `Replace:`.

### 2.10 String & date functions

| DataWeave | Liquid (DotLiquid) |
|---|---|
| `upper` / `lower` | `Upcase` / `Downcase` |
| `capitalize` | `Capitalize` |
| `trim` | `Strip` |
| `s ++ t` (concat) | `{{ s }}{{ t }}` or `\| Append:` |
| `s[0 to 2]` | `\| Slice: 0, 3` |
| `replace ... with ...` | `\| Replace: 'x', 'y'` ⚠️ regex |
| `contains(s, "x")` | `{% if s contains 'x' %}` |
| `now()` | `'now' \| Date: 'fmt'` |
| `payload.date as Date {format: "yyyyMMdd"}` | `\| Date: 'yyyyMMdd'` (format only!) |
| `now() + \|P7D\|` (date math) | ❌ none — pre-compute with `addDays()` workflow expression |
| `as Number` / `as String` coercion | quote-or-not in the template; `Plus: 0` trick to force numeric |

### 2.11 Variables & reusable logic

```dataweave
var discounted = (p) -> p * 0.9    // lambda
var taxRate = 0.18                  // constant
```

```liquid
{%- assign taxRate = 0.18 -%}
{%- capture header -%}...reusable fragment...{%- endcapture -%}
{%- comment %} lambdas/functions: ❌ none. Repeated logic = repeated template code {% endcomment -%}
```
No user-defined functions, no modules, no imports. Reuse = `capture` fragments within one template, or split into multiple Transform actions.

---

## 3. What DataWeave does that Liquid CAN'T — and the Azure answer

| DataWeave capability | Liquid status | Azure escape hatch |
|---|---|---|
| `groupBy` + per-group aggregation | O(n²) hack | **Function + LINQ `GroupBy`** ⭐ |
| `reduce` with complex accumulators | manual, floats drift | Function (`Aggregate`, `decimal`) |
| Custom functions / recursion / modules | ❌ | Function (real C# methods, NuGet) |
| `distinctBy $.prop` on objects | ❌ (`Uniq` = whole values) | LINQ `DistinctBy` |
| Numeric-correct `orderBy` | ❌ (string sort) | LINQ `OrderBy` |
| Date arithmetic | ❌ | workflow expressions (`addDays`) pre-transform, or Function |
| Type coercion & validation (`as Number`, fail on bad data) | ❌ silent | Parse JSON schema validation before; Function for strictness |
| Multi-input transforms (payload + vars + lookups merged) | single `content` input only | Compose a combined object in the workflow *first*, then transform |
| Streaming huge payloads | ❌ (in-memory template) | Functions/ADF for large data |
| Automatic well-formed output | ❌ manual commas/escaping | discipline + golden-file tests; or serialize in code |
| Reading `attributes`, MEL context | n/a | pass what you need into the Compose that feeds `content` |

**The composite pattern you'll actually use** (multi-input example):

```text
Mule:   Transform Message (payload + vars.customer + lookup) — one DW script
Azure:  1) SQL lookup action
        2) Compose: { "order": @{triggerBody()}, "customer": @{body('Get_customer')} }
        3) Transform JSON→JSON (Liquid) on the composed object → content.order.*, content.customer.*
```

---

## 4. Porting workflow: how to convert an existing DW script

1. **Extract the mapping spec** from the DW script (source path → target path → rule) into a table — don't port line-by-line.
2. **Classify each rule:** rename/format (→ Liquid direct), small conditional/lookup (→ Liquid `if`/`case`), group/reduce/precision/dates-math (→ mark for Function or pre-compute).
3. If >30% of rules are marked for code → **write the whole map as a Function** (don't split one logical map across two engines).
4. Write the target document skeleton first, then fill holes — template-first, the Liquid way.
5. Test with your old MUnit test data as golden files (inputs + expected outputs you already trust!).
6. Watch for the classic port bugs: quote-wrapped numbers, `Default:` eating `false`, comma guards in filtered loops, case-sensitive property names, sentence-case filters.

**Practice porting exercise:** take your 3 most complex production DataWeave scripts and run them through this workflow. Whatever step 2 taught you about each script *is* your interview answer to "how would you migrate MuleSoft transformations to Azure?"
