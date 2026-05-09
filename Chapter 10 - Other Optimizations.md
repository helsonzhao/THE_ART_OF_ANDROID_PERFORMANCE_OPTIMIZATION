In the previous chapters, we have learned about the most commonly performed performance optimizations in Android development, and these optimization directions can greatly enhance the program experience. However, in the area of user experience optimization, there is never enough. Therefore, in the final chapter, the author will also explain the remaining optimization directions, including power consumption optimization, traffic optimization, disk usage optimization, and degradation optimization, to help readers form a complete performance optimization system.&#x20;

Today's mobile devices are becoming increasingly powerful, and we often don't need to deliberately optimize power consumption, network usage, and disk occupancy. However, this doesn't mean that we don't need to optimize in these areas. Once problems such as severe power consumption and high data usage occur, they will still affect the user experience of the application and may even lead to user complaints. Optimization in these areas usually focuses on preventing serious anomalies from occurring and requires timely repair and loss prevention when serious anomalies do occur. Therefore, we first need to establish good monitoring, and based on comprehensive monitoring, we can then proceed with further governance and optimization.

Finally, there is degradation optimization, which differs from the optimizations we learned earlier. It elevates the perspective of performance optimization from a local and single viewpoint to an overall one, thus forming a comprehensive performance optimization solution. The author uses this case as the last chapter of this book, also aiming to provide readers with a new perspective on performance optimization and encourage them to explore more possibilities in performance optimization.

# 10.1 Power consumption optimization

Optimization of power consumption is divided into two steps. The first step is to do a good job in monitoring power consumption. Only based on power consumption monitoring can we further carry out governance and optimization. To do a good job in monitoring the power consumption of programs, the best approach is to first understand how the Android system performs application power consumption statistics, and then draw inspiration from it to implement and improve the power consumption monitoring plan for programs. So, let's first take a look at the principle of how the Android system performs power consumption statistics.&#x20;

## 10.1.1 Power Consumption Statistics Principle

The Android system has a function to count the power consumption of applications. As shown in Figure 10.1, in the battery option of the Android device settings page, you can see the power consumption of applications. Therefore, if we understand how the Android system counts power consumption, we will naturally know how to monitor the power consumption of our applications.&#x20;

![Figure 10.1 Program Power Consumption Statistics](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter10_img_1.png)

For the system, it is not possible to use devices such as ammeters to perform statistics on the power consumption of programs, as these hardware components are not supported by mobile phones. Therefore, when the system performs power consumption statistics on programs, it is based on the duration of use of modules such as the screen, CPU, Bluetooth, and network during the program's operation, and calculates the power consumption of the program according to the formula "rated power of the module *×* time consumed by module use". Although this statistical method cannot achieve very high precision, it can basically reflect the power consumption of each application.&#x20;

### 1. Module Power

Let's first take a look at the power consumption of each module. The power consumption of each module is different. Classified by calculation method, there are mainly the following three categories.

* The first category includes common device modules such as cameras (Camera), flashlights (FlashLight), and media players (MediaPlayer). Their operating power is basically consistent with the rated power, so calculating the module's power consumption only requires counting the module's usage duration and then multiplying it by the rated power.&#x20;

* The second category is the calculation method that data modules such as Wifi, mobile data, and Bluetooth need to follow, which is calculated in stages. Their operating power can be divided into different levels. For example, when the phone's Wifi signal is relatively weak, the Wifi module must operate at a relatively high power level to maintain the data link. Therefore, the power consumption calculation of such modules is somewhat similar to our daily electricity bill calculation, requiring "tiered billing".&#x20;

* The third category is the method where modules such as CPU and screen require multi-segment power consumption accumulation. In addition to each CPU core needing to calculate power consumption step by step like the data module, each cluster of the CPU (Cluster), which generally contains one or more cores with the same specifications, also has additional power consumption. Moreover, the entire CPU processor chip also has power consumption. Therefore, CPU power consumption = sum of power consumption of each core + power consumption of each cluster (Cluster) + chip power consumption. The screen module will calculate in multiple stages based on the duration of the program's screen lock (WakeLock) hold, and then according to different brightness levels and the corresponding power consumption at each brightness level.&#x20;

The power consumption of each module needs to be defined by the manufacturer itself, and the definition file for power consumption is generally located in the power\_profile.xml file of "/system/framework/framework-res.apk". Figure 10-2 shows the power\_profile.xml file of the author's test device.&#x20;

![Figure 10-2 power\_profile Power Definition File](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter10_img_2.png)

We cannot directly view the files in the APK package, so we need to use the apktool tool to decompile framework-res.apk. Some of the data in the decompiled power\_profile.xml is shown in the figure below.

![Figure 10-3 Power consumption definition data](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter10_img_3.png)

The description of each module in the power\_profile data can be found in Google's official documentation, and some of the data is shown in the following table.&#x20;

| Name        | Description                                                                                                              | Example Value        | Remarks                                                                                                                                               |
| ----------- | ------------------------------------------------------------------------------------------------------------------------ | -------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| ambient.on  | Additional power consumption of the screen in low-power/ low-light/ always-on mode (not off mode).                       |  100 mA              | -                                                                                                                                                     |
| screen.on   | Additional power consumption when the screen is turned on at the lowest brightness.                                      | 200 mA               | Includes touch controller and display backlight. The brightness is 0, rather than the minimum value of 10% or 20% typically set by Android.           |
| screen.full | The additional power consumption when the screen is at maximum brightness compared to when it is at minimum brightness.  | 100-300 milliamperes | Multiply this value by a certain ratio (based on screen brightness) and then add it to the screen.on value to calculate the screen power consumption. |
| wifi.on     | Additional power consumption when WLAN is turned on but not receiving, transmitting signals, or performing scans.        | 2 milliamperes       | -                                                                                                                                                     |
| wifi.active | Additional power consumption when sending or receiving signals via WLAN.                                                 | 31 mA                | -                                                                                                                                                     |

### 2. Module Time Consumption

Now that we've understood the power of the module, let's take a look at how the module's time consumption is calculated. Actually, whenever a power-consuming module is working or changing its state, it will notify the system service BatteryStatsService, and BatteryStatsService will call the [BatteryStatsImpl](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r9:frameworks/base/services/core/java/com/android/server/power/stats/BatteryStatsImpl.java) object to perform time-consuming statistics. BatteryStatsImpl holds timers and power consumption calculators for each module, which are used to calculate the usage duration and power consumption of each module. The process is shown in Figure 10-4.

![Figure 10-4 Module power consumption statistics flow](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter10_img_4.png)

The simplified code for the constructor function of BatteryStatsImpl is as follows. The logic in the code mainly initializes the batterystats.bin file, which is used to store the working duration of the module, and initializes the timers for each module.

```java
private BatteryStatsImpl(Clock clock, File systemDir, Handler handler,
        PlatformIdleStateCallback cb, MeasuredEnergyRetriever energyStatsCb,
        UserInfoProvider userInfoProvider) {
    init(clock);

    if (systemDir == null) {
        mStatsFile = null;
        mBatteryStatsHistory = new BatteryStatsHistory(mHistoryBuffer);
    } else {
        //init batterystats.bin file，used for store the working duration of module
        mStatsFile = new AtomicFile(new File(systemDir, "batterystats.bin"));
        mBatteryStatsHistory = new BatteryStatsHistory(this, systemDir, mHistoryBuffer);
    }
    ……
    //init tinmer of each module
    initTimersAndCounters();
    ……
}
```

The initTimersAndCounters function initializes the timers of each module in the system. Since it involves a large number of modules, the code for this function is relatively long. We can take a look at a partial code implementation of this function.&#x20;

