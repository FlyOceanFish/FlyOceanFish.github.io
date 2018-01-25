---
title: 带你初识Swift4.0,踏入Swift的大门 # 这是标题
date: 2018-01-25 10:27:00
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- iOS # 一级分类
- Swift
tags:   # 这里写的标签会自动汇集到 tags 页面上
- iOS
- Swift
---
# 背景
由于Swift4之前的版本也看过好几遍，不过好久没看有点忘记了，不过这次看也是非常得心应手。项目中也准备引入Swift，所以作者再次详细看了[The Swift Programming Language (Swift 4.0.3)](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language)英文官方文档一遍，并且详细列举了本人认为大家比较常用并且应该掌握的所有知识点。这里不做深入探究，如果哪里不懂的话，可以自行深入研究。
# 所有知识点
* 数组可以用“+”表示两个数组组合
* 字符串截取等操作很简洁
```
let greeting = "Hello, world!"
let index = greeting.index(of: ",") ?? greeting.endIndex
let beginning = greeting[..<index]
// beginning is "Hello"
```
* Dictionary可以遍历，并且支持移除操作
```Swift
airports["APL"] = nil
或
removeValue(forKey:)
```
遍历:
```Swift
for (airportCode, airportName) in airports {
    print("\(airportCode): \(airportName)")
}
```
* switch语法很强大，很灵活，支持任何类型，比如字符串、范围、管道等等。而且不用break语句
```
case "a", "A":
case 1..<5:
case (-2...2, -2...2)
case let (x, y) where x == y
```
* guard关键词使用使语句更简单明了
* #available判断api版本更简洁
```
if #available(iOS 10, macOS 10.12, *) {
    // Use iOS 10 APIs on iOS, and use macOS 10.12 APIs on macOS
} else {
    // Fall back to earlier iOS and macOS APIs
}
```
* 函数可以返回多个值
```
func minMax(array: [Int]) -> (min: Int, max: Int) {
    var currentMin = array[0]
    var currentMax = array[0]
    for value in array[1..<array.count] {
        if value < currentMin {
            currentMin = value
        } else if value > currentMax {
            currentMax = value
        }
    }
    return (currentMin, currentMax)
}
```
* 函数参数可以设置默认值
```
func someFunction(parameterWithoutDefault: Int, parameterWithDefault: Int = 12) {
    // If you omit the second argument when calling this function, then
    // the value of parameterWithDefault is 12 inside the function body.
}
someFunction(parameterWithoutDefault: 3, parameterWithDefault: 6) // parameterWithDefault is 6
someFunction(parameterWithoutDefault: 4) // parameterWithDefault is 12
```
* 可变参数`...`
```
func arithmeticMean(_ numbers: Double...) -> Double {
    var total: Double = 0
    for number in numbers {
        total += number
    }
    return total / Double(numbers.count)
}
arithmeticMean(1, 2, 3, 4, 5)
```
* `inout` 可以修改外部变量。与c语言指针有点类似
```
func swapTwoInts(_ a: inout Int, _ b: inout Int) {
    let temporaryA = a
    a = b
    b = temporaryA

```
* 函数可以与属性变量一样对待。可以传递、取值等
* 闭包(Closures)与Block类似

```
{ (parameters) -> return type in
    statements
}
```
*`@escaping`和`@autoclosure`修饰闭包
`@escaping`使闭包在函数执行结束之前不会被执行。
```
var completionHandlers: [() -> Void] = []
func someFunctionWithEscapingClosure(completionHandler: @escaping () -> Void) {
    completionHandlers.append(completionHandler)//闭包没有执行，而是加到数组中了
}
```
`@autoclosure`简化了闭包的代用，不过代码会变的难懂，苹果建议尽量不使用
```
// customersInLine is ["Alex", "Ewa", "Barry", "Daniella"]
func serve(customer customerProvider: () -> String) {
    print("Now serving \(customerProvider())!")
}
serve(customer: { customersInLine.remove(at: 0) } )
// Prints "Now serving Alex!"
```
使用之后：
```
// customersInLine is ["Ewa", "Barry", "Daniella"]
func serve(customer customerProvider: @autoclosure () -> String) {
    print("Now serving \(customerProvider())!")
}
serve(customer: customersInLine.remove(at: 0))//大括号没有啦
// Prints "Now serving Ewa!"
```
* 枚举功能很强大，很灵活，可以枚举的类型非常多，比如`string`、`character`、`integer`。
```
enum Barcode {
    case upc(Int, Int, Int, Int)
    case qrCode(String)
}

var productBarcode = Barcode.upc(8, 85909, 51226, 3)

switch productBarcode {
case .upc(let numberSystem, let manufacturer, let product, let check):
    print("UPC: \(numberSystem), \(manufacturer), \(product), \(check).")
case .qrCode(let productCode):
    print("QR code: \(productCode).")
}
```
```
enum Planet: Int{}
enum ASCIIControlCharacter: Character {}
enum CompassPoint: String {}
```
* indirect表示枚举中使用了自己
```
enum ArithmeticExpression {
    case number(Int)
    indirect case addition(ArithmeticExpression, ArithmeticExpression)
    indirect case multiplication(ArithmeticExpression, ArithmeticExpression)
}
```
或
```
indirect enum ArithmeticExpression {
    case number(Int)
    case addition(ArithmeticExpression, ArithmeticExpression)
    case multiplication(ArithmeticExpression, ArithmeticExpression)
}
```
* `struct`&&`class`
`struct`传递是值传递；`class`是引用传递
>In Swift, many basic data types such as String, Array, and Dictionary are implemented as structures. This means that data such as strings, arrays, and dictionaries are copied when they are assigned to a new constant or variable, or when they are passed to a function or method.

