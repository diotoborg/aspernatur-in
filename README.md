# JSOBin
JSOBin是一个用于序列化javascript对象的规范和实现   

+ 优点 对比JSON
    + 更小
        + 使用二进制格式
        + 对整数进行变长压缩
    + 更好的支持
        + 支持 循环引用
        + 支持 有向无环式引用
        + 支持 Bigint
        + 支持 部分javascript内建类
            + 支持 Map类
            + 支持 Set类
            + 支持 ArrayBuffer类
            + 支持 TypedArrey类
        + 支持 自定义类
            + 支持 自定义类的序列化和反序列化函数
        + 支持 symbol (有限制)
        + 支持 undefined
        + 无损的支持 64位浮点数 (没有转换成十进制带来的偏差)
            + 支持 负 0
            + 支持 正/负 无限
            + 支持 NaN
+ 缺点 对比JSON
    + 人类不可读 几乎抛弃了所有可读性

关于序列化后格式的规范 见此仓库中的 [standard.md](./standard.md)   


## 使用 JSOBin

1 导入JSOBin   

```javascript
import { JSOBin } from "@diotoborg/aspernatur-in";
```


2 创建用于操作的上下文   

JSOBin的编解码依赖一个上下文来操作   
因为我们需要在其中注册需要序列化的类或者其他的东西   

```javascript
let jsob = new JSOBin();
```


3 序列化 (编码)   

这样就编码完成了!

```javascript
let serializedBinaryFormat = jsob.encode({
    testObjectKey: "test object value"
});
```


4 反序列化 (解码)   

这样就解码完成了!

```javascript
let deserializedObject = jsob.decode(serializedBinaryFormat);
```


5 实现自定义类的编解码

需要实现自定义类的编解码需要将类在上下文中注册   
否则将会像普通js对象一样处理   
用于编解码的上下文都需要注册相同的类才能使得此类支持编解码   
且这些上下文中对同一类登记的标识符必须相同   
不同的类必须使用不同的标识符   

现在jsob这个上下文支持MyClass的编解码了!

```javascript
jsob.addClass("myClassName", MyClass);
```

6 自定义类序列化

也许你不喜欢类的默认序列化和反序列化方案   
或者他们可能不会正常工作?   
这是大概正常的 因为默认的反序列化方案不会调用构造函数等等原因   
很多情况下我们需要自定义的类序列化函数   

```javascript
import { serializationFunctionSymbol, deserializationFunctionSymbol } from "@diotoborg/aspernatur-in"; // 需要额外引用这两个符号

class MyClass
{
    /*
        这个函数在此类序列化时被调用
        他的this指向正在被序列化的对象(类的实例)
        然后他需要返回一个对象
        返回对象的键必须是字符串
        返回对象的值可以是任何其他的@diotoborg/aspernatur-in支持的类型
        当然 如果返回对象的值中存在此类 或者其他的循环 可能导致此函数被递归调用最后永远循环下去!
        所以小心使用 总之尽量避免返回对象的值中有此类本身或者其他在序列化过程可能造成循环的一切
    */
    [serializationFunctionSymbol]()
    {
        return { // 在这里导出足够多的数据以便在反序列化过程中还原此类实例
            someValue: "",
            someObject: {},
            somethingElse: [
                ""
            ]
        };
    }

    /*
        这个函数是静态的
        当需要反序列化此类时将调用此函数
        会将 数据对象 作为其参数传入
        这里的数据对象就是你在上面的函数中返回的对象!
        然后 你可以返回创建的类的实例 他会被放在本来的位置上
        或者 你也可以返回和此类无关的其他类实例或对象
        总之 返回的值将被放在这个类实例原本在@diotoborg/aspernatur-in中的位置上
    */
    static [deserializationFunctionSymbol](dataObj)
    {
        let ret = new MyClass();

        // do something here

        return ret;
    }
}
```

7 "序列化"安全函数

虽然我不确定这么做的具体用途   
但如果你愿意你可以将函数绑定并进行编解码   
注意: 这并不会导出函数的代码等 仅仅是将函数与对应标识符绑定   
当你确定没有安全问题后 可以这样将函数绑定到上下文   
与类的编解码相同 所有相关的上下文都需要绑定对应的函数才能完成函数的编解码   


```javascript
function testFunction()
{
    console.log("test");
}

jsob.addSafetyFunction("testFunction", testFunction); // 绑定函数

let functionBindTest = jsob.encode({ // 尝试序列化
    test: testFunction
});

let testObj = jsob.decode(functionBindTest); // 尝试反序列化

testObj.test(); // 他工作了!
```