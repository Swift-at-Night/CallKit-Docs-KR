# VoIP 가장 좋은 예시 (VoIP Best Practice)

![PushKit](https://img.shields.io/badge/PushKit-67c7c3?style=for-the-badge&logo=swift&logoColor=white)

VoIP 앱은 기기의 셀룰러 서비스가 대신 인터넷 연결을 사용하여 사용자가 전화를 걸고 받을 수 있도록 합니다.
VoIP 앱은 네트워크에 의존하기 때문에 전화를 거는 동안 높은 에너지 사용량을 보입니다. 
그리고 사용하지 않을 때는 에너지 절약을 위해 완전히 유휴(Idle) 상태이어야 합니다.

## VoIP 푸시 알림 사용하여 지속적인 연결 피하기

과거에 VoIP앱은 전화나 다른 데이터를 받기 위해 항상 서버와의 네트워크 연결을 유지했어야 했습니다. 
즉, 복잡한 코드를 작성하여 서버와 앱 간의 연결고리가 끊어지지 않게 주기적으로 메세지를 주고 받도록 해야 했습니다.
심지어 앱이 사용중이 아니어도 그래야 했습니다. 
이러한 기술은 기기가 자주 깨어나게 하여 에너지 낭비를 야기했습니다.
그리고 사용자가 VoIP 앱을 종료하면 더이상 서버로 부터 아무것도 받을 수 없었습니다.

이러한 지속적인 연결 대신에 개발자들은 반드시 푸시킷(`PushKit`) API 프레임워크를 사용해야 합니다.
이 프레임워크는 앱이 원격 서버로 부터 푸시(데이터를 사용할 수 있을 때의 알림)를 받을 수 있게 합니다.
푸시를 받을 때마다 앱이 작동하도록 관련 메소드가 호출됩니다.
예를 들어, VoIP 앱은 전화가 오면 알림창을 보여줄 수 있고 해당 전화를 수락(Accept)할 지 거절(Reject)할 지에 대한 옵션을 제공합니다. 
사용자가 수락을 한 경우 해당 콜을 시작하기 위한 몇가지 사전 단계들을 시작합니다.

VoIP 푸시를 받기 위해 푸시킷을 사용하는 것은 아래와 같은 많은 이점을 갖고 있습니다.
- VoIP 푸시가 발생했을 때만 앱을 깨워 에너지를 절약할 수 있음.
- 앱이 동작을 수행하기 전에 반드시 사용자 응답을 필요로 하는 일반(Standard) 푸시 알림과는 달리 VoIP 푸시는 처리를 위해 바로 앱으로 직행.
- VoIP 푸시는 높은 우선 순위를 갖는 알림이며 딜레이 없이 바로 전달.
- VoIP 푸시는 일반 푸시 알림보다 더 많은 데이터를 갖고 있을 수 있음.
- 앱이 돌고 있지 않을 때 VoIP 푸시를 받으면 앱이 자동으로 다시 시작.
- 앱에 푸시를 처리하기 위한 런타임이 주어짐. 백그라운드 상태에서도 상관없이 해당.

> **Note** 푸시킷의 VoIP 지원은 iOS 8 이상부터 가능하다.  

## VoIP 푸시 알림을 받기 위한 준비

백그라운드 동작을 지원하는 모든 앱과 같이, 당신의 VoIP 앱은 **반드시 백그라운드 모드를 활성화 해야합니다**.
**Xcode Project** > **Capabilities pane** 에서 아래 사진과 같이 **Background Modes**를 활성화 하고 **Voice over IP** 항목을 체크하도록 합니다.

그리고 반드시 VoIP 앱을 위한 **인증서**(certificate)를 발급받아야 합니다.
각각의 VoIP 앱은 각각의 앱 ID와 매핑되는 개별의 VoIP Services 인증서를 필요로 합니다. 
이 인증서는 당신의 알림 서버가 VoIP 서비스에 연결될 수 있도록 합니다.
[애플 개발자 멤버 센터](https://developer.apple.com/certificates) (Apple Developer Member Center)에 가서 아래와 같이 새 **VoIP Services Certificate** 를 발급 받도록 합니다. 
그런 다음 발급 받은 인증서를 다운 받고 **키체인**(Keychain Access app)에 불러오도록 합니다(Import).

## VoIP 푸시 알림 설정하기

당신의 앱이 VoIP 푸시 알림을 받을 수 있도록 설정하려면 
(1)`AppDelegate`(또는 앱 내의 다른 장소)를 푸시킷 프레임워크에 연결해야 합니다. 그런 다음 
(2)`PKPushRegistry` 객체를 생성하고 
(3)그것의 `delegate`를 `self`로 설정합니다. 그리고 
(4)VoIP 푸시를 받을 수 있도록 등록합니다.

```swift
// MARK: - VoIP 푸시 알림 설정하기
// 푸시킷 프레임워크에 연결
import PushKit
	
// 실행할 때 VoIP 등록
func application(application: UIApplication, didFinishLaunchingWithOptions launchOptions: NSDictionary?) -> Bool {
    self.voipRegistration()
    return true
}

// VoIP 알림 등록
func voipRegistration {
    let mainQueue = dispatch_get_main_queue()
    // 푸시 레지스트리 객체 생성
    let voipRegistry: PKPushRegistry = PKPushRegistry(mainQueue)
    // 레지스트리의 delegate에 self를 설정
    voipRegistry.delegate = self
    // 푸시타입을 VoIP로 설정
    voipRegistry.desiredPushTypes = [PKPushTypeVoIP]
}
```

그런 다음, 업데이트 된 **푸시 크레덴셜**(Credentials)을 다루기 위한 delegate 메소드를 구현해줘야 합니다. 
만약 앱이 일반 푸시 알림과 VoIP 푸시 둘 다 받는다면, 앱은 서로 다른 두개의 푸시 토큰을 받게 됩니다.
두 토큰들 모두 해당 알림들을 받기 위해 반드시 해당 서버로 잘 전달 되어야 합니다.

```swift
// MARK: - 업데이트 된 푸시 크레덴셜 다루기
func pushRegistry(registry: PKPushRegistry!, didUpdatePushCredentials credentials: PKPushCredentials!, forType type: String!) {
    // VoIP 푸시토큰(푸시 크레덴셜의 프로퍼티)를 서버에 등록
}
```

마지막으로 푸시를 처리하기 위해 delegate 메소드를 설정해야 합니다. 앱이 돌고 있지 않을 때 푸시를 받으면 앱이 자동으로 시작될 것 입니다.

```swift
// MARK: -들어오는 푸시 다루기
func pushRegistry(registry: PKPushRegistry!, didReceiveIncomingPushWithPayload payload: PKPushPayload!, forType type: String!) {
    // 받은 푸시 처리하기
}
```

VoIP 푸시 알림에 대해 더 알고 싶다면  [PushKit Framework Reference](https://developer.apple.com/documentation/pushkit) 를 참고하세요.
