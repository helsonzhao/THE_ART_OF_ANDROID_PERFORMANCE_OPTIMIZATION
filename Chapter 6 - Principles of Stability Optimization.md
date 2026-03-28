Among all directions of performance optimization, stability optimization is the most important one, because even if a program is optimized well in other directions, but if it frequently becomes unresponsive or crashes during use, users will not tolerate it, and there is a high probability that they will uninstall the program or reduce its usage time. Therefore, stability is the cornerstone of a program.

To do a good job in stability optimization, one must first master the underlying knowledge and principles related to stability. The common types of stability issues mainly include ANR and Crash. Therefore, in this chapter, we will delve into the principles of the occurrence of these two types of problems, the underlying mechanisms, and other fundamental knowledge.&#x20;

# 6.2 ANR&#x20;

ANR (Application Not Responding) refers to the situation where an application fails to respond to user operations for an extended period. When this occurs, the system will display a pop-up window, allowing us to choose whether to forcefully close the program. As shown in Figure 6-1, it is a common ANR pop-up window.

<img src="https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter6_img_1.png" alt="Figure 6-1 ANR pop-up window" width="350" style="margin:auto;"/>

## 6.2.1 Types of ANR

There are four types of ANR defined in the Android system as follows:

* InputDispatching TimedOut : Triggered when the app fails to process input events such as touch and key events within 5 seconds

* BroadcastReceiver Timeout : Triggered when the onReceiver method fails to complete execution within 10 seconds for foreground broadcasts and 60 seconds for background broadcasts

* Service Timeout : Triggered when the foreground Service fails to start within 20 seconds or the background Service fails to start within 200 seconds

* ContentProvider Timeout : Triggered if the publication process is not completed within 10 seconds

Simply having a basic understanding of the type definitions of these several types of ANR is far from sufficient for ANR optimization. Instead, it is necessary to delve into the Android source code to understand the mechanisms and principles behind these ANRs. There are many articles on the internet that introduce the principles of these four types of ANR, but most of them are redundant and complex, making it easy for people to get lost in the ocean of code. Therefore,I will try to explain the mechanisms and principles behind these types of ANR in a concise and focused manner here.&#x20;

### 1. InputDispatching TimedOut

Before understanding the timeout of input event distribution, we need to first understand what input event distribution is. It refers to the process by which the Android system passes user operation events such as touch and key presses to components such as Activity and Fragment of the corresponding program. This process mainly takes place in the InputFlinger process and involves multiple steps such as event capture, transmission, processing, and response. The main member objects in this process are EventHub, InputReader, and InputDispatcher, and the detailed functions of these objects are as follows:&#x20;

* EventHub: EventHub receives raw input event data from the underlying input device driver, converts the raw event data into input event objects such as touch events (MotionEvent), key events (KeyEvent), etc., and passes them to the InputReader thread.

* InputReader: InputReader is a thread that continuously reads input events from the EventHub and further converts, processes, and classifies the input events based on the device type and input source. For example, it converts multi-touch events into gesture events, key events into character events, etc.

* InputDispatcher: InputDispatcher is also a thread, and from the name of this object, it can be understood that its main function is to dispatch events. After receiving the processed input events from InputReader, it will distribute the input events to the appropriate windows or programs according to the distribution strategy. When dispatching events, it will set a timeout period based on the type of the input event. If the consumption result from the corresponding window or program is not received within the timeout period, InputDispatcher will consider the window or application unresponsive and trigger "InputDispatching TimedOut".

The simplified process of these three objects handling event distribution is shown in Figure 6-2.&#x20;

![Figure 6-2 EventHub, InputReader, and InputDispatcher object event distribution process](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter6_img_2.png)



The process of event dispatching is very long, and we mainly focus on the triggering logic of ANR. During the dispatching process, there are multiple reasons that can trigger the "InputDispatching TimedOut" ANR, and these reasons can be classified into two categories: window not ready and window processing timeout, as follows:&#x20;

* Window not ready: If the window that InputDispatcher is about to dispatch to is not ready to receive new input events, such as when the window is paused, the window connection is dead, or the window connection is full, InputDispatcher will wait for the window to be ready. If the waiting time exceeds 5 seconds, an ANR will be triggered

* Window processing timeout: After InputDispatcher dispatches an input event to a window, if the window fails to return the event processing result to InputDispatcher within 5 seconds, InputDispatcher will consider the window processing to have timed out, triggering an ANR

In real-world scenarios, most input event distribution timeouts are caused by window processing timeouts, and the most common cause of window processing timeouts is the main thread's task processing timeout. Therefore, we will now explore the mechanism and process of ANR caused by window processing timeouts. This process mainly consists of the following four steps:&#x20;

1. For InputDispatcher, each window is maintained by a Connection object. During event dispatch, InputDispatcher first finds the correct window Connection object.&#x20;

2. After the InputDispatcher finds the corresponding Connection, it will distribute the event to the program window through Socket communication. The program window will receive the event through the InputChannel object. After the InputDispatcher finishes distributing the event, it will then put the event into the waitQueue.&#x20;

3. If the main thread of the program's window consumes this event, InputChannel will notify InputDispatcher via Socket communication to remove this event from the waitQueue&#x20;

4. When InputDispatcher performs the next event dispatch, it will determine whether there are any events in the waitQueue that have not been removed within five seconds. If so, it will consider an ANR to have occurred.&#x20;

This process is shown in Figure 6-3.&#x20;

![Figure 6-3 InputDispatcher event distribution process](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter6_img_3.png)



Next, we will delve deeper into the implementation of three key steps through the source code: how InputDispatcher dispatches events, how the waitqueue adds and removes events, and how InputDispatcher determines the occurrence of ANR based on events in the waitqueue.

**1) InputDispatcher dispatches events**

InputDispatcher is a continuously running thread that repeatedly executes the dispatchOnce method to distribute events. The code for this method is as follows

