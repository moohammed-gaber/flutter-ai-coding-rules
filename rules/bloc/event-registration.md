# Bloc Event Registration

**AI Code Review · Episode 01**

Register events explicitly so each handler has a clear responsibility and an intentional concurrency policy.

## Copy-ready rule

Copy the following block into `AGENTS.md` or `rules.md`:

```md
## Bloc Event Registration

- Register each concrete Bloc event with its own
  `on<ConcreteEvent>(handler)` exactly once.
- Do not use a base-event handler with `if/else`, `switch`,
  or runtime type checks to dispatch different event types.
- Keep each handler focused on its corresponding event.
- Choose an event transformer based on the required behavior:
  - `restartable()` when only the latest event should remain active.
  - `droppable()` when new events should be ignored while processing.
  - `sequential()` when events must be processed one at a time in order.
  - `concurrent()` when overlapping processing is appropriate.
- Do not apply the same transformer to every event automatically.
- Transformers apply per event registration; do not assume they
  coordinate different event types.
- Await asynchronous work that belongs to the handler.
- Follow these rules when creating or updating Blocs.
```

## Wrong

This constructor routes multiple event types through one base-event handler:

```dart
SearchBloc(this.repository) : super(const SearchState()) {
  on<SearchEvent>((event, emit) async {
    if (event is SearchQueryChanged) {
      await _onQueryChanged(event, emit);
    } else if (event is SearchNextPageRequested) {
      await _onNextPageRequested(event, emit);
    }
  });
}
```

Adding an event now requires changing a central dispatcher. The registration also hides the different processing needs of typing a query and requesting another page.

## Right

Register each concrete event with its own handler:

```dart
import 'package:bloc/bloc.dart';
import 'package:bloc_concurrency/bloc_concurrency.dart';

// Constructor excerpt; event, state, repository, and handler
// definitions belong to the application's search feature.
SearchBloc(this.repository) : super(const SearchState()) {
  on<SearchQueryChanged>(
    _onQueryChanged,
    transformer: restartable(),
  );
  on<SearchNextPageRequested>(
    _onNextPageRequested,
    transformer: droppable(),
  );
}
```

The query handler favors the latest input. The pagination handler ignores repeated requests while its current request is running. Add `bloc_concurrency` as a project dependency to use these transformers.

## Choose the behavior deliberately

| Transformer | Suitable when | Example |
| --- | --- | --- |
| `restartable()` | New input supersedes previous handler work | Search query changes |
| `droppable()` | Extra requests during active work can be discarded | Repeated load-more taps |
| `sequential()` | Every event must finish before the next starts | Ordered local mutations |
| `concurrent()` | Operations can safely overlap | Independent work with safe state updates |

## Concurrency boundaries

Bloc processes events concurrently by default unless you configure another transformer. Make that behavior intentional during review.

Each registration has its own processing boundary. Two different event types using `sequential()` can still overlap. In the search example, a query change can overlap a page request; handlers must guard against stale results, for example by checking a query revision before emitting.

`restartable()` cancels the previous handler, but does not automatically abort an underlying HTTP request or undo a side effect. Use cancellation support from your client when required. It also does not debounce input.

This rule is a project convention, not a Bloc API restriction: Bloc supports base-event registrations. Avoid registering both a base event and its concrete subtypes for the same work, because a matching event can reach both handlers.

## Review checklist

- Does every concrete event have one explicit registration?
- Does each handler handle only its corresponding event?
- Does the transformer match the feature's behavior?
- Are shared state and cross-event races handled?
- Is asynchronous handler work awaited?

## References

- [Bloc migration guide: event registration, defaults, and independent handlers](https://bloclibrary.dev/migration/)
- [Official bloc_concurrency documentation](https://pub.dev/packages/bloc_concurrency)

[Back to the series](../../README.md)
