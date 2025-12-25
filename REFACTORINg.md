

# Refactoring Examples

This document provides small, concrete examples of common refactoring methods. 

All examples are intentionally small and use C++ syntax to stay readable.

---

## Introduce Special Case

Use this when you have repeated `nullptr` checks (or “missing value” conditionals) throughout the code. Instead, introduce a “special case” object (often `UnknownCustomer`, `NullUser`, etc.) that behaves safely.

<table>
	<thead>
		<tr>
			<th>Before</th>
			<th>After</th>
		</tr>
	</thead>
	<tbody>
		<tr>
			<td>
				<pre><code class="language-cpp">#include &lt;string&gt;

struct Customer {
	std::string name;
	double discountRate; // 0.10 = 10%
};

std::string customerName(const Customer* customer) {
	if (customer == nullptr) return "(guest)";
	return customer-&gt;name;
}

double applyDiscount(double total, const Customer* customer) {
	if (customer == nullptr) return total;
	return total * (1.0 - customer-&gt;discountRate);
}</code></pre>
			</td>
			<td>
				<pre><code class="language-cpp">#include &lt;memory&gt;
#include &lt;string&gt;

struct CustomerLike {
	virtual ~CustomerLike() = default;
	virtual std::string name() const = 0;
	virtual double discountRate() const = 0;
};

struct GuestCustomer final : CustomerLike {
	std::string name() const override { return "(guest)"; }
	double discountRate() const override { return 0.0; }
};

struct RealCustomer final : CustomerLike {
	explicit RealCustomer(std::string n, double rate)
		: n_(std::move(n)), rate_(rate) {}

	std::string name() const override { return n_; }
	double discountRate() const override { return rate_; }

private:
	std::string n_;
	double rate_;
};

std::shared_ptr&lt;CustomerLike&gt; normalizeCustomer(std::shared_ptr&lt;CustomerLike&gt; c) {
	if (c) return c;
	return std::make_shared&lt;GuestCustomer&gt;();
}

double applyDiscount(double total, const CustomerLike& customer) {
	return total * (1.0 - customer.discountRate());
}</code></pre>
			</td>
		</tr>
	</tbody>
</table>

**Benefits**
- Reduces scattered `nullptr` checks and branching.
- Makes behavior explicit (a guest is a real domain concept).
- Improves testability by centralizing “missing” behavior.

---

## Replace Conditional With Polymorphism

Use this when a conditional (e.g., a `switch` on type) selects behavior. Replace it with polymorphism so adding new cases doesn’t grow a single conditional hotspot.

<table>
	<thead>
		<tr>
			<th>Before</th>
			<th>After</th>
		</tr>
	</thead>
	<tbody>
		<tr>
			<td>
				<pre><code class="language-cpp">#include &lt;string&gt;