```c++
void InputDispatcher::dispatchOnce() {
    nsecs_t nextWakeupTime = LONG_LONG_MAX;
    { 
        std::scoped_lock _l(mLock);
        mDispatcherIsAlive.notify_all();
        // 1. Dispatch input event
        if (!haveCommandsLocked()) {
            dispatchOnceInnerLocked(&nextWakeupTime);
        }
        // 2. Handle commands
        if (runCommandsLockedInterruptable()) {
            nextWakeupTime = LLONG_MIN;
        }
        // 3. Handle ANR and return the next thread wake-up time
        const nsecs_t nextAnrCheck = processAnrsLocked();
        nextWakeupTime = std::min(nextWakeupTime, nextAnrCheck);

        if (nextWakeupTime == LONG_LONG_MAX) {
            mDispatcherEnteredIdle.notify_all();
        }
    } 
    nsecs_t currentTime = now();
    int timeoutMillis = toMillisecondTimeoutDelay(currentTime, nextWakeupTime);
    // 4. Thread sleep
    mLooper->pollOnce(timeoutMillis);
}
```

The main tasks of this method are as follows:&#x20;

1. Call the dispatchOnceInnerLocked method to dispatch events&#x20;

2. The runCommandsLockedInterruptable method is called to process input commands, such as orientation, volume adjustment, etc., which are all input commands. This method encapsulates the input commands and then distributes them to the target object for processing.&#x20;

3. Call processAnrsLocked to determine whether an ANR has occurred. This method will check the waitQueue to see if there are any input events that have not been processed for more than 5 seconds. If so, it will throw an ANR.

4. Based on performance considerations, the current thread is put to sleep for a certain period of time until the sleep ends or it is awakened by a new input event to be dispatched

dispatchOnceInnerLocked is the method for event dispatch. However, since the internal logic and paths of this method are very numerous, we will first use the Sequence Diagram, as shown in 6-4, to understand the main path of this method here.

![Figure 6-4 Event Distribution Sequencing Diagram](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter6_img_4.png)

In the Sequence Diagram, the method of sequence 3, findToucheWindowTargetsLocked, will find the target window based on the focus, and the method of sequence 7, startDispatchCyclelocked, will distribute the event to the InputChannel of the program through the corresponding Connection. We mainly look at the implementation of this method. The simplified code of this method is as follows.&#x20;

```c++
void InputDispatcher::startDispatchCycleLocked(nsecs_t currentTime, const sp<Connection>& connection) {
    // Traverse the Connection's outboundQueue until it is empty or the Connection status is abnormal
    while (connection->status == Connection::STATUS_NORMAL &&
            !connection->outboundQueue.isEmpty()) {
        // Retrieve the DispatchEntry object at the head of the queue
        DispatchEntry* dispatchEntry = connection->outboundQueue.head;
        // Set the dispatch time to the current time
        dispatchEntry->deliveryTime = currentTime;
        status_t status;
        // Get the corresponding EventEntry object
        EventEntry* eventEntry = dispatchEntry->eventEntry;
        // 1. Based on the event type, call the appropriate publish function to send the input event to the target window or application
        switch (eventEntry->type) {
            case EventEntry::TYPE_KEY: {
                status = connection->inputPublisher.publishKeyEvent(……);
                break;
            }
            case EventEntry::TYPE_MOTION: {
                status = connection->inputPublisher.publishMotionEvent(……);
                break;
            }
            default:
                return;
        }
        // 2. Check if the sending status is normal
        if (status) {
            if (status == WOULD_BLOCK) {
                 /* If the sending status is WOULD_BLOCK,
                    it indicates that the InputChannel buffer of the target window or application is full and cannot accept new input events.
                    In this case, the dispatch cycle is directly interrupted, triggering an ANR. */
               ……
            } else {
                /* If the sending status is another error, it indicates an exception occurred during the sending process.
                   The dispatch cycle is also directly interrupted, triggering an ANR. */
                abortBrokenDispatchCycleLocked(currentTime, connection, true);
            }
            return;
        }
        // 3. If the sending status is normal, remove the DispatchEntry object from the outboundQueue
        connection->outboundQueue.dequeue(dispatchEntry);
        // Add the DispatchEntry object to the tail of the waitQueue, awaiting feedback from the target window or application
        connection->waitQueue.enqueueAtTail(dispatchEntry);
        // Add the DispatchEntry object to mAnrTracker for tracking input event timeout situations
        mAnrTracker.insert(dispatchEntry);
    }
}
```

There are mainly three processes in this method, which are as follows:&#x20;

1. Select the event distribution function (publishMotionEvent or publishKeyEvent) corresponding to the target window (connection) based on the type of the input event (eventEntry->type) to perform event distribution

2. If event distribution fails, distribution will be interrupted and an ANR will be triggered. The ANR at this time is an ANR caused by the window not being ready&#x20;

3. Finally, place this input event into the waitQueue for subsequent ANR timeout judgment

By now, we have understood how InputDispatcher performs event dispatching. In the final step, InputDispatcher places events into the waitQueue. If an event in this queue remains unremoved for more than 5 seconds, it will trigger an ANR. So, let's continue to see how events in the waitQueue are removed.&#x20;

**2) InputDispatcher removes events from the waitQueue**

After the main thread of the target window has finished processing the input event, it will notify the InputDispatcher via Socket that the event has been consumed. The InputDispatcher will handle this process in the handleReceiveCallback function, and the simplified code of this function is as follows.

```c++
int InputDispatcher::handleReceiveCallback(int events, sp<IBinder> connectionToken) {
    std::scoped_lock _l(mLock);
    std::shared_ptr<Connection> connection = getConnectionLocked(connectionToken);
    ……
    if (!(events & (ALOOPER_EVENT_ERROR | ALOOPER_EVENT_HANGUP))) {
        ……
        for (;;) {
            //1. Receive event consumption feedback returned by the window
            Result<InputPublisher::ConsumerResponse> result =
                    connection->inputPublisher.receiveConsumerResponse();
            if (!result.ok()) {
                status = result.error().code();
                break;
            }

            if (std::holds_alternative<InputPublisher::Finished>(*result)) {
                const InputPublisher::Finished& finish =
                        std::get<InputPublisher::Finished>(*result);
                //2. Process the event consumption feedback returned by the window
                finishDispatchCycleLocked(currentTime, connection, finish.seq, 
                                            finish.handled,
                                            finish.consumeTime);
            } else if (std::holds_alternative<InputPublisher::Timeline>(*result)) {
               ……
            }
            gotOne = true;
        }
        if (gotOne) {
            //3. Execute command
            runCommandsLockedInterruptable();
            if (status == WOULD_BLOCK) {
                return 1;
            }
        }

    } else {
        ……
    }

    removeInputChannelLocked(connection->inputChannel->getConnectionToken(), notify);
    return 0;
}
```

The main processes in this method are explained as follows:&#x20;

