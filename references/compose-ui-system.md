# Compose UI System — Design System Reference

Every screen you build inherits from this system. Deviation requires a reason. The goal is coherence: a user moving through different features of your app should feel like they're moving through one product, not a collection of screens built by different people on different days.

---

## Design Token Architecture — Three Layers

### Layer 1: Primitive Tokens

Raw values. Never used directly in composables — they exist only to feed Layer 2.

```kotlin
// core/ui/src/main/kotlin/com/example/core/ui/tokens/PrimitiveTokens.kt
internal object PrimitiveTokens {
    // Color primitives
    val Teal10 = Color(0xFF002021)
    val Teal40 = Color(0xFF00696C)
    val Teal80 = Color(0xFF4DD8DB)
    val Teal90 = Color(0xFF6FF4F7)

    val Slate10 = Color(0xFF1A1C1E)
    val Slate20 = Color(0xFF2E3133)
    val Slate90 = Color(0xFFE1E3E5)
    val Slate99 = Color(0xFFFBFCFE)

    val Error40 = Color(0xFFBA1A1A)
    val Error80 = Color(0xFFFFB4AB)

    // Spacing primitives (multiples of 4dp)
    val Space4  = 4.dp
    val Space8  = 8.dp
    val Space12 = 12.dp
    val Space16 = 16.dp
    val Space20 = 20.dp
    val Space24 = 24.dp
    val Space32 = 32.dp
    val Space40 = 40.dp
    val Space48 = 48.dp
    val Space64 = 64.dp

    // Shape primitives
    val Radius4  = 4.dp
    val Radius8  = 8.dp
    val Radius12 = 12.dp
    val Radius16 = 16.dp
    val Radius24 = 24.dp
    val Radius28 = 28.dp
}
```

### Layer 2: Semantic Tokens

Role-based names. These are what you use in component implementations.

```kotlin
// core/ui/src/main/kotlin/com/example/core/ui/tokens/SemanticTokens.kt
object AppSpacing {
    val XS  = PrimitiveTokens.Space4   // 4dp  — icon padding, dense chip insets
    val S   = PrimitiveTokens.Space8   // 8dp  — related items, list item gaps
    val M   = PrimitiveTokens.Space16  // 16dp — screen horizontal padding
    val L   = PrimitiveTokens.Space24  // 24dp — section gaps
    val XL  = PrimitiveTokens.Space32  // 32dp — major section separators
    val XXL = PrimitiveTokens.Space48  // 48dp — hero padding
}

object AppRadius {
    val ExtraSmall = PrimitiveTokens.Radius4   // chip, badge
    val Small      = PrimitiveTokens.Radius8   // button, text field
    val Medium     = PrimitiveTokens.Radius12  // card, dialog
    val Large      = PrimitiveTokens.Radius16  // bottom sheet, expanded card
    val ExtraLarge = PrimitiveTokens.Radius28  // large surface, navigation drawer
    val Full       = 50.dp                     // FAB, avatar, pill
}
```

### Layer 3: Component Tokens

Component-specific overrides. Keep these in the composable file that owns the component, not in a global token file — they're private details, not shared contracts.

```kotlin
// Within the component file, not exported:
private object CardTokens {
    val HorizontalPadding = AppSpacing.M
    val VerticalPadding = AppSpacing.S
    val Elevation = 1.dp
    val ImageAspectRatio = 16f / 9f
}
```

---

## Material 3 Theme Setup

### Complete Theme

