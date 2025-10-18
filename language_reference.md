# Language Reference

## Variables

### Affinity and Placement

All variable declarations carry both an affinity and a placement even when the declaration omits those clauses. The compiler automatically assigns omitted values an affinity of `application` and stores the variable in the application's heap defined by `y.variables.placements.application_heap`.

Declaring an explicit affinity still follows the usual syntax, and the following keywords are now recognized:

- `application` — the default application-wide scope.
- `per application` — an explicit alias for the application affinity, useful when you want to emphasize that a variable is shared across the entire application.

When the declaration's affinity is either `application` or `per application`, omitting a placement clause continues to place the variable in `y.variables.placements.application_heap`.

### Examples

```text
var cache_size: u32;                   // affinity application, placement y.variables.placements.application_heap
var request_counter: u64 affinity per application;
```

The second example uses the explicit `per application` affinity, but both declarations resolve to the same placement and lifecycle.
