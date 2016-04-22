---
layout: post
section-type: post
title: "Dagger 2 and Testing: Adios setComponent()"
category: Testing
tags: [ 'Dagger 2', 'Testing' ]
---

There seems to be a common pattern when setting up Dagger 2 with testing frameworks like Espresso and Robolectric. Basically, it requires modifying production code (typically the `Application` class by adding a `setComponent(Component component)`) to allow tests to override the `Application` `Component` with a `Component` that contains `Modules` with mock versions of the dependencies provided by the production `Modules`. 

I never liked the `setComponent()` approach. Regardless of how minimal the change is, we should not have to modify production code to accomodate our tests. Hence, I would like to propose a different approach.

For this [example](https://github.com/emmano/DaggerTestExample) I will be using Robolectic 3 as the testing framework (this pattern can also be used with Espresso). This sample will have a single `Application` wide `Component`.

Let's start by creating a new project with a simple `MainActivity`. `MainActivity` will only have a `TextView` that displays text that is provided by Dagger. 

{% highlight java %}
package me.emmano.daggertestexample;

import android.os.Bundle;
import android.support.v7.app.AppCompatActivity;
import android.widget.TextView;
import javax.inject.Inject;

public class MainActivity extends AppCompatActivity {

    @Inject String text;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        ((DaggerApplication)getApplication()).getApplicationComponent().inject(this);

    }

    @Override
    protected void onResume() {
        super.onResume();
        TextView textView = (TextView) findViewById(R.id.hello_text_view);
        textView.setText(text);
    }
}
{% endhighlight %}

We will have an `ApplicationComponent` that will inject `MainActivity`. `ApplicationComponent` only needs one module to be generated, `ApplicationModule`. `ApplicationModule` provides a `String` used by the `TextView` in `MainActivity`. In `DaggerApplication` we get an instance of our `Component` and store it in a memeber variable. We then generate a getter for the `Component` so we can get it from `MainActivity`. In `MainActivity` we get the `Component` from `DaggerApplication` and inject ourselves. This is pretty much the bare bone minimum setup to get Dagger 2 working.


{% highlight java %}
package me.emmano.daggertestexample;


import javax.inject.Singleton;

import dagger.Component;

@Singleton
@Component(modules = ApplicationModule.class)
public interface ApplicationComponent {
    void inject(MainActivity mainActivity);
}
{% endhighlight %}

{% highlight java %}
package me.emmano.daggertestexample;

import javax.inject.Singleton;

import dagger.Module;
import dagger.Provides;

@Module
public class ApplicationModule {

    @Singleton
    @Provides
    String provideTestString() {
        return "Hello, World!";
    }
}

{% endhighlight %}


{% highlight java %}
package me.emmano.daggertestexample;

import android.app.Application;

public class DaggerApplication extends Application {

  private ApplicationComponent applicationComponent;

  @Override public void onCreate() {
    super.onCreate();
    applicationComponent =
        DaggerApplicationComponent.builder().applicationModule(new ApplicationModule()).build();
  }

  public ApplicationComponent getApplicationComponent() {
    return applicationComponent;
  }
}
{% endhighlight %}

Now here comes the interesing part. How do we test this?

Let's start by creating a new Robolectric test for `MainActivity`, `MainActivityTest`. `MainActivityTest` needs to somewhow override the `ApplicationComponent`. We do this by proving a `Module` that returns a `String` that is "under our control" so we can verify the `TextView` is using the `String` provided by Dagger. We will then create a new `Component` for the test and use the new `Module` and its dependencies to inject our test object. This setup will be really similar to the current production setup.


{% highlight java %}
package me.emmano.daggertestexample;

import android.widget.TextView;
import org.junit.Before;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.robolectric.Robolectric;
import org.robolectric.RobolectricGradleTestRunner;
import org.robolectric.RuntimeEnvironment;
import org.robolectric.annotation.Config;
import org.robolectric.util.ActivityController;

import static org.junit.Assert.assertEquals;


@RunWith(RobolectricGradleTestRunner.class)
@Config(application = DaggerTestApplication.class, sdk = 21, constants = BuildConfig.class)
public class MainActivityTest {

  private ActivityController<MainActivity> mainActivityActivityController;

  @Before public void setUp() throws Exception {

    mainActivityActivityController = Robolectric.buildActivity(MainActivity.class).create();
    TestApplicationComponent testApplicationComponent =
        ((DaggerTestApplication) RuntimeEnvironment.application).getTestApplicationComponent();
    testApplicationComponent.inject(mainActivityActivityController.get());
  }