```kotlin
// core/ui/src/main/kotlin/com/example/core/ui/theme/AppTheme.kt

private val LightColorScheme = lightColorScheme(
    primary          = PrimitiveTokens.Teal40,
    onPrimary        = Color.White,
    primaryContainer = PrimitiveTokens.Teal90,
    onPrimaryContainer = PrimitiveTokens.Teal10,
    secondary        = Color(0xFF4A6364),
    onSecondary      = Color.White,
    secondaryContainer = Color(0xFFCCE8E9),
    onSecondaryContainer = Color(0xFF051F20),
    error            = PrimitiveTokens.Error40,
    surface          = PrimitiveTokens.Slate99,
    onSurface        = PrimitiveTokens.Slate10,
    surfaceVariant   = Color(0xFFDAE4E5),
    onSurfaceVariant = Color(0xFF3F4849),
    outline          = Color(0xFF6F797A),
)

private val DarkColorScheme = darkColorScheme(
    primary          = PrimitiveTokens.Teal80,
    onPrimary        = Color(0xFF003738),
    primaryContainer = Color(0xFF004F52),
    onPrimaryContainer = PrimitiveTokens.Teal90,
    secondary        = Color(0xFFB0CCCD),
    onSecondary      = Color(0xFF1B3435),
    surface          = PrimitiveTokens.Slate10,
    onSurface        = PrimitiveTokens.Slate90,
    surfaceVariant   = PrimitiveTokens.Slate20,
    onSurfaceVariant = Color(0xFFBEC8C9),
)

@Composable
fun AppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    dynamicColor: Boolean = true,
    content: @Composable () -> Unit
) {
    val colorScheme = when {
        dynamicColor && Build.VERSION.SDK_INT >= Build.VERSION_CODES.S -> {
            val context = LocalContext.current
            if (darkTheme) dynamicDarkColorScheme(context) else dynamicLightColorScheme(context)
        }
        darkTheme -> DarkColorScheme
        else -> LightColorScheme
    }

    MaterialTheme(
        colorScheme = colorScheme,
        typography = AppTypography,
        shapes = AppShapes,
        content = content
    )
}
```

### Custom CompositionLocals for Tokens Material 3 Doesn't Cover

```kotlin
// Elevation levels not exposed by MaterialTheme
val LocalElevation = staticCompositionLocalOf { AppElevation() }

data class AppElevation(
    val none: Dp = 0.dp,
    val subtle: Dp = 1.dp,
    val card: Dp = 2.dp,
    val modal: Dp = 8.dp,
)

// Usage:
val elevation = LocalElevation.current.card
```

---

## Typography

### Font Loading

```kotlin
// core/ui/src/main/res/font/inter_regular.ttf  (body font)
// core/ui/src/main/res/font/inter_medium.ttf
// core/ui/src/main/res/font/inter_semibold.ttf
// core/ui/src/main/res/font/fraunces_bold.ttf  (display font — creates personality)

private val InterFontFamily = FontFamily(
    Font(R.font.inter_regular, FontWeight.Normal),
    Font(R.font.inter_medium, FontWeight.Medium),
    Font(R.font.inter_semibold, FontWeight.SemiBold),
)

private val FrauncesDisplay = FontFamily(
    Font(R.font.fraunces_bold, FontWeight.Bold),
)

val AppTypography = Typography(
    displayLarge = TextStyle(
        fontFamily = FrauncesDisplay,
        fontWeight = FontWeight.Bold,
        fontSize = 57.sp,
        lineHeight = 64.sp,
        letterSpacing = (-0.25).sp
    ),
    displayMedium = TextStyle(
        fontFamily = FrauncesDisplay,
        fontWeight = FontWeight.Bold,
        fontSize = 45.sp,
        lineHeight = 52.sp,
        letterSpacing = 0.sp
    ),
    headlineLarge = TextStyle(
        fontFamily = FrauncesDisplay,
        fontWeight = FontWeight.Bold,
        fontSize = 32.sp,
        lineHeight = 40.sp,
        letterSpacing = 0.sp
    ),
    headlineMedium = TextStyle(
        fontFamily = InterFontFamily,
        fontWeight = FontWeight.SemiBold,
        fontSize = 28.sp,
        lineHeight = 36.sp,
        letterSpacing = 0.sp
    ),
    titleLarge = TextStyle(
        fontFamily = InterFontFamily,
        fontWeight = FontWeight.SemiBold,
        fontSize = 22.sp,
        lineHeight = 28.sp,
        letterSpacing = 0.sp
    ),
    titleMedium = TextStyle(
        fontFamily = InterFontFamily,
        fontWeight = FontWeight.Medium,
        fontSize = 16.sp,
        lineHeight = 24.sp,
        letterSpacing = 0.15.sp
    ),
    bodyLarge = TextStyle(
        fontFamily = InterFontFamily,
        fontWeight = FontWeight.Normal,
        fontSize = 16.sp,
        lineHeight = 24.sp,
        letterSpacing = 0.5.sp
    ),
    bodyMedium = TextStyle(
        fontFamily = InterFontFamily,
        fontWeight = FontWeight.Normal,
        fontSize = 14.sp,
        lineHeight = 20.sp,
        letterSpacing = 0.25.sp
    ),
    labelLarge = TextStyle(
        fontFamily = InterFontFamily,
        fontWeight = FontWeight.Medium,
        fontSize = 14.sp,
        lineHeight = 20.sp,
        letterSpacing = 0.1.sp
    ),
    labelSmall = TextStyle(
        fontFamily = InterFontFamily,
        fontWeight = FontWeight.Medium,
        fontSize = 11.sp,
        lineHeight = 16.sp,
        letterSpacing = 0.5.sp
    ),
)
```

