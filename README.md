# Telephony


## 一、TelecomService 启动流程

从Android MM开始，手机开机时，都会加载call模块中的TelecomService，把Telecom服务添加到系统服务中，并对默认短信通过模块等默认app进行相应的授权。

模块名 LOCAL_PACKAGE_NAME = Telecom

**SystemServer启动TelecomLoaderService**

```
mSystemServiceManager.startService(TelecomLoaderService.class);
```

在SystemServer的startOtherService启动TelecomLoaderService，然而这个TelecomLoaderService现在起来并没有做什么
。需要等到回调onBootPhase，才会去启动TelecomService。顾名思义，这个类就是为了去加载TelecomService而存在的。
system_process启动完毕后，会回调所有SystemService的onBootPhase函数。

```
@Override
public void onBootPhase(int phase) {
    if (phase == PHASE_ACTIVITY_MANAGER_READY) {
        registerDefaultAppNotifier();
        registerCarrierConfigChangedReceiver();
        connectToTelecom();
    }
}
```

**TelecomLoaderService启动TelecomService**

在TelecomLoaderService的connectToTelecom()，绑定TelecomService。
实例化一个intent，也就是启动我们要绑定的服务。其中intent的相关信息如下

```
Action: "com.android.ITelecomServcie"

ComponentName: new ComponentName("com.android.server.telecom", "com.androdi.server.telecom.components.TelecomService");
```

从这里的参数可以找到对应要启动的服务是哪个包下的哪个服务。

```
private void connectToTelecom() {
    synchronized (mLock) {
        if (mServiceConnection != null) {
            // TODO: Is unbinding worth doing or wait for system to rebind?
            mContext.unbindService(mServiceConnection);
            mServiceConnection = null;
        }

        TelecomServiceConnection serviceConnection = new TelecomServiceConnection();
        Intent intent = new Intent(SERVICE_ACTION);
        intent.setComponent(SERVICE_COMPONENT);
        int flags = Context.BIND_IMPORTANT | Context.BIND_FOREGROUND_SERVICE
                | Context.BIND_AUTO_CREATE;

        // Bind to Telecom and register the service
        if (mContext.bindServiceAsUser(intent, serviceConnection, flags, UserHandle.SYSTEM)) {
            mServiceConnection = serviceConnection;
        }
    }
}
```

当绑定TelecomService服务后，会回调TelecomServiceConnection的onServiceConnected方法。
这个函数主要处理的就是对默认的call，sms，app授权运行时的权限，具体调用参考流程图。
同时，最主要的就是把Context.TELECOM_SERVICE添加到系统服务中，这里添加的就是TelecomServiceImpl的mBinderImpl。

```
ServiceManager.addService(Context.TELECOM_SERVICE, service);

packageManagerInternal.grantDefaultPermissionsToDefaultSmsApp(
                                        packageName, userId);
packageManagerInternal.grantDefaultPermissionsToDefaultDialerApp(
                                        packageName, userId);
packageManagerInternal.grantDefaultPermissionsToDefaultSimCallManager(
                                        packageName, userId);
```

**TelecomService加载TelecomSystem**

TelecomService的onBind方法，返回的IBinder对象就是上面说的TelecomServiceImpl的mBinderImpl

```
public IBinder onBind(Intent intent) {
    Log.d(this, "onBind");
    initializeTelecomSystem(this);
    /*White Pages Code begins*/
    if (CscFeature.getInstance().getEnableStatus(NameIDHelper.mEnableWhitePagesConfig)
            && NameIDHelper.isNameIDInstalledAndEnabled(this)) {
        NameIDHelper.init(this);
    }
    /*White Pages Code ends*/
    synchronized (getTelecomSystem().getLock()) {
        return getTelecomSystem().getTelecomServiceImpl().getBinder();
    }
}
```

initializeTelecomSystem主要初始化TelecomSystem对象，里面包含Call将会用到的各个对象，比如:

* 1.CallsManager

* 2.CallIntentProcessor

* 3.TelecomServiceImpl

* 4.TelecomSystemDB

在创建TelecomServiceImpl对象的时候，就会生成TelecomService.Stub mBinderImpl对象

```
private final ITelecomService.Stub mBinderImpl = new ITelecomService.Stub(){}
```

这个就是我们最终添加到系统服务的对象，至此，TelecomLoaderService的加载TelecomService的工作基本完成