```java
protected void initTimersAndCounters() {
    mScreenOnTimer = new StopwatchTimer(mClock, null, -1, null, mOnBatteryTimeBase);
    mScreenDozeTimer = new StopwatchTimer(mClock, null, -1, null, mOnBatteryTimeBase);
    ……
    mDeviceIdleModeFullTimer = 
            new StopwatchTimer(mClock, null, -14, null, mOnBatteryTimeBase);
    mDeviceLightIdlingTimer = 
            new StopwatchTimer(mClock, null, -15, null, mOnBatteryTimeBase);
    mDeviceIdlingTimer = new StopwatchTimer(mClock, null, -12, null, mOnBatteryTimeBase);
    mPhoneOnTimer = new StopwatchTimer(mClock, null, -3, null, mOnBatteryTimeBase);
    ……
    mWifiActivity = new ControllerActivityCounterImpl(mClock, mOnBatteryTimeBase,
            NUM_WIFI_TX_LEVELS);
    mBluetoothActivity = new ControllerActivityCounterImpl(mClock, mOnBatteryTimeBase,
            NUM_BT_TX_LEVELS);
    mModemActivity = new ControllerActivityCounterImpl(mClock, mOnBatteryTimeBase,
            ModemActivityInfo.getNumTxPowerLevels());
    mMobileRadioActiveTimer = 
            new StopwatchTimer(mClock, null, -400, null, mOnBatteryTimeBase);
    mMobileRadioActivePerAppTimer = new StopwatchTimer(mClock, null, -401, null,
            mOnBatteryTimeBase);
    mMobileRadioActiveAdjustedTime = new LongSamplingCounter(mOnBatteryTimeBase);
    mMobileRadioActiveUnknownTime = new LongSamplingCounter(mOnBatteryTimeBase);
    ……
    mAudioOnTimer = new StopwatchTimer(mClock, null, -7, null, mOnBatteryTimeBase);
    mVideoOnTimer = new StopwatchTimer(mClock, null, -8, null, mOnBatteryTimeBase);
    mFlashlightOnTimer = new StopwatchTimer(mClock, null, -9, null, mOnBatteryTimeBase);
    mCameraOnTimer = new StopwatchTimer(mClock, null, -13, null, mOnBatteryTimeBase);
    mBluetoothScanTimer = new StopwatchTimer(mClock, null, -14, null, mOnBatteryTimeBase);
    ……
}
```

Having understood the initialization of the timers for each module, let's take a look at how these modules perform work duration statistics through a few examples.&#x20;

**1) Wifi Module**

When Wi-Fi is turned on, the BatteryStatsService will call the noteWifiOnLocked method in the BatteryStatsImpl object. In this method, a timer will be started to begin timing, and the time information will be written to the mStatsFile through the recordState2StartEvent method.

```java
public void noteWifiOnLocked(long elapsedRealtimeMs, long uptimeMs) {
    if (!mWifiOn) {
        mHistory.recordState2StartEvent(elapsedRealtimeMs, uptimeMs,
                HistoryItem.STATE2_WIFI_ON_FLAG);
        mWifiOn = true;
        mWifiOnTimer.startRunningLocked(elapsedRealtimeMs);
        scheduleSyncExternalStatsLocked("wifi-off", ExternalStatsSync.UPDATE_WIFI);
    }
}


```

When Wi-Fi is turned off, call the noteWifiOffLocked function to stop the timer, and update the stored working duration information through the recordState2StopEvent function.&#x20;

```java
public void noteWifiOffLocked(long elapsedRealtimeMs, long uptimeMs) {
    if (mWifiOn) {
        mHistory.recordState2StopEvent(elapsedRealtimeMs, uptimeMs,
                HistoryItem.STATE2_WIFI_ON_FLAG);
        mWifiOn = false;
        mWifiOnTimer.stopRunningLocked(elapsedRealtimeMs);
        scheduleSyncExternalStatsLocked("wifi-on", ExternalStatsSync.UPDATE_WIFI);
    }
}
```

**2) Audio Module**

The statistical method of the Audio module is similar. When opening and closing Audio, time information is written into the cache information through the recordStateStartEvent and recordStateStopEvent methods.&#x20;

```java
public void noteAudioOnLocked(int uid, long elapsedRealtimeMs, long uptimeMs) {
    uid = mapUid(uid);
    if (mAudioOnNesting == 0) {
        mHistory.recordStateStartEvent(elapsedRealtimeMs, uptimeMs,
                HistoryItem.STATE_AUDIO_ON_FLAG);
        mAudioOnTimer.startRunningLocked(elapsedRealtimeMs);
    }
    mAudioOnNesting++;
    getUidStatsLocked(uid, elapsedRealtimeMs, uptimeMs)
            .noteAudioTurnedOnLocked(elapsedRealtimeMs);
}

public void noteAudioOffLocked(int uid, long elapsedRealtimeMs, long uptimeMs) {
    if (mAudioOnNesting == 0) {
        return;
    }
    uid = mapUid(uid);
    if (--mAudioOnNesting == 0) {
        mHistory.recordStateStopEvent(elapsedRealtimeMs, uptimeMs,
                HistoryItem.STATE_AUDIO_ON_FLAG);
        mAudioOnTimer.stopRunningLocked(elapsedRealtimeMs);
    }
    getUidStatsLocked(uid, elapsedRealtimeMs, uptimeMs)
            .noteAudioTurnedOffLocked(elapsedRealtimeMs);
}
```

### 3. Power consumption calculation

Once the system has the time taken for each module to operate, it can calculate the power consumption of each module. The power consumption of each program that we see on the system settings page is ultimately obtained through the [getCurrentBatteryUsageStats function](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r9:frameworks/base/services/core/java/com/android/server/power/stats/BatteryUsageStatsProvider.java) in the BatteryUsageStatsProvider content provider. The partial code implementation in this method is as follows:

```java
private BatteryUsageStats getCurrentBatteryUsageStats(BatteryStatsImpl stats,
        BatteryUsageStatsQuery query, long currentTimeMs) {
    ……
    final List<PowerCalculator> powerCalculators = getPowerCalculators();
    // Traverse the power calculators for each module
    for (int i = 0, count = powerCalculators.size(); i < count; i++) {
        PowerCalculator powerCalculator = powerCalculators.get(i);
        if (powerComponents != null) {
            boolean include = false;
            for (int powerComponent : powerComponents) {
                if (powerCalculator.isPowerComponentSupported(powerComponent)) {
                    include = true;
                    break;
                }
            }
            if (!include) {
                continue;
            }
        }
        // Calculate the power consumption of the module and pass it into batteryUsageStatsBuilder
        powerCalculator.calculate(batteryUsageStatsBuilder, 
                stats, 
                realtimeUs, 
                uptimeUs, query);
    }

   ……
   return batteryUsageStats;
}
```

It can be seen that in this method, each PowerCalculator, which is the power consumption module calculator, will be traversed, and then the calculate method will be called to calculate the power consumption. Here, the author still uses the Wifi module as an example to explain. The implementation class of this module is [WifiPowerCalculator.java](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r9:frameworks/base/services/core/java/com/android/server/power/stats/WifiPowerCalculator.java), and the partial code of the calculate method of this object is as follows:&#x20;

```java
public void calculate(BatteryUsageStats.Builder builder, BatteryStats batteryStats,
        long rawRealtimeUs, long rawUptimeUs, BatteryUsageStatsQuery query) {
    BatteryConsumer.Key[] keys = UNINITIALIZED_KEYS;
    long totalAppDurationMs = 0;
    double totalAppPowerMah = 0;
    final PowerDurationAndTraffic powerDurationAndTraffic = new PowerDurationAndTraffic();
    ……
    // Calculate the power consumption of the Wi-Fi module
    calculateApp(powerDurationAndTraffic, app.getBatteryStatsUid(), powerModel,
            rawRealtimeUs, BatteryStats.STATS_SINCE_CHARGED,
            batteryStats.hasWifiActivityReporting(), consumptionUC);
       ……
    // Calculate how long the Wi-Fi can continue to be used under the current battery level
    calculateRemaining(powerDurationAndTraffic, powerModel, batteryStats, rawRealtimeUs,
            BatteryStats.STATS_SINCE_CHARGED, batteryStats.hasWifiActivityReporting(),
            totalAppDurationMs, totalAppPowerMah, consumptionUC);
    
    ……
}
```

In the above code, the calculateApp method is mainly used to calculate the energy consumption when the program transmits data via Wifi. It calculates the power consumption based on the data sending and receiving duration, idle duration of the application, and the power consumption parameters defined in power\_profile.xml. The calculateRemaining method is used to estimate how long the device can continue to use Wifi with the current battery level. Let's take a look at how calculateApp calculates power consumption. The simplified code implementation is as follows. In the calculatePower method at the end of the process, the simple formula "module power × module time" is used to calculate power consumption.

