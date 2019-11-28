---
title: Adding a Flutter screen to an Android app
short-title: Add a Flutter screen
description: Learn how to add a single Flutter screen to your existing Android app.
---

This guide describes how to add a single Flutter screen to an existing Android
app. A Flutter screen can be added as a normal, opaque screen, or as a
see-through, translucent screen. Both options are described in this guide.

## Add a normal Flutter screen

{% asset
development/add-to-app/android/add-flutter-screen/add-single-flutter-screen_header.png
class="mw-100" alt="Add Flutter Screen Header" %}

### Step 1: Add FlutterActivity to AndroidManifest.xml

Flutter provides `FlutterActivity` to display a Flutter experience within an
Android app. Like any other `Activity`, `FlutterActivity` must be
registered in your `AndroidManifest.xml`. Add the following XML to your
`AndroidManifestxml` file under your `application` tag:

```xml
<activity
  android:name="io.flutter.embedding.android.FlutterActivity"
  android:theme="@style/LaunchTheme"
  android:configChanges="orientation|keyboardHidden|keyboard|screenSize|locale|layoutDirection|fontScale|screenLayout|density|uiMode"
  android:hardwareAccelerated="true"
  android:windowSoftInputMode="adjustResize"
  />
```

The reference to `@style/LaunchTheme` can be replaced by any Android theme that
you'd like to apply to your `FlutterActivity`. The choice of theme dictates the
colors applied to Android's system chrome, like Android's navigation bar, and 
the background color of the `FlutterActivity` just before the Flutter UI renders 
itself for the first time.

### Step 2: Launch FlutterActivity

With `FlutterActivity` registered in your manifest file, add code to launch
`FlutterActivity` from whatever point in your app that you'd like. The following
example shows `FlutterActivity` being launched from an `OnClickListener`.

Java:
```java
myButton.setOnClickListener(new OnClickListener() {
  @Override
  public void onClick(View v) {
    startActivity(
      FlutterActivity.createDefaultIntent(currentActivity)
    );
  }
});
```

Kotlin:
```kotlin
myButton.setOnClickListener {
  startActivity(
    FlutterActivity.createDefaultIntent(this)
  )
}
```

The above example assumes that your Dart entrypoint is called `main()`, and your
initial Flutter route is '/'. The Dart entrypoint cannot be changed via `Intent`,
but the initial route can be changed via `Intent`. The following example
demonstrates how to launch a `FlutterActivity` that initially renders a custom
route in Flutter.

Java:
```java
myButton.addOnClickListener(new OnClickListener() {
  @Override
  public void onClick(View v) {
    startActivity(
      FlutterActivity
        .withNewEngine()
        .initialRoute("/my_route")
        .build(currentActivity)
      );
  }
});
```

Kotlin:
```kotlin
myButton.setOnClickListener {
  startActivity(
    FlutterActivity
      .withNewEngine()
      .initialRoute("/my_route")
      .build(this)
  )
}
```

Replace `"/my_route"` with your desired initial route.

The use of the `withNewEngine()` factory method configures a `FlutterActivity`
that internally create its own `FlutterEngine` instance. This comes with a 
non-trivial initialization time. The alternative approach is to instruct
`FlutterActivity` to use a pre-warmed, cached `FlutterEngine`, which minimizes
Flutter's initialization time. That approach is discussed next.

### Step 3: (Optional) Use a cached FlutterEngine

Every `FlutterActivity` creates its own `FlutterEngine` by default. Each
`FlutterEngine` has a non-trivial "warm-up" time. This means that launching a
standard `FlutterActivity` comes with a brief delay before your Flutter
experience becomes visible. To minimize this delay, you can warm-up a
`FlutterEngine` before arriving at your `FlutterActivity` and then you can use
your pre-warmed `FlutterEngine` instead.

To pre-warm a `FlutterEngine`, find a reasonable location in your app to
instantiate a `FlutterEngine`. The following example arbitrarily pre-warms a 
`FlutterEngine` in the `Application` class:

Java:
```java
public class MyApplication extends Application {
  @Override
  public void onCreate() {
    super.onCreate();
    // Instantiate a FlutterEngine.
    flutterEngine = new FlutterEngine(this);

    // Start executing Dart code to pre-warm the FlutterEngine.
    flutterEngine.getDartExecutor().executeDartEntrypoint(
      DartEntrypoint.createDefault()
    );

    // Cache the FlutterEngine to be used by FlutterActivity.
    FlutterEngineCache
      .getInstance()
      .put("my_engine_id", flutterEngine);
  }
}
```

Kotlin:
```kotlin
class MyApplication : Application() {
  lateinit var flutterEngine : FlutterEngine

  override fun onCreate() {
    super.onCreate()

    // Instantiate a FlutterEngine.
    flutterEngine = FlutterEngine(this)

    // Start executing Dart code to pre-warm the FlutterEngine.
    flutterEngine.dartExecutor.executeDartEntrypoint(
      DartExecutor.DartEntrypoint.createDefault()
    )

    // Cache the FlutterEngine to be used by FlutterActivity.
    FlutterEngineCache
      .getInstance()
      .put("my_engine_id", flutterEngine)
  }
}
```

The ID passed to the `FlutterEngineCache` can be whatever you'd like. Make sure
that you pass the exact same ID to any `FlutterActiity` or `FlutterFragment`
that should use the cached `FlutterEngine`. Using `FlutterActivity` with a
cached `FlutterEngine` is discussed next.