1. First, receive the event consumption notification returned by the window through the receiveConsumerResponse function&#x20;

2. Then, in the finishDispatchCycleLocked method, the task of handling the received event is encapsulated into a Command&#x20;

3. Finally, call the runCommandsLockedInterruptable method to execute the above encapsulated processing event consumption Command task

Next, we'll mainly take a look at what's done in finishDispatchCycleLocked, and the code implementation of this function is as follows:

```c++
void InputDispatcher::finishDispatchCycleLocked(nsecs_t currentTime,
                                                const std::shared_ptr<Connection>& connection,
                                                uint32_t seq, bool handled, nsecs_t consumeTime) {

    if (connection->status == Connection::Status::BROKEN ||
        connection->status == Connection::Status::ZOMBIE) {
        return;
    }

    // Encapsulate the window callback processing task as a Command
    auto command = [this, currentTime, connection, seq, handled, consumeTime]() REQUIRES(mLock) {
        doDispatchCycleFinishedCommand(currentTime, connection, seq, handled, consumeTime);
    };
    // Add the Command to the queue
    postCommandLocked(std::move(command));
}

void InputDispatcher::doDispatchCycleFinishedCommand(nsecs_t finishTime,
                                                     const std::shared_ptr<Connection>& connection,
                                                     uint32_t seq, bool handled,
                                                     nsecs_t consumeTime) {
    ……

    // Loop through the waitQueue
    dispatchEntryIt = connection->findWaitQueueEntry(seq);
    if (dispatchEntryIt != connection->waitQueue.end()) {
        dispatchEntry = *dispatchEntryIt;
        // Remove the input event
        connection->waitQueue.erase(dispatchEntryIt);
        const sp<IBinder>& connectionToken = connection->inputChannel->getConnectionToken();
        mAnrTracker.erase(dispatchEntry->timeoutTime, connectionToken);
        ……
    }
    startDispatchCycleLocked(now(), connection);
}
```

It can be seen that the real event handling function is the doDispatchCycleFinishedCommand method, which traverses the waitQueue to find the corresponding event. Once found, the event will be removed from the waitQueue. This handling function is added to the Command queue, and the command will only be retrieved from the Command queue and executed when the runCommandsLockedInterruptable function is called.&#x20;



**3) InputDispatcher's determination of ANR**

Next, let's look at the last step in the loop-executing method dispatchOnce, which is to call the processAnrsLocked function to determine ANR. The code implementation of this function is as follows.

```c++
void InputDispatcher::processAnrsLocked(nsecs_t currentTime) {
    // Iterate through all Connection objects
    for (size_t i = 0; i < mConnectionsByFd.size(); i++) {
        const sp<Connection>& connection = mConnectionsByFd.valueAt(i);
        // Get the waitQueue
        Queue<DispatchEntry>* waitQueue = &connection->waitQueue;
        if (waitQueue->isEmpty()) {
            continue;
        }
        // Get the target application and window objects
        sp<InputApplicationHandle> applicationHandle = connection->inputApplicationHandle;
        sp<InputWindowHandle> windowHandle = connection->inputWindowHandle;
        // Iterate through the DispatchEntry objects in the waitQueue
        for (DispatchEntry* dispatchEntry = waitQueue->head; dispatchEntry; dispatchEntry = dispatchEntry->next) {
            // Get the EventEntry object
            EventEntry* eventEntry = dispatchEntry->eventEntry;
            // Get the timeout duration
            nsecs_t timeout = getDispatchingTimeoutLocked(applicationHandle, windowHandle);
            nsecs_t startTime = dispatchEntry->deliveryTime;
            nsecs_t waitTime = currentTime - startTime;
            // Check if it has timed out
            if (waitTime >= timeout) {
                // Call the onANRLocked function to trigger an ANR
                onANRLocked(currentTime, applicationHandle, windowHandle, eventEntry->eventTime, startTime, "input dispatching timed out");
                // Break out of the loop and continue with the next Connection object
                break;
            }
        }
    }
}
```

From the source code implementation, it can be seen that the processAnrsLocked function will iterate through the waitQueue of all window Connection objects, compare the timeout of the input event with the current time, and if the timeout has been exceeded, call the onANRLocked function to trigger an ANR. The code for the onAnrLocked function is as follows.

```c++
void InputDispatcher::onAnrLocked(const std::shared_ptr<Connection>& connection) {
    if (connection == nullptr) {
        LOG_ALWAYS_FATAL("Caller must check for nullness");
    }
    if (connection->waitQueue.empty()) {
        ALOGI("Not raising ANR because the connection %s has recovered",
              connection->inputChannel->getName().c_str());
        return;
    }

    DispatchEntry* oldestEntry = *connection->waitQueue.begin();
    const nsecs_t currentWait = now() - oldestEntry->deliveryTime;
    //Collect ANR Info
    std::string reason =
            android::base::StringPrintf("%s is not responding. Waited %" PRId64 "ms for %s",
                                        connection->inputChannel->getName().c_str(),
                                        ns2ms(currentWait),
                                        oldestEntry->eventEntry->getDescription().c_str());
    sp<IBinder> connectionToken = connection->inputChannel->getConnectionToken();
    //Send ANR Info to window
    updateLastAnrStateLocked(getWindowHandleLocked(connectionToken), reason);
    //Send to WindowManagerService
    processConnectionUnresponsiveLocked(*connection, std::move(reason));

    cancelEventsForAnrLocked(connection);
}
```

The onAnrLocked function will print out the ANR timeout information, send the ANR signal to the target window via Binder, and send the ANR to WindowManagerService for further processing through the processConnectionUnresponsiveLocked method.&#x20;

### 2.  BroadcastReceiver TimedOut

As one of the four major components of Android, broadcasts are used relatively frequently in actual scenarios. However, this article will not provide too much introduction to the broadcast component, but mainly focus on the relevant processes of ANR caused by broadcasts. First, we need to understand the types of broadcasts, which mainly fall into three categories, as follows:&#x20;

* Normal Broadcast: Normal broadcast is asynchronous. The sender does not care about the processing results of the receiver and will not wait for the receiver's response. The broadcast sent via the sendBroadcast method is a normal broadcast.

* Ordered Broadcast: Ordered broadcast is synchronous, where the sender needs to wait for the processing result of the receiver, and it can be sent via the sendOrderedBroadcast method.

