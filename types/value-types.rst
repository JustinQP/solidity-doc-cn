.. include:: ../glossaries.rst

.. index:: ! value type, ! type;value

.. _value-types:

值类型
======

以下类型也称为值类型，因为这些类型的变量将始终按值来传递。
也就是说，当这些变量被用作函数参数或者用在赋值语句中时，总会进行值拷贝。

.. index:: ! bool, ! true, ! false

布尔类型
--------

``bool`` ：可能的取值为字面常量值 ``true`` 和 ``false`` 。

运算符：

*  ``!`` （逻辑非）
*  ``&&`` （逻辑与， "and" ）
*  ``||`` （逻辑或， "or" ）
*  ``==`` （等于）
*  ``!=`` （不等于）

运算符 ``||`` 和 ``&&`` 都遵循同样的短路（ short-circuiting ）规则。就是说在表达式 ``f(x) || g(y)`` 中，
如果 ``f(x)`` 的值为 ``true`` ，那么 ``g(y)`` 就不会被执行，即使会出现一些副作用。

.. index:: ! uint, ! int, ! integer
.. _integers:

整型
----

``int`` / ``uint`` ：分别表示有符号和无符号的不同位数的整型变量。
支持关键字 ``uint8`` 到 ``uint256`` （无符号，从 8 位到 256 位）以及 ``int8`` 到 ``int256``，以 ``8`` 位为步长递增。
``uint`` 和 ``int`` 分别是 ``uint256`` 和 ``int256`` 的别名。

运算符：

* 比较运算符： ``<=`` ， ``<`` ， ``==`` ， ``!=`` ， ``>=`` ， ``>`` （返回布尔值）
* 位运算符： ``&`` ， ``|`` ， ``^`` （异或）， ``~`` （位取反）
* 移位运算符： ``<<`` （左移位） ， ``>>`` （右移位）
* 算数运算符： ``+`` ， ``-`` ， 一元运算 ``-`` ， 一元运算 ``+`` ， ``*`` ， ``/`` ， ``%`` （取余或叫模运算） ， ``**`` （幂）


对于整形 ``X``，可以使用 ``type(X).min`` 和 ``type(X).max`` 去获取这个类型的最小值与最大值。


