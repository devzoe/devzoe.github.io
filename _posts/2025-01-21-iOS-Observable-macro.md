---
title: "Observable Object protocol에서 Observable macro로 마이그레이션하는 것이 꼭 필요할까?"
date: 2025-01-21 00:00:00 +0900
categories:
- iOS
tags:
- iOS
- SwiftUI
- Combine
- Observable
---

“Combine을 주로 사용하시는데 Observable Object와 Observable macro의 차이점이 무엇인가요?”

최근 모 연합 동아리 면접을 보면서 이런 질문을 받았습니다.

잠시 머릿속이 백지 상태가 되고… 

정확한 답변을 하지 못한 채, 면접이 마무리가 되고… 해당 내용으로 블로그 포스팅을 해야겠다고 생각하여 정리하게 되었습니다…^^;


## ObservableObject

저는 지금까지 ViewModel 클래스에서 ObservableObject 프로토콜을 채택하여 @Published 를 사용해 객체나 속성의 상태 관리를 했습니다. View에서 해당 ViewModel을 StateObject, ObservedObject로 선언하고, ViewModel의 데이터를 UI에 바로 반영하는 구조를 많이 사용했습니다.


그렇다면 ObservableObject는 왜 사용하는 것일까요? 공식 문서를 보면서 알아봅시다.

```swift
protocol ObservableObject: AnyObject

@propertyWrapper
struct Published<Value>
```

공식 문서에 따르면 ObservableObject는 프로토콜로 AnyObject를 채택하고 있습니다.

ObservableObject 프로토콜을 채택하고 있는 클래스 인스턴스를 관찰하다가 @Publishd로 선언된 어떠한 값이 변경되면 objectWillChange.send()를 호출하여 구독자에게 데이터 변경을 방출하는 방식으로 동작합니다. Published를 안 붙이면  objectWillChange.send()를 직접 호출해서 구독자에게 알려줘야 합니다. 

정리하자면, ObservableObject를 채택하고 있는 클래스 인스턴스를 만들면 해당 클래스에 선언된 데이터의 상태 변화를 즉각적으로 알 수 있기 때문에  ObservableObject를 사용한다고 볼 수 있습니다. 

어떤 구조로 ObservableObject를 사용하는지 간단한 메모앱을 만든다고 가정하고 만들어봅시다.

```swift
struct Memo {
    var title: String
    var text: String
    var writer: String
    var date: Date
    init() {
        self.title = "제목"
        self.text = "내용"
        self.writer = "작성자"
        self.date = Date()
    }
}

class MemoViewModel: ObservableObject {
    @Published var memo: Memo = Memo()
    
    func saveMemo() {
        //memo를 서버로 보냄
    }
}

struct MemoView: View {
    @StateObject var viewModel = MemoViewModel()
    var body: some View {
        VStack {
            TextField("Title", text: $viewModel.memo.title)
            TextEditor(text: $viewModel.memo.text)
            Text("\(viewModel.memo.writer)")
            Button {
                viewModel.saveMemo()
            } label: {
                Text("저장하기")
            }
        }
    }
}
```

새로운 메모를 작성하고, 저장하는 로직이라고 합시다. 메모에는 <제목, 내용, 작성자, 작성일>의 데이터가 있습니다.

먼저, 데이터 관리를 위해 메모 구조체로 Entity를 만듭니다. 

해당 데이터와 UI를 연결하는 ViewModel인 MemoViewModel을 클래스로 만들고, ObservableObject 프로토콜을 채택합니다. 그리고 상태 변화 관찰을 위해 @Published로 메모 인스턴스를 만듭니다. 

