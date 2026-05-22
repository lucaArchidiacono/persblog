---
layout: page
title: SwiftUI routing
permalink: /swiftui-routing
description: How to navigate in SwiftUI
---

You might not like to hear this, but SwiftUI is more like React & React Routing than any other mobile UI framework.

Before I show you the code, which you can easily just copy paste and use in your codebase, let me show you what I mean with the previous sentence.

Let's say you build a view.

Inside that view you have a button.

And when the user taps that button, you want to navigate to another view.

```swift
import SwiftUI

struct HomeView: View {
    var body: some View {
        VStack {
            Text("Hello World")

            // I am inside a view.
            // From here I want to navigate to another view.
            Button("Settings") {
                print("I want to see Settings")
            }
        }
    }
}
```

If this were React, you would probably think about it like this:

```tsx
function HomeView() {
    const router = useRouter();

    return (
        <div>
            <p>Hello World</p>

            {/* I am inside a view. */}
            {/* From here I want to navigate somewhere. */}
            <button onClick={() => router.push("/settings")}>
                Settings
            </button>
        </div>
    );
}
```

Of course the router also needs to know what `"/settings"` means.

So somewhere in your app you have a route registry.

In React this could look like this:

```tsx
const routes = [
    {
        path: "/",
        element: <HomeView />,
    },
    {
        path: "/settings",
        element: <SettingsView />,
    },
];
```

Then your view can navigate to one of those registered routes.

```tsx
function HomeView() {
    const router = useRouter();

    return (
        <button onClick={() => router.push("/settings")}>
            Settings
        </button>
    );
}
```

Some React routers call this `navigate("/settings")`.
Some call this `router.push("/settings")`.
The idea is the same.

The view does not create `SettingsView` directly.
It only says where it wants to go.

That is exactly the mental model I want in SwiftUI.

```swift
router.push(.settings)
```

Or if you want to go back:

```swift
router.pop()
```

The main difference is that in Swift you do not need to navigate with strings. You can navigate with types.

For me the setup looks like this:
- `Destination` is for normal stack navigation.
- `Page` is for sheets and full screen covers.
- `Router` stores the current path and presented page.
- `AppRegistry` knows which SwiftUI view belongs to which route.

## Declare your routes

First declare all views your app can navigate to.

Keep this small in the beginning. You can always add more cases later.

```swift
import SwiftUI

enum Destination: Hashable {
    case settings
}

enum Page: Hashable, Identifiable {
    case profile

    var id: Self { self }
}
```

This is already nicer than string based routing. You cannot accidentally write `"/setings"` and wonder why nothing works.

## Build a generic router

The router only stores navigation state.

`D` is the type used for `NavigationStack`.
`P` is the type used for sheets and full screen covers.

```swift
import SwiftUI

@MainActor
@Observable
final class Router<D: Hashable, P: Hashable & Identifiable> {
    var path: [D] = []
    var sheet: P?
    var fullScreen: P?

    var isPresenting: Bool {
        sheet != nil || fullScreen != nil
    }

    var isRoot: Bool {
        path.isEmpty && !isPresenting
    }

    func push(_ destination: D) {
        path.append(destination)
    }

    func present(_ page: P) {
        sheet = page
    }

    func presentFullScreen(_ page: P) {
        fullScreen = page
    }

    func pop() {
        guard !path.isEmpty else { return }
        path.removeLast()
    }

    func pop(last count: Int) {
        guard !path.isEmpty else { return }
        path.removeLast(min(count, path.count))
    }

    func dismiss() {
        sheet = nil
        fullScreen = nil
    }
}
```

That is almost everything.

If you compare it to React, this is your `router.push`, `router.back`, and modal state in one small object.

## Register your views

Now the app needs to know which view belongs to which route.

This is the same idea as a route registry in React.

```swift
import SwiftUI

extension View {
    func withDestination() -> some View {
        navigationDestination(for: Destination.self) { destination in
            switch destination {
            case .settings:
                SettingsView()
            }
        }
    }

    func withSheet(using page: Binding<Page?>) -> some View {
        sheet(item: page) { page in
            PageHost(page: page)
        }
    }

    func withFullScreen(using page: Binding<Page?>) -> some View {
        fullScreenCover(item: page) { page in
            PageHost(page: page)
        }
    }
}

private struct PageHost: View {
    let page: Page

    var body: some View {
        switch page {
        case .profile:
            NavigationContainer {
                ProfileView()
            }
        }
    }
}
```

In a real app this file grows over time. That is fine.

The nice thing is that all available routes are declared in one place --> `DECLARATIVE`

## Put it into SwiftUI

Now wrap your screen in a small `NavigationContainer`.

```swift
import SwiftUI

struct NavigationContainer<Content: View>: View {
    @Bindable var router: Router<Destination, Page>
    let content: Content

    init(
        router: Router<Destination, Page> = Router(),
        @ViewBuilder content: () -> Content
    ) {
        self.router = router
        self.content = content()
    }

    var body: some View {
        NavigationStack(path: $router.path) {
            content
                .withDestination()
        }
        .withSheet(using: $router.sheet)
        .withFullScreen(using: $router.fullScreen)
        .environment(router)
    }
}
```

This is the place where SwiftUI and your router meet.

The `NavigationStack` reads the path. The sheet reads the optional page. The full screen cover reads the optional page.

## Use it from a view

Now your first example can navigate.

```swift
import SwiftUI

struct HomeView: View {
    @Environment(Router<Destination, Page>.self) private var router

    var body: some View {
        VStack {
            Text("Hello World")

            Button("Settings") {
                router.push(.settings)
            }

            Button("Profile") {
                router.present(.profile)
            }
        }
    }
}
```

And if the `SettingsView` wants to go back:

```swift
import SwiftUI

struct SettingsView: View {
    @Environment(Router<Destination, Page>.self) private var router

    var body: some View {
        Button("Back") {
            router.pop()
        }
    }
}
```

This is the React mental model again.

The view does not create the destination view. It only says where it wants to go.

## Share the router globally

For small apps one router is enough.

```swift
@MainActor
@Observable
final class NavigationManager {
    let router = Router<Destination, Page>()
}
```

Then inject it once.

```swift
@main
struct MyApp: App {
    @State private var navigationManager = NavigationManager()

    var body: some Scene {
        WindowGroup {
            NavigationContainer(router: navigationManager.router) {
                HomeView()
            }
        }
        .environment(navigationManager)
    }
}
```

For bigger apps I usually create one router per tab.

```swift
@MainActor
@Observable
final class NavigationManager {
    var homeRouter = Router<Destination, Page>()
    var settingsRouter = Router<Destination, Page>()
}
```

That way every tab owns its own stack, just like most users expect.

And that is basically it.

SwiftUI navigation becomes much easier once you stop thinking about pushing views and start thinking about pushing values.