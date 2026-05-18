# Image Loading — Coil 3, Custom Loading States, Performance

Images are frequently the largest performance variable in an Android app. A poorly configured image loader produces OOM crashes, janky scrolling, and wasted bandwidth. A well-configured one is invisible — images appear instantly, scroll is buttery, and memory stays stable.

---

## Coil 3 Setup

```toml
[versions]
coil = "3.0.4"

[libraries]
coil-compose     = { group = "io.coil-kt.coil3", name = "coil-compose", version.ref = "coil" }
coil-network-okhttp = { group = "io.coil-kt.coil3", name = "coil-network-okhttp", version.ref = "coil" }
coil-video       = { group = "io.coil-kt.coil3", name = "coil-video", version.ref = "coil" }
coil-svg         = { group = "io.coil-kt.coil3", name = "coil-svg", version.ref = "coil" }
coil-gif         = { group = "io.coil-kt.coil3", name = "coil-gif", version.ref = "coil" }
```

### ImageLoader configuration

```kotlin
// core/ui/src/main/kotlin/.../ImageLoaderFactory.kt
class AppImageLoaderFactory @Inject constructor(
    @ApplicationContext private val context: Context,
    private val okHttpClient: OkHttpClient
) : ImageLoaderFactory {

    override fun newImageLoader(): ImageLoader = ImageLoader.Builder(context)
        .components {
            add(OkHttpNetworkFetcherFactory(callFactory = { okHttpClient }))
            add(VideoFrameDecoder.Factory())   // for video thumbnails
            add(SvgDecoder.Factory())          // for SVG support
            add(ImageDecoderDecoder.Factory()) // for animated GIF/AVIF/WebP
        }
        .memoryCache {
            MemoryCache.Builder()
                .maxSizePercent(context, percent = 0.25) // 25% of available memory
                .build()
        }
        .diskCache {
            DiskCache.Builder()
                .directory(context.cacheDir.resolve("image_cache"))
                .maxSizeBytes(50L * 1024 * 1024) // 50MB disk cache
                .build()
        }
        .respectCacheHeaders(false) // ignore Cache-Control from servers that misconfigure it
        .crossfade(true)
        .build()
}

// Register in Application
@HiltAndroidApp
class AppApplication : Application(), ImageLoaderFactory {
    @Inject lateinit var imageLoaderFactory: AppImageLoaderFactory

    override fun newImageLoader(): ImageLoader = imageLoaderFactory.newImageLoader()
}
```

---

## AsyncImage — basic usage

```kotlin
@Composable
fun ProductImage(
    imageUrl: String,
    contentDescription: String?,
    modifier: Modifier = Modifier
) {
    AsyncImage(
        model = ImageRequest.Builder(LocalContext.current)
            .data(imageUrl)
            .crossfade(300)
            .size(400, 400)          // explicit size → downsample to save memory
            .memoryCacheKey(imageUrl)
            .diskCacheKey(imageUrl)
            .build(),
        contentDescription = contentDescription,
        contentScale = ContentScale.Crop,
        modifier = modifier
    )
}
```

---

## SubcomposeAsyncImage — custom loading states

Use `SubcomposeAsyncImage` when you need full control over loading, success, and error composables:

```kotlin
@Composable
fun RichImage(
    imageUrl: String,
    contentDescription: String?,
    modifier: Modifier = Modifier,
    placeholderShape: Shape = MaterialTheme.shapes.medium
) {
    SubcomposeAsyncImage(
        model = ImageRequest.Builder(LocalContext.current)
            .data(imageUrl)
            .crossfade(true)
            .build(),
        contentDescription = contentDescription,
        modifier = modifier
    ) {
        val state = painter.state

        when {
            state is AsyncImagePainter.State.Loading -> {
                // Skeleton shimmer matching the image dimensions
                ShimmerBox(modifier = Modifier.fillMaxSize(), shape = placeholderShape)
            }
            state is AsyncImagePainter.State.Error -> {
                // Error state with retry
                ImageErrorPlaceholder(
                    onRetry = { painter.restart() },
                    modifier = Modifier.fillMaxSize()
                )
            }
            else -> {
                // Successful load — fade in
                SubcomposeAsyncImageContent(
                    modifier = Modifier
                        .fillMaxSize()
                        .clip(placeholderShape)
                )
            }
        }
    }
}

@Composable
private fun ImageErrorPlaceholder(onRetry: () -> Unit, modifier: Modifier = Modifier) {
    Box(
        modifier = modifier
            .background(MaterialTheme.colorScheme.surfaceVariant)
            .clickable(onClick = onRetry),
        contentAlignment = Alignment.Center
    ) {
        Column(horizontalAlignment = Alignment.CenterHorizontally) {
            Icon(
                imageVector = Icons.Default.BrokenImage,
                contentDescription = null,
                tint = MaterialTheme.colorScheme.onSurfaceVariant
            )
            Spacer(modifier = Modifier.height(AppSpacing.XS))
            Text(
                text = "Tap to retry",
                style = MaterialTheme.typography.labelSmall,
                color = MaterialTheme.colorScheme.onSurfaceVariant
            )
        }
    }
}
```

---

## LazyList Performance — Critical Rules

### Problem: each LazyList item loads full resolution image