**Font pairing principle**: A display face (Fraunces, Playfair, Cormorant, Canela) for headlines gives personality and signals premium quality. A humanist sans (Inter, DM Sans, Plus Jakarta Sans) for body text maintains readability across sizes. The contrast between the two creates typographic hierarchy you can *feel* when scrolling, not just read.

---

## Shape System

```kotlin
// core/ui/src/main/kotlin/com/example/core/ui/theme/AppShapes.kt

val AppShapes = Shapes(
    extraSmall = RoundedCornerShape(AppRadius.ExtraSmall), // 4dp — chips, badges
    small      = RoundedCornerShape(AppRadius.Small),      // 8dp — buttons, fields
    medium     = RoundedCornerShape(AppRadius.Medium),     // 12dp — cards
    large      = RoundedCornerShape(AppRadius.Large),      // 16dp — bottom sheets
    extraLarge = RoundedCornerShape(AppRadius.ExtraLarge), // 28dp — dialogs
)
```

### Custom Shapes for Non-Rectangular Clipping

```kotlin
// Ticket punch-out shape (circle cut from top-left corner)
val TicketShape = GenericShape { size, _ ->
    val punchRadius = 16.dp.toPx()
    moveTo(punchRadius, 0f)
    arcTo(
        rect = Rect(0f, -punchRadius, punchRadius * 2, punchRadius),
        startAngleDegrees = 0f,
        sweepAngleDegrees = -180f,
        forceMoveTo = false
    )
    lineTo(0f, size.height)
    lineTo(size.width, size.height)
    lineTo(size.width, 0f)
    close()
}

// Wave bottom edge
val WaveBottomShape = GenericShape { size, _ ->
    moveTo(0f, 0f)
    lineTo(size.width, 0f)
    lineTo(size.width, size.height * 0.85f)
    quadraticBezierTo(
        size.width * 0.75f, size.height,
        size.width * 0.5f, size.height * 0.85f
    )
    quadraticBezierTo(
        size.width * 0.25f, size.height * 0.7f,
        0f, size.height * 0.85f
    )
    close()
}
```

---

## Color Roles — When to Use Each

| Role | Use |
|------|-----|
| `primary` | Most important action, active nav indicator |
| `onPrimary` | Content on `primary` backgrounds |
| `primaryContainer` | Tinted backgrounds for selected/highlighted states |
| `secondary` | Supporting actions, complementary accents |
| `surface` | Default screen background |
| `surfaceVariant` | Differentiated surfaces: sidebars, input fills |
| `surfaceContainerLow/High` | Layered cards with distinct elevation tones |
| `outline` | Borders, dividers, unfocused field strokes |
| `outlineVariant` | Subtle separators, low-emphasis dividers |
| `error` | Destructive actions, validation errors only |

Never use `primary` for backgrounds unless it's a floating action button or a filled button. Never hardcode colors outside the token system — if Material 3 doesn't have a role for what you need, create a CompositionLocal.

---

## Brush & Gradient System

```kotlin
// core/ui/src/main/kotlin/com/example/core/ui/brush/AppBrushes.kt

object AppBrushes {

    fun heroGradient(colorScheme: ColorScheme): Brush =
        Brush.verticalGradient(
            colors = listOf(
                Color.Transparent,
                colorScheme.surface.copy(alpha = 0.6f),
                colorScheme.surface
            ),
            startY = 0f
        )

    fun primaryGradient(colorScheme: ColorScheme): Brush =
        Brush.linearGradient(
            colors = listOf(
                colorScheme.primary,
                colorScheme.tertiary ?: colorScheme.secondary
            ),
            start = Offset(0f, 0f),
            end = Offset(Float.POSITIVE_INFINITY, Float.POSITIVE_INFINITY)
        )

    fun radialGlow(color: Color, radius: Float): Brush =
        Brush.radialGradient(
            colors = listOf(color.copy(alpha = 0.3f), Color.Transparent),
            radius = radius
        )
}

// Usage — gradient text:
Text(
    text = "Gradient Headline",
    style = MaterialTheme.typography.displayMedium,
    modifier = Modifier.drawWithCache {
        val gradient = Brush.horizontalGradient(
            colors = listOf(Color(0xFF6C63FF), Color(0xFFFF6584))
        )
        onDrawWithContent {
            drawWithLayer {
                drawContent()
                drawRect(brush = gradient, blendMode = BlendMode.SrcAtop)
            }
        }
    }
)
```

