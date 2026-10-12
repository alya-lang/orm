# orm

[![CI](https://github.com/alya-lang/orm/actions/workflows/ci.yml/badge.svg)](https://github.com/alya-lang/orm/actions/workflows/ci.yml)
[![License](https://img.shields.io/github/license/alya-lang/orm?color=blue&label=License)](LICENSE)
[![Alya](https://img.shields.io/badge/dynamic/toml?url=https%3A%2F%2Fraw.githubusercontent.com%2Falya-lang%2Form%2Fmain%2Falya.toml&query=%24.package.alya-version&label=Alya&color=orange&prefix=%3E%3D)](https://github.com/alya-lang/alya)
[![Package Version](https://img.shields.io/badge/dynamic/toml?url=https%3A%2F%2Fraw.githubusercontent.com%2Falya-lang%2Form%2Fmain%2Falya.toml&query=%24.package.version&label=Version&color=brightgreen)](alya.toml)

Lightweight object-relational mapping for Alya: schema definitions, query builder, and typed row access over db

---

## 🌟 Features

- 📋 **Descriptor-Driven Models**: Declare columns once (`define_model` + `field`), and every operation derives from the descriptor — no stringly SQL at call sites.
- 🔒 **Injection-Proof by Construction**: Column names validate against the model, values bind as `?` parameters; hostile input throws instead of reaching SQL.
- 🎯 **Strict Value Contracts**: Values must arrive in column type (ints, strings, byte arrays, null; floats as round-trip text). Mistyped or missing values throw a `DbError` naming the column — never silent corruption.
- 🔄 **Full CRUD**: `create_table` / `drop_table`, `insert`, `find_by_id` / `find_all` / `where`, `count`, `update`, `delete_where` / `delete_by_id`, plus transactions through the `db` facade.
- 🧩 **Thin Layer over `db`**: No drivers, no C code, no new dependencies beyond `db` — adapters, pooling, and engines stay where they belong.
- 🧪 **Real Roundtrip Suite**: Every claim is verified against in-memory SQLite through `db` (including injection, rollback, and error-shape cases).

---

## 📁 Project Architecture

```text
orm/
├── alya.toml               # Package manifest (depends on db v0.2.0)
├── src/
│   ├── lib.alya            # Public API facade (schema, writes, reads + ModelDef methods)
│   └── types.alya          # FieldDef, ModelDef, KIND_* constants, find helpers
├── examples/
│   └── demo.alya           # Runnable walkthrough (model → CRUD → struct fill → strictness)
├── tests/
│   └── test_basic.alya     # Contract suite over db + sqlite adapter (in-memory)
└── benches/
    └── bench_basic.alya    # DDL generation + row-map micro-benchmarks
```

> [!NOTE]
> **Visibility & Modularity:** Symbols annotated with `pub` are exported to external consumers. Symbols without `pub` remain strictly internal to their declaring module.

---

## 📦 Installation

Add `orm` to the `[dependencies]` section in your `alya.toml`:

```toml
[dependencies]
orm = { git = "https://github.com/alya-lang/orm", branch = "main" }
```

Or install it directly using the Alya package CLI:

```bash
alya add orm --git https://github.com/alya-lang/orm --branch main
alya install
```

### Package Features

| Feature | Default | Description |
|:---|:---:|:---|
| `schema` | ✅ | DDL generation and table lifecycle (`create_table_sql`, `create_table`, `drop_table`). |
| `write` | ✅ | Row writes (`insert`, `update`, `delete_where`, `delete_by_id`). |
| `query` | ✅ | Row reads (`find_all`, `where`, `find_by_id`, `count`). |

Model definitions (`define_model`, `field`, kinds) always work.

```bash
# Full build (default)
alya install
alya test

# Slim build (model definitions only)
alya install --no-default-features
alya test --no-default-features
```

---

## 🚀 Quick Start

```alya
import "db" as db
import "orm" as pkg

function main()
    # 1. Define the model once (kinds mirror db ParamKind values)
    let users = pkg::define_model("users", [pkg::field("id", pkg::KIND_INT, 1), pkg::field("name", pkg::KIND_TEXT), pkg::field("score", pkg::KIND_FLOAT)])

    # 2. Open through db and create the table
    let conn = db::open(db::sqlite_driver(), ":memory:")
    pkg::create_table(conn, users)

    # 3. Insert (floats as round-trip text via str() at this proven site)
    let row = map()
    row["id"] = 1
    row["name"] = "Alice"
    row["score"] = str(95.5)
    pkg::insert(conn, users, row)

    # 4. Read back
    let found = pkg::find_by_id(conn, users, 1)
    say f"User #1: {found["name"]} -> {found["score"]}"

    # 5. Close
    db::close(conn)
end

main()
```

---

## 📖 API Reference

### Schema

| Symbol | Kind | Description |
|---|---|---|
| `KIND_INT` / `KIND_FLOAT` / `KIND_TEXT` / `KIND_BLOB` | `pub const` | Column kinds (`1`, `2`, `3`, `5`; mirror `db` values). |
| `field(name, kind, pk)` | `pub function` | Validated column definition (`pk = 1` requires `KIND_INT`). |
| `define_model(table, fields)` | `pub function` | Table model from ordered columns (non-empty). |
| `find_field(model, name)` | `pub function` | Column lookup, or null when absent. |
| `pk_field(model)` | `pub function` | Primary-key column, or null when none. |
| `field_name(f)` / `field_kind(f)` | `pub function` | Layout-pinned field readers. |
| `FieldDef` | `pub struct` | Column model (`name`, `kind`, `pk`). |
| `ModelDef` | `pub struct` | Table model (`table`, `fields`). |

### Writes

| Function | Arguments | Returns | Description |
|---|---|---|---|
| `create_table(conn, model)` | `conn, ModelDef` | `i32` | Creates the table from `create_table_sql`. |
| `create_table_sql(model)` | `ModelDef` | `str` | Renders DDL without executing (logging/migrations). |
| `drop_table(conn, model)` | `conn, ModelDef` | `i32` | Drops the table. |
| `insert(conn, model, values)` | `conn, ModelDef, Map` | `i32` | Inserts one row; every column needs a strictly-typed value. |
| `update(conn, model, values, cond)` | `conn, ModelDef, Map, Map?` | `i32` | Updates matching rows (`cond` null = all rows). |
| `delete_where(conn, model, cond)` | `conn, ModelDef, Map` | `i32` | Deletes matching rows (refuses empty filters). |
| `delete_by_id(conn, model, id)` | `conn, ModelDef, value` | `i32` | Deletes one row by primary key. |

### Reads

| Function | Arguments | Returns | Description |
|---|---|---|---|
| `find_all(conn, model, order, direction, limit)` | `conn, ModelDef, ...` | `Array` | All rows with optional ordering/limit. |
| `where(conn, model, cond, order, direction, limit)` | `conn, ModelDef, Map?, ...` | `Array` | Filtered rows (unknown keys throw). |
| `find_by_id(conn, model, id)` | `conn, ModelDef, value` | `Map?` | One row by pk, or null when missing. |
| `count(conn, model, cond)` | `conn, ModelDef, Map?` | `i32` | Matching row count. |

### ModelDef Methods

`create_table`, `drop_table`, `insert`, `update`, `delete_where`, `find_all`, `where`, `count` are also available as `ModelDef` methods with `self` first (e.g. `users.insert(conn, row)`).

> [!TIP]
> **Internal Helpers & Documentation:** Public symbols carry `##` Markdown docstrings for `alya doc`. Value maps should be built with index assignment (`m["k"] = v`), never as mixed literals (heterogeneous literals demote value types at check time).

---

## 🧪 Running Tests & Benchmarks

Run the automated test suite using `alya test`:

```bash
alya test
```

Generate static API documentation:

```bash
alya doc . -o docs --markdown
```

Run the benchmark suite:

```bash
alya run benches/bench_basic.alya
```

Run the example demo:

```bash
alya run examples/demo.alya
```

Check code formatting:

```bash
alya fmt . --check
```

Run static code linter:

```bash
alya lint . --check
```

---

## 🤝 Contributing

Contributions are welcome! Please follow these steps:

1. Fork the repository and clone it locally
2. Install dependencies:
   ```bash
   alya install
   ```
3. Create your feature branch (`git checkout -b feature/my-feature`)
4. Verify tests and formatting before opening a PR:
   ```bash
   alya test
   alya fmt . --check
   ```
5. Commit your changes (`git commit -m "feat: add feature"`) and open a Pull Request

---

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
