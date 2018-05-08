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
你不仅需要保证 `<script>` 标签必须按照正确的顺序摆放，还要确保它们不会打乱。如果有人不小心这么做了，你的应用可能会在运行时给你抛个错误出来。当函数尝试寻找在全局作用域的 jQuery 却没找到时，它会报错停止运行。

![02_module_scope_03](https://hacks.mozilla.org/files/2018/03/02_module_scope_03.png)

这会让代码难以维护，删除老的代码或者标签会变得惊险又刺激，你不知道什么时候会踩到地雷。代码之间的依赖关系是隐式的，任何函数都可能依赖全局作用域中的变量，你也不知道哪个函数会依赖哪块 `<script>` 了。

另一个问题是因为变量在全局作用域中，任何代码都有可能修改变量。恶意代码可以趁机达到某些目的，非恶意代码也可能不小心对它们进行修改。

## 模块化如何解决这些问题？

模块化提供了一种更好的方式来组织这些变量和函数，你可以用它将一段有意义的变量和函数组合起来。  
函数和变量被放进模块作用域中，模块作用域负责将变量提供给模块中的函数使用。  
和函数作用域不同的是，模块作用域同样提供了将变量提供给其他模块使用的方法，模块中的变量、类或函数能被显式地声明它们可以被其他模块用到。  
这样的声明叫做导出（export）。其他模块可以显式地声明依赖任何一个被导出的变量、类或函数。

![02_module_scope_04](https://hacks.mozilla.org/files/2018/03/02_module_scope_04.png)

由于依赖关系是显式声明的，我们很容易看出缺少了某个模块后带来的后果。  
得益于这种从模块中导出和导入变量的能力，我们能很容易地将所有代码切分成独立工作的代码块，我们可以合并和重组这些代码块——就像乐高积木一样——用模块来构建各种不同类型的应用。  
为了将模块功能引入 JS，人们已经做了多种尝试。如今我们主要使用两种模块系统：CommonJS 因为历史原因被用在 Node.js 的模块系统中，ESM 是 ES 标准中新加入的模块化方案，浏览器现在已经支持 ESM，Node.js 也正在向支持 ESM 迈进。

现在让我们来进一步了解 ESM 的内部机制。

## ESM 的运行机制

当在开发中使用模块时，我们建立了一个依赖关系的图，不同依赖之间的联系来自我们的导入声明（import statements）。这些导入声明能让浏览器或者 Node 清楚地知道它们应该加载哪些代码，我们给出一个文件作为入口，然后根据它的导入声明来找到所有剩下的代码。

![04_import_graph](https://hacks.mozilla.org/files/2018/03/04_import_graph.png)

但是，浏览器并不能直接使用文件本身，它们需要被解析为一种被称为模块记录（Module Records）的数据结构。

![05_module_record](https://hacks.mozilla.org/files/2018/03/05_module_record.png)

模块记录之后会被转化为模块实例，一个模块实例包含了两样东西：代码和状态。  
代码是指令的集合，但是只有代码本身不能做任何事，你需要一些原料来供指令使用。  
状态是什么？状态即是这些原料。状态是变量在任何时间点的实际的值。当然，这些变量只是内存中一些数据的昵称罢了。

模块实例将这两者组合起来：代码（一串指令的列表）和状态（所有变量的值）。

![06_module_instance](https://hacks.mozilla.org/files/2018/03/06_module_instance.png)


我们所需要的正是每一个模块的模块实例。模块读取的过程就是从入口文件拿到整个依赖关系图的模块实例的过程。

对于 ESM，这个过程可以分为三个步骤：
1. 构造（Construction）——加载脚本并解析为模块记录。
2. 实例化（Instantiation）——给所有导出变量分配内存空间（但是先不进行初始化），然后让对应的导入变量、导出变量指向这个地址，这被称为链接（linking）。
3. 赋值（Evaluation）——运行代码求值，对分配的内存空间进行赋值。

![07_3_phases](https://hacks.mozilla.org/files/2018/03/07_3_phases.png)

CommonJS 中加载文件、实例化和赋值是一起执行的，而 ESM 将上述三个步骤在不同的阶段执行，所以人们经常把 ESM 描述为异步的。  
ESM 的标准中确实定义了这种异步特性，我们会在下文中详细阐述。

模块本身是没有异步的必要性的，但是模块的加载不仅涉及了上述的三个阶段，还跟如何加载文件有关。ESM 标准不会告诉你如何去加载文件，这部分内容定义在其他标准里。  
[ESM 标准](https://tc39.github.io/ecma262/#sec-modules) 告诉你该如何将文件解析为模块记录、如何实例化并赋值，但是它并不会告诉你一开始我们如何拿到这些文件。  
我们通过 loader 拉取到文件，不同的平台对 loader 会有不同的定义。对于浏览器来说，这部分内容定义在 [HTML 规范](https://html.spec.whatwg.org/#fetch-a-module-script-tree) 中。

![07_loader_vs_es](https://hacks.mozilla.org/files/2018/03/07_loader_vs_es.png)

loader 实际控制着如何加载模块，它会调用 ESM 方法—— `ParseModule`、`Module.Instantiate` 和 `Module.Evaluate`，就像提线木偶一样操纵着 JS 引擎。

![08_loader_as_puppeteer](https://hacks.mozilla.org/files/2018/03/08_loader_as_puppeteer.png)

现在让我们来进一步阐述各个步骤。

## 构造

对于每个模块、构造阶段会发生三件事：
1. 定位模块所在的文件——即模块解析（module resolution）。
2. 拉取文件（从一个 URL 下载或者在文件系统中加载）。
3. 将文件解析为模块记录。

## 定位并加载文件

loader 会负责找到并下载文件。首先需要找到入口文件，在 HTML 中，我们可以通过 `<script>` 标签来告诉 loader 入口文件的位置。

![08_script_entry](https://hacks.mozilla.org/files/2018/03/08_script_entry.png)

但是 loader 如何找到 `main.js` 所依赖的模块呢？在 import 语句中我们会使用模块定位符（module specifier）来指定下一个模块的位置。

![09_module_specifier](https://hacks.mozilla.org/files/2018/03/09_module_specifier.png)

值得一提的是：不同平台对定位符字符串有不同的解析机制，当下有一些能在 Node 解析的定位符无法在浏览器被解析，但是人们在 [致力解决这些问题](https://github.com/domenic/package-name-maps)。  
在这些问题解决之前，浏览器只接受 URL 作为定位符并从加载文件。而文件在被解析之前并不会告诉我们下一步依赖的模块，我们并不能一次性地拿到整个应用的依赖关系。  
这意味着我们只能一层一层地前进，先解析一个文件，找到它的依赖，再对它的依赖们执行同样的步骤。

![10_construction](https://hacks.mozilla.org/files/2018/03/10_construction.png)

可想而知，如果主线程一直等待下载这些文件，会有大量任务被阻塞，再浏览器中加载文件会占用大量的时间。