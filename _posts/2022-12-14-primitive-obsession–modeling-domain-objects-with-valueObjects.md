---
layout: post
title: Primitive Obsession – Modeling Domain Objects with ValueObjects
thumbnail-img: /assets/img/posts/primitive-obsession/1.webp
share-img: /assets/img/posts/primitive-obsession/1.webp
tags: [domain-driven-design, value-objects, primitive-obsession, code-smells, clean-code]
---

When developing our applications, we use objects of various types. These objects, depending on the complexity of the developed application, encapsulate a wide range of data types and business rules within their structures.

We often define the data within the structures of objects using the built-in types provided by the environment in which we have developed, such as primitive types (`bool`, `int`, `string`, `long`, `double`). For example, many properties within a `user entity`, such as name, surname, password, and email, are often defined as `string` for both speed and simplicity.

However, defining these properties within objects in this way, especially in projects with intensive business logic, can lead to significant code complexity (primitive obsession code smell) as the project progresses.

For example, consider developing a financial application. Let’s assume that in this financial application, we store the IBAN numbers of all users in the `account entity`. If we were to do this with primitive types, the resulting `Entity` would be something like the following.

{% highlight csharp %}
public class Account : Entity, IAggregateRoot
{
    private int _id { get; private set; }
    private string _name { get; private set; }
    private string _ownerName { get; private set; }
    private string _ownerEmail { get; private set; }
    private string _currency { get; private set; }
    private string _balance { get; private set; }
    private string _iban{ get; private set; } // <---

    public Account(int id, string name, string _ownerName, string _ownerEmail, string _currency, decimal balance, string iban)
    {
        _id = id;
        _name = name;
        _ownerName = ownerName;
        _ownerEmail = ownerEmail;
        _currency = currency;
        _balance = balance;
        _iban = iban;
    }
}

// As an example, we can use the above code as follows in our services
...
    var account = new Account(
                  id: 1, 
                  name: "Private Account Name", 
                  ownerName: "John Doe",
                  ownerEmail: "john@doe.com",
                  currency: "AED",
                  balance: 1000,
                  iban: "TR10....."); // <---
...
{% endhighlight %}

However, as you may have noticed, using this code could lead to a lot of problems. In case of any bugs in the data sent to us by the client (web, IoT, mobile, integrated companies, etc.), or if developers on the client side are unaware of how the data should be formatted, it is highly likely that we will encounter issues.

We want IBAN like:

- TR2884….
- AE98123….
- QA89DOH…

instead following:

- 192039…
- ASDAD….
- 123123RIEOQ….

To safeguard against such unwanted situations and ensure the consistency of data, we can use `ValueObjects` in our development. A concept frequently encountered in Domain-Driven Design (DDD), ValueObjects are objects that exist within entities without having their own identity; instead, they provide information about them. We are concerned not with who these objects are but with what they are, what they represent.

As the name implies, ValueObjects are focused on values, setting them apart from Reference Objects. In ValueObjects, only instances with differing values can exist or should exist, in contrast to Reference Objects where different instances can have identical values (e.g., User entities with the same username, surname, and password but representing two different users). Having only one instance representing the same value can be beneficial in systems with heavy traffic and frequent object creation operations, leading to memory gains (Flyweight pattern). However, in highly distributed systems, this approach may raise performance concerns, and the choice between `copying` and `sharing` depends on the system.

An important point to note is that ValueObjects must be `Immutable`(although there are exceptions in certain cases). Let’s explain this through our financial application example. Imagine that we use a red color ValueObject to represent Accounts affected by fraud operations in our system. By using the Flyweight pattern, all accounts in the system point to a single red color instance, ensuring memory efficiency. However, the day came when we decided to change the status of one account in our financial application from red to green. If our ValueObject were mutable, changing the status of one account’s ValueObject to green would turn all red accounts in the entire system green (AliasingBug). This would lead to undesired consequences in the system. Therefore, an essential characteristic of ValueObjects is that they need to be `Immutable`.

To be more concrete:

{% highlight csharp %}
public class Color : ValueObject
{
    private readonly int _red;
    private readonly int _green;
    private readonly int _blue;
    
    public Color(int red, int green, int blue)
    {
        _red=red;
        _green = green;
        _blue = blue;
    }

    public override bool Equals(object obj)
    {
        var other = obj as Color;
        return other != null &&
            this.Red == other.Red &&
            this.Green == other.Green &&
            this.Blue == other.Blue;
    }

    public static bool operator == (Color lhs, Color rhs)
    {
        if (Object.ReferenceEquals(lhs, null)) {
            return Object.ReferenceEquals(rhs, null);
        }
        return lhs.Equals(rhs);
    }
}
{% endhighlight %}

For instance, instead of recreating a ValueObject representing the color green each time, it makes sense to use the existing green color instance everywhere. To achieve this behavior, we need to `override` the default equality check behaviors in the environment we are using. Otherwise, our code, by comparing references in memory, would claim that two objects are not the same even if their values are identical. For example:

{% highlight csharp %}
new Color(100,100,100) == new Color(100,100,100) // returns false without override
// ValueObjects should override this method to ensure equality is based on the content rather than reference.
{% endhighlight %}