**总结**

TelecomLoaderService加载TelecomService的过程和他的名字非常符合，完成的工作比较简单，
启动TelecomService，授权默认应用，最后附上一张时序图。






## 二、MO CALL流程

**Call文件目录**

从上层InCallUI一直到Telephony Framework层，总共包括下面五个部分。


* InCallUI : packages/apps/InCallUI (system/priv-app/InCallUI.apk)

* Telecom framwork : Frameworks/base/Telecomm (system/framework/framework.jar)

* Telecom : android/packages/services/Telecomm (system/priv-app/Telecom.apk)

* TeleService : android/packages/services/Telephony (system/priv-app/TeleService.apk)

* Telephony Framework:Frameworks/opt/Telephony （system/framework/telephony-common.jar）

**MO Call 总流程图**

从总体上看，MO Call 分两步走，首先是先显示InCallUI界面，其次，发送指定到Telephony Framework，更新上层Call状态

启动InCallUI Log

```
//CallIntentProcessor@processOutgoingCallIntent
   07-10 14:50:14.271  1029  1029 I Telecom : com.android.server.telecom.CallIntentProcessor: onReceive - isUnknownCall: false: PCR.oR@Abg
   07-10 14:50:14.271  1029  1029 D Telecom : com.android.server.telecom.CallIntentProcessor: processOutgoingCallIntent: : PCR.oR@Abg

   //CallsManager@startOutgoingCall
   07-10 14:50:14.487  1029  1029 V Telecom : com.android.server.telecom.Call: setTargetPhoneAccount() accountHandle = null: PCR.oR@Abg
   07-10 14:50:14.488  1029  1029 V Telecom : com.android.server.telecom.CallsManager: startOutgoingCall found accounts = [ComponentInfo{com.android.phone/com.android.services.telephony.TelephonyConnectionService}, [5d82928ef18b3e949e61cd661cec88f28f396828], UserHandle{0}, ComponentInfo{com.android.phone/com.android.services.telephony.TelephonyConnectionService}, [8658334b1c0ce79413b631f85fb80b88b4cd87e7], UserHandle{0}]: PCR.oR@Abg
   07-10 14:50:14.515  1029  1029 D Telecom : CallsManager: startOutgoingCall() phoneAccountHandle = ComponentInfo{com.android.phone/com.android.services.telephony.TelephonyConnectionService}, [5d82928ef18b3e949e61cd661cec88f28f396828], UserHandle{0}: PCR.oR@Abg
   07-10 14:50:14.555  1029  1029 V Telecom : com.android.server.telecom.Call: setState NEW -> CONNECTING: PCR.oR@Abg
   07-10 14:50:14.555  1029  1029 I Telecom : Event: Call TC@2: SET_CONNECTING, ComponentInfo{com.android.phone/com.android.services.telephony.TelephonyConnectionService}, [5d82928ef18b3e949e61cd661cec88f28f396828], UserHandle{0}: PCR.oR@Abg
   07-10 14:50:14.557  1029  1029 V Telecom : com.android.server.telecom.CallsManager: addCall([TC@2, CONNECTING, null, tel:1234567890, A, childs(0), has_parent(false), unknown, unknown [Capabilities:] [Properties:]]): PCR.oR@Abg

   //InCallController@bindToInCallService
   07-10 14:50:14.561  1029  1029 I Telecom : com.android.server.telecom.InCallController$EmergencyInCallServiceConnection: Attempting to bind to InCall [ComponentInfo{com.samsung.android.incallui/com.android.incallui.InCallServiceImpl} supportsExternal? false], with Intent { act=android.telecom.InCallService cmp=com.samsung.android.incallui/com.android.incallui.InCallServiceImpl launchParam=MultiScreenLaunchParams { mDisplayId=0 mBaseDisplayId=0 mFlags=0 } (has extras) }: PCR.oR@Abg

   //InCallServiceImpl@onBind
   07-10 14:50:14.568  8802  8802 I InCall  : InCallServiceImpl -  - perf - onBind : start

   //InCallPresenter@setUp
   07-10 14:50:14.577  8802  8802 I InCall  : InCallPresenter -  - perf - setUp()
   //IncallPresenter@onCallListChange (NO_CALLS->NO_CALLS)
   07-10 14:50:14.577  8802  8802 I InCall  : InCallPresenter -  - onCallListChange: start
   07-10 14:50:14.577  8802  8802 D InCall  : InCallPresenter -  - onCallListChange oldState= NO_CALLS newState=NO_CALLS
   07-10 14:50:14.577  8802  8802 D InCall  : InCallPresenter -  - startOrFinishUi: NO_CALLS -> NO_CALLS
   07-10 14:50:14.577  8802  8802 D InCall  : InCallPresenter -  - onCallListChange newState changed to NO_CALLS
   07-10 14:50:14.578  8802  8802 I InCall  : InCallPresenter -  - Phone switching state: NO_CALLS -> NO_CALLS
   07-10 14:50:14.583  8802  8802 I InCall  : InCallPresenter -  - onCallListChange: end
   07-10 14:50:14.646  8802  8802 I InCall  : InCallPresenter -  - perf - Finished InCallPresenter.setUp
   07-10 14:50:14.661  8802  8802 I InCall  : InCallServiceImpl -  - perf - onBind : end

   //InCallController@onConnected
   07-10 14:50:14.688  1029  1029 D Telecom : com.android.server.telecom.InCallController$InCallServiceBindingConnection$1: onServiceConnected: ComponentInfo{com.samsung.android.incallui/com.android.incallui.InCallServiceImpl} false true: ICSBC.oSC@Abk
   07-10 14:50:14.688  1029  1029 I Telecom : com.android.server.telecom.InCallController: perf - onConnected to ComponentInfo{com.samsung.android.incallui/com.android.incallui.InCallServiceImpl}: ICSBC.oSC@Abk
   07-10 14:50:14.689  1029  1029 I Telecom : com.android.server.telecom.InCallController: Adding 1 calls to InCallService after onConnected: ComponentInfo{com.samsung.android.incallui/com.android.incallui.InCallServiceImpl}, including external calls: ICSBC.oSC@Abk

   //InCallService@addCall
   07-10 14:50:14.689  8802  8802 D InCallService: InCallService - handleMessage: 1
   07-10 14:50:14.691  8802  8823 D InCallService: InCallServiceBinder - addCall: id(TC@2), state(CONNECTING), [Capabilities:]
   07-10 14:50:14.691  8802  8802 D InCallService: InCallService - handleMessage: 2
   //Call@internalUpdate
   07-10 14:50:14.691  8802  8802 D TelecomParcelCall: Do NOT make VideoCallImpl because mVideoCallProvider is null
   07-10 14:50:14.691  8802  8802 D TelecomCall: internalUpdate - mVideoCallImpl: null, videoCallChanged: false, isVideoCallProviderChanged(): true

   //CallList@onCallAdded
   07-10 14:50:14.731  8802  8802 D InCall  : CallList - onCallAdded: callState=13  //13 CONNECTING
   07-10 14:50:14.740  8802  8802 D InCall  : CallList - needToUpdate: true
   07-10 14:50:14.740  8802  8802 D InCall  : CallList -   [Call_1, CONNECTING, [Capabilities:], children:[], parent:null, conferenceable:[], videoState:Audio Only, mSessionModificationState:0, VideoSettings:(CameraDir:-1)(CameraId:1)]
   07-10 14:50:14.740  8802  8802 I InCall  : CallList - onUpdate - [Call_1, CONNECTING, [Capabilities:], children:[], parent:null, conferenceable:[], videoState:Audio Only, mSessionModificationState:0, VideoSettings:(CameraDir:-1)(CameraId:1)]

   //InCallPresenter@onCallListChange (NO_CALLS->PENDING_OUTGOING)
   07-10 14:50:14.740  8802  8802 I InCall  : InCallPresenter -  - onCallListChange: start
   07-10 14:50:14.740  8802  8802 D InCall  : InCallPresenter -  - perf - startContactInfoSearch
   07-10 14:50:15.039  8802  8802 D InCall  : InCallPresenter$ContactInfoCallback -  - onContactInfoComplete
   07-10 14:50:15.039  8802  8802 D InCall  : InCallPresenter -  - onCallListChange oldState= NO_CALLS newState=PENDING_OUTGOING
   07-10 14:50:15.039  8802  8802 D InCall  : InCallPresenter -  - startOrFinishUi: NO_CALLS -> PENDING_OUTGOING
   07-10 14:50:15.179  8802  8802 D InCall  : InCallPresenter -  - onCallListChange newState changed to PENDING_OUTGOING
   07-10 14:50:15.179  8802  8802 I InCall  : InCallPresenter -  - Phone switching state: NO_CALLS -> PENDING_OUTGOING
```