```java
private void calculateApp(PowerDurationAndTraffic powerDurationAndTraffic,
        BatteryStats.Uid u, @BatteryConsumer.PowerModel int powerModel,
        long rawRealtimeUs, int statsType, boolean hasWifiActivityReporting,
        long consumptionUC) {
    ……
    if (hasWifiActivityReporting && mHasWifiPowerController) {
          ……
        final long rxTime = rxTimeCounter.getCountLocked(statsType);
        final long txTime = txTimeCounter.getCountLocked(statsType);
        final long idleTime = idleTimeCounter.getCountLocked(statsType);
        // Calculate total power consumption based on Wi-Fi upload, download, and idle time
        powerDurationAndTraffic.durationMs = idleTime + rxTime + txTime;
        if (powerModel == BatteryConsumer.POWER_MODEL_POWER_PROFILE) {
            powerDurationAndTraffic.powerMah =
                    calcPowerFromControllerDataMah(rxTime, txTime, idleTime);
        } else {
            powerDurationAndTraffic.powerMah = uCtoMah(consumptionUC);
        }
        ……
    } 
    ……
}
    
public double calcPowerFromControllerDataMah(long rxTimeMs, long txTimeMs, long idleTimeMs) {
    return mRxPowerEstimator.calculatePower(rxTimeMs)
            + mTxPowerEstimator.calculatePower(txTimeMs)
            + mIdlePowerEstimator.calculatePower(idleTimeMs);
}

public double calculatePower(long durationMs) {
    return mAveragePowerMahPerMs * durationMs;
}

```

## 10.1.2 Power Consumption Monitoring

Although Android calculates the power consumption of each program by summing up the power consumption of individual working modules, applications cannot obtain such detailed data, and the Android system only provides the BatteryManager object to applications for obtaining the current battery percentage. However, we can use BatteryManager to calculate the total power consumption of a program over a period of time. Based on the principle of Android's power consumption statistics, we can also follow a similar approach to calculate the usage time of each module to refine the power consumption statistics.

### 1. Total power consumption

The total power consumption of the program can be measured at a fixed frequency, such as once every 10 minutes, to calculate and report the power consumption of the in-memory program during this period. After the server receives the data, it can classify and aggregate the data on a daily or hourly basis, thus enabling the calculation of the user's power consumption while using the program. The code for power consumption statistics is as follows.&#x20;

```java
int beforeBattery = 0;
void startBatteryUsageMonitor(){
    BatteryManager mBatteryManager 
            = (BatteryManager) context.getSystemService(Context.BATTERY_SERVICE);
    // Schedule a battery usage statistics task to run every 10 minutes using a thread pool
    scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
        @Override
        public void run() {
            int battery =
                 mBatteryManager.getIntProperty(BatteryManager.BATTERY_PROPERTY_CAPACITY);
            if(beforeBattery != 0){
                // The battery consumption for this period is the current battery percentage minus the battery percentage from 10 minutes ago
                int batteryUsage = beforeBattery - battery;
                report(batteryUsage);
            }
            beforeBattery = battery;
        }
    }, 0, 10*60, TimeUnit.SECONDS);
}
```

However, such a solution is not very accurate, and we still need to exclude power consumption statistics in some scenarios. For example, in scenarios such as charging and background operations, we need to turn off the monitoring of power consumption.

* When the phone is charging, the statistics on power consumption are inaccurate. Therefore, we can monitor whether the device is in a charging state. If a notification indicating that the phone is charging is received within the statistical frequency, we will stop monitoring power consumption until a notification indicating that the user has stopped charging is received, after which we will resume the statistics. The monitoring code for whether the device is charging is as follows.&#x20;

```java
private BroadcastReceiver batteryReceiver = new BroadcastReceiver() {
    @Override
    public void onReceive(Context context, Intent intent) {
        String action = intent.getAction();
        
        // Handle power connected event
        if (action.equals(Intent.ACTION_POWER_CONNECTED)) {
            // Pause battery usage statistics when power is connected
            stopBatteryUsageMonitor();
        }
        
        // Handle power disconnected event
        if (action.equals(Intent.ACTION_POWER_DISCONNECTED)) {
            // Resume battery usage statistics when power is disconnected
            startBatteryUsageMonitor();
        }
    }
};
```

* When the program is in the background, it indicates that the user is using other programs. At this time, almost all the power consumption statistics are caused by other programs. If power consumption is still being counted at this time, it will affect the accuracy of the data. Therefore, when the program is in the background, we need to stop monitoring power consumption, and resume monitoring when it returns to the foreground. By registering LifecycleCallback to monitor the lifecycle of Activity, foreground and background determination can be achieved. Whenever an Activity executes the Resume cycle, the counter is incremented by one; when an Activity executes the Pause cycle, the counter is decremented by one. When the counter is zero, it indicates that the application is in the background, at which point power consumption statistics can be turned off. When the counter recovers from zero to one, it indicates that the application has entered the foreground from the background, and power consumption statistics can be restarted.&#x20;

```java
int mActivityCount = 0;
context.registerActivityLifecycleCallbacks(new Application.ActivityLifecycleCallbacks() {
    @Override
    public void onActivityResume(Activity activity, Bundle savedInstanceState) {
        // Restart battery usage statistics when the app is in the foreground
        if (mActivityCount == 0) {
            startBatteryUsageMonitor();
        }
        mActivityCount++;
    }

    @Override
    public void onActivityPause(Activity activity) {
        mActivityCount--;
        if (mActivityCount == 0) {
            // Pause battery usage statistics when the app is in the background
            stopBatteryUsageMonitor();
        }
    }
});
```

### 2. Refinement of power consumption

From the Android system's API, we can only obtain the overall power consumption. However, the total power consumption of the program is not very helpful to us in many cases, because even if there is an abnormal power consumption, we do not know where the problem lies. In order to more quickly detect and locate problems when abnormal power consumption occurs, we also need to further refine the types and scenarios of power consumption through monitoring. We can refine it in two directions: modules and scenarios. The following is the refinement plan.

**1) Consumption Module**

Although we cannot directly obtain the detailed types of power consumption, based on the previous knowledge, we know that the system calculates power consumption by multiplying the usage duration of a module by its power. Therefore, we can also monitor and count the usage duration of core modules to roughly attribute the power consumption of the program. Common modules with high power consumption, such as GPS, Audio, Camera, Video, etc., need to be counted both before and after manual use to calculate the usage duration, and then reported to the server. The server then attributes the power consumption based on the hourly or daily power consumption of the program and the usage duration of each module within the corresponding time period.&#x20;

**2) Consumption Scenarios**

In addition to refining the types of power consumption, we can also monitor the scenarios of power consumption, which can be scenarios with Activity as the dimension or custom scenarios. Let's first take a look at the scenarios with Activity as the dimension. Through LifecycleCallbacks, we can know the start and destruction of each Activity in the program. Therefore, we can use this mechanism to monitor the power consumption of Activity-level scenarios. We obtain the current power value before the Activity starts, and then obtain the power value again after the Activity ends. By calculating the difference, we can get the power consumption during this Activity scenario. We can further improve this solution. If charging is detected during the use of the scenario, we will abandon the power consumption statistics for this time. The code implementation is as follows.&#x20;

```java
context.registerActivityLifecycleCallbacks(new Application.ActivityLifecycleCallbacks() {
    @Override
    public void onActivityPaused(Activity activity, Bundle savedInstanceState) {
        String activityName = activity.getClass().getName();
        // Record the battery level before the activity starts
        startBatteryUsageMonitors(activityName);
    }

    @Override
    public void onActivityResumed(@NonNull Activity activity) {
        String activityName = activity.getClass().getName();
        // Record the battery level after the activity ends
        stopBatteryUsageMonitor(activityName);
    }
}
    
private HashMap<String, Integer> mSceneBatteryConsumeMap = new HashMap();

private void startBatteryUsageMonitors(String sceneName){
    mSceneBatteryConsumeMap.put(sceneName,
            mBatteryManager.getIntProperty(BatteryManager.BATTERY_PROPERTY_CAPACITY)); 
}

private void stopBatteryUsageMonitor(String sceneName){
    Integer sceneStartBattery = mSceneBatteryConsumeMap.remove(sceneName);  
    // Calculate the battery consumption of the scene only if no charging is happening and the value is available
    if(!charged && sceneStartBattery != null){
        Log.i(TAG,"Scene："+sceneName+" consumed:"+ (sceneStartBattery 
            - mBatteryManager.getIntProperty(BatteryManager.BATTERY_PROPERTY_CAPACITY)))
    }
}
```

In addition to the non-invasive scenario provided by Lifecycle, we can also offer two global singleton methods, enterScene and stopScene, so that in the code logic for starting and ending business operations, developers can manually call enterScene and stopScene to perform power consumption statistics, thereby achieving monitoring of power consumption in custom scenarios. The code is as follows.&#x20;

```java
void enterScene(String sceneName){
    // Start battery consumption statistics
    startBatteryUsageMonitors(sceneName);
}

void stopScene(String sceneName){
    // Receive battery consumption statistics
    stopBatteryUsageMonitor(sceneName);
}
```

