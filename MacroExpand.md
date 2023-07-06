<a href="https://colab.research.google.com/github/t-fuchi/MacroExpand/blob/main/MacroExpand.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

# C++のテンプレートのインスタンス化をマクロを使って楽にする

本投稿ではC++でのジェネリックプログラミングにおいて、オブジェクトファイルで配布する場合に手間がかかるインスタンス化の問題を軽減する方法を提案します。

C++のテンプレートはとても便利です。例えば以下のようにテンプレートを使えば、それぞれの型ごとにfunc()を定義しないですみます。


```python
%%writefile macro_test.cpp

#include <iostream>
#include <typeinfo>

template <typename T>
T func(T v) {
    std::cout << v << ":" << typeid(v).name() << std::endl;
    return v;
}

int main(int argc, char **argv) {
    uint32_t u32 = 1;
    func(u32);
    int32_t s32 = 1;
    func(s32);
    int64_t s64 = 1;
    func(s64);
    return 0;
}
```

    Overwriting macro_test.cpp



```python
!g++ macro_test.cpp
!./a.out
```

    1:j
    1:i
    1:l


ここでなんらかの事情によりオブジェクトファイル（.oや.aや.so）でのみ配布したい場合を考えます。その場合、以下のようにテンプレートの定義のみを記述しても、個別の型に対してインスタンス化されません。


```python
%%writefile macro_test.cpp
#include <iostream>

template <typename T>
T func(T v) {
    std::cout << v << ":" << typeid(v).name() << std::endl;
    return v;
}

```

    Overwriting macro_test.cpp



```python
!g++ macro_test.cpp -c
!nm -g macro_test.o
```

                     U __cxa_atexit
                     U __dso_handle
                     U _GLOBAL_OFFSET_TABLE_
                     U _ZNSt8ios_base4InitC1Ev
                     U _ZNSt8ios_base4InitD1Ev


上記でfuncを含むシンボルが生成されていないことが分かります。このままmacro_test.oをリンクしてもfunc()は使用できません。

この問題は以下のように特殊化した関数を定義しておくことで解決できます。


```python
%%writefile macro_test.cpp
#include <iostream>

template <typename T>
T func(T v) {
    std::cout << v << ":" << typeid(v).name() << std::endl;
    return v;
}

template <>
int func(int v) {
    return func<int>(v);
}

```

    Overwriting macro_test.cpp



```python
!g++ macro_test.cpp -c
!nm -g macro_test.o
```

                     U __cxa_atexit
                     U __dso_handle
                     U _GLOBAL_OFFSET_TABLE_
    0000000000000000 T _Z4funcIiET_S0_
                     U _ZNSt8ios_base4InitC1Ev
                     U _ZNSt8ios_base4InitD1Ev


と書きましたが、コメントで明示的実体化を教えていただきました。明示的実体化はこうします。


```python
%%writefile macro_test.cpp
#include <iostream>

template <typename T>
T func(T v) {
    std::cout << v << ":" << typeid(v).name() << std::endl;
    return v;
}

template int func(int v);

```

    Overwriting macro_test.cpp



```python
!g++ macro_test.cpp -c
!nm -g macro_test.o
```

                     U __cxa_atexit
                     U __dso_handle
                     U _GLOBAL_OFFSET_TABLE_
    0000000000000000 W _Z4funcIiET_S0_
    0000000000000000 W _ZNKSt9type_info4nameEv
                     U _ZNSolsEi
                     U _ZNSolsEPFRSoS_E
                     U _ZNSt8ios_base4InitC1Ev
                     U _ZNSt8ios_base4InitD1Ev
                     U _ZSt4cout
                     U _ZSt4endlIcSt11char_traitsIcEERSt13basic_ostreamIT_T0_ES6_
                     U _ZStlsISt11char_traitsIcEERSt13basic_ostreamIcT_ES5_PKc
                     U _ZTIi


funcを含むシンボルが生成されているのが分かります。色々な型に対して準備するには次のように列挙することになります。


```python
%%writefile macro_test.cpp
#include <iostream>

template <typename T>
T func(T v) {
    std::cout << v << ":" << typeid(v).name() << std::endl;
    return v;
}

template int func(int v);
template long func(int v);
template long long func(int v);
template unsigned int func(int v);
template unsigned long func(int v);
template unsigned long long func(int v);
...
```

    Overwriting macro_test.cpp


これは地味につらい(かと思いましたが、明示的実体化ではそれほど辛くありませんね^^;;)。こんなテンプレートが５つも６つもあったら堪りません。解決するC++の機能を探したのですが、見当たりませんでした。というか探し方が分かりませんでした。

マクロで型のリストを与えて生成する方法を探しましたが、マクロにはfor文のような機能は備わっていないようです。一旦は諦めましたが、良い方法が見つかったので共有します。

マクロの中からマクロを呼ぶ方法です。まず次のようにそれぞれの型を引数にしてマクロを生成するマクロ(EXPAND_TYPE)を定義します。


```python
%%writefile macro_expand.h

#define EXPAND_TYPE(M) \
M(int); \
M(long); \
M(long long); \
M(unsigned int); \
M(unsigned long); \
M(unsigned long long); \
M(float); \
M(double);
```

    Overwriting macro_expand.h


これの動作を確認してみましょう。


```python
%%writefile macro_expand.cpp
#include "macro_expand.h"
EXPAND_TYPE(func)
```

    Overwriting macro_expand.cpp



```python
!g++ -E macro_expand.cpp | grep -v "#"
```

    func(int); func(long); func(long long); func(unsigned int); func(unsigned long); func(unsigned long long); func(float); func(double);


改行されないので見にくいですが、コンパイラさんにはちゃんと伝わりそうです。

実際の使い方は以下のようにテンプレートを実体化する関数に展開するマクロ(FUNC)を定義してEXPAND_TYPEに渡します。





```python
%%writefile macro_expand.cpp
#include "macro_expand.h"

template <typename T>
T func(T v) {
    return v;
}

#define FUNC(type) template type func(type v);
EXPAND_TYPE(FUNC);
```

    Overwriting macro_expand.cpp



```python
!g++ -E macro_expand.cpp | grep -v "#"
```

    
    template <typename T>
    T func(T v) {
        return v;
    }
    
    
    template int func(int v);; template long func(long v);; template long long func(long long v);; template unsigned int func(unsigned int v);; template unsigned long func(unsigned long v);; template unsigned long long func(unsigned long long v);; template float func(float v);; template double func(double v);;;


うまくいきました。シンボルを確認してみましょう。


```python
!g++ macro_expand.cpp -c
!nm -g macro_expand.o
```

    0000000000000000 W _Z4funcIdET_S0_
    0000000000000000 W _Z4funcIfET_S0_
    0000000000000000 W _Z4funcIiET_S0_
    0000000000000000 W _Z4funcIjET_S0_
    0000000000000000 W _Z4funcIlET_S0_
    0000000000000000 W _Z4funcImET_S0_
    0000000000000000 W _Z4funcIxET_S0_
    0000000000000000 W _Z4funcIyET_S0_


funcがたくさん定義されているのがわかります。テンプレートごとにFUNCを定義してEXPAND_TYPEを呼ぶだけなので、少しはマシになったのではないでしょうか。
