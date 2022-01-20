# Using AndroidX [core-splashscreen](https://developer.android.com/reference/kotlin/androidx/core/splashscreen/package-summary) with Lottie integration

| Android 12 | Android 10 |
|------------|-------------|
| ![android12](https://user-images.githubusercontent.com/5563432/135660012-a333d524-2954-4760-b149-640fad8a868a.gif) | ![android10](https://user-images.githubusercontent.com/5563432/135660042-470a7176-c673-4d7e-9b2f-52b7d46db91e.gif)


## The overview
Android 12 splash screen supports animated vector drawables (AVD).
Unfortunately, it's really hard to design good animations with AVD, and
Lottie is a better alternative, but not possible to use directly
with the splash screen. So we need a little trick.

I took the Lottie file from
[lottiefiles.com](https://lottiefiles.com/77440-graph-diagram-animation)
and made a copy of the first frame of the animation as an SVG.

![lottie_capture](https://user-images.githubusercontent.com/5563432/135660233-a23fed6f-30c5-4955-b801-8929f4389fc7.png?s=200)


I then imported and converted the SVG into a vector drawable using the
[resource manager](https://developer.android.com/studio/write/resource-manager#import).

My [launcher icon](src/main/res/drawable/ic_launcher.xml)
is an adaptive icon using this vector drawable as the foreground.

Using the vector drawable, I made an [animated vector drawable](src/main/res/drawable/animated_lottie.xml)
where I rotate the circle of the icon.

Finally I synchronize the end of the splash screen animation with the start
of the lottie animation and manually dismiss the system splash screen to seamlessly
transition from it to my Lottie activity.

## The disclaimer
**Don't do that. It's fun but it's bad, really bad. Why?**

Your users want to use your app as quickly as possible and letting them 
wait on a beautiful but useless animation is against good UX.
On Android 12, the native splash screen runs in the system ui process, 
before your app's process even starts. This make you app launch _feel_ faster
since there is an immediate feedback after the user's click and _while_ your application load.

_Then why did you made this guide?_

Because it was fun, it was a nice way to show off the API and people 
like Lottie. But again, if you don't need it, don't do it. :)

You appliaction start should be as fast as possible and the splash screen 
should be dismissed as soon as you have some content to display.

## The actual story

### Setting things up in [build.gradle](./build.gradle)

Here the important bits are:

```gradle
dependencies {
    implementation 'androidx.core:core-splashscreen:1.0.0-beta01'
    def lottieVersion = "4.1.0"
    implementation "com.airbnb.android:lottie:$lottieVersion"
    ....
}

```

There are some other dependencies like `constraint-layout`, but they are not
mandatory when it comes to splash screens.

### Setting a splash screen theme
In your [`res/values/themes.xml`](src/main/res/values/themes.xml) create a splash screen theme

```xml
  <style name="Theme.Starting" parent="Theme.SplashScreen">
    <item name="windowSplashScreenBackground">#FFFFFF</item>
    <item name="windowSplashScreenAnimatedIcon">@drawable/animated_lottie</item>
    <item name="windowSplashScreenAnimationDuration">500</item>
    <item name="postSplashScreenTheme">@style/Theme.App</item>
  </style>
```

Along with it, your normal app theme that we reference in `postSplashScreenTheme`
```xml
 <style name="Theme.App" parent="Theme.MaterialComponents.DayNight.NoActionBar">
    <item name="android:statusBarColor">@android:color/transparent</item>
    <item name="android:navigationBarColor">@android:color/transparent</item>
 </style>
```

 >  #### Side Note on `windowSplashScreenAnimationDuration`:
 >  The value given here has no impact on the **actual** duration of the animation.
 >  The goal is to use it in the `OnExitAnimationListener` to synchronize the
 >  splash screen icon animation with our custom animation (here our Lottie animation).


Set the starting Theme in your [Manifest](./src/AndroidManifest.xml):
```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.vcaen.splashscreen.lottie">

  <application
      android:icon="@drawable/ic_launcher"
      android:theme="@style/Theme.Starting"
      ...>
    <activity
        ....
    </activity>
  </application>
```

### Setting up your activity
#### Layout

My [layout](src/main/res/layout/activity_lottie.xml) is a `ConstraintLayout`
where I've set the margins and sizes to ensure that my
`LottieAnimationView` is aligned with my icon. In practice, that's your designer job,
but I have better coding skills than design skills so I'm doing it here.

For the width and height, you can use `@dimen/splashscreen_icon_size`, 
but again, my assets are not aligned here so I have to manually set it.

```xml
 <com.airbnb.lottie.LottieAnimationView
      android:id="@+id/animationView"
      android:layout_width="258dp"
      android:layout_height="258dp"
      android:layout_marginEnd="30dp"
      android:layout_marginTop="26dp"
      ...
      app:lottie_autoPlay="false"
      app:lottie_loop="false"
      app:lottie_rawRes="@raw/lottie_anim" />
```

#### The [LottieActivity](src/main/java/com/vcaen/splashscreen/lottie/LottieActivity.kt)

```kotlin
 override fun onCreate(savedInstanceState: Bundle?) {

        // First, you need to get access to the `SplashScreen` instance.
        // You do that by calling `installSplashScreen()` preferably before
        //         `super.onCreate()`. On top of creating the instance, it also
        // set your Application theme to the one set in `postSplashScreenTheme`.
        val splashScreen = installSplashScreen()

        super.onCreate(savedInstanceState)

        // Our content view contains the Lottie animation
        setContentView(R.layout.activity_lottie)

        // We set the OnExitAnimationListener to customize our splash screen animation.
        // This will allow us to take over the splash screen removal animation.
        splashScreen.setOnExitAnimationListener { vp ->
            val lottieView = findViewById<LottieAnimationView>(R.id.animationView)
            lottieView.enableMergePathsForKitKatAndAbove(true)

            // We compute the delay to wait for the end of the splash screen icon
            // animation.
            val splashScreenAnimationEndTime =
                Instant.ofEpochMilli(vp.iconAnimationStartMillis + vp.iconAnimationDurationMillis)
            val delay = Instant.now(Clock.systemUTC()).until(
                splashScreenAnimationEndTime,
                ChronoUnit.MILLIS
            )

            // Once the delay expires, we start the lottie animation
            lottieView.postDelayed({
                vp.view.alpha = 0f
                vp.iconView.alpha = 0f
                lottieView!!.playAnimation()
            }, delay)

            // Finally we dismiss display our app content using a
            // nice circular reveal
            lottieView.addAnimatorListener(object : AnimatorListenerAdapter() {
                override fun onAnimationEnd(animation: Animator?) {
                    val contentView = findViewById<View>(android.R.id.content)
                    val imageView = findViewById<ImageView>(R.id.imageView)

                    val animator = ViewAnimationUtils.createCircularReveal(
                        imageView,
                        contentView.width / 2,
                        contentView.height / 2,
                        0f,
                        max(contentView.width, contentView.height).toFloat()
                    ).setDuration(600)

                    imageView.visibility = View.VISIBLE
                    animator.start()
                }
            })
        }
    }

    override fun onResume() {
        super.onResume()
        // We display our landing activity edge to edge just like the splash screen
        // to have a seamless transition from the system splash screen.
        // This is done in onResume() so we are sure that our Activity is attached
        // to its window.
        WindowCompat.setDecorFitsSystemWindows(window, false)
    }
```

# Questions/Issues/Contributions
You can DM me on twitter @vadimcaen if you have some questions. If you notice any issue, feel free to make a PR.

# Credit

Lottie Animation courtesy of Agung Hermansyah\
https://lottiefiles.com/77440-graph-diagram-animation