{{site.alert.note}}
  To warm up a `FlutterEngine`, you must execute a Dart
  entrypoint. Keep in mind that the moment `executeDartEntrypoint()` is invoked,
  your Dart entrypoint method will begin executing. If your Dart entrypoint
  invokes `runApp()` to run a Flutter app, your Flutter app will behave as if it
  is running in a window of zero size until this `FlutterEngine` is attached to a
  `FlutterActivity`, `FlutterFragment`, or `FlutterView`. Make sure that your app
  behaves appropriately between the time you warm it up and the time you display
  Flutter content.
{{site.alert.end}}

With a pre-warmed, cached `FlutterEngine`, you now need to instruct your
`FlutterActivity` to use the cached `FlutterEngine` instead of creating a
new one. To accomplish this, use `FlutterActivity`'s `withCachedEngine()`
builder:

Java:
```java
myButton.addOnClickListener(new OnClickListener() {
  @Override
  public void onClick(View v) {
    startActivity(
      FlutterActivity
        .withCachedEngine("my_engine_id")
        .build(currentActivity)
      );
  }
});
```

Kotlin:
```kotlin
myButton.setOnClickListener {
  startActivity(
    FlutterActivity
      .withCachedEngine("my_engine_id")
      .build(this)
  )
}
```

When using the `withCachedEngine()` factory method, pass the same ID that you
used when caching the desired `FlutterEngine`.

Now, when you launch `FlutterActivity`, there is significantly less delay in
the display of Flutter content.

{{site.alert.note}}
  When using a cached `FlutterEngine`, that `FlutterEngine` outlives any
  `FlutterActivity` or `FlutterFragment` that displays it. You should keep in
  mind that Dart code begins executing as soon as you pre-warm the 
  `FlutterEngine`, and continues executing after the destruction of your
  `FlutterActivity`/`FlutterFragment`. To stop executing and clear resources,
  obtain your `FlutterEngine` from the `FlutterEngineCache` and destroy the
  `FlutterEngine` with `FlutterEngine.destroy()`.
{{site.alert.end}}

{{site.alert.note}}
  Runtime performance is not the only reason that one might pre-warm and cache
  a `FlutterEngine`. A pre-warmed `FlutterEngine` executes Dart code independent
  from a `FlutterActivity`, which allows such a `FlutterEngine` to be used to
  execute arbitrary Dart code at any moment in time. Non-UI application logic
  can be executed in a `FlutterEngine`, like networking and data caching, as 
  well as background behavior within a `Service` or elsewhere. When using a
  `FlutterEngine` to execute behavior in the background, be sure to honor all 
  Android restrictions on background execution.
{{site.alert.end}}

{{site.alert.note}}
  Flutter's debug/release builds have drastically different
  performance characteristics. To evaluate the performance of Flutter, use a
  release build.
{{site.alert.end}}

## Add a translucent Flutter screen

{% asset
development/add-to-app/android/add-flutter-screen/add-single-flutter-screen-transparent_header.png
class="mw-100" alt="Add Flutter Screen With Translucency Header" %}

Most fullscreen Flutter experiences are opaque. However, some apps would like to
deploy a Flutter screen that looks like a modal, for example, a dialog, bottom sheet,
etc. Flutter supports translucent `FlutterActivity`s out of the box.

To make your `FlutterActivity` translucent, make the following adjustments to
the regular process of creating and launching a `FlutterActivity`.

### Step 1: Use a theme with translucency

Android requires a special theme property for `Activity`s that wish to render
with a translucent background. Create or update an Android theme with the
following property:

```xml
<style name="MyTheme" parent="@style/MyParentTheme">
  <item name="android:windowIsTranslucent">true</item>
</style>
```

Then, apply the translucent theme to your desired `FlutterActivity`.

```xml
<activity
  android:name="io.flutter.embedding.android.FlutterActivity"
  android:theme="@style/MyTheme"
  android:configChanges="orientation|keyboardHidden|keyboard|screenSize|locale|layoutDirection|fontScale|screenLayout|density|uiMode"
  android:hardwareAccelerated="true"
  android:windowSoftInputMode="adjustResize"
  />
```

Your `FlutterActivity` now supports translucency. Next, you need to launch your
`FlutterActivity` with explicit transparency support.

### Step 2: Start FlutterActivity with transparency

To launch your `FlutterActivity` with a transparent background, pass the
appropriate `BackgroundMode` to the `IntentBuilder`:

Java:
```java
// Using a new FlutterEngine.
startActivity(
  FlutterActivity
    .withNewEngine()
    .backgroundMode(FlutterActivity.BackgroundMode.transparent)
    .build(context)
);

// Using a cached FlutterEngine.
startActivity(
  FlutterActivity
    .withCachedEngine("my_engine_id")
    .backgroundMode(FlutterActivity.BackgroundMode.transparent)
    .build(context)
);
```

Kotlin:
```kotlin
// Using a new FlutterEngine.
startActivity(
  FlutterActivity
    .withNewEngine()
    .backgroundMode(FlutterActivity.BackgroundMode.transparent)
    .build(this)
);

// Using a cached FlutterEngine.
startActivity(
  FlutterActivity
    .withCachedEngine("my_engine_id")
    .backgroundMode(FlutterActivity.BackgroundMode.transparent)
    .build(this)
);
```

You now have a `FlutterActivity` with a transparent background.

{{site.alert.note}} Make sure that your Flutter content also includes a
  translucent background. If your Flutter UI paints a solid background color
  then it still appears as though your `FlutterActivity` has an opaque
  background. {{site.alert.end}}