enum class ShippingType { Standard, Express, Overnight };

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
	return 0.0; // unreachable in practice
}</code></pre>
			</td>
			<td>
				<pre><code class="language-cpp">struct ShippingPolicy {
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

double shippingCost(const ShippingPolicy& policy, double weightKg) {
	return policy.cost(weightKg);
}</code></pre>
			</td>
		</tr>
	</tbody>
</table>

**Benefits**
- Eliminates a growing `switch`/`if` chain.
- Makes new behavior additive (add a new policy type).
- Keeps each rule small and focused.

---

## Replace Constructor With Factory Function

Use this when object creation needs naming, defaults, validation, or conditional logic. A factory can enforce invariants and keep call sites clean.

<table>
	<thead>
		<tr>
			<th>Before</th>
			<th>After</th>
		</tr>
	</thead>
	<tbody>
		<tr>
			<td>
				<pre><code class="language-cpp">#include &lt;string&gt;

class Connection {
public:
	Connection(std::string host, int port, bool useTls, int timeoutMs)
		: host_(std::move(host)), port_(port), useTls_(useTls), timeoutMs_(timeoutMs) {}

private:
	std::string host_;
	int port_;
	bool useTls_;
	int timeoutMs_;
};

// Call sites must remember defaults
Connection conn("db.local", 5432, true, 10000);</code></pre>
			</td>
			<td>
				<pre><code class="language-cpp">#include &lt;string&gt;

struct ConnectionOptions {
	std::string host;
	int port = 5432;
	bool useTls = true;
	int timeoutMs = 10000;
};

class Connection {
public:
	static Connection create(ConnectionOptions opts) {
		// Central place for defaults/validation
		if (opts.host.empty()) {
			opts.host = "localhost";
		}
		return Connection(std::move(opts.host), opts.port, opts.useTls, opts.timeoutMs);
	}

private:
	Connection(std::string host, int port, bool useTls, int timeoutMs)
		: host_(std::move(host)), port_(port), useTls_(useTls), timeoutMs_(timeoutMs) {}

	std::string host_;
	int port_;
	bool useTls_;
	int timeoutMs_;
};

auto conn = Connection::create({ .host = "db.local" });</code></pre>
			</td>
		</tr>
	</tbody>
</table>

**Benefits**
- Names the creation intent and centralizes defaults/validation.
- Keeps constructors private if you want to enforce invariants.
- Lets you change construction logic without changing every call site.

---

## Replace Super Class With Delegation

Use this when inheritance exists mainly to reuse behavior, but the subclass doesn’t conceptually *is-a* superclass. Prefer composition: delegate behavior to a collaborator.

<table>
	<thead>
		<tr>
			<th>Before</th>
			<th>After</th>
		</tr>
	</thead>
	<tbody>
		<tr>
			<td>
				<pre><code class="language-cpp">#include &lt;iostream&gt;
#include &lt;string&gt;

class Logger {
public:
	void info(const std::string& msg) {
		std::cout &lt;&lt; "[info] " &lt;&lt; msg &lt;&lt; "\n";
	}
};

// Inheritance just to reuse Logger
class OrderService : public Logger {
public:
	void placeOrder(const std::string& orderId) {
		info("placing order " + orderId);
		// ...
	}
};</code></pre>
			</td>
			<td>
				<pre><code class="language-cpp">#include &lt;iostream&gt;
#include &lt;string&gt;

class Logger {
public:
	void info(const std::string& msg) {
		std::cout &lt;&lt; "[info] " &lt;&lt; msg &lt;&lt; "\n";
	}
};

class OrderService {
public:
	explicit OrderService(Logger& logger) : logger_(logger) {}

	void placeOrder(const std::string& orderId) {
		logger_.info("placing order " + orderId);
		// ...
	}

private:
	Logger& logger_;
};</code></pre>
			</td>
		</tr>
	</tbody>
</table>

**Benefits**
- Avoids incorrect “is-a” relationships.
- Makes dependencies explicit and swappable in tests.
- Reduces coupling to a base class’s internals.

---

## Pull Up Constructor Body

Use this when multiple subclasses repeat the same initialization logic that belongs in the base class. Pull the shared code into the superclass constructor.

<table>
	<thead>
		<tr>
			<th>Before</th>
			<th>After</th>
		</tr>
	</thead>
	<tbody>
		<tr>
			<td>
				<pre><code class="language-cpp">#include &lt;chrono&gt;
#include &lt;string&gt;

class Employee {
protected:
	std::string id;
	std::chrono::system_clock::time_point createdAt;

	Employee(std::string id_, std::chrono::system_clock::time_point createdAt_)
		: id(std::move(id_)), createdAt(createdAt_) {}
};

class Engineer : public Employee {
public:
	explicit Engineer(std::string id_)
		: Employee(std::move(id_), std::chrono::system_clock::now()) {}
};

class Manager : public Employee {
public:
	explicit Manager(std::string id_)
		: Employee(std::move(id_), std::chrono::system_clock::now()) {}
};</code></pre>
			</td>
			<td>
				<pre><code class="language-cpp">#include &lt;chrono&gt;
#include &lt;string&gt;

class Employee {
protected:
	std::string id;
	std::chrono::system_clock::time_point createdAt;

	explicit Employee(std::string id_)
		: id(std::move(id_)), createdAt(std::chrono::system_clock::now()) {}
};

class Engineer : public Employee {
public:
	explicit Engineer(std::string id_) : Employee(std::move(id_)) {}
};

class Manager : public Employee {
public:
	explicit Manager(std::string id_) : Employee(std::move(id_)) {}
};</code></pre>
			</td>
		</tr>
	</tbody>
</table>

**Benefits**
- Removes duplication across subclasses.
- Ensures consistent initialization.
- Simplifies subclass constructors.

---

## Extract Superclass

Use this when two or more classes share fields and behavior. Extract the shared parts into a superclass to reduce duplication and communicate a shared abstraction.

<table>
	<thead>
		<tr>
			<th>Before</th>
			<th>After</th>
		</tr>
	</thead>
	<tbody>
		<tr>
			<td>
				<pre><code class="language-cpp">#include &lt;string&gt;

class PhysicalProduct {
public:
	PhysicalProduct(std::string sku, double price)
		: sku_(std::move(sku)), price_(price) {}

	double totalPrice(int quantity) const {
		return price_ * quantity;
	}

private:
	std::string sku_;
	double price_;
};

class DigitalProduct {
public:
	DigitalProduct(std::string sku, double price)
		: sku_(std::move(sku)), price_(price) {}

	double totalPrice(int quantity) const {
		return price_ * quantity;
	}

private:
	std::string sku_;
	double price_;
};</code></pre>
			</td>
			<td>
				<pre><code class="language-cpp">#include &lt;string&gt;

class Product {
public:
	Product(std::string sku, double price)
		: sku_(std::move(sku)), price_(price) {}

	virtual ~Product() = default;

	double totalPrice(int quantity) const {
		return price_ * quantity;
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
};</code></pre>
			</td>
		</tr>
	</tbody>
</table>

**Benefits**
- Removes copy-paste duplication.
- Creates a single place to evolve shared logic.
- Makes shared concepts explicit.

---

## Introduce Parameter Object

Use this when you have functions with many parameters that often travel together. A parameter object reduces argument lists and makes call sites clearer.

<table>
	<thead>
		<tr>
			<th>Before</th>
			<th>After</th>
		</tr>
	</thead>
	<tbody>
		<tr>
			<td>
				<pre><code class="language-cpp">#include &lt;chrono&gt;
#include &lt;string&gt;

void scheduleMeeting(
	const std::string& title,
	std::chrono::system_clock::time_point start,
	int durationMinutes,
	const std::string& organizerEmail,
	const std::string& roomName
) {
	// ...
}

// scheduleMeeting("1:1", start, 30, "alex@company.com", "Room 4");</code></pre>
			</td>
			<td>
				<pre><code class="language-cpp">#include &lt;chrono&gt;
#include &lt;string&gt;

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

// scheduleMeeting(MeetingRequest{ "1:1", start, 30, "alex@company.com", "Room 4" });</code></pre>
			</td>
		</tr>
	</tbody>
</table>

**Benefits**
- Improves readability at call sites (named fields vs positional args).
- Makes it easier to add new parameters without breaking callers.
- Enables validation/defaulting in one place.

---

## Extract Class

Use this when one class is doing too much and has multiple responsibilities. Extract a focused class to reduce size and clarify ownership of data and behavior.

<table>
	<thead>
		<tr>
			<th>Before</th>
			<th>After</th>
		</tr>
	</thead>
	<tbody>
		<tr>
			<td>
				<pre><code class="language-cpp">#include &lt;string&gt;

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
		return street_ + ", " + city_ + " " + postalCode_;
	}

private:
	std::string name_;
	std::string email_;
	std::string street_;
	std::string city_;
	std::string postalCode_;
};</code></pre>
			</td>
			<td>
				<pre><code class="language-cpp">#include &lt;string&gt;

class Address {
public:
	Address(std::string street, std::string city, std::string postalCode)
		: street_(std::move(street)),
		  city_(std::move(city)),
		  postalCode_(std::move(postalCode)) {}

	std::string formatted() const {
		return street_ + ", " + city_ + " " + postalCode_;
	}

private:
	std::string street_;
	std::string city_;
	std::string postalCode_;
};

class UserProfile {
public:
	UserProfile(std::string name, std::string email, Address address)
		: name_(std::move(name)),
		  email_(std::move(email)),
		  address_(std::move(address)) {}

private:
	std::string name_;
	std::string email_;
	Address address_;
};</code></pre>
			</td>
		</tr>
	</tbody>
</table>

**Benefits**
- Reduces class size and complexity.
- Separates responsibilities (address logic lives with address).
- Improves reuse (address can be shared across other models).

---

## Overall benefits of refactoring

Across these techniques, refactored code typically becomes:

- Easier to read (less branching, clearer intent, better structure)
- Safer to change (new cases are additive, fewer ripple edits)
- Easier to test (smaller units, explicit collaborators)
- More reusable (logic moved into focused types/functions)

