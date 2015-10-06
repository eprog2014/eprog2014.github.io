---
title: "When class invariants are checked"
author: "Christian Vonrüti"
categories: mock-exam
---

This article gives an explanation as to why the following statement from the Mock Exam is false:

> The class invariant needs to hold before every procedure call.
>
> _Mock Exam_

There are two things amiss with this statement:

* Class invariants are not checked on every procedure call
* It does not state *which* class' invariant is checked on a procedure call

What are class invariants<sup>1</sup>?
===========================

Invariants of a class specify consistency requirements for all its objects.

One such requirement can either be met or violated by an object. Consequently, a requirement is expressed by a `boolean expression`.

Since boolean expressions allow to express dependencies between different features, checking an invariant after every single instruction is not very useful. This is demonstrated by the following, admittedly artificial, example.

Suppose we had expressed a dependency on two attributes and wanted to change their values, we can only do so by changing one after the other. Before getting the chance to re-satisfy the class invariant by changing the second attribute after having modified the first, the program would already have been aborted due to the first change violating the class invariant.


{% highlight eiffel linenos=table %}
-- Scenario: Class Invariant checked after every instruction
class
	APPLICATION

inherit
	ARGUMENTS

create
	make

feature {NONE}
	one: BOOLEAN
	two: BOOLEAN

	make
		do
			-- Initially, both attributes are False by default
			-- hence, the invariant is satisfied
			
			-- This would invalidate the invariant
			one := True
			-- Here, the program would crash
			two := True
		end

invariant
	both_the_same: one = two
end
{% endhighlight %}


This may be a toy example, but being able to specify constraints between different features is very useful in practice. Thus, enforcing the invariant after every single instruction is not useful in practice.

Relaxing
========

We could relax the guarantee that the object must satisfy the invariant after every instruction to only checking the invariant at the beginning and end of each routine, i.e. both procedure and function.

With this new policy, the above example could execute correctly.

Now follows an example as to why this new policy is too strict as well and how it prevents us from doing things we will often want to do in everyday programming.

{% highlight eiffel linenos=table %}
-- Scenario: Class Invariant checked before do and after end
class
    APPLICATION

inherit
    ARGUMENTS

create
    make

feature {NONE}
    one: BOOLEAN
    two: BOOLEAN

    make
			-- Special case, invariant must not hold 
			-- at beginning of creation procedure
        do
            -- Initially, both attributes are False by default
            -- hence, the invariant is satisfied

			-- Let's now adjust the attributes
			-- by using extra features to compute and set their values
			
			-- The invariant will be checked before
			--   the following instruction is completed.
			compute_and_set_one
			
			compute_and_set_two
		end
		
	compute_and_set_one
		do
			-- Do lots of other stuff to arrive at the value we need
			one := False
			-- Upon exiting this procedure, 
			-- the invariant is to be checked as per our policy.
			-- However, since only one attribute has been changed so far,
			-- the invariant is violated and the program will abort.		
		end
		
	compute_and_set_two
		do
			-- Do lots of other stuff to arrive at the value we need
			two := False
		end
invariant
    both_the_same: one = two
end
{% endhighlight %}

