---
layout: post
title: Defensive Copy on Struct
---

大多數人一定都知道.NET的型別系統有兩大分支：一個是參考型別(reference types)，另一個則是實值型別(value types)。

C# 從 7.0 之後引入了幾個跟實值型別效能相關的特性: 
1. [ref 區域變數和傳回](https://docs.microsoft.com/zh-tw/dotnet/csharp/whats-new/csharp-7#ref-locals-and-returns)
2. [以唯讀參考傳遞引數](https://docs.microsoft.com/zh-tw/dotnet/csharp/reference-semantics-with-value-types#passing-arguments-by-readonly-reference)
3. [ref readonly 傳回](https://docs.microsoft.com/zh-tw/dotnet/csharp/reference-semantics-with-value-types#ref-readonly-returns)
4. [readonly struct 類型](https://docs.microsoft.com/zh-tw/dotnet/csharp/reference-semantics-with-value-types#readonly-struct-type)
5. [ref struct 類型](https://docs.microsoft.com/zh-tw/dotnet/csharp/reference-semantics-with-value-types#ref-struct-type)
6. [readonly ref struct 類型](https://docs.microsoft.com/zh-tw/dotnet/csharp/reference-semantics-with-value-types#readonly-ref-struct-type)

所做的一切都是為了降低GC回收成本以提升效能。`ref struct`(Span<T>)實際上要等到 *dotnetcore 2.1* 才能看到成效，在此就先不討論了。

----
我們都知道以實值型別作為參數或是回結果型別的時候，預設都是使用副本傳遞(copy/return by value)。雖然在呼叫堆疊(call stack)上宣告的實值型別變數不會使用堆積(Heap)配置(不需要垃圾回收)，但如果實值型別體積過大，記憶體複製所帶來的效能影響還是很可觀，因此我們*不能一廂情願地為了效能而使用實值型別*。`ref`關鍵字就是為此而生的，可是 `ref`只能用在參數傳遞，回傳結果則要使用`out`參數。

```csharp
void Foo(ref int x, out int y)

int a = 0, y;
Foo(ref a,out y);
```

這種可怕的呼叫形式相信很多人都很反感。

所以在 C# 7.0 就新增了 `ref return` 的機制(有條件限制)

進入正題
=====
`readonly` 是 C# 一開始就有的關鍵字，但很多人可能不知道的是`readonly`套用在實值型別變數上會產生***Defensive Copy***的副作用。

```csharp
struct Value {
    int _x;
    public int X => _x;

    public Value(int x) => _x = x;

    public int Increment() => ++_x;
}

class Program {
    static readonly Value value = new Value(1);

    void Main(){
        Console.WriteLine($"Increment result: {value.Increment()}");
        Console.WriteLine($"X = {value.X}");
    }
}
```
希望你沒有被輸出結果嚇到
```
Increment result: 2
X = 1
```
`readonly`關鍵字其實是語意修飾詞而不是CLR所支援的特性。你可以這樣想：把變數宣告成***唯讀***的目的不就是為了***讓變數的內容維持不變***嗎? 如果套用的變數是參考型別變數，一旦被初始化過就無法重新指派。而因為實質型別是以整個物件內所有的內容值來判定內容是否一致，因此所謂的讓實質型別變數的內容維持不變就等於必須讓實質型別完全相等。所以編譯器為了避免任何有修改內容的行為而違反`readonly`語意，勢必要產生一個副本讓可能有副作有的程式碼使用，這個行為就叫做***Defensive Copy***。以上面的例子來說，不論有沒有呼叫*Increment*方法 *Defensive Copy* 一定會發生，因為編譯器無從得知開發人員是否會寫出修狀態的程式碼。

----
前面提到產生副本對效能是有傷害的。要具有唯讀語意又能避免 *Defensive Copy* 的答案自然是需要由程式員告訴編譯器：這個實質型別的內容完全無法修改。
你可能會想：把所有的成員變數宣告成 readonly，然後移除所有會修改狀態的邏輯總行了吧?
答案是：編譯器也許可以這樣設計，但比較沒有效率。

所以 C# 7.2 開始 readonly 可以當作宣告 struct 的修飾詞，這樣一來編譯器就可以明確的知道 struct 是否不可變，而且一旦宣告為 readonly struct，所有的成員變數也必須宣告為 readonly。

----
而同樣是 7.2 加入的`in 參數`又是做什麼用的呢?前面有提到為了避免大型實質物件參數傳遞的開銷，我們可以將參數宣告成`ref`(copy by reference)；可是 ref 沒有唯讀的語意。編譯器團隊之所以不用`readonly ref`的原因，其實是為了讓既有的程式碼可以更容易移植：`ref`、`out`參數在呼叫端也必須使用對應的關鍵字，而`in`參數呼叫形式與一般參數無異

```csharp
void Foo(in int a);

Foo(10); // it's ok without 'in'
```

最後以此類推 `readonly ref return` 就是具有唯讀語意的 `ref return`

>
實質型別的設計原則
* Mutable value types are evil
* Use readonly modifier on structs whenever possible

補充說明
====
Defensive Copy 的行為必須透過 IL 代碼才能觀察出來，以上面的例子來看

Defensive Copy
```
IL_0000:  ldstr       "X = {0}"
IL_0005:  ldsfld      UserQuery.value  /// 
IL_000A:  stloc.0                      /// 
IL_000B:  ldloca.s    00               ///   
IL_000D:  call        UserQuery+Value.get_X
IL_0012:  box         System.Int32
IL_0017:  call        System.String.Format
IL_001C:  call        System.Console.WriteLine
IL_0021:  ret         
```

消除後
```
IL_0000:  ldstr       "X = {0}"
IL_0005:  ldsflda     UserQuery.value  /// 
IL_000A:  call        UserQuery+Value.get_X
IL_000F:  box         System.Int32
IL_0014:  call        System.String.Format
IL_0019:  call        System.Console.WriteLine
IL_001E:  ret     
```

強烈建議閱讀相關文章：
>1. [The ‘in’-modifier and the readonly structs in C#](https://blogs.msdn.microsoft.com/seteplia/2018/03/07/the-in-modifier-and-the-readonly-structs-in-c/)
2. [Performance traps of ref locals and ref returns in C#](https://blogs.msdn.microsoft.com/seteplia/2018/05/03/avoiding-struct-and-readonly-reference-performance-pitfalls-with-errorprone-net/)
3. [Avoiding struct and readonly reference performance pitfalls with ErrorProne.NET](https://blogs.msdn.microsoft.com/seteplia/2018/05/03/avoiding-struct-and-readonly-reference-performance-pitfalls-with-errorprone-net/)
