# Shipping *El Pelegrí*: a year of arguing with HealthKit, Lottie, and myself

> A technical retrospective of a small iOS walking app, told as the six problems I actually had to solve.

---

## Cold open: "12,000 steps, 0.0 km"

The bug report came in plain Catalan, from a beta tester with a Xiaomi SmartBand 10 strapped to his wrist:

> *"Avui he caminat tot el matí i l'app diu que he fet 0 quilòmetres. Algun problema?"*

He had. So did I. His Health app showed twelve thousand steps and zero kilometres. *El Pelegrí* dutifully showed the same — because that's what HealthKit was telling it. The pilgrim on screen hadn't moved an inch. He was, technically, walking on the spot.

That email is the moment this article is really about. Everything else — the Lottie tuning, the SwiftData split, the in-app purchase plumbing — is in service of a single, harder problem: *making a step-tracking app honest when the platform underneath it isn't.*

> **📸 Suggested screenshot:** the WalkView main screen as a hero image. Avatar in motion, level bar visible, "add steps" button highlighted.

---

## What I was actually building

Before the engineering: a one-paragraph product brief, because the technical decisions only make sense once you know what they're protecting.

*El Pelegrí* (Catalan for *the Pilgrim*) is a walking app with a deliberately small surface area. There is one screen with one button. Steps you've walked accumulate in HealthKit; you tap the button to "claim" them; an animated character — a cat, a hamburger, a mammoth, a zombie, your pick of about thirty — walks across a background painted with real Catalan landmarks. You progress through 100 levels with overwrought titles ("Pelegrí de Montserrat", "Flama Eterna del Canigó", "Pelegrí Definitiu"). The final, very-long-term milestone is to walk the circumference of the Earth: 40,075 kilometres. At an average 7,500 steps a day, that's about fourteen years. Nobody is in a hurry.

What it deliberately does not have: leaderboards, friend lists, streaks-that-shame-you-for-missing-a-day, push notifications begging you to come back, weekly summaries about how lazy you've been compared to your cohort. It's a single-player game played against geography and time. The contrarian design is the point, and most of the engineering exists to defend it.

> **📸 Suggested screenshot:** the three-tab structure (Walk, Statistics, Settings) — could be a short composite of the three tabs side by side.

---

## The shape of the system, in one paragraph

The app is a single SwiftUI scene with three tabs (Walk, Statistics, Settings), driven by one `ObservableObject` called `MoonwalkStore` (the original codename was *Moonwalk*; the rename never reached the source tree). The store owns HealthKit, owns the in-memory game state as a Swift `struct` called `Progress`, and persists that state through a SwiftData container holding three `@Model` classes (`ProgressModel`, `EntryModel`, `RegisterDataModel`). Views read from the store, mutate the struct, and the store writes the struct back to SwiftData. That's the whole architecture.

> **🗺️ Suggested diagram:** a data-flow diagram —
> `HealthKit → HealthManager → MoonwalkStore`
> `MoonwalkStore ↔ Progress (struct, in-memory)`
> `MoonwalkStore ↔ ProgressModel (SwiftData, on-disk)`
> `Progress → SwiftUI views`
> Helpful as a visual anchor before the problem sections.

Now the six problems.

---

## Problem 1 — HealthKit is a creative writer

The Xiaomi email was not a one-off. Once I started looking, HealthKit's "step count" and "distance walking/running" turned out to be two parallel narratives that frequently disagree:

- The **Xiaomi case**: the device records steps but never records distance. HealthKit faithfully relays steps from the band and zero kilometres from anyone. The user has walked, but the app thinks they haven't.
- The **cross-source case**: the user's phone GPS-tracked a 5 km run while the user's smartband counted steps for the entire day. HealthKit returns both, attributed to different sources, with totals that imply each step was 1.4 metres long — accurate only for very tall people falling downhill.
- The **silent first-time-user case**: a freshly installed Apple Watch, no walking history yet. HealthKit has nothing to say about anything, and any naive division by zero or fallback to "average human" produces garbage.

The fix needed a third number — one that doesn't depend on either the step source or the distance source telling the truth. HealthKit ships one: `HKQuantityTypeIdentifier.walkingStepLength`, which is the user's own measured average step length, computed by iOS from gait analysis. On app start I pull the 90-day average:

```swift
func fetchAverageStepLengthInKm() async throws -> Double? {
    let stepLengthType = HKQuantityType.quantityType(forIdentifier: .walkingStepLength)!
    let ninetyDaysAgo = Calendar.current.date(byAdding: .day, value: -90, to: Date())!
    let predicate = HKQuery.predicateForSamples(withStart: ninetyDaysAgo, end: Date(), options: .strictStartDate)

    return try await withCheckedThrowingContinuation { continuation in
        let query = HKStatisticsQuery(
            quantityType: stepLengthType,
            quantitySamplePredicate: predicate,
            options: .discreteAverage
        ) { _, statistics, error in
            let avgMeters = statistics?.averageQuantity()?.doubleValue(for: .meter())
            continuation.resume(returning: avgMeters.map { $0 / 1000 })
        }
        self.healthStore.execute(query)
    }
}
```

If HealthKit has nothing to say — no walking history, no Apple Watch, no gait data — I fall back to **0.762 metres per step**, the WHO's average for adult humans. It looks like a deeply suspicious magic number to ship as `static let stepsToKmFactor: Double = 0.000762`, but it's correct, well-cited, and humans are surprisingly consistent.

With a per-user anchor in hand, every entry gets *reconciled* against it:

```swift
private func resolvedDistanceKm(steps: Double, actualDistanceKm: Double, stepLengthKm: Double) -> Double {
    let expectedDistanceKm = steps * stepLengthKm
    if actualDistanceKm == 0 { return expectedDistanceKm }

    let ratio = actualDistanceKm / expectedDistanceKm
    return (ratio < 0.8 || ratio > 1.2) ? expectedDistanceKm : actualDistanceKm
}
```

Three branches, in plain English:

1. **HealthKit gave us zero distance** (the Xiaomi case) → synthesise distance from steps.
2. **The two numbers are within ±20% of each other** → trust HealthKit's distance, the sources are consistent.
3. **The two numbers are more than ±20% apart** (the cross-source case) → assume two different sensors disagreed, throw away the suspect distance, recompute from steps.

> **📊 Suggested diagram:** a small decision flowchart of `resolvedDistanceKm`. Three boxes (steps, actual distance, step length) feeding into a diamond ("ratio in [0.8, 1.2]?"), with two outcomes. Useful because the prose lives or dies on whether the reader instantly internalises the three branches.

The next problem was obvious in hindsight: every user already in production had bad historical data sitting in their database. So I shipped a one-shot, idempotent migration, gated by a single `@AppStorage` flag:

```swift
@AppStorage("hasAppliedStepDistanceMigration") private var hasAppliedStepDistanceMigration: Bool = false

@MainActor
func migrateEntriesWithMissingDistance() {
    guard !hasAppliedStepDistanceMigration else { return }
    for (date, entry) in progress.entries where entry.steps > 0 {
        let resolved = resolvedDistanceKm(steps: entry.steps,
                                          actualDistanceKm: entry.distance,
                                          stepLengthKm: averageStepLengthKm)
        if resolved != entry.distance {
            progress.entries[date] = Entry(steps: entry.steps, distance: resolved, date: date)
        }
    }
    save()
    hasAppliedStepDistanceMigration = true
}
```

Two design choices worth flagging:

- The flag lives in `UserDefaults`, **not** in SwiftData. Different devices have different historical sources; the migration should run once per device, not once per iCloud account.
- The migration is *idempotent and silent.* Users never see "migrating data, please wait". The data gets quietly correct in the background and the next time they open the app, their pilgrim has moved a little further than they remembered. Nobody has ever complained about an unexpected bonus.

> **📸 Suggested screenshot:** a side-by-side of the iOS Health app showing "12,420 steps, 0.0 km" next to El Pelegrí showing "9.5 km walked today". Caption: *the gap closed by `resolvedDistanceKm`*.

---

## Problem 2 — One button, twenty seconds, two registers a day

The button is the soul of the app. It says "X steps pending — tap to add". You tap. A shower of shoe-print particles flies from the button to the total counter. The level bar nudges forward. The pilgrim, somewhere on a background painted with the Costa Brava, advances.

The question I get asked most often by other developers: *why a button at all? Why not just sum HealthKit silently in the background?*

Two answers, both load-bearing:

1. **The tap is the dopamine.** Watching a number jump after a deliberate gesture is what makes this a game rather than a dashboard. Without the tap, every other animation in the app loses its purpose.
2. **Without throttling, anyone with five minutes and the Health app can be on level 100 by Tuesday.** Adding one step at a time and re-syncing is trivially easy. If the game just believes whatever HealthKit says, the level system collapses.

So the store enforces a quiet rule, with constants I've tuned more times than I'd like to admit:

```swift
var dataExpirationThresholdInDays = 2
var maxStepsRegistersPerDay = 2
var additionalStepsRegistersByWatchingAds = 1
```

Two free taps per day, plus one extra if you watch a rewarded ad (or if you've bought *Remove Ads*, in which case you get the extra register without the ad). After that, today is done. Come back tomorrow.

The enforcement isn't a counter — it's a full **audit log**. Every register the player makes is persisted as a `RegisterDataModel` row:

```swift
@Model
class RegisterDataModel {
    var date: Date = Date()
    var registerType: RegisterType = .recentData   // .recentData | .olderData
    var watchedAd: Bool = false
    var progressModel: ProgressModel?
}
```

The throttle then asks the log how many of each kind happened today:

```swift
func regularRegistersDoneToday() -> Int {
    let today = Calendar.current.startOfDay(for: Date())
    return registerDataByDay[today]?
        .filter { $0.registerType == .recentData }
        .filter { $0.watchedAd == false }
        .count ?? 0
}
```

Building it as an audit log instead of a daily integer counter cost me ten extra lines and bought a lot of things for free: I can show the user a history of their own activity, I can prove to myself that the constants are tuned correctly by querying production data, and "reset progress" still leaves a clean history when needed. The cheapest defense against bugs in counter logic is to never use counters.

There's also a **second, completely separate flow** for backfilled data — anything older than two days, scanned at app launch and surfaced on the Statistics tab. It always requires a rewarded ad to import (with the same *Remove Ads* bypass). The rationale: pairing a new smartband and importing six months of historical steps is legitimate behaviour, but it's also the most obvious cheat vector. An ad watch is friction, not punishment.

Behind everything, a low-priority `Task` re-polls HealthKit every twenty seconds while the app is open:

```swift
fetchPendingStepsLoopTask = Task(priority: .utility) { [weak self] in
    while !Task.isCancelled {
        guard let self else { return }
        await calculatePendingStepsAndDistance()
        try? await Task.sleep(for: .seconds(20))
    }
}
```

Twenty seconds is the number I landed on after watching users open the app, take their phone for a walk around the block, and come back disappointed that the counter hadn't moved. Five seconds was wasteful battery. Sixty seconds felt dead. Twenty seconds is "I noticed you walked".

> **🎬 Suggested screenshot or short GIF:** the moment of tapping the button — shoeprint particles flying from button to total, counter ticking up with the easing animation, level bar nudging forward. This is the app's signature interaction and it has to be in the article.

> **📈 Optional chart:** a small "day in the life of the button" timeline — HealthKit ticks vs button taps vs ad watches across a single user-day. Makes the throttle decisions feel concrete instead of arbitrary.

---

## Problem 3 — Thirty avatars that all stand differently

The visual centrepiece is a Lottie character walking in place against a sliding painted background. There are about thirty unlockable characters: a cat at level 15, a hamburger at 45, a mammoth at 15, a zombie locked behind level 100. Each is a `.lottie` file authored by a different illustrator with a different idea of where the canvas baseline is.

```swift
struct Avatar: Equatable, Identifiable, Comparable {
    var fileName: String
    var offset: EdgeInsets
    var scale: CGFloat = 1
    var speed: Double = 1
    var mirror: Bool = false
    var unlockLevel: Int
}

static var pumpking_1: Avatar {
    Avatar(fileName: "pumpking_1",
           offset: EdgeInsets(leading: -70, bottom: 11),
           scale: 1.2, speed: 0.7, unlockLevel: 30)
}
```

Every one of those numbers is hand-tuned. I tried, twice, to compute the offsets programmatically from the Lottie's published bounding box. It did not work. Lottie's published bounds are a polite suggestion, not a contract: the visual centre of a character can sit anywhere inside the box depending on whether the artist drew with margin, included a shadow, included extending walking-cycle limbs. After the third attempt I gave up, opened `Avatar.swift`, and nudged numbers until each character's feet touched the painted ground.

The trickiest piece of the stage is the background. It sits behind the avatar and offsets itself vertically so that the avatar's feet always land on the painted horizon line. The math turned out to depend on whether the user has iOS's display zoom turned on:

```swift
var adjustedOffsetYForBackgroundImage: CGFloat {
    let screenHeight = UIScreen.main.bounds.height
    let adjustment = if UIScreen.main.scale != UIScreen.main.nativeScale {
        screenHeight - animationPosition.y - 136
    } else {
        screenHeight - animationPosition.y - 179
    }
    return adjustment
}
```

`UIScreen.main.scale != UIScreen.main.nativeScale` is iOS's polite way of telling you the user has display zoom enabled. The constant offset differs because the avatar's measured `animationPosition.y` shifts under us. There's no clean way to solve this without two magic numbers. I tried. I lost. I shipped the magic numbers, and they've now sat in production for months without a single user complaint, which is sometimes how engineering works.

Levels themselves go from 0 to 100, with thresholds tuned by hand in a spreadsheet — early levels are easy on purpose (5,000 steps to level 1, 15,000 to level 2), and the curve has deliberate soft easings at round-number milestones (level 20 lands at exactly 1,000,000 steps; level 47 at 10,000,000) before climbing again. The titles drift further into Catalan folklore the higher you go: "Caminaire Novell" → "Pelegrí de Montserrat" → "Foc de la Patum" → "Flama Eterna del Canigó" → "Pelegrí Definitiu". Localising those into English without losing the soul will be its own project.

> **📊 Suggested chart:** a line chart of "steps required" vs "level" (0–100). It would visually expose the eases at level 20 and level 47, and make the difficulty curve a tangible thing instead of a wall of numbers. Easy to generate from `Level.levelsStepsThresholds`.

> **📸 Suggested screenshot:** a composite of three or four avatars (cat, mammoth, beer cup, zombie) side by side, each on the same background. Caption: *each one is positioned by hand.* Visually proves the point.

---

## Problem 4 — SwiftData wants classes, my brain wants values

SwiftData's `@Model` macro requires reference types. My domain logic — totals, averages, level thresholds, milestone reachability — is much happier as immutable-ish Codable structs. Forced to pick a side, I picked both:

- `ProgressModel`: SwiftData `@Model` class, the on-disk shape.
- `Progress`: plain Swift struct, the in-memory shape the UI reads and mutates.

A `toDomainModel()` converts the class to the struct on load. A static `save(_:context:)` diffs the struct against the persisted instance and writes back:

```swift
static func save(_ progress: Progress, context: ModelContext) {
    var descriptor = FetchDescriptor<ProgressModel>()
    descriptor.fetchLimit = 1

    let model = (try? context.fetch(descriptor).first) ?? {
        let m = ProgressModel(...)
        context.insert(m)
        return m
    }()

    model.level = progress.level
    model.entries = progress.entries.values.map { EntryModel.fromDomainModel($0, progressModel: model) }
    ...
    try? context.save()
}
```

This looks like over-engineering for an app whose database holds one row called `ProgressModel`. It isn't. The struct gives me four things I'd refuse to give up:

1. **`didSet` hooks.** Mutating `entries` automatically recomputes the level. The view doesn't have to remember to call `updateLevel()`.
2. **Value semantics in SwiftUI.** `@Published var progress` triggers UI updates cleanly. Reference-type SwiftData models in `@Published` properties are a recipe for stale views and silent re-renders.
3. **Free `Codable`.** Mock data for SwiftUI previews is a one-liner: `Progress.fake()` generates 30 days of randomised entries with a seeded RNG and the previews are deterministic.
4. **Refactor freedom.** The day SwiftData is replaced by GRDB or by raw JSON files, no view changes. The bridge is the only thing that moves.

The one thing I'd change if I started over: move the mapping into a dedicated `ProgressMapper` type instead of static methods on the model class. Static methods were the path of least resistance. They aged.

There's one related decision worth surfacing: **iCloud sync is a boot-time flag**, not a runtime toggle.

```swift
let iCloudSync = UserDefaults().bool(forKey: "iCloudSync")
let configuration = ModelConfiguration(
    schema: schema,
    isStoredInMemoryOnly: false,
    cloudKitDatabase: iCloudSync ? .automatic : .none
)
```

SwiftData's CloudKit option is baked into the `ModelConfiguration` at container creation. The user toggles a setting, the new value lands in `UserDefaults`, and the *next* launch reconfigures the container. Anyone who tells you "just flip CloudKit on at runtime" hasn't actually tried. The UX cost is that the setting toggle has to make the restart requirement explicit — currently it doesn't, and that's the second thing I'd change.

> **🗺️ Suggested diagram:** the bridge — class on one side (`ProgressModel`, `EntryModel`, `RegisterDataModel`), struct on the other (`Progress`, `Entry`, `RegisterData`), arrows for `toDomainModel` (load) and `fromDomainModel` (save). Useful because every iOS developer who's wrestled with SwiftData will recognise the pattern.

---

## Problem 5 — Monetising without poisoning the well

Monetisation is restrained on purpose: a banner at the bottom of the Statistics screen, a couple of interstitials between rare flows, the rewarded ad (which is gameplay, not pure revenue), and one in-app purchase, *Remove Ads*. No subscriptions, no consumables, no "support the developer" tip jar.

The `IAPManager` is a small `@Observable @MainActor` singleton with three responsibilities:

1. **Listens to `Transaction.updates`** for the lifetime of the app and processes verified transactions.
2. **Caches entitlements in `UserDefaults`** so users keep their *Remove Ads* status while offline. StoreKit 2's `currentEntitlements` needs the network, and offline users get cranky:
    ```swift
    private func saveEntitlements() {
        UserDefaults.standard.set(Array(purchasedProductIDs), forKey: purchasedKey)
    }

    private func loadLocalEntitlements() {
        if let saved = UserDefaults.standard.array(forKey: purchasedKey) as? [String] {
            purchasedProductIDs = Set(saved)
        }
    }
    ```
3. **Refreshes from `AppStore.sync()`** on demand for "Restore Purchases".

The offline cache is technically less secure than always trusting the server. A determined attacker can flip the `UserDefaults` value and unlock *Remove Ads* without paying. In exchange they get… the same app, but without ads. They've already won; it's fine. This is the most important architectural decision in the entire monetisation layer, because the alternative — a paying user opening the app on a plane and seeing ads — would be a real, measurable regression that I would have to apologise for.

The one piece of UX cross-talk between IAP and the rewarded ad that I'm quietly proud of:

```swift
if store.canAddStepsToday() {
    store.addSteps()
} else if store.canAddStepsByWatchingAd() {
    if iapManager.isPurchased(.remove_ads) {
        store.addSteps(watchedAd: true)        // skip the ad, grant the reward
    } else {
        try adCoordinator.showAd { _ in
            store.addSteps(watchedAd: true)
        }
    }
}
```

If you bought *Remove Ads*, you skip the rewarded ad *and* you still get the extra register. It would have been trivially easy to read "Remove Ads" literally and let the rewarded flow rot, leaving paying users with strictly fewer features than free users on that one screen. People notice that kind of thing. They tell you in App Store reviews. They tell their friends.

> **📸 Suggested screenshot:** the *Remove Ads* paywall and the rewarded ad prompt, side by side. Caption: *the paywall replaces both.*

---

## Problem 6 — Making the dopamine land

The functional app could ship without any of this section. The actually-good app couldn't.

**Shoeprint particles.** When you tap the button, between 4 and 30 `shoeprints.fill` SF Symbols fly from the button's captured position to the total-counter's captured position, scaling and fading on the way. The origin of each particle is randomised within a ±50pt radius so they don't shoot in a single boring beam. Each one is delayed by `index * 0.1s` so they cascade. And critically, *each landing fires its own haptic tick*:

```swift
Task {
    for _ in 0..<Int(amount) {
        if isPresented {
            try? await Task.sleep(for: .seconds(0.1))
            HapticFeedback.impactLight.generate()
        }
    }
}
```

The cascade of micro-vibrations under your thumb is the moment the app stops feeling like a fitness tracker and starts feeling like a game. It cost a hundred lines of code and it might be the single most important hundred lines in the project.

**Animated counter.** The total-steps text doesn't snap to its new value, it eases there over one second with an ease-out cubic and SwiftUI's `contentTransition(.numericText(value:))`. Forty interpolation steps. Subtle, and absolutely worth it.

**Confetti on level up and milestone reached.** Plain `ConfettiSwiftUI`, triggered from a `onChange(of: store.progress.level)` listener in `ContentView` plus a half-second delay so it lands on top of the alert, not before it.

**GeoJSON tracks.** Each real-world milestone — Estanys de Tristaina, Estany de Banyoles, Pica d'Estats, Canigó — ships with a `.geojson` of the actual route, rendered as an `MKPolyline` on a MapKit overlay inside `MilestoneDetailView`. The pilgrim's progress is shown by drawing the portion of the polyline the user has "walked", in colour, against the rest in grey. The first time a user finishes a real-world route in-game, the map fills in completely. It's a small thing. It's not a small thing.

**Custom alerts.** Welcome, LevelUp, NewMilestone, IAPSuccess, IAPError, IAPRestore, ResetProgress, ResetPurchases, ChangeTrackingStartDateConfirmation, NoEmail. All driven through one `configureCustomAlert` modifier on the root `WindowGroup` so the visual language is consistent across the whole app. The alternative — a dozen `.alert(isPresented:)` modifiers scattered through the view hierarchy — would have been faster to build and worse forever.

> **📸 Suggested screenshot:** the `MilestoneDetailView` for Estany de Banyoles, showing the GeoJSON track over MapKit with the "walked" portion highlighted. This is the most underrated piece of UX in the whole app.

> **📸 Suggested screenshot:** a level-up moment captured mid-confetti, with the alert and the confetti both visible.

---

## What I'd ship differently if I started over

Three specific architectural regrets, in order of how much they'd actually help:

1. **HealthKit reconciliation should have shipped on day one.** The Xiaomi bug was predictable. I didn't predict it. Every step-tracking app in the App Store has this problem, and almost all of them ship with the naive "trust HealthKit" implementation. I shipped one of them too. Now I know better.
2. **CloudKit-on/CloudKit-off should require an explicit app restart in the UI.** Today the setting silently takes effect on the next launch, which is technically correct and practically confusing. Users assume toggles are instant. The cost of a "Restart to apply" copy line is zero; the cost of confused users is real.
3. **Avatar positioning should be data-driven from an asset manifest, not from compile-time Swift structs.** Adding a new character today requires a code change, a build, and a hand-tuning session in `Avatar.swift`. It should require dragging a `.lottie` file into a folder, opening a small admin screen, and dragging the character into the right position. I'll get there eventually. Probably.

I notice none of these are about HealthKit's quirks or Lottie's bounding boxes. Those are problems the platform gave me. The three things I'd actually do differently are decisions I made.

---

## Closing

The app is on the App Store. The pilgrim is, today, somewhere around Banyoles for the median user. The Earth is 40,075 km wide and the average user walks about 7,500 steps per day, which works out to roughly fourteen years for a full circumnavigation. That's fine. The whole point was that there was never any rush.

The code is roughly 8,000 lines of Swift. About 80% of that is UI and animation glue — the kind of code that doesn't show up in architecture diagrams and absolutely shows up in user reviews. The interesting parts are barely 500 lines, and most of them are in this article.

If you build a step-tracking app: expect HealthKit to lie, validate everything against a third anchor, treat your gameplay constants as a public API, and put twice as much engineering into the moment between "tap" and "the counter ticked up" than you think you need. Walk on, *pelegrí*.

*— Marc*

---

## Appendix — Visuals checklist

A consolidated list of the screenshots, diagrams, and charts proposed inline. Capture or sketch in roughly this order:

**Screenshots (from the live app):**
1. **Hero / opening:** WalkView main screen, avatar mid-stride, level bar visible, "add steps" button highlighted.
2. **Tab structure:** Walk / Statistics / Settings, composited side by side.
3. **HealthKit gap:** iOS Health app showing "12,420 steps, 0.0 km" next to El Pelegrí showing the corrected distance. (Easiest to fake on a simulator with a Xiaomi-style data source.)
4. **The button moment** (GIF if possible): shoeprint particles flying, counter ticking up, level bar nudging — the signature interaction.
5. **Avatar lineup:** four avatars composited side by side on the same background (cat, mammoth, beer cup, zombie). Caption: *each one is positioned by hand.*
6. **Remove Ads paywall + rewarded ad prompt:** side by side, captioned *the paywall replaces both.*
7. **MilestoneDetailView for Estany de Banyoles:** the GeoJSON track over MapKit, walked portion highlighted.
8. **Level-up confetti:** captured mid-animation with the alert visible.

**Diagrams (to draw — Figma, Excalidraw, or hand-sketched):**
9. **System data flow:** HealthKit → HealthManager → MoonwalkStore ↔ Progress (struct) ↔ ProgressModel (SwiftData) → Views.
10. **`resolvedDistanceKm` decision tree:** three inputs → diamond ("ratio in [0.8, 1.2]?") → three outcomes.
11. **SwiftData ↔ domain bridge:** class column (ProgressModel/EntryModel/RegisterDataModel) ↔ struct column (Progress/Entry/RegisterData), with `toDomainModel`/`fromDomainModel` arrows.

**Charts (generate from data):**
12. **Level threshold curve:** line chart of "steps required" vs "level" (0–100), generated from `Level.levelsStepsThresholds`. Will visually show the eases at level 20 and 47.
13. **Optional — "day in the life of the button":** timeline showing HealthKit ticks, button taps, and ad watches over a single user-day. Makes the throttle decisions feel concrete.