## 10.1.3 Power Consumption Management

With a comprehensive power consumption monitoring system in place, it becomes possible to monitor whether the power consumption of a program is abnormal. If the power consumption within a unit of time exceeds the threshold, such as 10% power consumption in 10 minutes, this is clearly abnormal. When we receive an abnormal alarm through the server, we need to investigate and optimize the abnormal power consumption. During the investigation and optimization process, we mainly have two directions: one is to optimize based on the logs reported by online monitoring, and the other is to identify and optimize high-power consumption modules through offline power consumption analysis.&#x20;

### 1. Online power consumption management

In the previous monitoring plan, we refined power consumption down to modules and scenarios, so we can locate specific power consumption anomalies in modules and scenarios based on monitoring. The governance of power consumption patterns mainly includes the following methods.&#x20;

* CPU: If the power consumption is caused by the CPU running at high load for a long time, it is very likely that there are many functions in the business code, such as dead loop functions, high-frequency functions, high-time-consuming functions, and invalid functions, which lead to abnormal CPU consumption. We need to conduct further analysis and governance based on the high-power consumption time periods in the power consumption monitoring, combined with the corresponding Logs and code logic.

* GPU and Screen: If excessive power consumption is caused by the GPU or screen, then it is necessary to reduce overdraw in the project, as well as scenarios that waste the GPU, such as excessive animations and animations in invisible areas. We can also further reduce the power consumption of the screen or GPU by actively reducing screen brightness, using a dark UI, and other solutions.

* Network: Regarding network power consumption, we need to reduce network access frequency, consolidate network requests, and reduce traffic data as much as possible without affecting business and performance. We can further optimize the download logic in the project and perform data pre-download and upload logic during charging scenarios.

* GPS: When managing the power consumption of GPS, we need to combine business scenarios, reasonably reduce accuracy, and reduce the request frequency to minimize the power consumption of GPS.

The power consumption management of business scenarios is basically the same as that of power consumption modules, except that we can further narrow the scope of management to a specific business execution process, mainly through the following methods&#x20;

* Reduce the frequency or discard periodic tasks in business scenarios. For example, when the business enters the background, periodic tasks can be frequency-reduced or discarded

* Combine data from multiple business requests into one&#x20;

* Reduce the execution of high-frequency functions that consume more resources in business operations, etc.

### 2. Offline power consumption analysis

Often, the useful information obtained through online monitoring is insufficient. Therefore, we can analyze the power consumption of the program offline to collect more useful information. Offline power consumption analysis can use the [battery-historian](https://developer.android.com/topic/performance/power/setup-battery-historian) provided by Google. For the installation tutorial, please refer to the official documentation.

When using Battery Historian, you first need to call the "adb shell dumpsys batterystats --reset" command to reset the device's power consumption information. Then, you need to use the phone for a period of time without charging. Next, connect the device and execute the "adb shell dumpsys batterystats > batterystats.txt" command to export the power consumption information file. Finally, import batterystats.txt into the analysis interface provided by Battery Historian.

We can filter the programs we want to analyze in the Battery Historian interface, as shown in Figure 10-5. In the analysis interface, we can obtain detailed information such as the program's power consumption, network information, screen time, number of wake locks, CPU time, etc. Based on this information, we can further analyze and locate the reasons for the program's high power consumption. Google's official documentation has a very detailed tutorial on using battery-historian, so we won't go into further detail here. Readers can refer to the official documentation and practice once.&#x20;

![Figure 10-5 Battery Historian Interface](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter10_img_5.png)

# 10.2 Traffic Optimization

Although data charges have become increasingly affordable, there are still many users whose data allowance is not sufficient. If excessive data consumption leads to users' financial losses, it is likely to attract complaints and damage the brand's reputation. In addition, optimizing data usage can also save server bandwidth, resulting in direct cost savings. Therefore, data optimization is not an optimization that we can afford to overlook.

## 10.2.1 **Traffic Consumption Monitoring**

To do a good job in traffic optimization, it is also necessary to start with good monitoring. There are three ways in Android to monitor traffic consumption, which are as follows.&#x20;

1. On Android versions below Android 9, you can read the "/proc/net/xt\_qtaguid/stats" file to obtain detailed traffic consumption information for applications

2. Use the methods provided by Android's TrafficStats object to obtain traffic consumption

3. Use Android's NetworkStatsManager object to obtain traffic consumption

Details of the usage of these three methods are as follows:

**1) t\_qtaguid Module**

Let's first look at the first method. For Android versions below 9, you can obtain the traffic consumption information of applications by directly reading the "/proc/net/xt\_qtaguid/stats" file. This file contains the network traffic statistical data of each application on the Android device. Below is an example of the data in the stats file.

```bash
idx iface acct_tag_hex uid_tag_int cnt_set rx_bytes rx_packets tx_bytes tx_packets
44  wlan0      0x0         10123      0      45148     186      32150     265
45  wlan0      0x0         10123      1        0        0         0        0
46  wlan0      0x0         10138      0      19775     84        13625    129
47  wlan0      0x0         10138      1        0        0         0        0
```

The explanations from left to right in the above data are as follows:&#x20;

| Field Name     | Explanation                                                                                                                                                                                                   |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| idx            | Serial number, representing the index of the record                                                                                                                                                           |
| iface          | Network interfaces, such as wlan representing the physical interface of a Wi-Fi network adapter, rmnet\_data representing the mobile data network interface, and io representing the local loopback interface |
| acct\_tag\_hex | Account tags are used to distinguish different types of traffic or different modules or threads within an application                                                                                         |
| uid\_tag\_int  | User Identification, used to identify which application the traffic statistical data belongs to                                                                                                               |
| cnt\_set       | Counting set, used to distinguish between foreground (1) and background (0) traffic                                                                                                                           |
| rx\_bytes      | Received Bytes, representing the total number of bytes received by the application                                                                                                                            |
| rx\_packets    | Received Packet Count, representing the number of data packets received by the application                                                                                                                    |
| tx\_bytes      | Sent ByteDance, representing the total number of bytes sent by the application                                                                                                                                |
| tx\_packets    | Number of Sent Packets, representing the quantity of data packets sent by the application                                                                                                                     |

Once we understand the meaning of each field, we can implement the process of reading. By reading the "stats" file line by line and finding the data segment corresponding to the ID of our application, and then performing data aggregation. When aggregating and counting traffic data, we also need to further determine whether it is Wi-Fi traffic consumption or non-Wi-Fi traffic consumption based on the iface field, because if Wi-Fi traffic consumption is high, we usually don't need to pay too much attention, but if mobile traffic consumption is high, we need to pay extra attention. The code implementation is as follows.&#x20;

```java
public static long[] readAppTraffic() throws IOException {
    long[] appTraffic = new long[2]; 
    BufferedReader reader = null;

    try {
        reader = new BufferedReader(new FileReader("/proc/net/xt_qtaguid/stats"));
        String line;

        while ((line = reader.readLine()) != null) {
            // Split each line by whitespace characters
            String[] parts = line.trim().split("\\s+");
            int currentUid = android.os.Process.myUid();
            if (parts.length >= 8) {
                String iface = parts[1];
                int uidTag = Integer.parseInt(parts[2]);
                long rxBytes = Long.parseLong(parts[5]);
                long txBytes = Long.parseLong(parts[7]);
                // Check the data type: Wi-Fi, mobile network, or local loopback, and verify if the UID matches the current process
                if ((iface.startsWith("wlan") 
                        || iface.startsWith("rmnet") 
                        || iface.startsWith("ccmni")) 
                        && uidTag == currentUid) {
                    if (iface.startsWith("wlan")) {
                        // Count Wi-Fi traffic consumption
                        appTraffic[0] += rxBytes + txBytes; 
                    } else {
                        // Count non-Wi-Fi traffic consumption
                        appTraffic[1] += rxBytes + txBytes; 
                    }
                }
            }
        }
    } finally {
        if (reader != null) {
            reader.close();
        }
    }

    return appTraffic;
}
```

The TrafficStats interface of the t\_qtaguid module can only query the total traffic volume at the current time, so a periodic task thread pool is needed to perform data collection at a fixed frequency. The implementation scheme is the same as that for collecting battery power earlier, which only requires calling readAppTraffic at regular intervals to obtain traffic data and then calculating the difference. The author will not elaborate further.&#x20;

**2) TrafficStats Interface**

Starting from Android 9, Google has gradually phased out support for the xt\_qtaguid module, so it is no longer possible to obtain traffic usage information by reading the xt\_qtaguid file on devices with higher versions. At this time, the TrafficStats class can be used to obtain the current traffic, which is relatively simple to implement and only requires directly calling the corresponding interface.&#x20;

