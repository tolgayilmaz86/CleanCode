# Clean Code

[ToC]

## Clean Code Rules

### General Rules

- Follow standard conventions.
- **Keep it simple, stupid.** Simpler is always better. Reduce complexity as much as possible.
- **Boy scout rule.** Leave the campground cleaner than you found it.
- **Always find the root cause.** Always look for the root cause of a problem.

### Design Rules

- Keep configurable data at high levels.
- Prefer polymorphism to if/else or switch/case.
- Separate multi-threading code.
- Prevent over-configurability.
- Use dependency injection.
- **Follow the Law of Demeter.** A class should know only its direct dependencies.

### Understandability Tips

- **Be consistent.** If you do something a certain way, do all similar things in the same way.
- Use explanatory variables.
- **Encapsulate boundary conditions.** Boundary conditions are hard to keep track of. Put the processing for them in one place.
- Prefer dedicated value objects to primitive types.
- **Avoid logical dependency.** Don't write methods that work correctly only because of something else in the same class.
- Avoid negative conditionals.

### Names Rules

- Choose descriptive and unambiguous names.
- Make meaningful distinctions.
- Use pronounceable names.
- Use searchable names.
- Replace magic numbers with named constants.
- **Avoid encodings.** Don't append prefixes or type information.

### Functions Rules

- Small.
- Do one thing.
- Use descriptive names.
- Prefer fewer arguments.
- Have no side effects.
- **Don't use flag arguments.** Split the method into several independent methods that the client can call without the flag.

### Comments Rules

- Always try to explain yourself in code.
- Don't be redundant.
- Don't add obvious noise.
- Don't use closing brace comments.
- **Don't comment out code.** Just remove it.
- Use as explanation of intent.
- Use as clarification of code.
- Use as warning of consequences.

### Source Code Structure

- Separate concepts vertically.
- Related code should appear vertically dense.
- Declare variables close to their usage.
- Dependent functions should be close.
- Similar functions should be close.
- Place functions in the downward direction.
- Keep lines short.
- Don't use horizontal alignment.
- Use white space to associate related things and disassociate weakly related ones.
- Don't break indentation.

### Objects and Data Structures

- Hide internal structure.
- Prefer data structures.
- Avoid hybrid structures (half object and half data).
- Should be small.
- Do one thing.
- Small number of instance variables.
- Base class should know nothing about its derivatives.
- Better to have many functions than to pass some code into a function to select a behavior.
- Prefer non-static methods to static methods.

### Tests

- One assert per test.
- Readable.
- Fast.
- Independent.
- Repeatable.

### Code Smells

- **Rigidity.** The software is difficult to change. A small change causes a cascade of subsequent changes.
- **Fragility.** The software breaks in many places due to a single change.
- **Immobility.** You cannot reuse parts of the code in other projects because of the risks and effort involved.
- **Needless complexity.**
- **Needless repetition.**
- **Opacity.** The code is hard to understand.

---

## Refactoring Examples

