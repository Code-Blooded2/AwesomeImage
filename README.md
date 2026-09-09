# ⚡ Awesome Image

A lightweight Kotlin Multiplatform image loading library with a type-safe API, automatic format detection, and first-class video thumbnail support — zero dependencies.

---

## Why?

Coil and Glide accept `Any?` as their image source. A typo compiles and crashes at runtime. Awesome Image uses a typed sealed class — wrong source type is a compile error, not a crash.

Video thumbnails in other libraries download the full file before extracting a frame. Awesome Image uses HTTP range requests to fetch only what's needed. Frame time is a first-class parameter — first frame, middle frame, or any specific timestamp.

---

## How it's different

| | Coil | Glide | Awesome Image |
|---|---|---|---|
| Source API | `model: Any?` — untyped | `load(Any?)` — untyped | Typed sealed class — compile-safe |
| Video thumbnail | Plugin, full download | Full download, no frame control | Range request, first/mid/specific frame |
| Animated WebP detection | MIME type only | MIME type only | Byte-level header peek |
| Format detection | MIME → extension | MIME → extension | MIME → extension → byte peek → fallback |
| In-flight deduplication | `EngineJob` registry | `EngineJob` registry | `ConcurrentHashMap<String, Deferred>` |
| Dependencies | OkHttp + Okio | Custom HTTP stack | None — pure Android SDK |
| Binary size | ~500KB | ~500KB | ~50KB |
| Annotation processing | None | Yes (`GlideApp`) | None |
| CMP support | Android only | Android only | Android + iOS (in progress) |

---

## Installation

Add the Maven URL to your `settings.gradle.kts`:

```kotlin
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
   maven { url = uri("https://code-blooded2.github.io/AwesomeImage/") }
    }
}
```

Add the dependency:

```kotlin
implementation("awesome.image:awesome-image:1.0.0")
```

---

## Setup

Initialize once in `Application.onCreate()`:

```kotlin
class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()
        AwesomeImageLoader.setup(this) {
            copy(
                memoryCacheSizeMb = 64,
                diskCacheSizeMb   = 256,
                crossfadeDuration = 200,
                maxBitmapSize     = 2048
            )
        }
    }
}
```

---

## Quick Start

```kotlin
AwesomeImage(
    source       = Source.network("https://example.com/photo.jpg"),
    modifier     = Modifier.fillMaxWidth().height(240.dp),
    placeholder  = Source.drawable(R.drawable.placeholder),
    error        = Source.drawable(R.drawable.error),
    contentScale = ContentScale.Crop
)
```

---

## Sources

All sources are type-safe sealed class variants. No stringly-typed `data` parameter.

```kotlin
// Remote image
Source.network("https://example.com/photo.jpg")

// Remote image with custom headers
ImageSource.Network(
    url     = "https://example.com/photo.jpg",
    headers = mapOf("Authorization" to "Bearer token")
)

// Video thumbnail — first frame (default)
Source.video("https://example.com/clip.mp4")

// Video thumbnail — middle frame
Source.video("https://example.com/clip.mp4", frameTimeUs = -1L)

// Video thumbnail — specific frame at 5 seconds
Source.video("https://example.com/clip.mp4", frameTimeUs = 5_000_000L)

// Drawable resource
Source.drawable(R.drawable.ic_logo)

// Asset file
Source.asset("images/banner.webp")

// Local file path — image or video, detected automatically
Source.file("/storage/emulated/0/DCIM/photo.jpg")
Source.file("/storage/emulated/0/Movies/clip.mp4")

// Android Uri — content://, file://, mediastore://
// image or video detected automatically from MIME type
Source.uri(uri)

// Base64 encoded image
Source.base64("data:image/png;base64,iVBORw0...")
```

---

## Video Thumbnails

Video format is detected automatically from MIME type, file extension, or byte-level inspection — no flag needed on the source:

```kotlin
// Local video from gallery — detected automatically
AwesomeImage(
    source       = Source.uri(mediaStoreUri),
    modifier     = Modifier.size(120.dp),
    placeholder  = Source.drawable(R.drawable.ic_video_thumb),
    contentScale = ContentScale.Crop
)

// Local file path — detected automatically
AwesomeImage(
    source   = Source.file("/storage/Movies/clip.mp4"),
    modifier = Modifier.size(120.dp)
)

// Network video — explicit via Source.video()
AwesomeImage(
    source   = Source.video("https://example.com/clip.mp4"),
    modifier = Modifier.size(120.dp)
)
```

Frame time control — unique to Awesome Image:

