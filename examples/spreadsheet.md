# Spreadsheet

A simple spreadsheet-like app which demonstrates use of [`regular-table`](https://github.com/jpmorganchase/regular-table).
Supports a simple expression language for cells starting with a `=`
character, such as `=sum(A2..C4)` for the sum of cell values within the
rectangular region (A2, C4).

# Data Model

The `<regular-table>` in question uses the simplest data model of all, the
humble 2D `Array`.  We'll start with an empty one in columnar-orientation.

```javascript
const NUM_COLUMNS = 200;
const NUM_ROWS = 1000;

const DATA = Array(NUM_COLUMNS)
    .fill()
    .map(() => Array(NUM_ROWS).fill());
```

In Excel-like fashion though, we'll want _alphabetic_ symbols for
`column_headers`, so we'll generate a sequence of those using 
`String.fromCharCode()`.

```javascript
const DATA_COLUMN_NAMES = generate_column_names();

function generate_column_names() {
    const nums = Array.from(Array(26));
    const alphabet = nums.map((val, i) => String.fromCharCode(i + 65));
    let caps = [],
        i = 1;
    while (caps.length < NUM_COLUMNS) {
        caps = caps.concat(alphabet.map(to_column_name));
        i++;
    }
    return caps;
}

function to_column_name(letter) {
    return Array(i).fill(letter).join("");
}
```

This leads to a simple Virtual Data Model based on `Array.prototype.slice()`.

```javascript
function dataListener(x0, y0, x1, y1) {
    return {
        num_rows: DATA[0].length,
        num_columns: DATA.length,
        row_headers: Array.from(Array(Math.ceil(y1) - y0).keys()).map((y) => [`${y + y0}`]),
        column_headers: DATA_COLUMN_NAMES.slice(x0, x1).map((x) => [x]),
        data: DATA.slice(x0, x1).map((col) => col.slice(y0, y1)),
    };
}
```

We can go ahead and register this `dataListener` with our `<regular-table>`
now, since nothing will happen within this cycle of the event loop until
`draw()` is called.

```javascript
const table = document.getElementsByTagName("regular-table")[0];
table.setDataListener(dataListener);
```

# Expression Language

Our expression language features this expansive standard library:

```javascript
function sum(arr) {
    return flat(arr).reduce((x, y) => parseInt(x) + parseInt(y));
}

function avg(arr) {
    const x = flat(arr);
    return x.reduce((x, y) => parseInt(x) + parseInt(y)) / x.length;
}
```

It will also internally use these helper functions:

* `flat(2, 6)` for cell references`B6`
* `slice(1, 3, 1, 5)` for rectangular slices `A3..A5`

```javascript
function flat(arr) {
    return arr
        .flat(1)
        .map((x) => parseInt(x))
        .filter((x) => !isNaN(x));
}

function slice(x0, y0, x1, y1) {
    return DATA.slice(x0, parseInt(x1) + 1).map((z) => z.slice(y0, parseInt(y1) + 1));
}
```

```javascript
function col2Idx(x) {
    return DATA_COLUMN_NAMES.indexOf(x);
}

function stringify(x, y) {
    let txt = DATA[x][y];
    let num = parseInt(txt);
    if (isNaN(num)) {
        num = txt;
    }
    return `${num}`;
}
```

The evaluation engine uses the most powerful, performant and _utilized_ general
purpose parsing framework available today: `Regex`.

```javascript
const RANGE_PATTERN = /([A-Z]+)([0-9]+)\.\.([A-Z]+)([0-9]+)/g;
const CELL_PATTERN = /([A-Z]+)([0-9]+)/g;
```

The `compile()` function simply removes the leading `=` and applies these
regular expressions via `replace()` - there is no need to handle nested cases,
since neither of these patterns are recursive.

```javascript
function compile(input) {
    const output = input
        .slice(1)
        .replace(RANGE_PATTERN, (_, x0, y0, x1, y1) => `slice(${col2Idx(x0)}, ${y0}, ${col2Idx(x1)}, ${y1})`)
        .replace(CELL_PATTERN, (_, x, y) => `stringify(${col2Idx(x)}, ${y})`);
    console.log(`Compiled '${input}' to '${output}'`);
    return eval(output);
}
```

# User Interaction



```javascript
table.addStyleListener(() => {
    for (const td of table.get_tds()) {
        td.setAttribute("contenteditable", true);
    }
});

table.draw();
```

`contenteditable` takes care of most of the basics for us, but we'll still
need to update our data model when the user evaluates a cell.  Given a cell,
this is a simple task of checking the first character for `"="` to determine
whether this cell needs to be `eval()`'d, then setting the Array contents of
`DATA` directly and calling `draw()` to update the `regular-table`.

```javascript
function write(active_cell) {
    const meta = table.getMeta(active_cell);
    if (meta) {
        let text = active_cell.textContent;
        if (text[0] === "=") {
            text = compile(text);
        }
        DATA[meta.x][meta.y] = text;
        active_cell.blur();
        table.draw();
    }
}
```

We'll call this function whenever the user evaluates a cell, such as when
the `return` key is pressed, by looking up the element with focus,
`document.activeElement`.

```javascript
table.addEventListener("keypress", (event) => {
    if (event.keyCode === 13) {
        event.preventDefault();
        const target = document.activeElement;
        write(target);
        increment(target);
    }
});
```

This also makes use of `increment()`, which uses some simple metadata-math
to look up the cell in the next row of this column, and `focus()` it.

```javascript
function increment(active_cell) {
    const meta = table.getMeta(active_cell);
    const rows = Array.from(table.querySelectorAll("tbody tr"));
    const next_row = rows[meta.y - meta.y0 + 1];
    if (next_row) {
        const td = next_row.children[meta.x + 1];
        td.focus();
    }
}
```

There are some simple quality-of-life improvements we can make as well.
By default, a `scroll` event such as initiated by the mouse wheel will cause
`regular-table` to re-render, which will result in the very un-spreadsheet 
like behavior of resetting a cell which has focus and was in a partial state
of edit.  To prevent this, we'll call `write()` when a scroll event happens.

```javascript
table.addEventListener("scroll", () => {
    write(document.activeElement);
});
```

In fact, let's go ahead and do this anytime focus is lost on an element within
our `<ergular-table>`.

```javascript
table.addEventListener("focusout", (event) => {
    write(event.target);
});
```

# HTML and CSS

There is not much elaborate about the HTML setup for this `regular-table`.

```html
<regular-table></regular-table>
```

However, we'd like an Excel-like User Experience, so let's liven up the default
theme with a trendy grid, which we can easily do purely via CSS, since these
cells are always `<td>` elements.  We're also going to limit the cells to
`22px` - they need to be big enough to click on, and as they start empty, they
may end up quite narrow.

```css
td {
    outline: none;
    border-right: 1px solid #eee;
    border-bottom: 1px solid #eee;
    min-width: 22px;
}
```

We'll do the same for `row_headers` and `column_headers` to separate them from
the editable cells.

```css
th {
    border-right: 1px solid #eee;
}
```


# Appendix (Dependencies)
 
```html
<script src="/dist/umd/regular-table.js"></script>
<link rel='stylesheet' href="/dist/css/material.css">
```

```block
license: apache-2.0
```