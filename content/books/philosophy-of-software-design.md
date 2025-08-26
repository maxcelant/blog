---
title: A Philosophy of Software Design
---
#### Chapter 2: Nature of Complexity
There are three categories of complexity:
1. **Change Amplification**: A seemingly simple modification necessitates changes in many places.
2. **Cognitive Load**: How much a developer has to know in order to complete the task.
3. **Unknown unknowns**: When it isn't obvious which piece of code must be modified to complete the task.

The primary causes of complexity are **dependencies** and **obscurity**. When some piece of software depends on something else, it is inherently more difficult to reason about, possible increasing change amplification and cognitive load. Obscurity occurs when something in the system isn't obvious. Increasing both cognitive load and unknown unknowns. 

#### Chapter 3: Working Code Isn't Enough
You should avoid being a purely tactical programmer. Spend ~20% of your time planning ahead and being strategic with your design. If you don't, you will accumulate tech debt over time. 

#### Chapter 4: Modules Should Be Deep
The point of an interface is that it should hide some complexity from the implementation. If you have to read the implementation of a function in order to understand how to use it, the interface is bad. 

The key to designing abstractions is to understand what is important, and to look for designs that minimize the amount of information that is important.

A deep module is one in which the interface hides lots of complexity. Think of the UNIX I/O mechanisms like `read`, `write`, etc. They are very simple and hide a ton of complexity.

#### Chapter 5: Information Hiding
The goal is to encapsulate the logic in such a way that it is invisible to the interface it provides. Strive for a tighter interface to improve dependencies between modules. 

Look for places where information leakage is occurring. Common cases include when two classes are closely coupled—this usually means that they can be merged into one.  

**Temporal decomposition** is a common pitfall in which you segregate code into classes based on how the events unfold in time. For instance, logic that reads a file, modifies it and writes it out feels like it should be split into three separate classes. However, both the reading and writing class have knowledge of the file format, which means a change in the file format will cause ripples in both the reading and writing class.

>[!hint]
>You can detect this code smell if switching the execution order of some logic between classes causes them to break. Additionally if you notice two classes relying on each other too much.

Information hiding can typically be made better by _making classes larger_.

#### Chapter 6: General Purpose Modules Are Deep
Make the implementation specific while keeping the interface simple. This will give you the flexibility to make it more general purpose later without trying to predict the future.

The author reflects on a project he made his students do in which they had to create a GUI text editor. He warns us that his students created a TextEditor class that was to specifically tied to the UserInterface class. This resulted in methods in the TextEditor such as `delete()` and `backspace()`, which directly mimicked the UI. This is problematic because now we need a 1:1 mapping for every interaction. In this case, a more general TextEditor API would've helped. Creating something like this:

```go
func insert(pos Position, newText string)
func delete(start, end Position)
```

Now this more general API can be used to implement something like `backspace()` or even more complex or specific mechanisms.

##### Questions to ask yourself
What is the simplest interface that will cover all of my current needs?
In how many situations will this method be used?
Is this API easy to use for my current needs?

#### Chapter 7: Different Layer, Different Abstraction
A common code smell is **pass-through methods** in which a method does very little except for calling an underlying method. To solve this, you need to understand the responsibilities of each class. Either expose the lower level class directly to the caller, redistribute the functionality, or merge the classes together.

A wrapper like this can be useful in situations where you have a high level dispatcher that can call one of N number of underlying methods of the same signature. This adds functionality, unlike a simple pass-through.

**Pass-through variables** are those that get drilled down, but are unused until they reach the final layer. If it changes, then all those layers need to be updated to account for it. One possible fix is to store it in an existing object so that the individual variable doesn't need to be passed down. Another approach is to introduce a context object which is shared by the system.

#### Chapter 8: Pull Complexity Downwards
It is more important for a module to have a simple interface than a simple implementation. Think deeply about if a particular variable you are exposing to the user as a config option has an obvious default or not, this could indicate that it's a bad option to have available and it would be better to just have a solid default set.

#### Chapter 9: Better Together Or Better Apart?
There are certain indications to help decide whether two classes should be combined or if they are better kept apart. 

