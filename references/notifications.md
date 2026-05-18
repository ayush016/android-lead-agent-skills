# Notifications — FCM, Channels, Deep Links, Handling

Notifications are a contract with the user: you promise to interrupt them only when it matters. A notification that is irrelevant, poorly timed, or hard to action destroys that trust faster than almost any other UX failure. Design notifications with the same care as screens.

---

## Notification Channels (Android 8+, required)

Every notification must belong to a channel. Users can disable, silence, or customise individual channels. Group them by user intent, not by technical category.

```kotlin
// core/notifications/src/main/kotlin/.../NotificationChannels.kt
object NotificationChannels {
    const val BREAKING_NEWS = "breaking_news"
    const val WEEKLY_DIGEST = "weekly_digest"
    const val DOWNLOADS = "downloads"
    const val REMINDERS = "reminders"
}

class NotificationChannelSetup @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val notificationManager = NotificationManagerCompat.from(context)

    fun createAll() {
        val channels = listOf(
            NotificationChannelCompat.Builder(
                NotificationChannels.BREAKING_NEWS,
                NotificationManagerCompat.IMPORTANCE_HIGH
            )
                .setName("Breaking news")
                .setDescription("Urgent stories as they happen")
                .setLightsEnabled(true)
                .setVibrationEnabled(true)
                .build(),

            NotificationChannelCompat.Builder(
                NotificationChannels.WEEKLY_DIGEST,
                NotificationManagerCompat.IMPORTANCE_LOW
            )
                .setName("Weekly digest")
                .setDescription("Your weekly reading summary")
                .setVibrationEnabled(false)
                .build(),

            NotificationChannelCompat.Builder(
                NotificationChannels.DOWNLOADS,
                NotificationManagerCompat.IMPORTANCE_LOW
            )
                .setName("Downloads")
                .setDescription("Download progress and completion")
                .setShowBadge(false)
                .build(),

            NotificationChannelCompat.Builder(
                NotificationChannels.REMINDERS,
                NotificationManagerCompat.IMPORTANCE_DEFAULT
            )
                .setName("Reminders")
                .setDescription("Article reading reminders you set")
                .build()
        )

        notificationManager.createNotificationChannelsCompat(channels)
    }
}

// Call in Application.onCreate
class AppApplication : Application() {
    @Inject lateinit var channelSetup: NotificationChannelSetup

    override fun onCreate() {
        super.onCreate()
        channelSetup.createAll()
    }
}
```

---

## Notification Permission (Android 13+)

```kotlin
// In Activity or composable — request permission before showing notifications
@Composable
fun RequestNotificationPermission(onResult: (Boolean) -> Unit) {
    val launcher = rememberLauncherForActivityResult(
        contract = ActivityResultContracts.RequestPermission()
    ) { isGranted -> onResult(isGranted) }

    LaunchedEffect(Unit) {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
            launcher.launch(Manifest.permission.POST_NOTIFICATIONS)
        } else {
            onResult(true) // no runtime permission needed below Android 13
        }
    }
}

// Check before posting
fun canShowNotifications(context: Context): Boolean {
    return if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
        ContextCompat.checkSelfPermission(
            context,
            Manifest.permission.POST_NOTIFICATIONS
        ) == PackageManager.PERMISSION_GRANTED
    } else {
        NotificationManagerCompat.from(context).areNotificationsEnabled()
    }
}
```

---

## Building Notifications

### Standard notification