  @Test public void textViewShouldDisplayCorrectTextBasedOnModule() throws Exception {
    mainActivityActivityController.resume();

    MainActivity testObject = mainActivityActivityController.get();

    assertEquals("test",
        ((TextView) testObject.findViewById(R.id.hello_text_view)).getText().toString());
  }
{% endhighlight %}

We are going to create `TestApplicationComponent` and `TestApplicationModule`. `TestApplicationComponent` will inject `MainActivity` and `TestApplicationModule` will provide a `String` that is under our control, "test" in this case.


{% highlight java %}
package me.emmano.daggertestexample;

import android.widget.TextView;
import org.junit.Before;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.robolectric.Robolectric;
import org.robolectric.RobolectricGradleTestRunner;
import org.robolectric.RuntimeEnvironment;
import org.robolectric.annotation.Config;
import org.robolectric.util.ActivityController;

import static org.junit.Assert.assertEquals;

@RunWith(RobolectricGradleTestRunner.class)
@Config(application = DaggerTestApplication.class, sdk = 21, constants = BuildConfig.class)
public class MainActivityTest {

  private ActivityController<MainActivity> mainActivityActivityController;

  @Before public void setUp() throws Exception {

    mainActivityActivityController = Robolectric.buildActivity(MainActivity.class).create();
    TestApplicationComponent testApplicationComponent =
        ((DaggerTestApplication) RuntimeEnvironment.application).getTestApplicationComponent();
    testApplicationComponent.inject(mainActivityActivityController.get());
  }

  @Test public void textViewShouldDisplayCorrectTextBasedOnModule() throws Exception {
    mainActivityActivityController.resume();

    MainActivity testObject = mainActivityActivityController.get();

    assertEquals("test",
        ((TextView) testObject.findViewById(R.id.hello_text_view)).getText().toString());
  }
}
{% endhighlight %}

{% highlight java %}
package me.emmano.daggertestexample;

import dagger.Module;
import dagger.Provides;
import javax.inject.Singleton;

@Module public class TestApplicationModule {

  @Singleton @Provides String provideTestString() {
    return "test";
  }
}
{% endhighlight %}

Just like in the production code, we use the `Application` class to store a reference of our `Component`, `TestApplicationComponent` in this case. Let's create `DaggerTestApplication`, store a reference to the test `Component` as a member, and generate its getter.

{% highlight java %}
package me.emmano.daggertestexample;

public class DaggerTestApplication extends DaggerApplication {

  private TestApplicationComponent testApplicationComponent;

  @Override public void onCreate() {
    super.onCreate();
    testApplicationComponent = DaggerTestApplicationComponent.builder()
        .testApplicationModule(new TestApplicationModule())
        .build();
  }

  public TestApplicationComponent getTestApplicationComponent() {
    return testApplicationComponent;
  }
}
{% endhighlight %}

 There is a little gotcha here. We cannot simply create a class that extends `Application`. This will generate a `ClassCastException` inside `MainActivity.onCreate()` because we are casting our `Application Context` to a `DaggerApplication`. In order to solve this problem, we make `DaggerTestApplication` extend `DaggerApplication`. The next step is to inject our test object, `MainActivity`, with our newly created `DaggerTestApplicationComponent`.

 We build the project and realize that we do not have an instance of `DaggerTestApplicationComponent`. In fact, if we try to look at the generated code we realize that there is no generated code for our test package. This is because the `apt` plugin does not tell the dagger annotation processor look for `Components` in our `test` directory. We need to tell it to do so manually by adding the following to `build.gradle` inside the `dependencies{}`:


{% highlight groovy %}
 testApt 'com.google.dagger:dagger-compiler:2.2'
{% endhighlight %}

The last thing we need to do is to tell Robolectric to use our custom test `Application`. This is done using `@Config(application=DaggerTestApplication.class)`. At this point we are ready to use `DaggerTestApplicationComponent` to inject our test object.

Let's run our simple test... and...

SUCCESS!

There are some evident "flaws" with this pattern:

1.  When our test runs, our test obeject gets injected twice. Once by the production code per-se. The second time, our test object is inject by `DaggerTestComponent` inside our test. This means that if we need to test an `Activity` that calls its `onCreate()` twice on the same test, we would have to re-inject the test object.

2.  This setup has a bit more boiler plate code.


Please let me know what you thing of this configuration in the comments below.

Big shoutout to [@mathematicalfunk](https://github.com/Mathematicalfunk) for reviewing this.

P.S.  Please note that this simple example does not require us to inject mock objects in our test that are used by our test object. Let's imagine `MainActivity` depended on a `UserService`. We can easily inject a mock version of `UserService` in `MainActivityTest` by telling `TestApplicationComponent` that we will be injecting the test as well.

`void inject(MainActivityTest);`

and providing the dependency via a `Module` (i.e. our `TestApplicationModule`). Now we can `@Inject` our test **AND** `MainActivity` with a mock.