---
layout: post
date: 2025-10-18 12:00:00 -0400
---

# Notes on writing correct software

## (from the trenches)

**Oct 16, 2025**

**Likes:** 1

There’s a _ ****_ kind of bug where you intend to solve a problem, but the code does not match your intent. The following notes contain _**heuristics**_ **(not rules)** I use to minimize this type of bug.

I’ve contributed 1000+ changes to [WarpStream](https://www.warpstream.com/), a distributed database built on top of object storage, which is a million+ line codebase at this point, and I have introduced < ~10 _discovered_ bugs **** so far partly due to the application of these heuristics. This includes _all_ of the most correctness sensitive components of the system.

 _(Note that there’s many other processes like layers upon layers of testing and code review also preventing bugs, but I have a particularly low incidence of bugs even then. I won’t be covering testing and code review in this post.)_

#### Manipulating risk tolerance

The risk you’re willing to take when making a change is directly proportional to how fast you can make it, but you increase the chance that you introduce a bug.

For example, if you have a feature flagged low blast radius change which you can remotely turn off in minutes, then it’s probably acceptable to roll the change out to production as soon as the tests pass without allocating too much of your brain on it.

On the other hand, if you’re working on a sensitive part of the storage engine that could corrupt a customers data, you could take less risk by _a)_ writing a proof of correctness for the change, _b)_ spending time adding tests for subtle edge cases, _c)_ letting the change bake in your staging environment for days, and only then rolling the change out.

My brain naturally tends towards minimizing uncertainty and I found that I could afford to take more risk on 80% of the changes I was making to gain speed. Your brain might be that of a gambler, getting a lot done, but naturally unsuited for a problem which requires some care. It’s up to you to understand where your brain stands and learn how to make the tradeoff so that you can successfully work on all kinds of problems.

#### Search

If you’re a programmer, one of the domains you allocate your brain on is the codebase. To write correct code, you have to understand its mysteries and build an accurate mental model of it. The search process of following the various _**references**_ (chains of function calls, variables, implementations of interfaces, etc) in the codebase is how you build this mental model.

From this follows the __ insight that _**code(information) should be coalesced in a single place as much as possible**._ Ideally, you can read something from top to bottom and understand exactly how it works rather than having to follow N different threads to ascertain your mental model. Gathering context to make a decision should be as easy as possible. The added benefit of this approach is that it is trivial to provide context to your LLM/agent since it is in one place.

In practice this means that I write long functions, avoid unnecessary branching in the function call tree, keep code in a single file as long as it’s practical, and minimize use of language features like interfaces, generics, and operator overloading. Code is written to make navigating it a joy because navigating code is how mental models are built.

##### Long functions

Contrary to popular advice, I don’t mind a 1000 line function.

Assume that instead of 1000 line function **F** , you have a function **G** which calls 20 50 line child functions. **F** can be read from top to bottom with flow, but to understand **G** you have to jump to twenty locations in the codebase to build an accurate mental model.

 _ **Jumping from G to the child functions a) has a context switching cost, and b) has a backtracking cost because you have to navigate back to G.**_ These _navigation_ costs might not seem bad to some(to me they’re already terrifying), but they can get _exponentially worse_ if the child functions also have child functions of their own.

Aside from the navigation costs described above, using child functions implies that the _**invariants maintained by the child function must be carefully documented** , ****_ so that _a)_ the caller never assumes an invariant which isn’t maintained by the child, _b)_ the caller doesn’t accidentally duplicate functionality which child already performs, _c)_ the child is aware of the invariants maintained by the parent, and _d)_ the child doesn’t duplicate functionality from the caller. _ ****_ For example, the child function should not perform any logging if the parent is performing the logging. These kinds of concerns are non-existent when the child function is inlined.

Now assume that instead of modifying **G** , you’re trying to debug a bug in **G**. In this case it’s clear that you will have to jump into the child functions, because the bug might be in the child functions and you can’t trust the invariants which the child functions document in the case of a bug. If the code was inlined, that entire navigation step could be avoided.

Lastly, if **G** has a child function **C_1** , then another caller will eventually, and sometimes unnecessarily, call this function. Any future modifications to **C_1** now must ensure that the modification is correct for the second caller of **C_1**. On the other hand, if **C_1** is inlined, you don’t have to ensure that the modification doesn’t break the second caller. A corollary of this point is that _**sometimes copying code is better than forcefully sharing code when it’s not necessary**. _Of course, there’s many strong reasons to share code, and I use functions to share code all the time, but just keep this in mind when you don’t have a strong reason.

To play the devil’s advocate, there’s a strong argument to be made _for_ **** child functions because they give some variables their own scope and ensure that variables are not accidentally accessed or modified by code which shouldn’t have access to them. Thankfully, modern programming languages like Go can create scopes without creating new functions as illustrated in the next example.
    
    
    func scopeExample()  {
        // scope 1.
        {
            // Read only validations.  
        }
    
        // scope 2.
        {
            // Some mutations.
        }
    }

