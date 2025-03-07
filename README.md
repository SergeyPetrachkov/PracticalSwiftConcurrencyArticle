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

Let's add a bit more context to it.

### The 4 don'ts of Swift Concurrency
Let's take a look at this code. It may not be perfect (far from it!), but it's the perfect illustration of **the 4 don'ts of Swift Concurrency**:

```Swift
final class MyViewModel {
// ...

    func viewDidLoad() {
       // ...
       task = Task { @MainActor in
          let result = try await longOperation()
          doSomething(with: result)
       }
    }
}
```


1) **Don't be deceived by selfless tasks** - we don't see any `self` keyword inside of the Task. But `self` will be retained implicitly.
2) **Don't believe in try without catch** - this code will compile just fine. We call a throwing function and we don't catch any errors. So, in this case, if the longOperations throws an error, we won't get our result. And we won't get the error. It will be "silently suppressed". Task can accept throwing closures, so be careful and don't forget to catch your errors ;)
   * **Tip!** It's nice to have Tasks that are one-liners. No lengthy logic. Just one function call. Then the compiler will help you not to miss this try-catch business and much more!
3) **Don't forget about the main thread** - in Swift Concurrency we don't operate threads directly, but it's important to keep in mind, that UI-related stuff must execute on the main thread. In Swift Concurrency we have Actors isolation mechanism that will make sure in compile time that certain code is executed on the main thread or outside of it. For the simplicity we can look at it as at a binary system: main/non-main thread. In this case inside the Task closure we have `@MainActor in`, which means that the execution **will start** on the main actor, when we `await` the `longOperation`, there will be a suspension point created and the work will be offloaded to one of the threads from the cooperative pool. Then, once the operation has finished, we'll get back to the main actor and execute `doSomething` on the main.
   * **Important!** This is fair only if the longOperation doesn't have MainActor isolation, and if there's no funky business like `DispatchQueue.main.async` somewhere there all of a sudden. So, **don't mix GCD with Swift Concurrency!**
   * **Important!** `await MainActor.run` is your last resort. It is only good if you need to build a bridge between the old world and the new Swift Concurrency. In your modern code try to avoid it. Rather try to design your entitities in a swift-concurrent way.