```java
synchronized long getCurrentBytes() {
    long totalRxBytes = TrafficStats.getTotalRxBytes();
    long totalTxBytes = TrafficStats.getTotalTxBytes();

    if (totalRxBytes != TrafficStats.UNSUPPORTED 
            || totalTxBytes != TrafficStats.UNSUPPORTED) {
        // Get total device traffic consumption
        long totalBytes = totalRxBytes + totalTxBytes;
        // Get current device traffic
        long uidRxBytes = TrafficStats.getUidRxBytes(android.os.Process.myUid());
        long uidTxBytes = TrafficStats.getUidTxBytes(android.os.Process.myUid());

        if (uidRxBytes != TrafficStats.UNSUPPORTED 
                || uidTxBytes != TrafficStats.UNSUPPORTED) {
            // Return the traffic consumption of the device
            return uidRxBytes + uidTxBytes;
        } 
        return -1;
    } else {
        return -1;
    }
}
```

**3）NetworkStatsManager&#x20;**

Although TrafficStats supports viewing the overall traffic consumption information of a specified program, it does not support differentiation based on network interfaces. Therefore, it cannot clearly distinguish between the traffic consumption of Wi-Fi and mobile data, and the ability to distinguish this is very important for traffic monitoring. In this case, we can use the third method, NetworkStatsManager, to perform traffic statistics.

NetworkStatsManager is a powerful tool in Android 6 and later versions for monitoring network traffic data. It provides access to historical network usage data and can be used to obtain information on multiple types of network traffic, including Wi-Fi and mobile data traffic. The following is how to use NetworkStatsManager. The queryDetailsForUid method provided by this object supports querying traffic consumption over a certain period, so there is no need to calculate the difference through periodic tasks.

```java
@RequiresApi(api = Build.VERSION_CODES.M)
public static long[] getNetworkUsageStats(Context context, int uid) {
    NetworkStatsManager networkStatsManager = 
           (NetworkStatsManager)context.getSystemService(Context.NETWORK_STATS_SERVICE);
    NetworkStats networkStats = null;
    long rxBytes = 0L;
    long txBytes = 0L;

    try {    
        // End time is the current time
        Instant endTime = Instant.now(); 
        // Start time is ten minutes before the current time
        Instant startTime = endTime.minus(Duration.ofMinutes(10)); 
        // Query the traffic data for the past ten minutes for the app by UID
        networkStats  = networkStatsManager.queryDetailsForUid(
                ConnectivityManager.TYPE_WIFI, 
                "", 
                startTime.toEpochMilli(), 
                endTime.toEpochMilli(), 
                android.os.Process.myUid());

        NetworkStats.Bucket bucket = new NetworkStats.Bucket();
        // Retrieve detailed traffic consumption data
        while (networkStats.hasNextBucket()) {
            networkStats.getNextBucket(bucket);
            rxBytes += bucket.getRxBytes();
            txBytes += bucket.getTxBytes();
        }
    } catch (RemoteException e) {
        e.printStackTrace();
    } finally {
        if (networkStats != null) {
            networkStats.close();
        }
    }

    return new long[]{rxBytes, txBytes};
}
```

## 10.2.2 Traffic Classification

Simply counting how much traffic has been consumed is often insufficient; we need to classify traffic in a more detailed manner to help us locate anomalies more quickly. In addition to basic classifications such as Wi-Fi traffic and mobile traffic, uplink (data transmission) traffic and downlink (data reception) traffic, we can also classify traffic by source or scenario.&#x20;

### 1. Consumption Source

During program usage, traffic consumption may come from OkHttp, may come from Webview, or may come from our custom Socket. If we can identify the sources of traffic consumption, it will greatly assist us in analyzing abnormal traffic consumption. Here, the author mainly explains how to monitor traffic consumption from OkHttp and Webview.&#x20;

**1) OkHttp Traffic Monitoring**

OkHttp is the most widely used network request library, which supports custom interceptors for adding custom logic during network requests. Therefore, we can add code to the request interceptor to monitor traffic. The sample code is as follows, which only prints the traffic consumption of this OKHttp request. In a real scenario, we can accumulate the traffic consumption of all OkHttp requests over a period of time, and then print and report it.

```java
public class OkHttpMonitorInterceptor implements Interceptor {

    private static final String TAG = "OkHttpInterceptor";

    @Override
    public Response intercept(Chain chain) throws IOException {
        Request request = chain.request();
        // Get the traffic consumption of the network request
        long txBytes = request().body() != null ? 
                request().body().contentLength() : 0;
        
        Response response = chain.proceed(request);
        // Get the traffic consumption of data reception
        long rxBytes = response.body() != null ? 
                response.body().contentLength() : 0;      
        // Get the URL link
        String url = request.url();

        Log.i(TAG, "[url]"+ url
                + " total:"+ (txBytes + rxBytes)
                + " [txBytes]:" + txBytes
                + " [rxBytes]:"+ rxBytes);
        return response;
    }
}
```

After defining the interceptor for traffic monitoring, simply add it when initializing OkHttp, and the code is as follows.&#x20;

```java
OkHttpClient client = new OkHttpClient.Builder()
        .addInterceptor(new TrafficMonitoringInterceptor())
        .build();
```

**2) Webview traffic monitoring**

When monitoring the traffic consumption of Webview, we can pass in a custom WebViewClient, record the traffic consumption at the start of the onLoadResource method of the WebViewClient, calculate the traffic consumption after loading is completed in the onPageFinished method, and the difference in traffic consumption is the traffic consumption of Webview during this period. The code implementation is as follows.&#x20;

```java
webView.setWebViewClient(new WebViewClient() {
    @Override
    public void onLoadResource(WebView view, String url) {
        super.onLoadResource(view, url);
        // Record the starting traffic consumption each time a resource is loaded
        webViewRxBytesStart = TrafficStats.getTotalRxBytes();
        webViewTxBytesStart = TrafficStats.getTotalTxBytes();
    }

    @Override
    public void onPageFinished(WebView view, String url) {
        super.onPageFinished(view, url);
        // Calculate the traffic consumption after loading is complete
        long webViewRxBytesEnd = TrafficStats.getTotalRxBytes();
        long webViewTxBytesEnd = TrafficStats.getTotalTxBytes();
        long rxBytes = webViewRxBytesEnd - webViewRxBytesStart;
        long txBytes = webViewTxBytesEnd - webViewTxBytesStart;

        // Print the traffic consumption during WebView page loading
        Log.i(TAG, "Url:"+url 
                +" WebView Rx Bytes: " + rxBytes
                + " WebView Tx Bytes: " + txBytes);
    }
});
```

### 2. Consumption Scenario&#x20;

Similar to the statistics of power consumption, the statistics of consumption scenarios can also use dimensions such as Activity, custom business, foreground and background as scenarios, and count the traffic consumption within this scenario before and after the start of the scenario. We can still add the ability to count traffic in startScene and stopScene, which can reduce the complexity of business-side calls and strengthen the functions of these two global methods, startScene and stopScene. The code is as follows.&#x20;

```java
void enterScene(String sceneName){
    // Start battery consumption statistics
    startBatteryUsageMonitors(sceneName);
    // Start traffic consumption statistics
    startTrafficUsageMonitors(sceneName);
}

void stopScene(String sceneName){
    // Stop battery consumption statistics
    stopBatteryUsageMonitor(sceneName);
    // Stop traffic consumption statistics
    stopTrafficUsageMonitor(sceneName);
}

private HashMap<String, long[]> mSceneTrafficConsumeMap = new HashMap();
void startTrafficUsageMonitors(String sceneName){
    mSceneTrafficConsumeMap.put(sceneName,
            new long[]{TrafficStats.getTotalRxBytes(),TrafficStats.getTotalTxBytes()});
}

void stopTrafficUsageMonitor(String sceneName){
    long[] beforeTraffic = mSceneTrafficConsumeMap.get(sceneName);
    long rxBytes = TrafficStats.getTotalRxBytes()- beforeTraffic[0] ;
    long txBytes = TrafficStats.getTotalTxBytes()- beforeTraffic[1];
    Log.i(TAG, " Rx Bytes: " + rxBytes+ "Tx Bytes: " + txBytes);
}
```

## 10.2.3 Traffi&#x63;**&#x20;**&#x4F;ptimizatio&#x6E;**&#x20;**

When we can identify the source of traffic consumption through monitoring, it becomes much easier to optimize. Here are some commonly used optimization solutions.&#x20;

