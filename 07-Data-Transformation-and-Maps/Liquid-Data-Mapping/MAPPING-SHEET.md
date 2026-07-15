# Liquid Mapping Sheet — Quick Reference (Azure Logic Apps / DotLiquid)

Print this. Keep it open while writing templates.

> **Golden rules:** input is under `content.` · filters are **Sentence-case** (`Upcase` not `upcase`) · tags are lowercase (`for`, `if`) · property names are case-sensitive · missing property = silent empty output · `{%- -%}` trims whitespace.

---

## 1. Syntax skeleton

```liquid
{{ content.field }}                          output a value
{{ content.field | FilterA | FilterB: arg }} filter pipeline (left → right)
{% assign x = content.a %}                   variable
{% capture y %}...multi-line...{% endcapture %}
{% comment %} not rendered {% endcomment %}
{%- ... -%}                                  trim surrounding whitespace
{% raw %}{{ not parsed }}{% endraw %}        literal braces
```

## 2. Control flow tags

| Tag | Syntax |
|---|---|
| if / elsif / else | `{% if a == 'x' %}...{% elsif a == 'y' %}...{% else %}...{% endif %}` |
| unless (inverted if) | `{% unless a == 'x' %}...{% endunless %}` |
| case / when | `{% case a %}{% when 'x' %}...{% when 'y' or 'z' %}...{% else %}...{% endcase %}` |
| for | `{% for item in content.arr %}...{% endfor %}` |
| for (empty fallback) | `{% for item in arr %}...{% else %}EMPTY{% endfor %}` |
| for modifiers | `{% for item in arr limit: 3 offset: 2 reversed %}` |
| range loop | `{% for i in (1..5) %}{{ i }}{% endfor %}` |
| break / continue | `{% break %}` `{% continue %}` (inside for) |
| counters | `{% increment c %}` `{% decrement c %}` (independent of assign vars) |
| XML→JSON loop (Azure only) | `{% JSONArrayFor item in content %}...{% endJSONArrayFor %}` (no trailing comma needed) |

**Operators in conditions:** `==` `!=` `>` `<` `>=` `<=` `and` `or` `contains` · specials: `nil` `empty` `blank` `true` `false`
⚠️ No parentheses; `and`/`or` bind right-to-left — nest `if`s for complex logic.

**`forloop` object:** `index` (1-based) · `index0` · `rindex` · `rindex0` · `first` · `last` · `length`

## 3. Filters by category (DotLiquid sentence-case)

### String
| Filter | Example → result |
|---|---|
| `Upcase` / `Downcase` | `'abc' \| Upcase` → `ABC` |
| `Capitalize` | `'dean' \| Capitalize` → `Dean` |
| `Append:` / `Prepend:` | `'PO' \| Append: '-1'` → `PO-1` |
| `Remove:` / `RemoveFirst:` | `'PO-PO-1' \| Remove: 'PO-'` → `1` |
| `Replace:` / `ReplaceFirst:` | `'a-b' \| Replace: '-', '_'` → `a_b` ⚠️ regex match! |
| `Split:` | `'a, b' \| Split: ', '` → array `[a, b]` |
| `Slice:` | `'(111)222' \| Slice: 1, 3` → `111` (start, length; 0-based) |
| `Truncate:` / `TruncateWords:` | `'abcdefgh' \| Truncate: 5` → `ab...` |
| `Strip` / `Lstrip` / `Rstrip` | trims whitespace |
| `StripNewlines` | removes newline chars |
| `NewlineToBr` | `\n` → `<br />` |
| `UrlEncode` / `UrlDecode` | URL escaping |
| `Escape` | HTML-escape (`&` → `&amp;`) |
| `Size` | `'abcd' \| Size` → `4` (also arrays) |

### Math
| Filter | Example → result |
|---|---|
| `Plus:` `Minus:` `Times:` | `10 \| Times: 9.75` → `97.5` |
| `DividedBy:` | `10 \| DividedBy: 3` → `3` (int ÷ int = int!); `10 \| DividedBy: 3.0` → `3.333…` |
| `Modulo:` | `10 \| Modulo: 3` → `1` |
| `Round` / `Round: n` | `2.567 \| Round: 2` → `2.57` |
| `Floor` / `Ceil` | `2.5 \| Floor` → `2` · `Ceil` → `3` |
| `AtLeast:` / `AtMost:` | clamp: `3 \| AtLeast: 5` → `5` |
| `Abs` | `-3 \| Abs` → `3` |

⚠️ Chained left-to-right; no precedence: `2 | Plus: 3 | Times: 4` = `20`, not `14`.