4) **Don't make your self weak** - I have to go through a lot of code in my day-to-day work. And I often see how people are tempted to put `[weak self]` inside the Task "just in case". And the next thing that they do is `guard let self else { return }`. We don't need that. There's no retain cycle there. Even if we hold the `task` in a property of `self` `Task` is not the one who holds the reference. The references are hold by Native.BuiltIn and once an executor finishes it's work (Task is finished/cancelled/error-thrown) the references will decrement and everything captured by it will be released (if nobody else is holding stuff).
   * **Important!** Rather than putting `[weak self]` there, think of how and when you're gonna **cancel** the task. If you don't trust me, trust the guys who invented [Swift Concurrency](https://github.com/swiftlang/swift/blob/c1ff2c339251c8adbdd63a08cb6ae45b339e15bd/stdlib/public/Concurrency/Task.swift#L80)

We can't properly cover this topic without touching the concept of isolation. We'll dive deeper into it later.

### Swift Concurrency. Where is the Concurrency?

In the [previous chapter](https://medium.com/@petrachkovsergey/the-4-donts-of-swift-concurrency-5614f39f8246) we've touched the topic of 4 don'ts of Swift Concurrency. We used the word Concurrency a lot. But how do we run our stuff **concurrently**?
We have a few options. In this article we'll take a look at those and say when we should use which one.

If we want to run things concurrently, we now have two instruments:
* async let syntax
* task groups (throwing/non-throwing/discarding)

One might say: we also have unstructured tasks. Well, yes, we do. But those are only good if we don't need to synchronize the execution and handle results. In most (if not all) cases we'd benefit from structured concurrency.

#### Async let

It is a special syntax sugar introduced in Swift 5.5 that allows use to start multiple async jobs in an async context and later await them as single values, or as a tuple, or as an array. You can find the detailed designs of async let [here](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0317-async-let.md).

`Async let` is very convenient when you need to run a finite number of jobs that return values of different type, and then use those values somewhere. Under the hood it spawns a new Task that inherits the local context and actor isolation. It's recommended that you await it, otherwise you might end up with some unexpected (not unexpected anymore!) behavior as described [here](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0317-async-let.md#async-let-error-propagation)

```Swift
// func boom() throws -> Int { throw Boom() }

func work() async -> Int {
  async let work: Int = boom()
  // never await work...
  return 0
  // implicitly: cancels work
  // implicitly: awaits work, discards errors
}
```

Okay, let's get practical.

Let's imagine that we are building a corporate tasks and builds tracker. We want to fetch details of a project and build versions that are associated with this project.
We can create a usecase like this:

```Swift
struct FetchProjectAndBuildsUseCase {

    private let versionsRepository: any VersionsRepository
    private let projectsRepository: any ProjectsRepository

    init(versionsRepository: any VersionsRepository, projectsRepository: any ProjectsRepository) {
        self.versionsRepository = versionsRepository
        self.projectsRepository = projectsRepository
    }

    func execute(projectId: Int) async throws -> (project: Project, versions: [Version]) {
        async let versions = versionsRepository.getVersions(request: .init(projectId: projectId))
        async let project = projectsRepository.getProject(by: projectId)

        let result = try await (project, versions)
        return result
    }
}
```

Things that you need to be aware of:
* the order of execution is not determined, we don't know if versions get downloaded before the project and vice versa. And we should not care about it! If we do, then we need to re-think how we organize this code.
* error handling is crucial here. If one of the tasks throws, then all the results will be lost.

#### Task Groups

Task groups are an essential part of Structured Concurrency. You can read the design doc [here](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md#task-groups-and-child-tasks). They can be [non-throwing, throwing](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md#task-groups), and [discarding](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0381-task-group-discard-results.md).

Task groups are perfect when you need to execute potentially infinite number of operations that return the same result type (but the usecases are not limited to just that).

You may be tempted to use Swift Concurrency and task groups to load images in your media sync process or process media files. But be careful with that. If you use `UIImage`, you'll get RAM spikes because of how images are presented in memory. And you may also starve the threads in the cooperative pool. The number of the threads is fixed. So, **swift concurrency is not the best candidate for long and expensive operations**. [More can be found here](https://swiftrocks.com/how-async-await-works-internally-in-swift) and [here](https://github.com/apple-oss-distributions/libdispatch).

Let's get back to building our corporate task tracker. Let's say we need to download ticket details that are associated with a build.

1) We create a TaskGroup with a special function `withThrowingTaskGroup`.
2) We specify a return type of a **single** task within the group. It's gonna be an optional `Ticket`, because I don't care about the tickets that failed to decode for example (unless it's a network issue).
3) For each of the tickets I add a new task to the group by `group.addTask`. I try to load ticket details. If it's a network problem, I **throw a CancellationError which cancells the whole task group**. If it's a different issue I just print an error because I'm lazy ;)
4) Then I await the tasks via `for await` of `AsyncSequence`
5) And then I return the array of tickets.

```Swift
struct FetchTicketsUseCase {

    private let ticketsRepository: any TicketsRepository

    init(ticketsRepository: any TicketsRepository) {
        self.ticketsRepository = ticketsRepository
    }

    func execute(for version: Version) async throws -> [Ticket] {
        try await withThrowingTaskGroup(of: Ticket?.self) { group in
            for ticketKey in version.associatedTicketKeys {
                group.addTask {
                    do {
                        return try await ticketsRepository.getTicket(key: ticketKey)
                    } catch is URLError { // for the simplicity I use this to indicate the network error, but of course it's not 100% valid
                        throw CancellationError()
                    } catch {
                        print(error) // I'm lazy and I don't care about the errors :) 
                        return nil
                    }
                }
            }
            var tickets: [Ticket] = []
            for try await ticket in group {
                if let ticket {
                    tickets.append(ticket)
                }
            }
            return tickets
        }
    }
}
```