This section provides small, concrete examples of common refactoring methods, grouped like the [refactoring.guru catalog](https://refactoring.guru/refactoring/techniques). Each section includes:

- A short introduction to the technique
- Two code samples side-by-side (**Before** / **After**)
- A quick summary of the benefits of the refactored version

All examples use modern C++ (C++20). Where these notes use a different name for a technique, the refactoring.guru name is shown next to it.

---

### Composing Methods

Break long methods into well-named pieces and remove needless indirection.

---

#### Extract Method

Use this when a code fragment can be grouped together and given a name. Move it into its own function and replace the fragment with a call.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
#include <format>
#include <iostream>
#include <string>
#include <vector>

struct Order {
    std::string customer;
    std::vector<double> amounts;
};

void printOwing(const Order& order) {
    std::cout << "*** Customer Owes ***\n";

    double outstanding = 0.0;
    for (double amount : order.amounts) {
        outstanding += amount;
    }

    std::cout << std::format("name: {}\n",
                             order.customer);
    std::cout << std::format("amount: {:.2f}\n",
                             outstanding);
}
```

</td>
<td>

```cpp
#include <format>
#include <iostream>
#include <string>
#include <vector>

struct Order {
    std::string customer;
    std::vector<double> amounts;
};

void printBanner() {
    std::cout << "*** Customer Owes ***\n";
}

double outstanding(const Order& order) {
    double result = 0.0;
    for (double amount : order.amounts) {
        result += amount;
    }
    return result;
}

void printDetails(const Order& order,
                  double amount) {
    std::cout << std::format("name: {}\n",
                             order.customer);
    std::cout << std::format("amount: {:.2f}\n",
                             amount);
}

void printOwing(const Order& order) {
    printBanner();
    printDetails(order, outstanding(order));
}
```

</td>
</tr>
</table>

**Benefits**
- Each function reads as one named step.
- Smaller pieces are easier to test and reuse.
- Comments that explained a block become unnecessary.

---

#### Inline Method

Use this when a function body is as clear as its name. Replace the calls with the body and delete the function.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
class Driver {
public:
    int rating() const {
        return moreThanFiveLateDeliveries() ? 2 : 1;
    }

private:
    bool moreThanFiveLateDeliveries() const {
        return lateDeliveries_ > 5;
    }

    int lateDeliveries_ = 0;
};
```

</td>
<td>

```cpp
class Driver {
public:
    int rating() const {
        return lateDeliveries_ > 5 ? 2 : 1;
    }

private:
    int lateDeliveries_ = 0;
};
```

</td>
</tr>
</table>

**Benefits**
- Removes indirection that adds nothing.
- One less name for readers to learn.
- Code reads straight through.

---

#### Extract Variable

Use this when an expression is hard to understand. Put its parts into `const` variables with names that explain them.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
#include <algorithm>

struct Order {
    int quantity;
    double itemPrice;
};

double price(const Order& o) {
    return o.quantity * o.itemPrice -
           std::max(0, o.quantity - 500) *
               o.itemPrice * 0.05 +
           std::min(o.quantity * o.itemPrice * 0.1,
                    100.0);
}
```

</td>
<td>

```cpp
#include <algorithm>

struct Order {
    int quantity;
    double itemPrice;
};

double price(const Order& o) {
    const double basePrice =
        o.quantity * o.itemPrice;
    const double quantityDiscount =
        std::max(0, o.quantity - 500) *
        o.itemPrice * 0.05;
    const double shipping =
        std::min(basePrice * 0.1, 100.0);
    return basePrice - quantityDiscount + shipping;
}
```

</td>
</tr>
</table>

**Benefits**
- Names explain what each part of the formula means.
- Long expressions become readable.
- Intermediate values are easy to inspect in a debugger.

---

#### Inline Temp

Use this when a temporary variable holds the result of a simple expression and adds nothing. Replace its uses with the expression itself.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
struct Order {
    double basePrice() const { return price_; }
    double price_ = 0.0;
};

bool hasDiscount(const Order& order) {
    double basePrice = order.basePrice();
    return basePrice > 1000;
}
```

</td>
<td>

```cpp
struct Order {
    double basePrice() const { return price_; }
    double price_ = 0.0;
};

bool hasDiscount(const Order& order) {
    return order.basePrice() > 1000;
}
```

</td>
</tr>
</table>

**Benefits**
- Removes clutter.
- One less name to track.
- Often a first step before Extract Method.

---

#### Replace Temp with Query

Use this when a local variable stores the result of an expression that other methods could also use. Move the expression into a method and call it instead.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
class Order {
public:
    double price() const {
        const double basePrice =
            quantity_ * itemPrice_;
        if (basePrice > 1000) {
            return basePrice * 0.95;
        }
        return basePrice * 0.98;
    }

private:
    int quantity_ = 0;
    double itemPrice_ = 0.0;
};
```

</td>
<td>

```cpp
class Order {
public:
    double price() const {
        return basePrice() * discountFactor();
    }

private:
    double basePrice() const {
        return quantity_ * itemPrice_;
    }

    double discountFactor() const {
        return basePrice() > 1000 ? 0.95 : 0.98;
    }

    int quantity_ = 0;
    double itemPrice_ = 0.0;
};
```

</td>
</tr>
</table>

**Benefits**
- The calculation is reusable by other methods.
- Methods get shorter.
- Fewer locals make further extractions easier.

---

#### Split Temporary Variable

Use this when one local variable is reused to hold different values. Give each value its own variable, ideally `const`.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
#include <iostream>

void printRectangle(double height,
                    double width) {
    double temp = 2 * (height + width);
    std::cout << "perimeter: " << temp << "\n";
    temp = height * width;
    std::cout << "area: " << temp << "\n";
}
```

</td>
<td>

```cpp
#include <iostream>

void printRectangle(double height,
                    double width) {
    const double perimeter = 2 * (height + width);
    std::cout << "perimeter: " << perimeter
              << "\n";
    const double area = height * width;
    std::cout << "area: " << area << "\n";
}
```

</td>
</tr>
</table>

**Benefits**
- Each variable has one meaning.
- `const` stops accidental reuse.
- Each piece is easier to extract later.

---

#### Remove Assignments to Parameters

Use this when a function assigns new values to its parameters. Use a local variable instead, so a parameter always means what the caller passed in.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
int discount(int inputVal, int quantity) {
    if (quantity > 50) {
        inputVal -= 2;
    }
    if (inputVal > 100) {
        inputVal -= 1;
    }
    return inputVal;
}
```

</td>
<td>

```cpp
int discount(const int inputVal,
             const int quantity) {
    int result = inputVal;
    if (quantity > 50) {
        result -= 2;
    }
    if (result > 100) {
        result -= 1;
    }
    return result;
}
```

</td>
</tr>
</table>

**Benefits**
- Parameters keep their original meaning.
- `const` parameters make the rule compiler-checked.
- The flow of values is easier to follow.

---

#### Replace Method with Method Object

Use this when a long method has so many tangled local variables that Extract Method is impractical. Turn the method into a class whose fields are the locals, then split it freely.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
class Account {
public:
    int gamma(int inputVal, int quantity,
              int yearToDate) const {
        int importantValue1 =
            inputVal * quantity + delta();
        int importantValue2 =
            inputVal * yearToDate + 100;
        if (yearToDate - importantValue1 > 100) {
            importantValue2 -= 20;
        }
        int importantValue3 = importantValue2 * 7;
        // ... many more lines using all locals
        return importantValue3 -
               2 * importantValue1;
    }

    int delta() const { return 42; }
};
```

</td>
<td>

```cpp
class Account {
public:
    int gamma(int inputVal, int quantity,
              int yearToDate) const;
    int delta() const { return 42; }
};

class Gamma {
public:
    Gamma(const Account& account, int inputVal,
          int quantity, int yearToDate)
        : account_(account), inputVal_(inputVal),
          quantity_(quantity),
          yearToDate_(yearToDate) {}

    int compute() {
        value1_ = inputVal_ * quantity_ +
                  account_.delta();
        value2_ = inputVal_ * yearToDate_ + 100;
        adjustForYearToDate();
        value3_ = value2_ * 7;
        return value3_ - 2 * value1_;
    }

private:
    void adjustForYearToDate() {
        if (yearToDate_ - value1_ > 100) {
            value2_ -= 20;
        }
    }

    const Account& account_;
    int inputVal_, quantity_, yearToDate_;
    int value1_ = 0, value2_ = 0, value3_ = 0;
};

inline int Account::gamma(int inputVal,
                          int quantity,
                          int yearToDate) const {
    return Gamma(*this, inputVal, quantity,
                 yearToDate).compute();
}
```

</td>
</tr>
</table>

**Benefits**
- Breaks up a huge method without passing locals around.
- Each step becomes a small private method.
- The original class stays uncluttered.

---

#### Substitute Algorithm

Use this when you find a clearer or simpler way to compute the same result. Replace the whole body with the new algorithm.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
#include <string>
#include <vector>

std::string foundPerson(
    const std::vector<std::string>& people) {
    for (const auto& person : people) {
        if (person == "Don") {
            return "Don";
        }
        if (person == "John") {
            return "John";
        }
        if (person == "Kent") {
            return "Kent";
        }
    }
    return "";
}
```

</td>
<td>

```cpp
#include <algorithm>
#include <array>
#include <string>
#include <string_view>
#include <vector>

std::string foundPerson(
    const std::vector<std::string>& people) {
    constexpr std::array<std::string_view, 3>
        candidates{"Don", "John", "Kent"};
    const auto it = std::ranges::find_first_of(
        people, candidates);
    return it != people.end() ? *it : "";
}
```

</td>
</tr>
</table>

**Benefits**
- Simpler code is easier to verify.
- Standard algorithms state the intent.
- The candidate list is data, so it is easy to change.

---

### Moving Features between Objects

Put behavior and data in the class where they belong.

---

#### Move Method

Use this when a method uses another class's data more than its own. Move it to the class it depends on and leave a forwarding call (or nothing) behind.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
#include <string>

class Customer {
public:
    std::string country;
    int loyaltyYears = 0;
};

class Order {
public:
    explicit Order(const Customer& c)
        : customer_(c) {}

    double discountRate() const {
        if (customer_.loyaltyYears > 5) {
            return 0.10;
        }
        return customer_.country == "TR" ? 0.05
                                         : 0.0;
    }

private:
    const Customer& customer_;
};
```

</td>
<td>

```cpp
#include <string>

class Customer {
public:
    std::string country;
    int loyaltyYears = 0;

    double discountRate() const {
        if (loyaltyYears > 5) {
            return 0.10;
        }
        return country == "TR" ? 0.05 : 0.0;
    }
};

class Order {
public:
    explicit Order(const Customer& c)
        : customer_(c) {}

    double discountRate() const {
        return customer_.discountRate();
    }

private:
    const Customer& customer_;
};
```

</td>
</tr>
</table>

**Benefits**
- Behavior lives next to the data it uses.
- Reduces coupling between classes.
- Other users of `Customer` can reuse the rule.

---

#### Move Field

Use this when a field is used more by another class, or describes another concept. Here the interest rate depends on the account type, so it moves to `AccountType`.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
class AccountType {
public:
    bool isPremium() const { return premium_; }
private:
    bool premium_ = false;
};

class Account {
public:
    double interest(double amount,
                    int days) const {
        return interestRate_ * amount * days / 365;
    }

private:
    AccountType type_;
    double interestRate_ = 0.03;
};
```

</td>
<td>

```cpp
class AccountType {
public:
    bool isPremium() const { return premium_; }
    double interestRate() const {
        return interestRate_;
    }
private:
    bool premium_ = false;
    double interestRate_ = 0.03;
};

class Account {
public:
    double interest(double amount,
                    int days) const {
        return type_.interestRate() * amount *
               days / 365;
    }

private:
    AccountType type_;
};
```

</td>
</tr>
</table>

**Benefits**
- Data lives with the concept it describes.
- Changing the rate for a type is a single edit.
- Prepares the ground for moving related methods.

---

#### Extract Class

Use this when one class is doing too much and has multiple responsibilities. Extract a focused class to reduce size and clarify ownership of data and behavior.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
#include <string>

class UserProfile {
public:
    UserProfile(std::string name,
                std::string email,
                std::string street,
                std::string city,
                std::string postalCode)
        : name_(std::move(name)),
          email_(std::move(email)),
          street_(std::move(street)),
          city_(std::move(city)),
          postalCode_(std::move(postalCode)) {}

    std::string formattedAddress() const {
        return street_ + ", " +
               city_ + " " + postalCode_;
    }

private:
    std::string name_;
    std::string email_;
    std::string street_;
    std::string city_;
    std::string postalCode_;
};
```

</td>
<td>

```cpp
#include <string>

class Address {
public:
    Address(std::string street,
            std::string city,
            std::string postalCode)
        : street_(std::move(street)),
          city_(std::move(city)),
          postalCode_(std::move(postalCode)) {}

    std::string formatted() const {
        return street_ + ", " +
               city_ + " " + postalCode_;
    }

private:
    std::string street_;
    std::string city_;
    std::string postalCode_;
};

class UserProfile {
public:
    UserProfile(std::string name,
                std::string email,
                Address address)
        : name_(std::move(name)),
          email_(std::move(email)),
          address_(std::move(address)) {}

private:
    std::string name_;
    std::string email_;
    Address address_;
};
```

</td>
</tr>
</table>

**Benefits**
- Reduces class size and complexity.
- Separates responsibilities (address logic lives with address).
- Improves reuse (address can be shared across other models).

---

#### Inline Class

Use this when a class does almost nothing and no new responsibilities are planned for it. Move its features into the class that uses it and delete it. It is the inverse of [Extract Class](#extract-class).

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
#include <string>

class TelephoneNumber {
public:
    std::string areaCode;
    std::string number;
};

class Person {
public:
    std::string phone() const {
        return "(" + phone_.areaCode + ") " +
               phone_.number;
    }

private:
    std::string name_;
    TelephoneNumber phone_;
};
```

</td>
<td>

```cpp
#include <string>

class Person {
public:
    std::string phone() const {
        return "(" + areaCode_ + ") " + number_;
    }

private:
    std::string name_;
    std::string areaCode_;
    std::string number_;
};
```

</td>
</tr>
</table>

**Benefits**
- Removes a class that didn't pull its weight.
- Fewer types to navigate.
- Undoes an extraction that didn't pay off.

---

#### Hide Delegate

Use this when clients reach through one object to get another (`a.b().c()`). Add a delegating method so clients don't depend on the chain. It is the inverse of [Remove Middle Man](#remove-middle-man).

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
#include <string>
#include <utility>

class Department {
public:
    explicit Department(std::string manager)
        : manager_(std::move(manager)) {}
    const std::string& manager() const {
        return manager_;
    }
private:
    std::string manager_;
};

class Person {
public:
    explicit Person(const Department& d)
        : department_(d) {}
    const Department& department() const {
        return department_;
    }
private:
    const Department& department_;
};

// Client must know about Department
std::string managerOf(const Person& p) {
    return p.department().manager();
}
```

</td>
<td>

```cpp
#include <string>
#include <utility>

class Department {
public:
    explicit Department(std::string manager)
        : manager_(std::move(manager)) {}
    const std::string& manager() const {
        return manager_;
    }
private:
    std::string manager_;
};

class Person {
public:
    explicit Person(const Department& d)
        : department_(d) {}
    const std::string& manager() const {
        return department_.manager();
    }
private:
    const Department& department_;
};

// Client only knows about Person
std::string managerOf(const Person& p) {
    return p.manager();
}
```

</td>
</tr>
</table>

**Benefits**
- Clients are shielded from changes to the delegate.
- Follows the Law of Demeter.
- Fewer classes are coupled to `Department`.

---

#### Remove Middle Man

Use this when a class does nothing but delegate to another object. Remove the pass-through and let clients talk directly to the delegate.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
#include <string>

class Department {
public:
    std::string getManagerName() const {
        return managerName_;
    }
private:
    std::string managerName_ = "Alice";
};

// Middle man - just forwards calls
class Employee {
public:
    explicit Employee(Department& dept)
        : dept_(dept) {}

    std::string getManagerName() const {
        return dept_.getManagerName();
    }

    Department& getDepartment() {
        return dept_;
    }

private:
    Department& dept_;
};

// Usage
auto name = employee.getManagerName();
```

</td>
<td>

```cpp
#include <string>

class Department {
public:
    std::string getManagerName() const {
        return managerName_;
    }
private:
    std::string managerName_ = "Alice";
};

class Employee {
public:
    explicit Employee(Department& dept)
        : dept_(dept) {}

    Department& getDepartment() {
        return dept_;
    }

private:
    Department& dept_;
};

// Usage - talk to Department directly
auto name = employee.getDepartment()
                    .getManagerName();
```

</td>
</tr>
</table>

**Benefits**
- Reduces pointless delegation methods.
- Makes the real dependency visible.
- Simplifies the middle class.

---

#### Introduce Foreign Method

Use this when a library class lacks a method you need and you can't change it. In C++, write a free function that takes the library type as its first argument.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
#include <chrono>

using namespace std::chrono;

void issueInvoice(year_month_day previousEnd) {
    // "Next day" written inline wherever needed
    const year_month_day newStart{
        sys_days{previousEnd} + days{1}};
    // ...
}
```

</td>
<td>

```cpp
#include <chrono>

using namespace std::chrono;

// std::chrono has no nextDay(), so add one
[[nodiscard]] constexpr year_month_day
nextDay(year_month_day date) {
    return sys_days{date} + days{1};
}

void issueInvoice(year_month_day previousEnd) {
    const auto newStart = nextDay(previousEnd);
    // ...
}
```

</td>
</tr>
</table>

**Benefits**
- The logic gets a name and is reused instead of copied.
- Callers read like the domain ("next day").
- Easy to move into a richer type later.

---

#### Introduce Local Extension

Use this when you need several extra methods on a library class you can't change. Create a wrapper that adds them, so the extensions live in one place. Prefer wrapping over inheriting from standard types, which aren't designed as base classes.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
#include <chrono>

using namespace std::chrono;

// Helpers scattered across the codebase
year_month_day nextDay(year_month_day d) {
    return sys_days{d} + days{1};
}

bool isWeekend(year_month_day d) {
    const weekday wd{sys_days{d}};
    return wd == Saturday || wd == Sunday;
}
```

</td>
<td>

```cpp
#include <chrono>

using namespace std::chrono;

class Date {
public:
    constexpr explicit Date(year_month_day d)
        : ymd_(d) {}

    constexpr Date nextDay() const {
        return Date{sys_days{ymd_} + days{1}};
    }

    constexpr bool isWeekend() const {
        const weekday wd{sys_days{ymd_}};
        return wd == Saturday || wd == Sunday;
    }

    constexpr year_month_day value() const {
        return ymd_;
    }

private:
    year_month_day ymd_;
};
```

</td>
</tr>
</table>

**Benefits**
- All extensions live in one class.
- Callers get a richer, domain-specific type.
- Wrapping avoids inheriting from types not built for it.

---

### Organizing Data

Replace primitives and raw data with types that carry meaning.

---

#### Change Value to Reference

Use this when many identical copies of an object should really be one shared entity, so a change to it is seen everywhere. Get the object from a repository instead of creating copies.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
#include <string>
#include <utility>

class Customer {
public:
    explicit Customer(std::string name)
        : name_(std::move(name)) {}
    void setCreditLimit(double l) { limit_ = l; }
private:
    std::string name_;
    double limit_ = 0.0;
};

class Order {
public:
    explicit Order(const std::string& customer)
        : customer_(customer) {}  // own copy
private:
    Customer customer_;
};
```

</td>
<td>

```cpp
#include <map>
#include <memory>
#include <string>
#include <utility>

class Customer {
public:
    explicit Customer(std::string name)
        : name_(std::move(name)) {}
    void setCreditLimit(double l) { limit_ = l; }
private:
    std::string name_;
    double limit_ = 0.0;
};

class CustomerRepository {
public:
    std::shared_ptr<Customer>
    get(const std::string& name) {
        auto& slot = customers_[name];
        if (!slot) {
            slot = std::make_shared<Customer>(name);
        }
        return slot;
    }
private:
    std::map<std::string,
             std::shared_ptr<Customer>> customers_;
};

class Order {
public:
    Order(const std::string& customer,
          CustomerRepository& repo)
        : customer_(repo.get(customer)) {}
private:
    std::shared_ptr<Customer> customer_;
};
```

</td>
</tr>
</table>

**Benefits**
- One object per real-world entity.
- Updates are visible to everyone holding it.
- No duplicated state drifting apart.

---

#### Change Reference to Value

Use this when a shared object is small and rarely changes, so sharing it only adds lifetime management. Make it an immutable value compared by content. It is the inverse of [Change Value to Reference](#change-value-to-reference).

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
#include <memory>
#include <string>
#include <utility>

class Currency {
public:
    explicit Currency(std::string code)
        : code_(std::move(code)) {}
    const std::string& code() const {
        return code_;
    }
    void setCode(std::string c) {
        code_ = std::move(c);
    }
private:
    std::string code_;
};

struct Money {
    double amount;
    std::shared_ptr<Currency> currency;
};

// Compares identity, not content
bool sameCurrency(const Money& a,
                  const Money& b) {
    return a.currency == b.currency;
}
```

</td>
<td>

```cpp
#include <string>
#include <utility>

class Currency {
public:
    explicit Currency(std::string code)
        : code_(std::move(code)) {}
    const std::string& code() const {
        return code_;
    }
    bool operator==(const Currency&) const = default;
private:
    std::string code_;
};

struct Money {
    double amount;
    Currency currency;
};

bool sameCurrency(const Money& a,
                  const Money& b) {
    return a.currency == b.currency;
}
```

</td>
</tr>
</table>

**Benefits**
- No shared ownership or lifetimes to manage.
- Equality by content is what callers expect.
- Immutable values are safe to copy and share across threads.

---

#### Duplicate Observed Data

Use this when domain data is stored only inside a UI class. Move the data into a domain object and keep the UI in sync through an observer, so the logic doesn't depend on widgets.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
#include <string>

struct TextField {
    std::string text;
};

// Domain logic lives in the window
class IntervalWindow {
public:
    void onEndChanged() {
        const int start =
            std::stoi(startField_.text);
        const int end = std::stoi(endField_.text);
        lengthField_.text =
            std::to_string(end - start);
    }

private:
    TextField startField_, endField_, lengthField_;
};
```

</td>
<td>

```cpp
#include <functional>
#include <string>
#include <utility>
#include <vector>

// Domain object: no UI types
class Interval {
public:
    using Observer = std::function<void()>;

    void onChange(Observer o) {
        observers_.push_back(std::move(o));
    }
    void setEnd(int end) {
        end_ = end;
        notify();
    }
    int length() const { return end_ - start_; }

private:
    void notify() {
        for (auto& o : observers_) o();
    }

    int start_ = 0, end_ = 0;
    std::vector<Observer> observers_;
};

struct TextField {
    std::string text;
};

class IntervalWindow {
public:
    explicit IntervalWindow(Interval& interval)
        : interval_(interval) {
        interval_.onChange([this] { refresh(); });
    }
    void onEndChanged() {
        interval_.setEnd(std::stoi(endField_.text));
    }

private:
    void refresh() {
        lengthField_.text =
            std::to_string(interval_.length());
    }

    Interval& interval_;
    TextField endField_, lengthField_;
};
```

</td>
</tr>
</table>

**Benefits**
- Domain logic can be tested without a UI.
- Several views can observe the same data.
- UI classes shrink to presentation only.

---

#### Self Encapsulate Field

Use this when a class reads its own fields directly but you want one point of control, for example for validation, lazy loading or letting subclasses change how a value is obtained. Access the fields through accessors, even inside the class.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
class Range {
public:
    Range(int low, int high)
        : low_(low), high_(high) {}

    bool includes(int value) const {
        return value >= low_ && value <= high_;
    }

private:
    int low_;
    int high_;
};
```

</td>
<td>

```cpp
#include <algorithm>

class Range {
public:
    Range(int low, int high)
        : low_(low), high_(high) {}
    virtual ~Range() = default;

    bool includes(int value) const {
        return value >= low() && value <= high();
    }

protected:
    virtual int low() const { return low_; }
    virtual int high() const { return high_; }

private:
    int low_;
    int high_;
};

class CappedRange final : public Range {
public:
    using Range::Range;
protected:
    int high() const override {
        return std::min(Range::high(), 100);
    }
};
```

</td>
</tr>
</table>

**Benefits**
- One place to add validation, logging or lazy loading.
- Subclasses can redefine how data is obtained.
- Less code depends on the raw representation.

---

#### Replace Data Value with Object

Use this when a primitive field (often a string) starts needing its own behavior or validation. Turn it into a small class.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
#include <string>
#include <utility>

class Order {
public:
    explicit Order(std::string customer)
        : customer_(std::move(customer)) {}

    const std::string& customerName() const {
        return customer_;
    }

private:
    std::string customer_;  // just a string...
};
```

</td>
<td>

```cpp
#include <stdexcept>
#include <string>
#include <utility>

class Customer {
public:
    explicit Customer(std::string name)
        : name_(std::move(name)) {
        if (name_.empty()) {
            throw std::invalid_argument(
                "customer name is empty");
        }
    }
    const std::string& name() const {
        return name_;
    }

private:
    std::string name_;
};

class Order {
public:
    explicit Order(Customer customer)
        : customer_(std::move(customer)) {}

    const std::string& customerName() const {
        return customer_.name();
    }

private:
    Customer customer_;
};
```

</td>
</tr>
</table>

**Benefits**
- Validation lives in one place.
- A named type is clearer than a raw string.
- Room to grow (address, credit rating) without touching `Order`.

---

#### Replace Array with Object

Use this when an array holds different kinds of data by position. Replace it with a struct whose fields have names and proper types.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
#include <array>
#include <string>

void report() {
    std::array<std::string, 2> row;
    row[0] = "Liverpool";  // team name
    row[1] = "15";         // wins
    const int wins = std::stoi(row[1]);
    // ...
}
```

</td>
<td>

```cpp
#include <string>

struct Performance {
    std::string name;
    int wins = 0;
};

void report() {
    const Performance row{.name = "Liverpool",
                          .wins = 15};
    const int wins = row.wins;
    // ...
}
```

</td>
</tr>
</table>

**Benefits**
- Fields have names instead of magic indexes.
- Each field has a proper type, so no parsing.
- The struct can gain methods later.

---

#### Change Unidirectional Association to Bidirectional

Use this when two classes each need the other, but the link only goes one way. Add the back link and keep both sides in sync from a single place.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
#include <memory>
#include <string>
#include <utility>

class Customer {
public:
    std::string name;
};

class Order {
public:
    void setCustomer(std::shared_ptr<Customer> c) {
        customer_ = std::move(c);
    }
private:
    std::shared_ptr<Customer> customer_;
};

// A Customer cannot list its orders
```

</td>
<td>

```cpp
#include <memory>
#include <set>
#include <string>
#include <utility>

class Order;

class Customer {
public:
    std::string name;
    const std::set<Order*>& orders() const {
        return orders_;
    }
private:
    friend class Order;  // only Order edits links
    std::set<Order*> orders_;  // non-owning
};

class Order {
public:
    ~Order() { setCustomer(nullptr); }

    void setCustomer(std::shared_ptr<Customer> c) {
        if (customer_) {
            customer_->orders_.erase(this);
        }
        customer_ = std::move(c);
        if (customer_) {
            customer_->orders_.insert(this);
        }
    }
private:
    std::shared_ptr<Customer> customer_;
};
```

</td>
</tr>
</table>

**Benefits**
- Both sides can navigate the relationship.
- One method keeps the link consistent.
- No need to scan every order to find a customer's.

---

#### Change Bidirectional Association to Unidirectional

Use this when one direction of a two-way link is no longer used. Remove it to cut coupling and the code that keeps both sides in sync.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
#include <set>

class Order;

class Customer {
public:
    void addOrder(Order* o) { orders_.insert(o); }
    void removeOrder(Order* o) { orders_.erase(o); }
private:
    std::set<Order*> orders_;  // never read
};

class Order {
public:
    void setCustomer(Customer* c) {
        if (customer_) customer_->removeOrder(this);
        customer_ = c;
        if (customer_) customer_->addOrder(this);
    }
private:
    Customer* customer_ = nullptr;
};
```

</td>
<td>

```cpp
class Customer {
    // No knowledge of orders
};

class Order {
public:
    void setCustomer(const Customer* c) {
        customer_ = c;
    }
private:
    const Customer* customer_ = nullptr;
};
```

</td>
</tr>
</table>

**Benefits**
- Less coupling between the classes.
- No sync logic to get wrong.
- `Customer` can be used without `Order`.

---

#### Encapsulate Field

Use this when a class exposes a public field that has rules attached to it. Make it private and add accessors, so the class controls how it's read and changed.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
#include <string>

class Person {
public:
    std::string name;
    int age = 0;  // nothing stops age = -5
};

void birthday(Person& p) {
    p.age = p.age + 1;
}
```

</td>
<td>

```cpp
#include <stdexcept>
#include <string>
#include <utility>

class Person {
public:
    const std::string& name() const {
        return name_;
    }
    void setName(std::string name) {
        name_ = std::move(name);
    }
    int age() const { return age_; }
    void setAge(int age) {
        if (age < 0) {
            throw std::invalid_argument("age < 0");
        }
        age_ = age;
    }

private:
    std::string name_;
    int age_ = 0;
};

void birthday(Person& p) {
    p.setAge(p.age() + 1);
}
```

</td>
</tr>
</table>

**Benefits**
- The class can enforce its invariants.
- The representation can change without breaking callers.
- Every modification goes through one searchable place.

---

#### Encapsulate Collection

Use this when a class hands out its collection directly, so callers can change it behind its back. Return a read-only view and add methods to add or remove elements.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
#include <string>
#include <vector>

class Person {
public:
    std::vector<std::string>& courses() {
        return courses_;
    }
private:
    std::vector<std::string> courses_;
};

// Callers mutate freely
void enroll(Person& p) {
    p.courses().push_back("Refactoring");
    p.courses().clear();  // oops
}
```

</td>
<td>

```cpp
#include <span>
#include <string>
#include <utility>
#include <vector>

class Person {
public:
    std::span<const std::string> courses() const {
        return courses_;
    }
    void addCourse(std::string course) {
        courses_.push_back(std::move(course));
    }
    void removeCourse(const std::string& course) {
        std::erase(courses_, course);
    }
private:
    std::vector<std::string> courses_;
};

void enroll(Person& p) {
    p.addCourse("Refactoring");
}
```

</td>
</tr>
</table>

**Benefits**
- The class keeps control of its contents.
- `std::span` gives callers a cheap, read-only view.
- Changes are explicit and easy to find.

---

#### Replace Magic Number with Symbolic Constant

Use this when a literal number with a special meaning appears in code. Give it a name with a `constexpr` constant, or use a standard one.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
double potentialEnergy(double mass,
                       double height) {
    return mass * 9.81 * height;
}

double circleArea(double radius) {
    return 3.14159 * radius * radius;
}
```

</td>
<td>

```cpp
#include <numbers>

constexpr double gravity = 9.81;  // m/s^2

double potentialEnergy(double mass,
                       double height) {
    return mass * gravity * height;
}

double circleArea(double radius) {
    return std::numbers::pi * radius * radius;
}
```

</td>
</tr>
</table>

**Benefits**
- The name explains what the number means.
- A single place to change the value.
- `std::numbers::pi` is more precise than a hand-typed literal.

---

#### Replace Type Code with Class

Use this when a field holds a type code (int or string constants) that doesn't change behavior. Replace it with a dedicated type so the compiler rejects invalid values. In modern C++ this is usually an `enum class`.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
class Person {
public:
    static constexpr int O = 0;
    static constexpr int A = 1;
    static constexpr int B = 2;
    static constexpr int AB = 3;

    explicit Person(int bloodGroup)
        : bloodGroup_(bloodGroup) {}

private:
    int bloodGroup_;
};

Person p{7};  // compiles, but 7 is invalid
```

</td>
<td>

```cpp
enum class BloodGroup { O, A, B, AB };

class Person {
public:
    explicit Person(BloodGroup bloodGroup)
        : bloodGroup_(bloodGroup) {}

private:
    BloodGroup bloodGroup_;
};

Person p{BloodGroup::A};  // 7 won't compile
```

</td>
</tr>
</table>

**Benefits**
- Values are type-checked.
- Can't be mixed up with other integers.
- Signatures document themselves.

---

#### Replace Type Code with Subclasses

Use this when a type code that never changes drives behavior through conditionals. Create a subclass for each code and move the behavior, and any code-specific data, into them.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
enum class EmployeeType {
    Engineer, Salesperson, Manager
};

class Employee {
public:
    explicit Employee(EmployeeType t)
        : type_(t) {}

    double payAmount(double salary) const {
        switch (type_) {
            case EmployeeType::Engineer:
                return salary;
            case EmployeeType::Salesperson:
                return salary + commission_;
            case EmployeeType::Manager:
                return salary + bonus_;
        }
        return salary;
    }

private:
    EmployeeType type_;
    double commission_ = 500;  // sales only
    double bonus_ = 1000;      // managers only
};
```

</td>
<td>

```cpp
class Employee {
public:
    virtual ~Employee() = default;
    virtual double payAmount(double salary) const {
        return salary;
    }
};

class Engineer final : public Employee {};

class Salesperson final : public Employee {
public:
    double payAmount(double salary) const override {
        return salary + commission_;
    }
private:
    double commission_ = 500;
};

class Manager final : public Employee {
public:
    double payAmount(double salary) const override {
        return salary + bonus_;
    }
private:
    double bonus_ = 1000;
};
```

</td>
</tr>
</table>

**Benefits**
- Each subclass holds only the data it needs.
- No `switch` to update when a type is added.
- The compiler checks every override.

---

#### Replace Type Code with State/Strategy

Use this when a type code drives behavior but can change at runtime, so subclasses won't work. Move the behavior into separate state objects that can be swapped. With a fixed set of states, `std::variant` is a lightweight modern alternative to a class hierarchy.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
enum class EmployeeType {
    Engineer, Salesperson, Manager
};

class Employee {
public:
    void promote(EmployeeType t) { type_ = t; }

    double payAmount(double salary) const {
        switch (type_) {
            case EmployeeType::Engineer:
                return salary;
            case EmployeeType::Salesperson:
                return salary + 500;
            case EmployeeType::Manager:
                return salary + 1000;
        }
        return salary;
    }

private:
    EmployeeType type_ = EmployeeType::Engineer;
};
```

</td>
<td>

```cpp
#include <variant>

struct Engineer {
    double pay(double s) const { return s; }
};
struct Salesperson {
    double pay(double s) const { return s + 500; }
};
struct Manager {
    double pay(double s) const { return s + 1000; }
};

using Role =
    std::variant<Engineer, Salesperson, Manager>;

class Employee {
public:
    void promote(Role role) { role_ = role; }

    double payAmount(double salary) const {
        return std::visit(
            [salary](const auto& role) {
                return role.pay(salary);
            },
            role_);
    }

private:
    Role role_ = Engineer{};
};
```

</td>
</tr>
</table>

**Benefits**
- Behavior can change at runtime.
- Each role's rules live in their own type.
- `std::visit` won't compile if a role is missing `pay()`.

---

#### Remove Subclass

*refactoring.guru calls this **Replace Subclass with Fields**.*

Use this when a subclass does too little to justify its existence—often just a flag or a single override. Replace with a field or parameter in the parent.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
#include <string>

class Employee {
public:
    virtual ~Employee() = default;
    virtual std::string getType() const = 0;

protected:
    std::string name_;
};

// Subclass just to return a string
class FullTimeEmployee : public Employee {
public:
    std::string getType() const override {
        return "full-time";
    }
};

// Subclass just to return a string
class Contractor : public Employee {
public:
    std::string getType() const override {
        return "contractor";
    }
};
```

</td>
<td>

```cpp
#include <string>

enum class EmployeeType {
    FullTime,
    Contractor
};

class Employee {
public:
    Employee(std::string name, EmployeeType type)
        : name_(std::move(name)), type_(type) {}

    std::string getType() const {
        switch (type_) {
            case EmployeeType::FullTime:
                return "full-time";
            case EmployeeType::Contractor:
                return "contractor";
        }
        return "unknown";
    }

private:
    std::string name_;
    EmployeeType type_;
};
```

</td>
</tr>
</table>

**Benefits**
- Reduces class hierarchy complexity.
- Avoids subclasses that exist only for a label.
- Simpler when behavior doesn't actually vary.

---

### Simplifying Conditional Expressions

Make conditional logic easier to read and extend.

---

#### Consolidate Conditional Expression

Use this when several conditions lead to the same result. Combine them into one condition and move it into a well-named function.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
struct Employee {
    int seniority;
    int monthsDisabled;
    bool isPartTime;
};

double disabilityAmount(const Employee& e) {
    if (e.seniority < 2) {
        return 0;
    }
    if (e.monthsDisabled > 12) {
        return 0;
    }
    if (e.isPartTime) {
        return 0;
    }
    return 100.0 * e.seniority;
}
```

</td>
<td>

```cpp
struct Employee {
    int seniority;
    int monthsDisabled;
    bool isPartTime;
};

bool isEligibleForDisability(const Employee& e) {
    return e.seniority >= 2 &&
           e.monthsDisabled <= 12 && !e.isPartTime;
}

double disabilityAmount(const Employee& e) {
    if (!isEligibleForDisability(e)) {
        return 0;
    }
    return 100.0 * e.seniority;
}
```

</td>
</tr>
</table>

**Benefits**
- One check instead of a list of look-alike ones.
- The function name states the rule.
- The rule can be reused and tested on its own.

---

#### Consolidate Duplicate Conditional Fragments

Use this when the same code appears in every branch of a conditional. Move it outside the conditional.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
void send();

double finalPrice(double price,
                  bool isSpecialDeal) {
    double total = 0.0;
    if (isSpecialDeal) {
        total = price * 0.95;
        send();
    } else {
        total = price * 0.98;
        send();
    }
    return total;
}
```

</td>
<td>

```cpp
void send();

double finalPrice(double price,
                  bool isSpecialDeal) {
    const double total =
        price * (isSpecialDeal ? 0.95 : 0.98);
    send();
    return total;
}
```

</td>
</tr>
</table>

**Benefits**
- No duplicated statements to keep in sync.
- The branches show only what actually differs.
- Shorter code.

---

#### Decompose Conditional

Use this when a complex conditional, meaning its condition and its branches, is hard to read. Move the condition and each branch into functions with descriptive names.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
#include <chrono>

using namespace std::chrono;

double charge(year_month_day date,
              double quantity) {
    if (date.month() >= June &&
        date.month() <= August) {
        return quantity * 1.5 + 20.0;
    }
    return quantity * 1.2;
}
```

</td>
<td>

```cpp
#include <chrono>

using namespace std::chrono;

bool isSummer(year_month_day date) {
    return date.month() >= June &&
           date.month() <= August;
}

double summerCharge(double quantity) {
    return quantity * 1.5 + 20.0;
}

double regularCharge(double quantity) {
    return quantity * 1.2;
}

double charge(year_month_day date,
              double quantity) {
    return isSummer(date) ? summerCharge(quantity)
                          : regularCharge(quantity);
}
```

</td>
</tr>
</table>

**Benefits**
- The code reads like the business rule.
- Each piece is small and testable.
- Names replace explanatory comments.

---

#### Replace Conditional With Polymorphism

Use this when a conditional (e.g., a `switch` on type) selects behavior. Replace it with polymorphism so adding new cases doesn't grow a single conditional hotspot.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
#include <string>

enum class ShippingType {
    Standard, Express, Overnight
};

struct Order {
    ShippingType type;
    double weightKg;
};

double shippingCost(const Order& order) {
    switch (order.type) {
        case ShippingType::Standard:
            return 5.0 + 0.5 * order.weightKg;
        case ShippingType::Express:
            return 12.0 + 0.8 * order.weightKg;
        case ShippingType::Overnight:
            return 25.0 + 1.2 * order.weightKg;
    }
    return 0.0;
}
```

</td>
<td>

```cpp
struct ShippingPolicy {
    virtual ~ShippingPolicy() = default;
    virtual double cost(double weightKg) const = 0;
};

struct StandardShipping final : ShippingPolicy {
    double cost(double weightKg) const override {
        return 5.0 + 0.5 * weightKg;
    }
};

struct ExpressShipping final : ShippingPolicy {
    double cost(double weightKg) const override {
        return 12.0 + 0.8 * weightKg;
    }
};

struct OvernightShipping final : ShippingPolicy {
    double cost(double weightKg) const override {
        return 25.0 + 1.2 * weightKg;
    }
};

double shippingCost(const ShippingPolicy& p,
                    double weightKg) {
    return p.cost(weightKg);
}
```

</td>
</tr>
</table>

**Benefits**
- Eliminates a growing `switch`/`if` chain.
- Makes new behavior additive (add a new policy type).
- Keeps each rule small and focused.

---

#### Remove Control Flag

Use this when a boolean variable is used to control when a loop ends. Use `break`, `continue` or `return` instead.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
#include <string>
#include <vector>

void alert();

void checkSecurity(
    const std::vector<std::string>& people) {
    bool found = false;
    for (const auto& person : people) {
        if (!found) {
            if (person == "Don" || person == "John") {
                alert();
                found = true;
            }
        }
    }
}
```

</td>
<td>

```cpp
#include <string>
#include <vector>

void alert();

void checkSecurity(
    const std::vector<std::string>& people) {
    for (const auto& person : people) {
        if (person == "Don" || person == "John") {
            alert();
            return;
        }
    }
}
```

</td>
</tr>
</table>

**Benefits**
- The control flow is explicit.
- One less variable to track.
- The loop stops as soon as its job is done.

---

#### Replace Nested Conditional with Guard Clauses

Use this when nested conditionals hide the normal path through a function. Handle the special cases first with early returns. Tidy First calls this [Guard Clauses](#1-guard-clauses).

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
struct Employee {
    bool isDead;
    bool isSeparated;
    bool isRetired;
    double salary;
};

double payAmount(const Employee& e) {
    double result = 0.0;
    if (e.isDead) {
        result = 0.0;
    } else {
        if (e.isSeparated) {
            result = 100.0;
        } else {
            if (e.isRetired) {
                result = 200.0;
            } else {
                result = e.salary;
            }
        }
    }
    return result;
}
```

</td>
<td>

```cpp
struct Employee {
    bool isDead;
    bool isSeparated;
    bool isRetired;
    double salary;
};

double payAmount(const Employee& e) {
    if (e.isDead) return 0.0;
    if (e.isSeparated) return 100.0;
    if (e.isRetired) return 200.0;
    return e.salary;
}
```

</td>
</tr>
</table>

**Benefits**
- Flat structure instead of deep nesting.
- The normal path is obvious.
- Each special case is a single line.

---

#### Introduce Special Case

*refactoring.guru calls this **Introduce Null Object**.*

Use this when you have repeated `nullptr` checks (or "missing value" conditionals) throughout the code. Instead, introduce a "special case" object (often `UnknownCustomer`, `NullUser`, etc.) that behaves safely.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
#include <string>

struct Customer {
    std::string name;
    double discountRate;
};

std::string customerName(const Customer* c) {
    if (c == nullptr) return "(guest)";
    return c->name;
}

double applyDiscount(double total,
                     const Customer* c) {
    if (c == nullptr) return total;
    return total * (1.0 - c->discountRate);
}
```

</td>
<td>

```cpp
#include <memory>
#include <string>

struct CustomerLike {
    virtual ~CustomerLike() = default;
    virtual std::string name() const = 0;
    virtual double discountRate() const = 0;
};

struct GuestCustomer final : CustomerLike {
    std::string name() const override {
        return "(guest)";
    }
    double discountRate() const override {
        return 0.0;
    }
};

struct RealCustomer final : CustomerLike {
    RealCustomer(std::string n, double rate)
        : n_(std::move(n)), rate_(rate) {}
    std::string name() const override { return n_; }
    double discountRate() const override { return rate_; }
private:
    std::string n_;
    double rate_;
};

double applyDiscount(double total,
                     const CustomerLike& c) {
    return total * (1.0 - c.discountRate());
}
```

</td>
</tr>
</table>

**Benefits**
- Reduces scattered `nullptr` checks and branching.
- Makes behavior explicit (a guest is a real domain concept).
- Improves testability by centralizing "missing" behavior.

---

#### Introduce Assertion

Use this when code only works if some condition is true, but that assumption is only implied. Make it explicit with an assertion.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
class Project {
public:
    double memberExpenseLimit() const {
        return expenseLimit_;
    }
private:
    double expenseLimit_ = 0.0;
};

class Employee {
public:
    double expenseLimit() const {
        // needs either own limit or a project
        return expenseLimit_ != 0.0
                   ? expenseLimit_
                   : project_->memberExpenseLimit();
    }
private:
    double expenseLimit_ = 0.0;
    const Project* project_ = nullptr;
};
```

</td>
<td>

```cpp
#include <cassert>

class Project {
public:
    double memberExpenseLimit() const {
        return expenseLimit_;
    }
private:
    double expenseLimit_ = 0.0;
};

class Employee {
public:
    double expenseLimit() const {
        assert(expenseLimit_ != 0.0 ||
               project_ != nullptr);
        return expenseLimit_ != 0.0
                   ? expenseLimit_
                   : project_->memberExpenseLimit();
    }
private:
    double expenseLimit_ = 0.0;
    const Project* project_ = nullptr;
};
```

</td>
</tr>
</table>

**Benefits**
- The assumption is visible to readers.
- Violations fail fast in debug builds.
- Replaces a comment that could go stale.

---

### Simplifying Method Calls

Make interfaces simpler and easier to call correctly.

---

#### Add Parameter

Use this when a method needs information from its caller that it doesn't have. Add a parameter, and give it a default value if existing callers should keep working unchanged.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
#include <format>
#include <string>

std::string greeting(const std::string& name) {
    return std::format("Hello, {}!", name);
}
```

</td>
<td>

```cpp
#include <format>
#include <string>
#include <string_view>

std::string greeting(
    std::string_view name,
    std::string_view salutation = "Hello") {
    return std::format("{}, {}!", salutation,
                       name);
}
```

</td>
</tr>
</table>

**Benefits**
- The method gets what it needs instead of using globals.
- The default value keeps existing calls compiling.
- The dependency is visible in the signature.

---

#### Remove Parameter

Use this when a parameter isn't used in the method body. Remove it so callers don't have to pass meaningless values.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
struct Product {
    double price;
};

double priceWithTax(const Product& p,
                    double discount,  // unused
                    double taxRate) {
    return p.price * (1.0 + taxRate);
}

// Callers invent a value: priceWithTax(p, 0, 0.2)
```

</td>
<td>

```cpp
struct Product {
    double price;
};

double priceWithTax(const Product& p,
                    double taxRate) {
    return p.price * (1.0 + taxRate);
}

// priceWithTax(p, 0.2)
```

</td>
</tr>
</table>

**Benefits**
- Simpler signature.
- No meaningless arguments at call sites.
- Less coupling between caller and method.

---

#### Rename Method

Use this when a method's name doesn't explain what it does. Rename it. For a public API, keep the old name for a while as a `[[deprecated]]` forwarder.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
#include <string>

class Person {
public:
    std::string getTelNum() const {
        return phone_;
    }
private:
    std::string phone_;
};
```

</td>
<td>

```cpp
#include <string>

class Person {
public:
    std::string officePhoneNumber() const {
        return phone_;
    }

    [[deprecated("use officePhoneNumber()")]]
    std::string getTelNum() const {
        return officePhoneNumber();
    }

private:
    std::string phone_;
};
```

</td>
</tr>
</table>

**Benefits**
- The name says what the method does.
- `[[deprecated]]` lets callers migrate gradually, guided by compiler warnings.
- Fewer comments needed.

---

#### Separate Query from Modifier

Use this when a method both returns a value and changes state. Split it into a query with no side effects and a modifier.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
#include <cstddef>
#include <vector>

struct Message { /* ... */ };

class Inbox {
public:
    // Returns the count AND marks all as read
    std::size_t takeUnreadCount() {
        const auto count = unread_.size();
        unread_.clear();
        return count;
    }
private:
    std::vector<Message> unread_;
};
```

</td>
<td>

```cpp
#include <cstddef>
#include <vector>

struct Message { /* ... */ };

class Inbox {
public:
    [[nodiscard]] std::size_t unreadCount() const {
        return unread_.size();
    }
    void markAllRead() { unread_.clear(); }
private:
    std::vector<Message> unread_;
};
```

</td>
</tr>
</table>

**Benefits**
- The query can be called freely, for example in logs or tests.
- `const` documents and enforces that it has no side effects.
- `[[nodiscard]]` warns when the result is ignored.

---

#### Parameterize Method

Use this when several methods do the same thing with different values. Merge them into one method that takes the value as a parameter. It is the opposite of [Remove Flag Argument](#remove-flag-argument), which applies when a parameter switches between different behaviors.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
class Employee {
public:
    void tenPercentRaise() { salary_ *= 1.10; }
    void fivePercentRaise() { salary_ *= 1.05; }
private:
    double salary_ = 0.0;
};
```

</td>
<td>

```cpp
class Employee {
public:
    void raise(double percent) {
        salary_ *= 1.0 + percent / 100.0;
    }
private:
    double salary_ = 0.0;
};
```

</td>
</tr>
</table>

**Benefits**
- One implementation instead of copies.
- New values need no new method.
- Less code to maintain.

---

#### Introduce Parameter Object

Use this when you have functions with many parameters that often travel together. A parameter object reduces argument lists and makes call sites clearer.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
#include <chrono>
#include <string>

void scheduleMeeting(
    const std::string& title,
    std::chrono::system_clock::time_point start,
    int durationMinutes,
    const std::string& organizerEmail,
    const std::string& roomName)
{
    // ...
}

// Hard to read at call site
// scheduleMeeting("1:1", start, 30,
//     "alex@company.com", "Room 4");
```

</td>
<td>

```cpp
#include <chrono>
#include <string>

struct MeetingRequest {
    std::string title;
    std::chrono::system_clock::time_point start;
    int durationMinutes;
    std::string organizerEmail;
    std::string roomName;
};

void scheduleMeeting(const MeetingRequest& req) {
    // ...
}

// Clear at call site
// scheduleMeeting({
//     .title = "1:1",
//     .start = start,
//     .durationMinutes = 30,
//     .organizerEmail = "alex@company.com",
//     .roomName = "Room 4"
// });
```

</td>
</tr>
</table>

**Benefits**
- Improves readability at call sites (named fields vs positional args).
- Makes it easier to add new parameters without breaking callers.
- Enables validation/defaulting in one place.

---

#### Preserve Whole Object

Use this when you take several values out of an object only to pass them to a method. Pass the object itself.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
struct TemperatureRange {
    int low;
    int high;
};

class HeatingPlan {
public:
    bool withinRange(int low, int high) const {
        return low >= range_.low &&
               high <= range_.high;
    }
private:
    TemperatureRange range_{18, 24};
};

bool check(const HeatingPlan& plan,
           const TemperatureRange& daysTemp) {
    const int low = daysTemp.low;
    const int high = daysTemp.high;
    return plan.withinRange(low, high);
}
```

</td>
<td>

```cpp
struct TemperatureRange {
    int low;
    int high;
};

class HeatingPlan {
public:
    bool withinRange(
        const TemperatureRange& r) const {
        return r.low >= range_.low &&
               r.high <= range_.high;
    }
private:
    TemperatureRange range_{18, 24};
};

bool check(const HeatingPlan& plan,
           const TemperatureRange& daysTemp) {
    return plan.withinRange(daysTemp);
}
```

</td>
</tr>
</table>

**Benefits**
- Shorter parameter lists.
- Adding a field doesn't change the signature.
- The method can use more of the object later.

---

#### Remove Setting Method

Use this when a field should not change after construction. Remove the setter to make the object immutable or at least more constrained.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
#include <string>

class Person {
public:
    Person(std::string id, std::string name)
        : id_(std::move(id)),
          name_(std::move(name)) {}

    const std::string& getId() const {
        return id_;
    }

    // Dangerous: ID should never change
    void setId(const std::string& id) {
        id_ = id;
    }

    const std::string& getName() const {
        return name_;
    }

    void setName(const std::string& name) {
        name_ = name;
    }

private:
    std::string id_;
    std::string name_;
};
```

</td>
<td>

```cpp
#include <string>

class Person {
public:
    Person(std::string id, std::string name)
        : id_(std::move(id)),
          name_(std::move(name)) {}

    const std::string& getId() const {
        return id_;
    }

    // No setter for id_ - it's immutable

    const std::string& getName() const {
        return name_;
    }

    void setName(const std::string& name) {
        name_ = name;
    }

private:
    const std::string id_;  // now const
    std::string name_;
};
```

</td>
</tr>
</table>

**Benefits**
- Prevents accidental mutation of identity fields.
- Makes invariants explicit and compiler-enforced.
- Easier to reason about object state.

---

#### Remove Flag Argument

*refactoring.guru calls this **Replace Parameter with Explicit Methods**.*

Use this when a boolean or enum argument controls which branch of logic runs. Replace with separate, well-named functions.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
#include <string>

class Booking {
public:
    double calculatePrice(bool isPremium) {
        if (isPremium) {
            return basePrice_ * 0.8;  // 20% off
        } else {
            return basePrice_;
        }
    }

private:
    double basePrice_ = 100.0;
};

// Caller: what does 'true' mean?
auto price = booking.calculatePrice(true);
```

</td>
<td>

```cpp
#include <string>

class Booking {
public:
    double regularPrice() const {
        return basePrice_;
    }

    double premiumPrice() const {
        return basePrice_ * 0.8;
    }

private:
    double basePrice_ = 100.0;
};

// Caller: intent is clear
auto price = booking.premiumPrice();
```

</td>
</tr>
</table>

**Benefits**
- Call sites become self-documenting.
- Removes boolean blindness (what does `true` mean?).
- Each function has a single, clear purpose.

---

#### Replace Parameter with Method Call

Use this when a caller computes a value and passes it in, but the method could get the value itself. Let the method call it and drop the parameter.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
class Order {
public:
    double finalPrice() const {
        const double base = quantity_ * itemPrice_;
        const int level = quantity_ > 100 ? 2 : 1;
        return discountedPrice(base, level);
    }

private:
    double discountedPrice(double base,
                           int level) const {
        return level == 2 ? base * 0.9
                          : base * 0.95;
    }

    int quantity_ = 0;
    double itemPrice_ = 0.0;
};
```

</td>
<td>

```cpp
class Order {
public:
    double finalPrice() const {
        const double base = quantity_ * itemPrice_;
        return discountedPrice(base);
    }

private:
    int discountLevel() const {
        return quantity_ > 100 ? 2 : 1;
    }

    double discountedPrice(double base) const {
        return discountLevel() == 2 ? base * 0.9
                                    : base * 0.95;
    }

    int quantity_ = 0;
    double itemPrice_ = 0.0;
};
```

</td>
</tr>
</table>

**Benefits**
- Shorter parameter lists.
- Callers can't pass an inconsistent value.
- The logic for the value lives in one place.

---

#### Hide Method

Use this when a method isn't used outside its class, or only by subclasses. Make it `private` or `protected`.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
#include <string>

class Report {
public:
    std::string render() const {
        return header() + body_;
    }
    std::string header() const {  // internal only
        return "== Report ==\n";
    }
private:
    std::string body_;
};
```

</td>
<td>

```cpp
#include <string>

class Report {
public:
    std::string render() const {
        return header() + body_;
    }
private:
    std::string header() const {
        return "== Report ==\n";
    }
    std::string body_;
};
```

</td>
</tr>
</table>

**Benefits**
- Smaller public interface.
- Internals can change without breaking callers.
- The class's real API is obvious.

---

#### Replace Constructor With Factory Function

*refactoring.guru calls this **Replace Constructor with Factory Method**.*

Use this when object creation needs naming, defaults, validation, or conditional logic. A factory can enforce invariants and keep call sites clean.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
#include <string>

class Connection {
public:
    Connection(std::string host,
               int port,
               bool useTls,
               int timeoutMs)
        : host_(std::move(host)),
          port_(port),
          useTls_(useTls),
          timeoutMs_(timeoutMs) {}

private:
    std::string host_;
    int port_;
    bool useTls_;
    int timeoutMs_;
};

// Caller must remember defaults
Connection conn("db.local", 5432, true, 10000);
```

</td>
<td>

```cpp
#include <string>

struct ConnectionOptions {
    std::string host;
    int port = 5432;
    bool useTls = true;
    int timeoutMs = 10000;
};

class Connection {
public:
    static Connection create(ConnectionOptions opts) {
        if (opts.host.empty()) {
            opts.host = "localhost";
        }
        return Connection(
            std::move(opts.host),
            opts.port,
            opts.useTls,
            opts.timeoutMs);
    }

private:
    Connection(std::string host, int port,
               bool useTls, int timeoutMs)
        : host_(std::move(host)),
          port_(port),
          useTls_(useTls),
          timeoutMs_(timeoutMs) {}

    std::string host_;
    int port_;
    bool useTls_;
    int timeoutMs_;
};

auto conn = Connection::create({.host = "db.local"});
```

</td>
</tr>
</table>

**Benefits**
- Names the creation intent and centralizes defaults/validation.
- Keeps constructors private if you want to enforce invariants.
- Lets you change construction logic without changing every call site.

---

#### Replace Error Code with Exception

Use this when a function returns a special value to signal an error and every caller has to remember to check it. Throw an exception instead. If the failure is common and expected, C++23's `std::expected` is an alternative.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
class Account {
public:
    // Returns -1 on failure
    int withdraw(int amount) {
        if (amount > balance_) {
            return -1;
        }
        balance_ -= amount;
        return 0;
    }
private:
    int balance_ = 0;
};

void pay(Account& a) {
    if (a.withdraw(100) == -1) {
        // handle error, if someone remembers
    }
}
```

</td>
<td>

```cpp
#include <stdexcept>

class InsufficientFunds
    : public std::runtime_error {
public:
    using std::runtime_error::runtime_error;
};

class Account {
public:
    void withdraw(int amount) {
        if (amount > balance_) {
            throw InsufficientFunds(
                "balance too low");
        }
        balance_ -= amount;
    }
private:
    int balance_ = 0;
};

void pay(Account& a) {
    try {
        a.withdraw(100);
    } catch (const InsufficientFunds&) {
        // handle error
    }
}
```

</td>
</tr>
</table>

**Benefits**
- Errors can't be silently ignored.
- The normal path isn't cluttered with checks.
- The return value is free for real results.

---

#### Replace Exception with Test

Use this when you throw an exception for a condition the caller could easily check first. Check the condition instead and keep exceptions for the unexpected.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
#include <cstddef>
#include <stdexcept>
#include <vector>

double valueAt(const std::vector<double>& values,
               std::size_t index) {
    try {
        return values.at(index);
    } catch (const std::out_of_range&) {
        return 0.0;
    }
}
```

</td>
<td>

```cpp
#include <cstddef>
#include <vector>

double valueAt(const std::vector<double>& values,
               std::size_t index) {
    if (index >= values.size()) {
        return 0.0;
    }
    return values[index];
}
```

</td>
</tr>
</table>

**Benefits**
- The normal flow is easier to follow.
- No exception overhead for an expected case.
- Exceptions stay reserved for real errors.

---

### Dealing with Generalization

Move features up and down class hierarchies, or replace inheritance.

---

#### Pull Up Field

Use this when two or more subclasses have the same field. Move the field to the superclass to reduce duplication and clarify shared data.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
#include <string>

class Employee {
public:
    virtual ~Employee() = default;
};

class Engineer : public Employee {
protected:
    std::string name_;  // duplicated
};

class Manager : public Employee {
protected:
    std::string name_;  // duplicated
};
```

</td>
<td>

```cpp
#include <string>

class Employee {
public:
    virtual ~Employee() = default;

protected:
    std::string name_;  // pulled up
};

class Engineer : public Employee {};

class Manager : public Employee {};
```

</td>
</tr>
</table>

**Benefits**
- Eliminates duplicated field definitions across subclasses.
- Makes shared data explicit in the hierarchy.
- Simplifies subclasses.

---

#### Pull Up Method

Use this when two or more subclasses have identical (or nearly identical) methods. Move the method to the superclass to remove duplication.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
#include <string>

class Employee {
public:
    virtual ~Employee() = default;

protected:
    std::string name_;
};

class Engineer : public Employee {
public:
    std::string getName() const {
        return name_;  // duplicated
    }
};

class Manager : public Employee {
public:
    std::string getName() const {
        return name_;  // duplicated
    }
};
```

</td>
<td>

```cpp
#include <string>

class Employee {
public:
    virtual ~Employee() = default;

    std::string getName() const {
        return name_;  // pulled up
    }

protected:
    std::string name_;
};

class Engineer : public Employee {};

class Manager : public Employee {};
```

</td>
</tr>
</table>

**Benefits**
- Removes duplicated method implementations.
- Single place to maintain the logic.
- Smaller, cleaner subclasses.

---

#### Pull Up Constructor Body

Use this when multiple subclasses repeat the same initialization logic that belongs in the base class. Pull the shared code into the superclass constructor.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
#include <chrono>
#include <string>

class Employee {
protected:
    std::string id;
    std::chrono::system_clock::time_point createdAt;

    Employee(std::string id_,
             std::chrono::system_clock::time_point t)
        : id(std::move(id_)), createdAt(t) {}
};

class Engineer : public Employee {
public:
    explicit Engineer(std::string id_)
        : Employee(std::move(id_),
            std::chrono::system_clock::now()) {}
};

class Manager : public Employee {
public:
    explicit Manager(std::string id_)
        : Employee(std::move(id_),
            std::chrono::system_clock::now()) {}
};
```

</td>
<td>

```cpp
#include <chrono>
#include <string>

class Employee {
protected:
    std::string id;
    std::chrono::system_clock::time_point createdAt;

    explicit Employee(std::string id_)
        : id(std::move(id_)),
          createdAt(std::chrono::system_clock::now()) {}
};

class Engineer : public Employee {
public:
    explicit Engineer(std::string id_)
        : Employee(std::move(id_)) {}
};

class Manager : public Employee {
public:
    explicit Manager(std::string id_)
        : Employee(std::move(id_)) {}
};
```

</td>
</tr>
</table>

**Benefits**
- Removes duplication across subclasses.
- Ensures consistent initialization.
- Simplifies subclass constructors.

---

#### Push Down Field

Use this when a field is only used by some subclasses. Move it down from the superclass to only those subclasses that need it.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
#include <string>

class Employee {
public:
    virtual ~Employee() = default;

protected:
    std::string name_;
    double quota_;  // only Salesperson uses
};

class Engineer : public Employee {};

class Salesperson : public Employee {
public:
    bool metQuota(double sales) const {
        return sales >= quota_;
    }
};
```

</td>
<td>

```cpp
#include <string>

class Employee {
public:
    virtual ~Employee() = default;

protected:
    std::string name_;
};

class Engineer : public Employee {};

class Salesperson : public Employee {
public:
    bool metQuota(double sales) const {
        return sales >= quota_;
    }

private:
    double quota_;  // pushed down
};
```

</td>
</tr>
</table>

**Benefits**
- Keeps base class minimal and focused.
- Avoids polluting subclasses that don't need the data.
- Makes subclass-specific state explicit.

---

#### Push Down Method

Use this when a method is only relevant to some subclasses. Move it from the superclass to only those subclasses that need it.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
#include <string>

class Employee {
public:
    virtual ~Employee() = default;

    // Only Salesperson uses this
    virtual double commission(double sales) const {
        return sales * 0.1;
    }

protected:
    std::string name_;
};

class Engineer : public Employee {};

class Salesperson : public Employee {};
```

</td>
<td>

```cpp
#include <string>

class Employee {
public:
    virtual ~Employee() = default;

protected:
    std::string name_;
};

class Engineer : public Employee {};

class Salesperson : public Employee {
public:
    double commission(double sales) const {
        return sales * 0.1;  // pushed down
    }
};
```

</td>
</tr>
</table>

**Benefits**
- Keeps the base class lean and general.
- Clarifies which behavior belongs to which subclass.
- Prevents accidental use of irrelevant methods.

---

#### Extract Subclass

Use this when a class has features that only some of its instances use. Move those features into a subclass.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
#include <optional>

class JobItem {
public:
    JobItem(double unitPrice, int quantity,
            std::optional<double> employeeRate)
        : unitPrice_(unitPrice),
          quantity_(quantity),
          employeeRate_(employeeRate) {}

    double totalPrice() const {
        // labor items use the employee's rate
        const double price = employeeRate_
                                 ? *employeeRate_
                                 : unitPrice_;
        return price * quantity_;
    }

private:
    double unitPrice_;
    int quantity_;
    std::optional<double> employeeRate_;
};
```

</td>
<td>

```cpp
class JobItem {
public:
    JobItem(double unitPrice, int quantity)
        : unitPrice_(unitPrice),
          quantity_(quantity) {}
    virtual ~JobItem() = default;

    double totalPrice() const {
        return unitPrice() * quantity_;
    }

protected:
    virtual double unitPrice() const {
        return unitPrice_;
    }

private:
    double unitPrice_;
    int quantity_;
};

class LaborItem final : public JobItem {
public:
    LaborItem(int quantity, double employeeRate)
        : JobItem(0.0, quantity),
          employeeRate_(employeeRate) {}

protected:
    double unitPrice() const override {
        return employeeRate_;
    }

private:
    double employeeRate_;
};
```

</td>
</tr>
</table>

**Benefits**
- Each class holds only the data it uses.
- No optional fields or conditionals.
- A new kind of item is just another subclass.

---

#### Extract Superclass

Use this when two or more classes share fields and behavior. Extract the shared parts into a superclass to reduce duplication and communicate a shared abstraction.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
#include <string>

class PhysicalProduct {
public:
    PhysicalProduct(std::string sku, double price)
        : sku_(std::move(sku)), price_(price) {}

    double totalPrice(int qty) const {
        return price_ * qty;
    }

private:
    std::string sku_;
    double price_;
};

class DigitalProduct {
public:
    DigitalProduct(std::string sku, double price)
        : sku_(std::move(sku)), price_(price) {}

    double totalPrice(int qty) const {
        return price_ * qty;
    }

private:
    std::string sku_;
    double price_;
};
```

</td>
<td>

```cpp
#include <string>

class Product {
public:
    Product(std::string sku, double price)
        : sku_(std::move(sku)), price_(price) {}

    virtual ~Product() = default;

    double totalPrice(int qty) const {
        return price_ * qty;
    }

protected:
    std::string sku_;
    double price_;
};

class PhysicalProduct : public Product {
public:
    using Product::Product;
};

class DigitalProduct : public Product {
public:
    using Product::Product;
};
```

</td>
</tr>
</table>

**Benefits**
- Removes copy-paste duplication.
- Creates a single place to evolve shared logic.
- Makes shared concepts explicit.

---

#### Extract Interface

Use this when several clients use the same part of a class's interface, or two classes share part of their interface. Move that part into an abstract interface. A C++20 concept can express the same contract at compile time.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
#include <string>

class Employee {
public:
    double rate() const { return rate_; }
    bool hasSpecialSkill() const { return true; }
    std::string name() const { return name_; }
    // ... many more members
private:
    double rate_ = 50.0;
    std::string name_;
};

// Only needs rate() and hasSpecialSkill()
double charge(const Employee& e, int days) {
    const double base = e.rate() * days;
    return e.hasSpecialSkill() ? base * 1.05
                               : base;
}
```

</td>
<td>

```cpp
#include <concepts>
#include <string>

class Billable {
public:
    virtual ~Billable() = default;
    virtual double rate() const = 0;
    virtual bool hasSpecialSkill() const = 0;
};

class Employee final : public Billable {
public:
    double rate() const override { return rate_; }
    bool hasSpecialSkill() const override {
        return true;
    }
    std::string name() const { return name_; }
private:
    double rate_ = 50.0;
    std::string name_;
};

double charge(const Billable& b, int days) {
    const double base = b.rate() * days;
    return b.hasSpecialSkill() ? base * 1.05
                               : base;
}

// Compile-time alternative: a C++20 concept
template <typename T>
concept BillableLike = requires(const T& t) {
    { t.rate() } -> std::convertible_to<double>;
    { t.hasSpecialSkill() } -> std::same_as<bool>;
};
```

</td>
</tr>
</table>

**Benefits**
- Clients depend only on what they use.
- Other classes (say, a contractor) can be billed too.
- The role gets an explicit name.

---

#### Collapse Hierarchy

Use this when a subclass is practically the same as its superclass. Merge them into one class.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
#include <string>
#include <utility>

class Employee {
public:
    explicit Employee(std::string name)
        : name_(std::move(name)) {}
    virtual ~Employee() = default;
    const std::string& name() const {
        return name_;
    }
private:
    std::string name_;
};

// Adds nothing anymore
class Salesperson : public Employee {
public:
    using Employee::Employee;
};
```

</td>
<td>

```cpp
#include <string>
#include <utility>

class Employee {
public:
    explicit Employee(std::string name)
        : name_(std::move(name)) {}
    const std::string& name() const {
        return name_;
    }
private:
    std::string name_;
};
```

</td>
</tr>
</table>

**Benefits**
- Fewer classes to understand.
- No virtual destructor or dispatch needed.
- Removes a misleading distinction.

---

#### Form Template Method

Use this when subclasses implement algorithms with the same steps in the same order but different details. Move the shared outline into the base class and let subclasses override only the steps that differ.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
#include <format>
#include <string>

class TextStatement {
public:
    std::string render(const std::string& name,
                       double amount) const {
        return std::format("Statement for {}\n",
                           name) +
               std::format("Amount: {:.2f}\n",
                           amount);
    }
};

class HtmlStatement {
public:
    std::string render(const std::string& name,
                       double amount) const {
        return std::format("<h1>{}</h1>", name) +
               std::format("<p>{:.2f}</p>", amount);
    }
};
```

</td>
<td>

```cpp
#include <format>
#include <string>

class Statement {
public:
    virtual ~Statement() = default;

    // Template method: the fixed outline
    std::string render(const std::string& name,
                       double amount) const {
        return header(name) + line(amount);
    }

protected:
    virtual std::string header(
        const std::string& name) const = 0;
    virtual std::string line(
        double amount) const = 0;
};

class TextStatement final : public Statement {
protected:
    std::string header(
        const std::string& name) const override {
        return std::format("Statement for {}\n",
                           name);
    }
    std::string line(
        double amount) const override {
        return std::format("Amount: {:.2f}\n",
                           amount);
    }
};

class HtmlStatement final : public Statement {
protected:
    std::string header(
        const std::string& name) const override {
        return std::format("<h1>{}</h1>", name);
    }
    std::string line(
        double amount) const override {
        return std::format("<p>{:.2f}</p>", amount);
    }
};
```

</td>
</tr>
</table>

**Benefits**
- The algorithm's structure is defined once.
- Subclasses can't get the order of steps wrong.
- A new format only fills in the steps.

---

#### Replace Super Class With Delegation

*refactoring.guru calls this **Replace Inheritance with Delegation**.*

Use this when inheritance exists mainly to reuse behavior, but the subclass doesn't conceptually *is-a* superclass. Prefer composition: delegate behavior to a collaborator.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
#include <iostream>
#include <string>

class Logger {
public:
    void info(const std::string& msg) {
        std::cout << "[info] " << msg << "\n";
    }
};

// Inheritance just to reuse Logger
class OrderService : public Logger {
public:
    void placeOrder(const std::string& id) {
        info("placing order " + id);
        // ...
    }
};
```

</td>
<td>

```cpp
#include <iostream>
#include <string>

class Logger {
public:
    void info(const std::string& msg) {
        std::cout << "[info] " << msg << "\n";
    }
};

class OrderService {
public:
    explicit OrderService(Logger& logger)
        : logger_(logger) {}

    void placeOrder(const std::string& id) {
        logger_.info("placing order " + id);
        // ...
    }

private:
    Logger& logger_;
};
```

</td>
</tr>
</table>

**Benefits**
- Avoids incorrect "is-a" relationships.
- Makes dependencies explicit and swappable in tests.
- Reduces coupling to a base class's internals.

---

#### Replace Delegation with Inheritance

Use this when a class forwards calls for nearly all of another class's public interface, and it truly *is a* specialized version of it. Inherit instead of writing forwarding methods. It is the inverse of [Replace Super Class With Delegation](#replace-super-class-with-delegation); prefer delegation unless the forwarding really covers everything.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
#include <string>
#include <utility>

class Person {
public:
    explicit Person(std::string name)
        : name_(std::move(name)) {}
    const std::string& name() const {
        return name_;
    }
    void setName(std::string n) {
        name_ = std::move(n);
    }
private:
    std::string name_;
};

class Employee {
public:
    explicit Employee(std::string name)
        : person_(std::move(name)) {}
    // Forwarding boilerplate
    const std::string& name() const {
        return person_.name();
    }
    void setName(std::string n) {
        person_.setName(std::move(n));
    }
private:
    Person person_;
};
```

</td>
<td>

```cpp
#include <string>
#include <utility>

class Person {
public:
    explicit Person(std::string name)
        : name_(std::move(name)) {}
    const std::string& name() const {
        return name_;
    }
    void setName(std::string n) {
        name_ = std::move(n);
    }
private:
    std::string name_;
};

class Employee : public Person {
public:
    using Person::Person;
};
```

</td>
</tr>
</table>

**Benefits**
- Removes forwarding boilerplate.
- `Employee` gets new `Person` features automatically.
- Models a genuine "is a" relationship.

---

### Other Refactorings

Techniques that are not in the refactoring.guru catalog.

---

#### Remove Dead Code

Use this when code is no longer reachable or used. Dead code adds noise, confuses readers, and increases maintenance burden. Delete it.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
#include <string>

class Report {
public:
    std::string generate() {
        return "Current Report";
    }

    // No longer called anywhere
    std::string generateLegacy() {
        return "Old Report Format";
    }

    // Commented out - dead code
    // std::string generateV2() {
    //     return "V2 Report";
    // }

private:
    int unusedField_ = 0;  // never read
};
```

</td>
<td>

```cpp
#include <string>

class Report {
public:
    std::string generate() {
        return "Current Report";
    }
};
```

</td>
</tr>
</table>

**Benefits**
- Reduces cognitive load (less code to read and understand).
- Eliminates confusion about what's actually used.
- Smaller codebase is easier to maintain and compile.

---

#### Remove Field

Use this when a field is no longer used or its purpose has been absorbed elsewhere. Delete it to reduce noise and memory footprint.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
#include <string>
#include <chrono>

class Order {
public:
    Order(std::string id, double total)
        : id_(std::move(id)),
          total_(total),
          createdAt_(std::chrono::system_clock::now()),
          // Legacy field, no longer used
          legacyCode_(""),
          // Redundant, computed from items
          itemCount_(0) {}

    const std::string& getId() const {
        return id_;
    }

    double getTotal() const {
        return total_;
    }

private:
    std::string id_;
    double total_;
    std::chrono::system_clock::time_point createdAt_;
    std::string legacyCode_;  // unused
    int itemCount_;           // unused
};
```

</td>
<td>

```cpp
#include <string>
#include <chrono>

class Order {
public:
    Order(std::string id, double total)
        : id_(std::move(id)),
          total_(total),
          createdAt_(std::chrono::system_clock::now()) {}

    const std::string& getId() const {
        return id_;
    }

    double getTotal() const {
        return total_;
    }

private:
    std::string id_;
    double total_;
    std::chrono::system_clock::time_point createdAt_;
};
```

</td>
</tr>
</table>

**Benefits**
- Reduces memory usage and object size.
- Eliminates confusion about unused data.
- Simplifies constructors and maintenance.

---

#### Replace Command With Function

Use this when you have a command object (or functor) that only wraps a single action with no state. Replace it with a simple function to reduce boilerplate.

<table>
<tr>
<th>Before</th>
<th>After</th>
</tr>
<tr>
<td>

```cpp
#include <string>
#include <memory>

// Command interface
class Command {
public:
    virtual ~Command() = default;
    virtual void execute() = 0;
};

// Concrete command - just prints
class PrintCommand : public Command {
public:
    explicit PrintCommand(std::string msg)
        : message_(std::move(msg)) {}

    void execute() override {
        std::cout << message_ << "\n";
    }

private:
    std::string message_;
};

// Usage
void runCommand(Command& cmd) {
    cmd.execute();
}

PrintCommand cmd("Hello");
runCommand(cmd);
```

</td>
<td>

```cpp
#include <string>
#include <functional>
#include <iostream>

// Simple function instead of command class
void printMessage(const std::string& msg) {
    std::cout << msg << "\n";
}

// Usage with function pointer or lambda
void runAction(std::function<void()> action) {
    action();
}

runAction([] { printMessage("Hello"); });

// Or just call directly
printMessage("Hello");
```

</td>
</tr>
</table>

**Benefits**
- Eliminates boilerplate class hierarchies for simple actions.
- Functions are simpler to write, read, and test.
- Lambdas and `std::function` provide flexibility without inheritance.

---

### Overall benefits of refactoring

Across these techniques, refactored code typically becomes:

- Easier to read (less branching, clearer intent, better structure)
- Safer to change (new cases are additive, fewer ripple edits)
- Easier to test (smaller units, explicit collaborators)
- More reusable (logic moved into focused types/functions)

---

## Tidy First?

> **A Personal Exercise in Empirical Software Design**  
> by Kent Beck

---

### 📚 About This Book

*Tidy First?* is a practical guide by Kent Beck, creator of Extreme Programming and pioneer of software patterns, that suggests when and where you might apply "tidyings" to improve your code while keeping the overall structure of the system in mind.

#### What You'll Learn

- **Software Design Theory**: Coupling, cohesion, discounted cash flows, and optionality
- **Behavior vs Structure**: The fundamental difference between changes to a system's behavior and changes to its structure
- **Incremental Improvement**: How to make large changes in small, safe steps
- **Human-Centered Design**: Approaching software design as an exercise in human relationships

---

### 📖 Table of Contents

#### Part I: Tidyings

1. [Guard Clauses](#1-guard-clauses)
2. [Dead Code](#2-dead-code)
3. [Normalize Symmetries](#3-normalize-symmetries)
4. [New Interface, Old Implementation](#4-new-interface-old-implementation)
5. [Reading Order](#5-reading-order)
6. [Cohesion Order](#6-cohesion-order)
7. [Move Declaration and Initialization Together](#7-move-declaration-and-initialization-together)
8. [Explaining Variables](#8-explaining-variables)
9. [Explaining Constants](#9-explaining-constants)
10. [Explicit Parameters](#10-explicit-parameters)
11. [Chunk Statements](#11-chunk-statements)
12. [Extract Helper](#12-extract-helper)
13. [One Pile](#13-one-pile)
14. [Explaining Comments](#14-explaining-comments)
15. [Delete Redundant Comments](#15-delete-redundant-comments)

#### Part II: Managing

16. [Separate Tidying](#16-separate-tidying)
17. [Chaining](#17-chaining)
18. [Batch Sizes](#18-batch-sizes)
19. [Rhythm](#19-rhythm)
20. [Getting Untangled](#20-getting-untangled)
21. [First, After, Later, Never](#21-first-after-later-never)

#### Part III: Theory

22. [Beneficially Relating Elements](#22-beneficially-relating-elements)
23. [Structure and Behavior](#23-structure-and-behavior)
24. [Economics: Time Value and Optionality](#24-economics-time-value-and-optionality)
25. [A Dollar Today > A Dollar Tomorrow](#25-a-dollar-today--a-dollar-tomorrow)
26. [Options](#26-options)
27. [Options Versus Cash Flows](#27-options-versus-cash-flows)
28. [Reversible Structure Changes](#28-reversible-structure-changes)
29. [Coupling](#29-coupling)
30. [Constantine's Equivalence](#30-constantines-equivalence)
31. [Coupling Versus Decoupling](#31-coupling-versus-decoupling)
32. [Cohesion](#32-cohesion)
33. [Conclusion](#33-conclusion)

---

### Preface

#### What Is Tidy First?

> *"I have to change this code, but it's messy. What should I do first?"*
>
> *"Maybe I should tidy the code before I make the change. Maybe. Some. Or maybe not?"*

**Tidy First?** describes:

- When to tidy messy code before changing what it computes
- How to tidy messy code safely and efficiently
- How to stop tidying messy code
- Why tidying works

Software design is an exercise in human relationships. In *Tidy First?* we start with the proverbial person in the mirror—with the programmer's relationship with themself.

#### Target Audience

This book is meant for:
- Programmers
- Lead developers
- Hands-on software architects
- Technical managers

It is not tied to any programming language, and all developers will be able to read and apply the concepts.

#### Book Structure

| Part | Focus |
|------|-------|
| **Part I: Tidyings** | A catalog of small design "moves" you can make when faced with messy code |
| **Part II: Managing** | How to fit tidying into a personal development workflow |
| **Part III: Theory** | The deeper theory behind software design: coupling, cohesion, and economics |

---

### Part I: Tidyings

> *Tidyings are a subset of refactorings. Tidyings are the cute, fuzzy little refactorings that nobody could possibly hate on.*

---

#### 1. Guard Clauses

**Problem:** Nested conditions are hard to read.

```pseudocode
// Before
if (condition)
    if (not other condition)
        ...some code...
```

**Solution:** Use guard clauses to exit early.

```pseudocode
// After
if (not condition) return
if (other condition) return
...some code...
```

**Benefits:**
- Easier to read
- Preconditions are explicit
- Code is easier to analyze

> ⚠️ **Don't overdo it.** A routine with seven or eight guard clauses is not easier to read.

---

#### 2. Dead Code

**Rule:** Delete it. That's all. If the code doesn't get executed, just delete it.

**Common objections:**
- *"Someone took time to write it"* → Sunk cost fallacy
- *"What if we need it later?"* → That's what version control is for

**Best practice:** Delete only a little code in each tidying diff for easy reversion.

---

#### 3. Normalize Symmetries

**Problem:** The same problem solved differently in different places makes code confusing.

**Example:** Lazy initialization can be written many ways:

```pseudocode
// Variant 1
foo()
    return foo if foo not nil
    foo := ...
    return foo

// Variant 2
foo()
    if foo is nil
        foo := ...
    return foo

// Variant 3 (tricky)
foo()
    return foo not nil ? foo : foo := ...
```

**Solution:** Pick one way and convert all variants to match.

> *As a reader, you expect that difference means difference.*

---

#### 4. New Interface, Old Implementation

**Scenario:** You need to call a routine, but the interface is difficult/complicated/confusing.

**Solution:**
1. Implement the interface you wish you could call
2. Call that new interface
3. Implement it by simply calling the old one
4. Migrate callers later

This approach also applies to:
- **Coding backward** — Start with the last line of a routine
- **Coding test-first** — Start with the test that needs to pass
- **Designing helpers** — "If only I had a routine that did X..."

---

#### 5. Reading Order

**Scenario:** You read a file, get to the end, and find the detail that would have helped you understand everything earlier.

**Solution:** Reorder the code in the order a reader would prefer to encounter it.

**Guidelines:**
- You're a reader. You just read it. So you know.
- Resist tidying other things at the same time
- Be careful with order-sensitive languages

---

#### 6. Cohesion Order

**Problem:** Making a behavior change requires changing several widely dispersed spots in the code.

**Solution:** Reorder the code so elements you need to change are adjacent.

**Works for:**
- Routines in a file
- Files in directories
- Code across repositories

**When to decouple vs. improve cohesion:**

```
cost(decoupling) + cost(change) < cost(coupling) + cost(change)
```

---

#### 7. Move Declaration and Initialization Together

**Problem:** Variables and their initialization drift apart, making code harder to read.

```pseudocode
// Before
fn()
    int a
    ...some code that doesn't use a
    a = ...
    int b
    ...some more code
    b = ...a...
```

```pseudocode
// After
fn()
    int a = ...
    ...some code that doesn't use a
    ...some more code
    int b = ...a...
```

> ⚠️ **Remember:** Maintain data dependency order. If `a` is used to initialize `b`, initialize `a` first.

---

#### 8. Explaining Variables

**Problem:** Large, complex expressions are hard to understand.

```pseudocode
// Before
return new Point(
    ...big long expression...,
    ...another big long expression...
)
```

```pseudocode
// After
x := ...big long expression...
y := ...another big long expression...
return new Point(x, y)
```

Or better yet, use meaningful names like `width`, `height`, `top`, `left`.

---

#### 9. Explaining Constants

**Problem:** Magic numbers and repeated literal constants.

```pseudocode
// Before
if response.code = 404
    ...blah blah blah...
```

```pseudocode
// After
PAGE_NOT_FOUND := 404
if response.code = PAGE_NOT_FOUND
    ...blah blah blah...
```

> ⚠️ **Be careful:** The same literal can mean different things in different contexts.

---

#### 10. Explicit Parameters

**Problem:** Data used by a routine wasn't passed explicitly.

```pseudocode
// Before
params = { a: 1, b: 2 }
foo(params)

function foo(params)
    ...params.a...
    ...params.b...
```

```pseudocode
// After
function foo(params)
    foo_body(params.a, params.b)

function foo_body(a, b)
    ...a...
    ...b...
```

This also applies to environment variables buried deep in code.

---

#### 11. Chunk Statements

**The simplest tidying:** Put a blank line between logical sections of code.

> *"Oh, this part does this and then that part does that."*

**Follow-up tidyings:**
- Explaining Variables
- Extract Helper
- Explaining Comments

---

#### 12. Extract Helper

**Scenario:** A block of code inside a routine has an obvious purpose and limited interaction with the rest.

**Solution:** Extract it as a helper routine. Name it after the **purpose**, not how it works.

**Special cases:**

1. **Change isolation:**
```pseudocode
// Before
routine()
    ...stuff that stays the same...
    ...stuff that needs to change...
    ...stuff that stays the same...

// After
helper()
    ...stuff that needs to change...

routine()
    ...stuff that stays the same...
    helper()
    ...stuff that stays the same...
```

2. **Temporal coupling:**
```pseudocode
// If you frequently see:
foo.a()
foo.b()

// Create:
ab()
    a()
    b()
```

---

#### 13. One Pile

**When to use:** Code has been split into many tiny pieces in a way that hinders understanding.

**Solution:** Inline as much code as you need until it's all in one big pile. Then tidy from there.

**Symptoms to look for:**
- Long, repeated argument lists
- Repeated code, especially conditionals
- Poor naming of helper routines
- Shared mutable data structures

> *The biggest cost of code is the cost of reading and understanding it, not the cost of writing it.*

---

#### 14. Explaining Comments

**When you understand something non-obvious:** Write it down.

**Guidelines:**
- Write only what wasn't obvious from the code
- Think from the perspective of someone else
- Address likely questions preemptively

**Good times to comment:**
- Upon finding a defect: `// Be sure to change ../foo if you add another case`
- File headers explaining why readers might find the file useful

---

#### 15. Delete Redundant Comments

**Rule:** If a comment says exactly what the code says, remove it.

```pseudocode
// Bad - redundant
getX()
    # return X
    return X
```

**Note:** Only delete *completely* redundant comments. Tidy with an eye toward making comments redundant through better code structure.

---

### Part II: Managing

> *Tidying is geek self-care.*

---

#### 16. Separate Tidying

**Key principle:** Put tidyings in their own PRs, separate from behavior changes.

**Evolution of awareness:**

1. **Undifferentiated mass** — Change is change
2. **Awareness** — Some changes are behavior (B), some are structure (S)
3. **Sequencing** — Chunking → Extract helper → Easier behavior change
4. **Separate PRs** — Each switch between tidying and behavior = new PR

**Trade-offs:**
- Big PRs show whole picture but overwhelm reviewers
- Tiny PRs invite feedback but risk going off track
- Fast reviews encourage smaller PRs (reinforcing loop)

---

#### 17. Chaining

**Reality:** Tidyings are like potato chips. One leads to another.

| Tidying | Leads To |
|---------|----------|
| Guard clause | Explaining helper, explaining variable |
| Dead code | Reading order, cohesion order |
| Normalize symmetries | Reading order, grouping |
| New interface | Converting callers (fanout) |
| Reading order | Normalize symmetries |
| Cohesion order | Extraction candidates |
| Explaining variables | Explaining helper, delete comments |
| Chunk statements | Explaining comments, extract helper |
| Extract helper | Guard clause, constants, delete comments |
| One pile | Chunk statements, comments, helpers |

> ⚠️ **Warning:** Be wary of changing too much, too fast. Practice tidyings like notes of a scale.

---

#### 18. Batch Sizes

**Question:** How many tidyings per batch before integrating?

**Costs that rise with larger batches:**
- **Collisions** — Higher chance of merge conflicts
- **Interactions** — Higher chance of accidental behavior changes
- **Speculation** — Tidying "just because"

**Costs that rise with smaller batches:**
- **Review overhead** — Fixed cost per change

**Solution:** Reduce the cost of review to enable smaller batches.

---

#### 19. Rhythm

**Timescale:** Minutes to an hour of tidying before making a behavior change.

> More than an hour likely means you've lost track of the minimum set of structure changes needed.

**The "pave the path" effect:**
- Behavior changes cluster in 20% of files (Pareto)
- Tidyings cluster in those same spots
- Eventually, most changes happen in already-tidied code

---

#### 20. Getting Untangled

**Scenario:** You're changing behavior, see a helpful tidying, and suddenly everything is tangled.

**Options:**

1. **Never tidy** — Miss out on cleaner code
2. **Tidy after** — Complete behavior change first
3. **Tidy first** — Do the tidying, then behavior change
4. **Separate paths** — Work in parallel with version control

---

#### 21. First, After, Later, Never

**When to tidy:**

| Timing | Description |
|--------|-------------|
| **First** | Tidy immediately before your behavior change |
| **After** | Complete behavior change, then tidy |
| **Later** | Add to backlog for future consideration |
| **Never** | The cost exceeds the benefit |

---

### Part III: Theory

> *Software design is an exercise in human relationships.*

---

#### 22. Beneficially Relating Elements

**Software design defined:** Creating beneficial relationships between elements of a system.

**Elements can be:**
- Functions, classes, modules
- Files, directories, repositories
- Services, systems

**Key insight:** The relationships matter as much as the elements themselves.

---

#### 23. Structure and Behavior

**Two types of changes:**

| Type | Observable By | Example |
|------|---------------|---------|
| **Behavior** | Running the program | New feature, bug fix |
| **Structure** | Reading the code | Rename, extract method |

> *Tidyings are structure changes that enable behavior changes.*

---

#### 24. Economics: Time Value and Optionality

**Two economic principles:**

1. **Time value of money** — A dollar today is worth more than a dollar tomorrow
2. **Optionality** — Having choices has value

**Applied to software:**
- Shipping features sooner creates value sooner
- Good structure creates options for future changes

---

#### 25. A Dollar Today > A Dollar Tomorrow

**Implication:** Deliver value sooner when possible.

**But:** Structure enables faster future delivery, so investment in structure can pay off.

---

#### 26. Options

**Options in software:**
- Ability to add features easily
- Ability to change direction
- Ability to scale

**Good structure increases options.**

---

#### 27. Options Versus Cash Flows

**Trade-off:**
- Cash flows (features) = immediate value
- Options (structure) = future flexibility

**Balance:** Tidy enough to keep options open, ship enough to create value.

---

#### 28. Reversible Structure Changes

**Key property of tidyings:** They are reversible.

**Benefits:**
- Low risk experimentation
- Easy to back out if wrong
- Encourages trying things

---

#### 29. Coupling

**Definition:** The degree to which changes in one element require changes in another.

**Types:**
- **Content coupling** — One element directly modifies another's internals
- **Common coupling** — Elements share global data
- **Control coupling** — One element controls flow of another
- **Stamp coupling** — Elements share composite data structures
- **Data coupling** — Elements share only necessary data (best)

---

#### 30. Constantine's Equivalence

**Larry Constantine's insight:**

> The cost of software is approximately equal to the cost of changing software, which is dominated by the cost of the big changes, which is dominated by coupling.

**Therefore:** Reducing coupling reduces cost.

---

#### 31. Coupling Versus Decoupling

**Decoupling costs:**
- Time and effort to restructure
- Risk of introducing bugs
- Cognitive load on team

**When to decouple:**

```
cost(decoupling) + cost(change) < cost(coupling) + cost(change)
```

---

#### 32. Cohesion

**Definition:** The degree to which elements within a module belong together.

**High cohesion means:**
- Elements work toward a single purpose
- Changes are localized
- Code is easier to understand

**Relationship to coupling:** High cohesion tends to produce low coupling.

---

#### 33. Conclusion

**Key takeaways:**

1. **Tidy in small steps** — Minutes, not hours
2. **Separate structure from behavior** — Different PRs
3. **Think economically** — Balance options vs. cash flows
4. **Design for humans** — Code is read more than written
5. **Keep tidying reversible** — Low risk, high reward

---

### 💡 Quick Reference

#### Tidying Decision Tree

```
Need to make a behavior change?
│
├─ Is the code hard to understand?
│   ├─ Yes → Consider tidying first
│   └─ No → Make the change
│
├─ Will tidying take > 1 hour?
│   ├─ Yes → Break into smaller tidyings or skip
│   └─ No → Tidy first
│
└─ Is the tidying reversible?
    ├─ Yes → Low risk, go ahead
    └─ No → Proceed carefully
```

#### The 15 Tidyings

| # | Name | One-liner |
|---|------|-----------|
| 1 | Guard Clauses | Exit early, flatten nesting |
| 2 | Dead Code | Delete unused code |
| 3 | Normalize Symmetries | Same problem → same solution |
| 4 | New Interface, Old Implementation | Wrap with better interface |
| 5 | Reading Order | Arrange for readers |
| 6 | Cohesion Order | Group related code |
| 7 | Move Declaration + Init | Keep them together |
| 8 | Explaining Variables | Name complex expressions |
| 9 | Explaining Constants | Replace magic numbers |
| 10 | Explicit Parameters | Make inputs visible |
| 11 | Chunk Statements | Blank lines between sections |
| 12 | Extract Helper | Pull out coherent blocks |
| 13 | One Pile | Inline over-split code |
| 14 | Explaining Comments | Document the non-obvious |
| 15 | Delete Redundant Comments | Remove noise |

---
