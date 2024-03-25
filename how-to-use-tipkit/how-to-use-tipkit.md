### Содержание

* [Вступление](#intro)
* [Базовая настройка](#base)
* [Добавляем View](#addView)
    * [Встраиваемые](#inline)
    * [Всплывающие](#popover)
* [Добавляем кнопки в подсказку(есть видео)](#addButtons)
* [Закрытие подсказки](#closeTip)
* [Правила(есть видео)](#rules)
    * [Добавляем правило](#addRules)
    * [Добавляем View](#addViewRules)
* [Preview](#preview)

---

[TipKit](https://developer.apple.com/documentation/tipkit) - <span id="intro">позволяет легко отображать подсказки в приложениях. Появился в iOS 17 и доступен для iPhone, iPad, Mac, Apple Watch и Apple TV.</span>

![](tipkit-example.png)

# <p id="base">Базовая настройка</p>

В точке входа приложения импортируем `TipKit` и добавляем `Tips.configure`.

```swift
import SwiftUI
import TipKit

@main
struct TipKitExampleApp: App {
    var body: some Scene {
        WindowGroup {
            TipKitDemo()
                .task {
                    try? Tips.configure([
                        .displayFrequency(.immediate),
                        .datastoreLocation(.applicationDefault)
                    ])
                }
        }
    }
}
```

`Tips.configure()` - конфигурация состояния всех подсказок в приложении.

`displayFrequency` - частота отображения подсказки: 
<ul>
    <li>immediate - будут отображаться сразу</li>
    <li>hourly - ежечасно</li>
    <li>daily - ежедневно</li>
    <li>weekle - еженедельно</li>
    <li>monthly - ежемесячно</li>
</ul>

`datastoreLocation` - расположение хранилища данных, по умолчанию является каталогом `support`.

# <p id="protocol">Реализуем протокол</p>

Чтобы создать подсказку нужно принять протокол Tip, этот протокол определяет содержание и условия в подсказке. Подсказка состоит из обязательного поля `title` и опциональных `message` и `image`.

```swift
struct InlineTip: Tip {
    var title: Text {
        Text("Для начала")
    }

    var message: Text? {
        Text("Проведите пальцем влево/вправо для навигации. Коснитесь фотографии, чтобы просмотреть ее детали.")
    }

    var image: Image? {
        Image(systemName: "hand.draw")
    }
}
```

# <p id="addView">Добовляем View</p>

Подсказки бывают двух видов:

### <p id="inline">Встраиваемые</p>

Временно перестраивает интерфейс вокруг себя, чтобы ничего не было перекрыто (недоступно в tvOS).

Нужно создать экземпляр `TipView` и передать ему подсказку для отображения.

```swift
struct TipKitDemo: View {

    private let inlineTip = InlineTip()

    var body: some View {
        VStack {
            Image("pug")
                .resizable()
                .scaledToFit()
                .clipShape(RoundedRectangle(cornerRadius: 12))
            TipView(inlineTip) // Inline Tip
        }
        .padding()
    }
}
```

![](tipkit-inline-code.png)

Так же можно указать `arrowEdge` - напраление стрелочки подсказки.

```swift
    TipView(inlineTip, arrowEdge: .top)
    TipView(inlineTip, arrowEdge: .leading)
    TipView(inlineTip, arrowEdge: .trailing)
    TipView(inlineTip, arrowEdge: .bottom)
```

![](arrow-edge.png)

### <p id="popover">Всплывающие</p>

Отображаются в виде наложения, не меняя представления.

Нужно прикрепить модификатор `popoverTip` кнопке или другим элементам интерфейса.

```swift
struct TipKitDemo: View {
    
    private let popoverTip = PopoverTip()
    
    var body: some View {
        HStack {
            Image(systemName: "heart")
                .font(.largeTitle)
                .popoverTip(popoverTip, arrowEdge: .bottom) // Popover Tip
        }
        
    }
}
```
![](tipkit-popover-code.png)

# <p id="addButtons">Добавляем кнопоки в подсказку</p>

Чтобы подсказки были более интерактивными и имели больше возможностей, в протокол нужно добавить поле `actions`:

```swift
var actions: [Action] {
    Action(id: "reset-password", title: "Сбросить Пароль")
    Action(id: "not-reset-password", title: "Отменить сброс")
}
```

В замыкании `TipView`, по `action.id` мы обращаемся к нашим `actions`.

```swift
struct ActionTip: View {
    @State private var textFieldData = ""
    @State private var isShowRessetField = false
    
    private var tip = PasswordTip()
    
    var body: some View {
        VStack(spacing: 20) {
            Spacer()
            TipView(tip, arrowEdge: .bottom) { action in
                
                if action.id == "reset-password" {
                    isShowRessetField = true
                }
                
                if action.id == "not-reset-password" {
                    isShowRessetField = false
                }
                
            }
            Button("Войти") {}
            TextField("Введите новый пароль", text: $textFieldData)
                .textFieldStyle(.roundedBorder)
                .opacity(isShowRessetField ? 1 : 0)
            Spacer()
        }
        .padding()
    }
}
```
![](action-tipkit.mp4)
<video src="action-tipkit.mp4" controls></video>

# <p id="closeTip">Закрытие подсказки</p>

Подсказку можно закрыть, нажав на крестик или закрыть кодом , используя метод `invalidate`, который отменяет подсказку и не позводяет ей отображаться.

```swift
inlineTip.invalidate(reason: .actionPerformed)
```

`.actionPerformed` - пользователь выполнил действие, описанное в подсказке.

`.displayCountExceeded` - подсказка показана максимальное количество раз.

`.actionPerformed` - пользователь явное закрыл подсказку.


# <p id="rules">Правила</p>

TipKit позволяет создавать отдельные правила `rules` для каждой подсказки. 

Правила позволяют использовать условия на основе состояний `@Parameter` или пользовательских событий.

### <p id="addRules">Добавляем правило</p>

Параметр `hasViewedGetStartedTip` со значением **false**

`Rule` проверяет значение переменной `hasViewedGetStartedTip`, когда значение равно true, подсказка должна отображаться.

```swift
struct FavoriteRuleTip: Tip {

    var title: Text {
        Text("Добавить в избранное")
    }
    
    var message: Text? {
        Text("Этот пользователь будет добавлен в папку избранное.")
    }

    var rules: [Rule] {
        #Rule(Self.$hasViewedGetStartedTip) { $0 == true }
    }

    @Parameter
    static var hasViewedGetStartedTip: Bool = false
}
```

### <p id="addViewRules">Добавляем View</p>

```swift
struct ParameterRule: View {
    @State private var showDetail = false
    
    private var favoriteRuleTip = FavoriteRuleTip()
    private var gettingStartedTip = GettingStartedTip()
    
    var body: some View {
        VStack {
            ZStack(alignment: .topTrailing) {
                Image("pug")
                    .resizable()
                    .scaledToFit()
                    .clipShape(RoundedRectangle(cornerRadius: 14))
                Image(systemName: "heart")
                    .padding()
                    .font(.largeTitle)
                    .foregroundStyle(.white)
                    .popoverTip(favoriteRuleTip, arrowEdge: .top)
            }
            .onTapGesture {
                withAnimation {
                    showDetail.toggle()
                }
                
                // пользователь выполнил действие описанное в подсказке, отключаем подсказку
                gettingStartedTip.invalidate(reason: .actionPerformed)
                
                FavoriteRuleTip.hasViewedGetStartedTip = true
            }
            if showDetail {
                Text("Мопс - приземистая собака с почти квадратным телом и приплюснутой мордочкой с обилием складок. У нее широко расставленные круглые умные глаза и характерная походка.")
            } else {
                TipView(GettingStartedTip())
            }
            Spacer()
        }
        .padding()
    }
}
```
<video src="rules-tipkit.mp4" controls></video>

# <p id="preview">Preview</p>

Если закрыть подсказку, то она больше не будет отображаться в приложении. Поэтому для preview нужно сбросить хранилище данных подсказок `Tips.resetDatastore()`: 

```swift
#Preview {
    TipKitDemo()
        .task {
            try? Tips.resetDatastore()
            
            try? Tips.configure([
                .displayFrequency(.immediate),
                .datastoreLocation(.applicationDefault)
            ])
        }
}
```