Important things to keep in mind:
* error handling is important, catch your errors or you'll lose the progress
* the order of execution is not determined: if we start tasks in this order: 1,2,3,4,5 we may get the results in any random order imaginable, so if you need your results sorted in the same order as you started the tasks, you have to do it yourself
* no child task can outlive it's parent, that's the beauty of structured concurrency. All tasks will be finished before you get back to the awaited task group.

P.S. 

Remember how people would ask questions like this during iOS interviews?

You need to execute multiple async operations and show the results in the same order as you enqueued those operations. How would you do that?

And then you'd start talking about how DispatchGroups are cool and stuff. 

Now, let's do it with swift concurrency.

```Swift
func loadSongsInfo(request: Request) async -> [SongInfo] {
        await withTaskGroup(of: (Int, SongInfo?).self) { group in
            for enumeratedSearcher in otherStreamingsSearchers.enumerated() {
                group.addTask {
                    do {
                        return try await (enumeratedSearcher.offset, enumeratedSearcher.element.search(request: request))
                    } catch {
                        return (enumeratedSearcher.offset, nil)
                    }
                }
            }
            var results: [(Int, SongInfo?)] = []
            for await result in group {
                results.append(result)
            }
            return results.sorted(by: { $0.0 < $1.0 }).compactMap { $0.1 }
        }
}
```

I hope you enjoyed this chapter. In the upcoming chapters we're finally going to talk about actors and we'll try to find an entry point into Concurrency in our iOS projects :)


### Actors, isolation, sendability: practical tips for iOS developers

With the new concurrency model, we've got quite a few new keywords and a bunch of concepts that we need to learn and adopt in our daily work. But it also brings fresh ideas to the concepts we've all been taught about one way or another. Let's go.

#### Thread safety.
Old but gold. The goal of Swift concurrency is to provide **compile-time safety**. In other words, the code that can cause data races (not race conditions) should not compile. It's achieved via actors model and the concept of sendability.

#### Sendability.
It's one of the most confusing parts for many developers. Let's not be too boring and get the practical side of it. When we say something is _sendable_, we mean that it's safe to pass that something between concurrency contexts (from one Task to another simply put). `Sendable` is just an empty protocol that acts as a marker for a compiler. When compiler sees a type that conforms to Sendable, it goes like this:

Aha, this type conforms to Sendable, let me do a quick check:
1) Is it an actor? If yes, then it's sendable by default. The nature of actors is that they are deadlock-free, datarace-safe. If it's not an actor, then we go further.
2) Is it a value type? (Struct or enum)? Value types are sendable, but sometimes you need to add the `: Sendable` explicitly, for example if you're dealing with modular apps and you have public structs defined in one module and you want to use them from another module. [More cases here](https://developer.apple.com/documentation/swift/sendable#Sendable-Structures-and-Enumerations)
3) Is it a class? To satisfy the sendability requirements, the class must be final, contain no mutable shared state, contain only sendable properties, have no superclass (or only NSObject). There's one more way: if the class is isolated to the main actor. These classes can have stored properties that are mutable and nonsendable, because MainActor will take case of the thread-safety (we'll talk about it later). [More here](https://developer.apple.com/documentation/swift/sendable#Sendable-Classes)
4) None of the above? Well, mate, no. It's not Sendable.

But what if you know that the class is thread-safe and you need the compiler to shut up? Then you can use `@unchecked Sendable`. You can put it to your class and it will be excluded from further checks. 
When can we use the unchecked sendable? **Ideally**, only when we are certain that the class is data-race free and we've achieved that via Locks or DispatchQueues or Atomics or any other way.
But sometimes we may use it in tests, when generating mocks for our interfaces. Most of the times we don't really need thread safe mocks, do we? (Unless we test multithreaded code).

