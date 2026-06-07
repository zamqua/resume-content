+++
title = "C++ Review Notes"
author = ["Mohammed Azam"]
date = 2026-06-03T11:27:00-05:00
tags = ["cpp", "programming"]
draft = false
+++

Here are my C++ review notes from my reading of \`A Tour of C++ 3rd Ed\`. I wrote
example code for topics I was interested and wanted to review.


## The Basic {#the-basic}


### Initialization {#initialization}

Choose Uniform Initialization for following reasons:

1.  Prevents narrowing conversion.
2.  Make it clear to the compiler and reader that you're i type, an array, a
    struct, or a class object.nitializing and not declaring a function.
3.  It is called uniform initialization since the same format can be used for
    premitive type, an array, a struct, or a class object.

<!--listend-->

```C++

#include <iostream>
#include <vector>


struct Aircraft {
    std::string manufacturer;
    std::string equipmentCode;
    std::string model;
    int numberOfSeats;

    friend std::ostream& operator<<(std::ostream& os, const Aircraft& aircraft);
};

std::ostream& operator << (std::ostream& os, const Aircraft& aircraft) {
    os << aircraft.manufacturer << " "
       << aircraft.equipmentCode << " "
       << aircraft.model << " "
       << aircraft.numberOfSeats;
    return os;
}

int main()
{
    double d1 = 2.3;   // Copy initialization
    double d2 (2.3);   // Direct initialization
    double d3 {2.3};   // Direct list initialization or Uniform Initialization or Brace Initialization
    double d4 = {2.3}; // Copy list initialization

    std::cout
      << "d1:" << d1 << ", "
      << "d2:" << d2 << ", "
      << "d3:" << d3 << ", "
      << "d4:" << d4 << ", "
      << std::endl;


    Aircraft aircraft {"Boeing", "73H", "737-MAX 8", 300};
    std::cout << "Aircraft: "
              << aircraft.manufacturer << " "
              << aircraft.equipmentCode << " "
              << aircraft.model << std::endl;

    std::vector<Aircraft> aircrafts {
      {"Boeing", "73H", "737-MAX 8", 300},
      {"Airbus", "A321", "A350-900", 500},
      {"Embraer", "E876", "E876-700", 350}
    };

    for (const auto& aircraft : aircrafts) {
      std::cout << aircraft << std::endl;
    }

    return 0;
  }

```

```text
d1:2.3, d2:2.3, d3:2.3, d4:2.3,
Aircraft: Boeing 73H 737-MAX 8
Boeing 73H 737-MAX 8 300
Airbus A321 A350-900 500
Embraer E876 E876-700 350
```


### Constants: const and constexpr {#constants-const-and-constexpr}

1.  const means "this value won't change" but the value itself can be known at
    either compile time or runtime.
2.  constexpr is an explicit hint to the compiler that the value must be known at
    compile time. The compiler verifies this and refuses to compile if it can't.
3.  The compiler does not automatically trace through code to find compile-time
    values — it needs constexpr hints to do so, which keeps compile times
    reasonable.
4.  There are places in C++ where a compile-time constant is required — such as
    array sizes and template arguments.
5.  For constexpr to work on an object, the entire chain must be constexpr — the
    object, the function using it, and the types involved must all be evaluable
    at compile time.
6.  constexpr on a function means it can be evaluated at compile time when given
    compile-time inputs, but also runs normally at runtime when given runtime
    inputs — avoiding the need for duplicate functions.

<!--listend-->

```C++
#include <iostream>
#include <string>
#include <string_view>


struct Aircraft {
    std::string manufacturer;
    std::string equipmentCode;
    std::string model;
    int numberOfSeats;

    friend std::ostream& operator<<(std::ostream& os, const Aircraft& aircraft);
};

std::ostream& operator << (std::ostream& os, const Aircraft& aircraft) {
    os << aircraft.manufacturer << " "
       << aircraft.equipmentCode << " "
       << aircraft.model << " "
       << aircraft.numberOfSeats;
    return os;
}

enum class Placement {Window, Middle, Aisle};
enum class CabinClass {Economy, PremiumEconomy, Business, First};
enum class SeatType {Standard, AngleFlat, LieFlat};

struct Seat {
  std::string seatNumber; // 12A
  Placement placement;    // W
  CabinClass cabinClass;
  SeatType seatType;

  friend std::ostream& operator<<(std::ostream& os, const Seat& seat);
};