### Glass/Frosted Surface

```kotlin
// Frosted glass card using paint modifier with BlurMaskFilter
@Composable
fun GlassCard(
    modifier: Modifier = Modifier,
    content: @Composable BoxScope.() -> Unit
) {
    Box(
        modifier = modifier
            .clip(MaterialTheme.shapes.large)
            .background(
                MaterialTheme.colorScheme.surface.copy(alpha = 0.65f)
            )
            .paint(
                painter = rememberVectorPainter(image = ImageVector.vectorResource(R.drawable.ic_blur_bg)),
                contentScale = ContentScale.Crop,
                alpha = 0.15f
            )
            .border(
                width = 1.dp,
                brush = Brush.verticalGradient(
                    listOf(Color.White.copy(alpha = 0.3f), Color.Transparent)
                ),
                shape = MaterialTheme.shapes.large
            )
            .padding(AppSpacing.M),
        content = content
    )
}
```

---

## Spacing & Layout Rhythm

All spacing must be a multiple of 4dp. Deviation is a bug, not a style choice.

```
Screen horizontal padding: AppSpacing.M (16dp)
List item vertical padding: AppSpacing.S (8dp) top + bottom
Section gap: AppSpacing.L (24dp)
Related items gap: AppSpacing.S (8dp)
Icon-to-label gap: AppSpacing.XS (4dp)
FAB bottom padding: AppSpacing.L (24dp) + nav bar inset
```

```kotlin
// Correct insets usage — always consume, never ignore
Modifier
    .fillMaxSize()
    .padding(
        top = WindowInsets.statusBars.asPaddingValues().calculateTopPadding(),
        bottom = WindowInsets.navigationBars.asPaddingValues().calculateBottomPadding()
    )

// Or with the convenience extension:
Modifier
    .fillMaxSize()
    .safeDrawingPadding()  // applies both statusBars + navigationBars + displayCutout
```

---

## Component Patterns

### Screen-Level Composable Convention

Each screen has one public function. Internal sections are private `@Composable` functions not exported from the module. The ViewModel provides the state; the screen function owns no business logic.

```kotlin
// Public — one per screen
@Composable
fun ProfileScreen(
    onNavigateToSettings: () -> Unit,
    modifier: Modifier = Modifier,
    viewModel: ProfileViewModel = hiltViewModel()
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    ProfileScreenContent(
        uiState = uiState,
        onAction = viewModel::onAction,
        onNavigateToSettings = onNavigateToSettings,
        modifier = modifier
    )
}

// Internal — testable without ViewModel
@Composable
private fun ProfileScreenContent(
    uiState: ProfileUiState,
    onAction: (ProfileAction) -> Unit,
    onNavigateToSettings: () -> Unit,
    modifier: Modifier = Modifier
) {
    when (uiState) {
        is ProfileUiState.Loading -> ProfileSkeleton(modifier)
        is ProfileUiState.Error   -> ProfileError(uiState.message, onAction, modifier)
        is ProfileUiState.Ready   -> ProfileReady(uiState, onAction, onNavigateToSettings, modifier)
    }
}
```

### Slot-Based API

Use `content: @Composable () -> Unit` slots when the component is a container whose inner content varies. Use explicit parameters when the component knows its own content.

```kotlin
// Slot API — container doesn't know what goes inside
@Composable
fun AppBottomSheet(
    onDismiss: () -> Unit,
    modifier: Modifier = Modifier,
    header: @Composable ColumnScope.() -> Unit = {},
    content: @Composable ColumnScope.() -> Unit
) {
    ModalBottomSheet(onDismissRequest = onDismiss) {
        Column(modifier = modifier.padding(horizontal = AppSpacing.M)) {
            header()
            Spacer(modifier = Modifier.height(AppSpacing.S))
            content()
            Spacer(modifier = Modifier.windowInsetsBottomHeight(WindowInsets.navigationBars))
        }
    }
}
```

### State Hoisting — Three Levels

**Level 1 — Fully stateless**: Receives all state as parameters, emits all events as lambda callbacks. Use for any component that might be previewed or tested in isolation.

