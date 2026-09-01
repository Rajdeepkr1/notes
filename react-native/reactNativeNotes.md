# React Native — Deep Dive Roadmap

We'll go from fundamentals → core building blocks → navigation & state → native integration → platform-specific concerns → performance → production.

*Builds directly on the React notes (components, hooks, state management, JSX all carry over identically) — this file focuses specifically on what changes and what's added when targeting native mobile instead of the DOM: native primitives instead of HTML elements, a different styling/layout model, navigation, native modules, and platform-specific build/deployment concerns.*

---

## 1. React Native Fundamentals

**Definition:** React Native is a framework for building native mobile applications (iOS and Android) using React's component model and JSX (React notes' sections 1–2) — the same `useState`/`useEffect`/component-composition mental model applies unchanged, but instead of rendering to the DOM, React Native renders to **actual native platform views** (`UIView` on iOS, `android.view.View` on Android) — meaning a React Native app is a genuinely native app, not a webpage wrapped in a browser shell.

**React Native vs Flutter vs native vs Ionic/Capacitor — Definition:**
| | React Native | Flutter | Native (Swift/Kotlin) | Ionic/Capacitor |
|---|---|---|---|---|
| Rendering | Real native views | Custom-drawn (Skia engine) | Real native views | WebView (HTML/CSS/JS) |
| Language | JavaScript/TypeScript | Dart | Swift/Kotlin | Web tech |
| Code sharing w/ web | High (React notes carry over) | None | None | Very high |
| Performance ceiling | Near-native | Near-native | Native | Lower (WebView overhead) |
| Best for | Teams with React experience wanting near-native UX | Teams wanting pixel-perfect, highly custom UI | Max performance/platform-API access, single-platform | Simple apps, max code reuse from an existing web app |

**The New Architecture: Fabric, TurboModules, JSI (brief) — Definition:** older React Native versions communicated between JavaScript and native code through an asynchronous, serialized **"Bridge"** (batched JSON messages) — a real performance bottleneck for anything latency-sensitive (animations, gestures). The **New Architecture** replaces this with **JSI (JavaScript Interface)** — direct, synchronous JS-to-native calls without serialization — powering **Fabric** (the new, more efficient rendering system) and **TurboModules** (lazily-loaded, JSI-based native modules, section 7) — the practical takeaway: modern React Native apps no longer pay the old bridge's serialization tax, closing much of the historical performance gap with fully native apps.

