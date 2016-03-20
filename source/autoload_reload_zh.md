自动加载和重新加载常量
===================================

本篇介绍Rails中自动加载和重新加载常量是如何工作的.

读完本文，你将学到：

* Ruby常量的几个主要方面
* 什么是 `autoload_paths`
* 自动加载常量是如何工作的
* 什么是 `require_dependency`
* 重新加载常量是如何工作的
* 常见自动加载常量的陷阱的解决方案

--------------------------------------------------------------------------------


简介
------------

在正常的Ruby程序中类在使用前必须先加载相关的依赖

```ruby
require 'application_controller'
require 'post'

class PostsController < ApplicationController
  def index
    @posts = Post.all
  end
end
```

大多数Rubyist会发现上面的代码require是多余的：如果类在定义的时候和该文件名所匹配
，为什么还要加载一次，这个问题很容易解决，我们可以查看对应的文件。

此外，`Kernel#require`会加载一次文件，想要在不用重启服务的情况下
看到代码的更新，使开发更顺畅，在开发环境下使用`Kernel#load`,在生产环境中使用`Kernel#require`.

事实上，Ruby on Rails提供了这些特性，我们只需要这样写：

```ruby
class PostsController < ApplicationController
  def index
    @posts = Post.all
  end
end
```

本篇将介绍它是如何工作的


常量进阶
-------------------

虽然常量在其他编程语言中很常见，但在Ruby中是常量一个内容很丰富的话题.

这超出了本文的讨论范围，但我们仍然要突出的介绍几个关键主题。
准确把握以下部分有助于了解autoloading和reload。

### 嵌套

Class和module的嵌套可以用来创建命名空间：

```ruby
module XML
  class SAXParser
    # (1)
  end
end
```

嵌套在任何给定的地方是封闭嵌套类的结合或者module对象，
通过Module.nesting来查看嵌套的层级关系，举个例子，在上一个例子中
嵌套（1）如下：

```ruby
[XML::SAXParser, XML]
```

嵌套是由类和module对象组成，和访问他们的常量无关，与他们的名字也没有关系，理解
这一点非常重要

举个实例，下面的定义与上一个类似：

```ruby
class XML::SAXParser
  # (2)
end
```

但是嵌套（2）却不同：

```ruby
[XML::SAXParser]
```

`XML` 并不在这个嵌套中.

我们可以看到，在这个例子中类和模块的名字属于特定的嵌套，与命名空间没有关系.
甚至，他们是完全独立的，例如：

```ruby
module X
  module Y
  end
end
 
module A
  module B
  end
end
 
module X::Y
  module A::B
    # (3)
  end
end
```

嵌套(3)包含了A::B, X::Y两个module对象:

```ruby
[A::B, X::Y]
```

因此，它不但没有在A处结束，而且A也不在这个嵌套中，但是这个嵌套包含`X::Y`,
并且与`A::B`相互独立.

嵌套是由Ruby解释器内部堆栈进行维护的，根据以下规则进行修改:

* 当程序执行到一个类的body时，`class`关键字后面的类对象入栈，执行完后出栈.

* 当程序执行到一个模块的body时，`module`关键字后面的模块对象入栈，执行完后出栈.

* `class << object`打开单例类时，object入栈，end结束后出栈.

* 当instance_eval被字符串调用的时候，接受者的单例被加入到被eval的代码的嵌套中.

* 顶层作用域的嵌套由`Kernel#load`解释，默认为空，如果`load`方法接受被调用，并
且第二个参数为true，那么一个匿名的module会被创建并添加到嵌套中.

我们发现blocks并不会修改nesting堆栈, blocks传递给`Class.new` 和 `Module.new`
并不会重新定义class和module, 因此也不会改变堆栈信息.


### 类和模块的定义决定常量的分配

我们假设下面的代码片段来创建一个类:

```ruby
class C
end
```

Ruby创建了一个常量`C`在`Object`对象中，并存储在常量类对象中，
并给这个实例的设置一个名字为"C"（Object.const_get("C")）


更确切的说,

```ruby
class Project < ActiveRecord::Base
end
```

等同于常量的赋值

```ruby
Project = Class.new(ActiveRecord::Base)
```