Of course I don’t believe that all code should be inlined and that the entire codebase should be a single function. It’s a judgement call, but I tend towards long functions whenever possible.

##### Interfaces

In this section, I’m going to talk about Go interfaces in particular and why I hate using them. I love the general concept of an interface(not a Go interface), and in Go you can use a struct and a few associated functions to model it. _**The reason I model interfaces this way is because every time I run into a Go interface while navigating code, the next step is to always go look at the implementation of the interface.**_ This implies that for my purposes of ascertaining correctness a Go interface just stands in the way.

If you only ever expect an interface to have one implementation, then don’t use an interface. By using a Go interface, you’re introducing at least 1 layer of branching when navigating the code base. If someone clicks on an interface value to see how it works, they’ll have to first navigate to the implementations of the interface.

If an interface has multiple implementations, then figuring out which implementation you care about in the moment is another hurdle which adds more navigation. One of my worst programming nightmares is looking for the implementations of an interface and finding a 100 of them.

 _ **I only ever use interfaces to provide a test only implementation in rare cases, or for general purpose APIs for which I expect many implementations** (you can’t really help but use one in this case.)_

##### Operator overloading

I have not programmed in a language which uses operator overloading recently, but if it were available to me, I wouldn’t use it.

Basically, I never want to wonder what the following code means semantically. 
    
    
    type test struct {
    	a int
    	b int
    }
    
    func main() {
    	x := test{1, 2}
    	y := test{1, 2}
    	if x == y {
    	    fmt.Println(”yay!”)
    	}
    }

Overloading the == to operator in this code potentially adds another navigation step.

##### Generics

 _T_ hey obscure details and lead to worse mental models. I avoid them as much as possible, and only use them when I need to write a general purpose library like a custom data structure which I expect to be used all over the codebase.

##### Code navigation

We’ve talked about writing code to make navigation easy. But when I started programming the process of code navigation which I followed was incomplete. It was mostly based on cmd+f and regex pattern matching which are sufficient, but not optimal tooling for codebase navigation.

Before diving into what I do now, we need to talk about the process of adding functionality to code. To add functionality you must ensure that changes you’re making keep the existing functionality unchanged, or if you do change existing functionality, you do it in a way which doesn’t break the system.

Say you need to change the properties of a variable _**x**_. Maybe it was a _uint64_ , but you need to change it to an _int_ for some reason. To ensure that this change is valid you must check every single reference _**x**_. Say _**x**_ is used to compute _**y**_ so you now need to check every single reference to _**y**_. Essentially you must perform a _**complete**_ ****_**depth first search(DFS)**_ across the _**reference**_ graph. The search is called complete, because you must not miss any reference to ensure maximal correctness.

Similarly, if you’re changing a function, you must navigate every reference(caller) of the function, and then every reference of those caller functions, and so on.

 _ **It’s clear that as a codebase grows, the process of navigating a codebase becomes exponentially more complicated.**_ Since code navigation is how correct mental models of the code are built, I structure code to prioritize navigation.

So, how do you perform a complete DFS **** through the codebase? I use the _**find all references** _functionality provided by the Go language server and combine it with _**forwards and backwards navigation**_ functionality provided by vscode.

