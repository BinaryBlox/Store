# Store [![Swift](https://img.shields.io/badge/swift-5.1-orange.svg?style=flat)](#)
<img src="https://raw.githubusercontent.com/alexdrone/Dispatch/master/docs/dispatch_logo_small.png" width=300 alt="Dispatch" align=right />

Unidirectional, transactional, operation-based Store implementation for **Swift** (and **SwiftUI**).

## Overview

Store eschews MVC in favour of a unidirectional data flow. When a user interacts with a view, the view propagates an action through a central dispatcher, to the various stores that hold the application's data and business logic, which updates all of the views that are affected.

This works especially well with *SwiftUI*'s declarative programming style, which allows the store to send updates without specifying how to transition views between states.

- **Stores**: Holds the state of your application. You can have multiple stores for multiple domains of your app.
- **Actions**: You can only perform state changes through actions. Actions are small pieces of data (typically *enums* or *structs*) that describe a state change. By drastically limiting the way state can be mutated, your app becomes easier to understand and it gets easier to work with many collaborators.
- **Transaction**:  A single execution of an action.
- **Views**: A simple function of your state. This works especially well with *SwiftUI*'s declarative programming style.

### Store

Stores contain the application state and logic. Their role is somewhat similar to a model in a traditional MVC, but they manage the state of many objects — they do not represent a single record of data like ORM models do. More than simply managing a collection of ORM-style objects, stores manage the application state for a particular domain within the application.

This allows an action to result in an update to the state of the store. After the stores are updated, they notify the observers that their state has changed, so the views may query the new state and update themselves.

```swift
struct Counter: ModelType {
  var count = 0
}

let store = Store<Counter>()
```

### Action

An action represent an operation on the store.

It can be represented using an enum:

```swift
enum CounterAction: ActionType {
  case increase
  case decrease

  var identifier: String {
    switch self {
    case .increase: return "INCREASE"
    case .decrease: return "DECREASE"
    }
  }

  func perform(context: TransactionContext<Store<Counter>, Self>) {
    defer { context.operation.finish() }
    switch self {
    case .increase: context.store.updateModel { $0.count += 1 }
    case .decrease: context.store.updateModel { $0.count -= 1 }

    }
  }
}

```

Or a struct:

```swift
struct INCREASE: ActionType {
  let count: Int
  
  func perform(context: TransactionContext<Store<Counter>, Self>) {
    defer { context.operation.finish() }
    context.store.updateModel { $0.count += 1 }
  }
}
```

### Transaction

A transaction represent an excution of a given action.
The dispatcher can run transaction in three different modes: `async`, `sync`, and `mainThread`.
Additionally the trailing closure of the `run` method can be used to run a completion closure for the actions that have had run.

# Getting started

TL;DR

```swift
import SwiftUI
import Store

struct Counter: ModelType {
  var count = 0
}

enum CounterAction: ActionType {
  case increase(ammount: Int)
  case decrease(ammount: Int)

  var identifier: String {
    switch self {
    case .increase(_): return "INCREASE"
    case .decrease(_): return "DECREASE"
    }
  }

  func perform(context: TransactionContext<Store<Counter>, Self>) {
    defer {
      context.operation.finish()
    }
    switch self {
    case .increase(let ammount):
      context.store.updateModel { $0.count += ammount }
    case .decrease(let ammount):
      context.store.updateModel { $0.count -= ammount }
    }
  }
}

// MARK: - UI

struct ContentView : View {
  @EnvironmentObject var store: Store<Counter>
  var body: some View {
    Text("counter \(store.model.count)").tapAction {
      store.run(action: CounterAction.increase(ammount: 1))
    }
  }
}

// MARK: - Preview

#if DEBUG
struct ContentView_Previews : PreviewProvider {
    static var previews: some View {
        ContentView().environmentObject(Store<Counter>())
    }
}
#endif
```

### Middleware

Documentaton in progress...

# Advanced use

Dispatch takes advantage of *Operations* and *OperationQueues* and you can define complex dependencies between the operations that are going to be run on your store.

Also middleware support is available allowing you to quickly add some aspect-oriented feature to your design.

### Chaining actions

```swift
store.run(actions: [
  CounterAction.increase(ammount: 1),
  CounterAction.increase(ammount: 1),
  CounterAction.increase(ammount: 1),
]) { context in
  // Will be executed after all of the transactions are completed.
}

```
Actions can also be executed in a synchronous fashion.

```swift
store.run(action: CounterAction.increase(ammount: 1), strategy: .mainThread)
store.run(action: CounterAction.increase(ammount: 1), strategy: .sync)
```

### Tracking transaction state

Sometimes it's useful to track the state of a transaction (it might be useful to update the UI state to reflect that).

```swift
store.run(action: CounterAction.increase(ammount: 1)).$state.sink { state in
  switch(state) {
  case .pending: ...
  case .started: ...
  case .completed: ...
  }
}
```

### Dealing with errors

```swift
struct INCREASE: ActionType {
  let count: Int
  
  func perform(context: TransactionContext<Store<Counter>, Self>) {
    defer { context.operation.finish() }
    // The operation terminates here because an error has been raised in this dispatch group.
    guard !context.killOnGroupError() { else return }
    // Kill the transaction and set TransactionGroupError.lastError.
    guard store.model.count != 42 { context.kill(error: Error("Max count reach") }
    // Business as usual.  
    context.store.updateModel { $0.count += 1 }
  }
}
```

