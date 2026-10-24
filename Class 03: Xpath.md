# XPath Axes – Revision & Interview Notes (Playwright)

---

## 1. What is an XPath Axis?

- An **XPath axis** lets you locate an element **using a neighbouring element** (parent, child, sibling, ancestor, etc.) when the target has **no usable attributes** (no id, class, name…).
- Flow: **write XPath for a nearby element → navigate to the target using an axis**.
- You never write the XPath directly for the target element.
- Axes let you traverse the DOM in **any direction** (top→bottom *and* bottom→top).

### Syntax

```
//<starting-element-xpath>/<axis>::<target-tag>[optional attributes]
```

- Keyword followed by **double colon `::`**
- After `::` give the **tag name of the target** (or `*` for all tags)
- Attributes (if any) go inside `[ ]`

---

## 2. Axes at a Glance

| Axis | Returns | Direction |
|---|---|---|
| `self` | The node itself | – |
| `parent` | Immediate parent only | Up |
| `child` | Immediate children only | Down |
| `ancestor` | Parent + grandparent + all above | Up |
| `descendant` | Children + grandchildren + all below | Down |
| `following` | All nodes after the current one in document order (siblings **+ their children**), excluding own descendants | Forward |
| `following-sibling` | Only siblings **after** the node (same parent), **no children** | Forward |
| `preceding` | All nodes before the current one (siblings **+ their children**) | Backward |
| `preceding-sibling` | Only siblings **before** the node (same parent), **no children** | Backward |

### Mental model (the B1/B2/B3 tree from the class)

```
body
└── A
    ├── B1 (X1, X2, X3)
    ├── B2 (Y1, Y2 [M1,M2..], Y3)   ← self
    ├── B3 (Z1, Z2)
    └── B4
```

- Parent of Y1/Y2/Y3 → **B2**
- Child of B2 → **Y1, Y2, Y3**
- Ancestor of B2 → **A, body**
- Descendant of B2 → **Y1, Y2, Y3 + their children (grandchildren)**
- Following-sibling of B2 → **B3, B4**
- Following of B2 → **B3, B4 + Z1, Z2 + children of B4**
- Preceding-sibling of B2 → **B1**
- Preceding of B2 → **B1 + X1, X2, X3**

---

## 3. Each Axis in Detail

### 3.1 `self`
- The node itself.
- **Optional** – if XPath already locates the element, `self` is unnecessary.
- Syntax: `//td[text()='Germany']/self::td`

### 3.2 `parent`
- Returns **only the immediate parent**.
- Example: parent of the `Germany` cell is the whole `<tr>` (row).
- Syntax: `//td[text()='Germany']/parent::tr`
- Note: `/..` also gives the parent, but `parent::` is the axis form.

### 3.3 `child`
- Returns **direct children only**.
- Example: all `<td>` of the 2nd row.
- Syntax: `//table[@id='customers']//tr[2]/child::td` → **3 elements**

### 3.4 `ancestor`
- Parent + grandparent + … up to root. **Bottom → top** approach.
- `ancestor::*` → all ancestors (12 in the demo); `ancestor::table` → only the table.
- Syntax: `//td[text()='Germany']/ancestor::table`

### 3.5 `descendant`
- Children + grandchildren + all levels below. Opposite of ancestor.
- `descendant::*` → 29 elements; `descendant::td` → **18 cells** in the demo table.
- Syntax: `//table[@id='customers']/descendant::td`

### 3.6 `following` vs `following-sibling`
- `following::td` → all TDs after Germany in doc order (**35** in the demo), includes TDs in later rows.
- `following-sibling::td` → only TDs in the **same row** after the node.
- Germany is the last cell in its row → `following-sibling::td` = **0 elements**.
- Getting just the next cell: `//td[text()='Germany']/following::td[1]`

