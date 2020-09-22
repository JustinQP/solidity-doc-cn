
.. index:: ! library, callcode, delegatecall

.. _libraries:

************
库
************

库与合约类似，它们只需要在特定的地址部署一次，并且它们的代码可以通过 EVM 的 ``DELEGATECALL``
(Homestead 之前使用 ``CALLCODE`` 关键字)特性进行重用。
这意味着如果库函数被调用，它的代码在调用合约的上下文中执行，即 ``this`` 指向调用合约，特别是可以访问调用合约的存储。
因为每个库都是一段独立的代码，所以它仅能访问调用合约明确提供的状态变量（否则它就无法通过名字访问这些变量）。
因为我们假定库是无状态的，所以如果它们不修改状态（也就是说，如果它们是 ``view`` 或者 ``pure`` 函数），
库函数仅可以通过直接调用来使用（即不使用 ``DELEGATECALL`` 关键字），
特别是，除非能规避 Solidity 的类型系统，否则是不可能销毁任何库的。

库可以看作是使用他们的合约的隐式的基类合约。虽然它们在继承关系中不会显式可见，但调用库函数与调用显式的基类合约十分类似
（如果 ``L`` 是库的话，可以使用 ``L.f()`` 调用库函数）。此外，就像库是基类合约一样，对所有使用库的合约，库的 ``internal`` 函数都是可见的。
当然，需要使用内部调用约定来调用内部函数，这意味着所有内部类型，内存类型都是通过引用而不是复制来传递。
为了在 EVM 中实现这些，内部库函数的代码和从其中调用的所有函数都在编译阶段被包含到调用合约中，然后使用一个 ``JUMP`` 调用来代替 ``DELEGATECALL``。


.. index:: using for, set

下面的示例说明如何使用库（但也请务必看看 :ref:`using for <using-for>` 有一个实现 set 更好的例子）。

::

    pragma solidity >=0.6.0 <0.7.0;

      // 我们定义了一个新的结构体数据类型，用于在调用合约中保存数据。
      struct Data {
        mapping(uint => bool) flags;
      }

    library Set {

      // 注意第一个参数是“storage reference”类型，因此在调用中参数传递的只是它的存储地址而不是内容。
      // 这是库函数的一个特性。如果该函数可以被视为对象的方法，则习惯称第一个参数为 `self` 。
      function insert(Data storage self, uint value)
          public
          returns (bool)
      {
          if (self.flags[value])
              return false; // 已经存在
          self.flags[value] = true;
          return true;
      }

      function remove(Data storage self, uint value)
          public
          returns (bool)
      {
          if (!self.flags[value])
              return false; // 不存在
          self.flags[value] = false;
          return true;
      }

      function contains(Data storage self, uint value)
          public
          view
          returns (bool)
      {
          return self.flags[value];
      }
    }

    contract C {
        Data knownValues;

        function register(uint value) public {
            // 不需要库的特定实例就可以调用库函数，
            // 因为当前合约就是“instance”。
            require(Set.insert(knownValues, value));
        }
        // 如果我们愿意，我们也可以在这个合约中直接访问 knownValues.flags。
    }

当然，你不必按照这种方式去使用库：它们也可以在不定义结构数据类型的情况下使用。
函数也不需要任何存储引用参数，库可以出现在任何位置并且可以有多个存储引用参数。

调用 ``Set.contains``，``Set.insert`` 和 ``Set.remove`` 都被编译为外部调用（ ``DELEGATECALL`` ）。
如果使用库，请注意实际执行的是外部函数调用。
``msg.sender``， ``msg.value`` 和 ``this`` 在调用中将保留它们的值，
（在 Homestead 之前，因为使用了 ``CALLCODE``，改变了 ``msg.sender`` 和 ``msg.value``)。

以下示例展示了如何在库中使用内存类型和内部函数来实现自定义类型，而无需支付外部函数调用的开销：

