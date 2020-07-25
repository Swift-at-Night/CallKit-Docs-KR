# CallKit-Docs-KR

![CallKit](https://img.shields.io/badge/CallKit-66e268?style=for-the-badge&logo=swift&logoColor=white)

CallKit 애플 프레임워크에 관한 문서를 한글로 번역 및 정리한 내용입니다.

여러분의 앱의 VoIP 서비스를 위해 시스템-통화 UI를 띄어주고 다른 앱 및 시스템과 통화 서비스를 조정(Coordinate)합니다.

## 개요

콜킷(`CallKit`)을 사용하면 사스템 상의 다른 통화 관련 앱과 여러분의 통화서비스를 접목시킬 수 있습니다.  콜킷은 통화 인터페이스를 제공하고, 여러분의 VoIP 서비스로 백엔드 커뮤니케이션을 다룰 수 있습니다.  오가는 전화를 위해, 콜킷은 네이티브 전화앱과 동일한 인터페이스를 띄웁니다. 그리고 콜킷은 “방해금지 모드”와 같은 시스템 레벨의 동작에 적절하게 대응합니다.
추가적으로,`Call Directory app extension`을 통해 여러분의 서비스와 관련된 차단 번호 목록과 Caller ID 정보를 제공할 수 있습니다.

## 전화받기
앱이 전화를 받을 수 있도록 설정하기 위해서 우선, `CXProvider` 객체를 생성하고 글로벌하게 접근할 수 있도록 저장해야 합니다. 푸시킷(`PushKit`)에 의해 생성된 VoIP 푸시 알림과 같은 외부 알림에 응답하기 위해 앱은 이 `CXProvider에게` 수신 전화(Incoming Call)를 리포트 하도록 합니다.

```swift
// 전화 알림받기
// MARK: PKPushRegistryDelegate
func pushRegistry(_ registry: PKPushRegistry, didReceiveIncomingPushWith payload: PKPushPayload, forType type: PKPushType) {
    // report new incoming call
}
```
> **NOTE** VoIP 푸시 알림과 푸시킷에 대해 더 알고 싶다면, Voice Over IP Best Practices 를 참고하세요.

외부 알림이 제공한 정보들을 통해 앱은 `UUID`와 `CXCallUpdate` 객체를 생성하여 해당 콜과 `caller`를 식별하도록 합니다. 그 다음 생성된 UUID와 CXCallUpdate 객체를 `reportNewIncomingCall(with:update:completion:)` 메소드를 통해 `CXProvider` 에게 전달하도록 합니다.
 
 ```swift
 // 걸려온 전화 다루기
 if let uuidString = payload.dictionaryPayload["UUID"] as? String,
    let identifier = payload.dictionaryPayload["identifier"] as? String,
    let uuid = UUID(uuidString: uuidString) {
    let update = CXCallUpdate()    
    update.callerIdentifier = identifier
    
    provider.reportNewIncomingCall(with: uuid, update: update) { error in
        // …
    }
}
 ```
 
그 다음에 해당 콜이 연결되면 시스템은 `provider(_:perform:)` 메소드를 호출합니다(`CXProviderDelegate`). 델리게이트 메소드의 구현부에 `AVAudioSession`을 설정하고 동작이 완료되면 `fulfill`를 호출합니다.

 ```swift
 // 통화를 위해 오디오 초기화 하기
 func provider(_ provider: CXProvider, perform action: CXAnswerCallAction) {
    // configure audio session
    action.fulfill()
}
 ```
 
 ## 전화 걸기
사용자는 아래의 3가지 방법으로 VoIP 앱을 통해 전화를 걸 수 있습니다. 
- 앱 내부에서 관련 기능을 통해서(앱 내의 전화걸기 버튼) 
- 지원되는 커스텀 URL 스킴을 통해 링크 열어서 ([Using URL Schemes to Communicate with Apps](https://developer.apple.com/library/archive/documentation/iPhone/Conceptual/iPhoneOSProgrammingGuide/Inter-AppCommunication/Inter-AppCommunication.html#//apple_ref/doc/uid/TP40007072-CH6-SW1) 참고)
- 시리를 통해서 ([INStartAudioCallIntentHandling](https://developer.apple.com/documentation/sirikit/instartaudiocallintenthandling) 프로토콜 참고)

  전화를 걸기 위해서 앱은 `CXCallController` 객체에서 `CXStartCallAction` 객체를 요청합니다. 이 액션은 `UUID` 와 `CXHandler`를 포함합니다. (1)`UUID`는 콜을 식별하기 위함이고 (2)`CXHandler` 객체는 수신인(recipient, callee)를 지정하기 위함 입니다.

```swift
// 전화 걸기
let uuid = UUID()
let handle = CXHandle(type: .emailAddress, value: "jappleseed@apple.com")
 
let startCallAction = CXStartCallAction(call: uuid)
startCallAction.destination = handle
 
let transaction = CXTransaction(action: startCallAction)
callController.request(transaction) { error in
    if let error = error {
        print("Error requesting transaction: \(error)")
    } else {
        print("Requested transaction successfully")
    }
}
```
 
상대가 전화를 받으면 (`answer`), 시스템은 `provider(_:perform:)` 메소드를 호출합니다(`CXProviderDelegate`). 구현부에 `AVAudioSession`을 설정하고 동작이 완료되면 `fulfill`를 호출합니다.
 
```swift
// 통화 초기화 하기
func provider(_ provider: CXProvider, perform action: CXAnswerCallAction) {
    // configure audio session
    action.fulfill()
}
```
 
 ## 통화 식별 및 차단하기
 
 앱에 `Call Directory app extension` 을 생성하여 전화번호를 통해 발신자를 식별하고 차단할 수 있습니다.
 
 > **NOTE** `Call Directory extension` 안의 전화번호들은 `CXCallDirectoryPhoneNumber` 타입으로 표현되고 국가번호 + 전화번호 형태로 구성되어 있습니다.
 
 ### Call Directory App Extension 생성하기
 
 Application Extensions 목록에 **Call Directory Extension**을 선택하여 새 프로젝트 타겟을 추가하면 앱에 `Call Directory extension`을 생성할 수 있습니다.
 전화 차단과 식별 두가지 모두 `CXCallDirectoryProvider` 서브클래스의 `beginRequest(with:)` 메소드의 구현부에서 설정할 수 있습니다. 이 메소드는 시스템이 앱 extension을 실행시킬 때 호출됩니다. App extension 들이 어떻게 동작하는 지 더 알아보려면 [App Extension Programming Guide](https://developer.apple.com/library/archive/documentation/General/Conceptual/ExtensibilityPG/index.html#//apple_ref/doc/uid/TP40014214) 을 참고하세요.

 ### 발신자 식별하기
 
 전화가 올 때, 시스템은 우선 매칭되는 전화 번호가 없는지 사용자의 연락처들을 확인합니다. 매칭 되는 번호가 없을 경우, 시스템은 전화번호를 식별하기 위해 매칭되는 항목(Entry)을 해당 앱의 Call Directory extension 에서 찾아봅니다. 이는 (1)사용자를 위해 시스템 연락처와 분리된 SNS같은 연락처 목록 또는 (2)배달 알림 이나 고객 서비스 지원과 같은 앱 내부에 설정되어 있는 수신전화를 식별하기 위한 연락처 목록을 유지한다는 점에서 매우 유용합니다.
 예를 들어, 어떤 사용자가 SNS 앱 상에서 Jane 이라는 사람과 친구지만 실제 그녀의 전화번화를 갖고 있지 않다고 가정해보겠습니다. 연락처가 없음에도 불구하고 그 사용자의 아이폰은 Jane으로 부터 전화가 왔을 때, 시스템을 통해 “Unknown Caller” 가 아닌 “(앱 이름) Caller ID: Jane Appleseed”를 띄어줄 수 있습니다. 왜냐하면 그 SNS 앱은 Call Directory app extension을 갖고 있어서 사용자의 모든 친구들의 전화번호를 다운로드 받고 추가할 수 있기 때문입니다. 발신자(incoming caller)들의 정보를 식별하도록 하기 위해서는 `beginRequest(with:)` 의 구현부 안에 `addIdentificationEntry(withNextSequentialPhoneNumber:label:)` 메소드를 사용해야 합니다.

```swift
// 발신자 정보 식별하기
class CustomCallDirectoryProvider: CXCallDirectoryProvider {
    override func beginRequest(with context: CXCallDirectoryExtensionContext) {
        let labelsKeyedByPhoneNumber: [CXCallDirectoryPhoneNumber: String] = [ … ]

        for (phoneNumber, label) in labelsKeyedByPhoneNumber.sorted(by: <) {
            context.addIdentificationEntry(withNextSequentialPhoneNumber: phoneNumber, 
                                                            label: label)        
        }

        context.completeRequest()
    }
}
```

이 메소드는 오직 시스템이 앱 extension을 실행했을 때만 호출 되지, 각각의 개별 통화에 대해 호출되는 게 아니기 때문에 반드시 통화 식별 정보를 한번에 지정해야 합니다. 예를 들어 걸려오는 전화에 대한 정보를 찾기 위해 웹 서비스에 요청할 수 없습니다.

### 전화 차단하기

전화가 걸려오면 시스템은 우선 차단되어야 하는 전화인지 아닌지 확인하기 위해 사용자의 차단목록을 찾아 봅니다. 만약 해당 전화번호가 사용자 또는 시스템에 의해 등록된 차단 목록에 없다면, 시스템은 이번엔 앱의 Call Directory extension 을 수색합니다. 이는 알려진 판매원(Solicitor)들의 데이터 베이스를 유지하거나 어떤 기준들에 매칭되는 전화 번호들을 차단할 수 있도록 해준다는 점에서 유용합니다. 특정 전화번호를 차단하기 위해서는 `beginRequest(with:)` 의 구현부 안에 `addBlockinEntry(withNextSequentialPhoneNumber:)` 메소드를 사용해야 합니다.

> **NOTE** Call Directory app extension을 통해 `beginRequest(with:)`의 구현부 안에 식별자를 추가하거나 전화번호를 차단하도록 지정할 수 있습니다.

```swift
// 특정 번호의 전화 차단하기
class CustomCallDirectoryProvider: CXCallDirectoryProvider {
    override func beginRequest(with context: CXCallDirectoryExtensionContext) {
        let blockedPhoneNumbers: [CXCallDirectoryPhoneNumber] = [ … ]
        for phoneNumber in blockedPhoneNumbers.sorted(by: <) {
            context.addBlockingEntry(withNextSequentialPhoneNumber: phoneNumber)
        }
        
        context.completeRequest()
    }
}
```