**Expo vs bare React Native CLI — Definition:** **Expo** is a managed toolchain and set of libraries built on top of React Native, providing a curated set of pre-built native modules (camera, location, notifications, etc.), a simplified build service (EAS Build, section 15), and OTA updates (section 16) — the recommended default starting point for most new apps, the same "batteries included" tradeoff already discussed for Next.js vs plain React (Next.js notes' section 1) or Django vs Flask (Python Backend notes). The **bare React Native CLI** gives full, unmediated control over native iOS/Android project files — necessary when an app needs a genuinely custom native module Expo doesn't provide, or extremely fine-grained native build configuration — Expo apps can also "eject"/prebuild into a bare-equivalent project when that need arises, so the choice isn't fully irreversible.

**Project setup (Expo):**

```bash
npx create-expo-app my-app --template
cd my-app
npx expo start   # scan the QR code with Expo Go, or run an emulator
```

---

## 2. Core Components & JSX Differences from Web

**Definition:** React Native does not render HTML — there is no `<div>`, `<span>`, or `<button>` — every visible piece of UI is one of a small set of built-in **native components**, each mapping directly to a real native view under the hood, imported from the `react-native` package rather than being implicit browser globals the way HTML elements are in web React.

```tsx
import { View, Text, Image, ScrollView } from 'react-native';

function ProfileCard() {
  return (
    <View style={{ padding: 16 }}>
      <Image source={{ uri: 'https://example.com/avatar.png' }} style={{ width: 64, height: 64 }} />
      <Text>Ada Lovelace</Text>
    </View>
  );
}
```

**`View`, `Text`, `Image`, `ScrollView`, `FlatList`, `SectionList` — Definition:** `View` is the fundamental layout container, React Native's equivalent of a `<div>` (supports Flexbox layout, section 3, but no other CSS display modes); `Text` is required to wrap **any** displayed text — unlike web, plain text cannot be a direct child of `View`; `Image` renders a native image view; `ScrollView` renders all of its children up front inside a scrollable container (fine for short, known-length content); `FlatList`/`SectionList` render large or unbounded lists **efficiently**, only mounting the items currently visible on screen (virtualization, covered in depth in section 14) — critical for any list of unknown or large length, where a plain `ScrollView` would eagerly render every item and tank performance.

**Why there's no `<div>`/`<span>` — Definition:** the browser's DOM and its full HTML element vocabulary simply don't exist at runtime on mobile — React Native's renderer (Fabric, section 1) translates React's virtual element tree directly into instructions for creating real native platform views, so only the specific set of components React Native (or a third-party native module) actually implements a native-view mapping for can be used — this is the single most important mental adjustment coming from web React: you cannot reach for arbitrary HTML tags or `dangerouslySetInnerHTML`-style raw markup.

**`Pressable`/`TouchableOpacity` vs web's `onClick` — Definition:** there is no `onClick` in React Native — touch interaction is handled via dedicated pressable components: `Pressable` (the modern, most flexible option, exposing press state for custom styling) and the older `TouchableOpacity`/`TouchableHighlight` (simpler, built-in visual feedback) — both accept an `onPress` handler, React Native's equivalent of a click handler, reflecting that "press" (not "click") is mobile's native interaction primitive.

```tsx
import { Pressable, Text } from 'react-native';

<Pressable onPress={() => console.log('pressed')} style={({ pressed }) => ({ opacity: pressed ? 0.5 : 1 })}>
  <Text>Tap me</Text>
</Pressable>
```

**Platform-specific component variants — Definition:** some components render meaningfully differently, or only exist, on one platform (e.g. iOS-only `ActionSheetIOS`) — covered fully in section 10's platform-specific patterns; the practical implication for this section is that "the same JSX" doesn't always guarantee pixel-identical behavior across iOS and Android, since each ultimately maps to that platform's own native view implementation with its own default look and feel.

---

## 3. Styling & Layout

**`StyleSheet.create()` vs inline styles — Definition:** React Native styling uses a JavaScript object syntax closely resembling CSS-in-JS (camelCased properties, numeric values treated as density-independent pixels rather than needing a `px` unit) — `StyleSheet.create()` wraps a plain style-object map, providing minor performance benefits (styles are validated and can be referenced by ID rather than recreated each render) and better tooling/type-checking versus writing raw inline style objects directly in JSX, the recommended default for anything beyond a one-off dynamic value.

```tsx
import { StyleSheet, View, Text } from 'react-native';

const styles = StyleSheet.create({
  container: { flex: 1, padding: 16, backgroundColor: '#fff' },
  title: { fontSize: 20, fontWeight: 'bold' },
});

function Card() {
  return (
    <View style={styles.container}>
      <Text style={styles.title}>Hello</Text>
    </View>
  );
}
```

**Flexbox as the only layout system — Definition:** React Native's layout engine (Yoga) implements Flexbox exclusively — there is no CSS Grid, no `float`, no `position: absolute` as a general layout tool (though `position: 'absolute'` exists for the same overlay use cases as web) — every layout problem is solved with `flexDirection`, `justifyContent`, `alignItems`, and `flex`, the same properties already covered for web Flexbox in the HTML/CSS notes' section 5. **Key default difference from web:** `flexDirection` defaults to `'column'` in React Native (versus `'row'` on the web), reflecting that most mobile UI stacks vertically by default — a common source of "why doesn't this layout look like it does on web" confusion for developers coming from the browser.

**`Dimensions` API, responsive design without media queries — Definition:** since there's no CSS and therefore no `@media` queries, responsive layout relies on the `Dimensions.get('window')` API (returning the current screen width/height) or the `useWindowDimensions()` hook (the reactive, re-render-on-rotation equivalent) — combined with Flexbox's inherently proportional sizing (`flex: 1` filling available space) to build layouts that adapt across the wide range of physical screen sizes mobile devices come in, the mobile-specific analogue of the responsive-design techniques covered in the HTML/CSS notes' section 12.

**Platform-specific styling (`Platform.select()`) — Definition:** `Platform.select({ ios: {...}, android: {...} })` returns a different style value depending on the running platform — used when a design genuinely needs to differ between iOS and Android (e.g. matching each platform's native shadow conventions, since iOS uses `shadow*` properties while Android uses `elevation`) — covered further as part of the broader platform-specific patterns in section 10.

---

## 4. Navigation

**Definition:** React Native has no built-in router — **React Navigation** is the de facto standard library providing screen-to-screen navigation, implemented as its own JavaScript-driven navigation stack (rather than the browser's native history/URL-driven navigation that React Router, React notes' section 9, builds on) since mobile has no address bar or URL concept by default.

**Stack, Tab, and Drawer navigators — Definition:** `createNativeStackNavigator()` provides the classic "push a new screen on top, swipe/back-button to go back" pattern (the dominant navigation model on both iOS and Android); `createBottomTabNavigator()` provides the familiar bottom tab bar switching between top-level sections of an app; `createDrawerNavigator()` provides a slide-out side menu — these are commonly **nested** (a bottom tab navigator where one tab itself contains its own stack navigator) to build the layered navigation structures typical of real apps.

```tsx
const Stack = createNativeStackNavigator();

function App() {
  return (
    <NavigationContainer>
      <Stack.Navigator>
        <Stack.Screen name="Home" component={HomeScreen} />
        <Stack.Screen name="Profile" component={ProfileScreen} />
      </Stack.Navigator>
    </NavigationContainer>
  );
}
```

**Passing params, typed navigation with TypeScript — Definition:** `navigation.navigate('Profile', { userId: 42 })` passes parameters to the destination screen, read there via `route.params` — with TypeScript, defining a `RootStackParamList` type mapping each screen name to its expected params gives fully type-checked navigation calls and param access, catching a typo'd screen name or missing param at compile time rather than as a runtime crash.

```tsx
type RootStackParamList = { Home: undefined; Profile: { userId: number } };
type ProfileProps = NativeStackScreenProps<RootStackParamList, 'Profile'>;

function ProfileScreen({ route }: ProfileProps) {
  return <Text>User: {route.params.userId}</Text>;
}
```

**Deep linking — Definition:** configuring a URL scheme (`myapp://profile/42`) or universal/app links lets the OS open the app directly to a specific screen from an external source (a push notification, section 11; a link shared outside the app) — React Navigation's `linking` config maps URL patterns to screens/params, the mobile equivalent of a web app's URL-based routing (React notes' section 9), just triggered by the OS rather than the browser's address bar.

**Comparison with React Router** — React Router (React notes) is declarative and URL-driven, tightly coupled to the browser's `History` API; React Navigation is declarative but manages its own in-memory navigation state and screen stack, since mobile has no equivalent browser history mechanism to build on — the component-based, declarative *philosophy* carries over from web React, but the underlying navigation model is a genuinely mobile-native concept (a stack of screens) rather than a URL-to-component mapping.

---

## 5. State Management

**Local state with hooks (recap)** — `useState`/`useReducer`/`useEffect` (React notes' sections 2–3) work completely unchanged in React Native — this is one of the biggest practical benefits of the framework: the exact same hooks-based mental model transfers directly, with zero mobile-specific relearning required for local component state.

**Context API in a mobile app (recap)** — React's Context API (React notes' section 5) also carries over unchanged, and remains the natural choice for app-wide, low-frequency-update state (current theme, authenticated user) in a mobile app exactly as on web.

**Redux Toolkit / Zustand in React Native — Definition:** both major web-React global-state libraries (React notes' section 5) work identically in React Native — no mobile-specific API differences — the same tradeoffs (Redux Toolkit's more structured, boilerplate-reduced approach vs Zustand's minimal, hook-based simplicity) already discussed there apply unchanged; the choice of state library is essentially independent of the web-vs-mobile question.

**Persisting state (AsyncStorage, MMKV) — Definition:** there is no `localStorage` in React Native — `AsyncStorage` (`@react-native-async-storage/async-storage`) provides an asynchronous, persistent key-value store as the closest equivalent, commonly paired with a state library's persistence middleware (e.g. `redux-persist`) to automatically rehydrate app state on launch; **MMKV** (`react-native-mmkv`) is a newer, significantly faster alternative built on JSI (section 1) offering synchronous reads/writes — the practical choice between them is largely a performance/maturity tradeoff, with MMKV increasingly favored for its speed when JSI is available.

---

## 6. Networking & Data Fetching

**`fetch` in React Native, differences from browser fetch — Definition:** React Native provides a `fetch` implementation with the same API surface as the browser's (React notes' section 7 covers the general pattern) — the practical differences are environmental rather than API-level: no CORS enforcement (since there's no browser origin-security model on native), and networking during local development against a machine on the same network requires using that machine's actual local IP rather than `localhost` (which on a physical device or emulator refers to the device itself, not the development machine).

**TanStack Query in a mobile context (recap)** — TanStack Query (React notes' section 10) works identically in React Native, and is equally valuable here — its caching, background-refetching, and (particularly relevant for mobile) built-in retry/offline-handling behavior directly addresses the flakier, more intermittent network conditions mobile apps routinely encounter compared to a typically-more-stable desktop browser connection.

**Offline-first patterns, `NetInfo` — Definition:** `@react-native-community/netinfo` exposes the device's current connectivity state (and changes to it) reactively — used to detect when a device goes offline and adapt behavior accordingly (queue mutations to replay once connectivity returns, show a persistent offline banner, fall back to cached data) — offline-first design is a substantially more central concern in mobile development than typical web development, since mobile users routinely move in and out of connectivity in ways a desktop web user rarely does.

---

## 7. Native Modules & Native APIs

**Camera, location, notifications, biometrics — Definition:** Expo (section 1) provides pre-built, well-maintained modules wrapping the most common native device capabilities — `expo-camera`, `expo-location`, `expo-notifications` (section 11), `expo-local-authentication` (biometrics) — each exposing a JavaScript API over the underlying native iOS/Android SDK, so most apps never need to write native code directly to access standard device hardware/OS features.

```tsx
import * as Location from 'expo-location';

async function getCurrentLocation() {
  const { status } = await Location.requestForegroundPermissionsAsync();
  if (status !== 'granted') return null;
  return await Location.getCurrentPositionAsync({});
}
```

**Bridging custom native code (TurboModules, brief) — Definition:** when an app needs native functionality no existing library provides, a **TurboModule** (section 1's New Architecture) can be written — native Swift/Kotlin code exposed to JavaScript through a strongly-typed interface, using JSI for direct, synchronous calls — a genuinely native-development task (requiring Swift/Kotlin knowledge), reserved for cases where no pre-built module already covers the need, since writing and maintaining a custom native module carries real ongoing cost (must be kept working across OS/React Native version upgrades on both platforms).

**Permissions handling (iOS vs Android differences) — Definition:** both platforms require explicit user permission for camera, location, notifications, etc., but with different declaration mechanisms — iOS requires usage-description strings in `Info.plist` (surfaced automatically by Expo config in `app.json`) explaining *why* the permission is needed (shown in the OS permission prompt); Android declares required permissions in `AndroidManifest.xml` — covered further in section 10's platform-specific concerns; a request that isn't properly declared on the relevant platform fails/crashes rather than simply prompting the user, making this a common source of platform-specific bugs.

---

## 8. Animations

**`Animated` API fundamentals — Definition:** React Native's original, built-in animation system — `Animated.Value` holds an animatable number, driven by `Animated.timing()`/`Animated.spring()`, and interpolated (`.interpolate()`) into style values — animations can run on the **native thread** (`useNativeDriver: true`) for properties it supports (transform, opacity), keeping them smooth even if the JavaScript thread is simultaneously busy — a critical distinction from web CSS animations, where JS-thread blocking is a much rarer concern.

```tsx
const opacity = useRef(new Animated.Value(0)).current;

useEffect(() => {
  Animated.timing(opacity, { toValue: 1, duration: 300, useNativeDriver: true }).start();
}, []);

<Animated.View style={{ opacity }}><Text>Fading in</Text></Animated.View>
```

**Reanimated 2/3 for high-performance, UI-thread animations — Definition:** `react-native-reanimated` is the modern, recommended animation library — built on JSI (section 1), it lets animation *logic itself* (not just the animated values) run directly on the native UI thread via "worklets" (small JS functions compiled to run natively) — this matters specifically because gesture-driven animations (dragging, swiping) need to respond to touch input with zero perceptible lag, which requires the animation logic to react to gesture updates without ever needing a round-trip back to the (potentially busy) JavaScript thread — the same underlying JSI performance motivation described in section 1, applied specifically to animation.

**Gesture handling with `react-native-gesture-handler` — Definition:** replaces React Native's built-in, JavaScript-thread-based touch responder system with native-thread gesture recognition — commonly paired directly with Reanimated (both built for the same native-thread-first performance philosophy) to build fluid, native-feeling swipe-to-dismiss, drag-and-drop, and pinch-to-zoom interactions that would visibly stutter if driven purely through the JS thread.

---

## 9. Forms & User Input

**`TextInput` fundamentals, keyboard handling — Definition:** `TextInput` is React Native's text-entry component, React-controlled exactly like a web `<input>` (React notes' section 4's controlled-component pattern) via `value`/`onChangeText` — but additionally must account for the **on-screen keyboard**, which occupies real screen space and can obscure the very input being edited, a concern that simply doesn't exist for a physical-keyboard web form.

```tsx
const [text, setText] = useState('');
<TextInput value={text} onChangeText={setText} placeholder="Enter your name" style={styles.input} />
```

**Form libraries (React Hook Form) — Definition:** React Hook Form (React notes' section 4) works in React Native with no API differences from web — its uncontrolled, ref-based approach to minimizing re-renders is equally valuable in a mobile context, particularly for longer forms on lower-powered devices where excess re-renders are more visibly costly than on a typical desktop.

**`KeyboardAvoidingView`, keyboard-related UX pitfalls — Definition:** `KeyboardAvoidingView` automatically adjusts its child layout (padding or shifting position) to keep the currently-focused input visible above the on-screen keyboard — required on any screen with a `TextInput` near the bottom of the visible area; a genuinely mobile-specific UX concern with no web equivalent, and a very common source of "the input field is hidden behind the keyboard" bugs when omitted or misconfigured (the `behavior` prop needs different values on iOS `'padding'` vs Android `'height'`, another platform-specific wrinkle, section 10).

---

## 10. Platform-Specific Development (iOS vs Android)

**`Platform.OS`, `.ios.tsx`/`.android.tsx` file extensions — Definition:** `Platform.OS === 'ios'` provides a simple runtime check for conditional logic/rendering; for larger platform-specific differences, naming a file `Component.ios.tsx` and `Component.android.tsx` lets the bundler (Metro, section 15) automatically pick the correct implementation per platform at build time, based purely on the importing file's extensionless import path — a cleaner separation than scattering `Platform.OS` checks throughout a single shared file when the divergence is substantial.

**Safe area handling (notches, status bars) — Definition:** modern phones have non-rectangular safe display areas (notches, home indicators, rounded corners, camera cutouts) — `react-native-safe-area-context`'s `SafeAreaView`/`useSafeAreaInsets()` ensures content isn't rendered underneath these physical obstructions — a concern entirely specific to modern mobile hardware, with no meaningful web equivalent, and one of the most common "looks broken on a real device but fine in a basic simulator" bug categories for teams that skip testing on actual notched hardware.

**Permission model differences between iOS and Android (recap)** — see section 7; beyond the declaration mechanism, the *runtime behavior* differs too — iOS permission prompts are typically one-time (a permanent grant/deny requiring a trip to Settings to change), while Android has historically supported more granular, revocable, and (on modern versions) time-limited permission grants — code handling permission-denied states needs to account for both models' differing re-prompt/recovery flows.

---

## 11. Push Notifications

**Expo Notifications / Firebase Cloud Messaging — Definition:** `expo-notifications` provides a unified API over each platform's underlying push infrastructure — Apple Push Notification service (APNs) on iOS, Firebase Cloud Messaging (FCM) on Android — abstracting away the platform-specific device-token registration and delivery mechanics, so application code largely doesn't need to handle iOS/Android push differently at the API level, even though the underlying delivery systems are entirely separate services.

**Local vs remote notifications — Definition:** a **local** notification is scheduled and triggered entirely on-device (e.g. a reminder set for a future time, requiring no server or network at all); a **remote** (push) notification is sent from a backend server through APNs/FCM to a specific device, requiring the app to have first registered for and sent its device push token to that backend — the same server-initiated-vs-client-initiated distinction, just applied to notifications specifically rather than general networking.

**Handling notification taps, deep linking into the app — Definition:** a notification typically carries a data payload (e.g. `{ screen: 'Profile', userId: 42 }`) that the app reads when the user taps it, using that payload to deep-link (section 4) directly to the relevant screen — the standard pattern connecting a push notification to actually landing the user in the right place within the app, rather than just opening to whatever the default launch screen happens to be.

---

## 12. Authentication in Mobile Apps

**Token storage (SecureStore vs AsyncStorage) — Definition:** `AsyncStorage` (section 5) stores data **unencrypted** on-device — genuinely unsafe for sensitive values like a JWT/refresh token, since a compromised or rooted/jailbroken device could read it directly; `expo-secure-store` wraps each platform's native secure storage (iOS Keychain, Android Keystore) to store sensitive tokens **encrypted at rest** — the mobile-specific parallel to the web notes' repeated point about `localStorage` being unsafe for tokens (favoring HttpOnly cookies there instead) — here, SecureStore is the equivalent safer default.

**Biometric authentication (Face ID/Touch ID) — Definition:** `expo-local-authentication` lets an app require Face ID/Touch ID (iOS) or fingerprint/face unlock (Android) before allowing access to a sensitive screen or before decrypting a locally-stored token — commonly layered on top of (not as a replacement for) a real backend-issued auth token, adding a device-level "prove it's really you, right now" gate in addition to the token proving "this device was previously authenticated."

**OAuth flows in a mobile context — Definition:** unlike a web app's simple redirect-based OAuth flow, mobile OAuth (Google/GitHub/etc. login) needs to open the provider's login page in a genuinely secure, isolated browser context — `expo-auth-session` (using the system's native in-app browser, e.g. `SFSafariViewController` on iOS) is the recommended approach, specifically **avoiding** a plain in-app `WebView` for login flows, since a WebView is controlled by the app itself and could in principle intercept credentials — a mobile-specific security consideration with no direct web equivalent.

---

## 13. Testing React Native Applications

**Unit testing with Jest (recap)** — Jest (React notes' section 14) is React Native's default test runner too (`jest-expo` preset) — the same unit-testing fundamentals (assertions, mocking, `describe`/`it` structure) apply unchanged.

**Component testing with React Native Testing Library — Definition:** the React Native-specific sibling of React Testing Library (React notes' section 14) — same philosophy (query by what the user sees: text, accessibility role/label, rather than implementation details) adapted to React Native's component set (`getByText`, `fireEvent.press` instead of `fireEvent.click`).

**E2E testing with Detox/Maestro — Definition:** genuine end-to-end tests running against a real compiled app on a simulator/emulator or physical device — **Detox** is a gray-box framework tightly synchronized with the app's own event loop (reducing test flakiness from timing issues); **Maestro** is a newer, simpler, YAML-driven black-box alternative — both fill the same role Playwright/Cypress (React notes' section 14) fill for web: validating real, full user flows against a genuinely running application rather than a test-environment simulation of one.

---

## 14. Performance Optimization

**Avoiding unnecessary re-renders (recap, mobile-specific concerns)** — `React.memo`, `useMemo`, `useCallback` (React notes' section 11) apply identically in React Native — but the *cost* of an unnecessary re-render is often more visible on mobile, given generally less powerful hardware than a typical development/desktop machine, making this optimization discipline somewhat higher-stakes in practice than it might feel developing primarily on a fast desktop browser.

**`FlatList` performance tuning — Definition:** `FlatList` (section 2) only renders items currently near the visible viewport (**virtualization**) rather than the entire list — but several props meaningfully affect how well this works in practice: `keyExtractor` (a stable, unique key per item, the same React list-key correctness requirement from the React notes' section 2, critical here for the virtualization/recycling machinery to work correctly); `getItemLayout` (pre-computing item dimensions, letting `FlatList` skip an expensive on-the-fly layout-measurement pass, dramatically improving scroll performance for fixed-height items); `windowSize`/`maxToRenderPerBatch` (tuning how much content is rendered ahead of/behind the visible viewport, trading memory for scroll smoothness).

**Bridge/JSI performance considerations, why Reanimated exists (recap)** — see sections 1 and 8; the historical bridge's async, serialized JS-native communication is the root motivation behind both the New Architecture generally and Reanimated specifically — understanding *why* these tools exist (avoiding round-trips across a slow communication boundary) makes it easier to recognize similar bottlenecks in custom code, not just in animation/gesture libraries.

**Bundle size and startup time (Hermes engine) — Definition:** **Hermes** is a JavaScript engine purpose-built for React Native, optimized specifically for fast app startup (via ahead-of-time bytecode compilation, rather than parsing/compiling JS from source on every launch) and lower memory usage compared to using a general-purpose JS engine like V8/JavaScriptCore directly — enabled by default in modern React Native/Expo projects, directly addressing mobile users' low tolerance for slow app-launch times.

---

## 15. Native Build Process & Tooling

**Metro bundler — Definition:** React Native's JavaScript bundler (its Webpack/Vite equivalent, React notes' section 1) — bundles the app's JS/TS source into a single bundle the native runtime executes, with fast-refresh support for the same near-instant edit-reflected-immediately development loop already covered for Vite in the React notes.

**iOS build process (brief) — Definition:** requires Xcode, CocoaPods (iOS's native dependency manager, resolving and linking native library dependencies into the Xcode project), and code-signing provisioning profiles (Apple's mechanism for authorizing which devices/accounts can run a given build) — genuinely requires a Mac to build for iOS locally, a real practical constraint for teams without Mac hardware (addressed by EAS Build, below).

**Android build process (brief) — Definition:** uses Gradle (Android's build system) to produce either an **APK** (a directly-installable package, useful for testing/sideloading) or an **AAB** (Android App Bundle — the required format for Play Store submission, letting Google Play generate optimized, device-specific APKs at install time rather than shipping one universal APK to every device).

**EAS Build (Expo's managed build service) — Definition:** builds both iOS and Android binaries in the cloud, **without requiring local Xcode/Android Studio setup at all** — directly solves the "need a Mac to build for iOS" constraint above, and is the standard, recommended build path for Expo-managed projects, the mobile-development equivalent of offloading a build to CI rather than requiring every developer's machine to have the full native toolchain installed.

---

## 16. Over-the-Air Updates & CodePush

**Expo Updates / CodePush concepts — Definition:** because a React Native app's actual UI logic lives in its JavaScript bundle, that bundle can be **pushed to already-installed apps directly, without going through app store review** — `expo-updates` (or the older, still-used CodePush) downloads a new JS bundle on app launch and applies it, letting bug fixes and UI changes reach users in minutes rather than waiting days for app store review — a capability with no meaningful web-development parallel (a web app is *always* effectively "updated instantly" simply by nature of being served fresh on every page load) but a significant, mobile-specific advantage over the traditional app-store-review-gated release cycle.

**What can and can't be updated OTA — Definition:** only JavaScript/asset changes can be delivered OTA — any change touching **native code** (adding a new native module, section 7; upgrading the React Native version itself; changing native permissions/configuration) requires a full new binary submitted through the normal app store review process (section 17) — both app stores' guidelines explicitly restrict OTA updates to non-native changes specifically to prevent circumventing app review entirely, so OTA is a powerful complement to, never a full replacement for, standard app store releases.

---

## 17. App Store & Play Store Deployment

**App signing, provisioning — Definition:** every release build must be cryptographically signed — iOS via a certificate + provisioning profile tied to an Apple Developer account; Android via a signing key (the same key must be used for every future update to a given app, making secure key backup critical — a lost Android signing key means permanently losing the ability to update that app listing) — both platforms use signing specifically to guarantee that only the legitimate developer can publish updates to an existing app listing.

**Store submission process, review guidelines (brief) — Definition:** Apple's App Store review is a manual (though partly automated) human review process, typically taking anywhere from hours to a few days, checking both technical compliance and design/content guidelines; Google Play's review is generally faster and more automated — both stores maintain guidelines an app must comply with (data privacy disclosures, no misleading functionality, appropriate content ratings) — a genuinely mobile-specific deployment gate with no equivalent in web deployment (Deployment notes), where a team fully controls their own release timing.

**Versioning strategy — Definition:** apps carry two separate version concepts — a user-facing **semantic version** (`1.4.2`, communicating meaningful change to users/across releases) and an internal, strictly-incrementing **build number** (required by both stores to be higher than the previous submission's, even for a resubmission of the *same* semantic version after a rejected review) — teams typically automate build-number incrementing in CI to avoid the easy-to-forget manual step of bumping it on every submission.

---

## 18. React Native Interview Prep

**Common interview questions** — explain the New Architecture's JSI/Fabric/TurboModules and what problem it solves versus the old bridge (section 1); how does `FlatList` achieve good performance on large lists, and what props matter most for tuning it (section 14); how would you securely store an auth token on-device, and why not `AsyncStorage` (section 12); what's the difference between a local and a remote notification (section 11); walk through what can and can't be shipped via an OTA update and why (section 16); when would you choose Expo over the bare React Native CLI (section 1).

**React Native vs Flutter vs native development — final comparison table:**
| | React Native | Flutter | Native (Swift/Kotlin) |
|---|---|---|---|
| Learning curve for a React dev | Very low (same mental model) | Moderate (new language: Dart) | High (two entirely separate platform SDKs) |
| Code sharing across iOS/Android | High | Very high | None |
| UI fidelity to platform conventions | High (uses real native views) | Requires deliberate platform-adaptive design | Perfect, by definition |
| Ecosystem | Huge (JS/npm + RN-specific) | Growing, smaller than JS | Each platform's own mature, large ecosystem |
| Best for | Teams with existing React/JS expertise wanting near-native UX efficiently | Teams wanting maximum UI control/consistency across platforms | Max performance, deep platform-API access, single-platform-focused apps |

**Where React concepts carry over unchanged vs where mobile changes the model — Definition:** as a closing summary — components, JSX, hooks (`useState`/`useEffect`/`useMemo`/`useCallback`), Context, and the entire ecosystem of general-purpose state-management/data-fetching libraries (Redux Toolkit, Zustand, TanStack Query) all transfer **directly and completely unchanged** from the React notes; what's genuinely new and mobile-specific is: the native-component vocabulary replacing HTML (section 2), Flexbox-only layout (section 3), stack/tab-based navigation replacing URL-based routing (section 4), the native module/permissions/platform-differences layer (sections 7, 10), and the entirely separate build/release pipeline (sections 15–17) — understanding this split is the fastest way to leverage existing React knowledge while correctly identifying exactly where genuinely new, mobile-specific learning is actually required.
