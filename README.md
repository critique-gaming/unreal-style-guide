# Critique Gaming Unreal Engine style guide and tips

## Directory structure

Follow [this](https://github.com/Allar/ue5-style-guide/tree/v2) style guide.

Main takeaways:
* Prefix your assets with the asset type
* Don't make directories for each asset type. Instead put all assets related to one of the game's components together in one place.

## C++

We follow [Epic's Coding Standard Guide](https://docs.unrealengine.com/4.27/en-US/ProductionPipelines/DevelopmentSetup/CodingStandard/), with a few notable exceptions:

1. We don't put `{` and `}` on separate lines unless it's a function body. Epic uses Allman style. We use a variation of [K&R style](https://en.wikipedia.org/wiki/Indentation_style#K&R_style). We also always put braces around single if statements, because [it can be a source of errors when refactoring](https://dwheeler.com/essays/apple-goto-fail.html).

1. Don't add a copyright line at the top of the file. It's just extra boilerplate.

Additional things:

1. Strive to write semantic code above all else! The code should express the intent of what it is doing, even if it comes at a small performance penalty. Optimise later.

1. Meaningful comments or no comments at all. Adding `// Gets the value` to a `GetValue()` function only increases the line count and makes things harder to read.

1. "What" are you working with is more important than "how" are your doing something. First figure out the way your data is structured and how it flows and the implementation will follow.

1. When writing a class or some other interface, first sketch out how the public API will look like, then delimitate it from the implementation. To that end, use `protected/private` properly. If a piece of data or a function is only relevant to the class you're writing, it's `protected/private`. If something is still needed by some external helper class, but it's obviously not part of the public API, see if you can use the `friend` keyword before making it `public`.

1. Don't do the extreme OOP thing where you hide every instance variable behind a getter. It's fine to have public instance vars. Just don't go overboard with it and be prepared to refactor them into getters in the future, if it becomes necessary to add extra logic. 

1. Prefer `for (FName Name : Names)` to `for (int32 Index = 0; i < Names.Num(); i++)`. The first is more semantic. It expresses the intent of "do something with each Name in Names" better than adding the concept of an Index.

1. Use enums instead of many booleans. When unsure if a thing will have more than 2 states in the future, use an enum instead of a bool, because you'll most likely end up refactoring it into an enum anyway when it happens. Ex: Instead of `bIsWeapon`, prefer to make a ECardType enum that can be `Character` or `Weapon`. Then when we decide we want to have armors into the game, it's easier to add `ECardType::Armor`.

1. If you want to iterate through the possible values of an enum, use `for (EMyEnum Enum : TEnumRange<EMyEnum>())`. Don't forget to do `ENUM_RANGE_BY_FIRST_AND_LAST` to enable it for thar particular enum.

1. Don't add a `None` entry into the enum unless `None` is one of the actually valid options of the enum. Use `TOptional<EMyEnum>` if you need to have a "null state".

1. Enums are not ints!!! Yes, they may be represented internally by ints, but they are an abstract type and you shouldn't think of them as ints. Don't ever cast between an enum and an int unless you have a very good reason to do it.

1. TODO: When to pass by reference and when not to
