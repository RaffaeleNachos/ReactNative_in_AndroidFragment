# Proof Of Concept: React Native App integration in Android Native Fragment

*Disclaimer: this is a solution taken from [this stackoverflow answer](https://stackoverflow.com/questions/35221447/react-native-inside-a-fragment), just updated with some minor changes for the current version of React Native and a more detailed "how to".*

**If you want to use my project, just download it, `cd AndroidExample` and `npm install`.**

# Result
<img src="https://imgur.com/78m52T1.gif"/>

## Create React Native Application Example

First thing first, `npm install -g react-native`. You have to [set up the development environment for React Native](https://reactnative.dev/docs/environment-setup) (using React Native CLI quickstart, until "Creating new application paragraph"), then

```bash
npx react-native init RNExample
cd RNExample
```

then you can check if everything works by starting the React Native app with `npx react-native run-android`.

## Android Native Application Example

Create a folder for your integration project (in my case `mkdir AndroidExample`) and create a subfolder `/android`. Open Android studio and create a new project into `./AndroidExample/android` with API target Marshmallow (i will call the project RN_Integration).

In the root directory of your project folder create a new file `package.json`:

```json
{
  "name": "MyReactNativeApp",
  "version": "0.0.1",
  "private": true,
  "scripts": {
    "start": "npx react-native start",
    "android": "npx react-native run-android"
  }
}
```

then `npm install react` and `npm install react-native`.

#### Android Configuration (Gradle & Manifest)

In app's `build.gradle`  add:

```
dependencies {
    ...
    implementation "com.facebook.react:react-native:+"
    implementation "org.webkit:android-jsc:+"
    ...
}
```

In Project's `build.grade`add:

```
allprojects {
    repositories {
        maven {
            // All of React Native (JS, Android binaries) is installed from npm
            url "$rootDir/../node_modules/react-native/android"
        }
        maven {
            // Android JSC is installed from npm
            url("$rootDir/../node_modules/jsc-android/dist")
        }
        ...
    }
    ...
}
```

Enable autolinking of libraries, in `settings.gradle` add:

```
apply from: file("../node_modules/@react-native-community/cli-platform-android/native_modules.gradle"); applyNativeModulesSettingsGradle(settings)
```

while in App's `build.gradle` at the bottom add:

```
apply from: file("../../node_modules/@react-native-community/cli-platform-android/native_modules.gradle"); applyNativeModulesAppBuildGradle(project)
```

last step: your `AndroidManifest.xml` should be like this:

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.rn_integration">

    <uses-permission android:name="android.permission.INTERNET" /> <!-- add this line -->
    <uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW" /> <!-- add this line -->

    <!-- here add usesCleartextTraffic line and android:name="...MyApplication"-->
    <application
        android:usesCleartextTraffic="true"
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/AppTheme"
        android:name="com.example.rn_integration.MyApplication">
        <activity
            android:name=".MainActivity"
            android:label="@string/app_name"
            android:theme="@style/AppTheme.NoActionBar">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />

                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
        <activity android:name="com.facebook.react.devsupport.DevSettingsActivity" /> <!-- add this line-->
    </application>

</manifest>
```

## React Native Code Integration

### Permissions

The user must give runtime permission for `android.permission.SYSTEM_ALERT_WINDOW`. So, add in MainActivity the variable `private final int OVERLAY_PERMISSION_REQ_CODE = 1;` and in `onCreate()` add 

```java
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
            if (!Settings.canDrawOverlays(this)) {
                Intent intent = new Intent(Settings.ACTION_MANAGE_OVERLAY_PERMISSION,
                        Uri.parse("package:" + getPackageName()));
                startActivityForResult(intent, OVERLAY_PERMISSION_REQ_CODE);
            }
}
```

then add the variable `private ReactInstanceManager mReactInstanceManager;` and override the following method:

```java
@Override
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {
        super.onActivityResult(requestCode, resultCode, data);
        if (requestCode == OVERLAY_PERMISSION_REQ_CODE) {
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
                if (!Settings.canDrawOverlays(this)) {
                    // SYSTEM_ALERT_WINDOW permission not granted
                    Toast.makeText(MainActivity.this, "You must give permissions to correctly work", Toast.LENGTH_LONG).show();
                }
            }
        }
        mReactInstanceManager.onActivityResult(this, requestCode, resultCode, data);
    }
```

### Let's do the magic

Copy the files of the previous created React Native project example to the root directory of your integration project (AndroidExample in my case), in particular copy `App.js`, `app.json` and `index.js`. 

*NOTE: This is a basic example, if your React Native App uses other dependencies you have to copy also the `package.json` and `npm install` the other packages.*

In the Android studio project create an abstract class called ReactFragment that extends Fragment:

```java
public abstract class ReactFragment extends Fragment {
    private ReactRootView mReactRootView;
    private ReactInstanceManager mReactInstanceManager;

