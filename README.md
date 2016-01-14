# RxGroups

RxGroups lets you group RxJava `Observable`s together in groups and tie them to your Android Activity
or Fragment lifecycle.

## Usage

```groovy
compile 'com.airbnb:rxgroups:0.2.0'
```

Start with `ObservableManager` in order to create or retrieve groups.

```java
public class MyActivity extends Activity {
  private static final String OBSERVABLE_TAG = "arbitrary_tag";
  private TextView output;
  private ObservableGroup observableGroup;

  // The ObservableManager and your Observable need to be stored somewhere else, outside of your
  // activity lifecycle, so they can survive lifecycle and configuration changes. You could keep
  // them, for example, in your Application object or use a Singleton that is provided by Dagger,
  // for example.
  private ObservableManager observableManager;
  private Observable<Long> observable;

  private final Observer<Long> observer = new Observer<Long>() {
    @Override public void onCompleted() {
      Log.d(TAG, "onCompleted()");
    }

    @Override public void onError(Throwable e) {
      Log.e(TAG, "onError()", e);
    }

    @Override public void onNext(Long l) {
      output.setText(output.getText() + " " + l);
    }
  };

  @Override protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);

    output = (TextView) findViewById(R.id.txt_output);

    // Make sure no events are received until we are ready to display them. Locking the group
    // will cache any results so they can be delivered immediately when you unlock() if there are
    // any available.
    observableGroup.lock();

    observableManager = application.observableManager();
    if (savedInstanceState == null) {
      observableGroup = observableManager.newGroup();
    } else {
      observableGroup = observableManager.getGroup(savedInstanceState.getLong(GROUP_ID));
    }

    if (observableGroup.hasObservable(OBSERVABLE_TAG)) {
      // We've already subscribed to this observable before and it's still emitting items.
      // Maybe the screen was rotated or the activity was paused and then resumed. Let's get it
      // and resubscribe to it.
      observableGroup.<Long>observable(OBSERVABLE_TAG)
          .observeOn(AndroidSchedulers.mainThread())
          .subscribe(observer);
    }

    // Subscribe to the Observable when button is clicked. If you're already subscribed to it,
    // then you might not want to subscribe again. This is up to you to decide.
    myButton.setOnClickListener(v -> observable
        .compose(observableGroup.<Long>transform(OBSERVABLE_TAG))
        .observeOn(AndroidSchedulers.mainThread())
        .subscribe(observer));
  }

  @Override protected void onResume() {
    super.onResume();
    // Go ahead and unlock our group, so we can receive Observable events.
    observableGroup.unlock();
  }

  @Override protected void onDestroy() {
    super.onDestroy();
    if (isFinishing()) {
      // Unsubscribe Observers and clear references. No more events will be received and the
      // destroyed ObservableGroup is now unusable.
      observableManager.destroy(observableGroup);
    } else {
      // Activity is not finishing (maybe just rotating?) so just unsubscribe for now and assume
      // that it will be resubscribed later.
      observableGroup.unsubscribe();
    }
  }
}
```