```kotlin
// BAD — loads whatever resolution the server returns for every item
AsyncImage(
    model = article.imageUrl,
    contentDescription = null,
    modifier = Modifier.size(80.dp) // displayed at 80dp but potentially loaded at 2000px
)

// GOOD — tell Coil the target display size; it downsamples before decoding
AsyncImage(
    model = ImageRequest.Builder(LocalContext.current)
        .data(article.imageUrl)
        .size(
            width = 80.dp.toPxInt(),  // approximate pixel size
            height = 80.dp.toPxInt()
        )
        .build(),
    contentDescription = null,
    modifier = Modifier.size(80.dp)
)
```

### Problem: placeholders cause layout shift

```kotlin
// BAD — placeholder is a different size from the loaded image → layout shift
AsyncImage(
    model = imageUrl,
    placeholder = painterResource(R.drawable.ic_placeholder), // small icon
    modifier = Modifier.fillMaxWidth().wrapContentHeight() // height collapses then expands
)

// GOOD — enforce fixed size before and after load
AsyncImage(
    model = imageUrl,
    contentDescription = null,
    modifier = Modifier
        .fillMaxWidth()
        .height(200.dp)  // fixed height — no layout shift
)
```

### Problem: recomposition causes image flicker in lists

```kotlin
// BAD — image reloads when the composable recomposes because key changes
@Composable
fun ArticleCard(article: Article) {
    val timestamp = System.currentTimeMillis() // new key every recompose
    AsyncImage(
        model = ImageRequest.Builder(LocalContext.current)
            .data(article.imageUrl)
            .memoryCacheKey("$timestamp-${article.id}") // defeats caching
            .build(),
        ...
    )
}

// GOOD — stable cache key based on stable data
@Composable
fun ArticleCard(article: Article) {
    AsyncImage(
        model = ImageRequest.Builder(LocalContext.current)
            .data(article.imageUrl)
            .memoryCacheKey(article.imageUrl)  // stable
            .diskCacheKey(article.imageUrl)
            .build(),
        ...
    )
}
```

---

## Transformations

```kotlin
AsyncImage(
    model = ImageRequest.Builder(LocalContext.current)
        .data(userAvatarUrl)
        .transformations(
            CircleCropTransformation()    // circular avatar
        )
        .build(),
    contentDescription = "User avatar",
    modifier = Modifier.size(48.dp)
)

// Rounded corners via transformation (when Modifier.clip() would cause visual issues)
AsyncImage(
    model = ImageRequest.Builder(LocalContext.current)
        .data(imageUrl)
        .transformations(
            RoundedCornersTransformation(radius = 16f.dpToPx())
        )
        .build(),
    ...
)

// Multiple transformations applied in order
AsyncImage(
    model = ImageRequest.Builder(LocalContext.current)
        .data(imageUrl)
        .transformations(
            BlurTransformation(radius = 20, scale = 0.5f), // downsample then blur for perf
            RoundedCornersTransformation(radius = 8f.dpToPx())
        )
        .build(),
    ...
)
```

---

## Video Thumbnails

```kotlin
@Composable
fun VideoThumbnail(videoUrl: String, modifier: Modifier = Modifier) {
    AsyncImage(
        model = ImageRequest.Builder(LocalContext.current)
            .data(videoUrl)
            .decoderFactory(VideoFrameDecoder.Factory())
            .videoFrameMillis(1000) // grab frame at 1 second
            .build(),
        contentDescription = null,
        contentScale = ContentScale.Crop,
        modifier = modifier
    )
}
```

---

## SVG and Animated Images

```kotlin
// SVG — vector from URL
AsyncImage(
    model = ImageRequest.Builder(LocalContext.current)
        .data("https://example.com/logo.svg")
        .decoderFactory(SvgDecoder.Factory())
        .build(),
    contentDescription = "Logo",
    modifier = Modifier.size(48.dp)
)

// Animated GIF/WebP/AVIF
AsyncImage(
    model = ImageRequest.Builder(LocalContext.current)
        .data("https://example.com/animation.gif")
        .decoderFactory(ImageDecoderDecoder.Factory())
        .build(),
    contentDescription = null,
    modifier = Modifier.size(200.dp)
)
```

---

## Preloading Images

Preload images that are about to enter the viewport (e.g., next page items):

```kotlin
@Composable
fun PreloadImages(imageUrls: List<String>) {
    val context = LocalContext.current
    val imageLoader = LocalImageLoader.current

    LaunchedEffect(imageUrls) {
        imageUrls.forEach { url ->
            val request = ImageRequest.Builder(context)
                .data(url)
                .memoryCachePolicy(CachePolicy.ENABLED)
                .diskCachePolicy(CachePolicy.ENABLED)
                .build()
            imageLoader.execute(request)
        }
    }
}
```

---

## Cache Management

```kotlin
// Clear disk cache (e.g., on logout to remove user content)
fun clearImageCache(context: Context) {
    SingletonImageLoader.get(context).diskCache?.clear()
    SingletonImageLoader.get(context).memoryCache?.clear()
}

// Evict specific URL from cache (after updating a profile picture)
fun evictFromCache(context: Context, imageUrl: String) {
    val imageLoader = SingletonImageLoader.get(context)
    imageLoader.diskCache?.remove(imageUrl)
    imageLoader.memoryCache?.remove(MemoryCache.Key(imageUrl))
}
```
