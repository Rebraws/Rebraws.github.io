---
title: "Experimental Reflect C++ and detecting member variables with C++20"
date: 2022-02-26
tags: [C++, metaprogramming, template, member variable, detect member variable,
	cpp, programming, templates, has_member, concepts, C++20, requires]
header:
  image: "/images/cpp/morris.png"
---


### Reflect C++

Reflective programming is a language feature that provides the ability to
examine, introspect and even modify its structure. C++ does not
support reflection, but Matus Chochlik wrote a reflection TS implementation for clang. 

We will not use this implementation because it's not part of the standard yet, 
but I think it's worth seeing a few examples of how it works before proceeding.


All examples can be tested using compile explorer with x86-64 clang (reflection)
compiler 


The first thing we need it's a class to analyze
```
struct S {
  
  /* Constructor */
  constexpr S(int number, std::string_view text) : mNumber(number), 
    mText(text) {}

  /* Member variables */
  int mNumber;
  std::string_view mText;  
};
```


Once we have a class we need a meta-object reflecting the class S, we can get
this meta-object with reflexpr()

```
#include <experimental/reflect>
#include <iostream>
#include <string_view>

struct S { ... }

int main() {
  
  using mo = reflexpr(S);  

  return EXIT_SUCCESS;
}

```

Now that we have a meta-object, we can inspect the properties of our class

We can use get_data_members to get a meta-object sequence of all 
the member variables in our structure, and then we can use get_size to get
the size of the meta-object sequence (How many member variables the class has)

```
int main() {

  using mo = reflexpr(S);
  using member_sequence = std::experimental::reflect::get_data_members_t<mo>;

  std::cout << std::experimental::reflect::get_size<member_sequence>::value;

}
```

Now suppose that we want to print the name of the first S member variable, for this
task we can use get_element and get_name_v

Let's see the code

Note: Since the syntax is getting more complex I'm going to rename the namespace
to something simpler

```
int main() {

  namespace reflect = std::experimental::reflect;

  using mo = reflexpr(S);
  using member_sequence = reflect::get_data_members_t<mo>;

  std::cout << reflect::get_name_v<reflect::get_element_t<0, member_sequence>>;

}
```

Quite simple, right? But what happens if we have a huge class with many member variables
and we want to get all the variable names, well fortunately member_sequence is 
a meta-object sequence type and we can use unpack_sequence with a helper class
to achieve this.

Our helper class will look like this


```
template <class... T>
class Helper {  // Note: We could have used a struct too
public:
  static constexpr std::array<std::string_view, sizeof...(T)> getNames() {
    return {std::experimental::reflect::get_name_v<T>...};
  }
};
```

Now we can unpack the meta-object sequence with unpack_sequence, 
the first argument of unpack_sequence needs to be the template class, and the
second argument is the meta-object sequence type

So the code will look like this:

```
int main() {
  namespace reflect = std::experimental::reflect;

  using mo = reflexpr(S);
  using member_sequence = reflect::get_data_members_t<mo>;
   
  using unpacked = reflect::unpack_sequence_t<Helper, member_sequence>;

  static constexpr auto names = unpacked::getNames();

}
```

So now we have an array (names) with all the member variables name from the 
struct S, and for example we could make a function that returns true if 
there is a variable with the name "mNumber"

We could have something like this:

```
template <class Container>
constexpr auto has_mNumber_variable(Container names) -> bool {
  for (const auto& name : names) {
    if (name == "mNumber") { return true; }
  }
  return false;
}

int main() {
  namespace reflect = std::experimental::reflect;

  using mo = reflexpr(S);
  using member_sequence = reflect::get_data_members_t<mo>;
   
  using unpacked = reflect::unpack_sequence_t<Helper, member_sequence>;

  static constexpr auto names = unpacked::getNames();

  if constexpr(has_mNumber_variable(names)) { 
    std::cout << "Found mNumber variable!!" << '\n';
  } else {
    std::cout << "No mNumber variable found :(" << '\n';
  }
  
  return 0;
}
```

You can test this code in compile explorer: [link](https://godbolt.org/z/W7fb5o4br) 




### Detecting member variables with C++20

In the previous example we saw how to detect member variables using reflection, 
but as we said reflect is not part of the standard, so now we are going to see
how to detect member variables with C++20

There are several ways to do this, we are going to do two of them, first we are
going to implement it with templates and at the end we will do it with concepts
which is a new feature in C++20


We are going to use the same structure from the previous example
```
struct S {
  
  /* Constructor */
  constexpr S(int number, std::string_view text) : mNumber(number), 
    mText(text) {}

  /* Member variables */
  int mNumber;
  std::string_view mText;  
};
```

Now we want to find if the structure has some variable with the name "mNumber"

So we will declare a template class like this:

```
template <class T, class U = int>
struct has_member_variable {
  static constexpr bool value{false};
};
```

The has_member_variable template class contains a boolean attribute defaulted
to false, so if we test this template class we will notice that 
has_member_variable::value is always false

In order to detect if a variable exists we need to do a template specialization
like this

```
template <class T, class U = int>
struct has_member_variable {
  static constexpr bool value{false};
};

template <class T>
struct has_member_variable<T, decltype((void)T::mNumber, 0)> {
  static constexpr bool value{true};
};  

```


So now that we have a template specialization, when we instantiate the template
has_member_variable, if the class contains a member variable with the name mNumber
the compiler is going to use the specialized template

Let's see has_member_variable in an example


```
int main() {

  if constexpr (has_member_variable<S>::value) {
    return 5;
  }
  
  return 0;
} 

```


So in this example if the class S contains mNumber as a member variable
then the program returns 5, otherwise it returns 0

You can see the full example in compiler explorer: [link](https://godbolt.org/z/6cWfb59ed)



Now that we have seen how to use templates to detect if a member variable exists
we are going to do it again but using concepts, concepts are a new feature in C++20
and they are basically named boolean predicates on template parameters that are evaluated
at compile time

The definition of a concept has the following form:

```
template <parameters>
concept concept-name = constraint-expression;
```

Note that a constraint is a sequence of logical operations and operands 
that specifies requirements on template arguments


So now that we know what concepts are we will do a concept named has_member_variable 

```
template <class T>
concept has_member_variable = requires (T t) { t.mNumber; };
```

Now we have a concept named has_member_variable that requires to the template
argument to have a mNumber variable

We can test this concept with a program exactly like the one form the previous example


```
#include <string_view>

struct S {

  /* Constructor */
  constexpr S(int number, std::string_view text) : mNumber(number),
    mText(text) {}

  /* Member variables */
  int mNumber;
  std::string_view mText;
};

template <class T>
concept has_member_variable = requires (T t) { t.mNumber; };



int main() {
  
  if constexpr(has_member_variable<S>) {
    return 5; 
  } 

  return 0;
}

```

You can try this code in compile explorer: [link](https://godbolt.org/z/9f3x58vWe)
 