std::ostream& operator << (std::ostream& os, const Seat& seat) {

   // Why use constexpr for lambda?
   // It gives flexibility to be used in both compile-time and runtime contexts
   // without duplication. It signals to the compiler that this is a pure,
   // side-effect-free function, enabling potential optimizations
   constexpr auto placeementStr = [] (Placement placement) -> std::string_view {
    switch (placement) {
      using enum Placement;
      case Window: return "Windows";
      case Aisle: return "Aisle";
      case Middle: return "Middle";
      default: return "Unknown";
    }
  };
   constexpr auto cabinClassStr = [] (CabinClass cabinClass) -> std::string_view {
    switch (cabinClass) {
      using enum CabinClass;
      case Economy: return "Economy";
      case PremiumEconomy: return "PremiumEconomy";
      case Business: return "Business";
      case First: return "First";
      default: return "Unknown";
    }
  };
  constexpr auto seatTypeStr = [] (SeatType seatType) -> std::string_view {
    switch (seatType) {
      using enum SeatType;
      case Standard: return "Standard";
      case AngleFlat: return "AngleFlat";
      case LieFlat: return "LieFlat";
      default: return "Unknown";
    }
  };

  os << seat.seatNumber << " "
     << placeementStr(seat.placement) << " "
     << cabinClassStr(seat.cabinClass) << " "
     << seatTypeStr(seat.seatType) << std::endl;
  return os;
}

constexpr int getSeatSizeBasedOnAircraft(const Aircraft& aircraft) {
   return aircraft.numberOfSeats - 10;
}

int main()
{
  constexpr Aircraft aircraft {"Boeing", "73H", "737-MAX 8", 300};
  std::array<Seat, getSeatSizeBasedOnAircraft(aircraft)> seats;
  seats[0] = {"12A", Placement::Window, CabinClass::Economy, SeatType::Standard};
  std::cout << seats[0] << std::endl;
}

```

```text
12A Windows Economy Standard
```


#### Use of constexpr in templates {#use-of-constexpr-in-templates}

```C++
#include <iostream>
#include <type_traits>

template <typename T>
auto my_add(T a, T b) {
  if constexpr (std::is_same<T, const char*>::value) {
    if (a && b) {
      return (std::string(a) + std::string(b));
    }
    return std::string("");
  }
  else {
    return a + b;
  }
}

int main() {
  std::cout << my_add(1, 2) << std::endl;
  std::cout << my_add(1.1, 2.2) << std::endl;

  std::string a("a"), b("b");
  std::cout << my_add(a, b) << std::endl;

  std::cout << my_add("A", "B") << std::endl;
}
```

```text
3
3.3
ab
AB
```


### Pointers, Arrays, and References {#pointers-arrays-and-references}

```C++
char v[3];
v[0]='a'; v[1]='b'; v[2]='c';
// can't increment (v++), it is like "char* const"
std::cout << *v << *(v+1) << *(v+2) << std::endl;
```

```C++
#include <iostream>

constexpr int SIZE = 3;

void foo1(const char* const p) {
  //p[0]='m';
  //char x[SIZE]; x[0]='c'; x[1]='b'; x[2]='a';
  //p = x;
  std::cout << *(p+0) << *(p+1) << *(p+2) << "//";
}

// p is pointer to `const char`. You can modify its content but you can
// reassigne it.
void foo2(const char* p) {
  // p[0]='m';
  char x[SIZE]; x[0]='c'; x[1]='b'; x[2]='a';
  p = x;
  std::cout << *p++ << *p++ << *p++ << "//";
}

// p is const `char*`. You can't reassigne it but you can modify its content.
void foo3(char* const p) {
  p[0]='m';
  //char x[SIZE]; x[0]='c'; x[1]='b'; x[2]='a';
  //p = x;
  std::cout << *(p+0) << *(p+1) << *(p+2) << "//";
}

// r is an alias (reference) to the first element of the the array v.
void foo4(char& r) {
  r='n';
  std::cout << r << "//";
}

// r is an alias (const reference) to the first element of the the array v.
void foo5(const char& r) {
  //r='n';
  std::cout << r << "//";
}

void print(char v[]) {
  for (size_t i = 0; i < SIZE; ++i) {
    std::cout << v[i];
  }
  std::cout << std::endl;
}