* Sticky Broadcast: This type of broadcast is a special kind of ordinary broadcast that can remain in the system after being sent, allowing later-registered receivers to receive previously sent broadcasts. Sticky broadcasts can be sent via the sendStickyBroadcast function.

Among the three types of broadcasts mentioned above, only ordered broadcasts can lead to ANR, because this type of broadcast is synchronous. If the receiver takes more than 10 seconds to execute in the broadcast receiving function onReceive, the system will trigger an ANR of the "BroadcastReceiver Timeout" type. The ANR triggering process mainly consists of the following steps:

1\) ActivityManagerService starts the broadcast through the processNextBroadcast function

2\) During the startup process, if it is an ordered broadcast, a delayed task that triggers the ANR "BroadcastReceiver Timeout" will be started.

3\) Block the execution of the performReceiverLocked method, which triggers the Receiver callback onReceive function in the application. Only after the onReceive function has completed execution will the ActivityManagerService continue to execute the subsequent process and remove the previously started ANR-triggered delayed task.

From the process, we can see that if the delayed task triggered by ANR is not removed within the specified time, ANR will be triggered. The above process is shown in Figure 6-4. Next, we will further understand the triggering principle of this ANR through code implementation.

![Figure 6-4 Broadcast Receive Timeout Trigger Flow](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter6_img_5.png)



1. **Broadcast Start Entry**

The BroadcastQueueImpl member object in ActivityManagerService (AMS) is specifically designed to handle objects related to broadcasting. When AMS needs to start a broadcast, it will notify the Handler in BroadcastQueueImpl to trigger the processNextBroadcast method to start the broadcast. The code is as follows:&#x20;

```c++
// Handler in BroadcastQueueImpl
private final class BroadcastHandler extends Handler {
    public BroadcastHandler(Looper looper) {
        super(looper, null);
    }

    @Override
    public void handleMessage(Message msg) {
        switch (msg.what) {
            case BROADCAST_INTENT_MSG: {
                //1. Broadcast initiation entry point
                processNextBroadcast(true);
            } break;
            case BROADCAST_TIMEOUT_MSG: {
                //2. Broadcast timeout, which triggers a broadcast-type ANR
                synchronized (mService) {
                    broadcastTimeoutLocked(true);
                }
            } break;
        }
    }
}
```

This Handler only processes two tasks, one is the BROADCAST\_INTENT\_MSG message for broadcast startup, and the other is the BROADCAST\_TIMEOUT\_MSG message for triggering the ANR of broadcast timeout. Let's first take a look at the processNextBroadcast function, which is the entry function for broadcast startup. The code of this function is as follows:&#x20;

```c++
public void processNextBroadcastLocked(boolean fromMsg, boolean skipOomAdj) {
    // 1. Process parallel broadcasts
    while (mParallelBroadcasts.size() > 0) {
        ……
    }
    ……    
    boolean looped = false;
    //2. Process ordered broadcasts
    do {
        final long now = SystemClock.uptimeMillis();
        //3. Get the first broadcast from mOrderedBroadcasts
        r = mDispatcher.getNextBroadcastLocked(now);
        ……
        int numReceivers = (r.receivers != null) ? r.receivers.size() : 0;
        if (mService.mProcessesReady && !r.timeoutExempt && r.dispatchTime > 0) {
            if ((numReceivers > 0) &&
                    (now > r.dispatchTime + (2 * mConstants.TIMEOUT * numReceivers))) {
                //4. Start the broadcast timeout delay task
                broadcastTimeoutLocked(false); 
            }
        }
        ……
        if (r.receivers == null || r.nextReceiver >= numReceivers
                || r.resultAbort || forceReceive) {
            if (r.resultTo != null) {
                ……
                try {
                    //5. Synchronously call the receiver's onReceive method
                    performReceiveLocked(r, r.resultToApp, r.resultTo,
                            new Intent(r.intent), r.resultCode,
                            r.resultData, r.resultExtras, false, false, r.shareIdentity,
                            r.userId, r.callingUid, r.callingUid, r.callerPackage,
                            r.dispatchTime - r.enqueueTime,
                            now - r.dispatchTime, 0,
                            r.resultToApp != null
                                    ? r.resultToApp.mState.getCurProcState()
                                    : ActivityManager.PROCESS_STATE_UNKNOWN);
                } catch (RemoteException e) {
                    r.resultTo = null;
                }
                ……
            }
            //6. Cancel the ANR judgment task
            cancelBroadcastTimeoutLocked();
            ……
            continue;
        }
        ……
    } while (r == null);

    // Preprocess the receiver for the next broadcast
    ……
}
```

The main process in this method is described as follows:&#x20;

1\) First, loop through and process parallel broadcasts, which are ordinary broadcasts. When ordinary broadcasts are started, they are all first placed into the mParallelBroadcasts queue.

2\) Then start the loop to process the ordered broadcast

3\) When handling ordered broadcasts, the system first retrieves the ordered broadcast at the head of the queue from the mOrderdBroadcasts queue.

4\) Then call broadcastTimeoutLoced to start the delayed task for broadcast startup timeout.

5\) Then call the performReceiverLocked method, which in turn synchronously calls the receiver's onReceive method

6\) Finally, call the cancelBroadcastTimeoutLoced method to cancel the delayed task for ANR determination

Through this code, we can clearly understand the triggering mechanism of the ANR caused by broadcast startup timeout. If the broadcast receiver takes too long in the onReceive method, it will not have time to call the cancelBroadcastTimeoutLoced method to remove the delayed ANR task, so this ANR task will be triggered.&#x20;



* **ANR Trigger Task**

Next, let's look at the delayed task broadcastTimeoutLocked, which can trigger an ANR. The simplified code implementation of this function is as follows:

```c++
final void broadcastTimeoutLocked(boolean fromMsg) {
    ……
    try {
        long now = SystemClock.uptimeMillis();
        BroadcastRecord r = mDispatcher.getActiveBroadcastLocked();
        if (fromMsg) {
            ……
            long timeoutTime = r.receiverTime + mConstants.TIMEOUT;
            if (timeoutTime > now) {
                //Start the delayed task which trigger the ANR
                setBroadcastTimeoutLocked(timeoutTime);
                return;
            }
        }

        ……

    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
    }

}

final void setBroadcastTimeoutLocked(long timeoutTime) {
    if (! mPendingBroadcastTimeoutMessage) {
        Message msg = mHandler.obtainMessage(BROADCAST_TIMEOUT_MSG, this);
        mHandler.sendMessageAtTime(msg, timeoutTime);
        mPendingBroadcastTimeoutMessage = true;
    }
}
```

