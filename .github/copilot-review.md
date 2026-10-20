# Code Review — cachetools 8.0.0 (unreleased)

**Date:** 2026-10-20
**Reviewed:** `dev/v8.0.0` at `46345ef`, rebased onto
`v7.2.4` (`6f23e2a`). Changes since the last review (`64bf761`): the v7.2.4 merge
brought `_Unbound` (`95e6add`, "Fix #443: Restore autospec calls to `@cachedmethod`
decorated methods"), `--cov-branch` in `tox.ini` and the 7.2.4 `CHANGELOG.rst`
entry; `b172253` ("Autospec cleanup") merges the autospec tests; the lazy-expiry
docs and `test_ttl_iter_expired`/`test_ttu_iter_expired` are now committed, and
`ClassMethodTest`, lost in the rebase of `9182ec2`, is restored in the working tree.
Scope: all of `src/`, `tests/`, `docs/`, stubs, packaging and CI.

**CI status:** All `tox` environments pass locally on Python 3.11.2: `py` (384 passed),
`docs`, `doctest` (58 tests), `pyright` (0 errors), `ruff`, `ruff-format`.
3.12–3.15, the free-threaded builds and PyPy were not run locally.

**Coverage:** `tox` measures line and branch coverage (`--cov-branch` in `tox.ini`),
which CI uploads to Codecov.  100% locally on 3.11, where the pre-3.13 `@classmethod`
check (`_descriptor.py:32`) is reached; runs on 3.13+ alone miss that line.

## Code — Potential Issues

No functional regressions found. All items are design trade-offs or edge cases;
none blocks the release.

| # | Priority | Location | Finding |
|---|----------|----------|---------|
| 1 | Low | `_MethodDescriptor.__get__` (`super()` path) | `super().method` never hits the instance dict, so every call builds a throw-away wrapper (including `functools.update_wrapper`) and rescans the MRO before `setdefault` returns the stored one. Correct, but noticeably slower than the primary path. |
| 2 | Low | `Cache.__setitem__` / `__delitem__` | Size accounting assumes a value's size is stable after insertion; mutating cached values in place can desync `currsize`. By design and documented, but a known footgun. |
| 3 | Low | `_decorator.py` `_ConditionMixin` / `_ConditionInfoMixin` | Stampede prevention uses a pending set per wrapper, i.e. per instance for `@cachedmethod`. Two instances sharing one cache and condition do not coordinate pending keys. |
| 4 | Low | `_descriptor.py` `_Unbound` | Class access (`Cls.method`) now returns an `_Unbound` callable object instead of a function (#443).  Verified with a probe on 3.11: `pickle.dumps(Cls.method)` raises `PicklingError` for protocols 0–5 ("not the same object as `Cls.method`", since `__wrapped__` is pickled by `__qualname__`), whereas the previous `unbound` function pickled by name; `inspect.isroutine(Cls.method)` is `False`, so introspection tools that look for routines may treat it as a data attribute; `repr()` is `<cachetools._descriptor._Unbound object at ...>`; and `alias = Cls.method` in another class body no longer binds, so `Other().alias(1)` passes `1` as the receiver ("No '__dict__' attribute on 'int' instance").  Accepted cost of the autospec fix; a `__repr__` and `__reduce__` (e.g. via `_MethodDescriptor._lookup` plus `__get__(None)`) would cover the first and third points cheaply. |
| 5 | Info | `_MethodDescriptor.__get__` | `copy.copy(obj)` shallow-copies `obj.__dict__`, so the copy shares the original's wrapper, still bound to the original `obj` (`a2.get is a.get`).  `copy.deepcopy` and `pickle` are fine.  Accepted, `functools.cached_property` semantics; the data-descriptor fix was rejected for its per-call cost (*Maintainer Decisions* in `copilot-instructions.md`). |
| 6 | Info | `_descriptor.py` / `_cachedmethod.py` `_wrapper` | Stacked `@cachedmethod` silently uses the inner cache only, because `functools.update_wrapper(descriptor, method)` copies the inner descriptor's private state onto the outer one.  Stacking is unsupported and untested (*Maintainer Decisions* in `copilot-instructions.md`). |
| 7 | Info | `_descriptor.py` `_Unbound.__call__` | By design (see its docstring), autospec'd instances accept any arguments: `inspect.signature(mock.method)` is `(*args, **kwargs)` and `mock.method(1, 2)` does not raise for a one-parameter method (verified).  Autospec no longer checks call arity for `@cachedmethod` methods. |
| 8 | Info | `_cachedmethod.py` `_WrapperBase` | `inspect.signature(obj.method)` reports `(self, value)` for a `@cachedmethod` wrapper, because `__wrapped__` is the unbound function and the wrapper is a callable object rather than a `MethodType`. `@cached` on a method yields `(value)` since `__get__` returns a real bound method. Frameworks that `bind()` call arguments against the signature will see a spurious `self` parameter. Setting `__wrapped__` to the bound partial would fix introspection but break the documented `wrapper.__wrapped__(self, ...)` contract; a per-instance `__signature__` costs time on every first access. Document rather than fix. |
| 9 | Info | `_WrapperBase.__reduce__` | Unpickling resets `hits`/`misses` of `info=True` wrappers (documented in the code and asserted by `test_decorator_pickle_info`). Acceptable, since counters are a tuning aid. |
| 10 | Info | `_MethodDescriptor.__reduce__` / `_lookup` | Pickles reference `cachetools._descriptor._MethodDescriptor._lookup` by qualified name, so that path must stay importable for pickles to load across releases (same constraint as `@cached` pickling by `__qualname__`). `_lookup` also resolves through the owner's *current* class dict: a pickle loaded while `Cls.method` is reassigned (verified with `unittest.mock.patch.object`) silently yields a bound method of the replacement instead of a wrapper. |
| 11 | Info | `_MethodDescriptor.__reduce__` | A descriptor that never received `__set_name__` reduces to `_lookup(None, None)`: `pickle.dumps` succeeds, `pickle.loads` raises `AttributeError`. Such a descriptor is unusable anyway. |
| 12 | Info | `_MethodDescriptor.__set_name__` | Assigning one descriptor to the same attribute name in two classes keeps the first owner silently (verified: the second owner's instances work and store the wrapper under the same name; `_lookup` resolves through the first owner). Harmless as long as the first class keeps the attribute; the accepted case is pinned by `test_decorator_same_name`. |
| 13 | Info | `_cachedmethod.py` `_WrapperBase` | Each per-instance wrapper references `obj`, and `obj.__dict__` references the wrapper, so instances with active wrappers are freed only by the cyclic GC, not by reference counting (verified with `gc.disable()` and a weakref). Expected for this design; worth knowing for large or short-lived instances. |
| 14 | Info | `_decorator.py` condition mixins | Same-thread recursive re-entry on the same key deadlocks (`wait_for` on its own pending marker). |
| 15 | Info | `_MethodDescriptor.__get_unbound` / `__call__` | Lazy creation of the cached `_Unbound` object is unsynchronized; concurrent first class accesses may briefly see distinct (equivalent) objects. Benign. Two error messages are misleading at the margins: `Cls.method(None, 1)` reports "No `__dict__` attribute on 'int' instance" (the `None` receiver makes `__get__` return the `_Unbound` object, which then takes `1` as its receiver), and calling the raw descriptor from `Cls.__dict__` reports "Decorating class methods is not supported". |
| 16 | Info | `TTLCache.__delitem__` / `TLRUCache.__delitem__` | Deleting a key that is present but expired removes it and then raises `KeyError`. Consistent with `key in cache` being `False`; intentional. Neither `__delitem__` calls `expire()` (verified), unlike `__setitem__`, `__len__`, `__repr__`, `currsize` and `popitem`. |
| 17 | Info | `Cache.__setitem__` | With `maxsize=math.inf` and a `getsizeof` returning `math.inf`, the value is accepted (`inf > inf` is false), `currsize` becomes `inf`, and deleting the item leaves `currsize = nan`. Only non-negative sizes are validated. Pathological input; not worth a check on the hot path. |
| 18 | Info | `TLRUCache.__iter__` | Iterates the expiry heap, so `list(cache)` is in heap order (verified: inserting 1, 2, 3 with ttu 30/10/20 yields `[2, 1, 3]`), while `TTLCache` iterates in expiry (= insertion) order and `popitem()` uses LRU order. No ordering is documented; fine, but surprising next to `TTLCache`. |

## Type Stubs (`__init__.pyi`, `keys.pyi`, `func.pyi`)

| # | Priority | Finding |
|---|----------|---------|
| 1 | Info | `@classmethod` stacked over `@cached`/`func.*_cache` still reports a false "Argument missing" error. Python passes `cls` correctly at runtime, but pyright does not bind through the wrapper's `__get__` there. Rare and not encouraged. |
| 2 | Info | `_cached_wrapper_info.__get__` carries `# pyright: ignore[reportIncompatibleMethodOverride]`. Pyright cannot match the subclass overloads against the base's self-typed `Concatenate` overload. The override is sound. |
| 3 | Info | `func.pyi` `_cachetools_cache_wrapper` exposes only the `functools.lru_cache`-compatible API (`cache_info`, `cache_clear`, `cache_parameters`), while the runtime wrapper, being built on `cached(...)`, also has `cache`, `cache_key`, `cache_lock` and `cache_condition`. Those are undocumented for `cachetools.func`, so hiding them is defensible; just be aware that user code touching `lru_cache(...)(f).cache` type-checks only with a cast. |
| 4 | Info | `Cache.setdefault(key, default: _VT \| None = None) -> _VT` returns `_VT` even when `None` is inserted and returned (verified: `Cache(2).setdefault(1)` is `None` and stores it). typeshed's `MutableMapping.setdefault` uses two overloads for this. Minor unsoundness, consistent with the previous stub. |
| 5 | Info | `TTLCache.__init__`'s second overload keeps `ttl: Any  # FIXME: must be "addable" to _TT` — the one remaining `FIXME` in the code base. It documents a real limitation of the type system (no "addable to `_TT`" constraint), so it is a note rather than a leftover. |
| 6 | Info | `info: bool` (non-literal) arguments to `cached`/`cachedmethod`, as in `test_threading.py`, type-check only because pyright expands `bool` into `Literal[True] | Literal[False]` when matching overloads. mypy does not, so mypy users passing a `bool` variable will see "no overload matches". The project targets pyright only. |

## Tests — Gaps

| # | Priority | Finding |
|---|----------|---------|
| 1 | Low | No test checks that `copy.copy(obj)` gets its own wrapper, or pins the current sharing behavior (Code #5). |
| 2 | Low | `UnboundMethodTest` does not assert that class access exposes no `cache_clear`/`cache_info` (verified: both `hasattr` checks are `False`), and does not exercise class access for the lock/condition variants. |
| 3 | Low | No test pins the `_Unbound` trade-offs (Code #4), e.g. that `pickle.dumps(Cls.method)` fails, so a future change to class access cannot be judged against the current behavior. |
| 4 | Low | The pickle tests use the default protocol only. Protocols 0–5 were verified by hand; a `subTest` loop over `range(pickle.HIGHEST_PROTOCOL + 1)` would pin this, as `tests/__init__.py` does for caches. |
| 5 | Info | `test_threading.py` `run_cond_test` asserts `misses == 1`, but nothing forces the threads to overlap (the lock tests use a `Barrier` inside the function; a condition-guarded function cannot, since only one thread enters). `misses == 1` would also hold if the threads happened to run sequentially, so the test demonstrates correctness but cannot detect a broken stampede guard. Hard to fix deterministically; note only. |
| 6 | Info | `CacheTestMixin` deliberately checks key existence rather than the specific eviction victim; policy-specific checks live in the per-cache test files. No action needed. |
| 7 | Info | `test_func.py` calls decorators through `Callable[..., Any]`; one concrete test (`FIFODecoratorTest.test_method_typed`) covers the shared `_cachetools_cache_wrapper` stub. Enough. |

## Docs — `docs/index.rst`

`tox -e docs` and `tox -e doctest` pass with `-W`. The text is clear and consistent
(two spaces after sentences, double backticks for code, `versionchanged` entries for
7.0 and 8.0, `:meth:`/`:func:`/`:exc:` roles by kind).

| # | Priority | Finding |
|---|----------|---------|
| 1 | Low | The `@cachedmethod` docs do not mention that class access (`Cls.method`) yields a plain callable without `cache_clear()`/`cache_info()` that cannot be pickled (Code #4), that `copy.copy()` of an instance shares its wrappers (Code #5), that pickling an instance resets its `cache_info()` counters, or that `inspect.signature()` of a bound wrapper still shows `self` (Code #8). |
| 2 | Low | The pickling note on `Cache` (lines 80–88) says the cache classes "can generally be pickled" without saying what the exception is. Verified with a probe: pickling fails with `PicklingError` for a `getsizeof`, `timer` or `ttu` callable that is itself unpicklable, e.g. a lambda; the defaults and module-level functions such as `len` work. Suggest "can be pickled, provided that any `getsizeof`, `timer` or `ttu` callable is picklable". |
| 3 | Info | Neither the `@cached` nor the `cachetools.func.*_cache` docs mention that they may decorate methods. Worth a one-line caveat that the cache and counters are then shared across all instances, unlike `@cachedmethod` (verified: two instances share one `CacheInfo`). |
| 4 | Info | The `ExpCache` doctest (line 262) calls `time.sleep(1.5)`, which accounts for most of the `doctest` environment's runtime. A `Timer` object as in `tests/test_ttl.py` would make the example deterministic and instant, at the cost of a few more lines. |

## Docs — `README.rst`

No issues found. The examples are runnable as written and use the same PEP URL and
glossary links as `docs/index.rst`.

## `CHANGELOG.rst`

No issues found.

## Code comments and docstrings

No issues found.

## Keys & Func Modules

No issues found. `hashkey`/`typedkey` sort keyword arguments and cache the hash in
`_HashedTuple`, which is not pickled. Every `func.py` decorator is built on
`cached(..., condition=Condition(), info=True)`, so stampede prevention is on by
default, and `Condition()`'s default `RLock` keeps `test_decorator_needs_rlock`
passing. `_UnboundTTLCache` extends `TTLCache` with `math.inf` maxsize for
`maxsize=None`; `rr_cache(maxsize=None)` ignores `choice`, which is correct since an
unbounded `dict` never evicts.

## Packaging, CI and Tooling

No issues found. `pyproject.toml` requires `setuptools-scm` only for git-based sdist
file discovery (the version comes from `cachetools.__version__`), and `MANIFEST.in`
prunes `.github/`. The `ruff` rule set is the ruff default (currently a broad
preset including `UP`, `PYI`, `RUF`, `PL*`, `B`, `SIM`, `DTZ`, …); the two entries in
`[tool.ruff.lint] ignore` (`DTZ005`, `UP031`) are therefore effective. CI pins
actions by SHA with Dependabot grouping monthly updates against `develop`; the
matrix matches the classifiers (3.11–3.15, 3.13t–3.15t, PyPy 3.11). Codecov needs
the explicitly installed `coverage` package to convert `.coverage` to XML.
