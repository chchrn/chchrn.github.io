---
layout: post
title:  "Переезд на frameworks"
date:   2020-12-03
---

Долгое время у нас в проектах был общий код, который шарился между несколькими проектами. Всё время этот код был просто свалкой файлов, который мы подключали через git submodule в основной проект. Минусы такого подхода очевидны:

1) тесты общего кода жили гдето в целевом проекте

2) данный код толком нельзя было проверить при коммите в репу, потому что если код собирается в одном проекте - не факт что он соберется в другом. И мы на это натыкались каждый раз

Короче всё требовало создать отдельный framework, который уже будет шариться между проектами. 

После того, когда проект для framework-а был готов и внедрен в проект целевого приложения наступил этап, когда нужно все импорты Objective-C заменить с вида

```objectivec
#import "MyClass.h"
```

на вид

```objectivec
#import <MyLib/MyClass.h>
```

поэтому я написал несложный [скрипт](https://gist.github.com/chchrn/8ed1cf3ee02825310f23adf47a9d8310), который именно это и делает. 

Работает не очень быстро и скорее всего можно было что-то заменить в нем (а может и AppCode такое умеет), но в данный момент поcчитал что так будет проще, 5 минут работы и импорты обновлены 😂

На самом деле позже мы импортнули этот framework `@import Framework` и в этом тоже помог скрипт. 

Проектов то несколько и делать однотипные действия совсем не хотелось. 

[Скрипт на gist](https://gist.github.com/chchrn/8ed1cf3ee02825310f23adf47a9d8310) 

#### Проблемы

**Framework Search Path**

Код все так же мы подключаем через git submodules, но теперь вместо файлов в целевой проект включаем проект framework-а. И конечно у него есть зависимости от других библиотек (например таких как google/Promises), чтобы зависимости фреймворка слинковались из основного проекта пришлось прописать  `Framework Search Path` в проекте фреймворка. Выглядит как костыль, но как решить красиво не понимаю. Единственный выход - развивать библиотеку отдельно от основного кода и сначала разрабатывать библиотеку/писать тесты, выпускать новую версию и интегрировать в целевое приложение. Но в активной фазе разработки это сложно

**Configurations**
Была проблема в том, что в основном приложении есть Configurations отличные то Debug и Release, опытным путем выяснилось, что если проект подключен как подпроект, то в нем должны быть такие же схемы. Пришлось завести с либе схемы такие же как в основных приложениях


#### Refs


[https://stackoverflow.com/questions/34681435/how-to-add-a-framework-inside-another-framework-umbrella-framework/37553768](https://stackoverflow.com/questions/34681435/how-to-add-a-framework-inside-another-framework-umbrella-framework/37553768)

[https://stackoverflow.com/questions/42771940/how-to-create-dynamic-framework-which-uses-another-framework](https://stackoverflow.com/questions/42771940/how-to-create-dynamic-framework-which-uses-another-framework)

[https://stackoverflow.com/questions/51501424/creating-swift-framework-that-depends-on-another-framework-objc](https://stackoverflow.com/questions/51501424/creating-swift-framework-that-depends-on-another-framework-objc)

[https://medium.com/freelancer-engineering/modular-architecture-on-ios-and-how-i-decreased-build-time-by-50-23c7666c6d2f](https://medium.com/freelancer-engineering/modular-architecture-on-ios-and-how-i-decreased-build-time-by-50-23c7666c6d2f)

[https://tech.olx.com/modular-architecture-in-ios-c1a1e3bff8e9](https://tech.olx.com/modular-architecture-in-ios-c1a1e3bff8e9)
