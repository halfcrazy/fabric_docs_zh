========
远端交互
========

Fabric 的两大主要操作 `~fabric.operations.run` 和 `~fabric.operations.sudo` ，能够以类似于 ``ssh`` 程序的方式，把本地输入操作传送到远端（服务器）。例如，能显示密码提示的程序（数据的 dump 工具集或者用户名修改密码等）将会表现的正如你自己在直接操作一样。

然而，就像 ``ssh`` 本身不够直观一样，Fabric 对 ssh 特性的实现也受到了些限制。本篇接下来主要详细讨论这个问题。

.. note::
    读者们如果不熟悉Unix的输出和错误管道和终端设备的基本知识，不妨分别访问维基百科的词条：`Unix pipelines
    <http://en.wikipedia.org/wiki/Pipe_(Unix)>`_ 和 `Pseudo terminals
    <http://en.wikipedia.org/wiki/Pseudo_terminal>`。


.. _combine_streams:

stdout & stderr 合并
====================

第一个注意的问题就是标准的准错输出流控制，以及它们怎么按需分离和合并。

缓冲机制
--------

Fabric 0.9.x 及早期版本和 Python 本身都是按基本的“逐行”的缓存输出模式：直到发现有新一行字符要打印，否则文本不会打印输出给用户。这样处理大多数情况下没什么问题，但是一到用户需要处理类似提示的“半行”输出时，这将成为一个问题。

.. note::
    “行缓冲”式输出可以无征兆下使程序出现停止或冻结，因为提示会打印出文本而不换行，同时会等待用户输入，然后按回车。

新版 Fabric 的输入输出缓冲机智基于“逐字符”，目的就是能和提示进行交互。同时在使用 "Curses" 库或者其他方式重绘屏幕（想一下top）的时候，和复杂程序交互的副作用也会结伴而来。
这有使互动与复杂的程序，利用“curses”库或以其他方式重绘屏幕（想想 "top"）的方便的副作用。

流之外
------

不幸的是，同时打印到 stderr 和 stdout （正如多程序的做法）是指当两个流独立地一次打印一个字节，它们就开始混淆在一起。虽然这有时可以通过行缓冲一个流来而不管另一个流来缓解，它仍然是一个严重的问题。

为了解决这个问题，Fabric 在我们的 SSH 层设置了在低水平上融合两个流，并使输出显得更自然。该设置在 Fabric 里是以 :ref:`combine_stderr` 的环境变量和关键字参数来实现，默认为 ``True`` 。

介于默认设置，输出将正确显示，但代价就是 `~fabric.operations.run`/`~fabric.operations.sudo` 的返回值是一个的空 ``.stderr`` 属性，因为所有的输出将显示为标准输出。

相反，需要在 Python 层级鲜明的表达标准错误流时谁要是想不被面向用户的输出的乱码困扰（或谁想从有问题的命令中屏蔽掉 stdout 和 stderr），可以选择根据需要设置为 ``False`` 。


.. _pseudottys:

伪终端
======

在呈现交互式提示给用户时，另一个需要考虑的主要问题是呼应用户自己的输入。

呼应
----

典型终端应用程序或真正的文本终端（例如，使用时不需要一个运行在 Unix 系统的 GUI），是使用一个叫做 tty 或 PTY 终端（伪终端）来展示程序。诸如能够将用户输入的内容在重新返还在用户面前（通过标准输出），因为看不到之前输入内容的交互是很难的。考虑到密码提示的安全性，终端设备能够有条件的关闭呼应功能。

然而，程序不在任何一个 tty 或 PTY 里显示就运行也是可能的（考虑到 cron 后台守护进程），在这种情况下，任何标准输入的数据都不会载入呼应里。这是理想的情况，也是操作的 Fabric 的旧的默认模式。

Fabric 的解决之道
-----------------

不幸的是，在经由 Fabric 执行命令的情况下，当没有 PTY 存呼应用户的标准输入，Fabric 必须呼应他们了。这已经足够用于许多应用中了，但它也暴露了一个不安全的问题，就是密码提示的问题。

在安全性和满足最惊喜的原则都满足的情况下（用户通常期望事情就像他们在终端模拟器上运行时的一样）， Fabric 1.0 及更高版本强制使用了一个 PTY 。随着 PTY 启用，Fabric 只简单的允许远端处理呼应或隐藏标准输入，并且不呼应任何事情本身。

.. note::
    除了允许正常的呼应行为，一个PTY也意味着，当连接到终端设备会再做这样的行为不同的程序。例如，着色输出终端上而不是在后台运行的程序时，将打印彩色输出。警惕这一点，如果你检查 `~fabric.operations.run` 或 `~fabric.operations.sudo` 的返回值！

对于需要关闭 PTY 行为的情况，或许会用到 :option:`--no-pty` 命令行参数和 :ref:`always_use_pty` 环境变量。


合二为一
========

最后一点，请记住，善于使用伪终端意味着 stdout 和 stderr 的融合 -- 大多类似 :ref:`combine_stderr<combine_streams>` 设置一样。这是因为终端设备自然发送 stdout 和 stderr 到同一个地方 - 用户显示器 - 从而使其无法区分它们。

然而，在 Fabric 层面，两组的设置是彼此不同的并以各种方式进行组合。它们的缺省值 ``True``; 其它组合如下：

* ``run("cmd", pty=False, combine_stderr=True)``：会使 Fabric 呼应所有的标准输入本身，包括密码，以及可能改变的行为。 还挺管用，如果在 pty 下 ``cmd`` 命令表现不佳时，你是不用担心密码提示的。
* ``run("cmd", pty=False, combine_stderr=False)``：既设置为 False ，Fabric 会呼应标准输入，不会发出 PTY - 而这对所有命令来说极有可能导致意外发生，除了那些最简单的命令。然而，它也是以访问不同的标准错误流，这是偶尔有用的唯一途径。
* ``run("cmd", pty=True, combine_stderr=False)``：有效的，但不会真正使多大的差别，如 ``PTY=True`` 仍会导致合并后的数据流。可用于避免在会发生在 ``combine_stderr`` 边缘下（目前未知）的任何问题。



