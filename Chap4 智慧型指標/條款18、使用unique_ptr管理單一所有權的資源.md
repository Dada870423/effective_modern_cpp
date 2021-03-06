std::unique_ptr是最基本的智慧型指標，通常其與原始指標有著相同的大小，而大多數的操作也都與原始指標使用相同的指令集，所以如果在所有原始指標符合需求的情況下，使用智慧型指標應該也都能符合需求。

1. 擁有權
而std::unique_ptr擁有指向物件的單一所有權，也就是只有一個std::unique_ptr指向該物件，除了nullptr外，所有的std::unique_ptr都只有指向一個物件。
2. 搬移
當搬移一個std::unique_ptr時，會轉移所有權。而由於一個物件只會有一個std::unique_ptr指向他，故不支援複製語意。
3. 清除
清除時非nullptr的std::unique_ptr會清除所擁有的資源，預設行為是delete

適用情況：
std::unique_ptr其中一個常用的應用是設計模式中的factory方法的傳回型別。

假設有許多的投資方法，基底為investment
```cpp
class Investment
{
    ...
};
```

而三種不同的投資方法
```cpp
class Stock : public Investment
{
    ...
};

class Bond : public Investment
{
    ...
};

class ReadEstate : public Investment
{
    ...
};
```

而就像建立其他的智慧型指標一樣，我們也做一個創建investment的函式makeInvestment

```cpp
template <typename... Ts>
std::unique_ptr<Investment> makeInvestment(Ts&&... args);
```

使用者就可以在宣告std::unique_ptr的範圍中使用std::unique_ptr

```cpp
{
    ...
    auto pInvestment = makeInvestment(arguments...);
    ...
}   // 清除pInvestment
```

而無論程式是會安全的離開scope或是提前發生例外或break，std::unique_ptr在離開時都一定會呼叫destructor。

而std::unique_ptr中預設的destructor行為是使用delete，但也可以自行設定destructor的行為，若需要設定，必須在建立std::unique_ptr時就使用自行定義的刪除子(custom deleter)，刪除子可以是任何函式(e.g.lambda expression)。

以下就實作擁有可以輸出destructor log訊息的Investment物件

```cpp
auto delInvestment = [](Investment& pInvestment)
    {
        makelog(pInvestment);
        delete pInvestment;
    }

template <typename... Ts>
std::unique_ptr<Investment, decltype(delInvestment)> makeInvestment(Ts... args)
{
    std::unique_ptr<Investment, decltype(delInvestment)> pInv(nullptr, delInvestment);

    if (/* 需要產生Stock的條件 */)
    {
        pInv.reset(new Stock(std::forward<Ts>(args)...))
    }

    else if (/* 需要產生Bond的條件 */)
    {
        pInv.reset(new Bond(std::forward<Ts>(args)...))
    }

    else if (/* 需要產生RealEstate的條件 */)
    {
        pInv.reset(new RealEstate(std::forward<Ts>(args)...))
    }

    return pInv;
}
```

以使用者的角度來說，在執行makeInvestment後，智慧型指標便會負責資源的使用，不必擔心清除是否有重複清除或是沒有清除的情況。

以上實作包含以下幾點
- 自訂刪除子，刪除子的引數並需接受原始指標才能被std::unique_ptr所接受，使用lambda expression除了表示很方便外，也比傳統函式還要來的有效率，之後會對lambda expression和函式做比較，隨後在建立std::unique_ptr時使用建立刪除子。
- 自訂刪除子時，刪除子必須是std::unique_ptr的第二個型別參數，也就是上例中delInvestment的型別。
- 建立Investment的std::unique_ptr的流程是先創建萬用的nullptr，再使用std::unique_ptr::reset重建Investment指標，此處不能直接使用new Stock(...)來為pInv賦值，如果這樣做會產生compiler error。不過，如果原始指標是作為constructor參數，是可以直接被初始化的。

e.g.
```cpp
int* p = new int(1);
std::unique_ptr<int> ui(nullptr);
//ui = p;   // 產生compiler error

std::unique_ptr<int> ui1(new int(1));   // ok
std::unique_ptr<int> ui2(p);
```

使用C++14以上，更可以直接將回傳型別設為auto

```cpp
template <typename... Ts>
auto makeInvestment(Ts... args)
{
    std::unique_ptr<Investment, decltype(delInvestment)> pInv(nullptr, delInvestment);

    if (/* 需要產生Stock的條件 */)
    {
        pInv.reset(new Stock(std::forward<Ts>(args)...))
    }

    else if (/* 需要產生Bond的條件 */)
    {
        pInv.reset(new Bond(std::forward<Ts>(args)...))
    }

    else if (/* 需要產生RealEstate的條件 */)
    {
        pInv.reset(new RealEstate(std::forward<Ts>(args)...))
    }

    return pInv;
}
```

而一旦使用自訂的destructor後，不同型態的destructor會有不太一樣的硬體使用資源，若是使用函式指標作為destructor，會使得std::unique_ptr大小增加1~2個word，但若是使用lambda表達式且沒有擷取任何資訊，則不會使用額外的空間。所以如果可以，使用lambda表達式會是比較好的選擇。

```cpp
auto delInvestment1 = [](Investment& pInvestment)
    {
        makelog(pInvestment);
        delete pInvestment;
    }

template <typename... Ts>
std::unique_ptr<Investment, decltype(delInvestment1)> makeInvestment(Ts... args);    型別大小與Investment*相同

void delInvestment1(Investment& pInvestment)
{
    makelog(pInvestment);
    delete pInvestment;
}

template <typename... Ts>
std::unique_ptr<Investment, decltype(delInvestment2)> makeInvestment(Ts... args);    型別大小=Investment*大小+函式指標...
```

後面還有提到array和原始指標的用法，基本上std::unique_ptr<T[]>只能使用index參閱，而std::unique_ptr<T*>只能使用解參考，不過現在已經幾乎不使用array了，可以不用太在意。

而std::unique_ptr還有一個最重要的應用，就是可以直接轉換成std::shared_ptr。

```cpp
std::shared_ptr<Investment> sp = makeInvestment(argument...);
```

這也是為何std::unique_ptr適用於factory設計模式，因為開發者無法得知使用者希望factory函式能夠回傳單一所有權或是共享所有權的指標，所以先提供最萬用的std::unique_ptr，給予適當的彈性，若需要時在可以轉換為std::shared_ptr。