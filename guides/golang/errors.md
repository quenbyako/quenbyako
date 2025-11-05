# Error types in Go: A Quickie-Guide

[![CC BY-ND 4.0][cc-by-nc-nd-shield]][cc-by-nc-nd]

> [!NOTE]
> This work is licensed under a [Creative Commons Attribution-NonCommercial-NoDerivatives 4.0 International License][cc-by-nc-nd].

[cc-by-nc-nd]: https://creativecommons.org/licenses/by-nc-nd/4.0/
[cc-by-nc-nd-shield]: https://img.shields.io/badge/License-CC%20BY--NC--ND%204.0-lightgrey.svg

## Sentinel Errors

Sentinel errors are predefined error variables used to indicate specific system states. They’re usually created with errors.New and exported for use in other packages.

```go
var (
    ErrOutOfTea      = errors.New("no more tea available")
    ErrInvalidCupSize = errors.New("invalid cup size")
)

const water = 100

func makeTea(cups int) error {
    if cups <= 0 {
        return ErrInvalidCupSize
    }
    if cups > 5 {
        return ErrOutOfTea
    }

    return nil
}
```

**When to use:**

1. A process transitions into a well-defined state that doesn’t require extra context (resource not found, limit exceeded, etc.).
2. Completion / termination signals (e.g., `io.EOF`, `context.Canceled`).

**When NOT to use:**

1. When contextual data about the state is needed (e.g., `ErrValidationFailed` has a poor context of what's exactly validated and failed).
2. When the error **is an implementation detail** (DB, or internal service errors).
3. When a package would export 10+ error variables (consider grouping errors via types).

## Error structs

```go
// OperationFailedError is a custom type, that contains additional context
type OperationFailedError struct {
    RetryCount int
    Err        error // Original error
}

var _ error = (*OperationFailedError)(nil)

func ErrOperationFailed(retryCount int, err error) *OperationFailedError {
    return &OperationFailedError{
        RetryCount: retryCount,
        Err:        err,
    }
}

func (e *OperationFailedError) Error() string {
    return fmt.Sprintf("op failed: (retries: %d): %v", e.RetryCount, e.Err)
}

func (e *OperationFailedError) Unwrap() error { return e.Err }

// Usage example
func DoComplexOperation() error {
    err := someInternalCall()

    const retryLimit = 3
    for i := range retryLimit { // since 1.22 this code works perfectly
        if err == nil {
            return nil
        }
        // Some logic of retries...
    }

    return &OperationFailedError{
        RetryCount: retryLimit,
        Err:        err,
    }
}
```

**Required rules:**

1. A custom error type is **always, without exception, a pointer to a struct**;
2. Naming **must** end with the `Error` suffix (e.g., `ValidationError`, `NotFoundError`). The constructor function is prefixed with `Err` and must align with the type name (e.g., `ErrValidation() *ValidationError`, `ErrNotFound() *NotFoundError`);
3. Every error type **MUST** have a constructor function that accepts all required parameters and returns a pointer (`*YourCustomError`). Options are allowed, but the return type still must be always  `*YourCustomError`;
4. parameters may include only primitive types + one error (or a slice of errors). Allowed:
    1. `int`, `string`, `bool`, `float64`, `time.Time`, `struct{ID int}`
    2. `[]T`, `map[K]V`, (where `T`, `K`, `V` — primitive types only)
    3. `error`, `[]error` (but only one of them! Not both), name of parameter is only `Err/Errs` (or `err/errs`, depends on exporting rules)
    4. Pointers are acceptable if fields are optional,
5. If there is a field of type `error`, you **MUST** implement `Unwrap() error` returning that field; if there is a `[]error`, implement `Unwrap() []error`.

**Should implement:**

- Implement `fmt.Formatter` for custom formatting via `fmt.Printf("%+v", err)`;
- Provide an `example_test.go` showing when the error makes sense and how to handle it;

**Allowed, but only if truly necessary:**

- `Is(target error) bool`, but only if the type is tightly bound to a specific sentinel (e.g., `ErrNotFound` for `RecordNotFoundError`). In other cases it’s strongly discouraged — this introduces “runtime magic” and makes code harder to reason about. If implemented, it must be crystal clear and thoroughly documented.

## Wrapping Errors

`fmt.Errorf` follows roughly the same guidelines as sentinel errors, but adds wrapping via `%w`.

Simple error usage rules usually apply to wrapping because most of the time heavy context isn’t required:

- In 90% of cases: `fmt.Errorf("description: %w", err)` is enough.
- In 10% of cases you add context (resource ID, username, etc.). For example, `fs.PathError` stores the path, the operation, and the underlying error.

### Adding context in Errorf

It’s **acceptable** (yet not recommended) to wrap an error with contextual description:

```go
if err != nil {
    return fmt.Errorf("failed to process user %d: %w", userID, err)
}
```

However, if you see many repeated textual templates, consider introducing a custom error type—this is a strong signal the error carries valuable context you may later want to extract programmatically.

## Error assertions and extraction

### errors.Is() - for comparing sentinel errors

```go
if errors.Is(err, ErrNotFound) {
    // some handling
}
```

**Only compare sentinel errors**: the target should be a predefined exported variable, not a custom struct type instance.

### `errors.As()` - for data extraction from custom error types

Since custom errors are always pointer structs, you can use the pattern `e := new(YourErrorType)` — this makes the code clearer and easier to maintain.

```go
if e := new(ValidationError); errors.As(err, &e) {
    log.Printf("Field: %s", e.Field)
}
```

Instead of `e` you can choose any variable name (`valErr`, `ve`, or any name you like an think that it fits to the context), but it must be **ANYTHING EXCEPT `err`** (to avoid variable shadowing). Linters generally don’t catch accidental overwrites of `err`, which can break handler logic.

## Conclusion

Errors are an intuitive part of Go, yet really hard to form it into strict rules without limiting flexibility. Good error implementations come with experience and with understanding which failures are most frequent in your project — and especially how they’re handled. Perfection isn’t required; following the baseline rules will make your code more maintainable and easier for other developers to work with.
