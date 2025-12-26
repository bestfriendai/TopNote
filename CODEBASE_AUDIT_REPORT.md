# TopNote Codebase Improvement Blueprint

> **Generated:** December 2025
> **Platform Detected:** iOS Native (SwiftUI + WidgetKit)
> **App Category:** Productivity / Spaced Repetition Learning
> **Health Score:** 82/100

---

## Executive Summary

TopNote is a well-structured, production-ready iOS application implementing a widget-first productivity experience with sophisticated spaced repetition logic. The codebase demonstrates professional iOS development patterns with careful attention to data persistence, user privacy, and performance optimization.

**Key Strengths:**
- Clean feature-based architecture with proper separation of concerns
- Sophisticated spaced repetition engine with configurable policies
- Excellent App Group integration for widget-to-app communication
- Comprehensive test coverage using Swift Testing framework
- Forward-looking AI integration with FoundationModels (iOS 26+)

**Areas for Improvement:**
- Missing CI/CD pipeline for automated testing and deployment
- Some error handling patterns could be strengthened
- Widget timeline management could be optimized further
- Minor accessibility gaps in some UI components
- Documentation could be enhanced for API surfaces

---

## Phase 1: Architecture Analysis

### 1.1 Current Architecture Assessment

**Pattern:** Feature-Based MVVM with SwiftData
**Rating:** Strong

```
TopNote/
├── App/                    # @main entry point, lifecycle
├── Cards/                  # Feature module (6 sub-features)
│   ├── Details/           # Card editing & AI generation
│   ├── Folders/           # Folder management
│   ├── List/              # Card browsing & filtering
│   ├── Row/               # Card row display
│   ├── Shared/            # Reusable card components
│   └── Tags/              # Tag management
├── Helpers/               # Utility functions
├── Models/                # SwiftData models
├── Persistence/           # Data layer configuration
├── Tips/                  # TipKit integration
└── UI/                    # Custom UI components
```

**Positive Observations:**
- Clear separation between main app and widget extension
- Shared `ModelContainer` via App Group (`group.com.zacharysturman.topnote`)
- Models are well-encapsulated with computed properties for derived state
- Good use of Swift's type system with enums for card types, priorities, and policies

### 1.2 Identified Issues & Recommendations

#### P1: Missing Sendable Conformance for Concurrent Contexts

**File:** `TopNote/Models/Card.swift:33`

```swift
// ❌ CURRENT: Card class is not Sendable-safe for async contexts
@Model
final class Card {
    // Static mutable state is not thread-safe
    private static var lastWidgetReload: Date = .distantPast
    private static let widgetReloadThrottle: TimeInterval = 2.0

    static func throttledWidgetReload() {
        let now = Date()
        guard now.timeIntervalSince(lastWidgetReload) > widgetReloadThrottle else { return }
        lastWidgetReload = now
        WidgetCenter.shared.reloadAllTimelines()
    }
}
```

```swift
// ✅ RECOMMENDED: Use actor for thread-safe throttling
actor WidgetReloadThrottler {
    private var lastReload: Date = .distantPast
    private let throttleInterval: TimeInterval = 2.0

    static let shared = WidgetReloadThrottler()

    func reloadIfNeeded() {
        let now = Date()
        guard now.timeIntervalSince(lastReload) > throttleInterval else { return }
        lastReload = now
        Task { @MainActor in
            WidgetCenter.shared.reloadAllTimelines()
        }
    }
}

// Usage in Card methods:
extension Card {
    func skip(at currentDate: Date) {
        self.answerRevealed = false
        self.skipCount += 1
        self.skips.append(currentDate)
        scheduleNext(baseInterval: repeatInterval, from: currentDate, as: .skip)
        Task { await WidgetReloadThrottler.shared.reloadIfNeeded() }
    }
}
```