### 3.7 `preceding` vs `preceding-sibling`
- `preceding::td[1]` → the TD just before Germany in document order.
- `preceding-sibling::td` → TDs to the **left** in the same row (**2** for Germany).
- Single sibling: use index `[1]` or Playwright `.nth()`.

---

## 4. KEY DIFFERENCE: `following` vs `following-sibling` (Interview favourite)

| | `following` | `following-sibling` |
|---|---|---|
| Scope | Everything after the node in the DOM | Only nodes with the **same parent** |
| Includes children of siblings? | **Yes** | **No** |
| Includes later rows/branches? | Yes | No |

The same applies to `preceding` vs `preceding-sibling`.

**Ancestor vs Descendant:** ancestor = parent & above (upward); descendant = children & below (downward).

**Parent vs Ancestor:** parent = only one level up; ancestor = all levels up.
**Child vs Descendant:** child = one level down; descendant = all levels down.

---

## 5. Playwright Code Examples (TypeScript)

Test page: W3Schools HTML Tables (`table#customers`).

```ts
import { test, expect } from '@playwright/test';

test('xpath axes demo', async ({ page }) => {
  await page.goto('https://www.w3schools.com/html/html_tables.asp');

  // SELF
  const germanyCell = page.locator("//td[text()='Germany']/self::td");
  await expect(germanyCell).toHaveText('Germany');

  // PARENT -> whole row
  const parentRow = page.locator("//td[text()='Germany']/parent::tr");
  await expect(parentRow).toContainText('Maria Anders');
  console.log(await parentRow.textContent()); // all 3 cells as one string

  // CHILD -> all tds of 2nd row
  const secondRowCells = page.locator("//table[@id='customers']//tr[2]/child::td");
  await expect(secondRowCells).toHaveCount(3);

  // ANCESTOR -> the table
  const table = page.locator("//td[text()='Germany']/ancestor::table");
  await expect(table).toHaveAttribute('id', 'customers');

  // DESCENDANT -> all tds in table
  const allTDs = page.locator("//table[@id='customers']/descendant::td");
  await expect(allTDs).toHaveCount(18);

  // FOLLOWING -> next td after Germany (document order)
  const followingCell = page.locator("//td[text()='Germany']/following::td[1]");
  await expect(followingCell).toHaveText('<expected text>');

  // FOLLOWING-SIBLING -> none for Germany (last cell in row)
  const rightSiblings = page.locator("//td[text()='Germany']/following-sibling::td");
  await expect(rightSiblings).toHaveCount(0);

  // PRECEDING -> td just before Germany
  const precedingCell = page.locator("//td[text()='Germany']/preceding::td[1]");
  await expect(precedingCell).toHaveText('<expected text>');

  // PRECEDING-SIBLING -> tds to the left in the same row
  const leftSiblings = page.locator("//td[text()='Germany']/preceding-sibling::td");
  await expect(leftSiblings).toHaveCount(2);
  await expect(leftSiblings.nth(0)).toHaveText('Alfreds Futterkiste');
  await expect(leftSiblings.nth(1)).toHaveText('Maria Anders');
});
```

---

## 6. Playwright Notes & Gotchas

- `page.locator()` is **not async** – no `await` needed; it only creates a locator (type `Locator`).
- Locator returns the **same type** whether it matches 1, many or 0 elements – there is no separate method for multiple elements. The XPath decides.
- **Zero matches** → no exception on locator creation; count is simply 0 (assert with `toHaveCount(0)`).
- Assertions are **async** → always use `await expect(...)`: `toHaveText`, `toContainText`, `toHaveCount`, `toHaveAttribute`, `toBeVisible`.
- `toHaveText` = exact text; `toContainText` = substring (also accepts regex / arrays).
- Extracting text:
  - Single element → `textContent()`
  - Multiple elements → `allTextContents()`