**Level 2 — Internal hoisted state**: `rememberXxxState()` pattern. The component owns internal state (e.g., animation progress, scroll position) but delegates business-meaningful events upward.

**Level 3 — ViewModel-owned**: State lives in the ViewModel; composable subscribes with `collectAsStateWithLifecycle()`. Only the screen-level function touches the ViewModel.

---

## Preview Conventions

Every public `@Composable` needs all four preview variants. No exceptions.

```kotlin
@Preview(name = "Light", showBackground = true)
@Preview(name = "Dark", uiMode = Configuration.UI_MODE_NIGHT_YES, showBackground = true)
@Preview(name = "Large font", fontScale = 1.5f, showBackground = true)
@Preview(name = "RTL", locale = "ar", showBackground = true)
@Composable
private fun ProductCardPreview() {
    AppTheme {
        ProductCard(
            product = PreviewData.sampleProduct,
            onClick = {}
        )
    }
}
```

Provide preview data objects with realistic values, not "Lorem ipsum" or null-safe defaults. A preview with blank text teaches you nothing about how the component behaves.

---

## Advanced Visual Techniques

### Parallax Scroll Effect

```kotlin
@Composable
fun ParallaxHeroImage(
    imageUrl: String,
    scrollState: ScrollState,
    modifier: Modifier = Modifier
) {
    val parallaxFraction = 0.4f
    val offset by remember {
        derivedStateOf { scrollState.value * parallaxFraction }
    }

    Box(modifier = modifier.clipToBounds()) {
        AsyncImage(
            model = imageUrl,
            contentDescription = null,
            contentScale = ContentScale.Crop,
            modifier = Modifier
                .fillMaxWidth()
                .height(300.dp)
                .graphicsLayer { translationY = offset }
        )
    }
}
```

### Gradient Mesh Background

```kotlin
@Composable
fun MeshBackground(modifier: Modifier = Modifier) {
    val primary = MaterialTheme.colorScheme.primaryContainer
    val secondary = MaterialTheme.colorScheme.secondaryContainer
    val tertiary = MaterialTheme.colorScheme.tertiaryContainer

    Box(
        modifier = modifier
            .fillMaxSize()
            .drawBehind {
                // Layer multiple radial gradients at different positions
                drawRect(color = MaterialTheme.colorScheme.surface.copy(alpha = 1f))
                drawCircle(
                    brush = Brush.radialGradient(
                        colors = listOf(primary.copy(alpha = 0.6f), Color.Transparent),
                        radius = size.width * 0.7f,
                        center = Offset(0f, 0f)
                    ),
                    radius = size.width * 0.7f,
                    center = Offset(0f, 0f)
                )
                drawCircle(
                    brush = Brush.radialGradient(
                        colors = listOf(secondary.copy(alpha = 0.4f), Color.Transparent),
                        radius = size.width * 0.6f,
                        center = Offset(size.width, size.height * 0.3f)
                    ),
                    radius = size.width * 0.6f,
                    center = Offset(size.width, size.height * 0.3f)
                )
                drawCircle(
                    brush = Brush.radialGradient(
                        colors = listOf(tertiary.copy(alpha = 0.35f), Color.Transparent),
                        radius = size.width * 0.5f,
                        center = Offset(size.width * 0.3f, size.height)
                    ),
                    radius = size.width * 0.5f,
                    center = Offset(size.width * 0.3f, size.height)
                )
            }
    )
}
```

### Skeleton Shimmer

Mirror the exact geometry of real content. A shimmer that doesn't match the content layout is worse than a spinner — it creates layout shift when the content arrives.