[![](https://substackcdn.com/image/fetch/$s_!Jg8B!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F2faba921-a4b0-4892-b0bf-15ff1f6e2d0f_2758x676.png)](https://substackcdn.com/image/fetch/$s_!Jg8B!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F2faba921-a4b0-4892-b0bf-15ff1f6e2d0f_2758x676.png)vscode displaying references to the listStreams function

If you click on one of the references, vscode is going to jump to a new location in the codebase. You can use the forward and backward buttons in vscode to navigate through these locations. In the above example, once you investigate one of the references of the _**listStreams**_ **** function, you can jump back to the original location by clicking on the back button. This enables traversal through the reference graph in the codebase.

[![](https://substackcdn.com/image/fetch/$s_!UpJ8!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fa10880ce-4333-4f25-a2e2-5bcce98501f4_700x134.png)](https://substackcdn.com/image/fetch/$s_!UpJ8!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fa10880ce-4333-4f25-a2e2-5bcce98501f4_700x134.png)forward and back buttons to navigate the reference graph

If you’re willing to perform the exhausting work of making sure you traverse the reference graph correctly, and you’re willing to make sure that the code around each of these references in the graph doesn’t break after your change, then you can decrease the likelihood of bugs in your code. _(Note that even if you do this process perfectly without human error, you can still introduce other kinds of bugs.)_

 _ **Every single engineer I’ve worked with has told me that I’m extraordinarily good at spotting bugs just by reading code. Half of that skill is due to this navigation strategy.**_

Though I’ve been pretty happy with my navigation methodology, I think it’s possible to build new tooling which makes traversing the reference graph a joy. Even now I have to maintain quite a lot of mental state when navigating back and forth through the references and I think it’s possible to eliminate this mental overhead entirely.

 _(As an aside, I know that AI agents are built to be more general purpose, but I find it astonishing that they are not trained to use the language server, and are not trained to perform a complete DFS across the references using a language server)._

#### Implementation tips

 _The best way to avoid bugs is to not change code unless it’s necessary. ****_ But I follow the following strategies when I need to.

##### Invariants

Write down invariants when you write code. What properties is the function maintaining? What does the function expect from the caller? Does the function own the byte slice once it is passed into it? Why is the complicated nested loop guaranteed to terminate?

I think through invariants and a proof of correctness for every single line of code I write _(It’s a result of my CS background and going to a school which focussed on proofs.)_ This slows me down a lot, and I’m learning to take more risk when appropriate. 

Suppose a caller function **F** is calling some function **G** which is defined in a separate package from **F**. The author of function **G** was nice enough to document the invariants that **G** maintains, and what it expects from caller functions.

The question is, _a)_ should **F** rely on invariants maintained by **G**?, and _b)_ should **F** check the invariants maintained by **G**?.

I think if you can easily express a solution without relying on all the invariants from **G** , then you should do so. This ensures that your code will keep working correctly even if **G** breaks some of its invariants.

If you end up relying on all the invariants maintained by **G** , you should ideally assert that **G** has indeed maintained those invariants. In cases where **G** is defined in a separate package from **F** , or even in cases where **G** is part of a separate project which you’re pulling in, it’s unrealistic to go read **G** ’s code every time you use an arbitrary function **G**. So the least you could do is assert invariants to ensure that **G** is working correctly.

The next question is, should **G** assert invariants it expects from its input parameters? If **G** is particularly correctness sensitive and the assertion is not particularly performance sensitive, the invariants should be asserted by **G**.

If **G** does not document an invariant, then **F** should never rely on it. Don’t assume that **G** must obviously be doing something and then rely on that behaviour.

Lastly, invariants also serve as excellent documentation which your fellow engineers and AI will appreciate.

##### Refactoring

If I’m making a change, and I think I need to refactor code to express my change clearly, I would refactor without hesitation and I do this all the time. Ensure that the refactor is very obviously mechanical.

But sometimes people refactor code to make it cleaner or to change it so that it matches their coding style better. Which brings me to my next point: _**don’t refactor code when you don’t have a strong reason to do so**_. _a)_ You might introduce a bug _(I’ve seen this happen all the time)_ , and _b)_ you will break someone’s existing mental model about the code for no reason.

##### Pointer access

I don’t access fields for multiple pointers in a single line of code. If there’s ever a panic due to nil pointer access, the panic trace from the Go compiler doesn’t make it instantly clear _which_ pointer was nil.

If you access each pointer on its own line, then you can quickly use the line number from the panic trace to determine which pointer was nil.

##### Limiting data access

Say you have some large struct **S** with a lot of data and pointer fields and some function **F** which must access one of the pointer fields **P** in **S** and then modify the value pointed to by **P**. It is possible to write **F** as either **F_1** or **F_2** illustrated below.
    
    
    type P struct {
        a int
        b int
    }
    
    type S struct {
        x int
        y int
        p *P
    }
    
    // F_1 increments s.p.a by 2.
    //
    // INVARIANT: F_1 must not mutate any other field in S.
    func F_1(s *S) {
        s.p.a += 2
    }
    
    // F_2 increments p.a by 2.
    //
    // F_2 naturally ensures that arbitrary state in S
    // cannot be mutated as it is restricting data access.
    func F_2(p *P) {
        p.a += 2
    }
    
    func main() {
        state := S{
            x: 1, y: 1,
            p: &P{
                a: 2, b: 2,
            }
        }
    
        F_1(&state)
        F_2(state.p)
    }

 **F_1** and **F_2** have identical behaviour. They both increment **state.p.a** by 2. But **F_1** takes a pointer to **state** as a parameter and **F_2** takes **state.p** _(type *P **)**_ as a parameter.

Writing functions in the **F_2** style ensures that the function will never be able to modify fields in **state** like **x** or **y** which it shouldn’t have access to.

But say you’re a programmer who wants to see how **state.p** is used throughout the codebase. So you look for all references to **p** in the **state** struct. You will see 2 references to **state.p** , once in the **F_1** function where **p.a** is incremented and once in the **main** function where **F_2** is called. To fully understand how **state.p** is used, now you must jump into the **F_2** function, and find all references to the input parameter **p** in **F_2**. If **F_2** was written similarly to **F_1** , you wouldn’t have to call find all references twice.

I used to be a huge proponent of the **F_2** style of function which limits the data which the function can access and modify, but it adds an additional navigation step and you have to make that tradeoff.

#### Conclusion

Well that’s it. This is how I think about the details of coding for correctness right now. If you disagree I’m happy to discuss and potentially have my opinions changed.

Thanks for reading! Subscribe for free to receive new posts and support my work.

Subscribe
