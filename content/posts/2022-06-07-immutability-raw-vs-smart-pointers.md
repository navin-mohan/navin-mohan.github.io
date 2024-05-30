---
title: "Immutability — Raw vs Smart Pointers"
date: 2022-06-07T00:26:16+05:30
tags: [cpp]
showtags: true
summary: "A note on immutability of raw and smart pointers in C++."
url: /blog/2022/06/immutability-raw-vs-smart-pointers/
---
Let’s start with some (raw) pointer basics.

```cpp
const std::string* p = nullptr;
```

Now tell me what is the type of  `p`.  A pointer to `const std::string` . That was easy!

How about this one? What’s the type of  `q` ?

```cpp
std::string * const q = nullptr; 
```

In case you’re wondering, both are valid C++ code — here’s the [proof](https://godbolt.org/z/8E4W9oTo5). 

Little bit tricky but if you start from the identifier  `q`  and read outwards, it becomes clear that  `q`  is constant pointer  to  `std::string`.

Still confused?

Here’s a snippet detailing the difference.

```cpp
const std::string* p = nullptr;
p = new std::string("Hello"); // valid
*p = std::string("World"); // invalid
```

The pointer  `p`  can be modified but not the object pointed to by  `p` .

And for  `q`.

```cpp
std::string * const q = nullptr; 
q = new std::string("Hello"); // invalid
*q = std::string("World"); // valid
```

The pointer `q` cannot be modified since it is declared `const` but  the object pointed to by  `q`  is mutable.

Further, if you want an immutable pointer to an immutable object you could even do the following.

```cpp
const std::string * const s = nullptr;
s = new std::string("Hello"); // invalid
*s = std::string("World"); // invalid
```

Great job so far! Now you know how to use immutable pointers, pointers to immutable objects and immutable pointers to immutable objects.

## Enter Smart Pointers
> Smart pointers exist to save us hundreds of hours of debugging resource leaks which are often the result of a false sense of confidence that one is smart enough to handle raw pointers — that feeling you just had when you completed the previous section.  

Here’s a more [technical definition](https://www.educative.io/edpresso/what-are-smart-pointers) if you’re unfamiliar with smart pointers in C++.

Let’s make  `p`,  `q`, and  `s`  smart.

> Going forward I’ll be using  `std::unique_ptr`  for all the examples. It can be substituted with  `std::shared_ptr`  or  `std::weak_ptr`  to get the equivalent  smart pointers.  

### Mutable Pointer to an Immutable Object
```cpp
const std::unique_ptr<std::string> smart_p = nullptr;
```
If you guessed the above, then you’re wrong. That is actually an immutable unique pointer to a  mutable object.

Let’s see why.  

The devil is in the details.  `std::unique_ptr`  is a template class  defined in the C++ standard library which takes the pointer type as a template argument. So when  `const`  is prepended to the type name, it doesn’t affect the template argument and instead makes the resulting `std::unique_ptr<T>`  object immutable.

To make the object pointed by `std::unique_ptr` immutable, the template argument needs to contain `const`  specifier. 

The smart pointer equivalent of  `p` looks like the following.

```cpp
std::unique_ptr<const std::string> smart_p = nullptr;
```

### Immutable Pointer to an Mutable Object
We accidentally stumbled upon this in the previous example. Here’s the smart pointer equivalent of  `q`.

```cpp
const std::unique_ptr<std::string> smart_q = nullptr;
```

### Immutable Pointer to an Immutable Object
Similar to the raw pointer case, we can make the `std::unique_ptr`  and its template argument  `const`  to declare an immutable pointer to an immutable object.

```cpp
const std::unique_ptr<const std::string> smart_s = nullptr;
```

Hope you learned something new about pointers today! 😄