::

    pragma solidity >=0.6.0;

    struct bigint {
        uint[] limbs;
    }

    library BigInt {

        function fromUint(uint x) internal pure returns (bigint r) {
            r.limbs = new uint[](1);
            r.limbs[0] = x;
        }

        function add(bigint _a, bigint _b) internal pure returns (bigint r) {
            r.limbs = new uint[](max(_a.limbs.length, _b.limbs.length));
            uint carry = 0;
            for (uint i = 0; i < r.limbs.length; ++i) {
                uint a = limb(_a, i);
                uint b = limb(_b, i);
                r.limbs[i] = a + b + carry;
                if (a + b < a || (a + b == uint(-1) && carry > 0))
                    carry = 1;
                else
                    carry = 0;
            }
            if (carry > 0) {
                // 太差了，我们需要增加一个 limb
                uint[] memory newLimbs = new uint[](r.limbs.length + 1);
                for (i = 0; i < r.limbs.length; ++i)
                    newLimbs[i] = r.limbs[i];
                newLimbs[i] = carry;
                r.limbs = newLimbs;
            }
        }

        function limb(bigint _a, uint _limb) internal pure returns (uint) {
            return _limb < _a.limbs.length ? _a.limbs[_limb] : 0;
        }

        function max(uint a, uint b) private pure returns (uint) {
            return a > b ? a : b;
        }
    }

    contract C {
        using BigInt for bigint;

        function f() public pure {
            bigint memory x = BigInt.fromUint(7);
            bigint memory y = BigInt.fromUint(uint(-1));
            bigint memory z = x.add(y);
            assert(z.limb(1) > 0);
        }
    }


可以通过类型转换, 将库类型更改为 ``address`` 类型, 例如: 使用  ``address(LibraryName)``


由于编译器无法知道库的部署位置，我们需要通过链接器将这些地址填入最终的字节码中
（请参阅 :ref:`commandline-compiler` 以了解如何使用命令行编译器来链接字节码）。
如果这些地址没有作为参数传递给编译器，编译后的十六进制代码将包含 ``__Set______`` 形式的占位符（其中 ``Set`` 是库的名称）。
可以手动填写地址来将那 40 个字符替换为库合约地址的十六进制编码。

与合约相比，库的限制：

- 没有状态变量
- 不能够继承或被继承
- 不能接收以太币
- 不可以被销毁 destroyed

（将来有可能会解除这些限制）

.. _library-selectors:

库的函数签名与选择器
==============================================

尽管可以对 public 或 external 的库函数进行外部调用，但此类调用会被视为Solidity的内部调用，与常规的 :ref:`contract ABI<ABI>` 规则不同。

外部库函数比外部合约函数支持更多的参数类型，例如递归结构和指向存储的指针。

因此，计算用于计算4字节选择器的函数签名遵循内部命名模式以及可对合约ABI中不支持的类型的参数使用内部编码。


以下标识符可以作为函数签名中的类型：

 - 值类型, 非存储的（non-storage） ``string`` 及非存储的 ``bytes`` 使用和合约 ABI 中同样的标识符。
 - 非存储的数组类型遵循合约 ABI 中同样的规则，例如 ``<type>[]`` 为动态数组以及 ``<type>[M]`` 为``M``个元素的动态数组。
 - 非存储的结构体使用完整的命名引用，例如 ``C.S`` 用于 ``contract C { struct S { ... } }``.
 - 指向存储的指针类型使用其对应的非存储类型的类型标识符，但在其后面附加一个空格及``storage``。


除了指向存储的指针以外，参数编码与常规合约ABI相同，存储指针被编码为``uint256``值，指向它们所指向的存储插槽。


与合约 ABI 相似，选择器由签名的Keccak256哈希的前四个字节组成。可以使用 .selector 成员从Solidity中获取其值，如下所示：

::

    pragma solidity >=0.5.14 <0.7.0;

    library L {
        function f(uint256) external {}
    }

    contract C {
        function g() public pure returns (bytes4) {
            return L.f.selector;
        }
    }



库的调用保护
=============================

如果库的代码是通过 ``CALL`` 来执行，而不是 ``DELEGATECALL`` 或者 ``CALLCODE`` 那么执行的结果会被回退，
除非是对 ``view`` 或者 ``pure`` 函数的调用。

EVM 没有为合约提供检测是否使用 ``CALL`` 的直接方式，但是合约可以使用 ``ADDRESS`` 操作码找出正在运行的“位置”。
生成的代码通过比较这个地址和构造时的地址来确定调用模式。

更具体地说，库的运行时代码总是从一个 push 指令开始，它在编译时是 20 字节的零。当运行部署代码时，这个常数
被内存中的当前地址替换，修改后的代码存储在合约中。在运行时，部署时地址就成为了第一个被 push 到堆栈上的常数，
对于任何 non-view 和 non-pure 函数，调度器代码都将对比当前地址与这个常数是否一致。

这意味着库在链上存储的实际代码与编译器输出的 ``deployedBytecode`` 的编码是不同。

