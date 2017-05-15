---
layout: post
title: Kotlin with dagger-android basic setup
categories: [Android, Kotlin, Dagger 2]
comments: true
---
Since `dagger-android` is fairly new, every article on the Internet shows the "old" way of incorporating Dagger 2 into your Kotlin app, which involves writing some amounts of boilerplate code (well, so does any app using `dagger`, but at least with the addition of `dagger-android` you can try to cut it down a bit).

While for seasoned veterans it may be simple, I thought that, especially for less experienced developers, a fully working example could be useful.
This is an example of the most basic setup using `dagger-android` in Kotlin.
<!--more-->
First of all you neet to enable `kapt` -- Kotling Annotation Processing. Add these lines to your `build.gradle` file:

{% highlight groovy %}
kapt {
    generateStubs = true
}
{% endhighlight %}

You'll also need to set up "vanilla" Dagger 2 by adding these dependencies:
{%highlight groovy %}
provided 'com.google.dagger:dagger:2.11-rc2'
annotationProcessor 'com.google.dagger:dagger-compiler:2.11-rc2'
kapt 'com.google.dagger:dagger-compiler:2.11-rc2'
{% endhighlight %}

If you want to use Android `dagger` extensions (which I assume you do, since you're reading this post)  you'll also need the following:
{% highlight groovy %}
compile 'com.google.dagger:dagger-android:2.11-rc2'
annotationProcessor 'com.google.dagger:dagger-android-processor:2.11-rc2'
kapt 'com.google.dagger:dagger-android-processor:2.11-rc2'
{% endhighlight %}

One very important note to take here is that you'll need both the normal Dagger (the first listing) **as well as** the extensions (second listing).

Also, if you don't plan to use Java at all in your Dagger graph you can exclude the `annotationProcessor` lines but I thought I'd show them here just for completness' sake.

Apart from the dependencies there are a couple more steps you have to take.
First of all, your application needs to implement `HasActivityInjector` interface, which consists of adding a simple method and a field which then gets injected.

{% highlight kotlin %}
class App : Application(), HasActivityInjector {
    @Inject lateinit var activityInjector: DispatchingAndroidInjector<Activity>

    override fun activityInjector(): AndroidInjector<Activity> {
        return activityInjector
    }

    override fun onCreate() {
        super.onCreate()
        DaggerAppComponent.create()
                .inject(this)
    }
}
{% endhighlight %}

Graph creation stays the same as in vanilla dagger, except for 2 things -- you have to add a module (or modules) providing Activities to which you are going to inject dependencies as well as the helper `AndroidInjectorModule` to the main app component.

{%highlight kotlin %}
@Component(modules = arrayOf(
		AndroidInjectionModule::class,
		ApiModule::class,
		ActivitiesModule::class
)) interface AppComponent {
	fun inject(app: App)
}
{% endhighlight %}

A quick way of adding an activity module is to use the `@ContributesAndroidInjector` annotation
{% highlight kotlin %}
@Module
abstract class ActivitiesModule {
    @ActivityScope
    @ContributesAndroidInjector
    abstract fun provideMainActivityInjector(): MainActivity
}
{% endhighlight %}

One last step -- when injecting dependencies to your Activity you have to call `AndroidInjection.inject` **before** calling `super.onCreate`.
{% highlight kotlin %}
override fun onCreate(savedInstanceState: Bundle?) {
	AndroidInjection.inject(this)
	super.onCreate(savedInstanceState)
}
{% endhighlight %}

Alright, that's it! I hope it helps at least some of you save a bit of time!

You can try it for yourself and see the full example, which is available on my [GitHub](https://github.com/mewa/kotlin-dagger-android-example).

Once you get the grip it's still useful to refer to the official [Dagger 2 documentation](https://google.github.io/dagger//android.html), as `dagger-android` has some more tricks up its sleeve!