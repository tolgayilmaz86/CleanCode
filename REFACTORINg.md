# Refactoring Examples

This document provides small, concrete examples of common refactoring methods. Each section includes:

- A short introduction to the technique
- Two code samples side-by-side (**Before** / **After**)
- A quick summary of the benefits of the refactored version

All examples use C++ syntax.

---

## Table of Contents

1. [Introduce Special Case](#introduce-special-case)
2. [Replace Conditional With Polymorphism](#replace-conditional-with-polymorphism)
3. [Replace Constructor With Factory Function](#replace-constructor-with-factory-function)
4. [Replace Super Class With Delegation](#replace-super-class-with-delegation)
5. [Pull Up Constructor Body](#pull-up-constructor-body)
6. [Extract Superclass](#extract-superclass)
7. [Introduce Parameter Object](#introduce-parameter-object)
8. [Extract Class](#extract-class)
9. [Pull Up Field](#pull-up-field)
10. [Pull Up Method](#pull-up-method)
11. [Push Down Field](#push-down-field)
12. [Push Down Method](#push-down-method)
13. [Remove Dead Code](#remove-dead-code)
14. [Remove Flag Argument](#remove-flag-argument)
15. [Remove Middle Man](#remove-middle-man)
16. [Remove Setting Method](#remove-setting-method)
17. [Remove Subclass](#remove-subclass)
18. [Remove Field](#remove-field)
19. [Replace Command With Function](#replace-command-with-function)

---

## Introduce Special Case

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

## Replace Conditional With Polymorphism

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

## Replace Constructor With Factory Function

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

## Replace Super Class With Delegation

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

## Pull Up Constructor Body

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

## Extract Superclass

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

## Introduce Parameter Object

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

## Extract Class

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

## Pull Up Field

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

## Pull Up Method

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

## Push Down Field

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

## Push Down Method

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

## Remove Dead Code

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

## Remove Flag Argument

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

## Remove Middle Man

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

## Remove Setting Method

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

## Remove Subclass

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

## Remove Field

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

## Replace Command With Function

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

## Overall benefits of refactoring

Across these techniques, refactored code typically becomes:

- Easier to read (less branching, clearer intent, better structure)
- Safer to change (new cases are additive, fewer ripple edits)
- Easier to test (smaller units, explicit collaborators)
- More reusable (logic moved into focused types/functions)

