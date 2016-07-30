---
layout: post
title: "A story of functors"
date: 2016-06-14 17:12:00 +0000
comments: true
categories: 
---

Sannie, Pietie, and Gawie are working on a project together. 

Pietie's code produces a list of _Person_ objects. 

{% highlight go %}
type Person struct {
	first_name string
	last_name string
	full_name string
}

type People []Person

func FindPeople(...) (People, error) { ... }

{% endhighlight %}

Sannie has written code that takes a single _Person_ and combines the _first_name_ and _last_name_ fields into a _full_name_ field.

{% highlight go %}
func BuildFullName(person Person) Person { ... }
{% endhighlight %}

Gawie has to glue these two together so he writes a function that takes a list of _Person_ objects and returns a new list of _Person_ objects 
where each entry has had Sannie's transformation applied.

{% highlight go %}
func PeopleWithFullNames(in People) People {
	var people People
	for _, person := range in {
		people = append(people, BuildFullName(person))
	}
	return people
}
{% endhighlight %}

Soon enough Pietie adds a new function to his API. This function - _MitochondrialTree_ - returns a tree of _Person_ objects..

{% highlight go %}
type PersonTree struct {
	person Person
	children []Person
}

func MitochondrialTree(...) (PersonTree, error) { ... }
{% endhighlight %}

Now Gawie has to build an new function that can apply Sannie's _BuildFullName_ function to all the _Person_ objects in the _PersonTree_.

That's pretty annoying as he's already moved on to other stuff. Instead, Gawie asks Pietie to write a function that traverses the _PersonTree_
and can apply a given transformation to every _Person_ in the tree and return a new _PersonTree_ where every _Person_ has been updated according
to that transformation.

{% highlight go %}
func (pt PersonTree) Walk(fn func(Person) Person) PersonTree { ... }
{% endhighlight %}

Gawie now calls that instead.

{% highlight go %}
tree, err := MitochondrialTree(...)
if err != nil {
	panic(err)
}
withFullNames := tree.Walk(BuildFullName)
{% endhighlight %}

That is way cooler. Oh my word, just spare a minute to appreciate how much more fun that is.

Eagerly adding functionality to the data he presents Pietie soon adds a function that takes an arbitrary list of _Person_ objects and splits the list into disjoint mitochondrial trees.

{% highlight go %}
type PersonTrees []PersonTree

func PeopleToFamilies(people People) PersonTrees { ... }
{% endhighlight %}

Sticking to the established pattern he adds a function that traverses this structure:

{% highlight go %}
func (trees PersonTrees) Walk(fn func(Person) Person) PersonTrees { ... }
{% endhighlight %}

Gawie dutifully adds support for this new structure

{% highlight go %}
people, err := FindPeople(...)
if err != nil {
	panic(err)
}
trees, err := PeopleToFamilies(people)
if err != nil {
	panic(err)
}
withFullNames := trees.Walk(trees, BuildFullName)
{% endhighlight %}

That was very little extra work - in fact Gawie didn't mind at all as it was barely a context switch. Even the existing tests could just be copied and pasted.

Sannie has been delving into functional programming and notices that there seems to be an abstract pattern here.

She notices that Gawie gets a bunch of _Person_ objects, then calls _Walk()_ on that collection, passing her _BuildFullName()_ function to it and gets back a container with the exact same structure but with the individual entries updated.

Aiming at a beautiful refactor, she implements _Walk()_ for the _People_ type, too:

{% highlight go %}
type People []Person