    // This method returns the name of our top-level react-native component to show
    public abstract String getMainComponentName();

    @Override
    public void onAttach(Context context) {
        super.onAttach(context);
        mReactRootView = new ReactRootView(context);
        mReactInstanceManager =
                ((MyApplication) getActivity().getApplication())
                        .getReactNativeHost()
                        .getReactInstanceManager();

    }

    @Override
    public ReactRootView onCreateView(LayoutInflater inflater, ViewGroup group, Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        return mReactRootView;
    }


    @Override
    public void onActivityCreated(Bundle savedInstanceState) {
        super.onActivityCreated(savedInstanceState);
        mReactRootView.startReactApplication(
                mReactInstanceManager,
                getMainComponentName(),
                null
        );
    }
}
```

The `MyApplication` class will be created soon. Anyway, now we can create Fragments that render React Native components.

```java
public class SecondFragment extends ReactFragment {
    @Override
    public String getMainComponentName() {
        return "RNExample"; // name of our React Native component
    }
}
```

Make sure to add the correct React Native component name, you can find it in `app.json` file (the one we have copied before).

Now we have to add some methods to our MainActivity, and it should be like this:

```java
public class MainActivity extends AppCompatActivity implements DefaultHardwareBackBtnHandler {

    /*
     * Get the ReactInstanceManager, AKA the bridge between JS and Android
     * We use a singleton here so we can reuse the instance throughout our app
     * instead of constantly re-instantiating and re-downloading the bundle
     */
    private ReactInstanceManager mReactInstanceManager;
    private final int OVERLAY_PERMISSION_REQ_CODE = 1;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Toolbar toolbar = (Toolbar) findViewById(R.id.toolbar);
        setSupportActionBar(toolbar);

        /**
         * Get the reference to the ReactInstanceManager offered by MyApplication
         */
        mReactInstanceManager =
                ((MyApplication) getApplication()).getReactNativeHost().getReactInstanceManager();

        //Ask for permission
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
            if (!Settings.canDrawOverlays(this)) {
                Intent intent = new Intent(Settings.ACTION_MANAGE_OVERLAY_PERMISSION,
                        Uri.parse("package:" + getPackageName()));
                startActivityForResult(intent, OVERLAY_PERMISSION_REQ_CODE);
            }
        }
    }

    //Permission Result
    @Override
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {
        super.onActivityResult(requestCode, resultCode, data);
        if (requestCode == OVERLAY_PERMISSION_REQ_CODE) {
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
                if (!Settings.canDrawOverlays(this)) {
                    // SYSTEM_ALERT_WINDOW permission not granted
                    Toast.makeText(MainActivity.this, "You must give permissions to correctly work", Toast.LENGTH_LONG).show();
                }
            }
        }
        mReactInstanceManager.onActivityResult(this, requestCode, resultCode, data);
    }

    /*
     * Any activity that uses the ReactFragment or ReactActivty
     * Needs to call onHostPause() on the ReactInstanceManager
     */
    @Override
    protected void onPause() {
        super.onPause();

        if (mReactInstanceManager != null) {
            mReactInstanceManager.onHostPause();
        }
    }

    /*
     * Same as onPause - need to call onHostResume
     * on our ReactInstanceManager
     */
    @Override
    protected void onResume() {
        super.onResume();

        if (mReactInstanceManager != null) {
            mReactInstanceManager.onHostResume(this, this);
        }
    }

    // Open the React Native Developer Menu when you press the hardware menu button (use Ctrl/Cmd + M in Android Emulator)
    @Override
    public boolean onKeyUp(int keyCode, KeyEvent event) {
        if (keyCode == KeyEvent.KEYCODE_MENU && mReactInstanceManager != null) {
            mReactInstanceManager.showDevOptionsDialog();
            return true;
        }
        return super.onKeyUp(keyCode, event);
    }

    @Override
    public void invokeDefaultOnBackPressed() {
        super.onBackPressed();
    }
}
```

Now create the `MyApplication` class which contains the instance of `ReactInstanceManager` (the bridge between Android Native and React Native).

```java
public class MyApplication extends Application implements ReactApplication {
    private final ReactNativeHost mReactNativeHost = new ReactNativeHost(this) {
        @Override
        public boolean getUseDeveloperSupport() {
            return true;
        }

        @Override
        protected String getJSMainModuleName() {
            return "index";
        }

        @Override
        public List<ReactPackage> getPackages() {
            return Arrays.<ReactPackage>asList(
                    new MainReactPackage()
            );
        }
    };

    @Override
    public ReactNativeHost getReactNativeHost() {
        return mReactNativeHost;
    }
}
```

## Launch the App

Now build the Android Native Application with Gradle and before running you must start the React Native Metro server from the root folder of your integration project `npx react-native start`.

Run in Android Emulator, and as soon as you will go to the RN Fragment the Metro server will bundle the javascript and send it to the App. 

DONE!




