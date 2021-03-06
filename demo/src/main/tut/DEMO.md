## Unzipping Immutability

This page contains the materials for my talk “Unzipping Immutability”.
Here are some past and future presentations:

* [LX Scala, April 2016](http://www.lxscala.com/schedule/#session-2) ([video](https://vimeo.com/162214356)).
* [Pixels Camp, October 2016](https://github.com/PixelsCamp/talks/blob/master/unzipping-immutability_nick-stanchenko.md) ([video](https://www.youtube.com/watch?v=yeMvhuD689A)).
* [Scala By The Bay, November 2016](http://sched.co/7iTv).

You can use this page in two ways:

* as a reference/refresher on the concepts covered in the talk;
* as an interactive playground where you can try the same commands I presented.

Here is an overview:

* [Immutable data structures](#immutable-data-structures)
* [Lenses](#lenses)
* [Zippers](#zippers)
* [Useful resources](#useful-resources)

Throughout this page we will assume the following
declarations (each section might add its own):

```tut:silent
import reftree._
import reftree.demo.Data._
import scala.collection.immutable._
import scala.concurrent.duration.DurationInt
import java.nio.file.Paths
```

To start an interactive session, just run

```
$ sbt demo
@ render(List(1, 2, 3))
```

and open `diagram.png` in your favorite image viewer (hopefully one that
reloads images automatically on file change). You will also need to have
[GraphViz](http://www.graphviz.org/) installed. *The interactive session
already has all the necessary imports in scope.*

### Immutable data structures

```tut:silent
// extra declarations for this section
val diagram = Diagram(
  defaultOptions = Diagram.Options(density = 100),
  defaultDirectory = Paths.get("images", "data")
)
```

#### Lists

We’ll start with one of the simplest structures: a list.
It consists of a number of cells pointing to each other:

```tut
val list = List(1, 2, 3)
```

```tut:silent
diagram.render("list")(list)
```

<p align="center"><img src="images/data/list.png" width="20%" /></p>

Elements can be added to or removed from the front of the list with no effort,
because we can share the same cells across several lists.
This would not be possible with a mutable list,
since modifying the shared part would modify every data structure making use of it.

```tut
val add = 0 :: list
val remove = list.tail
```

```tut:silent
diagram.render("lists")(list, add, remove)
```

<p align="center"><img src="images/data/lists.png" width="20%" /></p>

However we can’t easily add elements at the end of the list, since the last cell
is pointing to the empty list (`Nil`) and is immutable, i.e. cannot be changed.
Thus we are forced to create a new list every time:

```tut:silent
diagram.renderAnimation(
  "list-append",
  tweakOptions = _.copy(onionSkinLayers = 3))(
  Utils.iterate(List(1))(_ :+ 2, _ :+ 3, _ :+ 4)
)
```

<p align="center"><img src="images/data/list-append.gif" width="40%" /></p>

This certainly does not look efficient compared to adding elements at the front:

```tut:silent
diagram.renderAnimation(
  "list-prepend",
  tweakOptions = _.copy(onionSkinLayers = 1, diffAccent = true))(
  Utils.iterate(List(1))(2 :: _, 3 :: _, 4 :: _)
)
```

<p align="center"><img src="images/data/list-prepend.gif" width="20%" /></p>

#### Queues

If we want to add elements on both sides efficiently, we need a different data structure: a queue.
The queue below, also known as a “Banker’s Queue”, has two lists: one for prepending and one for appending.

```tut
val queue1 = Queue(1, 2, 3)
val queue2 = (queue1 :+ 4).tail
```

```tut:silent
diagram.render("queues", tweakOptions = _.copy(verticalSpacing = 1.2))(queue1, queue2)
```

<p align="center"><img src="images/data/queues.png" width="40%" /></p>

This way we can add and remove elements very easily at both ends.
Except when we try to remove an element and the respective list is empty!
In this case the queue will rotate the other list to make use of its elements.
Although this operation is expensive, the usage pattern intended for a queue
makes it rare enough to yield great average (“ammortized”) performance:

```tut:silent
def add(n: Int)(q: Queue[Int]) = Utils.iterate(q, n)(q => q :+ (q.max + 1)).tail
def remove(n: Int)(q: Queue[Int]) = Utils.iterate(q, n)(q => q.tail).tail
def addRemove(n: Int)(q: Queue[Int]) = Utils.flatIterate(q)(add(n), remove(n)).tail

val queues = Utils.flatIterate(Queue(1, 2, 3), 3)(addRemove(2))

diagram.renderAnimation(
  "queue",
  tweakOptions = _.copy(diffAccent = true))(
  queues
)
```

<p align="center"><img src="images/data/queue.gif" width="40%" /></p>

#### Vectors

One downside common to both lists and queues we saw before is that to get an element by index,
we need to potentially traverse the whole structure. A `Vector` is a powerful data structure
addressing this shortcoming and available in Scala (among other languages, like Clojure).

Internally vectors utilize up to 6 layers of arrays, where 32 elements sit on the first layer,
1024 — on the second, 32^3 — on the third, etc.
Therefore getting any element by its index requires at most 6 pointer dereferences,
which can be deemed constant time (yes, the trick is that the number of elements that can
be stored is limited by 2^31).

The internal 32-element arrays form the basic structural sharing blocks.
For small vectors they will be recreated on most operations:

```tut
val vector1 = (1 to 20).toVector
val vector2 = vector1 :+ 21
```

```tut:silent
 diagram.render("vectors", tweakOptions = _.copy(verticalSpacing = 2))(vector1, vector2)
```

<p align="center"><img src="images/data/vectors.png" width="100%" /></p>

However as more layers leap into action, a huge chunk of the data can be shared:

```tut
val vector1 = (1 to 100).toVector
val vector2 = vector1 :+ 21
```

```tut:silent
 diagram.render("big-vectors", tweakOptions = _.copy(verticalSpacing = 2))(vector1, vector2)
```

<p align="center"><img src="images/data/big-vectors.png" width="100%" /></p>

If you want to know more, this structure is covered in great detail by Jean Niklas L’orange
[in his blog](http://hypirion.com/musings/understanding-persistent-vector-pt-1).
I also highly recommend watching [this talk](https://www.youtube.com/watch?v=pNhBQJN44YQ) by Daniel Spiewak.

#### Finger Trees

To conclude this section, I would like to share a slightly less popular, but beautifully designed
data structure called “finger tree” described in [this paper](http://www.cs.ox.ac.uk/ralf.hinze/publications/FingerTrees.pdf)
by Hinze and Paterson. Enjoy the read and this animation of a finger tree getting filled with some numbers:

```tut:silent
import de.sciss.fingertree.{FingerTree, Measure}
import reftree.contrib.FingerTreeInstances._

implicit val measure = Measure.Indexed

val fingerTrees = Utils.iterate(FingerTree(1), 21)(t ⇒ t :+ (t.measure + 1))

diagram.renderAnimation(
  "finger",
  tweakOptions = _.copy(diffAccent = true, verticalSpacing = 2, density = 75))(
  fingerTrees
)
```

<p align="center"><img src="images/data/finger.gif" width="100%" /></p>

### Lenses

So far we were looking into “standard” data structures,
but in our code we often have to deal with custom data structures comprising our domain model.
Updating this sort of data can be tricky if it’s immutable.
For case classes Scala gives us the `copy` method:

```scala
case class Employee(
  name: String,
  salary: Long
)
```

```tut
employee
val raisedEmployee = employee.copy(salary = employee.salary + 10)
```

However once composition comes into play, the resulting nested immutable data structures
would require a lot of `copy` calls:

```scala
case class Employee(
  name: String,
  salary: Long
)

case class Startup(
  name: String,
  founder: Employee,
  team: List[Employee]
)
```

```tut
startup
val raisedFounder = startup.copy(
  founder = startup.founder.copy(
    salary = startup.founder.salary + 10
  )
)
```

```tut:silent
// extra declarations for this section
import reftree.contrib.SimplifiedInstances.list
import reftree.contrib.LensInstances._

val diagram = Diagram(
  defaultOptions = Diagram.Options(density = 100),
  defaultDirectory = Paths.get("images", "lenses")
)
```

```tut:silent
diagram.render("startup")(startup, raisedFounder)
```

<p align="center"><img src="images/lenses/startup.png" width="100%" /></p>

Ouch!

A common solution to this problem is a “lens”.
In the simplest case a lens is a pair of functions to get and set a value of type `B` inside a value of type `A`.
It’s called a lens because it focuses on some part of the data and allows to update it.
For example, here is a lens that focuses on an employee’s salary
(using the excellent [Monocle library](https://github.com/julien-truffaut/Monocle)):

```tut
import monocle.macros.GenLens

val salaryLens = GenLens[Employee](_.salary)

salaryLens.get(startup.founder)
salaryLens.modify(s => s + 10)(startup.founder)
```

```tut:silent
diagram.render("salaryLens")(LensFocus(salaryLens, startup.founder))
```

<p align="center"><img src="images/lenses/salaryLens.png" width="40%" /></p>

We can also define a lens that focuses on the startup’s founder:

```tut
val founderLens = GenLens[Startup](_.founder)

founderLens.get(startup)
```

```tut:silent
diagram.render("founderLens")(LensFocus(founderLens, startup))
```

<p align="center"><img src="images/lenses/founderLens.png" width="100%" /></p>

It’s not apparent yet how this would help, but the trick is that lenses can be composed:

```tut
val founderSalaryLens = founderLens composeLens salaryLens

founderSalaryLens.get(startup)
founderSalaryLens.modify(s => s + 10)(startup)
```

```tut:silent
diagram.render("founderSalaryLens")(LensFocus(founderSalaryLens, startup))
```

<p align="center"><img src="images/lenses/founderSalaryLens.png" width="100%" /></p>

One interesting thing is that lenses can focus on anything, not just direct attributes of the data.
Here is a traversal — a more generic kind of lens — that focuses on all vowels in a string:

```tut:silent
diagram.render("vowelTraversal")(LensFocus(vowelTraversal, "example"))
```

<p align="center"><img src="images/lenses/vowelTraversal.png" width="40%" /></p>

We can use it to give our founder a funny name:

```tut
val employeeNameLens = GenLens[Employee](_.name)
val founderVowelTraversal = founderLens composeLens employeeNameLens composeTraversal vowelTraversal

founderVowelTraversal.modify(v => v.toUpper)(startup)
```

```tut:silent
diagram.render("founderVowelTraversal")(LensFocus(founderVowelTraversal, startup))
```

<p align="center"><img src="images/lenses/founderVowelTraversal.png" width="100%" /></p>

So far we have replaced the `copy` boilerplate with a number of lens declarations.
However most of the time our goal is just to update data.

In Scala there is a great library called [quicklens](https://github.com/adamw/quicklens)
that allows to do exactly that, creating all the necessary lenses under the hood:

```tut
import com.softwaremill.quicklens._

val raisedCeo = startup.modify(_.founder.salary).using(s => s + 10)
```

You might think this is approaching the syntax for updating mutable data,
but actually we have already surpassed it, since lenses are much more flexible:


```tut
val raisedEveryone = startup.modifyAll(_.founder.salary, _.team.each.salary).using(s => s + 10)
```


### Zippers

In our domain models we are often faced with recursive data structures.
Consider this example:

```scala
case class Employee(
  name: String,
  salary: Long
)

case class Hierarchy(
  employee: Employee,
  team: List[Hierarchy]
)

case class Company(
  name: String,
  hierarchy: Hierarchy
)
```

The `Hierarchy` class refers to itself.
Let’s grab a company object and display its hierarchy as a tree:

```tut:silent
// extra declarations for this section
import zipper._
import reftree.contrib.SimplifiedInstances.option
import reftree.contrib.ZipperInstances._

val diagram = Diagram(
  defaultOptions = Diagram.Options(density = 100),
  defaultDirectory = Paths.get("images", "zippers")
)
```

```tut:silent
diagram.render("company")(company.hierarchy)
```

<p align="center"><img src="images/zippers/company.png" width="100%" /></p>

What if we want to navigate through this tree and modify it along the way?
We can use [lenses](#lenses), but the recursive nature of the tree allows for a better solution.

This solution is called a “Zipper”, and was introduced by Gérard Huet in 1997.
It consists of a “cursor” pointing to a location anywhere in the tree — “current focus”.
The cursor can be moved freely with operations like `moveDownLeft`, `moveRight`, `moveUp`, etc.
Current focus can be updated, deleted, or new nodes can be inserted to its left or right.
Zippers are immutable, and every operation returns a new Zipper.
All the changes made to the tree can be committed, yielding a new modified version of the original tree.

Here is how we would insert a new employee into the hierarchy:

```tut:silent
val updatedHierarchy = Zipper(company.hierarchy).moveDownRight.moveDownRight.insertRight(newHire).commit
```

```tut:silent
diagram.render("updatedHierarchy")(company.hierarchy, updatedHierarchy)
```

<p align="center"><img src="images/zippers/updatedHierarchy.png" width="100%" /></p>

My [zipper library](https://github.com/stanch/zipper#zipper--an-implementation-of-huets-zipper)
provides a few useful movements and operations.

Let’s consider a simpler recursive data structure:

```scala
case class Tree(x: Int, c: List[Tree] = List.empty)
```

and a simple tree:

```tut
simpleTree
```

```tut:silent
diagram.render("simpleTree")(simpleTree)
```

<p align="center"><img src="images/zippers/simpleTree.png" width="50%" /></p>

When we wrap a Zipper around this tree, it does not look very interesting yet:

```tut:silent
val zipper1 = Zipper(simpleTree)
```

```tut:silent
diagram.render("zipper1")(simpleTree, zipper1)
```

<p align="center"><img src="images/zippers/zipper1.png" width="50%" /></p>

We can see that it just points to the original tree and has some other empty fields.
More specifically, a Zipper consists of four pointers:

```scala
case class Zipper[A](
  left: List[A],           // left siblings of the focus
  focus: A,                // the current focus
  right: List[A],          // right siblings of the focus
  top: Option[Zipper[A]]   // the parent zipper
)
```

In this case the focus is the root of the tree, which has no siblings,
and the parent zipper does not exist, since we are at the top level.

One thing we can do right away is modify the focus:

```tut:silent
val zipper2 = zipper1.update(focus ⇒ focus.copy(x = focus.x + 99))
```

```tut:silent
diagram.render("zipper2")(simpleTree, zipper1, zipper2)
```

<p align="center"><img src="images/zippers/zipper2.png" width="50%" /></p>

We just created a new tree! To obtain it, we have to commit the changes:

```tut:silent
val tree2 = zipper2.commit
```

```tut:silent
diagram.render("tree2")(simpleTree, tree2)
```

<p align="center"><img src="images/zippers/tree2.png" width="50%" /></p>

If you were following closely,
you would notice that nothing spectacular happened yet:
we could’ve easily obtained the same result by modifying the tree directly:

```tut:silent
val tree2b = simpleTree.copy(x = simpleTree.x + 99)

assert(tree2b == tree2)
```

The power of Zipper becomes apparent when we go one or more levels deep.
To move down the tree, we “unzip” it, separating the child nodes into
the focused node and its left and right siblings:

```tut:silent
val zipper2 = zipper1.moveDownLeft
```

```tut:silent
diagram.render("zipper1+2")(zipper1, zipper2)
```

<p align="center"><img src="images/zippers/zipper1+2.png" width="50%" /></p>

The new Zipper links to the old one,
which will allow us to return to the root of the tree when we are done applying changes.
This link however prevents us from seeing the picture clearly.
Let’s elide the parent field:

```tut:silent
import reftree.contrib.SimplifiedInstances.zipper
```

```tut:silent
diagram.render("zipper2b")(zipper2)
```

<p align="center"><img src="images/zippers/zipper2b.png" width="50%" /></p>

Great! We have `2` in focus and `3, 4, 5` as right siblings. What happens if we move right a bit?

```tut:silent
val zipper3 = zipper2.moveRightBy(2)
```

```tut:silent
diagram.render("zipper3")(zipper3)
```

<p align="center"><img src="images/zippers/zipper3.png" width="50%" /></p>

This is interesting! Notice that the left siblings are “inverted”.
This allows to move left and right in constant time, because the sibling
adjacent to the focus is always at the head of the list.

This also allows us to insert new siblings easily:

```tut:silent
val zipper4 = zipper3.insertLeft(Tree(34))
```

```tut:silent
diagram.render("zipper4")(zipper4)
```

<p align="center"><img src="images/zippers/zipper4.png" width="50%" /></p>

And, as you might know, we can delete nodes and update the focus:

```tut:silent
val zipper5 = zipper4.deleteAndMoveRight.set(Tree(45))
```

```tut:silent
diagram.render("zipper5")(zipper5)
```

<p align="center"><img src="images/zippers/zipper5.png" width="50%" /></p>

Finally, when we move up, the siblings at the current level are “zipped”
together and their parent node is updated:

```tut:silent
val zipper6 = zipper5.moveUp
```

```tut:silent
diagram.render("zipper6")(zipper6)
```

<p align="center"><img src="images/zippers/zipper6.png" width="50%" /></p>

You can probably guess by now that `.commit` is a shorthand for going
all the way up (applying all the changes) and returning the focus:

```tut:silent
val tree3a = zipper5.moveUp.focus
val tree3b = zipper5.commit

assert(tree3a == tree3b)
```

Here is an animation of the navigation process:

```tut:silent
val zippers = Utils.iterate(Zipper(simpleTree))(
  _.moveDownLeft,
  _.moveRight, _.moveRight, _.moveRight,
  _.moveDownLeft,
  _.moveRight, _.moveLeft,
  _.top.get,
  _.moveLeft, _.moveLeft, _.moveLeft,
  _.top.get
)

diagram.renderAnimation("navigation-tree")(
  zippers.map(ZipperFocus(_, simpleTree))
)

diagram.renderAnimation("navigation-zipper")(
  zippers
)
```

<p align="center">
  <img src="images/zippers/navigation-tree.gif" width="42%" />
  <img src="images/zippers/navigation-zipper.gif" width="52%" />
</p>

### Useful resources

#### Books, papers and talks

* [Purely functional data structures](http://www.amazon.com/Purely-Functional-Structures-Chris-Okasaki/dp/0521663504) by Chris Okasaki,
  and/or [his PhD thesis](https://www.cs.cmu.edu/~rwh/theses/okasaki.pdf) — *the* introduction to immutable data structures
* [What’s new in purely functional data structures since Okasaki](http://cstheory.stackexchange.com/a/1550) — an excellent StackExchange answer
  with pointers for further reading
* [Extreme cleverness](https://www.youtube.com/watch?v=pNhBQJN44YQ) by Daniel Spiewak — a superb talk
  covering several immutable data structures (implemented [here](https://github.com/djspiewak/extreme-cleverness))
* [Understanding Clojure’s Persistent Vectors, part 1](http://hypirion.com/musings/understanding-persistent-vector-pt-1)
  and [part 2](http://hypirion.com/musings/understanding-persistent-vector-pt-2) — a series of blog posts by Jean Niklas L’orange
* [Finger Trees](http://www.cs.ox.ac.uk/ralf.hinze/publications/FingerTrees.pdf) and
  [1-2 Brother Trees](http://www.cs.ox.ac.uk/ralf.hinze/publications/Brother12.pdf) described by Hinze and Paterson
* [Huet’s original Zipper paper](https://www.st.cs.uni-saarland.de/edu/seminare/2005/advanced-fp/docs/huet-zipper.pdf) — a great short read
  introducing the Zipper
* [Weaving a web](http://dspace.library.uu.nl/bitstream/handle/1874/2532/2001-33.pdf) by Hinze and Jeuring —
  another interesting Zipper-like approach

#### Scala libraries

* [zipper](https://github.com/stanch/zipper) — my Zipper implementation
* [Monocle](https://github.com/julien-truffaut/Monocle) — an “optics” library
* [Quicklens](https://github.com/adamw/quicklens) — a simpler way to update nested case classes
* [FingerTree](https://github.com/Sciss/FingerTree) — an implementation of the Finger Tree data structure