依据上面的log，抹去细枝末节，就可以大致跟进启动InCallUI的流程，那么，过滤一下上面的log，我们想知道此时Call状态的变化，只需查看下面两个关键字

```
InCallService: InCallServiceBinder
Phone switching state
```

一个在Telecom层打印，一个在InCallUI打印，从log提取Call的状态对比如下:

```
07-10 14:50:14.691  8802  8823 D InCallService: InCallServiceBinder - addCall: id(TC@2), state(CONNECTING), [Capabilities:]
07-10 14:50:15.078  8802  8815 D InCallService: InCallServiceBinder - updateCall: id(TC@2), state(CONNECTING), [Capabilities:]
07-10 14:50:15.139  8802  8815 D InCallService: InCallServiceBinder - updateCall: id(TC@2), state(DIALING), [Capabilities: CAPABILITY_SUPPORT_HOLD CAPABILITY_MUTE CAPABILITY_CANNOT_DOWNGRADE_VIDEO_TO_AUDIO]
07-10 14:50:25.832  8802  8821 D InCallService: InCallServiceBinder - updateCall: id(TC@2), state(ACTIVE), [Capabilities: CAPABILITY_HOLD CAPABILITY_SUPPORT_HOLD CAPABILITY_MUTE CAPABILITY_CANNOT_DOWNGRADE_VIDEO_TO_AUDIO]
07-10 14:50:30.170  8802  8816 D InCallService: InCallServiceBinder - updateCall: id(TC@2), state(DISCONNECTING), [Capabilities: CAPABILITY_HOLD CAPABILITY_SUPPORT_HOLD CAPABILITY_MUTE CAPABILITY_CANNOT_DOWNGRADE_VIDEO_TO_AUDIO]
07-10 14:50:30.525  8802  8816 D InCallService: InCallServiceBinder - updateCall: id(TC@2), state(DISCONNECTED), [Capabilities: CAPABILITY_HOLD CAPABILITY_SUPPORT_HOLD CAPABILITY_MUTE CAPABILITY_CANNOT_DOWNGRADE_VIDEO_TO_AUDIO]

07-10 14:50:14.578  8802  8802 I InCall  : InCallPresenter -  - Phone switching state: NO_CALLS -> NO_CALLS
07-10 14:50:15.179  8802  8802 I InCall  : InCallPresenter -  - Phone switching state: NO_CALLS -> PENDING_OUTGOING
07-10 14:50:15.639  8802  8802 I InCall  : InCallPresenter -  - Phone switching state: PENDING_OUTGOING -> OUTGOING
07-10 14:50:25.743  8802  8802 I InCall  : InCallPresenter -  - Phone switching state: OUTGOING -> INCALL
07-10 14:50:26.939  8802  8802 I InCall  : InCallPresenter -  - Phone switching state: INCALL -> INCALL
07-10 14:50:32.855  8802  8802 I InCall  : InCallPresenter -  - Phone switching state: INCALL -> NO_CALLS
```

