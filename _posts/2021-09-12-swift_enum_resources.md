---
layout: post
title:  "Swift Enum - правильное использование ресурсов"
date:   2021-09-12
---

Часто для значений из Enum-а требуется сопоставить или локализованную строку, чтобы показать на экране пользователю или иконку/картинку.   
Встречаю такой код:

```swift
enum LayerType {
	case wind
  case swell
}

// View слой (ViewController или Presentation слой)
var title = ""
switch layerType {
   case .wind: 
       title = NSLocalizedString("layerType_wind", comment: "")
   case .swell: 
       title = NSLocalizedString("layerType_swell", comment: "")
}
```

Такой код становится сложно поддерживать, потому что при добавлении case-ов в Enum потребуется изменять код во всех местах где используется Enum. То есть ходить по всем ViewController или View и менять Switch. 

Код с зависимостью от `rawValue` еще хуже:

```swift
enum LayerType: String {
	case wind = "layerType_wind"
  case swell = "layerType_swell"
}

// View слой (ViewController или Presentation слой)
var title = NSLocalizedString(layerType.rawValue, comment: "")
```

Догадаться, что где-то во View-слое используется `rawValue` невозможно. К тому же при таком подходе не будет ошибки компиляции.

Считаю, что лучше сделать отдельный Extension для Enum-а и все связанные ресурсы объявлять там. Например, вот так

```swift
enum LayerType: String {
    case wind
    case swell
}

extension LayerType {
    var localizedTitle: String {
        let map: [LayerType: String] = [
            .wind: "layerType_wind",
            .swell: "layerType_swell"
        ]
        guard let locKey = map[self] else {return "Undefined"}
        return NSLocalizedString(locKey, comment: "")
    }
}

// View слой (ViewController или Presentation слой)
let layerType = LayerType.wind
var title = layerType.localizedTitle
```

Таким образом все локализации и связанные картинки, видео будут объявляться в одном месте.