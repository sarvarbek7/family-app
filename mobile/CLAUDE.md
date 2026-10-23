# mobile — mobile client

Read the root [CLAUDE.md](../CLAUDE.md) first for repo-wide rules.

## Responsibility

- The mobile app for the Family app. It must ship on both iOS and Android from one Kotlin codebase.
- Local-first. Every screen works offline against the local database. Changes sync with the backend when a connection is available.
- Gets all data through the backend HTTP API. It holds no business rules that the backend does not also enforce.

## Stack

- Language: Kotlin.
- Platform: Kotlin Multiplatform, targeting Android and iOS.
- UI: not decided. Compose Multiplatform for shared UI is the default choice, rather than separate native UIs, unless decided otherwise.
- Networking: Ktor client with kotlinx.serialization.
- Concurrency: Kotlin coroutines and Flow.
- Build: Gradle with the Kotlin DSL and a version catalog in `gradle/libs.versions.toml`.
- Navigation, dependency injection and local storage libraries: not decided.
- Minimum OS versions: not decided.

## Commands

Run from `mobile/`. Confirm these once the project is scaffolded. Module names assume the standard KMP wizard layout.

- Build Android debug: `./gradlew :composeApp:assembleDebug`
- Install on Android device or emulator: `./gradlew :composeApp:installDebug`
- Run on iOS: open `iosApp/iosApp.xcodeproj` in Xcode and run. This requires macOS.
- Shared tests: `./gradlew :shared:allTests`
- All checks: `./gradlew check`

The Gradle build needs a JDK. None is on the PATH of this machine yet, so install one or point `JAVA_HOME` at the JDK bundled with Android Studio. iOS builds cannot be done on this Linux machine.

## Code layout

- Put shared logic in `commonMain`: models, API client, repositories, view models and, with Compose Multiplatform, the UI.
- Put platform-only code in `androidMain` and `iosMain`.
- Use `expect`/`actual` declarations or interfaces injected per platform for anything platform-specific, such as secure storage.
- Keep `commonMain` free of Android and iOS imports.

## Choosing a family

A user can belong to several families. They choose which one they are managing.

- After login, show a family picker listing the user's families.
- The profile menu has a family switcher.
- If the user has exactly one family, open it directly without showing the picker.
- If the user has no families yet, show onboarding to create a family or accept an invitation.
- Remember the last chosen family on this device and open it on the next launch, as long as the user is still a member.
- If the backend returns 404 for the chosen family, the membership has ended. Remove that family from the switcher and show the picker again.
- Switching families is instant and works offline, because every family's data is already in the local database.
- Background sync keeps every family up to date, not only the one currently shown.

## Rules

- Keep API calls in one dedicated layer in `commonMain`, shared by both platforms.
- Generate every new record's ID on the device as a UUID version 7. Never wait for the backend to assign an ID.
- Screens read from the local database, not straight from the API. Sync code is the only code that talks to the backend for data.
- A user can belong to several families. Store every family's local data tagged with its family ID, sync each family through its own family-scoped endpoints, and let the user switch between families in the UI.
- When the backend reports that the user is no longer a member of a family, delete that family's local data from the device on the next sync.
- The backend API base URL comes from build configuration, so dev, staging and production can differ.
- Store tokens in secure storage: the Keychain on iOS and Keystore-backed storage on Android. Never use plain preferences.
- Health data, as defined in the backend's "Health data encryption" section (body sizes, feeding, sleep, medical examinations and more), is medical data. The local database must be encrypted at rest, for example SQLCipher with whichever local storage library is chosen. Generate its key randomly on first launch and keep it in the Keychain or Keystore, never in code or preferences.
- Never write health data to plain files, logs, crash reports, analytics, notifications on a locked screen, or the clipboard. Exclude the local database from unencrypted device backups.
- On logout, delete the local database and its key.
- Handle offline and slow-network states on every screen that loads data.
- Every feature must work on both iOS and Android. Do not merge a platform-only feature without saying so.
- Never commit signing keys, keystores, provisioning profiles or `local.properties`.