**Mo Call 到Telephony Framework Log**

1、CallIntentProcessor 创建 NewOutGoingCallIntentBroadcaster 对象，并把intent 和 CallsManager创建的Call对象传递给他处理，他通过发送广播的形式，把android.intent.android.NEW_OUTGOING_CALL 发送给NewOutgoingCallBroadcastintentReceiver进一步处理，其实就是调用CallsManager的placeOutgingCall方法。

2、placeOutingCall方法最终调用Call对象的startCreateConnection方法，这里的Call，是Telecom Service，目录下的Call对象，

他本身实现了CreateConnectionResponse接口，顾名思义，这里创建完connection还会有个回调，表示创建成功与否的处理

这个方法把Call自身作为参数创建了CreateConnectionProcessor，由他来继续处理创建connection的工作。

3、接着，调用processor的attempNextPhoneAccount()，在这里会创建一个Response对象集成CreaseConnectionResponse，然后把它自身作为参数，传递给ConnectionServiceWrapper的createConnection方法，说到这个Response对象，它本身调用hanleCreateConnectionSuccess，最终也是调用第2说到的Call对象的实现

4、CreateConnection这里会去绑定com.android.services.telephony.TelephonyConnectionService这个服务，在链接成功之后，会把ConnectionServiceWrapper里面的Adapter对象，添加到ConnectionService的ConnectionServiceAdapter对象里面，然后回调BindCallback的onSuccess方法，同时，把上面说大到的Response对象添加到mPendingResponses Map 里面，待后续完成后取出来

