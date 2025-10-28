# Class templates

Unlike non template classes, we cant define class templates inside functions / block scope. In general, templates can only be defined in global/namespace scope or inside class declarations.

```cpp title="stack1.hpp"
#include <vector>
#include <cassert>

// declaring class template
// can use keyword class instead of typename
// T is just a type
template <typename T>
class Stack {
private:
    // uses vector so no need to manage memory
    // or implement copy and assignment funcs
    std::vector<T> elems;
public:
    void push(T const& elem);
    void pop();
    T const& top() const;
    bool empty() const {
        return elems.empty();
    }
};

template <typename T>
void Stack<T>::push(T const& elem){
    elems.push_back(elem);
}

template <typename T>
T Stack<T>::pop(){
    assert(!elems.empty());
    T elem = elems.back();
    elems.pop_back();
    return elem;
}

template <typename T>
T const& Stack<T>::top() const {
    assert(!elems.empty());
    return elems.back();
}
```

Code is instantiated only for template functions that are called, same for class templates, only the used member functions are instantiated. This is to save time and space and allow use of partial class templates. If a class template has static members, it will also be instantiated once for every type, where the class template is used.

```cpp title="stack1test.cpp"
#include "stack1.hpp"
#include <iostream>
#include <string>

int main(){
    Stack<int> intStack;
    Stack<std::string> stringStack;

    intStack.push(7);
    stringStack.push("hello");
    stringStack.pop();
}
```
intStack will use int as type T inside the class template, and string as T for stringStack. Above will instantiate push() and pop() for stringStack, and only push() for intStack.

## Concepts
Concept is used to express which operations are required for a template to be able to get instantiated (e.g has random access iterator, default constructible).

Without C++20 Concepts :
It will also not compile without this, but this will give better error messages.
```cpp
template <typename T>
class C {
    static_assert(std::is_default_constructible<T>::value, "Class C requires default constructible elements");
}
```

With C++20 Concepts :
```cpp
#include <concepts>
template <std::default_initializable T>
class C {
    T value;
};
```


## Friends
operator<< for class Stack<> is not a function template, but ordinary function that is instantiated with the class template, if needed. 
```cpp
template <typename T>
class Stack {
    ...
    friend std::ostream& operator<< (std::ostream& os, Stack<T> const& s){
        for(T const& elem : elems){
            os << elem << " ";
        }
        os << "\n";
        return os;
    }
};
```

## Class Template Specialization
Specializing class templates allow to optimize or fix implementations for certain types, similar to overloading function templates (e.g vector<bool> is specialized vector template). However we need to also specialize all member functions, and declare with leading template<>.

```cpp
template<>
class Stack<std::string> {
private:
    std::deque<std::string> elems;
public:
    void push(std::string const&);
    void pop();
    std::string const& top() const;
    bool empty() const {
        return elems.empty();
    }
};

void Stack<std::string>::push(std::string const& elem){
    elems.push_back(elem);
}

void Stack<std::string>::pop(){
    assert(!elems.empty());
    elems.pop_back();
}

std::string const& Stack<std::string>::top() const {
    assert(!elems.empty());
    return elems.back();
}
```

## Partial Specialization
We can provide special implementations, for example Stack<> for pointers.
```cpp
#include "stack1.hpp"

template<typename T>
// define class template, specialized for pointer Stack<T*>
class Stack<T*> { 
private:
    std::vector<T*> elems;
public:
    void push(T*);
    T* pop;
    T* top() const;
    bool empty() const {
        return elems.empty();
    }
}

template<typename T>
void Stack<T*>::push(T* elem){
    elems.push_back(elem);
}

template<typename T>
T* Stack<T*>::pop(){
    ...
}
template<typename T>
T* Stack<T*>::top() const {
    ...
}
```

## Multiple Parameter Specialization
```cpp
template<typename T1, typename T2>
class MyClass {

};

template<typename T>
class MyClass<T, T> {

};

template<typename T>
class MyClass<T, int> {

};

template<typename T1, typename T2>
class MyClass<T1*, T2*> {

};

MyClass<int, float> mif; // MyClass<T1, T2>
MyClass<float, float> mff; // MyClass<T, T>
MyClass<float, int> mfi; // MyClass<T, int>
MyClass<int*, float*> mp; // MyClass<T1*, T2*>
MyClass<int, int> mii; // Error, matches MyClass<T, T> & MyClass<T, int>
MyClass<int*, int*> mpp; // Error, matches MyClass<T, T> & MyClass<T1*, T2*>;
```

To fix the ambiguity, we can provide partial specialization for same type pointers : 
```cpp
template<typename T>
class MyClass<T*, T*> {
    ...
};
```

## Default Class Template Argument
We can define default values for the template arguments.
Here the container type is vector by default, if only 1 template argument is provided.

```cpp title="stack3.hpp"
template<typename T, typename Cont = std::vector<T>>
class Stack {
private:
    Cont elems;
public:
    void push(T const& elem);
    ...
}

template<typename T, typename Cont>
void Stack<T, Cont>::push (T const& elem) {
    elems.push_back(elem);
}
```
```cpp
// T int, Cont vector<int>
Stack<int> intStack;
// T double, Cont deque<double>
Stack<double, std::deque<double>> doubleStack;
```