同时也给常量设置了一个名字:

```ruby
Project.name # => "Project"
```

常量的赋值有一个特殊的规则，当这个指定的对象是一个匿名类或者模块，Ruby会
把常量的名字设置为这个对象的名字.


INFO. 现在不管这个常量和这个对象发生了什么，例如，常量可以被删除，
类对象可以被分配到一个不同的常量，或者没有分配常量等等，一旦名字被设置，它不改变。

同样，模块的创建使用关键字`module`

```ruby
module Admin
end
```

等同于常量的赋值

```ruby
Admin = Module.new
```

同时也给常量设置了一个名字:

```ruby
Admin.name # => "Admin"
```

WARNING. The execution context of a block passed to `Class.new` or `Module.new`
is not entirely equivalent to the one of the body of the definitions using the
`class` and `module` keywords. But both idioms result in the same constant
assignment.

Thus, when one informally says "the `String` class", that really means: the
class object stored in the constant called "String" in the class object stored
in the `Object` constant. `String` is otherwise an ordinary Ruby constant and
everything related to constants such as resolution algorithms applies to it.

同样的，在Controller中

```ruby
class PostsController < ApplicationController
  def index
    @posts = Post.all
  end
end
```

`Post`不是一个标准类的写法，而是一个普通的Ruby常量，如果Post.all调用正常
说明常量等同于对象，可以相应`all`.

这就是为什么我们要讨论的常量的自动加载，Rails可以在运行时加载常量.

### 常量存储在Modules中

字面意义上讲，常量属于模块. 类和模块有一个常量表，可以想象成一个hash表.

让我们来通过一个例子来理解这个概念，我们知道在大多数语言中String类使用起来非常方便
，下面我们会详细的阐述.

我们先来看下面这个模块的定义:

```ruby
module Colors
  RED = '0xff0000'
end
```

首先，当解释器执行到`module`关键字的时候，会在Object类对象的常量表中创建一个实体，
并存储在`Object`中."Colors"会关联到这个新创建的module对象，
此外解释器会把这个module对象的名字设置为"Colors".

然后，当解释器执行到module体的时候，会在模块对象(Colors)的常量表中创建一个实体，
并存储在`Object`中. 这个实体会从"RED"映射到字符串"0xff0000".

需要注意的是， `Colors::RED`与其他`RED`常量毫无关系，可以存在与其他类或者模块当中.
如果有，他们会在各自的常量表中讲其拆分.

在前几段中特别注意，在常量表中，类和模块对象、常量名称和值对象之间的区别。

### 解析算法

#### Relative Constants解析算法

在代码的任意给定位置，定义一个非空嵌套， *cref* 为第一个元素

忽略一些细节，解析算法大概是这样的:

1. 如果嵌套不为空，按嵌套的元素顺序依次查找，并且忽略这些元素的祖先.

2. 如果没有找到， 则去cref的祖先链中查找.

3. 如果还是没有找到，`const_missing`方法被触发，默认会抛出`NameError`错误，
   但是这个方法可以覆盖.

