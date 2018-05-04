# 图解 ES modules

*本文翻译自 [Lin Clark](http://code-cartoons.com/) 的文章[《ES modules: A cartoon deep-dive》](https://hacks.mozilla.org/2018/03/es-modules-a-cartoon-deep-dive/)*

ES modules （以下简称 ESM） 为 JavaScript 带来了标准化的模块系统，我们也曾为此付出了不小的努力——将近 10 年的标准化工作。  
随着 Firefox 60 将在 5 月发布，所有的主流浏览器都将支持 ESM；Node.js 正在着手支持 ESM；[ESM 的 WebAssembly 整合](https://www.youtube.com/watch?v=qR_b5gajwug) 也正在进行中。

许多 JS 开发者都知道 ESM 是一个老生常谈的话题了，但是实际上深入了解过它的人只是少数。  
现在就让我们一起来看看 ESM 到底解决了什么问题，以及它和其他模块系统有何异同。

## 模块化要解决什么问题？

JS 编程即是在管理变量。我们对变量进行赋值，或者对两个变量进行组合，将其放进另一个变量。

![01_variables](https://hacks.mozilla.org/files/2018/03/01_variables.png)

我们实在有太多的代码都在对变量进行处理，组织变量的方式将会对编写代码、维护代码产生巨大的影响。  
一次仅考虑少量变量会让事情变得更简单。JS 的作用域会帮助你这样做，作用域使函数无法访问定义在其他函数中的变量。

![02_module_scope_01](https://hacks.mozilla.org/files/2018/03/02_module_scope_01.png)

作用域很棒，它让我们专注于当前的函数，不必担心其他函数会对你的变量动手动脚。不过它的另一面就是，在不同函数中共用变量变得更难了。  
如果想在两个作用域中共用一个变量你会怎么做？通常的手段是将变量放在更上级的作用域中……比如全局作用域。  
可能这会让你想到使用 jQuery 的那些日子。在加载 jQuery 插件之前，首先要保证 jQuery 在你的全局作用域中。

![02_module_scope_02](https://hacks.mozilla.org/files/2018/03/02_module_scope_02.png)

这样做无可厚非，但是这种方式会导致一些恼人的问题。  
你不仅需要保证 script 标签必须按照正确的顺序摆放，还要确保它们不会打乱。如果有人不小心这么做了，你的应用可能会在运行时给你抛个错误出来。当函数尝试寻找在全局作用域的 jQuery 却没找到时，它会报错停止运行。

![02_module_scope_03](https://hacks.mozilla.org/files/2018/03/02_module_scope_03.png)

这会让代码难以维护，删除老的代码或者标签会变得惊险又刺激，你不知道什么时候会踩到地雷。代码之间的依赖关系是隐式的，任何函数都可能依赖全局作用域中的变量，你也不知道哪个函数会依赖哪块 script 了。

另一个问题是因为变量在全局作用域中，任何代码都有可能修改变量。恶意代码可以趁机达到某些目的，非恶意代码也可能不小心对它们进行修改。

## 模块化如何解决这些问题？

模块化提供了一种更好的方式来组织这些变量和函数，你可以用它将一段有意义的变量和函数组合起来。  
函数和变量被放进模块作用域中，模块作用域负责将变量提供给模块中的函数使用。  
和函数作用域不同的是，模块作用域同样提供了将变量提供给其他模块使用的方法，模块中的变量、类或函数能被显式地声明它们可以被其他模块用到。  
这样的声明叫做导出（export）。其他模块可以显式地声明依赖任何一个被导出的变量、类或函数。

![02_module_scope_03](https://hacks.mozilla.org/files/2018/03/02_module_scope_04.png)

由于依赖关系是显式声明的，我们很容易看出缺少了某个模块后带来的后果。  
得益于这种从模块中导出和导入变量的能力，我们能很容易地将所有代码切分成独立工作的代码块，我们可以合并和重组这些代码块——就像乐高积木一样——用模块来构建各种不同类型的应用。  
为了将模块功能引入 JS，人们已经做了多种尝试。如今我们主要使用两种模块系统：CommonJS 因为历史原因被用在 Node.js 的模块系统中，ESM 是 ES 标准中新加入的模块化方案，浏览器现在已经支持 ESM，Node.js 也正在向支持 ESM 迈进。

现在让我们来进一步了解 ESM 的内部机制。

## ESM 的运行机制

当在开发中使用模块时，我们建立了一个依赖关系的图，不同依赖之间的联系来自我们的导入声明（import statements）。这些导入声明能让浏览器或者 Node 清楚地知道它们应该加载哪些代码，我们给出一个文件作为入口，然后根据它的导入声明来找到所有剩下的代码。

![02_module_scope_03](https://hacks.mozilla.org/files/2018/03/04_import_graph.png)

但是，浏览器并不能直接使用文件本身，它们需要被解析为一种被称为模块记录（Module Records）的数据结构。

![02_module_scope_03](https://hacks.mozilla.org/files/2018/03/05_module_record.png)

模块记录之后会被转化为模块实例，一个模块实例包含了两样东西：代码和状态。  
代码是指令的集合，但是只有代码本身不能做任何事，你需要一些原料来供指令使用。  
状态是什么？状态即是这些原料。状态是变量在任何时间点的实际的值。当然，这些变量只是内存中一些数据的昵称罢了。

模块实例将这两者组合起来：代码（一串指令的列表）和状态（所有变量的值）。

![02_module_scope_03](https://hacks.mozilla.org/files/2018/03/06_module_instance.png)


我们所需要的正是每一个模块的模块实例。模块读取的过程就是从入口文件拿到整个依赖关系图的模块实例的过程。

对于 ESM，这个过程可以分为三个步骤：
1. 构造（Construction）——加载脚本并解析为模块记录。
2. 实例化（Instantiation）——给所有导出变量分配内存空间（但是先不进行初始化），然后让对应的导入变量、导出变量指向这个地址，这被称为链接（linking）。
3. 赋值（Evaluation）——运行代码求值，对分配的内存空间进行赋值。

![02_module_scope_03](https://hacks.mozilla.org/files/2018/03/07_3_phases.png)

ES 标准明确指出 ESM 是异步的，你可以理解为这三个步骤是分开执行的。而在 CommonJS 中，依赖模块的加载、实例化和赋值都是一起执行的。