As can be seen from the code, the setBroadcastTimeoutLocked method sends an ANR trigger task with a delay of timeoutTime and a message type of BROADCAST\_TIMEOUT\_MSG to mHander. If this task is not removed within the timeoutTime period, the broadcastTimeoutLocked function corresponding to the BROADCAST\_TIMEOUT\_MSG message will be executed to trigger an ANR. The simplified code implementation of this method is as follows:&#x20;

```c++
final void broadcastTimeoutLocked(boolean fromMsg) {
    ……
    try {
        ……

        //  Trigger ANR by AMS
        if (!debugging && app != null) {
            mService.appNotResponding(app, timeoutRecord);
        }

    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
    }

}
```

In the above code, the method mService.appNotResponding will call the appNotResponding method of AMS to trigger the ANR of the target process.&#x20;

### 3. **Service TimeOut**

Previously, we have delved into the principle of broadcast reception timeout through code. ANRs related to service startup timeout and content provider publication timeout are largely similar in principle, so the author will only provide a brief introduction next. The triggering process of the ANR related to service startup timeout is shown in Figure 6-5.

![Figure 6-5 Service startup process](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter6_img_6.png)



All relevant logic of Service is encapsulated in the member object ActiveServices of AMS. We only need to understand the entry function realStartServiceLocked method for Service startup to understand the process triggered by "Service Timeout". The simplified code of this method is as follows:

```c++
private void realStartServiceLocked(ServiceRecord r, ProcessRecord app,
        IApplicationThread thread, int pid, UidRecord uidRecord, boolean execInFg,
        boolean enqueueOomAdj) throws RemoteException {
    ……
    //1. Send 'SERVICE_TIMEOUT_MSG' delayed task to handler，namely ANR trigger task
    bumpServiceExecutingLocked(r, execInFg, "create",
            OOM_ADJ_REASON_NONE);

    boolean created = false;
    try {
        ……
        //2. Synchronously execute the target service's onCreate method.
        thread.scheduleCreateService(r, r.serviceInfo,
                null
                app.mState.getReportedProcState());
        r.postNotification(false);
        created = true;
    } catch (DeadObjectException e) {
        ……
        throw e;
    } finally {
        if (!created) {
            ……
            //3. Remove 'SERVICE_TIMEOUT_MSG' delayed task
            serviceDoneExecutingLocked(r, inDestroying, inDestroying, false,
                    OOM_ADJ_REASON_STOP_SERVICE);
            ……
        }
    }

    ……
}
```

The explanation of the main process in this method is as follows:&#x20;

1\) First, call the bumpServiceExecutinLoced method, which sends a SERVICE\_TIMEOUT\_MSG message via the Handler, that is, the ANR trigger message. The delay time of this message depends on whether the service is a foreground or background service. The timeout for foreground services is 20 seconds, and the timeout for background services is 200 seconds.

2\) Then, it notifies the process where the target service resides through the Binder mechanism to execute the service creation or binding operation. The process where the target service resides will schedule the onCreate lifecycle of the service through the scheduleCreateService method of the ActivityThread class.

3\) Finally, after the service completes the creation or binding operation, the serviceDoneExecutingLocked method is executed, which removes the previously sent SERVICE\_TIMEOUT\_MSG message, indicating that the service has started or bound normally and will not trigger an ANR.

If the SERVICE\_TIMEOUT\_MSG message is not removed within the specified time, the serviceTimeout method will be executed to trigger ANR, and the triggering of ANR is also carried out by calling the appNotResponding method of AMS.&#x20;

### 4. ContentProvider TimeOut&#x20;

ContentProvider, also known as content provider, is created along with the first startup of the application. During the first startup of the application, AMS will execute the attachApplicationLocked method, which will call the bindApplication method of the application through binder. This method will trigger a series of processes for application startup, such as publishing the ContentProvider configured in the manifest and triggering the onAttach and other lifecycle methods of the Application. The simplified code of this method is as follows:

```c++
private void attachApplicationLocked(@NonNull IApplicationThread thread,
        int pid, int callingUid, long startSeq) {

    ……
    //get the provider list of the application
    List<ProviderInfo> providers = normalMode
                                        ? mCpHelper.generateApplicationProvidersLocked(app)
                                        : null;

    //1. Send 'CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG' task，which trigger ContentProvider Timeout
    if (providers != null && mCpHelper.checkAppInLaunchingProvidersLocked(app)) {
        Message msg = mHandler.obtainMessage(CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG);
        msg.obj = app;
        mHandler.sendMessageDelayed(msg,
                ContentResolver.CONTENT_PROVIDER_PUBLISH_TIMEOUT_MILLIS);

    ……

    //2. Execute bindApplication method through binder调
    thread.bindApplication(processName, appInfo,
            app.sdkSandboxClientAppVolumeUuid, app.sdkSandboxClientAppPackage,
            providerList,
            instr2.mClass,
            profilerInfo, instr2.mArguments,
            instr2.mWatcher,
            instr2.mUiAutomationConnection, testMode,
            mBinderTransactionTrackingEnabled, enableTrackAllocation,
            isRestrictedBackupMode || !normalMode, app.isPersistent(),
            new Configuration(app.getWindowProcessController().getConfiguration()),
            app.getCompat(), getCommonServicesLocked(app.isolated),
            mCoreSettingsObserver.getCoreSettingsLocked(),
            buildSerial, autofillOptions, contentCaptureOptions,
            app.getDisabledCompatChanges(), serializedSystemFontMap,
            app.getStartElapsedTime(), app.getStartUptime());
    ……
}
```

From the above code, we can know that before calling the bindApplication method of the target program, AMS will first send a delayed task of CONTENT\_PROVIDER\_PUBLISH\_TIMEOUT to the Handler. This task will trigger an ANR for content provider publishing timeout. After the target program publishes the ContentProvider in the bindApplication method, it will notify AMS to remove this task message. From this, we can also find that the publishing of the content provider occurs during the program startup process, so when the content provider publishing takes too long and triggers an ANR, it will directly affect the normal startup of the program.

## 6.2.2 Common ANR Attributions

From the generation mechanisms of these four types of ANRs, the fundamental reason is that the main thread fails to complete task execution within the specified time. However, there are numerous factors that can cause the main thread to fail to complete tasks on time, and when summarized, they can all be attributed to these three categories:&#x20;