int main() {
  // array v is a pointer to the first element of contagious memory. It can't
  // move (char* const) but when passed to a function it decays.
  char v[SIZE]; v[0]='a'; v[1]='b'; v[2]='c';
  foo1(v); print(v);
  foo2(v); print(v);
  foo3(v); print(v);
  foo4(*v); print(v);
  foo5(*v); print(v);
  return 0;
}
```

```text
abc//abc
cba//abc
mbc//mbc
n//nbc
n//nbc
```

```C++
#include <iostream>

void print(int n[], int size) {
  for (size_t i = 0; i < size; ++i) {
    std::cout << n[i];
  }
  std::cout << std::endl;
}

int main() {
  int n[] = {0,1,2,3,4,5,6,7,8,9}; print(n,sizeof(n)/sizeof(int));
  for (auto& x : n) ++x; print(n,sizeof(n)/sizeof(int));
  return 0;
}

```

```text
0123456789
12345678910
```

1.  Array decays to pointer when passed in a fucntion.
2.  \`nullptr\` is a pointer and not an integer.
3.  Reference must be initialized.
4.  Reference cannot be null.
5.  Reference cannot be reassigned
6.  References have cleaner syntax.
7.  We still need pointers:
    1.  Pointers upport pointer arithmetic.
    2.  Pointers can be nullptr - useful to represent nothing or optional values.


## User-Defined Type {#user-defined-type}


### enum class {#enum-class}

```C++
#include <iostream>
#include <string>

enum Color { red=1, green, blue };
enum class TrafficLight  { red=100, green, yellow };

std::string getTL(TrafficLight tl) {
  if (tl == TrafficLight::red) return "Red";
  else if (tl == TrafficLight::green) return "Green";
  else if (tl == TrafficLight::yellow) return "Yellow";
  else return "Bad TL";
}

void print(TrafficLight tl) {
  std::cout << getTL(tl) << std::endl;
}

int main() {
  int red = 0;
  int color = red;
  std::cout << "color: " << color << std::endl;

  TrafficLight tl = TrafficLight::red; print(tl);
  return 0;
}

```


### Union {#union}

-   **union** is like a struct but all the data members are allocated at the same address.
-   The size of the **union** is the size of its biggest member.
-   use [std::variant](https://cppreference.com/cpp/utility/variant) instead of **union** to avoid errors.


## Modularity {#modularity}


### import vs include: {#import-vs-include}

-   \#include is a preprocessor macro that literally copies header content into
    each translation unit — leading to slow builds and macro pollution
-   import (C++20 modules) imports a precompiled interface — faster builds, no
    macro leakage, and explicit control over what is exposed via export
-   Modules are compiled once and reused, unlike headers which are recompiled for
    every translation unit that includes them
-   Create std.pcm

<!--listend-->

```text
clang++ -std=c++23 -stdlib=libc++ -Wno-reserved-module-identifier --precompile /usr/lib/llvm-18/share/libc++/v1/std.cppm -o /<path>/std.pcm
```

-   Cleate an alias to build with "import std".

<!--listend-->

```text
alias mcpp="/opt/homebrew/opt/llvm/bin/clang++ -std=c++23 -stdlib=libc++ -isysroot /Library/Developer/CommandLineTools/SDKs/MacOSX.sdk -fmodule-file=/<path>/std=std.pcm"
```

```C++
import std;

int main() {
    std::cout << "Hello from Org-mode with C++23 Modules!\n";
    return 0;
}



```

```text
Hello from Org-mode with C++23 Modules!
```


### Structure Binding {#structure-binding}

<a id="table--pax-profile"></a>

|       |                 |
|-------|-----------------|
| Alice | alice@gmail.com |

```C++
import std;

struct Passenger {
  std::string name;
  std::string emailAddress;

  friend std::ostream& operator << (std::ostream& os, const Passenger& pax);
};

std::ostream& operator << (std::ostream& os, const Passenger& pax) {
  os << pax.name << " " << pax.emailAddress;
  return os;
}

Passenger readPassenger() {
    // data[0][0] = "Alice"
    // data[0][1] = "alice@gmail.com"
    std::string name {data[0][0]};
    std::string emailAddress {data[0][1]};
  return {name, emailAddress};
}

int main() {
  Passenger pax = readPassenger();
  std::cout << pax << std::endl;

  // structure binding: mechanism for giving local names to members of a class
  // object. Here is an example of unpacking and giving local names.
  // const auto &[pax_name, pax_email] = pax;
  // std::cout << "Passenger: " << pax_name << ": " << pax_email << std::endl;
  return 0;
}

