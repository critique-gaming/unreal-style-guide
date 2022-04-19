# Critique Gaming Unreal Engine style guide and tips

## Directory structure

Follow [this](https://github.com/Allar/ue5-style-guide/tree/v2) style guide.

Main takeaways:
* Prefix your assets with the asset type
* Don't make directories for each asset type. Instead put all assets related to one of the game's components together in one place.

## C++

We follow [Epic's Coding Standard Guide](https://docs.unrealengine.com/4.27/en-US/ProductionPipelines/DevelopmentSetup/CodingStandard/), with a few notable exceptions:

1. We don't put `{` and `}` on separate lines unless it's a function body. Epic uses Allman style. We use a variation of [K&R style](https://en.wikipedia.org/wiki/Indentation_style#K&R_style). We also always put braces around single if statements, because [it can be a source of errors when refactoring](https://dwheeler.com/essays/apple-goto-fail.html).

2. Don't add a copyright line at the top of the file. It's just extra boilerplate.

Additional style rules, tips and tricks:

3. Strive to write semantic code above all else! The code should express the intent of what it is doing, even if it comes at a small performance penalty. Optimise later.

4. Meaningful comments or no comments at all. Adding `// Gets the value` to a `GetValue()` function only increases the line count and makes things harder to read.

5. "What" are you working with is more important than "how" are your doing something. First figure out the way your data is structured and how it flows and the implementation will follow. Refactoring data structures / migrating data is much more of a pain in the ass than refactoring code.

6. When writing a class or some other interface, first sketch out how the public API will look like, then delimitate it from the implementation. To that end, use `protected/private` properly. If a piece of data or a function is only relevant to the class you're writing, it's `protected/private`. If something is still needed by some external helper class, but it's obviously not part of the public API, see if you can use the `friend` keyword before making it `public`.

7. Don't do the extreme OOP thing where you hide every instance variable behind a getter. It's fine to have public instance vars. Just don't go overboard with it and be prepared to refactor them into getters in the future, if it becomes necessary to add extra logic. 

8. Prefer `for (FName Name : Names)` to `for (int32 Index = 0; i < Names.Num(); i++)`. The first is more semantic. It expresses the intent of "do something with each Name in Names" better than adding the concept of an Index.

9. Use enums instead of many booleans. When unsure if a thing will have more than 2 states in the future, use an enum instead of a bool, because you'll most likely end up refactoring it into an enum anyway when it happens. Ex: Instead of `bIsWeapon`, prefer to make a ECardType enum that can be `Character` or `Weapon`. Then when we decide we want to have armors into the game, it's easier to add `ECardType::Armor`.

10. If you want to iterate through the possible values of an enum, use `for (EMyEnum Enum : TEnumRange<EMyEnum>())`. Don't forget to do `ENUM_RANGE_BY_FIRST_AND_LAST` to enable it for thar particular enum.

11. Don't add a `None` entry into the enum unless `None` is one of the actually valid options of the enum. Use `TOptional<EMyEnum>` if you need to have a "null state".

12. Enums are not ints!!! Yes, they may be represented internally by ints, but they are an abstract type and you shouldn't think of them as ints. Don't ever cast between an enum and an int unless you have a very good reason to do it.

13. If you write a function definition in a `.h`, mark it as `inline` or `FORCEINLINE`. Otherwise, if you use the function in two different compilation units (`.cpp` files), you might get linker errors (redefined symbol). If you want to know more, read about the [One Definition Rule](https://en.cppreference.com/w/cpp/language/definition) (Side-note: `inline` doesn't really mean the compiler will decide to inline your function in modern compilers. It's only useful to prevent ODR issues like this one. The compiler decides wether to inline or not any given function regardless wether it has `inline` or not. If you don't trust the compiler, you can use `FORCEINLINE` to force it to ignore its own cost-benefit analysis and always inline).

14. Pass any complex types (bigger than 16 bytes) by reference. If it's passed as an input argument, mark it as `const`. Examples:

```c++
void Foo(FMyBigStruct InputArg); // Bad. Causes a copy
void Foo(const FMyBigStruct& InputArg); // Good

void Bar(FArray<int32> InputArg); // Bad. Causes a copy of the whole array
void Bar(const FArray<int32>& InputArg); // Good

void Baz(FName SomeName); // Good. FName is a small 64-bit struct
void Baz(const FName& SomeName); // Bad. The extra indirection causes a memory fetch and might cause the CPU cache to miss

for (FMyBigStruct Item : Items) // Bad
for (const FMyBigStruct& Item : Items) // Good
```

15. Don't return references to local variables from a function. They'll point to invalid memory after the function finishes.

```c++
FArray<int32>& GetValues()
{
    FArray<int32> Result = (1, 2, 3);
    return Result; // Bad! The reference becomes invalid as soon as the function returns.
}
```
---

16. Read about move semantics. [(1)](https://www.artima.com/articles/a-brief-introduction-to-rvalue-references) [(2)](https://jonasreich.de/build/blog/001-ue4-move-semantics.html). It's interesting. They allow you to return "big" containers like `TArray` and `TMap` that implement move constructors from functions without the cost of copying. You don't have to necessarily know how to implement one, but it's useful to know about their behaviour. For example:

```c++
FArray<int32> GetValues()
{
    FArray<int32> Result = (1, 2, 3, 4);
    return Result;
}

FArray<int32> Values = GetValues();
```

You'd think here, the `Result` array gets constructed, then copied into `Values` (which would involve a memory allocation and a copy of each item, potentially very expensive).

But no! `TArray` implements a move constructor!

An `TArray` is just a small struct with the number of items and a pointer to a block of storage memory for the array.

When the temporary value (r-value reference) that `GetValues()` returns is supposed to be copied into `Values`, a move happens instead (because the temp value won't ever be needed anymore). As part of its move constructor, the `Values` TArray just copies over the item count and the pointer to the memory storage from the temp array, which is much faster than its regular copy behaviour.

Still, don't do this:

```c++
struct FMyHugeStruct {
    uint8 ALotOfData[2048];
};

FMyHugeStruct GetHugeThing()
{
    FMyHugeStruct Result;
    // Fill in Result
    return Result;
}

// Unless the compiler decides to inline GetHugeThing(), this will still trigger a copy
FMyHugeStruct MyHugeThing = GetHugeThing();
```

Move semantics only works for objects that "own" external memory like `FString`/`TArray`/`TMap` without being too large themselves. If your struct is inherently large (`sizeof(YourStruct)` is big), the copy will still be expensive.

In these cases, prefer passing a reference to your output struct to `void GetHugeThing(FMyHugeStruct& Out)`.
