---
layout: post
title:  "Input系统—InputReader线程"
date:   2016-12-11 22:19:12
catalog:  true
tags:
    - android

---

> 基于Android 6.0源码， 分析InputManagerService的启动过程

    frameworks/base/services/core/java/com/android/server/input/InputManagerService.java
    frameworks/base/services/core/java/com/android/server/wm/InputMonitor.java
    frameworks/base/services/core/jni/com_android_server_input_InputManagerService.cpp
    
    frameworks/base/core/java/android/view/InputChannel.java
    frameworks/native/libs/input/KeyCharacterMap.cpp
      
      
## 一. InputReader起点

上一篇文章[Input系统—启动篇](http://gityuan.com/2016/12/10/input-manager/)，介绍IMS服务的启动过程会创建两个native线程，分别是InputReader,InputDispatcher，并且是可以调用Java代码的native线程。

#### 1.1 InputReader执行流
整体调用链：

    InputReaderThread.threadLoop
      InputReader.loopOnce
        EventHub.getEvents
        InputReader.processEventsLocked
        QueuedListener.flush

先来回顾一下InputReader对象的初始化过程:

    InputReader::InputReader(const sp<EventHubInterface>& eventHub,
            const sp<InputReaderPolicyInterface>& policy,
            const sp<InputListenerInterface>& listener) :
            mContext(this), mEventHub(eventHub), mPolicy(policy),
            mGlobalMetaState(0), mGeneration(1),
            mDisableVirtualKeysTimeout(LLONG_MIN), mNextTimeout(LLONG_MAX),
            mConfigurationChangesToRefresh(0) {
        //此处mQueuedListener的成员变量`mInnerListener`便是InputDispatcher对象
        mQueuedListener = new QueuedInputListener(listener);
        {
            AutoMutex _l(mLock);
            refreshConfigurationLocked(0);
            updateGlobalMetaStateLocked();
        } 
    }


接下来介绍InputReader线程的执行过程，从threadLoop为起点开始分析。

#### 1.2 threadLoop
[-> InputReader.cpp]

    bool InputReaderThread::threadLoop() {
        mReader->loopOnce(); //【见小节1.3】
        return true;
    }

threadLoop返回值true代表的是会不断地循环调用loopOnce()。另外，如果当返回值为false则会
退出循环。整个过程是不断循环的地调用InputReader的loopOnce()方法，先来回顾一下InputReader对象构造方法。


#### 1.3 loopOnce
[-> InputReader.cpp]

    void InputReader::loopOnce() {
        ...
        {
            AutoMutex _l(mLock);
            uint32_t changes = mConfigurationChangesToRefresh;
            if (changes) {
                timeoutMillis = 0;
                ...
            } else if (mNextTimeout != LLONG_MAX) {
                nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
                timeoutMillis = toMillisecondTimeoutDelay(now, mNextTimeout);
            }
        }

        //从EventHub读取事件，其中EVENT_BUFFER_SIZE = 256【见小节2.1】
        size_t count = mEventHub->getEvents(timeoutMillis, mEventBuffer, EVENT_BUFFER_SIZE);
        
        { // acquire lock
            AutoMutex _l(mLock);
             mReaderIsAliveCondition.broadcast();
            if (count) { //处理事件【见小节3.1】
                processEventsLocked(mEventBuffer, count);
            }
            if (oldGeneration != mGeneration) {
                inputDevicesChanged = true;
                getInputDevicesLocked(inputDevices);
            }
            ...
        } // release lock

        
        if (inputDevicesChanged) { //输入设备发生改变
            mPolicy->notifyInputDevicesChanged(inputDevices);
        }
        //发送事件到nputDispatcher【见小节4.1】
        mQueuedListener->flush();
    }

该方法主要功能：

1. 从EventHub读取事件; [见小节2.1]getEvents()
2. 对事件进行加工； [见小节3.1]processEventsLocked()
2. 将发送事件到InputDispatcher线程； [见小节4.1] QueuedListener->flush()

另外，整个过程还会检测配置是否改变，输出设备是否改变，如果改变则调用policy来通知。
    
## 二. EventHub

#### 2.1 getEvents
[-> EventHub.cpp]

    size_t EventHub::getEvents(int timeoutMillis, RawEvent* buffer, size_t bufferSize) {
        AutoMutex _l(mLock); //加锁

        struct input_event readBuffer[bufferSize];
        RawEvent* event = buffer; //原始事件
        size_t capacity = bufferSize; //容量大小为256
        bool awoken = false;
        for (;;) {
            nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
            ...
            while (mClosingDevices) {
                ...
            }

            if (mNeedToScanDevices) {
                mNeedToScanDevices = false;
                //扫描设备【见小节2.2】
                scanDevicesLocked();
                mNeedToSendFinishedDeviceScan = true;
            }

            while (mOpeningDevices != NULL) {
                Device* device = mOpeningDevices;
                mOpeningDevices = device->next;
                event->when = now;
                event->deviceId = device->id == mBuiltInKeyboardId ? 0 : device->id;
                event->type = DEVICE_ADDED; //添加设备的事件
                event += 1;
                mNeedToSendFinishedDeviceScan = true;
                if (--capacity == 0) {
                    break;
                }
            }
            ...

            bool deviceChanged = false;
            while (mPendingEventIndex < mPendingEventCount) {
                const struct epoll_event& eventItem = mPendingEventItems[mPendingEventIndex++];
                if (eventItem.data.u32 == EPOLL_ID_INOTIFY) {
                    ...
                    continue;
                }
                if (eventItem.data.u32 == EPOLL_ID_WAKE) {
                    ...
                    continue;
                }
                //获取设备ID所对应的device
                ssize_t deviceIndex = mDevices.indexOfKey(eventItem.data.u32);
                Device* device = mDevices.valueAt(deviceIndex);
                if (eventItem.events & EPOLLIN) {
                    //从设备不断读取事件，放入到readBuffer
                    int32_t readSize = read(device->fd, readBuffer,
                            sizeof(struct input_event) * capacity);
                    if (readSize == 0 || (readSize < 0 && errno == ENODEV)) {
                        deviceChanged = true; 
                        closeDeviceLocked(device);//设备已被移除则执行关闭操作
                    } else if (readSize < 0) {
                        ...
                    } else if ((readSize % sizeof(struct input_event)) != 0) {
                        ...
                    } else {
                        int32_t deviceId = device->id == mBuiltInKeyboardId ? 0 : device->id;
                        size_t count = size_t(readSize) / sizeof(struct input_event);
                        
                        for (size_t i = 0; i < count; i++) {
                            struct input_event& iev = readBuffer[i];
                            ...
                            event->when = nsecs_t(iev.time.tv_sec) * 1000000000LL
                                    + nsecs_t(iev.time.tv_usec) * 1000LL;
                            event->deviceId = deviceId;
                            event->type = iev.type;
                            event->code = iev.code;
                            event->value = iev.value;
                            event += 1;
                            capacity -= 1;
                        }
                        if (capacity == 0) {
                            mPendingEventIndex -= 1;
                            break;
                        }
                    }
                }
                ...
            }
            ...

            //立刻报告设备增加或移除事件
            if (deviceChanged) {
                continue;
            }

            if (event != buffer || awoken) {
                break;
            }
            mPendingEventIndex = 0;

            mLock.unlock(); //poll之前先释放锁
            release_wake_lock(WAKE_LOCK_ID);
            //等待input事件的到来
            int pollResult = epoll_wait(mEpollFd, mPendingEventItems, EPOLL_MAX_EVENTS, timeoutMillis);

            acquire_wake_lock(PARTIAL_WAKE_LOCK, WAKE_LOCK_ID);
            mLock.lock(); //poll之后再次请求锁

            
            if (pollResult < 0) {
                //出现错误
                mPendingEventCount = 0;
                if (errno != EINTR) {
                    usleep(100000); //系统发生错误，则休眠1s
                }
            } else {
                mPendingEventCount = size_t(pollResult);
            }
        }

        //返回所读取的事件个数
        return event - buffer;
    }
    
EventHub采用INotify + epoll机制实现监听目录/dev/input下的设备节点，经过EventHub将input_event结构体 + deviceId 转换成RawEvent结构体，如下：

[-> InputEventReader.h]

    struct input_event {
     struct timeval time; //事件发生的时间点
     __u16 type;
     __u16 code;
     __s32 value;
    };

    struct RawEvent {
        nsecs_t when; //事件发生的时间店
        int32_t deviceId; //产生事件的设备Id
        int32_t type; // 事件类型
        int32_t code;
        int32_t value;
    };

此处事件类型有DEVICE_ADDED(添加)，DEVICE_REMOVED(删除)，FINISHED_DEVICE_SCAN(扫描完成)，以及(type<FIRST_SYNTHETIC_EVENT)其他事件。

#### 2.2 scanDevicesLocked

    void EventHub::scanDevicesLocked() {
        //【见小节2.3】
        status_t res = scanDirLocked(DEVICE_PATH);
        ...
    }
    
此处DEVICE_PATH="/dev/input" 

#### 2.3 scanDirLocked

    status_t EventHub::scanDirLocked(const char *dirname)
    {
        char devname[PATH_MAX];
        char *filename;
        DIR *dir;
        struct dirent *de;
        dir = opendir(dirname);
        
        strcpy(devname, dirname);
        filename = devname + strlen(devname);
        *filename++ = '/';
        //读取/dev/input/目录下所有的设备节点
        while((de = readdir(dir))) {
            if(de->d_name[0] == '.' &&
               (de->d_name[1] == '\0' ||
                (de->d_name[1] == '.' && de->d_name[2] == '\0')))
                continue;
            strcpy(filename, de->d_name);
            //打开相应的设备节点【2.4】
            openDeviceLocked(devname);
        }
        closedir(dir);
        return 0;
    }

#### 2.4 openDeviceLocked

    status_t EventHub::openDeviceLocked(const char *devicePath) {
        char buffer[80];
        //打开设备文件
        int fd = open(devicePath, O_RDWR | O_CLOEXEC);
        InputDeviceIdentifier identifier;
        //获取设备名
        if(ioctl(fd, EVIOCGNAME(sizeof(buffer) - 1), &buffer) < 1){
        } else {
            buffer[sizeof(buffer) - 1] = '\0';
            identifier.name.setTo(buffer);
        }
        
        identifier.bus = inputId.bustype;
        identifier.product = inputId.product;
        identifier.vendor = inputId.vendor;
        identifier.version = inputId.version;
        
        //获取设备物理地址
        if(ioctl(fd, EVIOCGPHYS(sizeof(buffer) - 1), &buffer) < 1) {
        } else {
            buffer[sizeof(buffer) - 1] = '\0';
            identifier.location.setTo(buffer);
        }

        //获取设备唯一ID
        if(ioctl(fd, EVIOCGUNIQ(sizeof(buffer) - 1), &buffer) < 1) {
        } else {
            buffer[sizeof(buffer) - 1] = '\0';
            identifier.uniqueId.setTo(buffer);
        }
        //将identifier信息填充到fd
        assignDescriptorLocked(identifier);
        //设置fd为非阻塞方式
        fcntl(fd, F_SETFL, O_NONBLOCK);
        
        //获取设备ID，分配设备对象内存
        int32_t deviceId = mNextDeviceId++;
        Device* device = new Device(fd, deviceId, String8(devicePath), identifier);
        ...
        
        //注册epoll
        struct epoll_event eventItem;
        memset(&eventItem, 0, sizeof(eventItem));
        eventItem.events = EPOLLIN;
        if (mUsingEpollWakeup) {
            eventItem.events |= EPOLLWAKEUP;
        }
        eventItem.data.u32 = deviceId;
        if (epoll_ctl(mEpollFd, EPOLL_CTL_ADD, fd, &eventItem)) {
            delete device; //添加失败则删除该设备
            return -1;
        }
        ...
        //【见小节2.5】
        addDeviceLocked(device);
    }

#### 2.5 addDeviceLocked

    void EventHub::addDeviceLocked(Device* device) {
        mDevices.add(device->id, device); //添加到mDevices队列
        device->next = mOpeningDevices;
        mOpeningDevices = device;
    }


介绍了EventHub从设备节点获取事件的流程，当收到事件后接下里便开始处理事件。
    
## 三. InputReader

#### 3.1 processEventsLocked
[-> InputReader.cpp]

    void InputReader::processEventsLocked(const RawEvent* rawEvents, size_t count) {
        for (const RawEvent* rawEvent = rawEvents; count;) {
            int32_t type = rawEvent->type;
            size_t batchSize = 1;
            if (type < EventHubInterface::FIRST_SYNTHETIC_EVENT) {
                int32_t deviceId = rawEvent->deviceId;
                while (batchSize < count) {
                    if (rawEvent[batchSize].type >= EventHubInterface::FIRST_SYNTHETIC_EVENT
                            || rawEvent[batchSize].deviceId != deviceId) {
                        break;
                    }
                    batchSize += 1; //同一设备的事件打包处理
                }
                //数据事件的处理【见小节3.4】
                processEventsForDeviceLocked(deviceId, rawEvent, batchSize);
            } else {
                switch (rawEvent->type) {
                case EventHubInterface::DEVICE_ADDED:
                    //设备添加【见小节3.2】
                    addDeviceLocked(rawEvent->when, rawEvent->deviceId);
                    break;
                case EventHubInterface::DEVICE_REMOVED:
                    //设备移除
                    removeDeviceLocked(rawEvent->when, rawEvent->deviceId);
                    break;
                case EventHubInterface::FINISHED_DEVICE_SCAN:
                    //设备扫描完成
                    handleConfigurationChangedLocked(rawEvent->when);
                    break;
                default:
                    ALOG_ASSERT(false);//不会发生
                    break;
                }
            }
            count -= batchSize;
            rawEvent += batchSize;
        }
    }

事件处理总共哟以下几类类型：

- DEVICE_ADDED(设备增加), [见小节3.2]
- DEVICE_REMOVED(设备移除)
- FINISHED_DEVICE_SCAN(设备扫描完成)
- 数据事件, [见小节3.4]

先来说说DEVICE_ADDED设备增加的过程。

#### 3.2 addDeviceLocked

    void InputReader::addDeviceLocked(nsecs_t when, int32_t deviceId) {
        ssize_t deviceIndex = mDevices.indexOfKey(deviceId);
        if (deviceIndex >= 0) {
            return; //已添加的相同设备则不再添加
        }

        InputDeviceIdentifier identifier = mEventHub->getDeviceIdentifier(deviceId);
        uint32_t classes = mEventHub->getDeviceClasses(deviceId);
        int32_t controllerNumber = mEventHub->getDeviceControllerNumber(deviceId);
        //【见小节3.3】
        InputDevice* device = createDeviceLocked(deviceId, controllerNumber, identifier, classes);
        device->configure(when, &mConfig, 0);
        device->reset(when);

        mDevices.add(deviceId, device); //添加设备到mDevices
        ...
    }

#### 3.3 createDeviceLocked

    InputDevice* InputReader::createDeviceLocked(int32_t deviceId, int32_t controllerNumber,
            const InputDeviceIdentifier& identifier, uint32_t classes) {
        //创建InputDevice对象
        InputDevice* device = new InputDevice(&mContext, deviceId, bumpGenerationLocked(),
                controllerNumber, identifier, classes);
        ...
        
        //获取键盘源类型
        uint32_t keyboardSource = 0;
        int32_t keyboardType = AINPUT_KEYBOARD_TYPE_NON_ALPHABETIC;
        if (classes & INPUT_DEVICE_CLASS_KEYBOARD) {
            keyboardSource |= AINPUT_SOURCE_KEYBOARD;
        }
        if (classes & INPUT_DEVICE_CLASS_ALPHAKEY) {
            keyboardType = AINPUT_KEYBOARD_TYPE_ALPHABETIC;
        }
        if (classes & INPUT_DEVICE_CLASS_DPAD) {
            keyboardSource |= AINPUT_SOURCE_DPAD;
        }
        if (classes & INPUT_DEVICE_CLASS_GAMEPAD) {
            keyboardSource |= AINPUT_SOURCE_GAMEPAD;
        }
        
        //添加键盘类设备InputMapper
        if (keyboardSource != 0) {
            device->addMapper(new KeyboardInputMapper(device, keyboardSource, keyboardType));
        }

        //添加鼠标类设备InputMapper
        if (classes & INPUT_DEVICE_CLASS_CURSOR) {
            device->addMapper(new CursorInputMapper(device));
        }

        //添加触摸屏设备InputMapper
        if (classes & INPUT_DEVICE_CLASS_TOUCH_MT) {
            device->addMapper(new MultiTouchInputMapper(device));
        } else if (classes & INPUT_DEVICE_CLASS_TOUCH) {
            device->addMapper(new SingleTouchInputMapper(device));
        }
        ...
        return device;
    }

该方法主要功能：
    
- 创建InputDevice对象，将InputReader的mContext赋给InputDevice对象所对应的变量
- 根据设备类型来创建并添加相对应的InputMapper，同时设置mContext.

input设备类型有很多种，以上代码只列举部分常见的设备以及相应的InputMapper：

- 键盘类设备：KeyboardInputMapper
- 触摸屏设备：MultiTouchInputMapper或SingleTouchInputMapper
- 鼠标类设备：CursorInputMapper

介绍完设备增加过程，继续回到[小节3.1]除了设备的增删，更常见事件便是数据事件，那么接下来介绍数据事件的
处理过程。

#### 3.4 processEventsForDeviceLocked

    void InputReader::processEventsForDeviceLocked(int32_t deviceId,
            const RawEvent* rawEvents, size_t count) {
        ssize_t deviceIndex = mDevices.indexOfKey(deviceId);
        ...
        
        InputDevice* device = mDevices.valueAt(deviceIndex);
        if (device->isIgnored()) {
            return; //可忽略则直接返回
        }
        //【见小节3.5】
        device->process(rawEvents, count);
    }

#### 3.5 InputDevice.process

    void InputDevice::process(const RawEvent* rawEvents, size_t count) {
        size_t numMappers = mMappers.size();
        for (const RawEvent* rawEvent = rawEvents; count--; rawEvent++) {
            if (mDropUntilNextSync) {
                if (rawEvent->type == EV_SYN && rawEvent->code == SYN_REPORT) {
                    mDropUntilNextSync = false;
                } 
            } else if (rawEvent->type == EV_SYN && rawEvent->code == SYN_DROPPED) {
                mDropUntilNextSync = true; 
                reset(rawEvent->when);
            } else {
                for (size_t i = 0; i < numMappers; i++) {
                    InputMapper* mapper = mMappers[i];
                    //调用具体mapper来处理【见小节3.6】
                    mapper->process(rawEvent);
                }
            }
        }
    }

小节[3.3]createDeviceLocked创建设备并添加InputMapper，提到会有多种InputMapper。
这里以KeyboardInputMapper为例来展开说明

#### 3.6 KeyboardInputMapper.process
[-> InputReader.cpp ::KeyboardInputMapper]

    void KeyboardInputMapper::process(const RawEvent* rawEvent) {
        switch (rawEvent->type) {
        case EV_KEY: {
            int32_t scanCode = rawEvent->code;
            int32_t usageCode = mCurrentHidUsage;
            mCurrentHidUsage = 0;

            if (isKeyboardOrGamepadKey(scanCode)) {
                int32_t keyCode;
                uint32_t flags;
                //获取所对应的KeyCode【见小节3.7】
                if (getEventHub()->mapKey(getDeviceId(), scanCode, usageCode, &keyCode, &flags)) {
                    keyCode = AKEYCODE_UNKNOWN;
                    flags = 0;
                }
                //【见小节3.9】
                processKey(rawEvent->when, rawEvent->value != 0, keyCode, scanCode, flags);
            }
            break;
        }
        case EV_MSC: ...
        case EV_SYN: ...
        }
    }

#### 3.7 EventHub::mapKey
[-> EventHub.cpp]

    status_t EventHub::mapKey(int32_t deviceId,
            int32_t scanCode, int32_t usageCode, int32_t metaState,
            int32_t* outKeycode, int32_t* outMetaState, uint32_t* outFlags) const {
        AutoMutex _l(mLock);
        Device* device = getDeviceLocked(deviceId); //获取设备对象
        status_t status = NAME_NOT_FOUND;

        if (device) {
            sp<KeyCharacterMap> kcm = device->getKeyCharacterMap();
            if (kcm != NULL) {
                //根据scanCode找到keyCode【见小节3.8】
                if (!kcm->mapKey(scanCode, usageCode, outKeycode)) {
                    *outFlags = 0;
                    status = NO_ERROR;
                }
            }
            ...
        }
        ...
        return status;
    }

将事件的扫描码(scanCode)转换成键盘码(Keycode)

#### 3.8 KeyCharacterMap::mapKey
[-> KeyCharacterMap.cpp]
    
    status_t KeyCharacterMap::mapKey(int32_t scanCode, int32_t usageCode, int32_t* outKeyCode) const {
        if (usageCode) { 
            ... // usageCode=0，此时不走该分支
        }
        if (scanCode) {
            ssize_t index = mKeysByScanCode.indexOfKey(scanCode);
            if (index >= 0) {
                //根据scanCode找到keyCode
                *outKeyCode = mKeysByScanCode.valueAt(index);
                return OK;
            }
        }
        *outKeyCode = AKEYCODE_UNKNOWN;
        return NAME_NOT_FOUND;
    }
    

#### 3.9 InputMapper.processKey
[-> InputReader.cpp]

    void KeyboardInputMapper::processKey(nsecs_t when, bool down, int32_t keyCode,
            int32_t scanCode, uint32_t policyFlags) {

        if (down) {
            if (mParameters.orientationAware && mParameters.hasAssociatedDisplay) {
                keyCode = rotateKeyCode(keyCode, mOrientation);
            }

            ssize_t keyDownIndex = findKeyDown(scanCode);
            if (keyDownIndex >= 0) {
                //mKeyDowns记录着所有按下的键
                keyCode = mKeyDowns.itemAt(keyDownIndex).keyCode;
            } else {
                ...
                mKeyDowns.push(); //压入栈顶
                KeyDown& keyDown = mKeyDowns.editTop();
                keyDown.keyCode = keyCode;
                keyDown.scanCode = scanCode;
            }
            mDownTime = when; //记录按下时间点
            
        } else {
            ssize_t keyDownIndex = findKeyDown(scanCode);
            if (keyDownIndex >= 0) {
                //键抬起操作，则移除按下事件
                keyCode = mKeyDowns.itemAt(keyDownIndex).keyCode;
                mKeyDowns.removeAt(size_t(keyDownIndex));
            } else {
                return;  //键盘没有按下操作，则直接忽略抬起操作
            }
        }
        ...
        nsecs_t downTime = mDownTime;
        ...
        
        //创建NotifyKeyArgs对象, when记录eventTime, downTime记录按下时间；
        NotifyKeyArgs args(when, getDeviceId(), mSource, policyFlags,
                down ? AKEY_EVENT_ACTION_DOWN : AKEY_EVENT_ACTION_UP,
                AKEY_EVENT_FLAG_FROM_SYSTEM, keyCode, scanCode, newMetaState, downTime);
        //通知key事件【见小节3.10】
        getListener()->notifyKey(&args);
    }
    
其中：

- mKeyDowns记录着所有按下的键
- mDownTime记录按下时间点
- 此处KeyboardInputMapper的mContext指向InputReader，getListener()获取的便是mQueuedListener。
接下来调用该对象的notifyKey.

#### 3.10 notifyKey
[-> InputListener.cpp]

    void QueuedInputListener::notifyKey(const NotifyKeyArgs* args) {
        mArgsQueue.push(new NotifyKeyArgs(*args));
    }

mArgsQueued的数据类型为Vector<NotifyArgs*>，将该key事件压人该栈顶。 到此,整个事件加工完成,
再然后就是将事件发送给InputDispatcher线程.

## 四. QueuedListener

### 4.1 QueuedInputListener.flush
[-> InputListener.cpp]

    void QueuedInputListener::flush() {
        size_t count = mArgsQueue.size();
        for (size_t i = 0; i < count; i++) {
            NotifyArgs* args = mArgsQueue[i];
            //【见小节4.2】
            args->notify(mInnerListener);
            delete args;
        }
        mArgsQueue.clear();
    }

从InputManager对象初始化的过程可知，`mInnerListener`便是InputDispatcher对象。

### 4.2 NotifyKeyArgs.notify
[-> InputListener.cpp]

    void NotifyKeyArgs::notify(const sp<InputListenerInterface>& listener) const {
        listener->notifyKey(this); // this是指NotifyKeyArgs【见小节4.3】
    }

### 4.3 InputDispatcher.notifyKey
[-> InputDispatcher.cpp]

    void InputDispatcher::notifyKey(const NotifyKeyArgs* args) {
        if (!validateKeyEvent(args->action)) {
            return;
        }
        ...
        int32_t keyCode = args->keyCode;

        if (keyCode == AKEYCODE_HOME) {
            if (args->action == AKEY_EVENT_ACTION_DOWN) {
                property_set("sys.domekey.down", "1");
            } else if (args->action == AKEY_EVENT_ACTION_UP) {
                property_set("sys.domekey.down", "0");
            }
        }

        if (metaState & AMETA_META_ON && args->action == AKEY_EVENT_ACTION_DOWN) {
            int32_t newKeyCode = AKEYCODE_UNKNOWN;
            if (keyCode == AKEYCODE_DEL) {
                newKeyCode = AKEYCODE_BACK;
            } else if (keyCode == AKEYCODE_ENTER) {
                newKeyCode = AKEYCODE_HOME;
            }
            if (newKeyCode != AKEYCODE_UNKNOWN) {
                AutoMutex _l(mLock);
                struct KeyReplacement replacement = {keyCode, args->deviceId};
                mReplacedKeys.add(replacement, newKeyCode);
                keyCode = newKeyCode;
                metaState &= ~AMETA_META_ON;
            }
        } else if (args->action == AKEY_EVENT_ACTION_UP) {
            AutoMutex _l(mLock);
            struct KeyReplacement replacement = {keyCode, args->deviceId};
            ssize_t index = mReplacedKeys.indexOfKey(replacement);
            if (index >= 0) {
                keyCode = mReplacedKeys.valueAt(index);
                mReplacedKeys.removeItemsAt(index);
                metaState &= ~AMETA_META_ON;
            }
        }

        KeyEvent event;
        //初始化KeyEvent对象
        event.initialize(args->deviceId, args->source, args->action,
                flags, keyCode, args->scanCode, metaState, 0,
                args->downTime, args->eventTime);
        //mPolicy是指NativeInputManager对象。【小节4.3.1】
        mPolicy->interceptKeyBeforeQueueing(&event, /*byref*/ policyFlags);

        bool needWake;
        {
            mLock.lock();
            if (shouldSendKeyToInputFilterLocked(args)) {
                mLock.unlock();
                policyFlags |= POLICY_FLAG_FILTERED;
                //【见小节4.3.2】
                if (!mPolicy->filterInputEvent(&event, policyFlags)) {
                    return; //事件被filter所消费掉
                }
                mLock.lock();
            }

            int32_t repeatCount = 0;
            //创建KeyEntry对象
            KeyEntry* newEntry = new KeyEntry(args->eventTime,
                    args->deviceId, args->source, policyFlags,
                    args->action, flags, keyCode, args->scanCode,
                    metaState, repeatCount, args->downTime);
            //将KeyEntry放入队列【见小节4.3.3】
            needWake = enqueueInboundEventLocked(newEntry);
            mLock.unlock();
        }

        if (needWake) {
            //唤醒InputDispatcher线程【见小节4.3.5】
            mLooper->wake();
        }
    }


该方法的主要功能：

1. 调用NativeInputManager.interceptKeyBeforeQueueing，加入队列前执行拦截动作，但并不改变流程，调用链：
  - IMS.interceptKeyBeforeQueueing
  - InputMonitor.interceptKeyBeforeQueueing (继承IMS.WindowManagerCallbacks)
  - PhoneWindowManager.interceptKeyBeforeQueueing (继承WindowManagerPolicy)
2. 当mInputFilterEnabled=true(该值默认为false,可通过setInputFilterEnabled设置),则调用NativeInputManager.filterInputEvent过滤输入事件；
  - 当返回值为false则过滤该事件，不再往下分发；
3. 生成KeyEvent，并调用enqueueInboundEventLocked，将该事件加入到InputDispatcherd的成员变量mInboundQueue。


      
#### 4.3.1 interceptKeyBeforeQueueing

    void NativeInputManager::interceptKeyBeforeQueueing(const KeyEvent* keyEvent,
            uint32_t& policyFlags) {
        ...
        if ((policyFlags & POLICY_FLAG_TRUSTED)) {
            nsecs_t when = keyEvent->getEventTime(); //时间
            JNIEnv* env = jniEnv();
            jobject keyEventObj = android_view_KeyEvent_fromNative(env, keyEvent);
            if (keyEventObj) {
                // 调用Java层的IMS.interceptKeyBeforeQueueing
                wmActions = env->CallIntMethod(mServiceObj,
                        gServiceClassInfo.interceptKeyBeforeQueueing,
                        keyEventObj, policyFlags);
                ...
            } else {
                ...
            }
            handleInterceptActions(wmActions, when, /*byref*/ policyFlags);
        } else {
            ...
        }
    }

该方法会调用Java层的InputManagerService的interceptKeyBeforeQueueing()方法。

#### 4.3.2 filterInputEvent

    bool NativeInputManager::filterInputEvent(const InputEvent* inputEvent, uint32_t policyFlags) {
        jobject inputEventObj;

        JNIEnv* env = jniEnv();
        switch (inputEvent->getType()) {
        case AINPUT_EVENT_TYPE_KEY:
            inputEventObj = android_view_KeyEvent_fromNative(env,
                    static_cast<const KeyEvent*>(inputEvent));
            break;
        case AINPUT_EVENT_TYPE_MOTION:
            inputEventObj = android_view_MotionEvent_obtainAsCopy(env,
                    static_cast<const MotionEvent*>(inputEvent));
            break;
        default:
            return true; // 走事件正常的分发流程
        }

        if (!inputEventObj) {
            return true; // 走事件正常的分发流程
        }

        //调用Java层的IMS.filterInputEvent()
        jboolean pass = env->CallBooleanMethod(mServiceObj, gServiceClassInfo.filterInputEvent,
                inputEventObj, policyFlags);
        if (checkAndClearExceptionFromCallback(env, "filterInputEvent")) {
            pass = true; //出现Exception，则走事件正常的分发流程
        }
        env->DeleteLocalRef(inputEventObj);
        return pass;
    }
  
该方法会调用Java层的InputManagerService的filterInputEvent()方法。
再回到【小节4.3】可知，接下来执行enqueueInboundEventLocked。

#### 4.3.3 enqueueInboundEventLocked

    bool InputDispatcher::enqueueInboundEventLocked(EventEntry* entry) {
        bool needWake = mInboundQueue.isEmpty();
        //将该事件放入mInboundQueue队列尾部
        mInboundQueue.enqueueAtTail(entry);
        traceInboundQueueLengthLocked();

        switch (entry->type) {
        case EventEntry::TYPE_KEY: {
            KeyEntry* keyEntry = static_cast<KeyEntry*>(entry);
            if (isAppSwitchKeyEventLocked(keyEntry)) {
                if (keyEntry->action == AKEY_EVENT_ACTION_DOWN) {
                    mAppSwitchSawKeyDown = true; //按下事件
                } else if (keyEntry->action == AKEY_EVENT_ACTION_UP) {
                    if (mAppSwitchSawKeyDown) {
                        //其中APP_SWITCH_TIMEOUT=500ms
                        mAppSwitchDueTime = keyEntry->eventTime + APP_SWITCH_TIMEOUT;
                        mAppSwitchSawKeyDown = false;
                        needWake = true;
                    }
                }
            }
            break;
        }

        case EventEntry::TYPE_MOTION: {
            //当前App无响应且用户希望切换到其他应用窗口，则drop该窗口事件，并处理其他窗口事件
            MotionEntry* motionEntry = static_cast<MotionEntry*>(entry);
            if (motionEntry->action == AMOTION_EVENT_ACTION_DOWN
                    && (motionEntry->source & AINPUT_SOURCE_CLASS_POINTER)
                    && mInputTargetWaitCause == INPUT_TARGET_WAIT_CAUSE_APPLICATION_NOT_READY
                    && mInputTargetWaitApplicationHandle != NULL) {
                int32_t displayId = motionEntry->displayId;
                int32_t x = int32_t(motionEntry->pointerCoords[0].
                        getAxisValue(AMOTION_EVENT_AXIS_X));
                int32_t y = int32_t(motionEntry->pointerCoords[0].
                        getAxisValue(AMOTION_EVENT_AXIS_Y));
                //查询可触摸的窗口【见小节4.3.4】
                sp<InputWindowHandle> touchedWindowHandle = findTouchedWindowAtLocked(displayId, x, y);
                if (touchedWindowHandle != NULL
                        && touchedWindowHandle->inputApplicationHandle
                                != mInputTargetWaitApplicationHandle) {
                    mNextUnblockedEvent = motionEntry;
                    needWake = true;
                }
            }
            break;
        }
        }

        return needWake;
    }

AppSwitchKeyEvent是指keyCode等于以下值：

- AKEYCODE_HOME
- AKEYCODE_ENDCALL 
- AKEYCODE_APP_SWITCH


#### 4.3.4 findTouchedWindowAtLocked
[-> InputDispatcher.cpp]

    sp<InputWindowHandle> InputDispatcher::findTouchedWindowAtLocked(int32_t displayId,
            int32_t x, int32_t y) {
        //从前台到后台来遍历查询可触摸的窗口
        size_t numWindows = mWindowHandles.size();
        for (size_t i = 0; i < numWindows; i++) {
            sp<InputWindowHandle> windowHandle = mWindowHandles.itemAt(i);
            const InputWindowInfo* windowInfo = windowHandle->getInfo();
            if (windowInfo->displayId == displayId) {
                int32_t flags = windowInfo->layoutParamsFlags;

                if (windowInfo->visible) {
                    if (!(flags & InputWindowInfo::FLAG_NOT_TOUCHABLE)) {
                        bool isTouchModal = (flags & (InputWindowInfo::FLAG_NOT_FOCUSABLE
                                | InputWindowInfo::FLAG_NOT_TOUCH_MODAL)) == 0;
                        if (isTouchModal || windowInfo->touchableRegionContainsPoint(x, y)) {
                            return windowHandle; //找到目标窗口
                        }
                    }
                }
            }
        }
        return NULL;
    }
    
#### 4.3.5 Looper.wake
[-> system/core/libutils/Looper.cpp]

    void Looper::wake() {
        uint64_t inc = 1;
        
        ssize_t nWrite = TEMP_FAILURE_RETRY(write(mWakeEventFd, &inc, sizeof(uint64_t)));
        if (nWrite != sizeof(uint64_t)) {
            if (errno != EAGAIN) {
                ALOGW("Could not write wake signal, errno=%d", errno);
            }
        }
    }
    
[小节4.3]的过程会调用enqueueInboundEventLocked()方法来决定是否需要将数字1写入句柄mWakeEventFd来唤醒InputDispatcher线程.
满足唤醒的条件:

1. 执行enqueueInboundEventLocked方法前,mInboundQueue队列为空,执行完必然不再为空,则需要唤醒分发线程;
2. 当事件类型为key事件,且发生一对按下和抬起操作,则需要唤醒;
3. 当事件类型为motion事件,且当前可触摸的窗口属于另一个应用,则需要唤醒.

## 五. 总结

用一张图来整体概况InputReader线程的主要工作：

![input_reader](/images/input/input_reader.jpg)

小技巧IMS.filterInputEvent可以过滤无需上报的事件，当该方法返回值为false则代表是需要被过滤掉的事件，无机会交给InputDispatcher来分发。

InputReader的核心工作就是从EventHub获取数据后生成EventEntry事件，加入到InputDispatcher的mInboundQueue队列，再唤醒InputDispatcher线程。
EventHub采用INotify + epoll机制实现监听目录/dev/input下的设备节点，经过EventHub将input_event结构体 + deviceId 转换成RawEvent结构体。
对于/dev/input节点的event事件所对应的输入设备信息位于`/proc/bus/input/devices`，也可以通过`getevent`来获取事件. 不同的input事件所对应的物理input节点，比如常见的情形：

- 屏幕触摸和(MENU,HOME,BACK)3按键：对应一个input设备节点；
- power键和音量(下)键：对应一个input设备节点；
- 音量(上)键：对应一个input设备节点；

InputReader整个过程涉及多次事件封装转换，如下：

- getEvents：读取/dev/input节点，转换 input_event -> RawEvent
- processEventsLocked: 转换RawEvent -> NotifyKeyArgs(NotifyArgs)
- QueuedListener->flush：转换NotifyKeyArgs -> KeyEntry(EventEntry)

InputReader线程不断循环地执行InputReader.loopOnce(), 每次处理完生成的是EventEntry(比如KeyEntry, MotionEntry), 接下来的工作就交给InputDispatcher线程。
