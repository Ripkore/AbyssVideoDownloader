# Abyss Video Downloader - AI Agent Guide

## Project Overview
AbyssVideoDownloader is a CLI Kotlin application that downloads videos from multiple Vietnamese streaming providers (tvphim, sieutamphim, motchill, etc.) by fetching metadata from abyss.to and downloading encrypted video segments in parallel.

## Architecture

### Core Flow
1. **CLI Entry** → `Application.kt` parses arguments, extracts URL and resolution preference
2. **Provider Detection** → `ProviderDispatcher.getProviderForUrl()` routes to provider-specific ID extractor
3. **Metadata Fetch** → `VideoDownloader.getVideoMetaData()` calls abyss API, extracts segment URLs
4. **Parallel Download** → `VideoDownloader.downloadSegmentsInParallel()` uses Kotlin coroutines with `Semaphore` to limit concurrent downloads
5. **Merge & Decrypt** → Segments merged and decrypted with AES-CTR

### Service Boundaries
- **ProviderDispatcher**: Maps hosting domains to `Provider` implementations (each extracts video ID differently)
- **VideoDownloader**: Orchestrates download lifecycle using coroutines
- **HttpClientManager**: Centralized HTTP client management (via Koin singleton)
- **CryptoHelper**: AES-CTR encryption/decryption for segment data

### Data Models
- `SimpleVideo`: Segments and encryption key extracted from `Mp4` metadata
- `Config`: Download settings (resolution, output path, connection limit)
- `Datas`: Maps resolution labels to source objects with segments

## Build & Execution

### Build
```bash
./gradlew build          # Compile and run tests (JUnit 5)
./gradlew shadowJar     # Create fat JAR (abyss-dl-shadowJar.jar)
./gradlew proguard      # Obfuscate (ProGuard configured for rhino/httpcomponents exclusions)
```

### Run
```bash
java -jar abyss-dl.jar [URL_or_ID] [-o output.mp4] [-c connections] [-H "header:value"] [--verbose]
```
- Resolution auto-detection: append `_h` (high), `_l` (low), `_m` (medium) to URL/ID

### Key Dependencies
- **Kotlin Coroutines**: Parallel segment downloads with semaphore limits
- **Rhino**: JavaScript execution (extract video IDs from provider pages)
- **Jsoup**: HTML parsing
- **Gson**: JSON deserialization for API responses
- **Unirest**: HTTP requests
- **Koin**: DI (all core services are `single` instances except CLI args as `factory`)

## Critical Patterns

### Provider Extension Pattern
Adding a new video host:
1. Create `class NewHostProvider : Provider` in [src/main/kotlin/com/abmo/providers/](src/main/kotlin/com/abmo/providers/)
2. Implement `getVideoID(url: String): String?`
3. Register in `ProviderDispatcher.getProviderForUrl()` with host domain check
4. Common pattern: parse URL with Jsoup/regex or execute embedded JavaScript (see `TvphimProvider`)

### Segment Download with Progress
`VideoDownloader.downloadSegmentsInParallel()` uses:
- **Semaphore**: Limits concurrent coroutines to `config.connections` (1-10)
- **AtomicInteger/AtomicLong**: Thread-safe progress tracking
- **Flow**: `requestSegment()` chunks data via Unirest streaming
- Progress bar updated every 1s via `displayProgressBar()` utility

### Resolution Selection
In `Application.kt`:
```kotlin
when(resolution) {
    "h" -> videoSources.maxBy { it?.size!! }?.label
    "l" -> videoSources.minBy { it?.size!! }?.label
    "m" -> videoSources.sortedBy { it?.size }.let { /* median */ }
    else -> videoSources.maxBy { it?.size!! }?.label  // default high
}
```

### Temporary File Management
`initializeDownloadTempDir()` creates temp directory with `segment_$index` files; tracks completed segments to support resumable downloads.

## Testing
- Tests in [src/test/kotlin/](src/test/kotlin/) use JUnit 5
- `VideoRepositoryTest.kt` validates provider implementations
- No mocking framework configured; focus on integration tests with live providers

## Common Debugging
- **Enable verbose logging**: Pass `--verbose` flag (sets `Constants.VERBOSE = true`)
- **Logger utility**: [src/main/kotlin/com/abmo/common/Logger.kt](src/main/kotlin/com/abmo/common/Logger.kt) wraps println with levels
- **JavaScript execution issues**: Check [src/main/kotlin/com/abmo/executor/JavaScriptExecutor.kt](src/main/kotlin/com/abmo/executor/JavaScriptExecutor.kt) for Rhino sandbox
- **Decryption failures**: Verify key extraction in `SimpleVideo` model matches metadata structure

## Koin Dependency Injection
All services registered in [src/main/kotlin/com/abmo/di/KoinModule.kt](src/main/kotlin/com/abmo/di/KoinModule.kt):
```kotlin
single { VideoDownloader() }  // app-wide singleton
factory { (args: Array<String>) -> CliArguments(args) }  // factory with params
```
Inject via `by inject()` or `by inject { parametersOf(...) }` in KoinComponent subclasses.

## Git Considerations
- proguard-rules.pro excludes rhino/httpcomponents to prevent obfuscation issues
- gradle.properties versioning managed centrally (koinVersion, gsonVersion, etc.)