When does Apple use unchecked Sendable?

You can check out [my article](https://medium.com/@petrachkovsergey/mutex-in-swift-lang-8d241db2e543) about Mutex<T> in Swift language. TL;DR it's a wrapper over os_unfair_lock (for iOS and macOS) and the wrapper is marked as unchecked Sendable :) 

What else can be Sendable?

Closures and functions. Instead of conforming to the Sendable protocol, you mark sendable functions and closures with the @Sendable attribute. Any values that the function or closure captures must be sendable. In addition, sendable closures must use only by-value captures, and the captured values must be of a sendable type.

Ok, if we want to use Swift Concurrency, we can't skip the concept of sendability. Mostly because it's a building brick of the concept of Isolation.

#### Isolation

During WWDC speakers talked about isolation with an analogy of Sea of Concurrency and Islands of isolation. You can check out [this wwdc video](https://developer.apple.com/videos/play/wwdc2022/110351/). And for a long time I was pissed off with this analogy, and wanted to come up with my own that would be clearer :D I failed, of course.
During workshops I'm trying to show the synchronization and isolation with some hand gestures, but it's also not that accurate and clear.

Let's try and formulate this without analogies :) Let's just embrace the "new" jargon.

**Data isolation**

Swift Concurrency is all about compile-time thread safety. The compiler must be able to understand and verify the safety of all mutable state. The mechanism is called data isolation. Data isolation guarantees mutually exclusive access to a mutable state. It is a form of synchronization, conceptually similar to a lock. But unlike a lock, everything happens at compile-time. 
We work with data isolation either statically or dynamically. Static is unaffected in the runtime. Whenever we put some keywords and/or annotations to our code, it's static.
Sometimes we can't do that. For example when we're working with ObjC or any other non-Swift code. And also when we're dealing with the code that we don't own and that doesn't obey the rules of the data isolation. Then it's dynamic isolation's turn. You can read more [here](https://www.swift.org/migration/documentation/swift-6-concurrency-migration-guide/incrementaladoption/#Dynamic-Isolation)

What's more important is to talk about isolation domains.

**Isolation domains**

Data isolation is _a mechanism_. Isolation domain is a _unit of isolation._ 
All our code belongs to isolation domains.

1) non-isolated
   ```Swift
   struct Ticket {
      let id: UUID
      let title: String
      let description: String?
   }

   final class TicketsRepositoryImpl: TicketsRepository {
      func getTickets(request: TicketsRequest) async -> TicketsResponse {}
   }
   ```
2) isolated to an actor
   ```Swift
   actor TicketsCachingRepository: TicketsRepository {
       private var cachedTickets: [Ticket] = []
       func getTickets(request: TicketsRequest) async -> TicketsResponse {}

       nonisolated func funFact() -> String {
           "I can't access cachedTickets from here, I can do some other stuff like return this string :)"
       }
   }
   ```  
3) isolated to a global actor
   ```Swift
   @MainActor
   final class TicketsViewModel {
      @Published var viewState: TicketsViewState = .initial
   }
   ```

A Task always has an isolation domain. They can be isolated to an actor instance, a global actor, or could be non-isolated. This isolation can be established manually, but can also be inherited automatically based on context. Task isolation, just like all other Swift code, determines what mutable state is accessible.
Tasks can run both synchronous and asynchronous code. Regardless of the structure and how many tasks are involved, functions in the same isolation domain cannot run concurrently with each other. There will only ever be one task running synchronous code for any given isolation domain.

Going through isolation domains can only be done _via awaiting_. We'll talk about it in a bit.

**Isolation domains can be inherited and inferred.**

So, if you open UIKit, you'll see that everything UI-related is isolated to the MainActor. All your UIView, UIViewController, UIWindow, etc. So, if you inherit from any of that type, you'll inherit the MainActor isolation.