func (ps People) Walk(fn func(Person) Person) People {
{% endhighlight %}

Let's see where we stand with the _Walk()_ implementations

{% highlight go %}
type PersonTransform func(person Person) Person

func (ps People)         Walk(fn PersonTransform) People { ... }
func (tree PersonTree)   Walk(fn PersonTransform) PersonTree { ... }
func (trees PersonTrees) Walk(fn PersonTransform) PersonTrees { ... }
{% endhighlight %}

Now - since this is the future and golang in that wonderful future supports subtyping with the <T> parameter replaced by the receiver's type she defines the golden interface:

{% highlight go %}
type PersonWalker interface {
	Walk(fn PersonTransform) <T>
}
{% endhighlight %}

Notice that all the different structures implemenet the _PersonWalker_ interface.

Armed with this new interface Gawie can refactor his code to accept any structure Pietie dreams up, apply any transformation to the _Person_ objects in that
structure and return the same structure (regardless of what it was) with the _Person_ entries appropriately modified.

This means he writes one set of tests and never has to come back to this part of the project. Pietie can go wild and derive fantastic new structures of _Person_ objects,
perhaps he returns a list of disjoint social graphs from a list of people returned from _FindPerson()_. As long as Pietie implements _Walk()_ for his new
structure no code anywhere else in the project needs to change.

This is a super powerful way of working with structures of _Person_ objects. There is still one part of the _PersonWalker_ interface 
that feels like it is holding us back though. The _PersonTransform_ function seems redundant. I mean, all the structures consist of _Person_ objects, and _PersonTransform_
operates on a _Person_ object and returns a _Person_ object. We should be able to refactor this even more and have something like this:

{% highlight go %}
type Walker interface {
	Walk(fn func(<T>) <T>) AnyStructureConsistingOf<T>s
}
{% endhighlight %}

With such a general interface we suddenly separate the idea of a _PersonTree_ into two ideas: a _Person_ and a _Tree_. And a list of _PersonTree_ gets split into
_Person_ and _[]Tree_. But _Tree_ and _[]Tree_ are super general structures - so if Sannie helps Pietie implement _Walk()_ for a general tree they can use this pattern on trees of _Person_, of _Employee_, of _Dynasty_, of anything!

Specifically we end up with

{% highlight go %}
type PersonTree struct {
	node Person
	children []Person
}
func (pt PersonTree) Walk(fn func(Person) Person) PersonTree { ... }

// now becoming

type <T>Tree struct {
	node T
	children []T
}
func (t <T>Tree) Walk(fn func(<T>) <T>) <T>Tree { ... }

// and we define a PersonTree simply as
type PersonTree <Person>Tree

{% endhighlight %}

Where you can now replace the < T > with *any type* to get a _Tree_ of that type.

The same generalization can be made for a _slice_ or _array_, a _slice of trees_, a _graph_, or literally any container of anything. As long as that structure implements the _Walk()_ method we can use it to give structure to an otherwise unconnected list of objects.

To cruise along with the cool kids Sannie renames her _Walk()_ functions to _Fmap()_. She also gets points at the watercooler for mentioning that any type that implements _Fmap()_ is called a _Functor_. There are some nervous nods and some feet shuffling at the mention of _Functor_ so Sannie pushes ahead, trying to explain that really, on
a more abstract level, all this is really simple. In fact, Gawie's code really just takes a _Functor_ and _fmap_'s a transformation over it.

For no discernable reason Sannie is soon alone at the watercooler. Did one of her team mates hang around just a bit longer than the others? Maybe she should tell that dude about _fold_'s, they're super neat.

UPDATE (30-07-2016): I've learned more about Functors! You might be confused by the assertion that any type that implements 'Fmap' is called a Functor. This isn't quite true. I mean, I can implement Fmap and have it just print something to the screen. The Fmap function needs to satisfy two very specific properties (which our examples all do.) This first is that if you call Fmap with the identity function it should return the unmodified original structure:

{% highlight go %}

func (pt PersonTree) Fmap(fn func(Person) Person) PersonTree {
     return pt.Walk(fn)
}

func identity(person Person) Person {
    return person
}

reflect.DeepEqual(tree, tree.Fmap(identity))
#=> true
{% endhighlight %}

The second is that it should 'preserve structure' which means Fmap of two nested functions should be the same as two Fmap operations. Confusing as crap until you see the example:

{% highlight go %}

func upcase(person Person) Person {
     person.first_name = strings.ToUpper(person.first_name)
     person.last_name = strings.ToUpper(person.last_name)
     person.full_name = strings.ToUpper(person.full_name)
     return person
}

func l33tify(person Person) Person {
     person.first_name = l33tifyStr(person.first_name)
     person.last_name = l33tifyStr(person.last_name)
     person.full_name = l33tifyStr(person.full_name)
     return person
}

func l33tifyStr(str string) string {
     l33ts := map[string]string{
     	   "e": "3",
     	   "E": "3",
     	   "a": "4",
     	   "A": "4",
     }
     for before, after := range l33ts {
     	 str = strings.Replace(str, before, after, -1)
     }
     return str
}

func l33tifyThenUpcase(person Person) Person {
     return upcase(l33tify(person))
}

reflect.DeepEqual(
	tree.Fmap(l33tifyThenUpcase),
	tree.Fmap(l33tify).Fmap(upcase)
)
#=> true
{% endhighlight %}

Now - I may still be wrong as I'm not yet strong with the Haskells - but I'm pretty sure this is at least *more* correct.

The cool thing about all the maths behind this is that it works exactly the way we hoped it would. It is so intuitive that (similar to our starting examples) you just build the thing the way you would anyway and it turns out that it's a functor. I think about these mathematical properties as guard rails: some clever dude or dudette did the mental heavy-lifting and proved that these two properties were sufficient to completely define this amazing class of transformations called Functors.

Without these properties stated so formally, you'd never *really* know that it works the way you think it does, you'd just have a good feeling about it and maybe smile at the obvious elegance of your abstraction. Refactoring would be stressful as you'd never know whether stuff like changing the order of evaluation of the elements in your structure might screw things up, or whether some case left out of a switch statement could leave an entire class of structures broken after the transformation. 

Since you've broken the whole idea down into two very simple properties you can use a property-based testing tool like QuickCheck to test millions of different instances of your structure and make sure that these properties always hold even through major refactoring.

Don't miss out on the certainty and peace of mind that the formal reasoning gives you - with which you can refactor fearlessly as your tests will complain if you stop satisfying the required properties.

To learning!