# Navigation — Type-Safe Routes, Deep Links, Back Stack, Multi-Module

Navigation is not just moving between screens. It is the spatial grammar of your app — the contract users form about where they are, where they came from, and how to get back. A navigation architecture that breaks that contract (unexpected back behaviour, lost scroll position, dead deep links) destroys trust that takes weeks to rebuild.

---

## Type-Safe Navigation Setup

### Dependencies

```toml
[versions]
navigation = "2.8.5"
kotlin-serialization = "2.0.21"

[libraries]
navigation-compose = { group = "androidx.navigation", name = "navigation-compose", version.ref = "navigation" }
kotlin-serialization-json = { group = "org.jetbrains.kotlinx", name = "kotlinx-serialization-json", version = "1.7.3" }

[plugins]
kotlin-serialization = { id = "org.jetbrains.kotlin.plugin.serialization", version.ref = "kotlin-serialization" }
```

### Route Definitions — central module

Define all routes in `:core:navigation` so feature modules can navigate to each other without importing each other:

```kotlin
// core/navigation/src/main/kotlin/com/example/core/navigation/Routes.kt

@Serializable data object HomeRoute
@Serializable data object SearchRoute
@Serializable data class ArticleDetailRoute(val articleId: Long)
@Serializable data class ProfileRoute(val userId: Long, val origin: String = "default")
@Serializable data class ImageViewerRoute(val imageUrl: String, val caption: String?)

// Nested graph markers
@Serializable data object AuthGraph
@Serializable data object LoginRoute
@Serializable data object RegisterRoute
@Serializable data object OnboardingRoute
```

**Rules for route data classes:**
- Only primitives and `String` as parameters — no complex objects
- Use `Long` for IDs; never pass entity objects across destinations
- `String?` for optional params; avoid nullable primitives (default values instead)
- All routes are `@Serializable` — required for back stack state saving and process death recovery

---

## NavHost Setup

```kotlin
// app/src/main/kotlin/com/example/app/AppNavigation.kt

@Composable
fun AppNavigation(
    navController: NavHostController = rememberNavController()
) {
    SharedTransitionLayout {
        NavHost(
            navController = navController,
            startDestination = HomeRoute,
            modifier = Modifier.fillMaxSize()
        ) {
            composable<HomeRoute>(
                enterTransition = { fadeIn(tween(300)) + slideInHorizontally { it / 4 } },
                exitTransition = { fadeOut(tween(200)) + slideOutHorizontally { -it / 4 } },
                popEnterTransition = { fadeIn(tween(300)) + slideInHorizontally { -it / 4 } },
                popExitTransition = { fadeOut(tween(200)) + slideOutHorizontally { it / 4 } }
            ) {
                HomeScreen(
                    sharedTransitionScope = this@SharedTransitionLayout,
                    animatedVisibilityScope = this@composable,
                    onNavigateToArticle = { article ->
                        navController.navigate(ArticleDetailRoute(articleId = article.id))
                    },
                    onNavigateToSearch = { navController.navigate(SearchRoute) },
                    onNavigateToProfile = { userId ->
                        navController.navigate(ProfileRoute(userId = userId))
                    }
                )
            }

            composable<ArticleDetailRoute>(
                enterTransition = { fadeIn(tween(300)) },
                exitTransition = { fadeOut(tween(200)) }
            ) {
                val route: ArticleDetailRoute = it.toRoute()
                ArticleDetailScreen(
                    articleId = route.articleId,
                    sharedTransitionScope = this@SharedTransitionLayout,
                    animatedVisibilityScope = this@composable,
                    onBack = { navController.popBackStack() }
                )
            }

            composable<SearchRoute> {
                SearchScreen(onBack = { navController.popBackStack() })
            }

            // Nested auth graph
            navigation<AuthGraph>(startDestination = LoginRoute) {
                composable<LoginRoute> {
                    LoginScreen(
                        onLoginSuccess = {
                            navController.navigate(HomeRoute) {
                                popUpTo<AuthGraph> { inclusive = true }
                            }
                        },
                        onNavigateToRegister = { navController.navigate(RegisterRoute) }
                    )
                }
                composable<RegisterRoute> {
                    RegisterScreen(
                        onRegistrationComplete = {
                            navController.navigate(OnboardingRoute) {
                                popUpTo<AuthGraph> { inclusive = true }
                            }
                        }
                    )
                }
            }
        }
    }
}
```