.. warning::
  Solidity中的整数是有取值范围的。 例如``uint32``类型的取值范围是 ``0``到``2 ** 32-1```。
  如果整数的某些操作的结果不在取值范围内，则会被截断。 这些截断可能会让我们承担的严重后果，进一步参考 :ref:`小心处理溢出问题<underflow-overflow>`.

比较运算
^^^^^^^^^^^

比较整型的值

位运算
^^^^^^^^^^^^^^

位运算在数字的二进制补码表示上执行。
这意味着： ``~int256（0）== int256（-1）``。

移位
^^^^^^

移位操作的结果具有左操作数的类型，同时会截断结果以匹配类型。

 - 不管 ``x`` 正还是负，``x << y`` 相当于 ``x * 2 ** y``。
 - 如果 ``x`` 为正值，``x >> y`` 相当于 ``x / 2 ** y``。
 - 如果 ``x`` 为负值，``x >> y`` 相当于 ``(x + 1) / 2**y - 1`` (与将 ``x`` 除以 ``2**y`` 同时向负无穷大四舍五入)。
 - 在所有情况下，通过负值的 ``y`` 进行移位会引发运行时异常。

.. warning::
  在版本 ``0.5.0`` 之前，对于负 ``x`` 的右移 ``x >> y`` 相当于 ``x / 2 ** y`` ，即，右移使用向上舍入（向零）而不是向下舍入（向负无穷大）。

加、减、乘法运算
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

加法，减法和乘法具有通常的语义，值用两进制补码表示，意思是比如：``uint256（0） -  uint256（1）== 2 ** 256  -  1`` 。 我们在设计和编写智能合约时必须考虑到溢出问题。

表达式 ``-x`` 相当于 ``(T(0) - x)`` 这里 ``T`` 是指 ``x`` 的类型。 这意味着如果 ``x`` 的类型的类型是无符号整数类型 ``-x`` 不会是负数。
另外，如果 ``x`` 为负数， ``-x`` 也可以为正数。 由于两进制补码表示还需要小心::

    int x = -2**255;
    assert(-x == x);

这意味着即使数字是负数，也不能假设它的负数会是正数。


除法运算
^^^^^^^^^^^^

除法运算结果的类型始终是其中一个操作数的类型，整数除法总是产生整数。
在Solidity中，分数会取零。 这意味着 ``int256(-5) / int256(2) == int256(-2)`` 。

注意在智能合约中，在 :ref:`字面常量<rational_literals>` 上进行除法会保留精度（保留小数位）。

.. note::
  除以0 会发生错误（assert 类型错误）。

模运算（取余）
^^^^^^^^^^^^^^^

模运算 ``a％n`` 是在操作数 ``a`` 的除以 ``n`` 之后产生余数 ``r`` ，其中 ``q = int(a / n)`` 和 ``r = a - (n * q)`` 。 这意味着模运算结果与左操作数相同的符号相同（或零）。
对于 负数的a : ``a % n == -(a % n)``， 几个例子：

 * ``int256(5) % int256(2) == int256(1)``
 * ``int256(5) % int256(-2) == int256(1)``
 * ``int256(-5) % int256(2) == int256(-1)``
 * ``int256(-5) % int256(-2) == int256(-1)``

.. note::
  对0取模会发生错误（assert 类型错误）。

幂运算
^^^^^^^^^^^^^^

幂运算仅适用于无符号类型。 结果的类型总是等于基数的类型.
请注意这些类型足够大，以保证能容纳计算结果。


.. note::
  注意 ``0**0`` 在EVM中定义为 ``1`` 。


.. index:: ! ufixed, ! fixed, ! fixed point number

定长浮点型
----------

.. warning::
    Solidity 还没有完全支持定长浮点型。可以声明定长浮点型的变量，但不能给它们赋值或把它们赋值给其他变量。。

``fixed`` / ``ufixed``：表示各种大小的有符号和无符号的定长浮点型。
在关键字 ``ufixedMxN`` 和 ``fixedMxN`` 中，``M`` 表示该类型占用的位数，``N`` 表示可用的小数位数。
``M`` 必须能整除 8，即 8 到 256 位。
``N`` 则可以是从 0 到 80 之间的任意数。
``ufixed`` 和 ``fixed`` 分别是 ``ufixed128x19`` 和 ``fixed128x19`` 的别名。

运算符：

* 比较运算符：``<=``， ``<``， ``==``， ``!=``， ``>=``， ``>`` （返回值是布尔型）
* 算术运算符：``+``， ``-``， 一元运算 ``-``， 一元运算 ``+``， ``*``， ``/``， ``%`` （取余数）

.. note::
    浮点型（在许多语言中的 ``float`` 和 ``double`` 类型，更准确地说是 IEEE 754 类型）和定长浮点型之间最大的不同点是，
    在前者中整数部分和小数部分（小数点后的部分）需要的位数是灵活可变的，而后者中这两部分的长度受到严格的规定。
    一般来说，在浮点型中，几乎整个空间都用来表示数字，但只有少数的位来表示小数点的位置。

.. index:: address, balance, send, call, delegatecall, transfer

.. _address:

地址类型 Address
--------------------

地址类型有两种形式，他们大致相同：

 - ``address``：保存一个20字节的值（以太坊地址的大小）。
 - ``ddress payable`` ：可支付地址，与 ``address`` 相同，不过有成员函数 ``transfer`` 和 ``send`` 。

这种区别背后的思想是 ``address payable`` 可以接受以太币的地址，而一个普通的 ``address`` 则不能。


类型转换:

允许从 ``address payable`` 到 ``address`` 的隐式转换，而从 ``address`` 到 ``address payable`` 必须显示的转换, 通过 ``payable(<address>)`` 进行转换.


（注意:0.5版本时,执行这种转换的唯一方法是使用中间类型，先转换为 ``uint160`` 如,  address payable ap = address(uint160(addr)); )


:ref:`地址字面常量<address_literals>` 可以隐式转换为 ``address payable`` 。

``address`` 可以显式和整型、整型字面常量、``bytes20`` 及合约类型相互转换。转换时需注意：

如果 ``x`` 是整型或定长字节数组、字面常量或具有可支付的回退（ payable fallback  ）函数或 receive 接收函数 的合约类型，则转换形式 ``address(x)`` 的结果是 ``address payable`` 类型。
如果 ``x`` 是没有可支付的回退（ payable fallback ）函数或 receive 接收函数的合约类型，则 ``address(x)`` 将是 ``address`` 类型。
在外部函数签名（定义）中，``address`` 可用来表示 ``address`` 和 ``address payable`` 类型。


只能通过的表达式 ``payable(<address>)`` 将 ``address``  类型转换为 ``address payable`` 类型。


.. note::
    大部分情况下你不需要关心 ``address`` 与 ``address payable`` 之间的区别，并且到处都使用 ``address`` 。 例如，如果你在使用 :ref:`取款模式<withdrawal_pattern>`, 你可以（也应该）保存地址为 ``address`` 类型, 因为可以在
    ``msg.sender`` 对象上调用 ``transfer`` 函数, 因为 ``msg.sender`` 是 ``address payable``。

运算符:

* ``<=``, ``<``, ``==``, ``!=``, ``>=`` and ``>``

.. warning::
    如果将使用较大字节数组类型转换为 ``address`` ，例如 ``bytes32`` ，那么 ``address`` 将被截断。
    为了减少转换歧义，0.4.24及更高编译器版本要求我们在转换中显式截断处理。
    以32bytes值 ``0x111122223333444455556666777788889999AAAABBBBCCCCDDDDEEEEFFFFCCCC`` 为例， 如果使用 ``address(uint160(bytes20(b)))`` 结果是 ``0x111122223333444455556666777788889999aAaa``， 而使用 ``address(uint160(uint256(b)))`` 结果是 ``0x777788889999AaAAbBbbCcccddDdeeeEfFFfCcCc`` 。


.. note::
    ``address`` 和 ``address payable`` 的区别是在 0.5.0 版本引入的，同样从这个版本开始，合约类型不再继承自地址类型，
     不过如果合约有可支付的回退（ payable fallback ）函数或receive 函数，合约类型仍然可以显示转换为
    ``address`` 或 ``address payable`` 。

.. _members-of-addresses:

地址类型成员变量
^^^^^^^^^^^^^^^^
查看所有的成员，可参考 :ref:`address_related`。

* ``balance`` 和 ``transfer``

可以使用 ``balance`` 属性来查询一个地址的余额，
也可以使用 ``transfer`` 函数向一个可支付地址（payable address）发送 |ether| （以 wei 为单位）：

::

    address x = 0x123;
    address myAddress = this;
    if (x.balance < 10 && myAddress.balance >= 10) x.transfer(10);

如果当前合约的余额不够多，则 ``transfer`` 函数会执行失败，或者如果以太转移被接收帐户拒绝， ``transfer`` 函数同样会失败而进行回退。

.. note::
    如果 ``x`` 是一个合约地址，它的代码（更具体来说是, 如果有receive函数,  执行 :ref:`receive-ether-function`, 或者存在fallback函数,执行 :ref:`fallback-function` 函数）会跟 ``transfer`` 函数调用一起执行（这是 EVM 的一个特性，无法阻止）。
    如果在执行过程中用光了 gas 或者因为任何原因执行失败，|ether| 交易会被打回，当前的合约也会在终止的同时抛出异常。

* ``send``

``send`` 是 ``transfer`` 的低级版本。如果执行失败，当前的合约不会因为异常而终止，但 ``send`` 会返回 ``false``。

.. warning::
    在使用 ``send`` 的时候会有些风险：如果调用栈深度是 1024 会导致发送失败（这总是可以被调用者强制），如果接收者用光了 gas 也会导致发送失败。
    所以为了保证 |ether| 发送的安全，一定要检查 ``send`` 的返回值，使用 ``transfer`` 或者更好的办法：
    使用接收者自己取回资金的模式。

* ``call``， ``delegatecall`` 和 ``staticcall``


为了与不符合 |ABI| 的合约交互，或者要更直接地控制编码，提供了函数 ``call``，``delegatecall`` 和 ``staticcall`` 。
它们都带有一个 ``bytes memory`` 参数和返回执行成功状态（``bool``）和数据（``bytes memory``）。

函数 ``abi.encode``，``abi.encodePacked``，``abi.encodeWithSelector`` 和 ``abi.encodeWithSignature`` 可用于编码结构化数据。

例如::

    bytes memory payload = abi.encodeWithSignature("register(string)", "MyName");
    (bool success, bytes memory returnData) = address(nameReg).call(payload);
    require(success);



此外，为了与不符合 |ABI| 的合约交互，于是就有了可以接受任意类型任意数量参数的 ``call`` 函数。
这些参数会被打包到以 32 字节为单位的连续区域中存放。
其中一个例外是当第一个参数被编码成正好 4 个字节的情况。
在这种情况下，这个参数后边不会填充后续参数编码，以允许使用函数签名。

::

    address nameReg = 0x72ba7d8e73fe8eb666ea66babc8116a41bfb10e2;
    nameReg.call("register", "MyName");
    nameReg.call(bytes4(keccak256("fun(uint256)")), a);

.. warning::
    所有这些函数都是低级函数，应谨慎使用。
    具体来说，任何未知的合约都可能是恶意的，我们在调用一个合约的同时就将控制权交给了它，而合约又可以回调合约，所以要准备好在调用返回时改变相应的状态变量（可参考 :ref:`可重入<re_entance>` )，  与其他合约交互的常规方法是在合约对象上调用函数（x.f()）。


.. note::
    0.5.以前版本的 Solidity 允许这些函数接收任意参数，并且还会以不同方式处理 bytes4 类型的第一个参数。 在版本0.5.0中删除了这些边缘情况。

可以使用 ``gas`` |modifier| 调整提供的 gas 数量 ::

    address(nameReg).call{gas: 1000000}(abi.encodeWithSignature("register(string)", "MyName"));

类似地，也能控制提供的 |ether| 的值 ::

    address(nameReg).call{value: 1 ether}(abi.encodeWithSignature("register(string)", "MyName"));

最后一点，这些 |modifier| 可以联合使用。每个修改器出现的顺序不重要 ::

   address(nameReg).call{gas: 1000000, value: 1 ether}(abi.encodeWithSignature("register(string)", "MyName"));

以类似的方式，可以使用函数 ``delegatecall`` ：区别在于只调用给定地址的代码（函数），其他状态属性如（存储，余额 ...）都来自当前合约。 ``delegatecall`` 的目的是使用另一个合约中的库代码。 用户必须确保两个合约中的存储结构都适合委托调用 （delegatecall）。


.. note::
    在以太坊家园（homestead） 之前，只有 ``callcode`` 函数，它无法访问原始的 ``msg.sender`` 和 ``msg.value`` 值。 此函数已在0.5.0版中删除。

从以太坊拜占庭（byzantium）版本开始 提供了 ``staticcall`` ，它与 ``call`` 基本相同，但如果被调用的函数以任何方式修改状态变量，都将回退。

所有三个函数 ``call`` ，``delegatecall`` 和 ``staticcall`` 都是非常低级的函数，应该只把它们当作 *最后一招* 来使用，因为它们破坏了 Solidity 的类型安全性。

所有三种方法都提供 ``gas`` 选项，而 ``delegatecall`` 不支持 ``value`` 选项。

.. note::
    不管是读取状态还是写入状态，最好避免在合约代码中硬编码使用的 gas 值。这可能会引入”陷阱“，而且 gas 的消耗也是可能会改变的。


.. note::
    所有合约都可以转换为 ``address`` 类型，因此可以使用 ``address(this).balance`` 查询当前合约的余额。


.. index:: ! contract type, ! type; contract

.. _contract_types:

合约类型
--------------

每一个 :ref:`contract<contracts>` 定义都有他自己的类型。

您可以隐式地将合约转换为从他们继承的合约。
合约可以显式转换为 ``address`` 类型。

只有当合约具有 接收receive函数 或 payable 回退函数时，才能显式和 ``address payable`` 类型相互转换
转换仍然使用 ``address(x)`` 执行， 如果合约类型没有接收或payable 回退功能，则可以使用 ``payable(address(x))`` 转换为 ``address payable`` 。


可以参考 :ref:`地址类型<address>`.

.. note::
    在版本0.5.0之前，合约直接从地址类型派生的， 并且 ``address`` 和 ``address payable`` 之间没有区别。

如果声明一个合约类型的局部变量（ ``MyContract c`` ），则可以调用该合约的函数。 注意需要赋相同合约类型的值给它。

您还可以实例化合约（即新创建一个合约对象），参考 :ref:`'使用new创建合约'<creating-contracts>`。

合约和 ``address`` 的数据表示是相同的， 参考 :ref:`ABI<ABI>`。

合约不支持任何运算符。

合约类型的成员是合约的外部函数及 public 的 状态变量。


对于合约  ``C`` 可以使用 ``type(C)`` 获取合约的类型信息，参考 :ref:`类型信息<meta-type>` 。

.. index:: byte array, bytes32

定长字节数组
------------

关键字有：``bytes1``， ``bytes2``， ``bytes3``， ...， ``bytes32``。
``byte`` 是 ``bytes1`` 的别名。

运算符：

* 比较运算符：``<=``， ``<``， ``==``， ``!=``， ``>=``， ``>`` （返回布尔型）
* 位运算符： ``&``， ``|``， ``^`` （按位异或）， ``~`` （按位取反）
* 移位运算符： ``<<`` （左移位）， ``>>`` （右移位）
* 索引访问：如果 ``x`` 是 ``bytesI`` 类型，那么 ``x[k]`` （其中 ``0 <= k < I``）返回第 ``k`` 个字节（只读）。

该类型可以和作为右操作数的任何整数类型进行移位运算（但返回结果的类型和左操作数类型相同），右操作数表示需要移动的位数。
进行负数位移运算会引发运行时异常。

成员变量：

* ``.length`` 表示这个字节数组的长度（只读）.

.. note::
    可以将 ``byte[]`` 当作字节数组使用，但这种方式非常浪费存储空间，准确来说，是在传入调用时，每个元素会浪费 31 字节。
    更好地做法是使用 ``bytes``。

变长字节数组
------------

``bytes``:
    变长字节数组，参见 :ref:`arrays`。它并不是值类型。
``string``:
    变长 UTF-8 编码字符串类型，参见 :ref:`arrays`。并不是值类型。

.. index:: address, literal;address

.. _address_literals:

地址字面常量
---------------------------------------

比如像 ``0xdCad3a6d3569DF655070DEd06cb7A1b2Ccd1D3AF`` 这样的通过了地址校验和测试的十六进制字面常量会作为 ``address payable`` 类型。
而没有通过校验测试, 长度在 39 到 41 个数字之间的十六进制字面常量，会产生一个错误,您可以在零前面添加（对于整数类型）或在零后面添加（对于bytesNN类型）以消除错误。

.. note::
    混合大小写的地址校验和格式定义在 `EIP-55 <https://github.com/ethereum/EIPs/blob/master/EIPS/eip-55.md>`_ 中。

