# PracticalSwiftConcurrencyArticle
An article covering practical aspects of Swift Concurrency, common pitfalls, architectural shift and more.

**Licensing:**
I believe that the knowledge must be free for everybody, that's why I put it here. I will also publish it on Medium. But if for any reason they put a paywall there, everybody will be able to read it here.
© Sergei Petrachkov, 2024. All rights reserved. This article is protected under copyright law. Copying, redistribution, or adaptation of this content in any form is prohibited without prior written permission.

## Introduction to Swift Concurrency

Swift Concurrency was introduced to us along with Swift 5.5. The initial goal was to make the concurrency easier and reduce the steepness of the learning curve that GCD has. However, the new stuff has turned out to be way more complex and given birth to some huge misunderstanding accross the industry. But we'll get there.

Let's first talk about the swift concurrency as it was presented to us. Instead of closures or publishers we can now `await` functions and properties that have an `async` next to them.

```Swift
func longOperation() async -> Bool {
    // ...
    return true
}
// ...
let result = await longOperation()
```

Is it Swift Concurrency? Well, absolutely. Does this snippet have any value? Well, I don't think so.
In some sense Swift Concurrency reminds me of C# .NET, where I witnessed the introduction of async-await. And the common denominator back then was "Once you go async in one place, you can't go back".

**You cannot `await` anything outside of the concurrent context.** 
This is very important to embrace. 

**How do I enter the concurrent context?**
or 
**Where do I enter the concurrent context?**

These are the two questions that are often missed. The answers to these questions define the architectural shift in how we see iOS/macOS/watchOS/visionOS/whateverOS apps and the code we write.
But let's go step by step.

### How do I enter the concurrent context?
Well, for that you need to have a `Task`. You can read the implementation details [here](https://github.com/swiftlang/swift/blob/main/stdlib/public/Concurrency/Task.swift).
But in short: `Task` is a struct. This struct has a few versions of initialisers where you specify an `async` closure and a priority (if needed). The contents of this closure are your concurrent context. Inside this closure you can `await` async operations. You can also specify an actor isolation (we'll talk about it later). As in any closure, you can also capture some objects via capture groups. 
In some sense, `Task` initialiser is one of the most important things in the whole Swift Concurrency. It's an entry point for us - consumers of the framework. Then the job described in the closure gets dispatched to an associated executor. If you want to read more about the under-the-hood of Swift Concurrency I encourage you to:
1) Read this article: https://swiftrocks.com/how-async-await-works-internally-in-swift
2) Clone [Swift](https://github.com/swiftlang/swift/tree/main) and do you own research, it's highly entertaining.

But to keep it short and to avoid repeating what's already been described, we're not gonna dive deeper into that now.

So, practical answer to `how do I enter the concurrent context`:
We need to create a Task. In Swift when you create a Task, it starts running immediately (unlike in C#).

```Swift
Task {
    let result = await longOperation()
}
```

Now we're talking. We have created an unstructured Task, we've started it (well, the concurrency engine has started it). Everything inside the closure of the task is retained by an executor (it's a bit more complex than this, but let's just forget for a moment about all the details). Once the task is finished, everything is released.

Does this snippet look good to you?
To me, it doesn't. It's super abstract and it doesn't give me an idea of how I can use this task in my iOS project.

We can't properly cover this topic without touching the concept of isolation. We'll dive deeper into it later.
For now, let's find an entry point to the Concurrency for our iOS projects.

### Swift UI Entry point

If you're using SwiftUI and you're targeting recent iOS versions, then Apple have done the heavy lifting for you. 

Each and every SwiftUI view has a `.task {}` [modifier](https://developer.apple.com/documentation/swiftui/view/task(priority:_:)) or even [this one](https://developer.apple.com/documentation/swiftui/view/task(id:priority:_:)). This can be your entry point. And for the most of the apps it will be enough. 
```Swift
struct MyView: View {

  var body: some View {
     Text("Hello there")
       .task {
          // here you go
          let result = await longOperation()
       }
  }
}
```

Apple suggest using View layer of your app to be an entry point to the concurrency. And if you're using `task(id:priority:_:)` you get Tasks cancellation for free. 

### UIKit Entry point

But if you're using UIKit, the heavy lifting is on you. Depending on how you organize the presentation layer management, you need to choose which part of it will become the entry point. Is it going to be a ViewController (A View layer)? Or a ViewModel from MVVM, Interactor from VIP, Presenter from VIPER, or any other fancy word of your choosing.

It is important to align with your team(s) on how you see it. Some may say that the tests coverage of Tasks cancellation is an absolute must. Then moving it from View layer makes sense, because testing Views is usually more cumbersome than other layers.

### Paradigm shift

Remember I mentioned a paradigm shift in the beginning? Here we go. If you choose your View layer to be an entry point, then it may make sense to get your ViewModels or Interactors an async interface.

```Swift
@Observable
final class TasksViewModel {
    private(set) var items: [Item]
    // ...
    func start() async {
       items = await repository.fetchAll()
    }
}

struct TasksView: View {

  var viewModel: TasksViewModel

  var body: some View {
     List(viewModel.items) { item
        ItemRow(item)
     }
       .task {
          await viewModel.start()
       }
  }
}
```
