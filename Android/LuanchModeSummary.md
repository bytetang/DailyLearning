The features of UI knowledge will be forgotten soon if a long time without reviewing.

## Goals of this blog ##

1. Misunderstood lifecycle of activity
2. Intent flag and general scenarios


## Paris ##

these pairs represent the opposite meaning.  

`OnCreate` <-> `OnDestroy`

`OnStart` <-> `OnStop`  

`OnResume` <-> `OnPause` 

## Orders ##

### order of `onResume` and `OnStart` ###

`OnResume` only called when the activity visible. And the OnResume is after the lifecycle of OnStart. 

> It's worth to mention that if the Activity is setten to transparent, the OnResume() may be called when the new Activity starts.


### Orders of start a new Activity ###

Scenario of start Activity B from Activity A,
the lifecycle order is,

```java
OnPause(A) -> OnCreate(B) -> OnStart(B) -> OnResume(B) -> OnStop(A)
```

<b>Note that OnStop(A) is executed after the B render finish rather than after the A OnPause.</b>

About the pause state, The Activity will pause firstly after B starts to onCreate. If Activity has been paused, all events can't be execute then. 

when Activity A has been started again, the Activity will be bring to the front, remember the pairs above, the `onStart()` will be executed since the OnStop occurred, after this is `OnResume()`.

if you called the finish() when start B, A will trigger the `OnDestroy` after `OnStop`, the order is,

```java
OnPause(A) -> OnCreate(B) -> OnStart(B) -> OnResume(B) -> OnStop(A) -> <b>OnDestroy(A)</b>
```

likewise,if A been started again, reference the pair of OnCreate() <-> OnDestroy(), OnCreate() will be called. Otherwise, only OnNewIntent() can be triggered.


## Intent Flag ##

### FLAG_ACTIVITY_NEW_TASK ###

<b>Intent Scenario</b>

caution it's not represent will start a new stack for new activity. it' is generally used by activities that want to present a "launcher" style behavior. 

it's usually use with `FLAG_ACTIVITY_CLEAR_TASK` or `FLAG_ACTIVITY_CLEAR_TOP`

for instance, assuming you have finished a forgotten password flow, the InputEmailActivity, VerifyCodeActivity, SetPasswordActivity has been stared and all in the stack, you want to back to the home page now.
you should start home like this.

```
val intent = Intent(context, HomeActivity::class.java)
intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TOP)
context.startActivity(intent)
```

All activities above the home will be clear.


<b> Application Intent </b>

If you start the Activity with applicationContext, FLAG_ACTIVITY_NEW_TASK must be specific 

<b> How to start a new stack </b>

When using this flag, if a task is already running for the activity you are now starting, then a new activity will not be started; instead, the current task will simply be brought to the front of the screen with the state it was last in, so this same as the launcher mode of `SingleInstance`

If you want to start a new stack for new activities, please use 

`intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK or Inten.FLAG_ACTIVITY_MULTIPLE_TASK)`

## Activity Stack Manage ##

### Visit Activities lifecycle globally ###

If you want to visit the lifecycle of all activities, use the `registerActivityLifecycleCallbacks` API is a good idea without invading.

```java
 application.registerActivityLifecycleCallbacks(object: Application.ActivityLifecycleCallbacks {
            override fun onActivityPaused(activity: Activity) {
                
            }

            override fun onActivityStarted(activity: Activity) {
               
            }

            override fun onActivityDestroyed(activity: Activity) {

            }

            override fun onActivitySaveInstanceState(activity: Activity, outState: Bundle) {
            }

            override fun onActivityStopped(activity: Activity) {
                
            }

            override fun onActivityCreated(activity: Activity, savedInstanceState: Bundle?) {
                Log.d("Lifecycle", "${activity::class.simpleName} onActivityCreated")
                
            }

            override fun onActivityResumed(activity: Activity) {
               
            }

        })
```

### Finish Activity Stack ###

`finishAffinity()`

If only a single Activity stack is running in application,  call `finishAffinity()` will close all activity of stack, the application will exit when there are none visible pages. If the multiple task ara running, only the current stack in which activity stays on it will be close. the first of Activity of the back stack will be on the front



There is another way that can be used to close all pages via `registerActivityLifecycleCallbacks` visiter.

1. `val activities = HashMap<String, Activity>()`

2. `onActivityCreated`, cache the activity instance to map

```java
override fun onActivityCreated(activity: Activity, savedInstanceState: Bundle?) {
              
                activities[activity.toString()] = activity
            }
```

3. `onActivityDestroyed`, remove the activity 

```java
override fun onActivityDestroyed(activity: Activity) {

                if (activities.containsKey(activity.toString())) {
                    activities.remove(activity.toString())
                }
            }
```

4. foreach the map, finish all activity via instance.
