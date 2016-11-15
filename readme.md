# Getting Started

## 1. Include the Framework

### Gradle
For Gradle, add this dependency to your `build.gradle` in the `dependencies {...}` section:

```properties
compile 'de.uulm.vs.android.crdtframework:crdtframework:0.3'
```

While the repository is not available in jcenter, you also have to add the Bintray repository to your module by editing your `build.gradle`:

```properties
repositories {
    maven {
        url 'https://dl.bintray.com/rob-k/crdt-framework/'
    }
}
```

### Maven
For Maven, add this dependency to your `pom.xml` in the `<dependecies>...</dependecies>` section: 
```xml
<dependency>
  <groupId>de.uulm.vs.android.crdtframework</groupId>
  <artifactId>crdtframework</artifactId>
  <version>0.3</version>
  <type>pom</type>
</dependency>
```

While the repository is not available in jcenter, you also have to configure your Maven `settings.xml` file to resolve artifacts through Bintray (taken from [the Bintray docs](https://bintray.com/docs/usermanual/formats/formats_mavenrepositories.html#anchorMavenResolve)):

```xml
 <?xml version='1.0' encoding='UTF-8'?>
 <settings xsi:schemaLocation='http://maven.apache.org/SETTINGS/1.0.0 http://maven.apache.org/xsd/settings-1.0.0.xsd' xmlns='http://maven.apache.org/SETTINGS/1.0.0' xmlns:xsi='http://www.w3.org/2001/XMLSchema-instance'>
 <profiles>
    <profile>
        <repositories>
            <repository>
                <snapshots>
                    <enabled>false</enabled>
                </snapshots>
                <id>bintray-<username>-maven</id>
                <name>bintray</name>
                <url>https://dl.bintray.com/jaycroaker/maven</url>
            </repository>
        </repositories>
        <pluginRepositories>
            <pluginRepository>
                <snapshots>
                    <enabled>false</enabled>
                </snapshots>
                <id>bintray-<username>-maven</id>
                <name>bintray-plugins</name>
                <url>https://dl.bintray.com/jaycroaker/maven</url>
            </pluginRepository>
        </pluginRepositories>
        <id>bintray</id>
    </profile>
 </profiles>
 <activeProfiles>
    <activeProfile>bintray</activeProfile>
 </activeProfiles>
 </settings>
```

## 2. Require Permissions

Add the following lines to your `AndroidManifest.xml` to make your app ask for the required permissions.

``` xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="de.uulm.vs.android.fitshare"
    android:versionCode="1"
    android:versionName="1.0" >

    ...

    <!-- Add these lines if you want to support Wi-Fi P2P (recommended): -->
    <uses-permission android:name="android.permission.ACCESS_WIFI_STATE" />
    <uses-permission android:name="android.permission.CHANGE_WIFI_STATE" />
    <uses-permission android:name="android.permission.CHANGE_NETWORK_STATE" />
    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />

    <!-- Add these lines if you want to support Bluetooth (optional): -->
    <uses-permission android:name="android.permission.BLUETOOTH" android:required="false"/>
    <uses-permission android:name="android.permission.BLUETOOTH_ADMIN" android:required="false"/>

    ...

</manifest>
```


## 3. Create a CRDTController and ConnectionHandler and hook them up

You can do this in the `onCreate()` method of either one of your `Activity`s or, preferably, in your `Application`.

```java
/* The CRDTController instance to handle CRDTs */
public CRDTController crdtController;

/* The connection handler to handle connections with other devices */
public ExchangeConnectionHandler wifip2pConnectionHandler;

/* A randomly generated UUID for this application */
private static final String APP_UUID = "45007ff6-f627-44f7-afa1-fe9010ae8a6e";

@Override
public void onCreate() {

    if (this.crdtController == null) {
        // Create a new CRDT changeListener to handle the CRDTs
        this.crdtController = new CRDTController(this);
        crdtController.enableToasts(true);
    }

    if (this.wifip2pConnectionHandler == null) {
        // Create a new WiFiDirectBroadcastReceiver to handle WiFiP2P connections
        this.wifip2pConnectionHandler = new WiFiDirectBroadcastReceiver(this, APP_UUID, crdtController.getClientId(),
                this.crdtController);
        crdtController.addConnectionHandler(wifip2pConnectionHandler);
    }

    // Initially start device discovery and every 60 seconds after
    crdtController.startScheduledDiscovery(60);

    super.onCreate();
}
```

## 4. Make the App register and unregister the BroadcastReceivers and save the data of the CRDT-Framework to the device storage

This should be done in the `onResume()` and `onPause()` methods of your `Activity`s. It is recommended to outsource this code in a a method that will be called in all activities if you have more than one activity.

```java
/**
 * Registers the broadcast receivers with the intent values to be matched.
 * Can be called by activities as default implementation for
 * {@link android.app.Activity#onResume()}.
 */
public void onResume() {
    crdtController.registerReceivers();
}

/**
 * Unregisters the broadcast receivers and makes the CRDTController save its data to the device storage.
 * Can be called by activities as default implementation for {@link android.app.Activity#onPause()}.
 */
public void onPause() {
    crdtController.unregisterReceivers();

    // Save the data to the device storage
    crdtController.saveToStorage();
}
```

## 5. Get a CRDT-Instance

To get instances of new CRDTs call the `newXXXCRDT(id, ...)` methods of your `CRDT-Controller` instance. The `id` is a `String` that functions as identifier for the instance.

To get back instances that have been created previously (for example in another activity or previous to an application shutdown), call `getCRDT(id)`.

### Example:
```java
...
private final String COUNTER_CRDT = "counterCRDT";
private CounterCRDT counterCRDT;

@Override
protected void onCreate(Bundle savedInstanceState) {

    ...

    if ((counterCRDT = (CounterCRDT) crdtController
            .getCRDT(COUNTER_CRDT)) == null) {
        counterCRDT = crdtController.newCounterCRDT(COUNTER_CRDT, 0);
    }

    ...

}
```


## 6. Interact with the Controller

### Start discovery

Discovery of other devices (for all registered connection handlers) can be be initiated manually by calling `startDiscovery()`, or automatically every X seconds by using `startScheduledDiscovery(X)` after the controller has been initiated.

### Commit changes

Changes from other devices will be received on contact. However, if a connection to one or more devices is already existent, your changes can be commited to be sent over this connection immediately with `commitChanges()`. This will attempt to push on all registered connection handlers.

### Receive notifications

You can add one or more listeners in order to receive notifcations about changes to the framework. This listeners must implement the `CRDTNotificationReceiver` interface and can be added with `crdtController.addListener(listener);`. Notifications include successful merges, replication start and end, as well as errors. Additionally, information about available peers for both Bluetooth and Wi-Fi P2P and the respective states of these frameworks can be received.

### Transactions

You can group a set of changes to your CRDTs or simply delay the merge of received and local changes by using a transaction.

#### Start a transaction

Call the `beginTransaction()` method of your `CRDT-Controller` instance. After this call the framework will use a deep copy of your CRDTs on contact with other devices and received changes will be queued till the transaction is finished. Calling `commitChanges()` while a transaction is active will result in an error.

#### Rollback the transaction

Call `rollbackTransaction()` if you want to revert all changes that have been made within the transaction. All queued changes will be merged now.

#### Commit the transaction
Call `commitTransaction()` to finish a transaction. This will automatically call `commitChanges()` and all queued changes will be merged.


