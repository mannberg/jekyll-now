---
layout: post
title: Using enums to create flexible data models
---

In this post we're going to look at how enums can be used as alternatives to structs and classes for creating data models. In Swift, enum cases can have so called *associated values* of any type, which is basically the same thing as properties in structs/classes. This make them quite flexible, as each case can store values that are relevant only to itself, yet they are all part of the same enum type.     

Enough chit-chat, let's write some code! Let's say we're making an app for a restaurant, with a menu made up of different categories of food. We might create a simple struct like this:

{% splash %}
struct MenuItem {
    var name: String
    var price: Int
}
{% endsplash %}

This restaurant wants it's food to be grouped by either of the categories meat, fish or dessert, so we'll create an enum within our struct and add a case for each category.

{% splash %}
struct MenuItem {
    var name: String
    var price: Int
    var category: Category

    enum Category {
        case meat
        case fish
        case dessert
    }
}
{% endsplash %}

We can now create different types of dishes like this:

{% splash %}
let cheeseburger = MenuItem(name: "Cheeseburger", price: 4, category: .meat)
let bananaSplit = MenuItem(name: "Banana Split", price: 3, category: .dessert)
{% endsplash %}

After a bit of coding, we realise we need to add beverages to the menu. Since drinks also have names and prices it seems reasonable to group these with the other menu items:

{% splash %}
struct MenuItem {
    var name: String
    var price: Int
    var category: Category

    enum Category {
        case meat
        case fish
        case dessert
        case beverage
    }
}
{% endsplash %}

In this particular restaurant, all drinks come in sizes big or small, so we'll want a way to store that data as well. One way to do this would be to add a Size enum to our MenuItem struct, and then add a size property to hold this value. Since all edible menu items come in one size only it wouldn't make sense for them to have a size value, so that property would need to be optional:

{% splash %}
struct MenuItem {
    var name: String
    var price: Int
    var category: Category
    var size: Size?

    enum Category {
        case meat
        case fish
        case dessert
        case beverage
    }

    enum Size {
        case big
        case small
    }
}
{% endsplash %}

We could now create instances of MenuItem like this:

{% splash %}
let bigMojito = MenuItem(name: "Mojito", price: 3, category: .beverage, size: .big)
let bolognese = MenuItem(name: "Spaghetti Bolognese", price: 7, category: .meat, size: nil)
{% endsplash %}

This way of doing things is not very clean though, since the MenuItem struct is cluttered with a property that for most items is irrelevant and will be set to nil. Passing nil to the constructor doesn't look very nice, and for someone not familiar with the code it might not be obvious which kind of item is supposed to get a non-nil value and which one doesn't (big/small cheeseburger?).

Instead of passing nil around, we could for example add an init method with the size parameter given a default value of nil, but even so our struct would contain a property it would not need for the most part.

This is were things could be solved quite nicely by using an enum. As mentioned earlier, enum cases can store associated values of all kinds which make them just as fit for structuring data. Let's rethink our menu a bit and replace the MenuItem struct with an enum. We'll start by just adding the non-liquid categories to our enum:

{% splash %}
enum MenuItem {
    case meat(name: String, price: Int)
    case fish(name: String, price: Int)
    case dessert(name: String, price: Int)
}
{% endsplash %}

The parameters (name, price) are the associated values, and these are passed like any function arguments when creating an instance of the enum case. For example, we would now create our cheeseburger object like this:

{% splash %}
let cheeseburger = MenuItem.meat(name: "Cheeseburger", price: 4)
{% endsplash %}

The names, types and amount of associated values for each case are completely detached from those of the other cases. This will be clear as we'll now add our beverage case to the MenuItem enum. Since we still want our drinks to have different sizes, we'll also add the Size enum as a member of MenuItem.

{% splash %}
enum MenuItem {
    case meat(name: String, price: Int)
    case fish(name: String, price: Int)
    case dessert(name: String, price: Int)
    case beverage(name: String, price: Int, size: Size)

    enum Size {
        case big
        case small
    }
}
{% endsplash %}

Notice how we now have drink case with a unique size parameter. Instead of having to bother with passing nil values for other items, we have a nice set of categories which are all of the same type (MenuItem), but can be easily extendable. We could now create items like this:

{% splash %}
let seaBass = MenuItem.fish(name: "Grilled Sea Bass", price: 7)
let frozenMargarita = MenuItem.drink(name: "Frozen Margarita", price: 5, size: .big)
{% endsplash %}

What's really good about this technique is it's flexibility - we can easily add or remove properties from any case without affecting the others. For example, sometime down the road we might decide that our app users should be able to choose how their meat will be prepared (it's *doneness* - rare, medium etc.). We might also decide that all desserts should be free of charge, and thus won't need a price property. This could easily be changed in our enum without bothering with optionals:

{% splash %}
enum MenuItem {
    case meat(name: String, price: Int, doneness: Doneness)
    case fish(name: String, price: Int)
    case dessert(name: String)
    case drink(name: String, price: Int, size: Size)

    enum Doneness {
        case rare
        case medium
        case wellDone
    }

    ...
}

let steak = MenuItem.meat(name: "Steak", price: 8, doneness: .rare)
let freeDessert = MenuItem.dessert(name: "Creme Brulee")
{% endsplash %}

Just as struct and classes, enums in Swift can also contain logic in the form of methods and computed properties. Why not serve drinks at half the price on Sundays:

{% splash %}
extension Dish {
    var price: Int {
        switch self {
        case let .beverage(_, price, _):
            return Date.todayIsSunday ? price / 2 : price
        default:
            return price
        }
    }
}
{% endsplash %}

As you can see, enums can serve as a great alternative to structs and classes. Whenever you find yourself creating a struct containing an enum that declares different states defining it, chances are good you could refactor your struct to into being an actual enum.

Thanks for reading!