You can read more about data isolation and isolation domains [here](https://www.swift.org/migration/documentation/swift-6-concurrency-migration-guide/dataracesafety/)

#### Actors

Actor is a new construct in the Swift language. Actors can also be global. There's no magic behind the actos. Well, maybe a little bit.

If you look into the Swift lang repo, you'll find that there's an Actor protocol:

```Swift
public protocol Actor: AnyObject, Sendable {
    nonisolated var unownedExecutor: UnownedSerialExecutor { get }
}
```
And after some "magic", when you write:

```Swift
actor MyActor {}
```

to the compiler it's something like:

```Swift

final class MyActor: Actor {}
```

And global actor is just an actor that conforms to `GlobalActor` protocol and in result has a singleton:

```Swift
@globalActor public actor MainActor: GlobalActor {
  public static let shared = MainActor()
  // ....
}
```

You can find more under-the-hood stuff [here](https://swiftrocks.com/how-async-await-works-internally-in-swift)

#### Practical isolation

MainActor is a special one, it's probably the most important actor for iOS and macOS developers. It tries to make sure that the code that is isolated to the main actor runs on the main thread. Why did I say "tries to"? Well, because we can still break things by mixing up GCD with Swift Concurrency.
So, don't! :) 

MainActor and rules of isolation make impact on the whole mobile development: on how we organise our modules, on how we split the app into layers, and on how we generally architect our apps.

As we mentioned above, all communications between different isolation domains can only be done via `await`. We create a suspension point, run some code, then jump back and synchronize contexts. But **within** the same domain we can execute our synchronous code without having to await anything.
One ViewController can talk to another ViewController without awaiting anything, because they have the same MainActor isolation.

Let it sink in. 

Two entities isolated to the same actor can talk to each other without having to await anything (of course if they use sync functions).
Take a look at this code snippet. Don't mind the unpretty `weak var viewController`, it's a shortcut to show the problem.

```Swift
// This is nonisolated
final class MyNonIsolatedViewModel {

	weak var viewController: MyViewController?

	func start() {
		// compilation error: Call to main actor-isolated method 'displayData()' in a synchronous nonisolated context
		viewController?.displayData()
	}
}

// This is explicitly isolated to the MainActor
@MainActor
final class MyIsolatedViewModel {

	weak var viewController: MyViewController?

	func start() {
		// compiles just fine, because we don't cross isolation domains
		viewController?.displayData()
	}
}

// This inherits MainActor isolation from UIViewController
final class MyViewController: UIViewController {

	let isolatedViewModel = MyIsolatedViewModel()
	let nonIsolatedViewModel = MyNonIsolatedViewModel()

	override func viewDidLoad() {
		super.viewDidLoad()
		isolatedViewModel.viewController = self
		nonIsolatedViewModel.viewController = self
		nonIsolatedViewModel.start()
	}

	// This is also isolated to the main actor
	func displayData() {}
}
```

Ideally, we want our entities to talk to each other without many suspension points. We only want to await stuff when it's absolutely necessary.
So, try to think about where your code belongs.

#### Isolation and popular entities

Let's go through a few popular entities:
1. Coordinator/Router/Navigator. It's an entity that is responsible for navigation. Showing one screen from another screen. This can only happen from the main thread. This already happens on the main thread, otherwise there's a runtime crash. But Swift Concurrency model requires explicitness. So, it makes sense to isolate your coordinators/routers/navigators to the MainActor. Just put it on top of the class.
2. ViewController/UIKit View/SwiftUI View - already isolated to the main actor for us by Apple. Nothing to do from our side :)
3. ViewModel from MVVM. What does it do? It receives signals from View layer (isolated to the main actor!), it publishes changes back to the View layer. So, it makes sense to isolate your ViewModel to the MainActor. It will also allow the VM to be Sendable (it will probably have some mutable state, so the MainActor comes in handy). What else does the VM do? It talks to services/repositories/usecases/providers/whatever.
4. Service/Repository/Usecase/Provider/Worker - what are these? Well, in many projects people use different names, but let's just say these are the entities that do something business logic-related. Often it's an execution of some async code. In GCD world we often create queues to handle work there. In Swift Concurrency world we don't need it. Just leave those entities nonisolated or _actor-agnostic_ and keep the interface clean and async.
5. VIP stack. Interactor and Presenter. Interactor and Presenter are serving on purpose: split responsibilities of an object like ViewModel into smaller chunks. So, Interactor receives signals from View, asks Workers to do something, get the data, pushes it to Presenter, and then Presenter maps it to UI entities and pushes then to View. Long story short: Interactor and Presenter belong to MainActor. But of course there may be some performance optimisations. If we don't want to map things on the main thread, we'll benefit from `nonisolated` functions within Presenter. Just keep them _pure_ and _async_.
6. Delegates. Well, delegates have been with us since ObjC era. And they look kinda redundant in the world of modern Swift, when we have closures, publishers, async streams and stuff. But it's still something many developers use. Okay. Think about who's your delegate talking to. If that's a delegate talking to a coordinator, you know the answer. Put MainActor on top of that protocol. UI entities must do their stuff synchronously on the main thread. No other way. A delegate of a service? (Well, maybe it's time for refactoring) Keep it nonisolated.