**Source:** [Swift Concurrency Documentation](https://developer.apple.com/documentation/swift/actor)

---

#### P1: Force Unwrap in URL Construction

**File:** `TopNote/Models/Card.swift:56-59`

```swift
// ❌ CURRENT: Force unwrap can crash
var url: URL {
    guard let url = URL(string: "topnote://card/\(id)") else {
        fatalError("Failed to construct URL")
    }
    return url
}
```

```swift
// ✅ RECOMMENDED: Safe URL construction (UUID is always valid in URL)
var url: URL {
    // UUID string is always URL-safe, so this construction is deterministic
    URL(string: "topnote://card/\(id.uuidString)")!
}

// OR for maximum safety:
var url: URL {
    var components = URLComponents()
    components.scheme = "topnote"
    components.host = "card"
    components.path = "/\(id.uuidString)"
    return components.url ?? URL(string: "topnote://card/unknown")!
}
```

---

#### P2: JSON Encoding/Decoding Without Error Handling

**File:** `TopNote/Models/Card.swift:105-130`

```swift
// ❌ CURRENT: Silent failure on decode errors
var rating: [[RatingType: Date]] {
    get {
        guard let data = ratingData,
              let decoded = try? JSONDecoder().decode([[String: Date]].self, from: data)
        else { return [] }
        // ...
    }
    set {
        let dicts = newValue.map { dict -> [String: Date] in
            // ...
        }
        ratingData = try? JSONEncoder().encode(dicts) // Silent failure
    }
}
```

```swift
// ✅ RECOMMENDED: Add logging for debugging data corruption issues
var rating: [[RatingType: Date]] {
    get {
        guard let data = ratingData else { return [] }
        do {
            let decoded = try JSONDecoder().decode([[String: Date]].self, from: data)
            return decoded.compactMap { dict in
                var newDict: [RatingType: Date] = [:]
                for (key, value) in dict {
                    if let ratingType = RatingType(rawValue: key) {
                        newDict[ratingType] = value
                    }
                }
                return newDict.isEmpty ? nil : newDict
            }
        } catch {
            #if DEBUG
            print("[Card] Failed to decode rating data: \(error)")
            #endif
            return []
        }
    }
    set {
        let dicts = newValue.map { dict -> [String: Date] in
            var newDict: [String: Date] = [:]
            for (key, value) in dict {
                newDict[key.rawValue] = value
            }
            return newDict
        }
        do {
            ratingData = try JSONEncoder().encode(dicts)
        } catch {
            #if DEBUG
            print("[Card] Failed to encode rating data: \(error)")
            #endif
        }
    }
}
```

---

#### P2: Widget Timeline Efficiency

**File:** `TopNoteWidget/Provider/TopNoteWidgetProvider.swift:34-35`

```swift
// ❌ CURRENT: Fetches all cards on every timeline refresh
func fetchQueueCards(currentDate: Date, configuration: ConfigurationAppIntent) throws -> [Card] {
    let fetched = try context.fetch(FetchDescriptor<Card>()) // Fetches ALL cards
    // Then filters in memory...
}
```

```swift
// ✅ RECOMMENDED: Use predicate to filter at database level
func fetchQueueCards(currentDate: Date, configuration: ConfigurationAppIntent) throws -> [Card] {
    let selectedTypes = configuration.showCardType.map { $0.rawValue }
    let selectedFolderIDs = Set(configuration.showFolders.map(\.id))
    let includesNoFolder = configuration.showFolders.contains(where: { $0.isNoFolderSentinel })

    // Build predicate for database-level filtering
    var descriptor = FetchDescriptor<Card>(
        predicate: #Predicate<Card> { card in
            !card.isArchived && card.nextTimeInQueue <= currentDate
        },
        sortBy: [
            SortDescriptor(\.priorityRaw, order: .forward),
            SortDescriptor(\.nextTimeInQueue, order: .forward)
        ]
    )

    // Limit results for widget (we only need top card + count)
    descriptor.fetchLimit = 100

    var fetched = try context.fetch(descriptor)

    // Apply remaining filters that can't be expressed in predicate
    if !selectedTypes.isEmpty {
        fetched = fetched.filter { selectedTypes.contains($0.cardTypeRaw) }
    }

    if !configuration.showFolders.isEmpty {
        fetched = fetched.filter { card in
            if let folder = card.folder {
                return selectedFolderIDs.contains(folder.id)
            } else {
                return includesNoFolder
            }
        }
    }

    return fetched
}
```

**Source:** [SwiftData Performance - WWDC 2023](https://developer.apple.com/videos/play/wwdc2023/10187/)

---

## Phase 2: UI/UX Analysis (Apple HIG Compliance)

### 2.1 Positive Patterns

- **TipKit Integration:** Excellent onboarding with contextual tips (`Tips.swift`)
- **Swipe Actions:** Natural gesture-based interactions for card management
- **Navigation Split View:** Proper iPad/iPhone adaptive layout
- **Accessibility Identifiers:** Present on key UI elements for testing

### 2.2 Issues & Recommendations

#### P2: Missing Accessibility Labels on Custom Views

**File:** `TopNote/Cards/Row/CardRow.swift:57-91`

```swift
// ❌ CURRENT: Background selection lacks accessibility context
.background(
    RoundedRectangle(cornerRadius: 8)
        .fill(isSelected ? card.cardType.tintColor.opacity(0.12) : Color.clear)
)
.accessibilityIdentifier("CardRow-\(card.id.uuidString)")
// Missing: .accessibilityLabel, .accessibilityHint
```

```swift
// ✅ RECOMMENDED: Full accessibility support
VStack(spacing: 0) {
    // ... content
}
.accessibilityElement(children: .combine)
.accessibilityLabel("\(card.cardType.rawValue): \(card.content)")
.accessibilityHint(card.isEnqueue(currentDate: Date())
    ? "In queue. Swipe left to see options, swipe right to complete."
    : "Scheduled for \(card.displayedDateForQueue)")
.accessibilityAddTraits(isSelected ? .isSelected : [])
.accessibilityIdentifier("CardRow-\(card.id.uuidString)")
```

**Source:** [Apple Accessibility Programming Guide](https://developer.apple.com/documentation/accessibility)

---

#### P2: Touch Target Sizes on Widget Buttons

**File:** `TopNoteWidget/Views/Buttons/WidgetButtonStyle.swift`

Per Apple HIG, minimum touch target should be 44x44 points:

```swift
// ✅ RECOMMENDED: Ensure minimum touch targets
struct WidgetButton: View {
    var body: some View {
        Button(action: action) {
            content
        }
        .frame(minWidth: 44, minHeight: 44) // Minimum touch target
        .contentShape(Rectangle()) // Expand hit area
    }
}
```

**Source:** [Human Interface Guidelines - Touch Targets](https://developer.apple.com/design/human-interface-guidelines/accessibility#Touch-targets)

---

#### P3: Color Contrast for Priority Indicators

Ensure priority colors meet WCAG 2.1 AA contrast ratio (4.5:1 for normal text):

```swift
// Verify in PriorityType.swift that colors have sufficient contrast
extension PriorityType {
    var accessibleColor: Color {
        switch self {
        case .high: return Color(red: 0.8, green: 0.2, blue: 0.2) // Darker red
        case .med: return Color(red: 0.8, green: 0.6, blue: 0.0) // Darker orange
        case .low: return Color(red: 0.2, green: 0.6, blue: 0.2) // Darker green
        case .none: return Color.secondary
        }
    }
}
```

---

## Phase 3: Performance Analysis

### 3.1 Current Optimizations (Excellent)

The codebase already implements several performance best practices:

1. **Widget Reload Throttling** (`Card.swift:36-44`): 2-second throttle prevents excessive widget updates
2. **Cached Properties** (`Card.swift:132-200`): `@Transient` cached arrays avoid repeated JSON decoding
3. **Efficient Badge Count** (`TopNoteApp.swift:94-109`): Uses `fetchCount` instead of fetching all cards
4. **Query Optimization** (`CardRow.swift:23-24`): Removed per-row queries that caused 700+ separate fetches

### 3.2 Recommendations

#### P2: Add Index for Frequently Queried Fields

**File:** `TopNote/Models/Card.swift`

```swift
// ✅ RECOMMENDED: Add @Attribute macro for indexing
@Model
final class Card {
    @Attribute(.spotlight) var content: String = "" // Enables Spotlight search

    // Consider composite index for queue queries
    // Note: SwiftData doesn't support custom indexes yet,
    // but structure queries to leverage existing B-tree on id
}
```

---

#### P3: Lazy Image Loading in Cards

If images are re-enabled in the future:

```swift
// ✅ RECOMMENDED: Use AsyncImage pattern with caching
struct CardImageView: View {
    let imageData: Data?

    var body: some View {
        if let data = imageData, let uiImage = UIImage(data: data) {
            Image(uiImage: uiImage)
                .resizable()
                .aspectRatio(contentMode: .fit)
        } else {
            Image(systemName: "photo")
                .foregroundColor(.secondary)
        }
    }
}
```

---

## Phase 4: Security Hardening

### 4.1 Current Security Posture

**Overall Rating:** Good

- Local-first storage with optional CloudKit sync
- No external network calls (except CloudKit)
- App Group properly configured for widget communication
- No sensitive data stored in UserDefaults (only migration flags)

### 4.2 Recommendations

#### P2: Validate Deep Link Parameters

**File:** `TopNote/ContentView.swift:34-45`

```swift
// ❌ CURRENT: Limited validation
.onOpenURL { url in
    guard url.scheme == "topnote", url.host == "card" else { return }
    let cardId = url.pathComponents[1] // Could crash if path is empty
    // ...
}
```

```swift
// ✅ RECOMMENDED: Defensive deep link handling
.onOpenURL { url in
    handleDeepLink(url)
}

private func handleDeepLink(_ url: URL) {
    guard url.scheme == "topnote" else {
        print("[DeepLink] Invalid scheme: \(url.scheme ?? "nil")")
        return
    }

    switch url.host {
    case "card":
        guard url.pathComponents.count >= 2 else {
            print("[DeepLink] Missing card ID in path")
            return
        }
        let cardIdString = url.pathComponents[1]
        guard let uuid = UUID(uuidString: cardIdString) else {
            print("[DeepLink] Invalid UUID: \(cardIdString)")
            return
        }
        deepLinkedCardID = uuid
        selectedCardModel.selectCard(with: uuid, modelContext: modelContext, isNew: false)

    default:
        print("[DeepLink] Unknown host: \(url.host ?? "nil")")
    }
}
```

---

#### P3: Sanitize Export Data

**File:** `TopNote/Models/Card.swift:840-888`

```swift
// ✅ RECOMMENDED: Add content sanitization for CSV export
private static func escapeCSV(_ string: String) -> String {
    // Current implementation is good, but add length limit for safety
    let sanitized = string
        .trimmingCharacters(in: .whitespacesAndNewlines)
        .prefix(10000) // Prevent extremely large values

    let result = String(sanitized)
    if result.contains(",") || result.contains("\"") || result.contains("\n") || result.contains("\r") {
        return "\"\(result.replacingOccurrences(of: "\"", with: "\"\""))\""
    }
    return result
}
```

---

## Phase 5: Testing Analysis

### 5.1 Current Test Coverage

**Unit Tests:** Comprehensive (`UnitTests/UnitTests.swift`)
- Card scheduling and timing logic
- Skip policy multiplier validation
- Rating policy tests
- Repeat interval calculations

**UI Tests:** Present (`UITests/UITests.swift`)
- Card creation, editing, deletion
- Widget interaction testing

**Widget Tests:** Present (`UnitTests/WidgetTimelineTests.swift`)
- Timeline generation
- Image pipeline processing

### 5.2 Recommendations

#### P2: Add Edge Case Tests

```swift
// ✅ RECOMMENDED: Add boundary condition tests
@Suite("Card Edge Cases")
struct CardEdgeCaseTests {
    var modelContext: ModelContext
    var modelContainer: ModelContainer

    init() throws {
        let schema = Schema([Card.self, Folder.self, CardTag.self])
        let config = ModelConfiguration(schema: schema, isStoredInMemoryOnly: true)
        modelContainer = try ModelContainer(for: schema, configurations: config)
        modelContext = ModelContext(modelContainer)
    }

    @Test func repeatIntervalClampingMin() throws {
        let card = Card(
            createdAt: Date(),
            cardType: .todo,
            priorityTypeRaw: .none,
            content: "Test",
            repeatInterval: 1, // Very short interval
            skipPolicy: .aggressive
        )
        modelContext.insert(card)

        // Multiple skips should not go below minimum (24 hours)
        for _ in 0..<10 {
            card.skip(at: Date())
        }

        #expect(card.repeatInterval >= 24, "Interval should not go below 24 hours minimum")
    }

    @Test func repeatIntervalClampingMax() throws {
        let card = Card(
            createdAt: Date(),
            cardType: .flashcard,
            priorityTypeRaw: .none,
            content: "Test",
            repeatInterval: 8000, // Close to max
            ratingEasyPolicy: .aggressive
        )
        modelContext.insert(card)
        card.enqueue(at: Date())

        // Multiple easy ratings should not exceed max (8760 hours = 1 year)
        for _ in 0..<10 {
            card.submitFlashcardRating(.easy, at: Date())
        }

        #expect(card.repeatInterval <= 8760, "Interval should not exceed 8760 hours (1 year)")
    }

    @Test func emptyContentHandling() throws {
        let card = Card(
            createdAt: Date(),
            cardType: .note,
            priorityTypeRaw: .none,
            content: "" // Empty content
        )

        #expect(card.displayContent.isEmpty == true)
        #expect(card.displayAnswer == "Answer here...")
    }

    @Test func unicodeContentHandling() throws {
        let card = Card(
            createdAt: Date(),
            cardType: .flashcard,
            priorityTypeRaw: .none,
            content: "日本語テスト 🎌 العربية",
            answer: "Emoji: 👨‍👩‍👧‍👦"
        )

        #expect(card.content.count > 0)
        #expect(card.answer?.count ?? 0 > 0)
    }
}
```

---

#### P2: Add Performance Tests

```swift
// ✅ RECOMMENDED: Performance benchmarks
@Suite("Performance Tests")
struct PerformanceTests {
    @Test func largeCardSetFiltering() throws {
        let schema = Schema([Card.self, Folder.self, CardTag.self])
        let config = ModelConfiguration(schema: schema, isStoredInMemoryOnly: true)
        let container = try ModelContainer(for: schema, configurations: config)
        let context = ModelContext(container)

        // Create 1000 cards
        for i in 0..<1000 {
            let card = Card(
                createdAt: Date(),
                cardType: [.todo, .note, .flashcard][i % 3],
                priorityTypeRaw: .none,
                content: "Card \(i)"
            )
            context.insert(card)
        }
        try context.save()

        // Measure fetch time
        let startTime = CFAbsoluteTimeGetCurrent()
        let descriptor = FetchDescriptor<Card>(
            predicate: #Predicate { !$0.isArchived }
        )
        let results = try context.fetch(descriptor)
        let duration = CFAbsoluteTimeGetCurrent() - startTime

        #expect(results.count == 1000)
        #expect(duration < 1.0, "Fetch should complete in under 1 second")
    }
}
```

---

## Phase 6: CI/CD Recommendations

### 6.1 Missing CI/CD Pipeline (P0 - Critical)

The project currently has no automated CI/CD. This is a critical gap for production readiness.

#### Recommended GitHub Actions Workflow

Create `.github/workflows/ci.yml`:

```yaml
name: TopNote CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build-and-test:
    name: Build & Test
    runs-on: macos-14

    steps:
      - uses: actions/checkout@v4

      - name: Select Xcode
        run: sudo xcode-select -s /Applications/Xcode_15.2.app

      - name: Cache DerivedData
        uses: actions/cache@v4
        with:
          path: ~/Library/Developer/Xcode/DerivedData
          key: ${{ runner.os }}-deriveddata-${{ hashFiles('**/*.xcodeproj/project.pbxproj') }}
          restore-keys: |
            ${{ runner.os }}-deriveddata-

      - name: Build
        run: |
          xcodebuild build \
            -project TopNote.xcodeproj \
            -scheme TopNote \
            -destination 'platform=iOS Simulator,name=iPhone 15 Pro' \
            -configuration Debug \
            CODE_SIGNING_ALLOWED=NO

      - name: Run Unit Tests
        run: |
          xcodebuild test \
            -project TopNote.xcodeproj \
            -scheme TopNote \
            -destination 'platform=iOS Simulator,name=iPhone 15 Pro' \
            -testPlan TopNote \
            -resultBundlePath TestResults.xcresult \
            CODE_SIGNING_ALLOWED=NO

      - name: Upload Test Results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results
          path: TestResults.xcresult

  lint:
    name: SwiftLint
    runs-on: macos-14

    steps:
      - uses: actions/checkout@v4

      - name: Install SwiftLint
        run: brew install swiftlint

      - name: Run SwiftLint
        run: swiftlint --strict --reporter github-actions-logging
```

---

### 6.2 Recommended SwiftLint Configuration

Create `.swiftlint.yml`:

```yaml
disabled_rules:
  - line_length
  - file_length
  - type_body_length
  - function_body_length

opt_in_rules:
  - array_init
  - closure_end_indentation
  - closure_spacing
  - collection_alignment
  - contains_over_filter_count
  - empty_count
  - empty_string
  - enum_case_associated_values_count
  - explicit_init
  - first_where
  - force_unwrapping
  - implicitly_unwrapped_optional
  - last_where
  - legacy_multiple
  - literal_expression_end_indentation
  - multiline_arguments
  - multiline_parameters
  - operator_usage_whitespace
  - overridden_super_call
  - pattern_matching_keywords
  - prefer_self_type_over_type_of_self
  - private_action
  - private_outlet
  - prohibited_super_call
  - redundant_nil_coalescing
  - redundant_type_annotation
  - single_test_class
  - sorted_first_last
  - unavailable_function
  - unneeded_parentheses_in_closure_argument
  - vertical_parameter_alignment_on_call
  - yoda_condition

included:
  - TopNote
  - TopNoteWidget
  - UnitTests
  - UITests

excluded:
  - Pods
  - DerivedData
  - .build

force_cast: error
force_try: error
force_unwrapping: warning

identifier_name:
  min_length:
    warning: 2
    error: 1
  max_length:
    warning: 50
    error: 60
  excluded:
    - id
    - i
    - x
    - y

type_name:
  min_length: 3
  max_length: 50

nesting:
  type_level: 3
  function_level: 3

reporter: "xcode"
```

---

## Phase 7: Documentation Gaps

### 7.1 Missing API Documentation

Key models and functions should have comprehensive documentation:

```swift
/// A card represents a single item in the TopNote queue system.
///
/// Cards can be one of three types:
/// - **Todo**: A task to be completed
/// - **Flashcard**: A question/answer pair for spaced repetition learning
/// - **Note**: An informational snippet to be reviewed periodically
///
/// ## Scheduling System
///
/// Cards use a sophisticated spaced repetition algorithm where the interval
/// between appearances is modified by user actions:
///
/// ```
/// Skip (todos/flashcards): interval *= skipPolicy.skipMultiplier
/// Skip (notes): interval *= 1/skipPolicy.skipMultiplier (inverse - more time)
/// Easy rating: interval *= ratingEasyPolicy.easyMultiplier
/// Good rating: interval *= ratingMedPolicy.goodMultiplier
/// Hard rating: interval *= ratingHardPolicy.hardMultiplier
/// ```
///
/// Intervals are clamped between 24 hours (1 day) and 8760 hours (1 year).
///
/// ## Widget Integration
///
/// Cards are shared with the widget extension via an App Group container.
/// Widget reloads are throttled to prevent excessive updates.
///
/// - Note: All date/time operations use `Date()` for consistency.
/// - Warning: Do not modify `ratingData`, `completeData`, etc. directly.
///   Use the computed properties instead.
@Model
final class Card {
    // ...
}
```

---

## Production Readiness Checklist

### Security
- [x] No hardcoded secrets or API keys
- [x] Data stored locally with optional CloudKit sync
- [x] App Group properly configured
- [ ] Add deep link validation (P2)
- [ ] Add export data sanitization (P3)

### Performance
- [x] Widget reload throttling
- [x] Cached computed properties
- [x] Efficient badge count with fetchCount
- [x] Removed per-row queries
- [ ] Optimize widget timeline fetching (P2)

### Reliability
- [x] SwiftData with CloudKit backup
- [x] Migration logic for legacy data
- [x] Graceful degradation for AI features
- [ ] Add comprehensive error logging (P2)

### Testing
- [x] Unit tests with Swift Testing framework
- [x] UI tests for main flows
- [x] Widget timeline tests
- [ ] Add edge case tests (P2)
- [ ] Add performance benchmarks (P3)

### DevOps
- [ ] **Set up CI/CD pipeline (P0)**
- [ ] Add SwiftLint for code consistency
- [ ] Configure automated App Store deployment

### Accessibility
- [x] Accessibility identifiers for UI testing
- [ ] Add accessibility labels to card rows (P2)
- [ ] Verify touch target sizes (P2)
- [ ] Check color contrast ratios (P3)

### Documentation
- [ ] Add API documentation for models (P3)
- [ ] Document spaced repetition algorithm (P3)
- [x] README with feature overview

---

## Priority Matrix

| Issue | Impact | Effort | Priority |
|-------|--------|--------|----------|
| Missing CI/CD Pipeline | High | Medium | P0 |
| Thread-safe Widget Throttling | High | Low | P1 |
| Force Unwrap in URL | Medium | Low | P1 |
| Widget Timeline Optimization | Medium | Medium | P2 |
| Deep Link Validation | Medium | Low | P2 |
| Accessibility Labels | Medium | Low | P2 |
| Error Logging | Medium | Low | P2 |
| Edge Case Tests | Low | Medium | P2 |
| Export Sanitization | Low | Low | P3 |
| API Documentation | Low | Medium | P3 |
| Color Contrast | Low | Low | P3 |

---

## Resources & References

### Apple Documentation
- [SwiftData Documentation](https://developer.apple.com/documentation/swiftdata)
- [WidgetKit Documentation](https://developer.apple.com/documentation/widgetkit)
- [AppIntents Documentation](https://developer.apple.com/documentation/appintents)
- [Swift Testing Documentation](https://developer.apple.com/documentation/testing)
- [Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/)

### WWDC Sessions
- [Meet SwiftData - WWDC 2023](https://developer.apple.com/videos/play/wwdc2023/10187/)
- [Bring widgets to new places - WWDC 2023](https://developer.apple.com/videos/play/wwdc2023/10027/)
- [Meet Swift Testing - WWDC 2024](https://developer.apple.com/videos/play/wwdc2024/10179/)

### Accessibility
- [Accessibility Programming Guide](https://developer.apple.com/documentation/accessibility)
- [WCAG 2.1 Guidelines](https://www.w3.org/WAI/WCAG21/quickref/)

---

*Report generated by comprehensive codebase audit - December 2025*