.. index:: literal, literal;rational

.. _rational_literals:

有理数和整数字面常量
----------------------------

整数字面常量由范围在 0-9 的一串数字组成，表现成十进制。
例如，`69` 表示数字 69。
Solidity 中是没有八进制的，因此前置 0 是无效的。

十进制小数字面常量带有一个 ``.``，至少在其一边会有一个数字。
比如：``1.``，``.1``，和 ``1.3``。

科学符号也是支持的，尽管指数必须是整数，但底数可以是小数。
比如：``2e10``， ``-2e10``， ``2e-10``， ``2.5e1``。


为了提高可读性可以在数字之间加上下划线。
例如，十进制``123_000``，十六进制 ``0x2eff_abde``，科学十进制表示1_2e345_678都是有效的。
下划线仅允许在两位数之间，并且不允许下划线连续出现。添加到数字文字中下划线没有额外的语义，下划线会被编译器忽略。

数值字面常量表达式本身支持任意精度，除非它们被转换成了非字面常量类型（也就是说，当它们出现在变量表达式中时就会发生转换）。
这意味着在数值常量表达式中, 计算不会溢出而除法也不会截断。

例如， ``(2**800 + 1) - 2**800`` 的结果是字面常量 ``1`` （属于 ``uint8`` 类型），尽管计算的中间结果已经超过了 |evm| 的机器字长度。
此外， ``.5 * 8`` 的结果是整型 ``4`` （尽管有非整型参与了计算）。

