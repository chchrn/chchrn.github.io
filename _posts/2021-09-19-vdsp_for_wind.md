---
layout: post
title:  "DSP для отображения ветра"
date:   2021-09-19
tags: performance, vDSP, ios
---

Мы в приложении [Windy.app](https://apps.apple.com/us/app/windy-app-wind-weather/id997079492) показываем карту скорости ветра. На примере этой задачи рассмотрим как можно на iPhone делать быстрые операции на матрицах. 

Скорость ветра удобно представить в виде сетки, в узлах которой записаны значения.
Для сетки с шагом 0.25º данные в виде Float-ов займут `360 / 0.25 * 180 / 0. 25 * 4 = 4 Мб` . 
Если подумать то такая высокая точность для представления ветра нам не нужна. И должно было бы точности с размерностью UInt8, тогда общее кол-во данных можно было сократитить до 1 Мб. Для этого мы должны представить матрицу Float-ов в матрицу UInt8, для этого мы найдем min / max и просто равномерно переведем в UInt8. А на клиенте мы захотим перевести данные обратно, про это и будет статья. 

![](/assets/images/2021-09-19-vdsp_for_wind/wind_whb.jpeg){:width="100px"}

Не будем обсуждать как именно мы получили данные, главное что у на клиенте теперь есть массив UInt8 и `min` / `max`  оригинальных данных.
Чтобы на клиенте получить Float-ы из UInt8 нужно сделать следующие операции:
1. посчитать шаг округления `step = (max - min) / 256`
2. перевести UInt8 → Float
3. умножить значения на `step`
4. прибавить `min`

Для примера рассмотрим матрицу 1000 x 1000, это 1млн точек. 
Наивный алгоритм преобразования будет выглядеть следующим образом:
```swift
func floatArray(intValues: [UInt8],
                min: Float,
                max: Float) -> [Float] {
    let step = (max - min) / 256
    let result = intValues.map { (v: UInt8) -> Float in
        return Float(v) * step + min
    }
    return result
}
```
Время выполнения этого кода для матрицы 1000x1000 на iPhone 6s Plus - 160 мс. 

Казалось, что можно было сделать это быстрее. 
Apple предоставляет  несколько механизмов для работы с массивами / матрицами чисел. Один из них - [vDSP](https://developer.apple.com/documentation/accelerate/vdsp) во фреймворке Accelerate.  
Вот код для такого же преобразования, но c использованием библиотеки vDSP:
```swift
func floatArray(intValues: [UInt8],
                min: Float,
                max: Float) -> [Float] {
    var step = (max - min) / 256
    let count = intValues.count
    let stride = vDSP_Stride(1)
    let n = vDSP_Length(count)

    var fltArray = [Float](repeating: 0, count: count)
    vDSP_vfltu8(intValues, stride,
                &fltArray, stride,
                n)

    var mulArray = [Float](repeating: 0, count: count)
    vDSP_vsmul(fltArray, stride,
               &step,
               &mulArray, stride,
               n)

    var mMin = min
    var result = [Float](repeating: 0, count: count)
    vDSP_vsadd(mulArray, stride,
               &mMin,
               &result, stride,
               n)
    return result
}
```

Ниже таблица с разницей скорости работы методов

Девайс | Сетка | Array, sec | vDSP, sec | Прирост
:---: | :---: | :---: | :---: | :---:
iPhone 6 | 1000x1000 | 0.269 | 0.012 | x22.41
iPhone 6s plus | 1000x1000 | 0.18 | 0.0099 | x18.18
iPhone X | 1000x1000 | 0.202 | 0.0129 |  x15.65
 |  |  |  |  
iPhone 6 | 2000x2000 | 1.014 | 0.051 | x19.88 |
iPhone 6s plus | 2000x2000 | 0.705 | 0.0442 | x15.95
iPhone X | 2000x2000 | 0.789 | 0.0473 | x16.68  

[Исходники теста](https://gist.github.com/chchrn/da4e69965f3c667b15c1a2eb7546400d)


### Refs
[Документция Apple по vDSP](https://developer.apple.com/documentation/accelerate/vdsp)   
[Кейс для обработки сигналов для BLE](https://devzone.nordicsemi.com/nordic/nordic-blog/b/blog/posts/nrf_2d00_connect_2d00_simd_2d00_optimizations_2d00_in_2d00_swift)