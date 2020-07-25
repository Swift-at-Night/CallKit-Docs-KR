# 푸시킷 (PushKit)의 VoIP 알림에 대응하기

![PushKit](https://img.shields.io/badge/PushKit-67c7c3?style=for-the-badge&logo=swift&logoColor=white)
![CallKit](https://img.shields.io/badge/CallKit-66e268?style=for-the-badge&logo=swift&logoColor=white)

## VoIP 푸시 알림을 받고 시스템 통화 인터페이스를 띄우기

만약 당신의 앱이 VoIP 서비스를 제공한다면 사용자 기기에 걸려오는 전화를 다루기 위해 푸시킷(`PushKit`)을 사용해야 합니다. 
푸시킷은 전화를 받기 위해 앱을 돌릴 필요없이 콜들을 관리할 수 있는 효율적인 방법을 제공합니다. 
전화를 감지하면 사용자 기기로 서버가 해당 전화에 대한 정보와 함께 푸시 알림을 보냅니다.
이 알림을 받으면 해당 디바이스는 앱을 깨워 사용자에게 알리고 통화서비스에 연결할 수 있는 시간을 줍다.

iOS13 이상의 앱은 푸시킷을 사용할 때 VoIP 통화를 다루는 경우 콜킷(`CallKit`)을 사용해야 합니다. 
콜킷은 통화관련 서비스를 제공하는 앱들이 서로 원활하게 동작하도록 보장하고 방해금지모드와 같은 기능에도 대응합니다. 
또한 콜킷은 Incoming / Outgoing call의 화면 과 같은 시스템의 통화 관련 UI들을 제공합니다. 
이러한 인터페이스가 제공되기 때문에 푸시킷과의 상호작용을 관리하기 위해서는 콜킷을 사용하는 것이 좋습니다. 

> **Important** 앱이 콜킷을 지원하지 않는다면 푸시 알림을 다루는데에 푸시킷을 사용할 수 없습니다. 대신에 `UserNotifications` 프레임워크을 사용하여 앱의 푸시 알림 지원을 설정할 수 있습니다. 만약 같은 알림에 응답 작업(내용 해독)을 할 필요가 있다면, **notification service extension**을 사용하면 됩니다. 더 많은 관련 정보를 원하신다면 [UserNotification](https://developer.apple.com/documentation/usernotifications) 문서를 참고하세요.

> **Note** 푸시킷(PushKit)을 지원하도록 앱을 설정하는 방법은 [Supporting PushKit Notifications in Your App](https://developer.apple.com/documentation/pushkit/supporting_pushkit_notifications_in_your_app)를 참고해주세요.

## 앱의 콜 관리를 위한  Call Provider 객체 생성하기  

VoIP 앱이 시스템 통화 인터페이스를 보여주기 위해서는 반드시 콜킷을 사용해야 합니다.
`CXProvider` 객체를 통해 이러한 인터페이스를 보여줄 수 있습니다. 
`CXProvider` 객체는 Incoming / Outgoing call들을 위한 사용자의 상호작용(User Interaction)을 관리합니다. 
`CXProvider`를 앱 라이프 사이클 초기에 생성하여 앱 내의 통화 관련 소스코드에 사용이 가능하도록 해야합니다.

아래의 코드 예시는 어떻게 `CXProvider`를 생성하고 어떻게 커스텀 `Delegate` 를 할당하는지 보여준다. 

(1)당신의 서비스에 대한 세부정보를 포함하는 `CXProviderConfiguration` 객체를 통해 (2)`CXProvider` 객체를 초기화 해줍니다. (3)앱의 커스텀 객체를 `Delegate` 로 할당하고 (4)그 객체를 사용자 액션과 통화 관련 변화에 대응에 사용하도록 합니다.

```swift
// MARK: - CXProvider 생성 및 Delegate 할당

// Configure the app’s CallKit provider object.
let config = CXProviderConfiguration(localizedName: “VoIP Service”)
config.supportsVideo = true
config.supportedHandleTypes = [.phoneNumber]
config.maximumCallsPerCallGroup = 1

// Create the provider and attach the custom delegate object
// used by the app to respond to updates.
callProvider = CXProvider(configuration: config)
callProvider?.setDelegate(callManager, queue: nil)
```

각각의 `CXProvider` 객체는 앱 내 통화 서비스의 단일 인스턴스를 나타내고 시스템과의 상호작용을 가능케 합니다. 
앱들은 현재 사용자에 대한 전화통화들을 관리하는데에 오직 하나의 `CXProvider` 객체만이 필요합니다.  
만약 다수의 사용자들 또는 한 명의 사용자에 대해 여러개의 계정을 허용하는 서비스를 제공하고 있다면 걸려오는 전화를 알맞는 사용자 계정으로 연결시켜줘야 합니다.

## 서버에서 푸시 알림 생성하기

서버는 사용자간의 연결을 요구하는 통화 관련 작업의 대부분을 처리합니다. 
사용자가 앱을 열때마다, 해당 앱과 당신의 서버 간의 연결고리를 생성해야 합니다. 
사용자가 통화를 시작할 때, 이 연결고리를 사용하여 당신의 서버에 대한 콜백의 세부사항을 주고 받을 수 있습니다. 
그 다음에 서버는 반드시 발신자(Call Initiator)와 수신자(Call Recipient) 간의 연결을 시도해야 합니다. 
만약 전화 받는 사람의 앱이 돌아가고 있고 서버와의 연결이 활성화 되어 있다면 그 연결을 통해 직접적으로 앱과 서버가 소통할 수 있습니다. 
만약 앱이 꺼져있다면, 푸시 알림을 생성해서 해당 앱을 깨워야 합니다. 
각각의 푸시 알림은 디바이스 토큰(Device Token)과 통화 세부 정보가 담긴 페이로드(Payload)를 포함합니다.
당신은 이 정보들을 HTTP 요청(Request)로 묶어 APNs(Apple Push Notification service)로 보내도록 합니다.
푸시킷은 설정시간 때 디바이스 토큰을 제공하고 발급된 토큰은 사용자 디바이스의 타켓 주소(Target Address)로 사용됩니다.

VoIP 푸시 알림을 위한 HTTP 요청을 설정할 때 항상 아래의 정보에 따라 설정해야 합니다.
- `apns-expiration` 헤더 필드의 값을 `0` 또는 몇 초 정도로만 설정합니다. 
이는 시스템에 알림이 너무 늦게 전달되는 것을 방지합니다.
- 걸려오는 전화에 대한 정보를 `JSON` 페이로드에 담도록 합니다. 
예를 들어, 서버가 콜을 추적할 때 사용하는 콜 ID를 포함시켜 당신의 앱이 이 ID를 사용해 나중에 서버에서 확인할 수 있도록 해야합니다. 
그리고 걸려오는 전화에 대한 UI를 띄우기 위한 발신자에 대한 정보도 포함시킬 수 있습니다.

> **Note** 커스텀 서버가 **푸시 알림을 생성**하도록 설정하는 것에 대해 더 알고 싶다면 [Setting Up a Remote Notification Server](https://developer.apple.com/documentation/usernotifications/setting_up_a_remote_notification_server)  를, **푸시****알림을****보내는****방법**에 대해 더 알고 싶다면  [Sending Notification Requests to APNs](https://developer.apple.com/documentation/usernotifications/setting_up_a_remote_notification_server/sending_notification_requests_to_apns)를 참고하세요.

## 앱에서 VoIP 푸시 알림 대응하기

사용자들 중 한명이 전화를 걸 때, 서버는 반드시 전화 받는 사람의 앱에 연결되어 있어야 합니다. 
만약 서버가 해당 앱과의 활성화된 네트워크 연결이 없을 경우, 푸시 알림을 보내어 앱에 체크인을 요청할 수 있습니다. 
서버에서는 푸시 알림 요청을 구축하고 APNs 로 그 요청을 보내서 사용자의 디바이스까지 전달할 수 있습니다.
VoIP 푸시 알림을 위해서 시스템은 앱을 깨우거나 실행시키고 `PKPushRegistryDelegate` 의 `pushRegistry(_:didReceiveIncomingPushWith:for:completion:)` 메소드를 호출 시켜 그 알림을 앱 내의 `PKPushRegistry`객체에게 전달해야 합니다.
이 메소드를 통해 걸려오는 전화에 대한 UI가 보여지고 당신의 VoIP 서버와 연결이 성사됩니다.

아래의 예제 코드는 `pushRegistry(_:didReceiveIncomingPushWith:for:completion:)` 내에서 어떻게 VoIP 푸시 알림을 처리하는지 보여줍니다.

(1)푸시 알림의 페이로드 딕셔너리(Payload Dictionary)에서 통화 데이터를 추출한 다음 
(2)`CXCallUpdate` 객체를 생성하고 
(3)이 객체를 `CXProvider`의 `reportNewIncomingCall(with:update:completion:)` 메소드로 전달합니다. 콜킷이 해당 요청을 처리하는 동안 앞 작업과 
(4)**평행하게** VoIP 서버에 연결을 시켜줍니다. 
문제가 발생하면 언제든지 나중에 콜킷에 알릴 수 있습니다. 만약 
(5)콜킷이 성공적으로 콜을 다루면, `Completion Handler`가 앱 내에서 해당 콜을 관리할 수 있는 커스텀 데이터 구조들을 생성다.

```swift
// MARK: - VoIP 푸시 알림 대응하기
func pushRegistry(_ registry: PKPushRegistry, didReceiveIncomingPushWith payload: PKPushPayload, for type: PKPushType, completion: @escaping () -> Void) {
   if type == .voIP {
      // 푸시 알림 페이로드에서 콜 정보 추출
      if let handle = payload.dictionaryPayload[“handle”] as? String,
            let uuidString = payload.dictionaryPayload[“callUUID”] as? String,
            let callUUID = UUID(uuidString: uuidString) {

         // 콜 정보 데이터 구조들 설정
         let callUpdate = CXCallUpdate()
         let phoneNumber = CXHandle(type: .phoneNumber, value: handle)
         callUpdate.remoteHandle = phoneNumber
                
         // 콜킷에 해당 콜 리포트 및 콜 UI 보여주기
         callProvider?.reportNewIncomingCall(with: callUUID, update: callUpdate, completion: { (error) in
            if error == nil {
               // 시스템에서 해당 콜이 진행되도록 허용한다면, 해당 콜에 대한 데이터 기록을 생성
               let newCall = VoipCall(callUUID, phoneNumber: phoneNumber)
               self.callManager.addCall(newCall)
            }
            
            // 푸시킷에 해당 푸시알림이 처리 되었다는 것을 알림
            completion()
         })    
         
         // 비동기로(평행하게) 통화 서버에 등록하고 콜 처리. 
         // 필요시 콜킷에 업데이트 사항들을 리포트
         establishConnection(for: callUUID)
      }
   }
}
```

