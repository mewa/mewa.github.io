---
categories:
- Android
- Kotlin
- Dagger 2
aliases: ["/articles/2017-05/kotlin-with-dagger-android-example"]
comments: true
date: "2017-05-16T00:00:00Z"
title: Kotlin with dagger-android basic setup
---

Since `dagger-android` is fairly new, every article on the Internet shows the "old" way of incorporating Dagger 2 into your Kotlin app, which involves writing some amounts of boilerplate code (well, so does any app using `dagger`, but at least with the addition of `dagger-android` you can try to cut it down a bit). 

While for seasoned veterans it may be simple, I thought that, especially for less experienced developers, a fully working example could be useful.
This is an example of the most basic setup using `dagger-android` in Kotlin.

**Update (2018-02-24)**: When this guide was first written it was using `2.11-rc2` version of `dagger`. Dagger extensions for Android have evolved since and now it's even easier to set things up. Below you can find the updated guide using version `2.12`.
<!--more-->
##### Steps
First of all you neet to enable `kapt` -- Kotling Annotation Processing. Add these lines to your `build.gradle` file:

```groovy
apply plugin: 'kotlin-kapt'
```

You'll also need to set up "vanilla" Dagger 2 by adding these dependencies:
```groovy
provided "com.google.dagger:dagger:$daggerVersion"
annotationProcessor "com.google.dagger:dagger-compiler:$daggerVersion"
kapt "com.google.dagger:dagger-compiler:$daggerVersion"
```

If you want to use Android `dagger` extensions (which I assume you do, since you're reading this post)  you'll also need the following:
```groovy
implementation "com.google.dagger:dagger-android:$daggerVersion"
implementation "com.google.dagger:dagger-android-support:$daggerVersion"
annotationProcessor "com.google.dagger:dagger-android-processor:$daggerVersion"
kapt "com.google.dagger:dagger-android-processor:$daggerVersion"
```

One very important note to take here is that you'll need both the normal Dagger (the first listing) **as well as** the extensions (second listing).

Also, if you don't plan to use Java at all in your Dagger graph you can exclude the `annotationProcessor` lines but I thought I'd show them here just for completness' sake.

Apart from the dependencies there are a couple more steps you have to take.
First of all, your application needs to implement `HasActivityInjector` interface, which consists of adding a simple method and a field which then gets injected. The easiest step to do is to simply extend the `DaggerApplication` class, which also implements interfaces for injecting to other components such as `Fragment`s or `Service`s (you can see [the source](https://github.com/google/dagger/blob/master/java/dagger/android/DaggerApplication.java#L36) for more).

```kotlin
class App : DaggerApplication() {

    override fun applicationInjector(): AndroidInjector<out DaggerApplication> {
        return DaggerAppComponent.builder().create(this)
    }

}
```

Graph creation stays the same as in vanilla dagger, except for a couple things -- you have to add a module (or modules) providing Activities to which you are going to inject dependencies as well as the helper `AndroidSupportInjectionModule` to the main app component. Your component will also need to extend `AndroidInjector` and include the respective `Component.Builder`.

```kotlin
@Singleton
@Component(modules = arrayOf(
        AndroidSupportInjectionModule::class,
        ApiModule::class,
        ActivitiesModule::class
))
interface AppComponent : AndroidInjector<App> {

    @Component.Builder
    abstract class Builder : AndroidInjector.Builder<App>()

}
```

A quick way of adding an activity module is to use the `@ContributesAndroidInjector` annotation. Please note that the scope annotations are *optional*.
```kotlin
@Module
abstract class ActivitiesModule {
    @ActivityScope
    @ContributesAndroidInjector
    abstract fun provideMainActivityInjector(): MainActivity
}
```

One last step -- when injecting dependencies to your Activity it has to extend `DaggerActivity` class, which then takes care of injecting dependencies at the right time as well as managing their lifecycle.
```kotlin
class MainActivity : DaggerActivity() { /* definitions */ }
```

Alright, that's it! I hope it helps at least some of you save a bit of time!

You can try it for yourself and see the full example, which is available on my [GitHub](https://github.com/mewa/kotlin-dagger-android-example).

Last but not least, I would like to thank [Daniel Passos](https://github.com/danielpassos) for putting the effort to update the example repository code to the latest version of `dagger`. If you would still like to see the previous version, it's available under [this branch](https://github.com/mewa/kotlin-dagger-android-example/tree/dagger-2.11-rc2).

Once you get the grip it's still useful to refer to the official [Dagger 2 documentation](https://google.github.io/dagger//android.html), as `dagger-android` has some more tricks up its sleeve!