1. Main thread method takes a long time

A method in the main thread takes an extremely long time to execute and exceeds the threshold for ANR triggering, such as an IO task in the main thread or waiting for a lock. For this type of issue, once the specific abnormal method logic is identified, it can be easily fixed by using asynchronous processing, refining method granularity, etc.&#x20;

* Main thread message backlog

This type of problem is relatively difficult to locate because we will find that the tasks in the main thread all take a short time to execute, but ANR still occurs. The reason is that there is excessive message accumulation in the main thread. For example, if each task takes 100 ms to execute, but if there are 50 such tasks in the main thread message queue, then it will take 5 seconds to process all these messages, which will cause input events to fail to be responded to within the specified time and trigger ANR. The modified process is shown in Figure 6-6. For this type of problem, we often need to find the business and code logic that frequently send tasks to the main thread, and then optimize them specifically. In some more complex scenarios, sometimes it is not a single business that is sending a large number of tasks to the main thread, but most businesses are doing so. In this case, better architectural design is needed to avoid congestion in the main thread.&#x20;

![Figure 6-6 Main thread message stacking](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter6_img_7.png)



* Performance issues

When the program experiences performance anomalies such as high CPU usage, high memory usage, and frequent GC, the main thread may fail to obtain CPU time slices, leading to a degradation in task execution speed. At this time, ANR is also likely to occur. Therefore, when ANR is caused by performance issues, we need to address ANR through performance optimization in areas such as CPU and memory.&#x20;

# 6.2 Crash&#x20;

Crash refers to the situation where a program crashes due to a serious error. For Android programs, the crash rate generally needs to be at least less than 0.05%, so as to ensure a relatively good user experience. For programs with excellent stability optimization, the crash rate can even be less than 0.01%.

## 6.2.1 Java Crash

Crash mainly comes from two directions: the Java layer and the Native layer. We will start learning from the basics of Java Crash, proceeding from the easier to the more difficult.

### 1. Common exceptions

Most Java crashes are relatively easy to fix. As long as there are logs of the crash occurrence, attribution and location can be carried out based on the exception type recorded in the log information. Some common types of Java crashes are as follows.

* NullPointerException: An exception caused by accessing the properties or methods of a null object, which is prone to occur frequently in scenarios such as thread concurrency, abnormal timing calls, and overly long object passing chains

* IllegalStateException: Usually an exception caused by performing certain operations at an inappropriate time or in an inappropriate scenario. For example, calling findViewById after the onCreate lifecycle of an Activity, calling startActivity after the onPause lifecycle, updating UI elements on a non-UI thread, performing a query on a closed database connection, etc.

* IndexOutOfBoundsException: An exception that occurs when accessing a collection such as an array, list, or string, where the index exceeds the valid range. It commonly occurs in some concurrent scenarios, for example, when one thread has already removed an element from the collection, but another thread is still accessing it according to the original sequence.

* IllegalArgumentException: When a program uses incorrect parameters to call a function, such as passing a negative index or an empty string, etc.

* OutOfMemoryError: This exception occurs when the memory used by a program exceeds the available memory in the system.&#x20;

When fixing Java crashes, one should try to find the root cause of the exception, rather than simply fixing it through methods such as exception catching. This simple approach to fixing can easily introduce more difficult-to-diagnose exceptions into the program. For example, for the most common null pointer exception, one needs to find all the code logic that sets data to null, then determine whether the null pointer exception is caused by timing issues, multi-threading synchronization issues, or improper usage, and then perform targeted fixes on the code that sets data to null based on the cause.&#x20;

### 2. Exception Propagation and Capture

When an exception occurs in a certain code logic of a program, an exception will first be thrown. If there is a try-catch statement in our code logic to catch this exception, the program will execute subsequent code normally. If there is no try-catch statement, or if the catch block does not match the type of this exception, then the exception will be thrown up along the call stack until it is caught. If the exception is not caught even by the topmost method in the call stack, it will be handed over to the system's default exception handler (UncaughtExceptionHandler), which will output logs through System.err and then forcefully shut down the program.&#x20;

In actual development, we usually customize an exception handler. In the customized exception handler, we often obtain log information for locating exceptions and upload it to the server. Exceptions that occur in child threads and do not affect the program's performance can also be caught here, thus preventing the program from exiting. The implementation code for configuring a customized exception handler is as follows:&#x20;

```java
public class MyApplication extends Application {
  @Override
  public void onCreate() {
    super.onCreate();
    Thread.setDefaultUncaughtExceptionHandler(new MyCrashHandler());
  }
}
```

We need to set a custom exception handler for the main thread as early as possible during the startup of the program, which in the above code is in the onCreate lifecycle method of the Application, to catch and handle uncaught exceptions, as well as report exceptions. The custom exception handler needs to inherit from the system's UncaughtExceptionHandler class, and the code implementation is as follows:&#x20;

```java
//Custom UncaughtExceptionHandler
public class MyCrashHandler implements Thread.UncaughtExceptionHandler{

    @Override
    public void uncaughtException(Thread thread, Throwable ex) {
        //Save Crash File
        saveCrashInfo(ex);
        //Judge if catch exception
        if(catchError(thread,ex)){
            return;
        }
        //KillProcess for those unCatched exceptions
        android.os.Process.killProcess(android.os.Process.myPid());
        System.exit(1);
    }
}

```

In the implementation code of exception capture, generally only the exception is saved locally, and then uploaded to the server when the program starts next time. This is because the processing time after an exception occurs is relatively short, and the time it takes to write data locally is much shorter than uploading it to the server. Therefore, this can improve the capture rate of exception logs.&#x20;

In the function that catches exceptions in catchError, all exceptions from non-main threads can be caught, so that the program will not be terminated. However, we still need to report all exceptions; otherwise, it will affect the discovery and repair of exceptions. Although this approach is likely to lead to abnormal behavior during program use, it is still better than the program exiting. However, this approach cannot solve all problems, so we still need to continue adding some strategies to ensure that the program is affected to the minimum extent. Here, I introduce a strategy, and the process is as follows:&#x20;

1. Catch all exceptions. If no exceptions occur within a short period, stop further processing and only report them.&#x20;

2. If a Crash occurs repeatedly within a short period, determine whether the Activity can be forcefully closed&#x20;