Returning to our financial example, let’s examine what could happen when using a `ValueObject` for the IBAN number:

{% highlight csharp %}
public class Account : Entity, IAggregateRoot
{
    private int _id { get; private set; }
    private string _name { get; private set; }
    private string _ownerName { get; private set; }
    private string _ownerEmail { get; private set; }
    private string _currency { get; private set; }
    private string _balance { get; private set; }
    private AccountIBAN _iban { get; private set; } // <---

    public Account(int id, string name, string _ownerName, string _ownerEmail, string _currency, decimal balance, AccountIBAN iban)
    {
        _id = id;
        _name = name;
        _ownerName = ownerName;
        _ownerEmail = ownerEmail;
        _currency = currency;
        _balance = balance;
        _iban = iban;
    }
}

// As an example, we can use the above code as follows in our services
...
    var account = new Account(
                  id: 1, 
                  name: "Private Account Name", 
                  ownerName: "John Doe",
                  ownerEmail: "john@doe.com",
                  currency: "AED",
                  balance: 1000,
                  iban: new AccountIBAN("TR10.....")); // <---
...
{% endhighlight %}

With the above code, we defined the `IBAN` property as type `AccountIBAN`. The `AccountIBAN` class is:

{% highlight csharp %}
public class AccountIBAN : ValueObject
{
    public string IBAN { get; private set; }
    public AccountIBAN(string iban)
    {
        if (string.IsNullOrWhiteSpace(value))
            throw new ArgumentException("IBAN cannot be empty or whitespace.");

        // Perform IBAN format validation
        var regex = new Regex(@"^[A-Z]{2}\d{2}[A-Z0-9]{1,30}$");
        if (!regex.IsMatch(value))
            throw new ArgumentException("Invalid IBAN format.");

        // Additional validation logic based on IBAN structure and rules can be added here

        //If all configurations are correct, you can set the IBAN value.
        IBAN = iban; 
    }
}
{% endhighlight %}

By using the `AccountIBAN` class, we have safeguarded the structure of the `IBAN` type with our validation rules. If an undesirable `IBAN` number is provided, our code will throw an `exception`. This ensures that only data in the correct format is stored in the system, thereby maintaining data consistency. Additionally, instead of embedding IBAN type rules throughout our codebase, we have centralized this process.

Applying similar practices for other types such as email, currency, etc., would also be logical. Accordingly:

{% highlight csharp %}
public class Account : Entity, IAggregateRoot
{
    private int _id { get; private set; }
    private AccountName _name { get; private set; }
    private AccountOwnerName _ownerName { get; private set; }
    private AccountOwnerEmail _ownerEmail { get; private set; }
    private AccountCurrency _currency { get; private set; }
    private AccountBalance _balance { get; private set; }
    private AccountIBAN _iban{ get; private set; }

    public Account(int id, AccountName name, AccountOwnerName _ownerName, AccountOwnerEmail _ownerEmail, AccountCurrency _currency, AccountBalance balance, AccountIBAN iban)
    {
        _id = id;
        _name = name;
        _ownerName = ownerName;
        _ownerEmail = ownerEmail;
        _currency = currency;
        _balance = balance;
        _iban = iban;
    }
}

// As an example, we can use the above code as follows in our services
...
    var account = new Account(
                  id: 1, 
                  name: new AccountName("Private Account Name"), 
                  ownerName: new AccountOwnerName("John Doe"),
                  ownerEmail: new AccountOwnerEmail("john@doe.com"),
                  currency: new AccountCurrency("AED"),
                  balance: new AccountBalance(1000),
                  iban: new AccountIBAN("TR10....."));
...
{% endhighlight %}

This way, we have ensured the consistency of the data coming into our system.

{: .box-note}
Important Note: An object that is a ValueObject in one domain can be an Entity in another domain. There are no constants here; the type of objects can vary depending on the application. In the example above, we defined the information belonging to the Owner as a ValueObject, but in another application, it could be an Entity.

When ValueObjects come together, they also serve as a framework for properties that represent a composite value. Taking an example from an e-commerce system, when completing an order on the payment screen, we enter values such as address, zip code, city, district, etc. Although these values may seem different, when combined, they represent an address. Therefore, ShippingAddress and BillingAddress values can be defined as ValueObjects in this context. Let’s consider another example with a coordinate plane. We cannot express a point in the coordinate plane with just the (x) value; a point in the coordinate plane is represented as (x, y). Therefore, a point in the coordinate plane can be represented with a ValueObject.

Applying this to our financial example, if we implement ownerName and ownerEmail properties, they essentially define a single object, namely the owner of the account. Thus, we can define these two properties as a single ValueObject.

Finally, defining all types as ValueObjects can lead to another code smell type called `speculative generality`. It’s crucial to strike the right balance in this regard.

References:

- Evans, E. Domain-Driven Design: Tackling Complexity in the Heart of Software. (1st ed.)
- Fowler, M. Value Objects. [Online]. Available at: https://martinfowler.com/bliki/ValueObject.html
- Fowler, M. AliasingBug. [Online]. Available at: https://martinfowler.com/bliki/AliasingBug.html
- Khononov, V. Learning Domain-Driven Design: Aligning Software Architecture and Business Strategy. (1st ed.)