只要操作数是整型，任意整型支持的运算符都可以被运用在数值字面常量表达式中。
如果两个中的任一个数是小数，则不允许进行位运算。如果指数是小数的话，也不支持幂运算（因为这样可能会得到一个无理数）。

.. note::
    Solidity 对每个有理数都有对应的数值字面常量类型。
    整数字面常量和有理数字面常量都属于数值字面常量类型。
    除此之外，所有的数值字面常量表达式（即只包含数值字面常量和运算符的表达式）都属于数值字面常量类型。
    因此数值字面常量表达式 ``1 + 2`` 和 ``2 + 1`` 的结果跟有理数三的数值字面常量类型相同。

.. warning::
    在早期版本中（0.4.0之前），整数字面常量的除法也会截断，但在现在的版本中，会将结果转换成一个有理数。即 ``5 / 2`` 并不等于 ``2``，而是等于 ``2.5``。

.. note::
    数值字面常量表达式只要在非字面常量表达式中使用就会转换成非字面常量类型。
    在下面的例子中，尽管我们知道 ``b`` 的值是一个整数，但 ``2.5 + a`` 这部分表达式并不进行类型检查，因此编译不能通过。

::

    uint128 a = 1;
    uint128 b = 2.5 + a + 0.5;

.. index:: literal, literal;string, string