### Array
| Filter | Example → result |
|---|---|
| `Map: 'prop'` | pluck property from each element (≈ `*.prop`) |
| `Where: 'prop', 'val'` | keep elements where prop == val |
| `Sort: 'prop'` | ⚠️ natural/case-insensitive, string-based (numbers missort!) |
| `Uniq` | dedupe |
| `Join: ','` | array → string |
| `First` / `Last` | element access |
| `Concat: arr2` | append arrays |
| `Reverse` | reverse order |
| `Size` | length |
| `Compact` | remove nils |

### Utility
| Filter | Notes |
|---|---|
| `Default: 'x'` | replaces nil/empty/**false** ⚠️ |
| `Date: 'fmt'` | .NET (`'yyyy-MM-dd HH:mm'`) or strftime; `'now' \| Date: ...` for current time |

**Not available in DotLiquid/Azure:** `json`, `sum`, `group_by`, `sort_natural` (Sort *is* natural), `where` with operators, `base64_encode`. Escaping is manual (`Replace`), sums are loop+`Plus:`, group-by → Function.

## 4. Recipe card (copy-paste idioms)

```liquid
{%- comment %} ARRAY → JSON array, comma-safe {% endcomment -%}
[ {%- for x in content.items %}
  { "id": "{{ x.id }}" }{% unless forloop.last %},{% endunless %}
{%- endfor %} ]

{%- comment %} FILTER + comma-safe (forloop.last unusable!) {% endcomment -%}
[ {%- assign n = 0 -%}
  {%- for x in content.items -%}
    {%- if x.qty > 0 -%}
      {%- if n > 0 %},{% endif %}{ "id": "{{ x.id }}" }
      {%- assign n = n | Plus: 1 -%}
    {%- endif -%}
  {%- endfor %} ]

{%- comment %} SUM {% endcomment -%}
{%- assign total = 0 -%}
{%- for x in content.items -%}
  {%- assign lt = x.qty | Times: x.price -%}
  {%- assign total = total | Plus: lt -%}
{%- endfor -%}

{%- comment %} LOOKUP (small code table) {% endcomment -%}
{%- case x.code -%}
  {%- when 'A' %}{% assign out = '10' %}
  {%- when 'B' %}{% assign out = '20' %}
  {%- else %}{% assign out = '99' %}
{%- endcase -%}

{%- comment %} OPTIONAL JSON BLOCK (comma inside the if) {% endcomment -%}
"id": "{{ content.id }}"
{%- if content.ship %},
"shipping": { "amt": {{ content.ship.amount }} }
{%- endif %}

{%- comment %} CONDITIONAL BOOLEAN (unquoted) {% endcomment -%}
"isBulk": {% if content.qty > 100 %}true{% else %}false{% endif %}

{%- comment %} STRING → ARRAY {% endcomment -%}
{%- assign parts = content.csvField | Split: ',' -%}

{%- comment %} PLUCK + JOIN {% endcomment -%}
"skus": "{{ content.items | Map: 'sku' | Join: ';' }}"

{%- comment %} ESCAPE FREE TEXT FOR JSON (no json filter!) {% endcomment -%}
"note": "{{ content.note | Replace: '\\\\', '\\\\' | Replace: '"', '\"' }}"

{%- comment %} CURRENT TIMESTAMP {% endcomment -%}
"generatedAt": "{{ 'now' | Date: 'yyyy-MM-ddTHH:mm:ssZ' }}"
```

## 5. Gotcha checklist (debug in this order)

```text
[ ] Filter casing — Upcase not upcase (silent failure/empty output)
[ ] content. prefix present on all input paths
[ ] Property-name casing matches payload exactly
[ ] Trailing commas in arrays (check filtered loops especially)
[ ] Numbers/booleans NOT wrapped in quotes when target expects JSON types
[ ] Default: swallowing legitimate false values
[ ] DividedBy integer truncation (use 3.0 not 3)
[ ] Replace regex chars unescaped in match arg ( . ( ) [ ] * + ? \ )
[ ] Sort misordering numbers (string sort!)
[ ] Whitespace garbage in text/CSV output — add {%- -%}
[ ] Missing field rendering as empty string instead of failing (add explicit nil checks)
```

## 6. Where templates go & which action to pick

| Logic App type | Template storage | Actions |
|---|---|---|
| Consumption | Integration Account → Maps (type: Liquid) — IA must be linked to the Logic App | Transform **JSON→JSON / JSON→Text / XML→JSON / XML→Text** |
| Standard | Logic App resource → Artifacts → Maps (or linked IA) | Same four actions (built-in, fast) |

Input goes in the action's **Content** field (usually `body` of a previous action); output is the rendered template (parsed back to JSON for JSON→JSON).
