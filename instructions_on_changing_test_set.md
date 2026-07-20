# Instructions: Changing the Test Set Source in This Notebook

This document tells you **exactly where** the notebook reads the test set from disk
and **how** to point it at a different location. It is written for organisers /
graders who need to re-run the notebook against their own copy of the test file.

---

## TL;DR

- The test set is loaded from disk in **four** places (one per pipeline phase).
- Every one of them uses the **same pattern**: try a single hardcoded path first,
  and only fall back to a recursive search if that path does not exist.
- The default hardcoded path is:

  ```
  /kaggle/input/competitions/bengali-hallucination/test set.csv
  ```

  Note the **space** in `test set.csv` (not an underscore).
- To change the source, either (a) mount your test file at that exact path, or
  (b) edit the four `_HARD_TEST_CSV` lines listed below to your path.

You do **not** need to understand the pipeline to change the test source. Just
edit the four lines described in the table, or make your file available at the
default path and change nothing.

---

## The four load points (where the test set is read)

Each phase rebuilds its own view of the test set from disk so it can be run
standalone (e.g. after a kernel restart). That is why the same change must be
made in **four** places — they do not share a single loader in memory. Full code
for each is in the "details" section further down.

| # | Phase             | Cell (approx.) | Symbol built  | What to look for                                  |
|---|-------------------|----------------|---------------|---------------------------------------------------|
| 1 | Phase A (grounded)| ~11            | `test_df`     | `def load_test()`                                 |
| 2 | Phase B (closed-book) | ~82        | `test_df_full`| `def load_test_b()`                               |
| 3 | Phase G (grammar) | ~145           | `_gtest`      | `_gtest_path = ...` / `_gtest = pd.read_csv(...)` |
| 4 | Final merge (coverage check) | ~154| `_test_ids`   | `_test_ids = set(pd.read_csv(...)...)`            |

> Cell numbers are approximate — they shift if cells are added or removed. Find
> the cells by searching the notebook for the strings in the last column, or for
> `_HARD_TEST_CSV`.

---

## Expected file format

The loader expects a CSV with at least these columns:

| column        | meaning                                             |
|---------------|-----------------------------------------------------|
| `id`          | unique row identifier (kept as string)              |
| `context`     | grounding passage; `[NULL]` / empty means closed-book |
| `prompt_bn`   | the question (Bengali)                              |
| `response_bn` | the answer to be judged faithful / hallucinated     |

No `label` column is required (the test set is unlabelled). The three text
columns are NFC-normalised on load, and a row is treated as **closed-book**
when `context` is `[NULL]`, empty, `"nan"`, or `"None"`.

---

## How the loading pattern works

Every load point uses this shape:

```python
_HARD_TEST_CSV = "/kaggle/input/competitions/bengali-hallucination/test set.csv"
_path = _HARD_TEST_CSV if os.path.exists(_HARD_TEST_CSV) else <recursive_fallback>
df = pd.read_csv(_path)
```

- **If the hardcoded file exists**, it is used directly — fast and unambiguous.
- **If it does not exist**, the notebook falls back to a recursive search under
  `/kaggle/input` for a file named `test set.csv` (and, in some phases,
  `test_set.csv` as well). This is what makes the notebook portable across
  differently-named attached datasets.

So there are **two ways** to change the source:

1. **Preferred — no code change:** mount your test CSV at
   `/kaggle/input/competitions/bengali-hallucination/test set.csv`. Everything
   works unchanged.
2. **Edit the path:** change the `_HARD_TEST_CSV` string (and/or the filename in
   the fallback) in each of the four cells below.

---

## The four load points — details

The table of all four load points (with cell numbers) is at the **top** of this
document. The subsections below give the exact code and the fallback used at each.

### 1. Phase A — `load_test()` (builds `test_df`)

Find this function:

```python
def load_test() -> pd.DataFrame:
    _HARD_TEST_CSV = "/kaggle/input/competitions/bengali-hallucination/test set.csv"
    _path = _HARD_TEST_CSV if os.path.exists(_HARD_TEST_CSV) else _find(CFG.TEST_CSV)
    df = pd.read_csv(_path)
    return _clean(df, has_label=False)
```

To change the source, edit `_HARD_TEST_CSV`. The recursive fallback here is
`_find(CFG.TEST_CSV)`, where `CFG.TEST_CSV = "test set.csv"` is defined in the
config cell (~cell 7). If your file has a different **name**, change it there too.