.. _string_literals:

字符串字面常量及类型
---------------------

字符串字面常量是指由双引号或单引号引起来的字符串（``"foo"`` 或者 ``'bar'``）。
它们也可以分为多个连续的部分（``"foo" "bar"``  等效于``"foobar"``），这在处理长字符串时很有用。
不像在 C 语言中那样带有结束符；``"foo"`` 相当于 3 个字节而不是 4 个。
和整数字面常量一样，字符串字面常量的类型也可以发生改变，

但它们可以隐式地转换成 ``bytes1``，……，``bytes32``，如果合适的话，还可以转换成 ``bytes`` 以及 ``string``。

例如： ``bytes32 samevar = "stringliteral"`` 字符串字面常量在赋值给 ``bytes32`` 时被解释为原始的字节形式。


字符串字面常量支持下面的转义字符：

 - ``\<newline>`` (转义实际换行)
 - ``\\`` (反斜杠)
 - ``\'`` (单引号)
 - ``\"`` (双引号)
 - ``\b`` (退格)
 - ``\f`` (换页)
 - ``\n`` (换行符)
 - ``\r`` (回车)
 - ``\t`` (标签 tab)
 - ``\v`` (垂直标签)
 - ``\xNN`` (十六进制转义，见下文)
 - ``\uNNNN`` (unicode 转义，见下文)