UI를 위해 MemoView를 만들고, 데이터를 UI에 뿌리고 뷰 업데이트를 위해 memoViewModel을 [StateObject](https://developer.apple.com/documentation/swiftui/stateobject#overview)로 선언합니다. MemoView에서는 제목, 내용을 각각 텍스트필드, 텍스트에디터로 수정할 수 있습니다. 메모를 다 작성한 후, 저장하기 버튼을 클릭하면 memo 객체를 서버로 보내는 saveMemo()가 동작합니다. (이는 간단한 예시로 실제로는 네트워크 레이어를 나눠서 작성할 수 있습니다.)

## Observable macro

그런데 WWDC23에서 [Observation](https://developer.apple.com/documentation/observation)이 발표되고, [Observable macro](https://developer.apple.com/videos/play/wwdc2023/10149/)가 나타나면서 기존 [ObservableObject를 마이그레이션하는 글](https://developer.apple.com/documentation/swiftui/migrating-from-the-observable-object-protocol-to-the-observable-macro)도 애플에서 공개했습니다. 

그렇다면 Observable macro에서 어떤 점이 기존과 달라졌을까요? (기존보다 더 나은 점이 있기에 발표된 것일테니 장점 위주로 살펴봅시다.)

```swift
@attached(member, names: named(_$observationRegistrar), named(access), named(withMutation)) 
@attached(memberAttribute) 
@attached(extension, conformances: Observable) 
macro Observable()
```

먼저 공식 문서에 나와있는 Observable macro의 정의입니다.

간단히 해석하자면 “`@Observable` 이 붙은 타입에 대해, 자동으로 Observation 로직을 구성해주는 멤버들과 extension을 주입할 것이다”라는 의미입니다. 덕분에 매크로 사용 시 @Published 없이 관찰 가능한 타입을 만들 수 있게 되었습니다.

또한, Observable macro는 해당 인스턴스를 가지고 있는 뷰가 있다고 할 때, 해당 뷰에서 참조하고 있는 데이터가 변화할 때만 뷰를 다시 그린다는 장점이 있습니다.

ObservableObject에서는 해당 뷰가 참조하고 있지 않은 속성이 변화할 때도 뷰가 다시 그려지기 때문에 성능에 영향을 미칠 수 있다는 단점이 있었습니다.

이게 무슨 소리인지 메모 코드를 Observable macro로 마이그레이션 해보면서 설명드리겠습니다.

```swift
@Observable class MemoViewModel {
    var memo: Memo = Memo()
    
    func saveMemo() {
        //memo를 서버로 보냄
    }
}

struct MemoView: View {
    @State var viewModel = MemoViewModel()
    var body: some View {
        VStack {
            TextField("Title", text: $viewModel.memo.title)
            TextEditor(text: $viewModel.memo.text)
            Text("\(viewModel.memo.writer)")
            Button {
                viewModel.saveMemo()
            } label: {
                Text("저장하기")
            }
        }
    }
}

```

기존 코드에서 3가지가 달라졌습니다.

- ObservableObject 채택 → @Observable 추가
- @Published 삭제
- @StateObject → @State로 변경

이 외에도 바뀐 점은 [해당 글](https://developer.apple.com/documentation/swiftui/migrating-from-the-observable-object-protocol-to-the-observable-macro)을 확인해주세요.

동작에서 크게 바뀐 점은 ObservableObject를 채택한 클래스 인스턴스에서 memo의 date 값이 다른 곳에서 변경되면 MemoView가 다시 로드되지만, Observable macro에서는 date값이 변경된다고 해도 MemoView가 다시 로드되지 않는다는 점입니다. 만약 데이터가 다른 곳에서 수시로 변화하고, 해당 뷰에서는 참조하고 있지 않을 때, 화면이 다시 로드되는 불필요한 작업이 발생하지 않으므로 기존 동작 방식보다 앱 성능이 더 좋다고 말할 수 있습니다.

그렇다면 가장 중요한 포인트는 “기존 ObservableObject로 구현했던 코드를 반드시 Observable macro로 마이그레이션해야하나?” 입니다. 사실 “반드시”라고 붙는 문장의 대부분은 반드시 실행할 필요가 없는 것이 대다수인데요. 제가 생각하는 반드시 바꿔야한다의 기준은 1. deprecated 된다는 공지가 있다. 2. 기존 방식보다 사용자가 느낄 수 있는 정도로 성능이 올라간다. 입니다.

ObservableObject는 deprecated 된다는 공지가 없긴 하지만, 뷰의 불필요한 재로드가 발생한다는 단점이 있으므로 Observable macro로 마이그레이션하는 것이 장기적으로 좋을 것 같다는 것이 저의 의견입니다. 그리고 애플이 같은 동작, 다른 매커니즘을 내놓았다는 것은 지속적으로 업데이트되는 것은 뒤에 발표된 것이라는 의미이기도 하기에 Observable macro를 사용하는 것이 좋을 것 같다는 의견입니다.