* Business Logic Exception Fix: In some cases, excessive traffic consumption is caused by abnormal business logic code such as infinite loops. To address these issues, we need to locate the specific abnormal scenarios and time periods based on the previous monitoring, and then further investigate and fix them by combining Logs and business logic code.

* Data pre-download: When the user is connected to Wi-Fi, data that consumes a large amount of traffic is downloaded in advance. This is the most commonly used and effective optimization solution for saving user traffic.

* Data Compression: There are many data compression schemes, and they are all relatively common. Common data compression schemes include using Gzip compression for the content of POST requests to reduce the amount of transmitted data, compressing request headers, reducing the transmission of duplicate information by passing the MD5 value of the request header, and adopting data formats with higher compression ratios such as Protbuff to replace data formats like json and xml, etc.

* Incremental Update: Incremental update is also a commonly used solution in traffic optimization. This solution requires adding version control to the server-side data. When the client pulls data, it brings in the data version number and only pulls the data that has changed between different versions, which can reduce a lot of unnecessary traffic consumption.&#x20;

* Image Optimization: In many cases, images are one of the main sources of traffic consumption in a program. When optimizing images, we can adopt on-demand image loading, or when loading images, first load only the thumbnail of the corresponding size, i.e., only load the original image when the user views the large image. This approach not only saves traffic but also saves memory. In terms of image sources, using the WebP format instead of PNG or JPEG formats further compresses the image size, thereby reducing traffic consumption caused by image transmission.&#x20;

* Merge Request: Merge network requests to reduce the number of requests. For example, for some interface types such as statistics, real-time reporting is not required. Save the statistical information locally and then upload it uniformly according to the strategy. In this way, the header information only needs to be uploaded once, and if the data volume is large, it can also save a significant amount of traffic.&#x20;

# 10.3 Disk Occupancy Optimization

Surely many people have been troubled by WeChat's huge disk space occupation, but they couldn't uninstall the program due to concerns about losing chat records. However, if the program we develop doesn't have the advantages of WeChat, it will be mercilessly uninstalled by users when the issue of excessive disk space occupation arises. Therefore, we must ensure that the disk space occupied by the program is not excessive.

## 10.3.1 **Disk Monitoring**

If you want to ensure that the disk usage of a program does not become too large, the first step is still to monitor the program's disk usage. The monitoring solution for disk usage is relatively simple. You only need to obtain the corresponding File object based on the file path, and then call the length method to get the file size. To obtain the total disk usage of an entire directory, you need to traverse the directory and then accumulate the sizes. The code implementation is as follows. If the number of files is relatively large, the traversal may be time-consuming, so we'd better perform the statistics in a child thread.&#x20;

```java
File file = new File(filePath); // filePath is the path of the file or directory to check
long fileSize = file.length(); // Get the file size

// If filePath is a directory, traverse all files under the directory and accumulate their sizes
if (file.isDirectory()) {
    File[] files = file.listFiles();
    for (File f : files) {
        fileSize += f.length();
    }
}
```

Having learned how to calculate the file size of a specified directory, we also need to know the total disk size of the mobile device. By using the StatFs object provided by the system, we can obtain the total disk size of the specified directory. The code implementation is as follows. In the code, by passing the path of Environment.getDataDirectory().getPath, we can obtain the total disk size of the internal storage directory.&#x20;

```java
public static long getTotalSpace() {
    StatFs statFs = new StatFs(Environment.getDataDirectory().getPath());
    long totalBlocks = statFs.getBlockCountLong();
    long blockSize = statFs.getBlockSizeLong();
    return totalBlocks * blockSize;
}
```

In the above scenarios, a specified directory must be passed in to determine the file size of that directory or the total disk size of the directory. Therefore, we need to further understand the storage directories in Android.&#x20;

## 10.3.2 Storage Directory

Storage directories in Android are divided into internal storage directories and external storage directories, and their detailed descriptions are as follows.&#x20;

**1) Internal Storage**

When the system installs the APK package of an application, it copies and extracts the APK package to the "data/app" directory, and creates a data directory for the application under the path "data/data/package name/". This directory is the application's internal storage directory (InternalStorage), which is used to store the application's private data. Therefore, only the application itself can access these files, and other applications cannot. When the application is uninstalled, the data in the internal storage will be automatically deleted.&#x20;

We usually directly obtain the directory of internal storage through the following methods:&#x20;

* Environment.getDataDirectory(): Returns the file object for the "/data" directory

* context.getFilesDir(): Returns the directory file object for "/data/data/package\_name/files", which is typically used to store private files of the application

* context.getCacheDir(): Returns a file object with the path "/data/data/package\_name/cache", which is used to store temporary cache data.

* context.getDataDir(): The directory obtained is "/data/data/package name/", which is the root directory of the application's internal storage

**2) External Storage**

External Storage originally referred to the storage space of the SD card, which is the storage directory under the path "/sdcard/". However, mainstream mobile devices now do not have SD cards, so the directory of " / sdcard/" and the directory of "/storage/emulated/0" were merged into the same directory, collectively referred to as the external storage directory.The main purpose of the external storage directory is to store public files, such as photos, videos, and other public files, as shown in Figure 10-6. The data of these public files is accessible to all programs.&#x20;

![Figure 10-6 Public file directory under /sdcard/directory](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter10_img_6.png)

Through the Environment.getExternalStoragePublicDirectory(String type) method, we can obtain the directory of external public storage, where the parameter type has the following types.&#x20;

* DIRECTORY\_MUSIC: The path is "/storage/emulated/0/Music", used to store music files

* DIRECTORY\_PICTURES: The path is "/storage/emulated/0/Pictures", used to store image files

* DIRECTORY\_MOVIES: The path is "/storage/emulated/0/Movies", used for storing video files

* DIRECTORY\_DOWNLOADS: The path is "/storage/emulated/0/Download", used to store downloaded files

* DIRECTORY\_DCIM: The path is "/storage/emulated/0/DCIM", used to store photo and video files taken and recorded by the camera application

* DIRECTORY\_DOCUMENTS: The path is "/storage/emulated/0/Documents", used to store document files, such as PDF documents, Word documents, e-books, etc.

In addition to public directories, external storage directories also include private directories. Private directories can be accessed not only by the application itself but also by other applications through FileProvider. Therefore, external private directories are used to store data that is not highly private or sensitive and needs to be exposed to other applications. The usage of private directories in external storage is as follows:&#x20;

* contxt.getExternalCacheDir(): Get the file object for the directory with the path "/emulated/0/Android/data/package\_name/cache"

* context.getExternalFilesDir(null): Obtain a file object for the directory with the path "/emulated/0/Android/data/packagename/files"

## 10.3.3 Disk Optimization

The system provides a disk cleanup page for programs. We need to enter the information page of the specific application, then go to the storage and cache page, and we can perform disk cleanup, as shown in Figure 10-7.&#x20;

![Figure 10-7 Application disk cleanup page](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter10_img_7.png)

The Cleanup page provides two options: cleaning up storage space and clearing cache, and their differences are as follows:&#x20;

* **Clear Storage Space:** Clearing storage space will completely delete all data of the application, similar to reinstalling the application. This includes files, databases, settings, caches, etc. in both the internal and external storage of the application. Clearing storage space is a cautious operation because it is very likely to result in some irreparable losses. For example, if WeChat clears its storage space, all chat records will be lost, and if no backup has been made, they can never be retrieved.

* **Clear Cache:** This option only clears the cached data of the application in the device storage, which are the files under the paths of getCacheDir and getExternalCacheDir. Clearing the cache does not have a significant impact on the user. Therefore, we can store less important data or data that can be redownloaded in the cache. This way, when disk space is insufficient, the user can achieve a significant cleaning effect through this option. If this cleaning effect is not satisfactory, the user is likely to continue clearing storage space or uninstalling the program.

Although the system provides a manual disk cleanup page, relying on users to manually clean up the disk is already too late, as it has already left a bad user experience by then. Moreover, not all users know how to clean up the disk space of programs. Therefore, we need to proactively perform disk cleanup when we detect that the disk space is too large.

Since the disk does not grow in a short period, for performance reasons, we can check the disk size in the program at a relatively long interval, such as once a day. If the disk usage is normal, no optimization is needed; if it is abnormal, cleanup work is required. The threshold for abnormality can be determined based on the percentage of the total disk size. Usually, when it exceeds one-tenth, cleanup is necessary.

There are two scenarios for disk anomalies. The first is simply that disk usage increases as the user uses it. In this case, you only need to delete the Cache directory using the delete method of the File object.&#x20;

