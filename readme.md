# Getting Started

## 1. Include the Framework

### Gradle
For Gradle, add this dependency to your `build.gradle` in the `dependencies {...}` section:

```properties
compile 'de.uulm.vs.android.crdtframework:crdtframework:0.4'
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
    package="de.uulm.vs.android.crdtsampleapp"
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

For both the Wi-Fi P2P and the Bluetooth communication modules you will need to create a unique Version 4 UUID in order for the framework to only replicate with other devices running your app (you can simply generate a UUID with tools like [https://www.uuidgenerator.net/](https://www.uuidgenerator.net/)).

Example using the Wi-Fi P2P communication module:
```java
/* The CRDTController instance to handle CRDTs */
public CRDTController crdtController;

/* The connection handler to handle connections with other devices */
public ExchangeConnectionHandler wifip2pConnectionHandler;

/* A randomly generated UUID for this application.
 * REPLACE THIS with your own UUID! */
private static final String APP_UUID = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx";

@Override
public void onCreate() {

    if (this.crdtController == null) {
        // Create a new CRDT changeListener to handle the CRDTs
        this.crdtController = new CRDTController(this);
    }

    if (this.wifip2pConnectionHandler == null) {
        // Create a new WiFiDirectBroadcastReceiver to handle WiFiP2P connections
        this.wifip2pConnectionHandler = new WiFiDirectBroadcastReceiver(this, 
                APP_UUID, crdtController.getClientId(),
                this.crdtController);
        crdtController.addConnectionHandler(wifip2pConnectionHandler);
    }

    // Initially start device discovery and every 60 seconds after
    crdtController.startScheduledDiscovery(60);

    super.onCreate();
}
```

## 4. Make the App register and unregister the BroadcastReceivers and save the data of the CRDT-Framework to the device storage

This should be done in the `onResume()` and `onPause()` methods of your `Activity`s. It is recommended to outsource this code to a method that will be called in all activities (e.g. once again, in your `Application`), if you have more than one activity .

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
 * Unregisters the broadcast receivers and makes the CRDTController 
 * save its data to the device storage.
 * Can be called by activities as default implementation for 
 * {@link android.app.Activity#onPause()}.
 */
public void onPause() {
    crdtController.unregisterReceivers();

    // Save the data to the device storage
    crdtController.saveToStorage();
}
```

## 5. Get a CRDT-Instance

To get instances of new CRDTs call the `newXXXCRDT(id, ...)` methods of your `CRDT-Controller` instance. The `id` is a `String` that is used as identifier for the instance. Additional parameters are mostly default values.

Call `getCRDT(id)` to get instances back that have been created previously.

Possible use cases:

- access instances created in another activity
- access instances created in an ealier session (that have been persisted by the framework)
- access instances that were created by another client (and replicated by the framework)

### Example
You will often want to check if the CRDT with an specific identifier has already been created before creating a new instance, as shown in the example below:
```java
...
private final String COUNTER_CRDT = "myCounterCRDT";
private CounterCRDT myCounterCRDT;

@Override
protected void onCreate(Bundle savedInstanceState) {

    ...

    if ((myCounterCRDT = (CounterCRDT) crdtController
            .getCRDT(COUNTER_CRDT)) == null) {
        myCounterCRDT = crdtController.newCounterCRDT(COUNTER_CRDT, 0);
    }

    ...

}
```


## 6. Interact with the Controller

### Start discovery

Discovery of other devices (for all registered connection handlers) can be be initiated manually by calling `startDiscovery()`, or automatically every X seconds by using `startScheduledDiscovery(X)`. You usually want to do this right after the controller has been initiated.

### Commit changes

Changes from other devices will be received on contact. However, if a connection to one or more devices is already existent, your changes can be commited to be sent over this connection immediately with `commitChanges()`. This will result in an attempt to push on all registered connection handlers. In most cases you will want to call this method after every significant change to your CRDTs.

### Receive notifications

You can register one or more listeners in order to receive notifcations about changes to the framework. This listeners must implement the `CRDTNotificationReceiver` interface and can be added with `addListener(listener);`. 

Notifications include successful merges, replication start and end, as well as errors. The most important interface method is `mergeCompleted()`, which signals that an update from another devices has been received and was successfully merged, which enables you to update the UI accordingly.

Additionally, information about available peers for both Bluetooth and Wi-Fi P2P and the respective states of these frameworks can be received.

### Transactions

You can group a set of changes to your CRDTs or simply delay the merge of received and local changes by using a transaction. This can also be used if two or more CRDTs semantically belong together and should never be replicated seperately.

#### Start a transaction

Call the `beginTransaction()` method of your `CRDT-Controller` instance. After this call the framework will use a deep copy of your CRDTs on contact with other devices and received changes will be queued till the transaction is finished. Calling `commitChanges()` while a transaction is active will result in an error.

#### Rollback the transaction

Call `rollbackTransaction()` if you want to revert all changes that have been made within the current transaction. All queued changes will be merged now.

#### Commit the transaction
Call `commitTransaction()` to finish the transaction. This will automatically call `commitChanges()` and all queued changes will be merged.

## 7. Common Data Types

All data types in this framework inherit from the `CRDT` interface. This interface includes a method `getValue()` that returns the represented value of the CRDT (e.g. an Integer, a Set etc.).

### Counter 

A `CounterCRDT` is basically a number that can be incremented by 1 using `increment()` or by n using `increment(n)`, decrementation respectively.

### VectorClock

A `VectorClockCRDT` is a modified counter that is interpreted as integer vector and can be compared with other instances regarding their causal order (`EQUAL`, `AFTER`, `BEFORE` or `CONCURRENT`). The value of a specific client can be accessed with `getTimestamp(clientId)`.

### Voter

A `VoterCRDT` is another modified counter that can be used to represent up- and downvotes on a matter. Every client can upvote (`upvote()`) or downvote (`downvote()`), but they can only vote once (i.e. they can modify the counter by `+-1`). Subsequent upvote/downvote calls will result in a 'revoke' of the vote (e.g. calling `upvote()` twice by the same client will result in the counter changing by `+-0`). To get the voted value of the current client, you can call `getVotedValue()`.

### Maximum

The `MaximumCRDT` is a simple maximum implementation that holds an integer number and supports a `setMaximum(int)` method, which will max out the current and the new int.

### Set

The `LWWSetCRDT<T>` is a set that supports `add(object)` and `remove(object)` operations. It implements the LWW (last write wins) strategy by utilizing vector clocks.

### CRDTMap

The `CRDTMapCRDT<K,V>` is a map implementation. K can by of any Object type while V must be a CRDT. This is useful if you want to store an unknown or dynamic amount of CRDTs, since they all will be merged (assuming they have the same id).

### What should I do if I need a data type that is not covered by an existing implementation?

You can combine existing implementations by using one or more `CRDTMapCRDT`s while using the same keys.

You can also write your own CRDT implementation by implementing the `SimpleCRDT` interface and adding your instance(s) with `handleSimpleCRDTs(simpleCrdts)`. Make sure to provide a proper `merge(other)` implementation. 