```

```text
Alice alice@gmail.com
```


## Error Handling {#error-handling}

-   Termination — when the system cannot continue at all (e.g. out of memory,
    unrecoverable state). Can also happen implicitly when an exception goes
    uncaught.
-   Exceptions — for unexpected states where the immediate caller cannot handle
    the situation and it needs to bubble up the call stack (e.g. offer expired in
    offer store).
-   Error codes — for expected, handleable outcomes that the immediate caller can
    directly deal with (e.g. no inventory found).
-   **std::optional** — when absence of a value is a normal state, not an error
    (e.g. account code not found but pricing can continue, optional middle name).
-   RAII ties into all of these — resources are automatically cleaned up via
    destructors during stack unwinding, preventing leaks regardless of how the
    error is handled.
-   Performance — exceptions have a cost (stack unwinding, destructor calls) and
    may not be appropriate for resource constrained environments like embedded or
    IoT systems.


## Classes {#classes}

-   Classes are blueprints for user-defined types that encapsulate data and
    behavior through attributes and methods
-   Encapsulation is enforced through access specifiers — private, protected, and
    public
-   Private members are only accessible within the class itself
-   Public members are accessible from anywhere
-   Protected members are accessible within the class and by derived classes, but
    not by instances from outside
-   Inheritance types control how access specifiers carry over to derived classes:
    -   Public: public→public, protected→protected
    -   Protected: public→protected, protected→protected
    -   Private: everything→private, cutting off further inheritance

<!--listend-->

```C++

import std;

class Salary {
public:
  Salary(const Salary&) = delete;
  Salary& operator=(const Salary&) = delete;
  Salary(const Salary&&) = delete;
  Salary& operator=(const Salary&&) = delete;

  static Salary& instance() {
    static Salary salary;
    return salary;
  }

  std::optional<double> getSalary(int employeeNumber) {
    auto res = _salaryCol.find(employeeNumber);
    if (res != _salaryCol.end())
      return res->second;
    return std::nullopt;
  }
private:
  Salary() {
    _salaryCol[1] = 4000000;
    _salaryCol[2] = 2000000;
    _salaryCol[3] = 1000000;
    _salaryCol[4] = 200000;
  }
  std::map<int, double> _salaryCol;
};

constexpr std::string CEO_NAME{"Tom"};
constexpr int CEO_EMP_ID{1};
constexpr double CEO_STOCK_OPTION{100000};

struct Address {std::string street; std::string city; std::string state; int number; int zip;};
class Employee {
public:
  Employee(std::string name, std::string title, int employeeNumber):
    _name(std::move(name)), _title(std::move(title)), _employeeNumber(employeeNumber), _salary{Salary::instance()}
  {}

  std::optional<double> getEmployeeSalary() const {
    return _salary.getSalary(_employeeNumber);
  }
  friend std::ostream& operator << (std::ostream& os, const Employee& e);
protected:
  Address _address;
private:
  std::string _name;
  std::string _title;
  int _employeeNumber;
  Salary& _salary;
};

std::ostream& operator << (std::ostream& os, const Employee& e) {
  auto res = e.getEmployeeSalary();
  os << e._name << " " << e._title;
  if (res) {
    os << " " << *res;
  }
  return os;
}

class CEO : public Employee {
public:
  CEO(const CEO&) = delete;
  CEO& operator=(const CEO&) = delete;
  CEO(const CEO&&) = delete;
  CEO& operator=(const CEO&&) = delete;

  static const CEO& instance() {
    static const CEO ceo;
    return ceo;
  }

  std::optional<double> getEmployeeSalary() {
    auto res = getEmployeeSalary();
    if (res) {
      return (*res) * 0.50 + (_stockOptions * 0.05);
    }
    return res;
  }
private:
  CEO():Employee(CEO_NAME, "CEO", CEO_EMP_ID),_stockOptions(CEO_STOCK_OPTION) {}
  double _stockOptions;
};


  int main() {
    const CEO& ceo{CEO::instance()};
    std::vector<Employee> employees{ceo, {"John", "CFO", 2}, {"Mary", "CTO", 3}, {"Harry", "Engineer", 4}};

    for (const auto& employee : employees) {
      std::cout << employee << std::endl;
    }
    return 0;
  }
```

```text
Tom CEO 4e+06
John CFO 2e+06
Mary CTO 1e+06
Harry Engineer 200000
```