The second type is abnormal situations, such as the disk growing too quickly, where disk cleanup was performed just the day before but the threshold was triggered again the next day, or the disk still occupying a large amount of space after the Cache directory was cleaned. In such cases, it is often necessary to report the size of each file in the directory, and then developers will conduct further investigations based on the specific abnormal files and directories, combined with logs and code logic.&#x20;

# 10.4 Degradation Optimization

Previously, we have learned about performance optimization in various aspects such as memory, speed, smoothness, and stability. However, sometimes, even after completing all the optimizations we can carry out, the program may still experience performance anomalies such as overheating during operation, continuous high CPU load, and frequent memory threshold breaches. This is due to the excessive and heavy business operations of the program, especially for medium to large-scale applications running on low-end devices. Simply optimizing in terms of performance alone is difficult to bring about a better user experience. In such cases, degradation is needed to achieve better results.Degradation generally comes in two forms. One way is to redevelop a lightweight program for the application, also known as the Lite version, where some capabilities and logic are trimmed in the Lite version, and resource consumption is also reduced, enabling smooth operation on low-end devices. The second way is to adopt dynamic degradation, that is, when the performance of the device is in an abnormal state, business degradation is used to ensure the normal operation of the program.&#x20;

Reducing the resolution during video playback, reducing the amount of data during business data retrieval and parsing, and turning off some non-core functions during operation are all common ways of business degradation. Since business degradation needs to be based on the characteristics of the business and involves degrading functions or logic, it does not have much universality. Therefore, the author here is not introducing how to perform business degradation, but rather explaining how to start from the overall perspective and design a degradation framework to help businesses better complete degradation operations. The design of the degradation framework mainly needs to consider the following three issues:

1. Collection of performance metrics and judgment of anomalies

2. Scheduling of Degraded Tasks&#x20;

3. Measurement of Degradation Effect

Next, we will address these three issues one by one and design a comprehensive degradation framework.

## 10.4.1 Performance Indicator Collection and Anomaly Judgment

Degradation is only required when the program encounters performance anomalies. Therefore, the primary task of the degradation framework is to collect performance metrics, and then, based on the collected metric data, it can further determine whether the program is in a state of performance anomaly.

Common performance metrics include CPU usage, temperature, memory, etc. These performance metrics are generally collected at a fixed frequency. For example, CPU usage can be collected once every 10 seconds, temperature once every 30 seconds, and Java memory once every 60 seconds. The collection frequency needs to consider the impact on performance and the sensitivity of the metrics. For instance, collecting CPU usage requires reading and parsing files under the "proc/stat" path, which incurs a certain performance overhead, so it cannot be collected too frequently;Since the temperature change is relatively slow, the sampling frequency can also be longer. There is no absolute value for the specific sampling frequency; it needs to be adjusted according to the characteristics of the program to achieve an optimal empirical value.

After collecting performance metrics, it is necessary to determine whether an anomaly has occurred. Anomalies can be divided into three stages: low, medium, and high. The higher the level, the more severe the current performance, and a greater degree of degradation needs to be triggered to alleviate it. The judgment of the anomaly threshold is also an empirical value. We can adjust it to a suitable value based on the characteristics of the program and the device model. Here, the author presents a set of threshold cases set for low-end devices during the actual project development process.&#x20;

<table><colgroup><col width="100"><col width="242"><col width="481"></colgroup>
<thead>
<tr>
<th>Type</th>
<th>Sampling Frequency</th>
<th>Abnormal Threshold</th>
</tr>
</thead>
<tbody>
<tr>
<td>CPU Usage</td>
<td>Check once every 10 seconds</td>
<td><ul>
<li>If the CPU usage remains above 50% for 60 seconds, it is considered that the CPU is operating under a severely high load, and urgent business degradation is required; otherwise, the program will become abnormally laggy </li>
<li>If the CPU usage remains above 50% for 30 seconds, it is considered that the CPU is operating under moderately high load, the business needs to perform a degradation operation, and the program is in a laggy state at this time </li>
<li>If the CPU usage remains above 50% for 10 seconds, it is considered that the CPU has started to operate under high load, and the business needs to pay attention and perform a slight degradation to improve the program experience </li>
<li>If the CPU remains below 30% for 30 consecutive seconds, it is considered that the CPU stress has been alleviated, the program starts to perform normally, and the business can appropriately turn off the degradation logic. </li>
</ul></td>
</tr>
<tr>
<td>Memory</td>
<td>Check once every 60 seconds</td>
<td><ul>
<li>When the remaining Java memory is below 30MB, it reaches a critical memory shortage state. At this time, a deep memory degradation must be carried out; otherwise, an OOM will occur. </li>
<li>When Java memory remaining is below 50MB, it reaches a medium memory shortage state, and medium memory degradation logic needs to be performed </li>
<li>When Java memory remaining is below 70MB, it reaches a mild memory shortage state, and business parties need to reduce memory usage </li>
<li>When the remaining Java memory is above 70MB, the memory returns to normal, and the business is notified to turn off the memory degradation logic </li>
</ul></td>
</tr>
<tr>
<td>Temperature </td>
<td>Check once every 30 seconds </td>
<td><ul>
<li>When the battery temperature exceeds 40°C, it reaches a severely abnormal temperature state, and in-depth temperature degradation is required </li>
<li>When the battery temperature is greater than 38°C and less than or equal to 40°C, it reaches a medium abnormal temperature state, triggering temperature degradation </li>
<li>When the battery temperature is greater than 36°C and less than or equal to 38°C, it reaches a slightly abnormal temperature state, and the business party needs to pay attention and reduce the logical operations that cause the device to heat up </li>
<li>If the temperature returns below 36°, the temperature is normal, notify the business to turn off the temperature degradation logic </li>
</ul></td>
</tr>
</tbody>
</table>

## 10.4.2 Addition and Scheduling of Degradation Tasks

When the degradation framework collects performance metrics and determines that the current situation is a performance anomaly scenario, the most common approach is to notify each business. After receiving the notification, the business then performs degradation. For example, the LowMemoryKiller mechanism of the system uses the notification method to prompt registered and monitored businesses to clean up memory. The process of this mechanism is shown in Figure 10-8.&#x20;

![Figure 10-8 The degradation framework triggers degradation through notification](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter10_img_8.png)

However, in actual development, simply sending the notification of triggering degradation to each business unit will not yield very good results, because business units may not respond to the degradation notification, or only a few businesses may respond, so the effect will not be optimal. To achieve the best degradation effect, we need to have business units add degradation logic to the degradation framework, and then have the degradation framework schedule and execute degradation tasks to ensure the best degradation effect. The process of the degradation framework at this time is shown in Figure 10-9.&#x20;

![Figure 10-9 The degradation framework triggers degradation through task scheduling](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter10_img_9.png)

The degradation framework designed based on this architecture needs to further consider how the business should add degradation tasks to the degradation framework, as well as how the degradation framework should schedule and execute tasks.&#x20;

* Add Task

The addition of tasks should be kept simple and easy to use, so as to reduce the cost of business calls. Therefore, generally, only a global addDowngradeTask method needs to be provided. When business parties add downgrade tasks, they need to include the business name, so that we can clearly know which businesses have added downgrade processing logic and which have not. For businesses that have not registered, we need to specifically encourage the business parties to register the downgrade logic. In addition to the business name, some other parameters, such as the downgrade scenario and custom thresholds, also need to be included for the downgrade framework to schedule tasks.&#x20;

* Scheduled Task&#x20;

For downgrade tasks registered, the downgrade framework needs to carefully consider the timing of scheduling and the scheduling strategy.&#x20;

1\) Scheduling timing: The timing of scheduling is basically when the device is in a performance anomaly scenario. Under different devices, the judgment thresholds or conditions for performance anomalies vary. For high-end models, the program may still run smoothly when the CPU usage is above 70%, but for low-end models, it starts to lag when the CPU usage is above 50%. Therefore, different levels of models need to set a reasonable threshold based on empirical values.

2\) Scheduling Strategy: When the program experiences performance anomalies, the degradation framework will execute the degradation logic corresponding to the scenario. For example, when the CPU usage reaches the degradation threshold, the degradation framework will start executing the degradation tasks registered in the CPU list. When executing the degradation tasks, it is not necessary to execute all the degradation logic in the queue; instead, it can be executed in batches. If, during the execution of a batch of degradation logic, the CPU usage drops below the threshold, the subsequent degradation tasks can be skipped. This way, from a global perspective, only partial business degradation can lead to an improvement in user experience.