---

## Navigation Actions — clean ViewModel integration

Never navigate directly from a ViewModel. Emit a navigation event via SharedFlow; the composable collects it and calls the NavController:

```kotlin
// In ViewModel
sealed interface HomeNavigationEvent {
    data class OpenArticle(val articleId: Long) : HomeNavigationEvent
    data object OpenSearch : HomeNavigationEvent
}

private val _navigation = MutableSharedFlow<HomeNavigationEvent>(replay = 0)
val navigation: SharedFlow<HomeNavigationEvent> = _navigation.asSharedFlow()

fun onArticleClicked(articleId: Long) {
    viewModelScope.launch {
        _navigation.emit(HomeNavigationEvent.OpenArticle(articleId))
    }
}

// In composable
LaunchedEffect(Unit) {
    viewModel.navigation
        .flowWithLifecycle(lifecycle, Lifecycle.State.STARTED)
        .collect { event ->
            when (event) {
                is HomeNavigationEvent.OpenArticle ->
                    navController.navigate(ArticleDetailRoute(event.articleId))
                HomeNavigationEvent.OpenSearch ->
                    navController.navigate(SearchRoute)
            }
        }
}
```

---

## Back Stack Management

### Pop to a specific destination

```kotlin
// Clear everything back to Home (inclusive = false means Home stays)
navController.navigate(HomeRoute) {
    popUpTo<HomeRoute> { inclusive = false }
    launchSingleTop = true
}

// Log out — pop entire back stack
navController.navigate(AuthGraph) {
    popUpTo(0) { inclusive = true } // 0 = root of graph
    launchSingleTop = true
}
```

### Prevent duplicate destinations

```kotlin
navController.navigate(SearchRoute) {
    launchSingleTop = true
    restoreState = true
}
```

### Save and restore state for bottom navigation

```kotlin
@Composable
fun BottomNavBar(navController: NavHostController) {
    val currentBackStack by navController.currentBackStackEntryAsState()
    val currentDestination = currentBackStack?.destination

    NavigationBar {
        bottomNavItems.forEach { item ->
            NavigationBarItem(
                selected = currentDestination?.hierarchy?.any { it.hasRoute(item.route::class) } == true,
                onClick = {
                    navController.navigate(item.route) {
                        popUpTo(navController.graph.findStartDestination().id) {
                            saveState = true   // save state of popped screen
                        }
                        launchSingleTop = true
                        restoreState = true    // restore state when re-selecting
                    }
                },
                icon = { Icon(item.icon, contentDescription = item.label) },
                label = { Text(item.label) }
            )
        }
    }
}
```

---

## Deep Links

### Declare in Manifest

```xml
<activity android:name=".MainActivity">
    <intent-filter android:autoVerify="true">
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data
            android:scheme="https"
            android:host="example.com"
            android:pathPattern="/article/.*" />
    </intent-filter>
</activity>
```

### Wire to Navigation Compose

```kotlin
composable<ArticleDetailRoute>(
    deepLinks = listOf(
        navDeepLink {
            uriPattern = "https://example.com/article/{articleId}"
        },
        navDeepLink {
            uriPattern = "myapp://article/{articleId}"
        }
    )
) { backStack ->
    val route: ArticleDetailRoute = backStack.toRoute()
    ArticleDetailScreen(articleId = route.articleId, ...)
}
```

### Trigger deep link programmatically (notification tap)

```kotlin
fun buildArticleDeepLinkPendingIntent(context: Context, articleId: Long): PendingIntent {
    val deepLinkIntent = Intent(
        Intent.ACTION_VIEW,
        "https://example.com/article/$articleId".toUri(),
        context,
        MainActivity::class.java
    )

    return TaskStackBuilder.create(context).run {
        addNextIntentWithParentStack(deepLinkIntent)
        getPendingIntent(
            articleId.toInt(),
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
        )!!
    }
}
```

---

## Arguments and SavedStateHandle

### Extracting route arguments in ViewModel