## Type aliases
We can use alias for class template to define a new name. This define a new name for existing type, not a new type, and can be used interchangeably for Stack<int>.
Using `typedef` :
```cpp
typedef Stack<int> IntStack;
void foo(IntStack const& s);
IntStack istack[10];
```
Using `using` :
```cpp
using IntStack = Stack<int>;
void foo(IntStack const& s);
IntStack istack[10];
```
Alias declaration can also be templated for a family of types since C++11, called alias template.
```cpp
template<typename T>
using DequeStack = Stack<T, std::deque<T>>;
// DequeStack<int> same as Stack<int, std::deque<int>>
```

## Type traits suffix _t
Since C++14, shortcuts for all type traits _t to get the type. 
```cpp
std::add_const_t<T>
// instead of
typename std::add_const<T>::type
// by standard library defining : 
namespace std {
    template<typename T>
    using add_const_t = typename add_const<T>::type;
}
```

## Class Template Argument Deduction ([CTAD](https://en.cppreference.com/w/cpp/language/class_template_argument_deduction.html), C++17)
Since C++17, we can skip defining template arguments explicitly if the constructor is able to deduce all template parameters that dont have default value. Unlike function templates, class template arguments cant be deduced partially. If template argument list is specified, deduction does not take place.
```cpp

std::tuple t1(1, 2, 3);                // OK: deduction
std::tuple<int, int, int> t2(1, 2, 3); // OK: all arguments are provided
 
std::tuple<> t3(1, 2, 3);    // Error: no matching constructor in tuple<>.
                             //        No deduction performed.
std::tuple<int> t4(1, 2, 3); // Error
```
```cpp
Stack<int> intStack1;
Stack<int> intStack2 = intStack1; // explicit definition is ok
Stack intStack3 = intStack1; // since C++17
```
This is because we have constructor that accept initial arguments, hence it can deduce the element type. Example constructor accepting one element T : 
```cpp
// default constructor only available if no other constructor defined
Stack() = default; 
// Stack constructor creating vector<T> of 1 element, from element T elem
Stack(T const& elem) : elems({elem}) {} 
...
Stack intStack = 0; // Stack<int> deduced, constructor accepting 1 element T = int
Stack stringStack = "bottom"; // Stack<char const[7]> deduced
// passing by reference T const& elem, char const[7] doesnt decay to a raw pointer.
```
Passing the arguments T by value will decay the parameter to char const* instead
```cpp
Stack(T elem) : elems({std::move(elem)}) {}
Stack stringStack = "bottom"; // deduce to Stack<char const*>
```

## Deduction Guide
Handling raw character pointers can be disabled by defining deduction guides. For example, define that whenever string literal or C string is passed its instantiated with std::string. There are implicitly generated deduction guide, and user-defined.
```cpp
// templated guiding
template <typename T>
Wrapper(T*) -> Wrapper<T>;          // if pointer, deduce T
template <typename T>
Wrapper(std::vector<T>) -> Wrapper<T>;  // for container types

```
```cpp
// guiding type 
Stack(const char*) -> Stack<std::string>;

template <typename T>
class Stack {
private:
    std::vector<T> elems;
public:
    Stack(T const& elem) : elems({elem}) {}
};

// now :
Stack stringStack{"bottom"}; // Stack<std::string> deduced & valid
Stack stack2{stringStack}; // Stack<std::string>
Stack stack3(stringStack); // Stack<std::string>
Stack stack4 = {stringStack}; // Stack<std::string>
Stack stringStackInvalid = "bottom"; // invalid!! copy initialization, not direct initialization
// no implicit conversion from const char* to Stack<std::string>
```
By language rules, we can't copy initialize (using =) an object by passing a string literal to a constructor expecting a std::string. 

## Templatized Aggregates
[Aggregate classes](https://en.cppreference.com/w/cpp/language/aggregate_initialization.html) are class/struct that has : <br>
- No user provided/explicit/inherited constructor (copy, move, and default constructor) <br>
- No private / protected nonstatic data members (all visible nonstatic) <br>
- No virtual functions <br>
- No virtual base class <br>

For example : 
```cpp
template <typename T>
struct ValueWithComment {
    T value;
    std::string comment;
};

ValueWithComment<int> vc;
vc.value = 42;
vc.comment = "initial value";

// Since C++17, can define deduction guide for aggregate class templates :
ValueWithComment(char const*, char const*) 
    -> ValueWithComment<std::string>;
ValueWithComment vc2 = {"hello", "initial value"};
```
The struct defines aggregated parameterized for the type that value holds. Without the deduction guide, the initialization isnt possible, since it does not have constructor to perform the deduction against.

More from [cppreference](https://en.cppreference.com/w/cpp/language/class_template_argument_deduction.html) : 
```cpp
template<class A, class B>
struct Agg
{
    A a;
    B b;
};
// implicitly-generated guides are formed from default, copy, and move constructors
 
template<class A, class B>
Agg(A a, B b) -> Agg<A, B>;
// ^ This deduction guide can be implicitly generated in C++20
 
Agg agg{1, 2.0}; // deduced to Agg<int, double> from the user-defined guide 
 
template<class... T>
array(T&&... t) -> array<std::common_type_t<T...>, sizeof...(T)>;
auto a = array{1, 2, 5u}; // deduced to array<unsigned, 3> from the user-defined guide
```

## Nontype Class Template
Template parameters don't have to be types, can also be ordinary values. Like type parameter, we define code that a certain detail is still open until the code is used, but the detail is value instead of type.

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