## 10.4.3 Effect Metrics of the Degraded Framework

A well-developed framework not only needs to perform the functions of the framework excellently, but also requires indicators to measure the effectiveness and benefits of the framework. Therefore, to make the degradation framework more complete, the author here supplements the indicators that can be used to measure the effectiveness of the degradation framework.&#x20;

| **Indicator**                                 | **Description**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| --------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Overall key performance indicators            | After downgrading via the downgrading framework, if overall performance metrics such as smoothness, startup speed, frame rate, etc., have significantly improved, it indicates that the downgrading framework is effective; otherwise, it is ineffective. In this case, it is necessary to detect and troubleshoot whether the lack of obvious effect is caused by issues such as thresholds, callback strategies, etc., in the downgrading framework, or by unreasonable business downgrading logic.     |
| Business experience metrics                   | The job of the degradation framework is to degrade business operations. These operations generally have their own corresponding business metrics. For example, the business metrics of LIVE video operations include user viewing duration. If the degradation operation of the degradation framework improves the metrics of the business itself, it indicates that the degradation is valuable.                                                                                                         |
| Metrics for degradation logic                 | After each execution of a degradation task, the degradation framework requires certain metrics to measure the performance changes after degradation. If the execution of the degradation task does not improve the current performance of the program, it indicates that the effectiveness of the degradation logic is poor, and the business unit needs to be prompted to optimize and enhance the effectiveness of degradation.                                                                         |
| Evaluation indicators for degradation effects | We can also add the indicator of abnormal duration to measure the effectiveness of degradation, that is, when a device experiences performance anomalies, such as CPU overload, device overheating, insufficient memory, etc., how long these states will last. This duration can also reflect the duration of the quality of user experience. Through continuous optimization of the framework's scheduling strategy and the business's response strategy, this indicator can be continuously improved.  |

## 10.4.4 Solution Implementation

Previously, the concept and process of the degradation framework were introduced. Here, through the code of key steps, the implementation of the degradation framework solution will be explained. The author named the degradation framework DevicePerfManager and created the corresponding object. In the DevicePerfManager object, instances of monitoring objects such as CPU usage monitoring, memory monitoring, and temperature monitoring are held, and the degradation tasks are stored in an ArrayList container. The UML example diagram 10-10 is shown below.

![Figure 10-10 Downgrade Framework UML Diagram](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter10_img_10.png)

### 1. Register a downgrade task

Let's first see how the business party should register a downgrade task. As mentioned earlier, the registration method should be simple and easy to use, so simply providing a static method named addDowngradeTask is sufficient. The implementation of the method is as follows.

```java
public static addDowngradeTask(String key, 
                               PerfType perfType,
                               Runnable task){
    // After wrapping the input parameters into a DelegatePerfCallback, then put it into the container
    mDevicePerfCallbacks.add(new DelegatePerfCallback(devicePerfCallback,key,perfType));     
}
```

The explanations for the input parameters in the method are as follows:&#x20;

* key: The key value of the degradation logic, which can be used for deduplication and other logic

* PerfType: Monitors specified status types, including CPU, MEMORY, and TEMPERATURE

* task: Encapsulated degradation logic

The degradation tasks and input parameter configurations registered by the business party will be encapsulated into a DelegatePerfCallback object, which is then stored in the mDevicePerfCallbacks container. When a certain degradation condition is triggered, the degradation framework selects an appropriate Runnable task from the container for execution.&#x20;

### 2. Scheduling Timing and Strategy&#x20;

The author then takes the abnormal CPU load as an example to explain the implementation logic of CPU usage collection and abnormal triggering. The CPU usage collection can be encapsulated in the CpuUsageMonitor object and started in the initCpuUsageDetect method. The code implementation is as follows. When the CpuUsageMonitor detects the CPU usage, it then callbacks the data to the degradation framework DevicePerfManager through a callback mechanism.&#x20;

```c++
private void initCpuUsageDetect() {
    // Calculate CPU usage every 10 seconds
    mScheduleThreadPool.scheduleWithFixedDelay(new DynamicFixDelayRunnable() {
        @Override
        public void run() {
            float curCpuTime = getCpuTimesN();
            long curTime = System.currentTimeMillis();      
            long curTotalCpuStat = getTotalCPUTime();
            long curAppCpuStat = getAppCPUTime();
            if (mLastTotalCpuStat != 0 && mLastCpuTime != 0) {
                // Calculate CPU usage
                float usage = (curAppCpuStat - mLastAppCpuStat) / 
                        (float) (curTotalCpuStat - mLastTotalCpuStat) * 100f;
                for (ICpuPerfCallback cpuPerfCallback : mCpuPerfCallbacks) {
                    // Callback the CPU usage to the degradation framework
                    cpuPerfCallback.cpuUsage((int) usage);
                }
            }
            mLastTotalCpuStat = curTotalCpuStat;
            mLastAppCpuStat = curAppCpuStat;
        }
    }, 0, 10000, TimeUnit.MILLISECONDS);
}
```

In the downgrade framework DevicePerfManager, the CPU usage is obtained based on the callback from CpuUsageMonitor, and further determines whether to trigger a downgrade by combining with a threshold. The code is as follows. In the downgrade processing function cpuUsageDowngrade, we trigger the downgrade logic by executing the run method of the downgrade task Runnable. After executing a downgrade task, the CPU usage within the current 5 seconds is checked once to determine whether to continue the downgrade.&#x20;

```c++
mCpuSceneMonitor.addCpuPerfCallback(new ICpuPerfCallback(){
    @Override
    void cpuUsage(int usage){
        if(usage > CPU_USAGE_EXECPTION && !isCpuDowngrading){
            // Trigger downgrade
            cpuUsageDowngrade();
        }
    }
})

private void cpuUsageDowngrade() {
    // Execute the downgrade task in a child thread
    CoreCpuThreadPool.getThreadPool().execute(() -> {
        for (DelegatePerfCallback delegatePerfCallback : mDevicePerfCallbacks) {
            // Traverse and notify callbacks
            if (delegatePerfCallback.perfType == PerfType.CPU ) {
                isCpuDowngrading = true;
                // Execute the run method of the downgrade task
                delegatePerfCallback.task.run();                
                // After downgrade, check the CPU usage status within 5 seconds to determine whether to continue downgrading
                long beforeTotalCpuStat = getTotalCPUTime();
                long beforeAppCpuStat = getAppCPUTime();
                try {
                    Thread.sleep(5000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                // Calculate the current CPU status; if busy, continue downgrading, otherwise interrupt downgrade
                long curTotalCpuStat = getTotalCPUTimet();
                long curAppCpuStat = getAppCPUTime();
                int usage = (int) (((curAppCpuStat - beforeAppCpuStat) / 
                        (float) (curTotalCpuStat - beforeTotalCpuStat)) * 100);
                Log.i(TAG, "cpuUsage after downgrade" + usage);
                if (usage < mMinBusyCpuUsage) {
                    // If CPU usage returns to normal, interrupt downgrade
                    break;
                }
            }
        }
        isCpuDowngrading = false;
    });
}
```

Here, the author has only explained the implementation of one type of performance anomaly, namely high CPU load operation. Other scenarios such as temperature anomalies and memory anomalies are similar in mechanism and principle, so they will not be elaborated further. The case explained here is only a simplified version, mainly focusing on introducing the principle and thinking. The degradation framework in a real environment will be much more complex than this, because the thresholds for anomaly judgment are not the same for all business scenarios. Therefore, it supports business parties to input their own thresholds for anomaly judgment. When selecting degradation tasks, it does not degrade in order, but rather in the order of business priority and effectiveness. Moreover, degradation tasks are distinguished according to the severity level of the anomaly. Interested readers can further complete and improve this solution.&#x20;

| Source code appearing in this chapter:<br />power\_profile data definition:<https://source.android.com/devices/tech/power/values><br />BatteryStatsImpl.java：<br /><https://cs.android.com/android/platform/superproject/+/android-14.0.0_r9:frameworks/base/services/core/java/com/android/server/power/stats/BatteryStatsImpl.java><br />BatteryUsageStatsProvider<br /><https://cs.android.com/android/platform/superproject/+/android-14.0.0_r9:frameworks/base/services/core/java/com/android/server/power/stats/BatteryUsageStatsProvider.java><br />WifiPowerCalculator.java：<br /><https://cs.android.com/android/platform/superproject/+/android-14.0.0_r9:frameworks/base/services/core/java/com/android/server/power/stats/WifiPowerCalculator.java><br />battery-historian：<https://developer.android.com/topic/performance/power/setup-battery-historian> |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