In practice, it is often useful to implement one feature by dividing it into multiple different parts, where each part is itself a simpler and shorter feature. For one, this allows to give different parts of the code different names, such that the code becomes self documenting. In the above example, we have used [this technique, usually called Extract Method](http://www.refactoring.com/catalog/extractMethod.html), twice. 

Another possibility would have been to only extract the right-hand side of the assignment. In that case, we would have needed to make our extra features functions instead of procedures, such that they can *return* a result. This is demonstrated below, also note, that the names of the features have been changed to reflect this difference in semantics.

{% highlight eiffel linenos=table %}
-- Scenario: Class Invariant checked before do and after end
class
    APPLICATION

inherit
    ARGUMENTS

create
    make

feature {NONE}
    one: BOOLEAN
    two: BOOLEAN

    make
			-- Special case, invariant must not hold 
			-- at beginning of creation procedure
        do
            -- Initially, both attributes are False by default
            -- hence, the invariant is satisfied

			-- Let's now adjust the attributes
			-- by using functions to compute their values
			
			-- The invariant will be checked twice before
			--   the following instruction is completed.
			one := compute_one
			
			
			-- Upon entering compute_two, the invariant is checked.
			-- At that point, the invariant does not hold
			-- so the program will be aborted.
			two := compute_two
		end
		
	compute_one: BOOLEAN
		do
			one := False
			-- Upon exiting this function, 
			-- the invariant is to be checked as per our policy.
			-- By that point, the attribute has not yet been set.
			-- Hence, the program will be able to set the attribute.
		end
		
	compute_two: BOOLEAN
		do
			-- Upon entering this function, the invariant is checked.
			-- Since `one` has been set to False but `two`
			-- is still True, the program will abort.
			two := False
		end
invariant
    both_the_same: one = two
end
{% endhighlight %}

The aforementioned technique, extract method, is very useful in practice, thus it would be unwise to check the invariant upon entering and exiting every routine: Eiffel would prevent us from leveraging this powerful idea for code organization on many occasions and our code would have to be more long-winded in many situations just to get those additional safety guarantees.

Actual Eiffel Semantics
========================

Invariants are checked, whenever there is a qualified call to a feature and a return therefrom. More precisely, the invariants of the callee (i.e. the one, that is being called, the target) are checked and those same invariants when returning from those same call<sup>2</sup>.

A qualified call is a feature access of the form `target.feature_name`. The previous examples *only* used unqualified feature calls such as `compute_one` or `compute_two`, where the target (or receiver) of the call---i.e. the object on which the feature will be called is *not* explicitely specified.


{% highlight eiffel linenos=table %}
class
	APPLICATION

inherit
	ARGUMENTS

create
	make

feature {NONE}
	make
		local
			n: EVEN_NUMBER
		do
			create n
			
			n.set_number(2)
			-- n.number is now 2
			n.set_number(3)
			-- n.number is now 4
		end

end
--------------------------------
class
	EVEN_NUMBER

feature
	-- Initially zero
	number: INTEGER
	
	set_number(n: INTEGER)
		do
			number := n
			fix_number
		end

feature {NONE}
	fix_number
		do
			if number \\ 2 = 1 then
				number := number + 1
			end
		end
	
invariant
	number_is_even: number \\ 2 = 0
{% endhighlight %}

Upon entering `fix_number`, the invariant is not checked, because it's an unqualified call.

Every feature call has a receiver. For an unqualified call, this receiver is implicitly `Current`, which in this case is equal to `APPLICATION`'s `n`.

However, if we were to replace the `fix_number` call and make the implicit receiver explicit, i.e. `Current.fix_number`, then the invariant will be checked upon entering the feature `fix_number` and the program will be aborted.

If we were to omit the call to `fix_number` entirely, the program would crash upon returning from `set_number` to line 19/20 of `APPLICATION`.

This next example will not abort due to an invariant violation:

{% highlight eiffel linenos=table %}
class
	APPLICATION

inherit
	ARGUMENTS

create
	make

feature {NONE}
	make
		local
			n: EVEN_NUMBER
		do
			create n
			
			n.set_number(2)
			-- n.number is now 2
			n.set_number(3)
			-- n.number is now 4
		end

end
--------------------------------
class
	EVEN_NUMBER

feature
	-- Initially zero
	number: INTEGER
	
	set_number(n: INTEGER)
		do
			number := n
			print_number
			fix_number
			print_number
		end

feature {NONE}
	print_number
		do
			Io.put_integer(number)
		end
	
	fix_number
		do
			if number \\ 2 = 1 then
				number := number + 1
			end
		end
	
invariant
	number_is_even: number \\ 2 = 0
{% endhighlight %}


When we first call `print_number`, the class invariant of the one `EVEN_NUMBER` object is invalidated. Inside `print_number`, there is a qualified call `Io.put_integer()` however. But this call apparently does not trigger a contract check on `EVEN_NUMBER`.

This is ok, because as long as nobody (as in: a different object, such as the one denoted by `Io`) accesses a feature of an object in an inconsistent state, nothing unexpected can happen.

Whenever an object wants to access a feature of a different object, it needs to do so via a qualified call. If the call weren't qualified, the receiver would implicitly be `Current` and accessing a different object than itself via `Current` is impossible.

Optimizing Eiffel Semantics
===========================

One could say that the goal of class invariants is the following:

> An object (in a valid state) can be used (i.e. from a different object) to perform a certain computation. Provided all the preconditions of the feature in question have been met by the calling object, the computation must succeed and be itself valid.

Now suppose we would omit checking the invariant before returning from a qualified call.

You can now write a program that correctly  performs some computation on an object denoted by `x` but leaves that object in an inconsistent state. Since the computation may still be computed correctly and its effects may be validated by post-conditions as usual, there is no problem.

However, say you call that feature on `x` at 100 different places in your program and when you place one more call to the program, it will abort due to a class invariant violation, you may not readily identify the last call prior to the 101st call that left the class invariant violated. As such, it's more difficult to track down where your program has actually failed.

Wikipedia [writes on the matter](http://en.wikipedia.org/wiki/Fail-fast):

> Finding the cause of a failure is easier in a fail-fast system, because the system reports the failure with as much information as possible as close to the time of failure as possible. In a fault-tolerant system, the failure might go undetected, whereas in a system that is neither fault-tolerant nor fail-fast the failure might be temporarily hidden until it causes some seemingly unrelated problem later.

This is one<sup>3</sup> reason why optimizing it is not very practical.

Summary
=======

The class invariant helps you maintain a consistent object for whenever some object could want to access any of its features, while still giving you the flexibility to temporarily violate the class invariant in your own object (i.e. of the class you're programming). Maintaining sight of where and how you invalidate your class invariant is much easier, when you don't have to consider the entire program, but only a very small part.

You still have the possibility, however, to force checking the class invariant in your object by explicitly prefixing a feature call with `Current.`

<sup>1</sup>The above article uses plural and singular of invariants interchangeably. Technically, Eiffel doesn't need to support having multiple invariants per class, because multiple constraints may just be combined into one invariant by using `and`. However, by using multiple different invariants, Eiffel can easily provide feedback on which of the different possibly unrelated invariants has failed instead of only alerting you that the entire invariant has failed.

<sup>2</sup>This addresses the second bullet point on how the question could have been interpreted differently. One could have taken it to mean, that invariants of other objects (even from other classes) are being checked on calls.

<sup>3</sup>Checking the class invariant when returning from a qualified call also protects from ending up in an unexpectedly inconsistent state in mutual recursion.