``\xNN`` 表示一个 16 进制值，最终转换成合适的字节，而 ``\uNNNN`` 表示 Unicode 编码值，最终会转换为 UTF-8 的序列。

以下示例中的字符串长度为十个字节，它以换行符开头，后跟双引号，单引号，反斜杠字符，以及（没有分隔符）字符序列 ``abcdef`` 。

::

    "\n\"\'\\abc\
    def"


任何unicode行终结符（即LF，VF，FF，CR，NEL，LS，PS）都不会被当成字符串字面常量的终止符。 如果前面没有前置 ``\``，则换行符仅终止字符串字面常量。


.. index:: literal, bytes

十六进制字面常量
---------------------

十六进制字面常量以关键字 ``hex`` 打头，后面紧跟着用单引号或双引号引起来的字符串（例如，``hex"001122FF"`` ）。
字符串的内容必须是一个十六进制的字符串，它们的值将使用二进制表示。

它们的内容必须是十六进制数字，可以选择使用单个下划线作为字节边界分隔符。 字面常量的值将是十六进制序列的二进制表示形式。

用空格分隔的多个十六进制字面常量被合并为一个字面常量：
``hex"00112233" hex"44556677"`` 等同于 ``hex"0011223344556677"``


十六进制字面常量跟 :ref:`字符串字面常量 <string_literals>` 很类似，具有相同的转换规则