5、在上面说到的onSuccess里面，会继续调用ConnectionService的createConnection方法，这个其实是用他的子类，TelephonyConnectionService，紧接着，进一步调用onCreateOutGoingConnection方法，继续调用createConnectionFor方法，生成一个TelephonyConnection，然后调用placeOutGoingConnection，到这里，他就去回去调用TelephonyFramework的phone对象的Dial方法了，返回一个Telephony Framework的connection对象，而上面生成的TelephonyConnection会根据它来更新自己，并且调用phone注册registerForPreciseCallStateChanged事件监听，用以监听Call状态的变化

6、ConnectionService的createConnection方法最后，调用mAdapter.handleCreateConnectionComplete，继续回调，CallsManager.onSuccessfulOutgoingCall，在这里设置call的状态为dialing，继续回调到InCallController的onCallStateChanged，通过告之InCallService去更新Call的状态，回调到InCallUI的call对象的onStateChange，最后调用到CallList的onUpdate去通到InCallPresenter。

```
//dial out action
//NewOutgoingCallIntentBroadcaster@processIntent
07-10 14:50:14.590  1029  1029 V Telecom : com.android.server.telecom.NewOutgoingCallIntentBroadcaster: Processing call intent in OutgoingCallIntentBroadcaster.: PCR.oR@Abg
07-10 14:50:14.632  1029  1029 I Telecom : com.android.server.telecom.NewOutgoingCallIntentBroadcaster: Sending NewOutgoingCallBroadcast for [TC@2, CONNECTING, null, tel:1234567890, A, childs(0), has_parent(false), unknown, unknown [Capabilities:] [Properties:]] to UserHandle{0}: PCR.oR@Abg

//NewOutgoingCallIntentBroadcaster@broadcastIntent
07-10 14:50:14.632  1029  1029 V Telecom : com.android.server.telecom.NewOutgoingCallIntentBroadcaster: Broadcasting intent: Intent { act=android.intent.action.NEW_OUTGOING_CALL flg=0x10000000 launchParam=MultiScreenLaunchParams { mDisplayId=0 mBaseDisplayId=0 mFlags=0 } (has extras) }.: PCR.oR@Abg

//NewOutgoingCallIntentBroadcaster@onReceive
07-10 14:50:14.747  1029  1029 V Telecom : com.android.server.telecom.NewOutgoingCallIntentBroadcaster$NewOutgoingCallBroadcastIntentReceiver: onReceive: Intent { act=android.intent.action.NEW_OUTGOING_CALL flg=0x10000010 launchParam=MultiScreenLaunchParams { mDisplayId=0 mBaseDisplayId=0 mFlags=0 } bqHint=0 (has extras) }: NOCBIR.oR@Abw
07-10 14:50:14.747  1029  1029 I Telecom : com.android.server.telecom.NewOutgoingCallIntentBroadcaster$NewOutgoingCallBroadcastIntentReceiver: Received new-outgoing-call-broadcast for [TC@2, CONNECTING, null, tel:1234567890, A, childs(0), has_parent(false), unknown, unknown [Capabilities:] [Properties:]] with data 1234567890: NOCBIR.oR@Abw

//CallsManager@placeOutgoingCall
07-10 14:50:14.757  1029  1029 I Telecom : com.android.server.telecom.CallsManager: Creating a new outgoing call with handle: tel:1234567890: NOCBIR.oR@Abw
07-10 14:50:14.789  1029  1029 D Telecom : CallsManager : placeOutgoingCall() getTargetPhoneAccount() = ComponentInfo{com.android.phone/com.android.services.telephony.TelephonyConnectionService}, [5d82928ef18b3e949e61cd661cec88f28f396828], UserHandle{0}

//Call@CreateConnectionProcessor
07-10 14:50:14.790  1029  1029 V Telecom : com.android.server.telecom.CreateConnectionProcessor: CreateConnectionProcessor created for Call = [TC@2, CONNECTING, null, tel:1234567890, A, childs(0), has_parent(false), unknown, unknown [Capabilities:] [Properties:]]: NOCBIR.oR@Abw
07-10 14:50:14.790  1029  1029 V Telecom : com.android.server.telecom.CreateConnectionProcessor: process: NOCBIR.oR@Abw

//CreateConnectionProcessor@attemptNextPhoneAccount
07-10 14:50:14.798  1029  1029 V Telecom : com.android.server.telecom.CreateConnectionProcessor: attemptNextPhoneAccount: NOCBIR.oR@Abw
07-10 14:50:14.799  1029  1029 I Telecom : com.android.server.telecom.CreateConnectionProcessor: Trying attempt CallAttemptRecord(ComponentInfo{com.android.phone/com.android.services.telephony.TelephonyConnectionService}, [5d82928ef18b3e949e61cd661cec88f28f396828], UserHandle{0},ComponentInfo{com.android.phone/com.android.services.telephony.TelephonyConnectionService}, [5d82928ef18b3e949e61cd661cec88f28f396828], UserHandle{0}): NOCBIR.oR@Abw
07-10 14:50:14.800  1029  1029 I Telecom : com.android.server.telecom.CreateConnectionProcessor: Attempting to call from ComponentInfo{com.android.phone/com.android.services.telephony.TelephonyConnectionService}: NOCBIR.oR@Abw

//ConnectionServiceWrapper@createConnection
07-10 14:50:14.801  1029  1029 D Telecom : com.android.server.telecom.ConnectionServiceWrapper: createConnection([TC@2, CONNECTING, com.android.phone/com.android.services.telephony.TelephonyConnectionService, tel:1234567890, A, childs(0), has_parent(false), unknown, unknown [Capabilities:] [Properties:]]) via ComponentInfo{com.android.phone/com.android.services.telephony.TelephonyConnectionService}.: NOCBIR.oR@Abw
07-10 14:50:14.801  1029  1029 D Telecom : com.android.server.telecom.ConnectionServiceWrapper: bind(): NOCBIR.oR@Abw
07-10 14:50:14.801  1029  1029 I Telecom : Event: Call TC@2: BIND_CS, ComponentInfo{com.android.phone/com.android.services.telephony.TelephonyConnectionService}: NOCBIR.oR@Abw

//ServiceBinder@onServiceConnected
07-10 14:50:14.820  1029  1029 I Telecom : com.android.server.telecom.ServiceBinder$ServiceBinderConnection: Service bound ComponentInfo{com.android.phone/com.android.services.telephony.TelephonyConnectionService}: SBC.oSC@Ab4

//ConnectionService@createConnection
//TelephonyConnectionService@onCreateOutgoingConnection
07-10 14:50:14.836  1326  1326 I Telephony: TelephonyConnectionService: onCreateOutgoingConnection, request: ConnectionRequest xxxxxxxxxxxxxxx Bundle[mParcelledData.dataSize=664]

//TelephonyConnectionService@createConnectionFor
07-10 14:50:14.870  1326  1326 I Telephony: TelephonyConnectionService: createConnectionFor. phoneType : 1 / isImsCall : false

07-10 14:50:14.870  1326  1326 V Telephony: GsmConnection: onStateChanged, state: INITIALIZING

//TelephonyConnectionService@placeOutgoingCall
07-10 14:50:14.876  1326  1326 D Telephony: TelephonyConnectionService: placeOutgoingConnection

//TelephonyConnectionUtils@DoDial
07-10 14:50:14.876  1326  1326 D Telephony: TelephonyConnectionUtils: DoDial CallType : 0CallDomain : 1

//adapter.handleCreateConnectionComplete
07-10 14:50:15.074  1029  6422 V Telecom : com.android.server.telecom.Call: handleCreateConnectionSuccessful ParcelableConnection [act:ComponentInfo{com.android.phone/com.android.services.telephony.TelephonyConnectionService}, [5d82928ef18b3e949e61cd661cec88f28f396828], UserHandle{0}], state:3, capabilities:[Capabilities: CAPABILITY_SUPPORT_HOLD CAPABILITY_MUTE CAPABILITY_CANNOT_DOWNGRADE_VIDEO_TO_AUDIO], properties:[Properties:], extras:Bundle[mParcelledData.dataSize=1140]: CSW.hCCC@AcI

//CallsManager@onSuccessfulOutgoingCall
07-10 14:50:15.135  1029  6422 V Telecom : com.android.server.telecom.CallsManager: onSuccessfulOutgoingCall, [TC@2, CONNECTING, com.android.phone/com.android.services.telephony.TelephonyConnectionService, tel:1234567890, A, childs(0), has_parent(false), unknown, unknown [Capabilities: CAPABILITY_SUPPORT_HOLD CAPABILITY_MUTE CAPABILITY_CANNOT_DOWNGRADE_VIDEO_TO_AUDIO] [Properties:]]: CSW.hCCC@AcI
07-10 14:50:15.136  1029  6422 I Telecom : CallsManager : setCallState CONNECTING -> DIALING, call: [TC@2, CONNECTING, com.android.phone/com.android.services.telephony.TelephonyConnectionService, tel:1234567890, A, childs(0), has_parent(false), unknown, unknown [Capabilities: CAPABILITY_SUPPORT_HOLD CAPABILITY_MUTE CAPABILITY_CANNOT_DOWNGRADE_VIDEO_TO_AUDIO] [Properties:]]
07-10 14:50:15.136  1029  6422 V Telecom : com.android.server.telecom.Call: setState CONNECTING -> DIALING: CSW.hCCC@AcI
07-10 14:50:15.136  1029  6422 I Telecom : Event: Call TC@2: SET_DIALING, successful outgoing call: CSW.hCCC@AcI

//InCallController@onCallStateChanged
07-10 14:50:15.137  1029  6422 V Telecom : com.android.server.telecom.InCallController: onCallStateChanged: [TC@2, DIALING, com.android.phone/com.android.services.telephony.TelephonyConnectionService, tel:1234567890, A, childs(0), has_parent(false), unknown, unknown [Capabilities: CAPABILITY_SUPPORT_HOLD CAPABILITY_MUTE CAPABILITY_CANNOT_DOWNGRADE_VIDEO_TO_AUDIO] [Properties:]]: CSW.hCCC@AcI

//InCallServiceImpl@updateCall
07-10 14:50:15.191  8802  8802 D InCallService: InCallService - handleMessage: 3
07-10 14:50:15.191  8802  8802 D TelecomParcelCall: Do NOT make VideoCallImpl because mVideoCallProvider is null
07-10 14:50:15.340  8802  8815 D InCallService: InCallServiceBinder - updateCall: id(TC@2), state(DIALING), [Capabilities: CAPABILITY_SUPPORT_HOLD CAPABILITY_MUTE CAPABILITY_CANNOT_DOWNGRADE_VIDEO_TO_AUDIO]

//InCallUI.Call@onStateChanged
07-10 14:50:15.490  8802  8802 I InCall  : Call_1 - TelecomCallCallback onStateChanged call=Call [id: TC@2, state: DIALING, details: [pa: ComponentInfo{com.android.phone/com.android.services.telephony.TelephonyConnectionService}, [5d82928ef18b3e949e61cd661cec88f28f396828], UserHandle{0}, hdl: tel:***********, caps: [Capabilities: CAPABILITY_SUPPORT_HOLD CAPABILITY_MUTE CAPABILITY_CANNOT_DOWNGRADE_VIDEO_TO_AUDIO], props: [Properties:]]] newState=1
07-10 14:50:15.491  8802  8802 D InCall  : Call -  - updateFromTelecomCall: Call [id: TC@2, state: DIALING, details: [pa: ComponentInfo{com.android.phone/com.android.services.telephony.TelephonyConnectionService}, [5d82928ef18b3e949e61cd661cec88f28f396828], UserHandle{0}, hdl: tel:***********, caps: [Capabilities: CAPABILITY_SUPPORT_HOLD CAPABILITY_MUTE CAPABILITY_CANNOT_DOWNGRADE_VIDEO_TO_AUDIO], props: [Properties:]]]

//InCallPresenter@onCallListChange
07-10 14:50:15.637  8802  8802 I InCall  : InCallPresenter -  - onCallListChange: start
07-10 14:50:15.639  8802  8802 D InCall  : InCallPresenter -  - onCallListChange oldState= PENDING_OUTGOING newState=OUTGOING
07-10 14:50:15.639  8802  8802 D InCall  : InCallPresenter -  - startOrFinishUi: PENDING_OUTGOING -> OUTGOING
```

ps:流程图后面会补上，之前之前习惯性用Axure，现在准备转到Visio

参考

感谢作者 整理的那么好

https://blog.csdn.net/amd123456789/category_6678740.html
