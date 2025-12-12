# README

## Project: NL → DSL → AST → Python → Backtest

This repository contains a minimal end-to-end pipeline that converts simple natural-language trading rules into a small domain-specific language (DSL), parses that DSL into an AST, generates Python code that produces entry/exit signals over OHLCV data, and runs a basic backtest.

---

## Contents

* `strategy_pipeline.py` — Single-file implementation of the whole pipeline (NL→DSL→Parser→AST→codegen→backtest). (If you saved it under a different name, update accordingly.)
* `sample_data.csv` — (Optional) Example OHLCV CSV used in the demo.
* `README_and_DSL_Grammar.md` — This file (README + DSL grammar document).

---

## Requirements

* Python 3.8+
* Packages: `pandas`, `numpy`

Install dependencies with pip:

```bash
pip install pandas numpy
```

---

## How to run (quick)

1. Make sure `strategy_pipeline.py` is in your working directory.
2. (Optional) Place any input CSV file (OHLCV) with columns: `date,open,high,low,close,volume`.
3. Run the script:

```bash
python strategy_pipeline.py
```

This runs the built-in demo using `SAMPLE_DATA` and prints:

* Natural language input
* Generated DSL
* Parsed ASTs
* Signals (head)
* Backtest report (trades, total return, PnL, max drawdown)

---

## How to run (programmatic usage)

You can import the main functions from `strategy_pipeline.py` and call `end_to_end(nl_input, df)` directly from another script or notebook. Example:

```python
from strategy_pipeline import load_sample_df, end_to_end

# load data
csv_text = open('sample_data.csv').read()
df = load_sample_df(csv_text)

# run pipeline
nl_input = "Buy when the close price is above the 20-day moving average and volume is above 1M. Exit when RSI(14) is below 30."
result = end_to_end(nl_input, df)
```

Returned `result` contains keys: `dsl`, `entry_ast`, `exit_ast`, `signals`, `results`, `func_code`.

---

## File structure suggestions (if you create a repo)

```
repo/
├─ README.md
├─ strategy_pipeline.py
├─ sample_data.csv
├─ tests/
│  ├─ test_parser.py
│  ├─ test_codegen.py
│  └─ test_backtest.py
└─ docs/
   └─ dsl_grammar.md
```

---

## Notes & Limitations

* The NL→DSL mapper implemented in `nl_to_dsl` is heuristic-based and supports only a small set of templates. For production use, either expand the regex patterns or use a prompt-based LLM translation.
* Parser implemented is a small recursive-descent parser. Consider replacing with `lark` or `antlr` for complex grammar and better error messages.
* Generated code uses `exec()` for convenience. For security and maintainability, prefer evaluating AST directly or using a sandboxed codegen flow.
* Backtester is intentionally simple (enters/exits at bar `close`). For realistic simulation, add slippage, commission, next-bar fills, and position sizing.

---

# DSL Grammar Document

> This section describes the DSL used in the pipeline. It is intentionally compact and easy to parse. Use this as the reference when writing rules manually.

## Goals

* Express entry/exit rules for trading strategies
* Support boolean logic (AND/OR), comparisons, parentheses
* Support indicators: SMA, RSI
* Support cross events like `crosses_above(series, series_or_value)`
* Support time lookbacks such as `yesterday` (represented as `yesterday_high`, etc.)

## High-level format

A DSL file contains two optional blocks: `ENTRY:` and `EXIT:` (case-insensitive). Each block holds an expression describing rules. Example:

```
ENTRY:
close > SMA(close,20) AND volume > 1000000

EXIT:
RSI(close,14) < 30
```

If a block contains multiple conditions combined with `AND` or `OR`, parentheses are allowed for grouping.

## Tokens

* **IDENT**: `open`, `high`, `low`, `close`, `volume`, `yesterday_high`, etc.
* **NUMBER**: integer or float literal, e.g. `1000000`, `1.5`, `20`
* **FUNCTION**: `SMA`, `RSI`, `crosses_above` (case-insensitive)
* **OPERATORS**: `>`, `<`, `>=`, `<=`, `==`
* **BOOLEAN**: `AND`, `OR` (case-insensitive)
* **PUNCT**: `(` `)` `,`

## Grammar (EBNF-like)