.. index:: enum

.. _enums:

枚举类型
----------------

枚举是在Solidity中创建用户定义类型的一种方法。 它们是显示所有整型相互转换，但不允许隐式转换。 从整型显式转换枚举，会在运行时检查整数时候在枚举范围内，否则会导致异常（ :ref:`assert 类型异常 <assert-and-require>` ）。
枚举需要至少一个成员,默认值是第一个成员.

数据表示与C中的枚举相同：选项从“0”开始的无符号整数值表示。

::

    pragma solidity >=0.4.16 <0.7.0;

    contract test {
        enum ActionChoices { GoLeft, GoRight, GoStraight, SitStill }
        ActionChoices choice;
        ActionChoices constant defaultChoice = ActionChoices.GoStraight;

        function setGoStraight() public {
            choice = ActionChoices.GoStraight;
        }

        // 由于枚举类型不属于 |ABI| 的一部分，因此对于所有来自 Solidity 外部的调用，
        // "getChoice" 的签名会自动被改成 "getChoice() returns (uint8)"。
        // 整数类型的大小已经足够存储所有枚举类型的值，随着值的个数增加，
        // 可以逐渐使用 `uint16` 或更大的整数类型。
        function getChoice() public view returns (ActionChoices) {
            return choice;
        }

        function getDefaultChoice() public pure returns (uint) {
            return uint(defaultChoice);
        }
    }

.. 注意::
     枚举还可以在合约或库定义之外的文件级别上声明。

.. index:: ! function type, ! type; function

.. _function_types:

函数类型
----------------

函数类型是一种表示函数的类型。可以将一个函数赋值给另一个函数类型的变量，也可以将一个函数作为参数进行传递，还能在函数调用中返回函数类型变量。
函数类型有两类：
 - *内部（internal）* 函数类型
 - *外部（external）* 函数类型

内部函数只能在当前合约内被调用（更具体来说，在当前代码块内，包括内部库函数和继承的函数中），因为它们不能在当前合约上下文的外部被执行。
调用一个内部函数是通过跳转到它的入口标签来实现的，就像在当前合约的内部调用一个函数。

外部函数由一个地址和一个函数签名组成，可以通过外部函数调用传递或者返回。

函数类型表示成如下的形式 ::

    function (<parameter types>) {internal|external} [pure|constant|view|payable] [returns (<return types>)]

与参数类型相反，返回类型不能为空 —— 如果函数类型不需要返回，则需要删除整个 ``returns (<return types>)`` 部分。

函数类型默认是内部函数，因此不需要声明 ``internal`` 关键字。
 请注意，这仅适用于函数类型，合约中定义的函数明确指定可见性，它们没有默认值。

类型转换：


函数类型 ``A`` 可以隐式转换为函数类型 ``B`` 当且仅当:
它们的参数类型相同，返回类型相同，它们的内部/外部属性是相同的，并且 ``A`` 的状态可变性并不比 ``B`` 的状态可变性更具限制性，比如：

 - ``pure`` 函数可以转换为 ``view`` 和 ``non-payable`` 函数
 - ``view`` 函数可以转换为 ``non-payable`` 函数
 - ``payable`` 函数可以转换为 ``non-payable`` 函数

其他的转换则不可以。

关于 ``payable`` 和 ``non-payable`` 的规则可能有点令人困惑，但实质上，如果一个函数是 ``payable`` ，这意味着它
也接受零以太的支付，因此它也是 ``non-payable`` 。
另一方面，``non-payable`` 函数将拒绝发送给它的 |ether| ，
所以 ``non-payable`` 函数不能转换为 ``payable`` 函数。