- Picking from multiple matches: `.first()`, `.last()`, `.nth(index)` (**0-based**) – whereas XPath index `[1]` is **1-based**.
- `locator.toString/console.log(locator)` prints the locator, not text – extract text first.
- Remember to `import { Locator }` if you annotate types.
- The demo page was slow/flaky (timeouts at 30s) – rerun before assuming a bug.

---

## 7. Useful Syntax Cheat-Sheet

```
//td[text()='Germany']/self::td
//td[text()='Germany']/parent::tr
//tr[2]/child::td
//td[text()='Germany']/ancestor::table
//td[text()='Germany']/ancestor::*             # all ancestors
//table[@id='customers']/descendant::td
//table[@id='customers']/descendant::*         # all descendants
//td[text()='Germany']/following::td
//td[text()='Germany']/following::td[1]        # immediate next in doc order
//td[text()='Maria Anders']/following-sibling::td
//td[text()='Germany']/preceding::td[1]
//td[text()='Germany']/preceding-sibling::td
```

Using `*` instead of a tag name returns all matching nodes of that axis.

---

## 8. When to Use / Not Use

**Use when:**
- Element has **no unique attributes**
- Page is complex / dynamic
- You need to move **upward** (parent/ancestor) – CSS can't do that
- Tables: find a cell by its row/column neighbour

**Avoid when:**
- Direct attributes exist (id, name, data-testid, role, etc.)
- Performance matters – XPath is **slower than CSS**, and axes make it **slower still**
- Prefer Playwright's built-in locators (`getByRole`, `getByText`, `getByTestId`) first; use axes as a **last resort** (rare cases).

---

## 9. Interview Q&A

**Q1. What are XPath axes?**
Keywords that define a relationship (parent, child, sibling, ancestor…) between the current node and the target, so you can locate an element via a neighbouring node.

**Q2. Why use axes?**
When the target element has no attributes; you locate a stable neighbour and navigate to it.

**Q3. List the axes you know.**
self, parent, child, ancestor, descendant, following, following-sibling, preceding, preceding-sibling (also ancestor-or-self, descendant-or-self, attribute, namespace in the full spec).

**Q4. Difference between `following` and `following-sibling`?**
`following` = all nodes after the current one in document order, including children of siblings and later branches. `following-sibling` = only nodes sharing the same parent, no children.

**Q5. Difference between `ancestor` and `parent`?**
`parent` = one level up only; `ancestor` = all levels up.

**Q6. Difference between `child` and `descendant`?**
`child` = direct children; `descendant` = children, grandchildren and deeper.

**Q7. Difference between CSS and XPath traversal?**
CSS traverses only top → bottom; XPath can go both top → bottom and bottom → top.

**Q8. Is `self::` required?**
No. It's optional; the XPath already points at the node.

**Q9. What happens if an XPath matches nothing in Playwright?**
No exception when creating the locator; count is 0. Actions/assertions that need an element will time out (default 30s).

**Q10. Syntax of an axis expression?**
`/axisname::nodetest[predicate]`, e.g. `/ancestor::table[@id='customers']`.

**Q11. How to get the whole row from a cell?**
`//td[text()='Germany']/parent::tr` (or `/ancestor::tr`).

**Q12. How to get the cell immediately next to a given cell?**
`//td[text()='X']/following-sibling::td[1]`.

**Q13. Why not always use axes?**
They're slower and more brittle against DOM structure changes; use only when no stable attribute exists.

**Q14. Does `locator()` return different types for one vs many elements?**
No – always a `Locator`; the XPath determines the match count.

---

## 10. Quick Revision Summary

1. Axis = locate target through a **neighbour**.
2. Syntax: `/axis::tag`.
3. **parent / child** = 1 level; **ancestor / descendant** = all levels.
4. **following / preceding** = whole document order (with children); **-sibling** versions = same parent only.
5. XPath goes **both directions**, CSS only downward.
6. Slow → **last resort**, prefer attributes/built-in locators.
7. Playwright: `locator()` sync, assertions `await`, `.nth()` 0-based, XPath index 1-based.