```kotlin
class NotificationHelper @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val notificationManager = NotificationManagerCompat.from(context)

    fun showArticleNotification(article: NotificationArticle) {
        if (!canShowNotifications(context)) return

        val deepLinkPendingIntent = buildArticleDeepLinkPendingIntent(article.id)

        val notification = NotificationCompat.Builder(context, NotificationChannels.BREAKING_NEWS)
            .setSmallIcon(R.drawable.ic_notification)            // monochrome, will be tinted
            .setLargeIcon(loadBitmap(article.imageUrl))          // article thumbnail
            .setContentTitle(article.title)
            .setContentText(article.summary)
            .setStyle(
                NotificationCompat.BigTextStyle()
                    .bigText(article.summary)                    // expanded: full text
                    .setBigContentTitle(article.title)
                    .setSummaryText(article.category)
            )
            .setContentIntent(deepLinkPendingIntent)
            .setAutoCancel(true)                                 // dismiss on tap
            .setPriority(NotificationCompat.PRIORITY_HIGH)
            .setCategory(NotificationCompat.CATEGORY_NEWS)
            .addAction(buildShareAction(article))
            .addAction(buildSaveAction(article.id))
            .setColor(ContextCompat.getColor(context, R.color.notification_accent))
            .setColorized(true)
            .build()

        notificationManager.notify(article.id.toInt(), notification)
    }

    private fun buildShareAction(article: NotificationArticle): NotificationCompat.Action {
        val shareIntent = Intent.createChooser(
            Intent(Intent.ACTION_SEND).apply {
                type = "text/plain"
                putExtra(Intent.EXTRA_TEXT, article.url)
            },
            null
        )
        val pendingIntent = PendingIntent.getActivity(
            context,
            article.id.toInt() + 100,
            shareIntent,
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
        )
        return NotificationCompat.Action(
            R.drawable.ic_share,
            "Share",
            pendingIntent
        )
    }

    private fun buildSaveAction(articleId: Long): NotificationCompat.Action {
        val intent = Intent(context, NotificationActionReceiver::class.java).apply {
            action = NotificationActionReceiver.ACTION_SAVE_ARTICLE
            putExtra(NotificationActionReceiver.EXTRA_ARTICLE_ID, articleId)
        }
        val pendingIntent = PendingIntent.getBroadcast(
            context,
            articleId.toInt() + 200,
            intent,
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
        )
        return NotificationCompat.Action(R.drawable.ic_bookmark, "Save", pendingIntent)
    }

    private fun buildArticleDeepLinkPendingIntent(articleId: Long): PendingIntent {
        val deepLink = Intent(
            Intent.ACTION_VIEW,
            "https://example.com/article/$articleId".toUri(),
            context,
            MainActivity::class.java
        )
        return TaskStackBuilder.create(context).run {
            addNextIntentWithParentStack(deepLink)
            getPendingIntent(
                articleId.toInt(),
                PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
            )!!
        }
    }
}
```

### Progress notification (for downloads)

```kotlin
fun showDownloadProgress(fileId: Int, fileName: String, progress: Int, isComplete: Boolean) {
    val notification = NotificationCompat.Builder(context, NotificationChannels.DOWNLOADS)
        .setSmallIcon(if (isComplete) R.drawable.ic_check else R.drawable.ic_download)
        .setContentTitle(if (isComplete) "Download complete" else "Downloading...")
        .setContentText(fileName)
        .setProgress(100, progress, progress == 0 && !isComplete)
        .setOngoing(!isComplete)
        .setOnlyAlertOnce(true)  // silent updates after first alert
        .build()

    notificationManager.notify(fileId, notification)
}
```

---

## FCM — Firebase Cloud Messaging

### Setup

```kotlin
// google-services.json must be in app/ directory
// build.gradle.kts (project)
id("com.google.gms.google-services") version "4.4.2" apply false

// build.gradle.kts (app)
id("com.google.gms.google-services")
implementation(platform("com.google.firebase:firebase-bom:33.6.0"))
implementation("com.google.firebase:firebase-messaging-ktx")
```

### FCM Service