Rails的自动加载与这个算法不同，但是出发点是常量的名字会自动加载，更多可查看
[Relative References](#autoloading-algorithms-relative-references).

#### Qualified Constants解析算法

Qualified constants是这样的结构:

```ruby
Billing::Invoice
```

`Billing::Invoice`由两个常量组成，`Billing`是Relative Constants，其解析算法
使用Relative Constants解析算法.

INFO. 最前面的冒号使第一部分是绝对的，而不是相对的，比如`::Billing::Invoice`
会只在顶层作用域中查找`Billing`

另外`Billing`决定了`Invoice`的查找路径，这种结构为有父子关系的嵌套,它的解析过程
大概是这样的:

1. 首先，在其父和祖先链中查找

2. 如果没有找到，`const_missing`方法被触发，默认会抛出`NameError`错误，
   但是这个方法可以覆盖.

这个解析算法比相对常量的查找简单. 注意，
As you see, this algorithm is simpler than the one for relative constants. In
particular, the nesting plays no role here, and modules are not special-cased,
if neither they nor their ancestors have the constants, `Object` is **not**
checked.

Rails的自动加载与这个算法不同，但是出发点是常量的名字会自动加载，更多可查看
[Qualified References](#qualified-references).

名词解释
----------

### Parent Namespaces

Given a string with a constant path we define its *parent namespace* to be the
string that results from removing its rightmost segment.

举个例子, "A::B::C"的父命名空间是"A::B","A::B"的父命名空间是"A"，"A"的父命名空间是"".

当思考类和模块的时候，父名称空间的解释比较复杂。让我们考虑一个模块"A::B":

* 父命名空间"A"，有可能不在给定的嵌套中.

* 常量A有可能被其他代码从`Object`删除掉了.

* 如果`A`存在, the class or module that was originally in `A` may not be there
anymore. For example, if after a constant removal there was another constant
assignment there would generally be a different object in there.

* In such case, it could even happen that the reassigned `A` held a new class or
module called also "A"!

* In the previous scenarios M would no longer be reachable through `A::B` but
the module object itself could still be alive somewhere and its name would
still be "A::B".

The idea of a parent namespace is at the core of the autoloading algorithms
and helps explain and understand their motivation intuitively, but as you see
that metaphor leaks easily. Given an edge case to reason about, take always into
account that by "parent namespace" the guide means exactly that specific string
derivation.

### Loading Mechanism

Rails autoloads files with `Kernel#load` when `config.cache_classes` is false,
the default in development mode, and with `Kernel#require` otherwise, the
default in production mode.

`Kernel#load` allows Rails to execute files more than once if [constant
reloading](#constant-reloading) is enabled.

This guide uses the word "load" freely to mean a given file is interpreted, but
the actual mechanism can be `Kernel#load` or `Kernel#require` depending on that
flag.


Autoloading Availability
------------------------

Rails is always able to autoload provided its environment is in place. For
example the `runner` command autoloads:

```
$ bin/rails runner 'p User.column_names'
["id", "email", "created_at", "updated_at"]
```

The console autoloads, the test suite autoloads, and of course the application
autoloads.

By default, Rails eager loads the application files when it boots in production
mode, so most of the autoloading going on in development does not happen. But
autoloading may still be triggered during eager loading.

For example, given

```ruby
class BeachHouse < House
end
```

if `House` is still unknown when `app/models/beach_house.rb` is being eager
loaded, Rails autoloads it.


autoload_paths
--------------

As you probably know, when `require` gets a relative file name:

```ruby
require 'erb'
```

Ruby looks for the file in the directories listed in `$LOAD_PATH`. That is, Ruby
iterates over all its directories and for each one of them checks whether they
have a file called "erb.rb", or "erb.so", or "erb.o", or "erb.dll". If it finds
any of them, the interpreter loads it and ends the search. Otherwise, it tries
again in the next directory of the list. If the list gets exhausted, `LoadError`
is raised.

We are going to cover how constant autoloading works in more detail later, but
the idea is that when a constant like `Post` is hit and missing, if there's a
`post.rb` file for example in `app/models` Rails is going to find it, evaluate
it, and have `Post` defined as a side-effect.

Alright, Rails has a collection of directories similar to `$LOAD_PATH` in which
to look up `post.rb`. That collection is called `autoload_paths` and by
default it contains:

* All subdirectories of `app` in the application and engines. For example,
  `app/controllers`. They do not need to be the default ones, any custom
  directories like `app/workers` belong automatically to `autoload_paths`.

* Any existing second level directories called `app/*/concerns` in the
  application and engines.

* The directory `test/mailers/previews`.

Also, this collection is configurable via `config.autoload_paths`. For example,
`lib` was in the list years ago, but no longer is. An application can opt-in
by adding this to `config/application.rb`:

```ruby
config.autoload_paths += "#{Rails.root}/lib"
```

The value of `autoload_paths` can be inspected. In a just generated application
it is (edited):

```
$ bin/rails r 'puts ActiveSupport::Dependencies.autoload_paths'
.../app/assets
.../app/controllers
.../app/helpers
.../app/mailers
.../app/models
.../app/controllers/concerns
.../app/models/concerns
.../test/mailers/previews
```

INFO. `autoload_paths` is computed and cached during the initialization process.
The application needs to be restarted to reflect any changes in the directory
structure.


Autoloading Algorithms
----------------------

### Relative References

A relative constant reference may appear in several places, for example, in

```ruby
class PostsController < ApplicationController
  def index
    @posts = Post.all
  end
end
```

all three constant references are relative.

#### Constants after the `class` and `module` Keywords

Ruby performs a lookup for the constant that follows a `class` or `module`
keyword because it needs to know if the class or module is going to be created
or reopened.

If the constant is not defined at that point it is not considered to be a
missing constant, autoloading is **not** triggered.

So, in the previous example, if `PostsController` is not defined when the file
is interpreted Rails autoloading is not going to be triggered, Ruby will just
define the controller.

#### Top-Level Constants

On the contrary, if `ApplicationController` is unknown, the constant is
considered missing and an autoload is going to be attempted by Rails.

In order to load `ApplicationController`, Rails iterates over `autoload_paths`.
First checks if `app/assets/application_controller.rb` exists. If it does not,
which is normally the case, it continues and finds
`app/controllers/application_controller.rb`.

If the file defines the constant `ApplicationController` all is fine, otherwise
`LoadError` is raised:

```
unable to autoload constant ApplicationController, expected
<full path to application_controller.rb> to define it (LoadError)
```

INFO. Rails does not require the value of autoloaded constants to be a class or
module object. For example, if the file `app/models/max_clients.rb` defines
`MAX_CLIENTS = 100` autoloading `MAX_CLIENTS` works just fine.

#### Namespaces

Autoloading `ApplicationController` looks directly under the directories of
`autoload_paths` because the nesting in that spot is empty. The situation of
`Post` is different, the nesting in that line is `[PostsController]` and support
for namespaces comes into play.

The basic idea is that given

```ruby
module Admin
  class BaseController < ApplicationController
    @@all_roles = Role.all
  end
end
```

to autoload `Role` we are going to check if it is defined in the current or
parent namespaces, one at a time. So, conceptually we want to try to autoload
any of

```
Admin::BaseController::Role
Admin::Role
Role
```

in that order. That's the idea. To do so, Rails looks in `autoload_paths`
respectively for file names like these:

```
admin/base_controller/role.rb
admin/role.rb
role.rb
```

modulus some additional directory lookups we are going to cover soon.

INFO. `'Constant::Name'.underscore` gives the relative path without extension of
the file name where `Constant::Name` is expected to be defined.

Let's see how Rails autoloads the `Post` constant in the `PostsController`
above assuming the application has a `Post` model defined in
`app/models/post.rb`.

First it checks for `posts_controller/post.rb` in `autoload_paths`:

```
app/assets/posts_controller/post.rb
app/controllers/posts_controller/post.rb
app/helpers/posts_controller/post.rb
...
test/mailers/previews/posts_controller/post.rb
```

Since the lookup is exhausted without success, a similar search for a directory
is performed, we are going to see why in the [next section](#automatic-modules):

```
app/assets/posts_controller/post
app/controllers/posts_controller/post
app/helpers/posts_controller/post
...
test/mailers/previews/posts_controller/post
```

If all those attempts fail, then Rails starts the lookup again in the parent
namespace. In this case only the top-level remains:

```
app/assets/post.rb
app/controllers/post.rb
app/helpers/post.rb
app/mailers/post.rb
app/models/post.rb
```

A matching file is found in `app/models/post.rb`. The lookup stops there and the
file is loaded. If the file actually defines `Post` all is fine, otherwise
`LoadError` is raised.

### Qualified References

When a qualified constant is missing Rails does not look for it in the parent
namespaces. But there is a caveat: When a constant is missing, Rails is
unable to tell if the trigger was a relative reference or a qualified one.

For example, consider

```ruby
module Admin
  User
end
```

and

```ruby
Admin::User
```

If `User` is missing, in either case all Rails knows is that a constant called
"User" was missing in a module called "Admin".

If there is a top-level `User` Ruby would resolve it in the former example, but
wouldn't in the latter. In general, Rails does not emulate the Ruby constant
resolution algorithms, but in this case it tries using the following heuristic:

> If none of the parent namespaces of the class or module has the missing
> constant then Rails assumes the reference is relative. Otherwise qualified.

For example, if this code triggers autoloading

```ruby
Admin::User
```

and the `User` constant is already present in `Object`, it is not possible that
the situation is

```ruby
module Admin
  User
end
```

because otherwise Ruby would have resolved `User` and no autoloading would have
been triggered in the first place. Thus, Rails assumes a qualified reference and
considers the file `admin/user.rb` and directory `admin/user` to be the only
valid options.

In practice, this works quite well as long as the nesting matches all parent
namespaces respectively and the constants that make the rule apply are known at
that time.

However, autoloading happens on demand. If by chance the top-level `User` was
not yet loaded, then Rails assumes a relative reference by contract.

Naming conflicts of this kind are rare in practice, but if one occurs,
`require_dependency` provides a solution by ensuring that the constant needed
to trigger the heuristic is defined in the conflicting place.

### Automatic Modules

When a module acts as a namespace, Rails does not require the application to
defines a file for it, a directory matching the namespace is enough.

Suppose an application has a back office whose controllers are stored in
`app/controllers/admin`. If the `Admin` module is not yet loaded when
`Admin::UsersController` is hit, Rails needs first to autoload the constant
`Admin`.

If `autoload_paths` has a file called `admin.rb` Rails is going to load that
one, but if there's no such file and a directory called `admin` is found, Rails
creates an empty module and assigns it to the `Admin` constant on the fly.

### Generic Procedure

Relative references are reported to be missing in the cref where they were hit,
and qualified references are reported to be missing in their parent. (See
[Resolution Algorithm for Relative
Constants](#resolution-algorithm-for-relative-constants) at the beginning of
this guide for the definition of *cref*, and [Resolution Algorithm for Qualified
Constants](#resolution-algorithm-for-qualified-constants) for the definition of
*parent*.)

The procedure to autoload constant `C` in an arbitrary situation is as follows:

```
if the class or module in which C is missing is Object
  let ns = ''
else
  let M = the class or module in which C is missing

  if M is anonymous
    let ns = ''
  else
    let ns = M.name
  end
end

loop do
  # Look for a regular file.
  for dir in autoload_paths
    if the file "#{dir}/#{ns.underscore}/c.rb" exists
      load/require "#{dir}/#{ns.underscore}/c.rb"

      if C is now defined
        return
      else
        raise LoadError
      end
    end
  end

  # Look for an automatic module.
  for dir in autoload_paths
    if the directory "#{dir}/#{ns.underscore}/c" exists
      if ns is an empty string
        let C = Module.new in Object and return
      else
        let C = Module.new in ns.constantize and return
      end
    end
  end

  if ns is empty
    # We reached the top-level without finding the constant.
    raise NameError
  else
    if C exists in any of the parent namespaces
      # Qualified constants heuristic.
      raise NameError
    else
      # Try again in the parent namespace.
      let ns = the parent namespace of ns and retry
    end
  end
end
```


require_dependency
------------------

Constant autoloading is triggered on demand and therefore code that uses a
certain constant may have it already defined or may trigger an autoload. That
depends on the execution path and it may vary between runs.

There are times, however, in which you want to make sure a certain constant is
known when the execution reaches some code. `require_dependency` provides a way
to load a file using the current [loading mechanism](#loading-mechanism), and
keeping track of constants defined in that file as if they were autoloaded to
have them reloaded as needed.

`require_dependency` is rarely needed, but see a couple of use-cases in
[Autoloading and STI](#autoloading-and-sti) and [When Constants aren't
Triggered](#when-constants-aren-t-missed).

WARNING. Unlike autoloading, `require_dependency` does not expect the file to
define any particular constant. Exploiting this behavior would be a bad practice
though, file and constant paths should match.


Constant Reloading
------------------

When `config.cache_classes` is false Rails is able to reload autoloaded
constants.

For example, in you're in a console session and edit some file behind the
scenes, the code can be reloaded with the `reload!` command:

```
> reload!
```

When the application runs, code is reloaded when something relevant to this
logic changes. In order to do that, Rails monitors a number of things:

* `config/routes.rb`.

* Locales.

* Ruby files under `autoload_paths`.

* `db/schema.rb` and `db/structure.sql`.

If anything in there changes, there is a middleware that detects it and reloads
the code.

Autoloading keeps track of autoloaded constants. Reloading is implemented by
removing them all from their respective classes and modules using
`Module#remove_const`. That way, when the code goes on, those constants are
going to be unknown again, and files reloaded on demand.

INFO. This is an all-or-nothing operation, Rails does not attempt to reload only
what changed since dependencies between classes makes that really tricky.
Instead, everything is wiped.


Module#autoload isn't Involved
------------------------------

`Module#autoload` provides a lazy way to load constants that is fully integrated
with the Ruby constant lookup algorithms, dynamic constant API, etc. It is quite
transparent.

Rails internals make extensive use of it to defer as much work as possible from
the boot process. But constant autoloading in Rails is **not** implemented with
`Module#autoload`.

One possible implementation based on `Module#autoload` would be to walk the
application tree and issue `autoload` calls that map existing file names to
their conventional constant name.

There are a number of reasons that prevent Rails from using that implementation.

For example, `Module#autoload` is only capable of loading files using `require`,
so reloading would not be possible. Not only that, it uses an internal `require`
which is not `Kernel#require`.

Then, it provides no way to remove declarations in case a file is deleted. If a
constant gets removed with `Module#remove_const` its `autoload` is not triggered
again. Also, it doesn't support qualified names, so files with namespaces should
be interpreted during the walk tree to install their own `autoload` calls, but
those files could have constant references not yet configured.

An implementation based on `Module#autoload` would be awesome but, as you see,
at least as of today it is not possible. Constant autoloading in Rails is
implemented with `Module#const_missing`, and that's why it has its own contract,
documented in this guide.


Common Gotchas
--------------

### Nesting and Qualified Constants

Let's consider

```ruby
module Admin
  class UsersController < ApplicationController
    def index
      @users = User.all
    end
  end
end
```

and

```ruby
class Admin::UsersController < ApplicationController
  def index
    @users = User.all
  end
end
```

To resolve `User` Ruby checks `Admin` in the former case, but it does not in
the latter because it does not belong to the nesting. (See [Nesting](#nesting)
and [Resolution Algorithms](#resolution-algorithms).)

Unfortunately Rails autoloading does not know the nesting in the spot where the
constant was missing and so it is not able to act as Ruby would. In particular,
`Admin::User` will get autoloaded in either case.

Albeit qualified constants with `class` and `module` keywords may technically
work with autoloading in some cases, it is preferable to use relative constants
instead:

```ruby
module Admin
  class UsersController < ApplicationController
    def index
      @users = User.all
    end
  end
end
```

### Autoloading and STI

Single Table Inheritance (STI) is a feature of Active Record that enables
storing a hierarchy of models in one single table. The API of such models is
aware of the hierarchy and encapsulates some common needs. For example, given
these classes:

```ruby
# app/models/polygon.rb
class Polygon < ActiveRecord::Base
end

# app/models/triangle.rb
class Triangle < Polygon
end

# app/models/rectangle.rb
class Rectangle < Polygon
end
```

`Triangle.create` creates a row that represents a triangle, and
`Rectangle.create` creates a row that represents a rectangle. If `id` is the
ID of an existing record, `Polygon.find(id)` returns an object of the correct
type.

Methods that operate on collections are also aware of the hierarchy. For
example, `Polygon.all` returns all the records of the table, because all
rectangles and triangles are polygons. Active Record takes care of returning
instances of their corresponding class in the result set.

Types are autoloaded as needed. For example, if `Polygon.first` is a rectangle
and `Rectangle` has not yet been loaded, Active Record autoloads it and the
record is correctly instantiated.

All good, but if instead of performing queries based on the root class we need
to work on some subclass, things get interesting.

While working with `Polygon` you do not need to be aware of all its descendants,
because anything in the table is by definition a polygon, but when working with
subclasses Active Record needs to be able to enumerate the types it is looking
for. Let’s see an example.

`Rectangle.all` only loads rectangles by adding a type constraint to the query:

```sql
SELECT "polygons".* FROM "polygons"
WHERE "polygons"."type" IN ("Rectangle")
```

Let’s introduce now a subclass of `Rectangle`:

```ruby
# app/models/square.rb
class Square < Rectangle
end
```

`Rectangle.all` should now return rectangles **and** squares:

```sql
SELECT "polygons".* FROM "polygons"
WHERE "polygons"."type" IN ("Rectangle", "Square")
```

But there’s a caveat here: How does Active Record know that the class `Square`
exists at all?

Even if the file `app/models/square.rb` exists and defines the `Square` class,
if no code yet used that class, `Rectangle.all` issues the query

```sql
SELECT "polygons".* FROM "polygons"
WHERE "polygons"."type" IN ("Rectangle")
```

That is not a bug, the query includes all *known* descendants of `Rectangle`.

A way to ensure this works correctly regardless of the order of execution is to
load the leaves of the tree by hand at the bottom of the file that defines the
root class:

```ruby
# app/models/polygon.rb
class Polygon < ActiveRecord::Base
end
require_dependency ‘square’
```

Only the leaves that are **at least grandchildren** need to be loaded this
way. Direct subclasses do not need to be preloaded. If the hierarchy is
deeper, intermediate classes will be autoloaded recursively from the bottom
because their constant will appear in the class definitions as superclass.

### Autoloading and `require`

Files defining constants to be autoloaded should never be `require`d:

```ruby
require 'user' # DO NOT DO THIS

class UsersController < ApplicationController
  ...
end
```

There are two possible gotchas here in development mode:

1. If `User` is autoloaded before reaching the `require`, `app/models/user.rb`
runs again because `load` does not update `$LOADED_FEATURES`.

2. If the `require` runs first Rails does not mark `User` as an autoloaded
constant and changes to `app/models/user.rb` aren't reloaded.

Just follow the flow and use constant autoloading always, never mix
autoloading and `require`. As a last resort, if some file absolutely needs to
load a certain file use `require_dependency` to play nice with constant
autoloading. This option is rarely needed in practice, though.

Of course, using `require` in autoloaded files to load ordinary 3rd party
libraries is fine, and Rails is able to distinguish their constants, they are
not marked as autoloaded.

### Autoloading and Initializers

Consider this assignment in `config/initializers/set_auth_service.rb`:

```ruby
AUTH_SERVICE = if Rails.env.production?
  RealAuthService
else
  MockedAuthService
end
```

The purpose of this setup would be that the application uses the class that
corresponds to the environment via `AUTH_SERVICE`. In development mode
`MockedAuthService` gets autoloaded when the initializer runs. Let’s suppose
we do some requests, change its implementation, and hit the application again.
To our surprise the changes are not reflected. Why?

As [we saw earlier](#constant-reloading), Rails removes autoloaded constants,
but `AUTH_SERVICE` stores the original class object. Stale, non-reachable
using the original constant, but perfectly functional.

The following code summarizes the situation:

```ruby
class C
  def quack
    'quack!'
  end
end

X = C
Object.instance_eval { remove_const(:C) }
X.new.quack # => quack!
X.name      # => C
C           # => uninitialized constant C (NameError)
```

Because of that, it is not a good idea to autoload constants on application
initialization.

In the case above we could implement a dynamic access point:

```ruby
# app/models/auth_service.rb
class AuthService
  if Rails.env.production?
    def self.instance
      RealAuthService
    end
  else
    def self.instance
      MockedAuthService
    end
  end
end
```

and have the application use `AuthService.instance` instead. `AuthService`
would be loaded on demand and be autoload-friendly.

### `require_dependency` and Initializers

As we saw before, `require_dependency` loads files in an autoloading-friendly
way. Normally, though, such a call does not make sense in an initializer.

One could think about doing some [`require_dependency`](#require-dependency)
calls in an initializer to make sure certain constants are loaded upfront, for
example as an attempt to address the [gotcha with STIs](#autoloading-and-sti).

Problem is, in development mode [autoloaded constants are wiped](#constant-reloading)
if there is any relevant change in the file system. If that happens then
we are in the very same situation the initializer wanted to avoid!

Calls to `require_dependency` have to be strategically written in autoloaded
spots.

### When Constants aren't Missed

#### Relative References

Let's consider a flight simulator. The application has a default flight model

```ruby
# app/models/flight_model.rb
class FlightModel
end
```

that can be overridden by each airplane, for instance

```ruby
# app/models/bell_x1/flight_model.rb
module BellX1
  class FlightModel < FlightModel
  end
end

# app/models/bell_x1/aircraft.rb
module BellX1
  class Aircraft
    def initialize
      @flight_model = FlightModel.new
    end
  end
end
```

The initializer wants to create a `BellX1::FlightModel` and nesting has
`BellX1`, that looks good. But if the default flight model is loaded and the
one for the Bell-X1 is not, the interpreter is able to resolve the top-level
`FlightModel` and autoloading is thus not triggered for `BellX1::FlightModel`.

That code depends on the execution path.

These kind of ambiguities can often be resolved using qualified constants:

```ruby
module BellX1
  class Plane
    def flight_model
      @flight_model ||= BellX1::FlightModel.new
    end
  end
end
```

Also, `require_dependency` is a solution:

```ruby
require_dependency 'bell_x1/flight_model'

module BellX1
  class Plane
    def flight_model
      @flight_model ||= FlightModel.new
    end
  end
end
```

#### Qualified References

Given

```ruby
# app/models/hotel.rb
class Hotel
end

# app/models/image.rb
class Image
end

# app/models/hotel/image.rb
class Hotel
  class Image < Image
  end
end
```

the expression `Hotel::Image` is ambiguous, depends on the execution path.

As [we saw before](#resolution-algorithm-for-qualified-constants), Ruby looks
up the constant in `Hotel` and its ancestors. If `app/models/image.rb` has
been loaded but `app/models/hotel/image.rb` hasn't, Ruby does not find `Image`
in `Hotel`, but it does in `Object`:

```
$ bin/rails r 'Image; p Hotel::Image' 2>/dev/null
Image # NOT Hotel::Image!
```

The code evaluating `Hotel::Image` needs to make sure
`app/models/hotel/image.rb` has been loaded, possibly with
`require_dependency`.

In these cases the interpreter issues a warning though:

```
warning: toplevel constant Image referenced by Hotel::Image
```

This surprising constant resolution can be observed with any qualifying class:

```
2.1.5 :001 > String::Array
(irb):1: warning: toplevel constant Array referenced by String::Array
 => Array
```

WARNING. To find this gotcha the qualifying namespace has to be a class,
`Object` is not an ancestor of modules.

### Autoloading within Singleton Classes

Let's suppose we have these class definitions:

```ruby
# app/models/hotel/services.rb
module Hotel
  class Services
  end
end

# app/models/hotel/geo_location.rb
module Hotel
  class GeoLocation
    class << self
      Services
    end
  end
end
```

If `Hotel::Services` is known by the time `app/models/hotel/geo_location.rb`
is being loaded, `Services` is resolved by Ruby because `Hotel` belongs to the
nesting when the singleton class of `Hotel::GeoLocation` is opened.

But if `Hotel::Services` is not known, Rails is not able to autoload it, the
application raises `NameError`.

The reason is that autoloading is triggered for the singleton class, which is
anonymous, and as [we saw before](#generic-procedure), Rails only checks the
top-level namespace in that edge case.

An easy solution to this caveat is to qualify the constant:

```ruby
module Hotel
  class GeoLocation
    class << self
      Hotel::Services
    end
  end
end
```

### Autoloading in `BasicObject`

Direct descendants of `BasicObject` do not have `Object` among their ancestors
and cannot resolve top-level constants:

```ruby
class C < BasicObject
  String # NameError: uninitialized constant C::String
end
```

When autoloading is involved that plot has a twist. Let's consider:

```ruby
class C < BasicObject
  def user
    User # WRONG
  end
end
```

Since Rails checks the top-level namespace `User` gets autoloaded just fine the
first time the `user` method is invoked. You only get the exception if the
`User` constant is known at that point, in particular in a *second* call to
`user`:

```ruby
c = C.new
c.user # surprisingly fine, User
c.user # NameError: uninitialized constant C::User
```

because it detects a parent namespace already has the constant (see [Qualified
References](#qualified-references).)

As with pure Ruby, within the body of a direct descendant of `BasicObject` use
always absolute constant paths:

```ruby
class C < BasicObject
  ::String # RIGHT

  def user
    ::User # RIGHT
  end
end
```