```
program      := [ENTRY_BLOCK] [EXIT_BLOCK]
ENTRY_BLOCK  := 'ENTRY' ':' expression
EXIT_BLOCK   := 'EXIT'  ':' expression

expression   := or_expr
or_expr      := and_expr { 'OR' and_expr }
and_expr     := comp_expr { 'AND' comp_expr }
comp_expr    := arith_expr [ comp_op arith_expr ]
comp_op      := '>' | '<' | '>=' | '<=' | '=='
arith_expr   := primary
primary      := IDENT
              | NUMBER
              | FUNCTION_CALL
              | '(' expression ')'

FUNCTION_CALL := IDENT '(' [ argument_list ] ')'
argument_list := expression { ',' expression }

IDENT        := [A-Za-z_][A-Za-z0-9_]*
NUMBER       := digit+ ('.' digit+)?
```

**Notes:**

* `SMA(close,20)` is a `FUNCTION_CALL` where `IDENT` is `SMA` and arguments are `IDENT` and `NUMBER`.
* `crosses_above(close, yesterday_high)` is supported; `yesterday_high` is treated as an `IDENT` that maps to `df['high'].shift(1)` in codegen.

## Supported functions & semantics

* `SMA(series, period)` — Simple moving average using window `period` (integer).
* `RSI(series, period)` — Relative Strength Index with given period (integer).
* `crosses_above(series1, series2)` — True on rows where `series1` crosses above `series2` (prev <= prev2 and curr > curr2).

## Example DSL rules

1. Simple moving-average entry

```
ENTRY:
close > SMA(close,20) AND volume > 1000000
```

2. Cross entry using yesterday's high

```
ENTRY:
crosses_above(close, yesterday_high)
```

3. Exit using RSI

```
EXIT:
RSI(close,14) < 30
```

4. Nested logic

```
ENTRY:
(close > SMA(close,50) AND volume > 500000) OR (close > SMA(close,20) AND volume > 1000000)
```

## Mapping to code (notes for codegen)

* `IDENT` mapping:

  * `open` -> `df['open']`
  * `high` -> `df['high']`
  * `low` -> `df['low']`
  * `close` -> `df['close']`
  * `volume` -> `df['volume']`
  * `yesterday_high` -> `df['high'].shift(1)` (convention)

* `SMA(series, period)` -> call `sma(df['series'], period)` helper

* `RSI(series, period)` -> call `rsi(df['series'], period)` helper

* `crosses_above(a,b)` -> call `cross_above(a_expr, b_expr)` helper

Boolean `AND` / `OR` map to pandas boolean operations `&` and `|`.

## Error handling & validation

* Parser must ensure function argument counts are correct (e.g., `SMA` requires 2 args).
* Period arguments must be positive integers — codegen should coerce and clip to `>=1`.
* Unknown identifiers should raise validation errors.
* Division by zero or empty rolling windows must be guarded in helper functions.

## Assumptions & Limitations

* DSL favors simplicity over exhaustive natural-language coverage.
* Time windows like `last week` or `N-day window` are not implemented as first-class tokens — they can be expressed via functions or additional conventions (e.g., `sma(close,5)` represents 5-day window). If needed, extend DSL with `avg_over(close,5d)` style syntax.
* `yesterday` is mapped to `shift(1)`; `last_week` or other named windows should be defined explicitly if used.

---

# Quick troubleshooting

* If `sma(..., 0)` occurs, ensure DSL parser correctly parsed the period number. The SMA helper uses a safe guard `period = max(1, int(period))` to avoid `rolling(0)` errors.
* If boolean ops behave unexpectedly, ensure parentheses are used to disambiguate logic.

---

# Next steps / Possible enhancements

* Replace the custom parser with a formal `lark` grammar for more robust parsing and better error messages.
* Expand NL→DSL translator using prompt-based LLM to support more natural-language variations.
* Improve backtester (next-bar fills, slippage, commission, position sizing, long/short).
* Add unit tests for parser, AST generator, codegen and backtest.

---

If you'd like, I can also:

* Produce a `lark` grammar file and parser implementation.
* Scaffold tests (`pytest`) covering edge cases.
* Create a small GitHub repository ZIP and provide it to you.

Tell me which of the above you want next.
