# Constant/Nontype Template Parameter
In C++, nontype/constant template parameter is a value, rather than a type as argument for the template during instantiation. The parameters are compile time constants, hence values are known at the time code is compiled. Nontype template parameter can be used for both function and class templates. <br><br>
Quoting [cppreference](http://en.cppreference.com/w/cpp/language/template_parameters.html), 
"Before C++26, constant template parameter were called non-type template parameter in the standard wording. The terminology was changed by [P2841R6](https://wg21.link/P2841R6) / [PR#7587](https://wg21.link/EDIT7587)".

## Nontype Class Template
Stack with fixed size array for the elements, where we can define the size as a template parameter :
```cpp
#include <array>
#include <cassert>

template <typename T, std::size_t Maxsize>
class Stack {
private:
    std::array<T, Maxsize> elems;
    std::size_t numElems;

public:
    Stack();
    void push(T const& elem);
    void pop();
    T const& top() const;
    bool empty() const { return numElems == 0; }
    std::size_t size() const { return numElems; }
};

template <typename T, std::size Maxsize>
Stack<T, Maxsize>::Stack(){
    numElems = 0;
}

template <typename T, std::size Maxsize>
void Stack<T, Maxsize>::push(T const& elem){
    assert(numElems < Maxsize);
    elems[numElems] = elem;
    numElems++;
}

template <typename T, std::size Maxsize>
void Stack<T, Maxsize>::pop(){
    assert(!elems.empty());
    numElems--;
}

template <typename T, std::size_t Maxsize>
T const& Stack<T, Maxsize>::top() const {
    assert(!elems.empty());
    return elems[numElems - 1];
}
```
We need to specify element and max size in order to use it.
```cpp
Stack<int, 20> int20Stack;
Stack<int, 40> int40Stack;
Stack<std::string, 40> string40Stack;
```
Each template instantiation above is its own type, so int20Stack and int40Stack are different type, and no implicit/explicit type conversion between them are defined. Hence we cannot use them instead of the other, and can't assign to one other. We can still define default arguments as such :

```cpp
template<typename T = int, std::size_t Maxsize = 100>
class Stack {
    ...
};
```

## Nontype Function Template
Same as class template, we can also define nontype parameters for function templates. 
```cpp
template<int Val, typename T>
T addValue(T x){
    return x + Val;
}
```
This can be useful if functions or operations used as parameters, example :
```cpp
std::transform(source.begin(), source.end(), dest.begin(), addValue<5, int>);
```
The `int` type cannot be deduced, and we need to specify the argument `int` as `T`. Deduction only works for immediate calls and `std::transform()` need a complete type to deduce type of fourth parameter. No support for partial deduce/substitute. 
We can also specify parameter to deduce from previous parameter, or to ensure nontype value has same type as passed type, example:
```cpp
template<auto Val, typename T = decltype(Val)>
T foo();

template<typename T, T Val = T{}>
T bar()
```

## Restrictions of Nontype Template Parameters
In general, they can only be constant integral values (including enumerations), pointers to objects/functions/members, lvalue references to objects/functions, or `std::nullptr_t` (type of `nullptr`).
Floating point and class type objects are not allowed (before C++20, noted below).
```cpp
template<double VAT> // Floating point values NOT allowed as template parameter (before C++20)
double process (double v) {
    return v * VAT;
}

template<std::string name> // Class type objects NOT allowed
class MyClass {
    ...
};
```

Also, objects must not be string literals, temporaries, or data members and other subobjects. <br>
In C++11, objects had to have external linkage <br>
In C++14, objects had to have internal/external linkage <br>

```cpp
// NOT ALLOWED
template<char const* name> // String literal not allowed
class MyClass {
    ...
};
MyClass<"hello"> x;

// Workarounds : 
template<auto T>
class Message {
public:
    void print() {
        std::cout << T << '\n';
    }
};

extern char const s03[] = "hi"; // external linkage
char const s11[] = "hi"; // internal linkage

int main(){
    Message<s03> m03; // OK for all versions
    Message<s11> m11; // OK since C++11
    static char const s17[] = "hi"; // no linkage
    Message<s17> m17; // OK since C++17
}
```

### Compile Time Expressions
Arguments for nontype template parameters can also be any compile time expressions :
```cpp
template<int I, bool B>
class C;
...
C<sizeof(int) + 4, sizeof(int) == 4> c; // ok
C<42, sizeof(int) > 4> c; // ERROR, the > ends argument list
C<42, (sizeof(int) > 4)> c; // ok
```

## Template Parameter Type `auto`
Since C++17, auto can be used to define any type that is allowed as nontype parameter. This allows a more generic templates.
```cpp
#include <array>
#include <cassert>
// Maxsize is a value, where type isnt defined yet
template <typename T, auto Maxsize>
class Stack {
public:
    using size_type = decltype(Maxsize);
private:
    std::array<T, Maxsize> elems;
    size_type numElems;
public:
    Stack();
    void push(T const& elem);
    void pop();
    T const& top() const;
    bool empty() const { return numElems == 0; }
    size_type size() const { return numElems; }
};

template<typename T, auto Maxsize>
Stack<T, Maxsize>::Stack() : numElems(0) {}
// ...
// other member functions

int main() {
    Stack<int, 20u> int20Stack; // size_type unsigned int
    Stack<std::string, 40> stringStack; // size_type int
    Stack<int, 3.14> sd; // ERROR, floating point nontype argument not allowed
    auto size1 = int20Stack.size();
    auto size2 = stringStack.size();

    // _v shortcut for std::is_same<>::value
    if (!std::is_same_v<decltype(size1), decltype(size2)>) {
        // this will be printed
        std::cout << "size type differ" << '\n';
    }
}
```

We can also use `decltype(auto)` as type, that allows instantiation as a reference type.
```cpp
template<decltype(auto) N>
class C {
    ...
};
int i;
C<(i)> x; // N is int&
```

### C++20 Changes :
C++20 allows nontype template parameter to be :<br>
- A floating point type<br>
- A lambda closure type that has no capture<br>
- A nonclosure literal class type where all base classes and non-static data members are public and non-mutable and the types of all base classes and non-static data members are structural types or (possibly multi-dimensional) array thereof.