And during these transitions you'll get many challenges of the added constraints. So, think about how many layers you want to have in your apps architecture. Think about isolation domains. Think about sendability. Think about modularisation.

Talk is cheap. Show me the code.

Remember we were building a corporate issues tracker?
Let's build a screen that will show an internal release version with the associated Jira ticket keys.

We'll need a usecase that will download the JIRA tickets in parallel.
* This usecase is Sendable, because it doesn't contain any mutable state. All the components of the usecase are also Sendable.
* This usecase is non-isolated to any Actor. It's actor-agnostic.
```Swift
protocol FetchTicketsUseCaseProtocol: Sendable {
    func execute(for version: Version) async throws -> [Ticket]
}

struct FetchTicketsUseCase {
    private let ticketsRepository: any TicketsRepository

    init(ticketsRepository: any TicketsRepository) {
        self.ticketsRepository = ticketsRepository
    }

    func execute(for version: Version) async throws -> [Ticket] {
        try await withThrowingTaskGroup(of: Ticket?.self) { group in
            for ticketKey in version.associatedTicketKeys {
                group.addTask {
                    do {
                        return try await ticketsRepository.getTicket(key: ticketKey)
                    } catch is URLError { // for the simplicity I use this to indicate the network error, but of course it's not 100% valid
                        throw CancellationError()
                    } catch {
                        print(error) // I'm lazy and I don't care about the errors :) 
                        return nil
                    }
                }
            }
            var tickets: [Ticket] = []
            for try await ticket in group {
                if let ticket {
                    tickets.append(ticket)
                }
            }
            return tickets
        }
    }
}
```

