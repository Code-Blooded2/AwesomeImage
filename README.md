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
| Platform | Android only | Android only | ✅ Android + iOS |
| Source API | `model: Any?` — untyped | `load(Any?)` — untyped | Typed sealed class — compile-safe |
| Video thumbnail | Plugin, full download | Full download, no frame control | Range request, first/mid/specific frame |
| Animated WebP detection | MIME type only | MIME type only | Byte-level header peek |
| Format detection | MIME → extension | MIME → extension | MIME → extension → byte peek → fallback |
| In-flight deduplication | `EngineJob` registry | `EngineJob` registry | `ConcurrentHashMap<String, Deferred>` |
| Dependencies | OkHttp + Okio | Custom HTTP stack | None — pure SDK |
| Binary size | ~500KB | ~500KB | ~50KB |
| Annotation processing | None | Yes (`GlideApp`) | None |

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
// Android only
implementation("awesome.image:awesome-image:1.0.0")

// KMP / CMP — add in commonMain, covers Android and iOS
implementation("awesome.image:awesome-image:1.0.0")
```

---

## Setup

**Android** — initialize in `Application.onCreate()`:

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

**iOS** — initialize in your Compose entry point:

```kotlin
fun MainViewController(): UIViewController = ComposeUIViewController {
    AwesomeImageLoader.setup {
        copy(
            memoryCacheSizeMb = 64,
            diskCacheSizeMb   = 256,
            crossfadeDuration = 200,
            maxBitmapSize     = 2048
        )
    }
    App()
}
```

---

## Quick Start

Same API on both platforms — no platform conditionals needed:

```kotlin
AwesomeImage(
    source       = Source.network("https://example.com/photo.jpg"),
    modifier     = Modifier.fillMaxWidth().height(240.dp),
    placeholder  = Source.drawable(R.drawable.placeholder),
    error        = Source.asset("error.png"),
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

// Drawable resource (Android only)
Source.drawable(R.drawable.ic_logo)

// Asset file — relative path from assets root (Android + iOS)
Source.asset("images/banner.webp")
Source.asset("sticker.gif")

// Compose Multiplatform resources — use Source.uri() with Res.getUri()
Source.uri(Res.getUri("files/promo.webp"))

// URI string — content://, file://, mediastore://, file:///android_asset/
Source.uri("content://media/external/images/1")
Source.uri("https://example.com/image.jpg")

// Local file path — image or video, detected automatically
Source.file("/storage/emulated/0/DCIM/photo.jpg")
Source.file("/storage/emulated/0/Movies/clip.mp4")

// Base64 encoded image
Source.base64("data:image/png;base64,iVBORw0...")
```

---

## Compose Multiplatform Resources

For CMP projects using `Res`, pass the URI directly — no path manipulation needed:

```kotlin
// ✅ correct
AwesomeImage(
    source = Source.uri(Res.getUri("files/promo.webp")),
    modifier = Modifier.fillMaxWidth()
)

// ❌ wrong — don't strip the prefix manually
Source.asset(Res.getUri("files/promo.webp").replace("file:///android_asset/", ""))
```

`Source.uri()` handles `file:///android_asset/` paths internally via `AssetManager` on Android and native file access on iOS.

---

## Video Thumbnails

Video format is detected automatically from MIME type, file extension, or byte-level inspection:

```kotlin
// Local video from gallery — detected automatically
AwesomeImage(
    source       = Source.uri("content://media/external/video/1"),
    modifier     = Modifier.size(120.dp),
    placeholder  = Source.drawable(R.drawable.ic_video_thumb),
    contentScale = ContentScale.Crop
)

// Local file path — detected automatically
AwesomeImage(
    source   = Source.file("/path/to/clip.mp4"),
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
Source.video(url)                           // first frame (default)
Source.video(url, frameTimeUs = -1L)        // middle frame
Source.video(url, frameTimeUs = 5_000_000L) // frame at 5 seconds
```

> **Note:** For network video thumbnails, encode with `-movflags faststart` so the `moov` atom is at the start of the file. This allows frame extraction from the first 1MB without downloading the full video.

---

## Animated GIF & WebP

Animated formats are detected and rendered automatically on both platforms — no extra configuration:

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
1. MIME type from `ContentResolver` (Android) or HTTP headers
2. File extension
3. Byte-level header peek — catches animated WebP misreported as static

| Platform | Animated GIF | Animated WebP |
|---|---|---|
| Android | ✅ API 28+ (first frame on 27) | ✅ API 28+ |
| iOS | ✅ all versions via `CGImageSource` | ✅ all versions via `CGImageSource` |

---

## Platform Notes

| Source | Android | iOS |
|---|---|---|
| `Source.network()` | `HttpURLConnection` | `NSURLSession` |
| `Source.video()` | `MediaMetadataRetriever` | `AVAssetImageGenerator` |
| `Source.file()` | `BitmapFactory` | `UIImage` |
| `Source.asset()` | `AssetManager` | `NSBundle.mainBundle` |
| `Source.uri()` | `ContentResolver` + `AssetManager` | File path or URL string |
| `Source.drawable()` | `R.drawable` resource ID | Named image in asset catalog |
| `Source.base64()` | `Base64.decode` | `NSData` base64 |
| Animated GIF/WebP | `ImageDecoder` API 28+ | `CGImageSource` |
| Memory cache | LRU `LinkedHashMap` | `NSCache` — auto-evicts under pressure |

---

## Composable API

Same composable on both platforms:

```kotlin
@Composable
fun AwesomeImage(
    source: ImageSource,
    modifier: Modifier = Modifier,
    placeholder: ImageSource? = null,
    error: ImageSource? = null,
    contentScale: ContentScale = ContentScale.Crop,
    contentDescription: String? = null,
    alignment: Alignment = Alignment.Center,
    colorFilter: ColorFilter? = null,
    onState: ((ImageLoadState) -> Unit)? = null
)
```

Observe load state:

```kotlin
AwesomeImage(
    source   = Source.network("https://example.com/photo.jpg"),
    modifier = Modifier.size(200.dp),
    onState  = { state ->
        when (state) {
            is ImageLoadState.Loading  -> { /* show shimmer */ }
            is ImageLoadState.Static   -> { /* bitmap ready */ }
            is ImageLoadState.Animated -> { /* animated drawable ready */ }
            is ImageLoadState.Error    -> { /* handle error */ }
        }
    }
)
```

---

## Cache

Two-level cache — memory and disk:

```kotlin
AwesomeImageLoader.clearMemory()  // clear RAM cache only
AwesomeImageLoader.clearDisk()    // clear disk cache only
AwesomeImageLoader.clearAll()     // clear both
```

On Android, hook into system memory warnings:

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

- **`limitedParallelism(4)`** — dedicated decode dispatcher on both platforms
- **In-flight deduplication** — same URL requested 50 times = 1 actual fetch
- **Two-pass decode** — bounds read first, then subsampled — avoids OOM on large images
- **`RGB_565`** for Android gallery thumbnails — half the memory of `ARGB_8888`
- **`ParcelFileDescriptor`** for Android URI sources — single file descriptor, no double stream
- **`CGImageSource`** on iOS — hardware-accelerated frame decode
- **Coroutine cancellation** — `DisposableEffect` cancels in-flight decode when composable leaves composition

---

## Format Support

| Format | Android Static | Android Animated | iOS Static | iOS Animated |
|---|---|---|---|---|
| JPEG | ✅ | — | ✅ | — |
| PNG | ✅ | — | ✅ | — |
| WebP | ✅ | ✅ API 28+ | ✅ | ✅ |
| GIF | ✅ | ✅ API 28+ | ✅ | ✅ |
| BMP | ✅ | — | ✅ | — |
| HEIC / HEIF | ✅ | — | ✅ | — |
| MP4 / MKV / WebM / MOV / AVI / 3GP | thumbnail ✅ | — | thumbnail ✅ | — |

---

## Requirements

- Kotlin 2.0+
- Compose Multiplatform 1.8+
- Android Min SDK 26
- iOS 16+

---

## Roadmap

- [ ] Transformation support — circle crop, rounded corners, blur, grayscale
- [ ] Preload API — warm cache before composable is on screen
- [ ] `ImageSource.Bitmap` — wrap an existing bitmap in the composable pipeline
- [ ] GIF support on Android API 27 via `Movie` class

---

## License

```
Copyright 2026 By CodeBlooded (Faheem)

Licensed under the Apache License, Version 2.0
```