```kotlin
Source.video(url)                          // first frame (default)
Source.video(url, frameTimeUs = -1L)       // middle frame
Source.video(url, frameTimeUs = 5_000_000L) // frame at 5 seconds
```

> **Note:** For network video thumbnails, the server should encode with `-movflags faststart` so the `moov` atom is at the start of the file. This allows frame extraction from the first 1MB without downloading the full video.

---

## Animated GIF & WebP

Animated formats are detected and rendered automatically — no extra configuration:

```kotlin
AwesomeImage(
    source       = Source.network("https://example.com/animation.gif"),
    modifier     = Modifier.size(200.dp),
    contentScale = ContentScale.Crop
)

AwesomeImage(
    source   = Source.asset("sticker.webp"),
    modifier = Modifier.size(120.dp)
)
```

Detection priority:
1. MIME type from `ContentResolver` or HTTP headers
2. File extension
3. Byte-level header peek — catches animated WebP misreported as static

Animated `ImageDrawable` requires API 28+. On API 27 and below, the first frame is shown as a static image.

---

## Composable API

```kotlin
@Composable
fun AwesomeImage(
    source: ImageSource,
    modifier: Modifier = Modifier,
    placeholder: ImageSource? = null,       // shown while loading
    error: ImageSource? = null,             // shown on failure
    contentScale: ContentScale = ContentScale.Crop,
    contentDescription: String? = null,
    alignment: Alignment = Alignment.Center,
    colorFilter: ColorFilter? = null,
    onState: ((ImageLoadState) -> Unit)? = null  // observe load state
)
```

Observe load state:

```kotlin
AwesomeImage(
    source  = Source.network("https://example.com/photo.jpg"),
    modifier = Modifier.size(200.dp),
    onState = { state ->
        when (state) {
            is ImageLoadState.Loading -> { /* show shimmer */ }
            is ImageLoadState.Static  -> { /* bitmap ready */ }
            is ImageLoadState.Animated -> { /* animated drawable ready */ }
            is ImageLoadState.Error   -> { /* handle error */ }
        }
    }
)
```

---

## Cache

Two-level cache — memory (LRU) and disk:

```kotlin
AwesomeImageLoader.setup(this) {
    copy(
        memoryCacheSizeMb = 64,   // default 64MB
        diskCacheSizeMb   = 256,  // default 256MB
        maxBitmapSize     = 2048  // longest edge cap in px
    )
}
```

Cache keys are typed per source — URL, resource ID, file path, asset path, and Base64 hash never collide with each other.

Clear programmatically:

```kotlin
AwesomeImageLoader.clearMemory()  // clear RAM cache only
AwesomeImageLoader.clearDisk()    // clear disk cache only
AwesomeImageLoader.clearAll()     // clear both
```

Hook into system memory warnings in your `Application`:

```kotlin
override fun onTrimMemory(level: Int) {
    super.onTrimMemory(level)
    if (level >= ComponentCallbacks2.TRIM_MEMORY_MODERATE) {
        AwesomeImageLoader.clearMemory()
    }
}
```

---

## Performance

- **`limitedParallelism(4)`** — dedicated decode dispatcher, prevents thread starvation under heavy scroll
- **In-flight deduplication** — 50 simultaneous requests to the same URL = 1 actual fetch
- **Two-pass decode** — bounds read first, then subsampled decode — avoids OOM on large gallery images
- **`RGB_565`** for gallery thumbnails — half the memory of `ARGB_8888`
- **`ParcelFileDescriptor`** for URI sources — single file descriptor open, no double stream
- **Coroutine cancellation** — `DisposableEffect` cancels in-flight decode when composable leaves composition, off-screen items never complete

---

## Format Support

| Format | Static | Animated |
|---|---|---|
| JPEG | ✅ | — |
| PNG | ✅ | — |
| WebP | ✅ | ✅ API 28+ |
| GIF | ✅ (first frame on API 27) | ✅ API 28+ |
| BMP | ✅ | — |
| HEIC / HEIF | ✅ | — |
| MP4 / MKV / WebM / MOV / AVI / 3GP | thumbnail ✅ | — |

---

## Requirements

- Kotlin 2.0+
- Jetpack Compose 1.6+
- Min SDK 26

---

## Roadmap

- [ ] Compose Multiplatform (iOS) — architecture complete, fetchers in progress
- [ ] Transformation support — circle crop, rounded corners, blur, grayscale
- [ ] Preload API — warm cache before composable is on screen
- [ ] `ImageSource.Bitmap` — wrap an existing bitmap in the composable pipeline
- [ ] GIF support on API 27 via `Movie` class

---

## License

```
Copyright 2026 By CodeBlooded (Faheem)

Licensed under the Apache License, Version 2.0
```
