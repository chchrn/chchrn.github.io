---
layout: post
title:  "Логирование"
date:   2020-03-8
---

При разработке любого приложения возникает потребность выводить отладочные логи в консоль. Это могут быть ошибки или просто детали выполнения приложения. 

Часто в коде используют просто `NSLog`/`print()`, но такое решение далеко от идеального. Есть популярная библиотека CocoaLumberjack. Долгое время я использовал её, но всё логирование завязывается на статические методы или макросы, которые дергаются из любого метода/класса. 

Это распространенный подход, но совсем не объектно-ориентированный и не расширяемый. Чтобы разделить потоки логов от разных подсистем - придется както завязывать на константы в лог-сообщениях. Либо создавать новые макросы/статические методы.

Напрашивается решение - передавать некоторый объект через который будут логироваться все сообщения. Например, вот такой:

```swift
public protocol WlgLogEvent {
    var level: WlgLogLevel {get}
    var message: String {get}
}

public protocol WlgLog {
    func log<E: WlgLogEvent>(_ event: E)
}
```

Теперь некий абстрактный `WldLog` принимает абстрактные `WldLogEvent`. Подклассов для реализации конкретных ивентов можно придумать большое кол-во.

Самый простой - объект просто хранящий `level` и `message`

```swift
public class WlgLogEventDefault: WlgLogEvent {
    public let level: WlgLogLevel
    public let message: String

    public init(level: WlgLogLevel, message: String) {
        self.level = level
        self.message = message
    }
}
```

Теперь, чтобы залогировать некоторые операции можно просто передавать инстанс класса реализующего `WlgLog`:

```swift
class DownloadOperation {
    private let log: WldLog
    init(url: URL, log: WldLog) {
        ...
    }

    func start() {
        self.log.log(WlgLogEventDefault(.debug, "Starting download operation"))
    }  
}
```

Чтобы залогировать события можно создать класс `WlgOsLog`, который внутри себя будет использовать `os_log`, примерно вот так:

```swift
class WlgOsLog: WlgLog {
    private let subsystem: String
    private let category: String
    init(subsystem: String = "", category: String = "") {
        self.subsystem = subsystem
        self.category = category
    }

    func log<E: WlgLogEventI>(_ event: E) {
        if #available(iOS 10.0, macOS 10.12, tvOS 10.0, watchOS 3.0, *) {
            var osLogType = OSLogType.default
            var levelStr = ""
            switch event.level {
            case .debug:
                osLogType = .debug
                levelStr = "[DEBUG]"
            case .info:
                osLogType = .info
                levelStr = "[INFO]"
            case .error:
                osLogType = .error
                levelStr = "[ERROR]"
            default:
                osLogType = .default
            }
    
            let osLog = OSLog(subsystem: self.subsystem,
                              category: self.category)
            os_log("%{public}@ %{public}@",
                   log: osLog,
                   type: osLogType,
                   levelStr,
                   event.message)
        }
    }
}
```

В этом коде можно инстанцировать `WlgOsLog` с конкретной подсистемой и категорией. Тем самым упростив анализ логов в Console.app. При этом ничего не мешает использовать декорирование логгеров. И например, написать логгер отправляющий ошибки в Sentry/Firebase.

Чтобы не выводить ошибки в лог потребуется `WlgNullLog`, который не пишет никуда. Тогда в конструкторе объекта можно указывать дефолтную реализацию:

```swift
class DownloadOperation {
    private let log: WldLog
    init(url: URL, log: WldLog = WlgNullLog()) {
            ...
    }
}
```

Такой подход реализован в библиотеке [Wlog](https://github.com/chchrn/wlog)