如果当函数类型的变量还没有初始化时就调用它的话会引发一个异常。
如果在一个函数被 ``delete`` 之后调用它也会发生相同的情况。

如果外部函数类型在 Solidity 的上下文环境以外的地方使用，它们会被视为 ``function`` 类型。
该类型将函数地址紧跟其函数标识一起编码为一个 ``bytes24`` 类型。。

请注意，当前合约的 public 函数既可以被当作内部函数也可以被当作外部函数使用。
如果想将一个函数当作内部函数使用，就用 ``f`` 调用，如果想将其当作外部函数，使用 ``this.f`` 。

成员方法：

public（或 external）函数都有下面的成员：

* ``.address`` 返回函数的合约地址。
* ``.selector`` 返回 :ref:`ABI 函数选择器 <abi_function_selector>`
* ``.gas(uint)`` 返回一个可调用的函数对象，当被调用时，它将指定函数运行的Gas。参考 :ref:`外部函数调用 <external-function-calls>` 了解更多。 弃用,用 ``{gas: ...}`` 代替
* ``.value(uint)`` 返回一个可调用的函数对象，当被调用时，它将向目标函数发送指定数量的 |ether| （单位 wei）。 弃用,用 ``{value: ...}`` 代替 参考 :ref:`外部函数调用 <external-function-calls>` 了解更多。

下面的例子，显示如何使用成员::

    pragma solidity >=0.4.16 <0.7.0;
    // This will report a warning

    contract Example {
      function f() public payable returns (bytes4) {
        assert(this.f.address == address(this));
        return this.f.selector;
      }
      function g() public {
        this.f{gas: 10, value: 800}()
        // 新语法是
        // this.f{gas: 10, value: 800}()
      }
    }

如果使用内部函数类型的例子::

    pragma solidity >=0.4.16 <0.7.0;

    library ArrayUtils {
      // 内部函数可以在内部库函数中使用，
      // 因为它们会成为同一代码上下文的一部分
      function map(uint[] memory self, function (uint) pure returns (uint) f)
        internal
        pure
        returns (uint[] memory r)
      {
        r = new uint[](self.length);
        for (uint i = 0; i < self.length; i++) {
          r[i] = f(self[i]);
        }
      }
      function reduce(
        uint[] memory self,
        function (uint, uint) pure returns (uint) f
      )
        internal
        pure
        returns (uint r)
      {
        r = self[0];
        for (uint i = 1; i < self.length; i++) {
          r = f(r, self[i]);
        }
      }
      function range(uint length) internal pure returns (uint[] memory r) {
        r = new uint[](length);
        for (uint i = 0; i < r.length; i++) {
          r[i] = i;
        }
      }
    }

    contract Pyramid {
      using ArrayUtils for *;
      function pyramid(uint l) public pure returns (uint) {
        return ArrayUtils.range(l).map(square).reduce(sum);
      }
      function square(uint x) internal pure returns (uint) {
        return x * x;
      }
      function sum(uint x, uint y) internal pure returns (uint) {
        return x + y;
      }
    }

另外一个使用外部函数类型的例子::

    pragma solidity >=0.4.22 <0.7.0;

    contract Oracle {
      struct Request {
        bytes data;
        function(uint) external callback;
      }
      Request[] private requests;
      event NewRequest(uint);
      function query(bytes memory data, function(uint) external callback) public {
        requests.push(Request(data, callback));
        emit NewRequest(requests.length - 1);
      }
      function reply(uint requestID, uint response) public {
        // 这里检查回复来自可信来源
        requests[requestID].callback(response);
      }
    }

    contract OracleUser {
      Oracle constant private ORACLE_CONST = Oracle(0x1234567); // known contract
      uint private exchangeRate;
      function buySomething() public {
        ORACLE_CONST.query("USD", this.oracleResponse);
      }
      function oracleResponse(uint response) public {
        require(
            msg.sender == address(ORACLE_CONST),
            "Only oracle can call this."
        );
        exchangeRate = response;
      }
    }

.. note::
    Lambda 表达式或者内联函数的引入在计划内，但目前还没支持。