3. If the Activity cannot be forcefully closed or the exception cannot be resolved after closing, you can then clear the local cache and database&#x20;

4. If a crash still occurs repeatedly after clearing the local cache and database, a pop-up window will appear to remind the user to upgrade or change the program version&#x20;

The flowchart of this strategy is shown in Figure 6-7. Readers can design appropriate Crash fallback strategies based on actual scenarios and business characteristics.&#x20;

![Figure 6-7 Crash fallback strategy](https://raw.githubusercontent.com/helsonzhao/THE_ART_OF_ANDROID_PERFORMANCE_OPTIMIZATION/main/assets/chapter6_img_8.png)



### 3. OOM

OOM is a special type of Java Crash. For most crashes, the problem can be basically located through the stack where the crash occurred. However, it is almost impossible to locate the problem of OOM through the stack; instead, it can only be located through memory snapshots. Moreover, the management of most OOMs needs to be addressed through memory optimization, and only a small portion of OOMs are caused by abnormal code logic, such as infinite loop logic and abnormal data loading logic. This part of the abnormality needs to be analyzed and located through memory snapshot files, namely Hprof files.

To fix the online OOM exception, we rely on the Hprof file. Therefore, we usually capture the memory stack when an OOM occurs. The code implementation is as follows: directly calling the Debug.dumpHprofData interface provided by the system can obtain a memory snapshot.&#x20;

```java
private class MyCrashHandler implements Thread.UncaughtExceptionHandler {
    @Override
    public void uncaughtException(Thread thread, Throwable ex) {
      // Judge is OOM 
      if (ex instanceof OutOfMemoryError) {
        // dump hprof
        File hprofFile = new File(getFilesDir(), "dump.hprof");
        Debug.dumpHprofData(hprofFile.getAbsolutePath());
      }
    }
}
```

## 6.2.2 Native Crash

The governance of Native Crash is much more complex than that of Java Crash. Here, we will first learn some basic knowledge about Native Crash, and we will conduct more in-depth study in the subsequent practical chapters.&#x20;

### 1. Common Signals

All Java crashes have clear exception types, which can effectively help us focus on and locate issues. For Native crashes, there is also a similar mechanism: signals, which can help us clearly identify the type of Native crash.

A signal is a mechanism used by the operating system to notify a process that certain exceptional events have occurred. Each signal represents a specific event, and there are a total of 31 signals in Android. The following introduces the commonly used signals.&#x20;

<table>
<thead>
<tr>
<th>signal</th>
<th>Meaning </th>
<th>Explanation</th>
</tr>
</thead>
<tbody>
<tr>
<td>4 (SIGILL)</td>
<td>Illegal Instruction</td>
<td><ul>
<li>ILL_ILLOPC (Error Code 1): Illegal opcode, the application attempted to execute an invalid opcode.</li>
<li>ILL_ILLOPN (Error Code 2): Illegal operand, the application attempted to execute an invalid operand.</li>
</ul></td>
</tr>
<tr>
<td>6 (SIGABRT)</td>
<td>Voluntary Termination</td>
<td><ul>
<li>ABRT_NOOP (error code 0): The abort function was actively called in the Native code</li>
<li>ABRT_LOW_MEMORY (Error Code 1): Out of memory, the application voluntarily terminated due to insufficient memory.</li>
</ul></td>
</tr>
<tr>
<td>7 (SIGBUS)</td>
<td>Bus Error</td>
<td><ul>
<li>BUS_ADRALN (Error Code 1): Address alignment error, the application attempted to access an unaligned memory address.</li>
<li>BUS_ADRERR (Error Code 2): Illegal address error, the application attempted to access an invalid physical memory address.</li>
</ul></td>
</tr>
<tr>
<td>8 (SIGFPE)</td>
<td>Floating point exception </td>
<td><ul>
<li>FPE_INTDIV (Error Code 1): Division by zero, the application attempted to perform integer division and divided by zero.</li>
<li>FPE_INTOVF (Error Code 2): Integer overflow, the application attempted an integer operation and caused the result to overflow.</li>
</ul></td>
</tr>
<tr>
<td>9 (SIGKILL)</td>
<td>Kill Signal </td>
<td>A special signal, it is usually used to forcefully terminate a process. Unlike other signals, SIGKILL cannot be caught or ignored, and it will immediately terminate the target process. Since SIGKILL is a fixed signal, it has no specific error code </td>
</tr>
<tr>
<td>11 (SIGSEGV)</td>
<td>Segmentation fault</td>
<td><ol>
<li>SEGV_MAPERR: Indicates an error in accessing a memory mapping. The application attempted to access a memory region that is not mapped to its address space.</li>
<li>SEGV_ACCERR: Indicates an access permission error. The application attempted to access a memory area for which it does not have read or write permissions.</li>
</ol></td>
</tr>
<tr>
<td>13 (SIGPIPE)</td>
<td>Pipeline rupture </td>
<td>There is no specific error code. When a process receives the SIGPIPE signal, it usually means that it is attempting to write data to a closed pipe</td>
</tr>
<tr>
<td>16 (SIGSTKFLT)</td>
<td>Coprocessor stack error </td>
<td>Indicates a floating-point stack error, usually caused by the use of an invalid floating-point Instruction Set Architecture</td>
</tr>
<tr>
<td>15 (SIGTERM)</td>
<td>Terminate Process</td>
<td>Signal requesting normal termination of the process</td>
</tr>
<tr>
<td>19 (SIGSTOP)</td>
<td>Stop the process</td>
<td>A signal requesting the process to stop running, similar to the operation of pausing a process </td>
</tr>
</tbody>
</table>

### 2. Signal Transmission and Capture

Signals are a form of Inter-Process Communication in the Linux system, so we can also actively send signals through code. There are many signal sending functions, and the common methods are as follows:&#x20;

* kill function: used to send signals to a process or process group

* sigqueue function: It can only send signals to a single process and cannot send signals to a process group, but it can also carry an additional integer value and an additional data pointer to provide more information to the target process

* alarm function: used to set a timer, and when the timer times out, it sends a SIGALRM signal to the current process

* abort function: Used to abnormally terminate the current process. It sends a SIGABRT signal to the current process, causing the process to terminate immediately

* raise function: Used to send a specified signal to the current process. It can be used to trigger the handler function for a specific signal or to simulate the situation where another process sends a signal

When the code logic in the Native layer encounters an exception, the operating system in Kernel Mode will detect the exception and send a signal to the abnormal process through the above-mentioned function. We can capture these exception signals in the Native layer using the signal function or sigaction function provided by the Linux system. Although both functions can capture signals, the sigaction function offers higher flexibility, so in actual projects, signal capture and exception monitoring for Native Crash are generally implemented through this function. The sigaction function is as follows:&#x20;

```c++
#include <signal.h> 
int sigaction(int signum,const struct sigaction *act,struct sigaction *oldact));

struct sigaction {
    void (*sa_handler)(int); 
    void (*sa_sigaction)(int, siginfo_t *, void *); 
    sigset_t sa_mask; 
    int sa_flags; 
    void (*sa_restorer)(void);
};
```

Below is an explanation of the input parameters of this function:

* signum: The signal to be captured

* act: A pointer to an instance of the sigaction structure, and the explanation of the structure parameters is as follows

  * sa\_handler: A function pointer that points to the function used to handle signals. When the corresponding signal is received, the system will call this function for processing.

  * sa\_sigaction: A function pointer that specifies the function used to handle signals. Compared to sa\_handler, it can receive more parameters, including the signal number, signal additional information, and context information. When sa\_handler is not null, sa\_sigaction will be ignored.

  * sa\_mask: Specifies which signals should be blocked during the execution of the signal handling function

  * sa\_flags: Used to specify the representation options for signal handling. Common flags include SA\_RESTART, which indicates automatically restarting system calls interrupted by signals when the signal handling function returns; SA\_NOCLDSTOP, which indicates ignoring signals for child process stop or termination; and SA\_SIGINFO, which indicates using sa\_sigaction instead of sa\_handler as the signal handling function.

  * sa\_restorer: Deprecated, no need to set.

* oldact: Points to a structure of type sigaction, used to store information about the previous signal handling function and options. The sigaction function registering a signal will overwrite the original signal registration function. If you need to maintain the integrity of the original signal handling function, you can use the oldact parameter to save the original signal handling method and call it at an appropriate time.



# 6.3 Stability Optimization Methodology

When many developers optimize stability, they usually treat it as a bug and fix it after an exception occurs. Fixing just one bug is the simplest task in stability optimization, but this approach cannot effectively improve the stability of an application because the problem has actually already occurred. For stability optimization, we need a more comprehensive and systematic solution, that is, to proceed from three directions: monitoring, analysis and governance, and prevention of degradation, in order to ensure that the stability of the application can always be maintained at a good level.&#x20;

## 6.5.1 Monitoring

To ensure the stability of an application, it is necessary to promptly detect program anomalies throughout the entire process of user interaction with the program. Therefore, monitoring program anomalies in the online environment is the most crucial step in stability optimization.&#x20;

Monitoring should at least include these two capabilities: one is the ability to promptly detect anomalies, and the other is the ability to collect anomaly information when an anomaly occurs. Whether it is a Java Crash, Native Crash, or other anomalies such as ANR, OOM, etc., when designing a monitoring solution, both functions are required. The challenges of these two functions lie in how to improve the anomaly capture rate and how to collect sufficient information for analyzing and locating anomalies. These challenges will be explained in detail in the subsequent practical chapters.

Although there are currently many components specifically designed for stability monitoring on the market, such as Tencent's Bugly, in actual project scenarios, to improve efficiency and reduce reinventing the wheel, we can directly use the monitoring capabilities provided by these third parties. However, we still need to understand the principles and implementation plans of various stability monitoring methods, so that we can optimize and improve these third-party tools to make them more suitable for actual business scenarios.&#x20;

## 6.5.2 Analysis and Governance

In the analysis and governance phase, the most important aspect is analysis, which is often the most time-consuming and complex step. For ANR, it is necessary to analyze information such as the main thread logic, the status of each thread, and performance; for Java or Native crashes, it is necessary to analyze information such as the stack, key logs, and exception types; for OOM, it is necessary to analyze information such as memory snapshots.&#x20;

During anomaly analysis, we may often fail to effectively analyze anomalies due to insufficient information captured by monitoring. In such cases, we need to continuously improve monitoring and log collection capabilities until sufficient information is available. There are also many instances where, even with sufficient information, we may still be unable to effectively analyze anomalies due to hidden causes in the system or program. At this time, there is no need to be discouraged. If the impact of the anomaly is not particularly severe, we can rely on speculation and then verify it through multiple versions of iteration until the anomaly is located and fixed. If we really cannot analyze the problem, we can also change our approach, such as bypassing the anomaly by using a different implementation method.&#x20;

Once an anomaly is analyzed, remediation becomes a straightforward task. Simply modify the code where the anomaly occurs directly. If the anomaly occurs in a second-party or third-party library whose source code we cannot directly modify, we can use Hook technology to make the modification.&#x20;

## 6.5.3 Anti-deterioration

Anti-deterioration is also crucial for stability optimization. Before a program is released, if it follows the proper process, a long-term Monkey test will be conducted to detect the program's stability, and testers will also manually perform a large number of tests to ensure the program's stability. These are all offline anti-deterioration methods. In addition to these common offline solutions, there are still many things we can do, such as anti-deterioration solutions at the code specification level, including establishing a sound code review and code merging mechanism, promoting the use of Kotlin to replace Java to reduce null pointer exceptions, etc.; online anti-deterioration solutions, such as a comprehensive Crash fallback strategy, slow function detection, and other solutions.

If we fail to prevent degradation, even if we have made great efforts to improve stability, it is very likely that the stability indicators will experience a significant decline after several versions of iteration. Therefore, doing a good job in preventing degradation can help us consolidate the achievements in stability optimization more sustainably and reduce the time invested in subsequent stability optimization.&#x20;

| EventHub.cpp: https://cs.android.com/android/platform/superproject/+/android14-release:frameworks/native/services/inputflinger/reader/EventHub.cpp<br />InputDispatcher.cpp: https://cs.android.com/android/platform/superproject/+/android14-release:frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp<br />ActivityManagerService: https://cs.android.com/android/platform/superproject/+/android14-release:frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java<br />BroadcastQueueImpl: https://cs.android.com/android/platform/superproject/+/android14-release:frameworks/base/services/core/java/com/android/server/am/BroadcastQueueImpl.java<br />ActiveServices: https://cs.android.com/android/platform/superproject/+/android14-release:frameworks/base/services/core/java/com/android/server/am/ActiveServices.java |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

