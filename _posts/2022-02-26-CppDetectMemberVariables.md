---
title: "Experimental Reflect C++ and detecting member variables with C++20"
date: 2022-02-26
tags: [C++, metaprogramming, template, member variable, detect member variable,
	cpp, programming, templates, has_member, concepts, C++20, requires]
header:
  image: "/images/cpp/morris.jpg"
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
te first argument of unpack_sequence needs to be the template class, and the
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



<iframe width="800px" height="200px" src="https://godbolt.org/e#z:OYLghAFBqRAWIDGB7AJgUwKKoJYBdkAnAGhxAgDMcAbdAOwEMBbdEAcgEY3iLk68Ayoga0QHABw8%2BeAKoBndAAUAHuwAM3AFZji1BnVCIApACYAQqbPEFtRHhx9y9VAGFk1AK5M6IEwDZiZwAZHDp0ADkvACN0QhAAFgBmYgAHZDl8Bzo3T29fALSM%2Bz4QsMimGLik63RbYroBPAZCPByvH38auqzG5rxSiOjYhOS5Jpa2vM6xvoHyypGASmtkD0JEVjZTRNDETwwAaiNEl3RlFNicFn4RAHpCdAo648wjNQBBbd399COTscIoWAAH0AG44dAAdxeb0%2BJh2dD2HkOxxczUIDAAnjCPl9ET8/i4HAD0MwcZ8PgCPHYDgIjgB2Cy4j4HA63ABUBzcdCpdiIB3Zt1hrJQPLwZxShFpEFCeAOdCGJAOY1QIBAAKBYIhkIO4uUeEWBxAByY81iEAVFViy2FrJNABUzngIHqDQyLPSACKw22isYSqUMDwEA4UDyI%2BoQQ1%2BuUAWheB1ByBwqHdRi9PpZbM5AFl0FapaDmjgGFFaHIBUKs7KTWbCMcme9WSq1RqDFqoQ6nQ2fRnEo3YRyDgAJWoXKXipgpPTig57BhyCu8KXhlIMRAAayBJvQTVjyCimnQNIUAEcPPQNpXYZPpwxZ6j54uAHSvg728lPiuj6jjtMfFIPDLHBEBAW0ZnsRA5z4f1zilFsQHRLFUQQtsQXBKFiGVHAAC90GQChX2fCB7UWBNgF3cJmHQOQo3/Js7QePA1jod0EIlS5riaag1QeJ5jzwNUKLwYFGBYMFUQ/RJMCI9NvX7W05N7eSBw%2BIdHSnGdfjDCMsl1OB7wOJiWIrPBCAvA4cAoSy5SoOhUArUwTFNRUnMs1i8DgX5anzeg8DkW0DgI5VkBYaCblCWJWUFG98zvB8Ti/LlpAYSL62k2EYwDA4gxDAy5GBFyCzBYtS1oCBuSaNL5WouRDXjaSDiiZB3Ho1llwOCAYxy4NkFMPwarC40xJow100bO1LOsi1qL%2BeTPSOExnLrJyxsZIzdxY3VzPQBsGW9LMlKzYzCFYigRAUHtcQzZl3hrJhUroOjxszBiRrkNcrz4uo5uVPBVRADjAS4kReMeZ4FNu1lbQ8DIDBNZBfu%2BgMIAEMjIYY2HtxYYqzwvRFfmOBbkbsITd2BVB7wYQr80qAq8FRJg%2BoyjGYbh4ADlXdcN3QVMiY2/jSZALnN2BPHL3QYEGZOH9xywnHKjF9BzwlmFWazCCQPC2DJR6kN3t%2BkWeYB4SqJYWj0dUhirO1vVJXgBdCrrErATK9AZvNxY1omu0EJQYNCVRRaTAAMVWeza0VRNSrLdAwHjsA3NRIOwC2ABWFw6FTq6GLk2oFDa33/rVf25WTk5g/CRGisqaPXdj0Nw9Tcgk5OFP08z7OMehm6KQYk7WLUHO5LYZYeLYNPuB8NgNGIZB2BcSxLBCtYr22LhiEEmfR%2BIeAQFWPBALwUhyBQKcaGGDhAlwAg4iv3h%2BCEERWA4eIpH4eQlFUbfN4eTYN8hBiFI7AuBj3YJPYg09Z7zzYAAeWDEfIK1llDiD8LGPw8Q5x6HhhAEm9RDQVVCikC%2BUp16LG4FvDQyw95n2IbQEgZAupEJIUgbBII8FZB4DQcUhA5DkCiOobgURQjNExCA7gZ8uKwLoNQMRP8MAPQMKIeROAHh2BwKCGigjAjKGPMGTYs9ZS1G0dQHAUQMSEExG4DA2izJXHEcsJ4DBgByAAGralgRcaeG8H6CGEKIV%2B79ZAKBUNo7QV82FICXuYXQZi%2BGQGWMgFI9Q%2BFsFjCqImxhzCWBMAwA4sYADqIhqD5IKU6DEpS5AGVQMgHUhTRSaN4XpQpFxKb8C1rGWBagDjcBsAJLITh7KTB8FfYIkU6w6EKJkPgwzJnpGmXQOYiodB9PUXwXoEx3DtBWT5NZDRxj9HGcsq%2BMxNm5BGdYA5SyCxiGWHIVY6wX6/3QP/He48IFQO4DAlBaCMFYP0BzXB4N%2Bl8AIYvbJMSDjYHwPydeWE3Dn3oYtRIHByGb0EV7YgG4xBqDULodg8RuBMBxXiz5c92C9JAHiyho9liNIyI4eIQA%3D"></iframe>