* ===和!==用来比较对象
* `lazy`懒加载，使用的时候再加载
* 仅读属性。将get、set移除，直接返回，可以用来实例单例
```
struct Cuboid {
    var width = 0.0, height = 0.0, depth = 0.0
    var volume: Double {
        return width * height * depth
    }
}
```
* mutating用来修改struct和enum。因为二者是不能不能被自己的实例对象修改属性变量
```
struct Point {
    var x = 0.0, y = 0.0
    mutating func moveBy(x deltaX: Double, y deltaY: Double) {
        x += deltaX
        y += deltaY
    }
}
var somePoint = Point(x: 1.0, y: 1.0)
somePoint.moveBy(x: 2.0, y: 3.0)
print("The point is now at (\(somePoint.x), \(somePoint.y))")
// Prints "The point is now at (3.0, 4.0)"

**注意let不能修改**
let fixedPoint = Point(x: 3.0, y: 3.0)
fixedPoint.moveBy(x: 2.0, y: 3.0)
// this will report an error

```

```
enum TriStateSwitch {
    case off, low, high
    mutating func next() {
        switch self {
        case .off:
            self = .low
        case .low:
            self = .high
        case .high:
            self = .off
        }
    }
}
var ovenLight = TriStateSwitch.low
ovenLight.next()
// ovenLight is now equal to .high
ovenLight.next()
// ovenLight is now equal to .off
```
* `class`和`static`都可修饰类方法,`class`修饰的可以被子类重写
* 脚标(Subscripts)。可以类、结构体、枚举定义脚标从而快速访问属性等。
```
subscript(index: Int) -> Int {
    get {
        // return an appropriate subscript value here
    }
    set(newValue) {
        // perform a suitable setting action here
    }
}
```
* `final`可以防止被子类继承。
* `deinit` 只有类有，当类销毁时会被回调。
* `?`是否为nil;`??`设置默认值
* 异常处理
```
do {
    try expression
    statements
} catch pattern 1 {
    statements
} catch pattern 2 where condition {
    statements
}
```
`try!`明确知道不会抛出异常。
* `defer`代码块执行后必须执行的代码
```
func processFile(filename: String) throws {
    if exists(filename) {
        let file = open(filename)
        defer {
            close(file)
        }
        while let line = try file.readline() {
            // Work with the file.
        }
        // close(file) is called here, at the end of the scope.
    }
}

```
* `is`判断是否某个类型;`as?` 或 `as!`转换至某个类型
* `Extensions`与oc的`category`类似
```
extension SomeType: SomeProtocol, AnotherProtocol {
    // implementation of protocol requirements goes here
}
```
* Protocols
```
protocol SomeProtocol {
    // protocol definition goes here
}

```
使用
```
struct SomeStructure: FirstProtocol, AnotherProtocol {
    // structure definition goes here
}
```
```
protocol SomeProtocol {
    var mustBeSettable: Int { get set }//有get和set方法
    var doesNotNeedToBeSettable: Int { get }//只有get方法，不能单独设置，只能在初始化的时候设置。
}
```
* 检查是否遵循了某个协议
    * `is`遵循了返回`true`，否则返回`false`
    * `as?`遵循了正常转换，否则为nil
    * `as!`遵循了正常转换，否则抛出错误

* 泛型(Generic)；通常使用`T`、`U`、`V`等大写字母表示。
```
func someFunction<T: SomeClass, U: SomeProtocol>(someT: T, someU: U) {
    // function body goes here
}
```
* `weak`或`unowned`(与oc的`unsafe_unretained`类似)解决循环应用的问题
```
weak var tenant: Person?
```
* 访问控制权限（open > public > interal > fileprivate > private）

     * private

        private 访问级别所修饰的属性或者方法只能在当前类里访问。
（注意：Swift4 中，extension 里也可以访问 private 的属性。）

  * fileprivate

      fileprivate 访问级别所修饰的属性或者方法在当前的 Swift）

  * internal（默认访问级别，internal修饰符可写可不写）

      internal 访问级别所修饰的属性或方法在源代码所在的整个模块都可以访问。
如果是框架或者库代码，则在整个框架内部都可以访问，框架由外部代码所引用时，则不可以访问。
如果是 App 代码，也是在整个 App 代码，也是在整个 App 内部可以访问。

  * public

      可以被任何人访问。但其他 module 中不可以被 override 和继承，而在 module 内可以被 override 和继承。

  * open

      可以被任何人使用，包括 override 和继承。
