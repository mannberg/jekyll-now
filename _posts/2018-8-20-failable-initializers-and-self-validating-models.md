---
layout: post
title: Failable initializers and self-validating models
---

###### Syntax highlighting powered by [Splash](https://github.com/JohnSundell/Splash)

Often when coding, we need to deal with validating user input. A typical example would be a registration form, where you'll often have validation logic for email addresses, phone numbers, credit cards etc. This post is going to look at different examples of structuring this kind of code. Let's use a really simple example, where a user has to fill in mandatory values for name and email address, where the latter will be validated locally. After completing the form, we'll create a ```User``` object. We're not going to bother at all with the actual validation logic here, but rather on creating the interface for it. We'll start by writing code which can be improved and then refactor this as we go along.

First, well create a ```User``` struct:

{% splash %}
struct User {
    var name: String
    var emailAddress: String
}

let user = User(name: "Dohn Joe", emailAddress: "dohn.joe@somecompany.com")
{% endsplash %}

Before we pass user data on to our backend, we'll want some local validation logic for the email address. One possible place to put this code could be in some sort of dedicated validator struct:

{% splash %}
struct EmailValidator {
    func isValidEmailAddress(_ string: String) -> Bool {
        ...
        return valueOfValidation
    }
}
{% endsplash %}

We could now use our validator like this:

{% splash %}
let addressToValidate = "dohn.joe@somecompany.com"
if EmailValidator().isValidEmailAddress(addressToValidate) {
    let user = User(name: "Dohn Joe", emailAddress: addressToValidate)
}
{% endsplash %}

Besides a Validator being a inelegant name for a class or struct, another problem with ```EmailValidator``` is that it's usage is optional. By mistake we could bypass email validation and still create a user object, leaving room for potential errors:

{% splash %}
let user = User(name: "Dohn Joe", emailAddress: "")
{% endsplash %}

A possible (but not very elegant) way to solve this could be injecting the validator into our ```User``` struct. We could also add what is called a *failable initalizer* to ```User```, which is an init method that returns either an object or a nil value, depending on whatever conditions we set. In this way, we would always have to pass a valid email address to get a instance rather than getting nil:

{% splash %}
struct User {
    var name: String
    var emailAddress: String

    //Failable initializer, notice the question mark
    init?(name: String, emailAddress: String, validator: EmailValidator) {
        guard validator.isValidEmailAddress(address) else {
            return nil
        }

        self.name = name
        self.emailAddress = emailAddress
    }
}
{% endsplash %}

This would allow us to create users like this:

{% splash %}
if let user = User(name: "Dohn Joe", emailAddress: "dohn.joe@somecompany.com", validator: EmailValidator())
{
    //We have a user!
}
{% endsplash %}

Unfortunately, this is not pretty. What's good is that we now can't create a user object without a valid email address, but what's bad is that the data model is deviating from the real world (however abstract) object it is modelling. A user has a name and an email address, but it surely doesn't have a validator.

If we ever want to change our business logic to allow a user to register without an email address, we would now have to make not only one but two parameters optional, since it would not make sense to have a non-optional validator for an optional value:

{% splash %}
struct User {
    var name: String
    var emailAddress: String?

    init?(name: String, emailAddress: String?, validator: EmailValidator?) {
        ...
    }
}
{% endsplash %}

This would give us quite a weird interface where we could pass an email address without a validator:

{% splash %}
let user = User(name: "Dohn Joe", emailAddress: "dohn.joe@somecompany.com", validator: nil)
{% endsplash %}


A cleaner way to do this is to put the validation where it belongs, which is with the email address rather with the user. We'll create an ```EmailAddress``` struct containing both the actual ```String``` value for the address, and the validation logic. If we also add a failable initializer just as we did with ```User```, we make sure we only get an object instance to work with if our precondtions are met.

{% splash %}
struct EmailAddress {
    //A private setter allows us to retrieve, but not change, the string value from outside our struct
    private (set)var value: String

    init?(addressToValidate address: String) {
        guard isValid(address) else {
            return nil
        }

        self.value = address
    }

    private func isValid(_ string: String) -> Bool {
        ...
        return valueOfValidation
    }
}
{% endsplash %}

```User``` will then hold a reference to ```EmailAddress```:

{% splash %}
struct User {
    var name: String
    var emailAddress: EmailAddress

        init?(name: String, emailAddress: EmailAddress) {
            self.name = name
            self.emailAddress = emailAddress
        }
}
{% endsplash %}

We now have a lot cleaner api where the user does not have to bother with email validation, and we as coders don't have to bother with non-natural sounding structs named stuff like Validator.

{% splash %}
guard let validAddress = EmailAddress(addressToValidate: "dohn.joe@somecompany.com") else {
    print("Not a valid email address!")
}

guard let user = User(name: "Dohn Joe", emailAddress: validAddress) else {
    return
}
{% endsplash %}

If we're later adding more data types to ```User``` that also needs validation, we could follow the same structure. If we're adding several types we might want to create a protocol:

{% splash %}
protocol Validatable {
    var value: String { get }
    func isValid(_ string: String) -> Bool
}

struct EmailAdress: Validatable {...}

struct CreditCard: Validatable {
    private (set)var value: String

    init?(numberToValidate number: String) {
        self.value = number

        guard isValid(number) else {
            return nil
        }
    }

    func isValid(_ string: String) -> Bool {
       ...
       return valueOfValidation
    }
}
{% endsplash %}

We'll add an instance of ```CreditCard``` to ```User``` and now we can create user objects like this:

{% splash %}
guard let validAddress = EmailAddress(addressToValidate: "dohn.joe@somecompany.com") else {
    print("Not a valid email address!")
}

guard let validCreditCard = CreditCard(numberToValidate: "123456") else {
    print("Not a valid credit card number!")
}

guard let user = User(name: "Dohn Joe", emailAddress: validAddress, creditCard: validCreditCard) else {
    return
}
{% endsplash %}

That's all I had for this time. Hopefully you found something to your liking, and if not, that your disapproval of my coding style at least gave you some new fresh ideas on how to *not* do things :) I would love some feedback from anyone reading, so I'll make sure to get comments to work ASAP. Thank you for reading!