```kotlin
@AndroidEntryPoint
class AppFirebaseMessagingService : FirebaseMessagingService() {

    @Inject lateinit var notificationHelper: NotificationHelper
    @Inject lateinit var tokenRepository: FcmTokenRepository

    override fun onMessageReceived(remoteMessage: RemoteMessage) {
        super.onMessageReceived(remoteMessage)

        // Both notification and data payload can be present
        val data = remoteMessage.data

        when (data["type"]) {
            "breaking_news" -> {
                val article = NotificationArticle(
                    id = data["article_id"]?.toLong() ?: return,
                    title = data["title"] ?: return,
                    summary = data["summary"] ?: "",
                    imageUrl = data["image_url"],
                    url = data["url"] ?: "",
                    category = data["category"] ?: ""
                )
                notificationHelper.showArticleNotification(article)
            }
            "reminder" -> {
                val reminderId = data["reminder_id"]?.toInt() ?: return
                val message = data["message"] ?: return
                notificationHelper.showReminder(reminderId, message)
            }
        }
    }

    override fun onNewToken(token: String) {
        // Send the new token to your backend
        CoroutineScope(Dispatchers.IO).launch {
            tokenRepository.updateFcmToken(token)
        }
    }
}
```

### Register service in manifest

```xml
<service
    android:name=".service.AppFirebaseMessagingService"
    android:exported="false">
    <intent-filter>
        <action android:name="com.google.firebase.MESSAGING_EVENT" />
    </intent-filter>
</service>
```

### Retrieve and register token

```kotlin
class FcmTokenRepository @Inject constructor(
    private val api: NotificationApi,
    private val secureStorage: SecureStorage
) {
    suspend fun registerToken() {
        Firebase.messaging.token.await().let { token ->
            secureStorage.saveFcmToken(token)
            api.registerDevice(DeviceRegistrationRequest(token = token))
        }
    }

    suspend fun updateFcmToken(newToken: String) {
        secureStorage.saveFcmToken(newToken)
        // Sync to backend if user is logged in
        if (secureStorage.getAccessToken() != null) {
            api.updateDeviceToken(DeviceTokenUpdateRequest(token = newToken))
        }
    }
}
```

---

## Notification Action Receiver

Handle actions (Save, Like, Dismiss) without opening the app:

```kotlin
@AndroidEntryPoint
class NotificationActionReceiver : BroadcastReceiver() {

    @Inject lateinit var articleRepository: ArticleRepository

    override fun onReceive(context: Context, intent: Intent) {
        val pendingResult = goAsync()

        CoroutineScope(Dispatchers.IO).launch {
            try {
                when (intent.action) {
                    ACTION_SAVE_ARTICLE -> {
                        val articleId = intent.getLongExtra(EXTRA_ARTICLE_ID, -1L)
                        if (articleId != -1L) {
                            articleRepository.saveArticle(articleId)
                            // Update the notification to reflect saved state
                            NotificationManagerCompat.from(context)
                                .cancel(articleId.toInt())
                        }
                    }
                }
            } finally {
                pendingResult.finish()
            }
        }
    }

    companion object {
        const val ACTION_SAVE_ARTICLE = "com.example.app.action.SAVE_ARTICLE"
        const val EXTRA_ARTICLE_ID = "article_id"
    }
}
```

---

## Notification Groups (for bundling multiple notifications)

```kotlin
// Group key constant
private const val GROUP_KEY_NEWS = "com.example.app.NEWS"
private const val SUMMARY_ID = 9999

fun showGroupedNotifications(articles: List<NotificationArticle>) {
    // Individual notifications with group key
    articles.forEach { article ->
        val notification = NotificationCompat.Builder(context, NotificationChannels.BREAKING_NEWS)
            .setSmallIcon(R.drawable.ic_notification)
            .setContentTitle(article.title)
            .setContentText(article.summary)
            .setGroup(GROUP_KEY_NEWS)
            .setAutoCancel(true)
            .build()
        notificationManager.notify(article.id.toInt(), notification)
    }

    // Summary notification (inbox style shows all)
    val summary = NotificationCompat.Builder(context, NotificationChannels.BREAKING_NEWS)
        .setSmallIcon(R.drawable.ic_notification)
        .setStyle(
            NotificationCompat.InboxStyle()
                .also { style -> articles.forEach { style.addLine(it.title) } }
                .setSummaryText("${articles.size} new articles")
        )
        .setGroup(GROUP_KEY_NEWS)
        .setGroupSummary(true) // this is the summary
        .setAutoCancel(true)
        .build()

    notificationManager.notify(SUMMARY_ID, summary)
}
```