```kotlin
@Composable
fun ShimmerBox(
    modifier: Modifier = Modifier,
    shape: Shape = MaterialTheme.shapes.medium
) {
    val transition = rememberInfiniteTransition(label = "shimmer")
    val shimmerTranslateAnim by transition.animateFloat(
        initialValue = -1f,
        targetValue = 2f,
        animationSpec = infiniteRepeatable(
            animation = tween(durationMillis = 1200, easing = LinearEasing),
            repeatMode = RepeatMode.Restart
        ),
        label = "shimmer_translate"
    )

    val shimmerBrush = Brush.linearGradient(
        colors = listOf(
            MaterialTheme.colorScheme.surfaceVariant,
            MaterialTheme.colorScheme.surfaceVariant.copy(alpha = 0.4f),
            MaterialTheme.colorScheme.surfaceVariant,
        ),
        start = Offset(shimmerTranslateAnim * 1000f, shimmerTranslateAnim * 500f),
        end = Offset(shimmerTranslateAnim * 1000f + 600f, shimmerTranslateAnim * 500f + 300f)
    )

    Box(
        modifier = modifier
            .clip(shape)
            .background(shimmerBrush)
    )
}

// Skeleton layout that mirrors a real card
@Composable
fun ArticleCardSkeleton(modifier: Modifier = Modifier) {
    Card(modifier = modifier.fillMaxWidth().height(180.dp)) {
        Row(modifier = Modifier.padding(AppSpacing.M)) {
            ShimmerBox(
                modifier = Modifier.size(80.dp),
                shape = MaterialTheme.shapes.small
            )
            Spacer(modifier = Modifier.width(AppSpacing.M))
            Column(verticalArrangement = Arrangement.spacedBy(AppSpacing.S)) {
                ShimmerBox(modifier = Modifier.fillMaxWidth(0.75f).height(16.dp))
                ShimmerBox(modifier = Modifier.fillMaxWidth(0.5f).height(16.dp))
                ShimmerBox(modifier = Modifier.fillMaxWidth(0.9f).height(12.dp))
            }
        }
    }
}
```

### Animated Gradient Text

```kotlin
@Composable
fun GradientText(
    text: String,
    style: TextStyle = MaterialTheme.typography.headlineLarge,
    modifier: Modifier = Modifier
) {
    val gradientColors = listOf(
        MaterialTheme.colorScheme.primary,
        MaterialTheme.colorScheme.tertiary ?: MaterialTheme.colorScheme.secondary
    )

    Text(
        text = text,
        style = style,
        modifier = modifier.drawWithCache {
            val brush = Brush.linearGradient(
                colors = gradientColors,
                start = Offset(0f, 0f),
                end = Offset(size.width, 0f)
            )
            onDrawWithContent {
                drawWithLayer {
                    drawContent()
                    drawRect(brush = brush, blendMode = BlendMode.SrcAtop)
                }
            }
        }
    )
}
```

### Custom Press Indication (Non-Ripple)

```kotlin
class ScaleIndication : Indication {
    @Composable
    override fun rememberUpdatedInstance(interactionSource: InteractionSource): IndicationInstance {
        val isPressed by interactionSource.collectIsPressedAsState()
        val scale by animateFloatAsState(
            targetValue = if (isPressed) 0.94f else 1f,
            animationSpec = spring(stiffness = Spring.StiffnessHigh),
            label = "scale_indication"
        )
        return remember(scale) {
            object : IndicationInstance {
                override fun ContentDrawScope.drawIndication() {
                    scale(scale) { this@drawIndication.drawContent() }
                }
            }
        }
    }
}

val ScaleIndicationToken = ScaleIndication()

// Usage:
Box(
    modifier = Modifier
        .clickable(
            interactionSource = remember { MutableInteractionSource() },
            indication = ScaleIndicationToken,
            onClick = onClick
        )
)
```

### Lottie Animation Integration

```kotlin
@Composable
fun LottieIllustration(
    @RawRes lottieRes: Int,
    modifier: Modifier = Modifier,
    iterations: Int = LottieConstants.IterateForever,
    speed: Float = 1f
) {
    val composition by rememberLottieComposition(LottieCompositionSpec.RawRes(lottieRes))
    val progress by animateLottieCompositionAsState(
        composition = composition,
        iterations = iterations,
        speed = speed
    )

    LottieAnimation(
        composition = composition,
        progress = { progress },
        modifier = modifier
    )
}
```

### Pull-to-Refresh with Custom Indicator

```kotlin
@Composable
fun <T> PullRefreshScreen(
    isRefreshing: Boolean,
    onRefresh: () -> Unit,
    content: @Composable () -> Unit
) {
    val pullState = rememberPullToRefreshState()

    PullToRefreshBox(
        isRefreshing = isRefreshing,
        onRefresh = onRefresh,
        state = pullState,
        indicator = {
            // Custom indicator: spinning branded icon instead of default CircularProgressIndicator
            PullToRefreshDefaults.Indicator(
                state = pullState,
                isRefreshing = isRefreshing,
                color = MaterialTheme.colorScheme.primary,
                containerColor = MaterialTheme.colorScheme.primaryContainer,
                modifier = Modifier.align(Alignment.TopCenter)
            )
        }
    ) {
        content()
    }
}
```