---

### 2. Phase B — `load_test_b()` (builds `test_df_full`)

Find this function:

```python
def load_test_b() -> pd.DataFrame:
    _HARD_TEST_CSV = "/kaggle/input/competitions/bengali-hallucination/test set.csv"
    _path = _HARD_TEST_CSV if os.path.exists(_HARD_TEST_CSV) else _find_b(TEST_CSV_B)
    df = pd.read_csv(_path)
    return _clean_b(df, has_label=False)
```

Edit `_HARD_TEST_CSV` to change the source. The fallback here is
`_find_b(TEST_CSV_B)`, where `TEST_CSV_B = "test set.csv"` is defined a few lines
above in the same cell. Change that string too if your filename differs.

---

### 3. Phase G — `_gtest` (grammar phase)

Find these lines near the top of the cell:

```python
_HARD_TEST_CSV = "/kaggle/input/competitions/bengali-hallucination/test set.csv"
_gtest_path = _HARD_TEST_CSV if os.path.exists(_HARD_TEST_CSV) else next(
    (p for p in [CFG.TEST_CSV, "test set.csv", "test_set.csv",
                 *glob.glob("/kaggle/input/**/test*set*.csv", recursive=True)]
     if os.path.exists(p)))
_gtest = pd.read_csv(_gtest_path)
```

Edit `_HARD_TEST_CSV` to change the source. The fallback list already tries both
`test set.csv` and `test_set.csv`, plus a wildcard glob `test*set*.csv`, so it is
the most permissive of the four.

---

### 4. Final merge — `_test_ids` (coverage check)

Find this line in the final-merge cell:

```python
_HARD_TEST_CSV = "/kaggle/input/competitions/bengali-hallucination/test set.csv"
_test_ids = set(pd.read_csv(
    _HARD_TEST_CSV if os.path.exists(_HARD_TEST_CSV) else _find(CFG.TEST_CSV)
)["id"].astype(str))
```

This does **not** produce predictions — it reads the test file only to confirm
that the final submission covers **exactly** the test ids (no missing rows, no
extras). It must point at the **same** file as the three loaders above, or the
coverage assertion will fail. Edit `_HARD_TEST_CSV` to match.

---

## Recommended way to change the source (one edit, applied consistently)

If you expect to change the path more than once, define it **once** in the config
cell and reference it everywhere, instead of editing four separate strings:

1. In the config cell (~cell 7, next to `TEST_CSV = "test set.csv"`), add:

   ```python
   HARD_TEST_CSV = "/kaggle/input/competitions/bengali-hallucination/test set.csv"
   ```

2. In each of the four load points, replace the local
   `_HARD_TEST_CSV = "..."` line with:

   ```python
   _HARD_TEST_CSV = CFG.HARD_TEST_CSV   # or just HARD_TEST_CSV, depending on scope
   ```

Then there is a single source of truth for the path. (The four cells were kept
self-contained by default so each phase remains runnable standalone after a
kernel restart — that is the only reason the string is repeated.)

---

## Verifying the change

After editing, a correct configuration will:

- Load without a `FileNotFoundError` at each phase.
- Print the row counts, e.g. `[Phase A] ... test: N rows ...` and
  `[Phase B] rebuilt test_df_full: N rows ...`.
- Pass the final coverage assertions in the merge cell:
  - `... test ids have no prediction` must **not** fire, and
  - `... predicted ids are not in the test set` must **not** fire.

If a coverage assertion fires, the most common cause is that one of the four load
points is pointing at a **different** file than the others — re-check that all
four `_HARD_TEST_CSV` values (and any changed filename in the fallbacks) are
identical.

---

## Common pitfalls

- **Space vs. underscore in the filename.** The default filename is
  `test set.csv` with a **space**. If your file is `test_set.csv` with an
  underscore, either rename it, set `_HARD_TEST_CSV` to the underscore path, or
  rely on the Phase G fallback (which already tries both) — but the other three
  phases key on the exact `_HARD_TEST_CSV` string, so set it explicitly.
- **Only editing one or two of the four cells.** All four must agree. The final
  merge cross-checks coverage against the file it reads, so a mismatch surfaces
  as an assertion error rather than a silent wrong answer.
- **Path outside `/kaggle/input`.** The hardcoded-first check uses
  `os.path.exists`, so any readable absolute path works (e.g. a local mount).
  Only the **recursive fallback** is restricted to `/kaggle/input`.