```kotlin
@HiltViewModel
class ArticleDetailViewModel @Inject constructor(
    savedStateHandle: SavedStateHandle,
    private val getArticleUseCase: GetArticleUseCase
) : ViewModel() {

    // Type-safe extraction — survives process death and configuration changes
    private val route: ArticleDetailRoute = savedStateHandle.toRoute()
    val articleId: Long = route.articleId

    // Observe article data
    val article = getArticleUseCase(articleId)
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5_000),
            initialValue = ArticleUiState.Loading
        )
}
```

### Result passing between destinations (back-stack result)

```kotlin
// In destination B — set a result before popping back
navController.previousBackStackEntry
    ?.savedStateHandle
    ?.set("filter_result", selectedFilter)
navController.popBackStack()

// In destination A — observe the result
val savedStateHandle = navController.currentBackStackEntry?.savedStateHandle
val filterResult by savedStateHandle
    ?.getStateFlow("filter_result", defaultFilter)
    ?.collectAsStateWithLifecycle() ?: State(defaultFilter)
```

---

## Multi-Module Navigation

In a multi-module project, feature modules cannot import each other. The solution: a `:core:navigation` module that all features import, plus a navigation graph contribution pattern.

```kotlin
// core/navigation/src/main/kotlin/.../NavGraphBuilder.kt
// Each feature contributes a NavGraphBuilder extension

// feature/home/src/main/kotlin/.../HomeNavigation.kt
fun NavGraphBuilder.homeGraph(
    navController: NavController,
    sharedTransitionScope: SharedTransitionScope
) {
    composable<HomeRoute> {
        HomeScreen(
            sharedTransitionScope = sharedTransitionScope,
            animatedVisibilityScope = this@composable,
            onNavigateToArticle = { id ->
                navController.navigate(ArticleDetailRoute(id))
            }
        )
    }
}

// feature/article/src/main/kotlin/.../ArticleNavigation.kt
fun NavGraphBuilder.articleGraph(
    navController: NavController,
    sharedTransitionScope: SharedTransitionScope
) {
    composable<ArticleDetailRoute> { backStack ->
        val route: ArticleDetailRoute = backStack.toRoute()
        ArticleDetailScreen(
            articleId = route.articleId,
            sharedTransitionScope = sharedTransitionScope,
            animatedVisibilityScope = this@composable,
            onBack = { navController.popBackStack() }
        )
    }
}

// app/src/main/kotlin/.../AppNavigation.kt — assembles all feature graphs
SharedTransitionLayout {
    NavHost(navController, startDestination = HomeRoute) {
        homeGraph(navController, this@SharedTransitionLayout)
        articleGraph(navController, this@SharedTransitionLayout)
        searchGraph(navController)
        profileGraph(navController)
    }
}
```

---

## Navigation Testing

```kotlin
@RunWith(AndroidJUnit4::class)
class NavigationTest {

    @get:Rule
    val composeTestRule = createComposeRule()

    @Test
    fun clicking_article_navigates_to_detail() {
        val navController = TestNavHostController(
            ApplicationProvider.getApplicationContext()
        )
        navController.navigatorProvider.addNavigator(ComposeNavigator())

        composeTestRule.setContent {
            navController.setGraph(R.navigation.app_graph)
            AppNavigation(navController = navController)
        }

        // Click on an article card
        composeTestRule.onNodeWithTag("article_card_1").performClick()

        // Assert destination changed
        val route = navController.currentBackStackEntry?.toRoute<ArticleDetailRoute>()
        assertThat(route?.articleId).isEqualTo(1L)
    }
}
```

---

## Predictive Back Gesture

Enable gesture-aware back animations (Android 14+):

```xml
<!-- AndroidManifest.xml -->
<application android:enableOnBackInvokedCallback="true">
```

```kotlin
// Custom back handling with animation preview
@Composable
fun DetailScreen(onBack: () -> Unit) {
    BackHandler {
        // Custom logic before back (e.g., show "discard changes?" dialog)
        if (hasUnsavedChanges) {
            showDiscardDialog = true
        } else {
            onBack()
        }
    }
}
```

For shared element transitions, predictive back automatically scrubs the transition in reverse — no additional code required beyond the manifest flag.