We'll also need a ViewModel.
* The ViewModel will be isolated to the main actor
* The ViewModel will handle the entry point to the concurrency (and we'll discuss it more in the next article)
* The ViewModel will talk to our usecase, load the tickets, map it to the state and publish it to the View layer (wow, too many responsibilities? Maybe yes, maybe no).

```Swift
@MainActor
final class VersionDetailsViewModel: ObservableObject {

    // MARK: - Injectables
    private let version: Version
    private let fetchTicketsUsecase: FetchTicketsUseCaseProtocol

    // MARK: - State
    @Published private(set) var state: State
    private var currentTask: Task<Void, Never>?

    // MARK: - Init
    init(version: Version, fetchTicketsUsecase: FetchTicketsUseCaseProtocol) {
        self.version = version
        self.fetchTicketsUsecase = fetchTicketsUsecase
        self.state = .loading(VersionDetailsLoadingView.State(version: version))
    }

    // MARK: - Sync interface controlled by us
    // This is isolated to the main actor
    func send(_ action: Action) {
        switch action {
        case .onAppear, .onReload:
            currentTask?.cancel()
            currentTask = Task {
                await fetchData()
            }
        case .onDisappear:
            currentTask?.cancel()
        }
    }

    // MARK: - Private logic

    private func fetchData() async {
        do {
            // suspension point created, we're jumping off the Main Thread. And since our Usecase is non-isolated to anything, we're sure of it.
            let tickets = try await fetchTicketsUsecase.execute(for: version)
            // we're back
            if !Task.isCancelled {
                // this will be published on the main thread
                state = .loaded(.init(version: version, tickets: tickets))
            } else {
                print("VM is stopped. State won't be published")
            }
        } catch {
            state = .failed(State.ErrorViewModel(message: error.localizedDescription))
        }
    }
}
```

Let's spice things up a bit. Let's say I want my tickets to be cached. I don't want to download them every time I open the screen.
* Does it mean that our Repo/Usecase/Provider/Service/Worker layer becomes stateful? Well, yes. 
* Do we need isolation to protect it from data races? Yes!
* Do we need main actor? No!

This actor also handles the actor re-entrancy problem, so if we're already loading something, we don't start a new loading task, we're awaiting the existing one.

```Swift
actor TicketsCacheActor: TicketsRepository {

    private let repository: TicketsRepository
    private var tickets: [String: Task<Ticket, Error>] = [:]

    init(repository: TicketsRepository) {
        self.repository = repository
    }

    func getTickets() async throws -> [CorporateTestflightDomain.Ticket] {
        try await repository.getTickets()
    }

    func getTicket(key: String) async throws -> CorporateTestflightDomain.Ticket {
        print("Entering actor for \(key)")
        if let ticketTask = tickets[key] {
            print("Return cached value by \(key)")
            do {
                let value = try await ticketTask.value
                return value
            } catch {
                print("Previously cached task for the ticket \(key) returned error \(error). Will start a new task.")
            }
        }
        let newTicketTask = fetchTicketTask(key: key)
        print("Returning value by fetching \(key)")
        return try await newTicketTask.value
    }

    private func fetchTicketTask(key: String) -> Task<Ticket, Error> {
        print("Fetching value by \(key)")
        let ticketTask = Task { try await repository.getTicket(key: key) }
        print("Caching value by \(key)")
        tickets[key] = ticketTask
        return ticketTask
    }
}
```

We just inject this actor into our usecase, and we're good to go :) Clean architecture :D 

```Swift
FetchTicketsUseCase(ticketsRepository: TicketsCacheActor(TicketsRepositoryImpl(api: api))))
```

#### Where to go from here

These are the simple basics of the isolation. And there's more to it. But I hope this can help you out to start the journey and face Swift Concurrency challenges with more context.
Next time we'll discuss what is the best place for an entry point to Swift Concurrency. We'll also talk about Dependency Injection, containers, popular frameworks, modular apps and Swift Concurrency, and other challenges of the real world like bridging async-await with the old code, using unsafe Sendables, nonisolated functions, etc.


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

Remember I mentioned a paradigm shift in the beginning? Here we go. 
If you choose your View layer to be an entry point, then it may make sense to get your ViewModels or Interactors an async interface. (Note: I'm omitting any kind of actor-isolation code deliberately.)
What does it mean to us as developers? It means that we enetered the Concurrency really early and we can benefit from async-awaiting our way up (or down, depending on how you look at it) the scene stack. If our networking code, or repositories, or image processors, or any other things that are usually async use Swift Concurrency, then we can simply those async entities. No need to create unstructured tasks anywhere else. And a nice bonus with SwiftUI, the tasks will be cancelled automatically by SwiftUI engine when the view goes off the hierarchy. This will cover 90% of our requirements.

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
