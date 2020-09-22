******************
使用编译器
******************

.. index:: ! commandline compiler, compiler;commandline, ! solc, ! linker

.. _commandline-compiler:

使用命令行编译器
******************************

.. note:: 这一节并不适用于 :ref:`solcjs <solcjs>`


``solc`` 是 Solidity 源码库的构建目标之一，它是 Solidity 的命令行编译器。你可使用 ``solc --help`` 命令来查看它的所有选项的解释。该编译器可以生成各种输出，范围从简单的二进制文件、汇编文件到用于估计“gas”使用情况的抽象语法树（解析树）。如果你只想编译一个文件，你可以运行 ``solc --bin sourceFile.sol`` 来生成二进制文件。如果你想通过 ``solc`` 获得一些更高级的输出信息，可以通过 ``solc -o outputDirectory --bin --ast --asm sourceFile.sol`` 命令将所有的输出都保存到一个单独的文件夹中。

Before you deploy your contract, activate the optimizer while compiling using ``solc --optimize --bin sourceFile.sol``. By default, the optimizer will optimize the contract for 200 runs. If you want to optimize for initial contract deployment and get the smallest output, set it to ``--runs=1``. If you expect many transactions and don't care for higher deployment cost and output size, set ``--runs`` to a high number.

命令行编译器会自动从文件系统中读取并导入的文件，但同时，它也支持通过 ``prefix=path`` 选项将路径重定向。比如：

::

    solc github.com/ethereum/dapp-bin/=/usr/local/lib/dapp-bin/ =/usr/local/lib/fallback file.sol

这实质上是告诉编译器去搜索 ``/usr/local/lib/dapp-bin`` 目录下的所有以 ``github.com/ethereum/dapp-bin/`` 开头的文件，如果编译器找不到这样的文件，它会接着读取 ``/usr/local/lib/fallback`` 目录下的所有文件（空前缀意味着始终匹配）。``solc`` 不会从位于重定向目标之外和显式指定的源文件所在目录之外的文件系统读取文件，所以，类似 ``import "/etc/passwd";`` 这样的语句，编译器只会在你添加了 ``=/`` 选项之后，才会尝试到根目录下加载 ``/etc/passwd`` 文件。

如果重定向路径下存在多个匹配，则选择具有最长公共前缀的那个匹配。

When accessing the filesystem to search for imports, all paths are treated as if they were fully qualified paths.
This behaviour can be customized by adding the command line option ``--base-path`` with a path to be prepended
before each filesystem access for imports is performed. Furthermore, the part added via ``--base-path``
will not appear in the contract metadata.


出于安全原因，编译器限制了它可以访问的目录。在命令行中指定的源文件的路径（及其子目录）和通过重定向定义的路径可用于 ``import`` 语句，其他的则会被拒绝。额外路径（及其子目录）可以通过  ``--allow-paths /sample/path,/another/sample/path`` 进行配置。

Everything inside the path specified via ``--base-path`` is always allowed.

如果您的合约使用 :ref:`libraries <libraries>` ，您会注意到在编译后的十六进制字节码中会包含形如 ``__LibraryName____`` 的字符串。当您将 ``solc`` 作为链接器使用时，它会在下列情况中为你插入库的地址：要么在命令行中添加 ``--libraries "Math:0x12345678901234567890 Heap:0xabcdef0123456"`` 来为每个库提供地址，或者将这些字符串保存到一个文件中（每行一个库），并使用 ``--libraries fileName`` 参数。

如果在调用 ``solc`` 命令时使用了 ``--link`` 选项，则所有的输入文件会被解析为上面提到过的  ``__LibraryName____`` 格式的未链接的二进制数据（十六进制编码），并且就地链接。（如果输入是从stdin读取的，则生成的数据会被写入stdout）。在这种情况下，除了 ``--libraries`` 外的其他选项（包括 ``-o`` ）都会被忽略。

如果在调用 ``solc`` 命令时使用了 ``--standard-json`` 选项，它将会按JSON格式解析标准输入上的输入，并在标准输出上返回JSON格式的输出。
This is the recommended interface for more complex and especially automated uses. The process will always terminate in a "success" state and report any errors via the JSON output.
The option ``--base-path`` is also processed in standard-json mode.

.. note::
    The library placeholder used to be the fully qualified name of the library itself
    instead of the hash of it. This format is still supported by ``solc --link`` but
    the compiler will no longer output it. This change was made to reduce
    the likelihood of a collision between libraries, since only the first 36 characters
    of the fully qualified library name could be used.

.. _evm-version:
.. index:: ! EVM version, compile target

Setting the EVM version to target
*********************************

When you compile your contract code you can specify the Ethereum virtual machine
version to compile for to avoid particular features or behaviours.

.. warning::

   Compiling for the wrong EVM version can result in wrong, strange and failing
   behaviour. Please ensure, especially if running a private chain, that you
   use matching EVM versions.

On the command line, you can select the EVM version as follows:

.. code-block:: shell

  solc --evm-version <VERSION> contract.sol

In the :ref:`standard JSON interface <compiler-api>`, use the ``"evmVersion"``
key in the ``"settings"`` field:

.. code-block:: none

  {
    "sources": { ... },
    "settings": {
      "optimizer": { ... },
      "evmVersion": "<VERSION>"
    }
  }

Target options
--------------

Below is a list of target EVM versions and the compiler-relevant changes introduced
at each version. Backward compatibility is not guaranteed between each version.

- ``homestead``
   - (oldest version)
- ``tangerineWhistle``
   - Gas cost for access to other accounts increased, relevant for gas estimation and the optimizer.
   - All gas sent by default for external calls, previously a certain amount had to be retained.
- ``spuriousDragon``
   - Gas cost for the ``exp`` opcode increased, relevant for gas estimation and the optimizer.
- ``byzantium``
   - Opcodes ``returndatacopy``, ``returndatasize`` and ``staticcall`` are available in assembly.
   - The ``staticcall`` opcode is used when calling non-library view or pure functions, which prevents the functions from modifying state at the EVM level, i.e., even applies when you use invalid type conversions.
   - It is possible to access dynamic data returned from function calls.
   - ``revert`` opcode introduced, which means that ``revert()`` will not waste gas.
- ``constantinople``
   - Opcodes ``create2`, ``extcodehash``, ``shl``, ``shr`` and ``sar`` are available in assembly.
   - Shifting operators use shifting opcodes and thus need less gas.
- ``petersburg``
   - The compiler behaves the same way as with constantinople.
- ``istanbul`` (**default**)
   - Opcodes ``chainid`` and ``selfbalance`` are available in assembly.
- ``berlin`` (**experimental**)

.. _compiler-api:

编译器输入输出JSON描述
******************************************


下面展示的这些JSON格式是编译器API使用的，当然，在 ``solc`` 上也是可用的。有些字段是可选的（参见注释），并且它们可能会发生变化，但所有的变化都应该是后向兼容的。

编译器API需要JSON格式的输入，并以JSON格式输出编译结果。

注释是不允许的，这里仅用于解释目的。

输入说明
-----------------

.. code-block:: none

    {
      // 必选: 源代码语言，比如“Solidity”，“serpent”，“lll”，“assembly”等
      language: "Solidity",
      // 必选
      sources:
      {
        // 这里的键值是源文件的“全局”名称，可以通过remappings引入其他文件（参考下文）
        "myFile.sol":
        {
          // 可选: 源文件的kaccak256哈希值，可用于校验通过URL加载的内容。
          "keccak256": "0x123...",
          // 必选（除非声明了 "content" 字段）: 指向源文件的URL。
          // URL(s) 会按顺序加载，并且结果会通过keccak256哈希值进行检查（如果有keccak256的话）
          // 如果哈希值不匹配，或者没有URL返回成功，则抛出一个异常。
          "urls":
          [
            "bzzr://56ab...",
            "ipfs://Qma...",
            "file:///tmp/path/to/file.sol"
          ]
        },
        "mortal":
        {
          // 可选: 该文件的keccak256哈希值
          "keccak256": "0x234...",
          // 必选（除非声明了 "urls" 字段）: 源文件的字面内容
          "content": "contract mortal is owned { function kill() { if (msg.sender == owner) selfdestruct(owner); } }"
        }
      },
      // 可选
      settings:
      {
        // 可选: 重定向参数的排序列表
        remappings: [ ":g/dir" ],
        // 可选: 优化器配置
        optimizer: {
          // 默认为 disabled
          enabled: true,
          // 基于你希望运行多少次代码来进行优化。
          // 较小的值可以使初始部署的费用得到更多优化，较大的值可以使高频率的使用得到优化。
          runs: 200,
          // Switch optimizer components on or off in detail.
          // The "enabled" switch above provides two defaults which can be
          // tweaked here. If "details" is given, "enabled" can be omitted.
          "details": {
            // The peephole optimizer is always on if no details are given,
            // use details to switch it off.
            "peephole": true,
            // The unused jumpdest remover is always on if no details are given,
            // use details to switch it off.
            "jumpdestRemover": true,
            // Sometimes re-orders literals in commutative operations.
            "orderLiterals": false,
            // Removes duplicate code blocks
            "deduplicate": false,
            // Common subexpression elimination, this is the most complicated step but
            // can also provide the largest gain.
            "cse": false,
            // Optimize representation of literal numbers and strings in code.
            "constantOptimizer": false,
            // The new Yul optimizer. Mostly operates on the code of ABIEncoderV2
            // and inline assembly.
            // It is activated together with the global optimizer setting
            // and can be deactivated here.
            // Before Solidity 0.6.0 it had to be activated through this switch.
            "yul": false,
            // Tuning options for the Yul optimizer.
            "yulDetails": {
              // Improve allocation of stack slots for variables, can free up stack slots early.
              // Activated by default if the Yul optimizer is activated.
              "stackAllocation": true,
              // Select optimization steps to be applied.
              // Optional, the optimizer will use the default sequence if omitted.
              "optimizerSteps": "dhfoDgvulfnTUtnIf..."
            }
        },
        // 指定需编译的EVM的版本。会影响代码的生成和类型检查。可用的版本为：homestead，tangerineWhistle，spuriousDragon，byzantium，constantinople
        evmVersion: "byzantium",
        
        // Optional: Debugging settings
        "debug": {
          // How to treat revert (and require) reason strings. Settings are
          // "default", "strip", "debug" and "verboseDebug".
          // "default" does not inject compiler-generated revert strings and keeps user-supplied ones.
          // "strip" removes all revert strings (if possible, i.e. if literals are used) keeping side-effects
          // "debug" injects strings for compiler-generated internal reverts, implemented for ABI encoders V1 and V2 for now.
          // "verboseDebug" even appends further information to user-supplied revert strings (not yet implemented)
          "revertStrings": "default"
        }
        // 可选: 元数据配置
        metadata: {
          // 只可使用字面内容，不可用URLs （默认设为 false）
          useLiteralContent: true,
          // Use the given hash method for the metadata hash that is appended to the bytecode.
          // The metadata hash can be removed from the bytecode via option "none".
          // The other options are "ipfs" and "bzzr1".
          // If the option is omitted, "ipfs" is used by default.
          "bytecodeHash": "ipfs"
        },
        // 库的地址。如果这里没有把所有需要的库都给出，会导致生成输出数据不同的未链接对象
        libraries: {
          // 最外层的 key 是使用这些库的源文件的名字。
          // 如果使用了重定向， 在重定向之后，这些源文件应该能匹配全局路径
          // 如果源文件的名字为空，则所有的库为全局引用
          "myFile.sol": {
            "MyLib": "0x123123..."
          }
        }
        // 以下内容可以用于选择所需的输出。
        // 如果这个字段被忽略，那么编译器会加载并进行类型检查，但除了错误之外不会产生任何输出。
        // 第一级的key是文件名，第二级是合约名称，如果合约名为空，则针对文件本身（进行输出）。
        // 若使用通配符*，则表示所有合约。
        //
        // 可用的输出类型如下所示：
        //   abi - ABI
        //   ast - 所有源文件的AST
        //   legacyAST - 所有源文件的legacy AST
        //   devdoc - 开发者文档（natspec）
        //   userdoc - 用户文档（natspec）
        //   metadata - 元数据
        //   ir - 去除语法糖（desugaring）之前的新汇编格式
        //   irOptimized - Intermediate representation after optimization
        //   storageLayout - Slots, offsets and types of the contract's state variables.
        //   evm.assembly - 去除语法糖（desugaring）之后的新汇编格式
        //   evm.legacyAssembly - JSON的旧样式汇编格式
        //   evm.bytecode.object - 字节码对象
        //   evm.bytecode.opcodes - 操作码列表
        //   evm.bytecode.sourceMap - 源码映射（用于调试）
        //   evm.bytecode.linkReferences - 链接引用（如果是未链接的对象）
        //   evm.deployedBytecode* - 部署的字节码（具有evm.bytecode所有的选项）
        //   evm.deployedBytecode.immutableReferences - Map from AST ids to bytecode ranges that reference immutables
        //   evm.methodIdentifiers - 函数哈希值列表
        //   evm.gasEstimates - 函数的gas预估量
        //   ewasm.wast - eWASM S-expressions 格式（不支持atm）
        //   ewasm.wasm - eWASM二进制格式（不支持atm）
        //
        // 请注意，如果使用 `evm` ，`evm.bytecode` ，`ewasm` 等选项，会选择其所有的子项作为输出。 另外，`*`可以用作通配符来请求所有内容。
        //
        outputSelection: {
          // 为每个合约生成元数据和字节码输出。
          "*": {
            "*": [ "metadata"，"evm.bytecode" ]
          },
          // 启用“def”文件中定义的“MyContract”合约的abi和opcodes输出。
          "def": {
            "MyContract": [ "abi"，"evm.bytecode.opcodes" ]
          },
          // 为每个合约生成源码映射输出
          "*": {
            "*": [ "evm.bytecode.sourceMap" ]
          },
          // 每个文件生成legacy AST输出
          "*": {
            "": [ "legacyAST" ]
          }
        }
      }
    }


输出说明
------------------

.. code-block:: none

    {
      // 可选：如果没有遇到错误/警告，则不出现
      errors: [
        {
          // 可选：源文件中的位置
          sourceLocation: {
            file: "sourceFile.sol",
            start: 0,
            end: 100
          ],
        // Optional: Further locations (e.g. places of conflicting declarations)
          secondarySourceLocations: [
            {
              "file": "sourceFile.sol",
              "start": 64,
              "end": 92,
              "message": "Other declaration is here:"
            }
          ],
          // 强制: 错误类型，例如 “TypeError”， “InternalCompilerError”， “Exception”等.
          // 可在文末查看完整的错误类型列表
          type: "TypeError",
          // 强制: 发生错误的组件，例如“general”，“ewasm”等
          component: "general",
          // 强制：错误的严重级别（“error”或“warning”）
          severity: "error",
          // 可选: 引起错误的唯一编码
          "errorCode": "3141",
          // 强制
          message: "Invalid keyword",
          // 可选: 带错误源位置的格式化消息
          formattedMessage: "sourceFile.sol:100: Invalid keyword"
        }
      ],
      // 这里包含了文件级别的输出。可以通过outputSelection来设置限制/过滤。
      sources: {
        "sourceFile.sol": {
          // 标识符（用于源码映射）
          id: 1,
          // AST对象
          ast: {},
          // legacy AST 对象
          legacyAST: {}
        }
      },
      // 这里包含了合约级别的输出。 可以通过outputSelection来设置限制/过滤。
      contracts: {
        "sourceFile.sol": {
          // 如果使用的语言没有合约名称，则该字段应该留空。
          "ContractName": {
            // 以太坊合约的应用二进制接口（ABI）。如果为空，则表示为空数组。
            // 请参阅 https://github.com/ethereum/wiki/wiki/Ethereum-Contract-ABI
            abi: [],
            // 请参阅元数据输出文档（序列化的JSON字符串）
            metadata: "{...}",
            // 用户文档（natspec）
            userdoc: {},
            // 开发人员文档（natspec）
            devdoc: {},
            // 中间表示形式 (string)
            ir: "",
            // EVM相关输出
            evm: {
              // 汇编 (string)
              assembly: "",
              // 旧风格的汇编 (object)
              legacyAssembly: {},
              // 字节码和相关细节
              bytecode: {
                // 十六进制字符串的字节码
                object: "00fe",
                // 操作码列表 (string)
                opcodes: "",
                // 源码映射的字符串。 请参阅源码映射的定义
                sourceMap: "",
                // 如果这里给出了信息，则表示这是一个未链接的对象
                linkReferences: {
                  "libraryFile.sol": {
                    // 字节码中的字节偏移；链接时，从指定的位置替换20个字节
                    "Library1": [
                      { start: 0，length: 20 },
                      { start: 200，length: 20 }
                    ]
                  }
                }
              },
             
              deployedBytecode: {
                ...  // 与上面相同的布局
                "immutableReferences": [
                  // There are two references to the immutable with AST ID 3, both 32 bytes long. One is
                  // at bytecode offset 42, the other at bytecode offset 80.
                  "3": [{ "start": 42, "length": 32 }, { "start": 80, "length": 32 }]
                ]
              },
              // 函数哈希的列表
              methodIdentifiers: {
                "delegate(address)": "5c19a95c"
              },
              // 函数的gas预估量
              gasEstimates: {
                creation: {
                  codeDepositCost: "420000",
                  executionCost: "infinite",
                  totalCost: "infinite"
                },
                external: {
                  "delegate(address)": "25000"
                },
                internal: {
                  "heavyLifting()": "infinite"
                }
              }
            },
            // eWASM相关的输出
            ewasm: {
              // S-expressions格式
              wast: "",
              // 二进制格式（十六进制字符串）
              wasm: ""
            }
          }
        }
      }
    }


错误类型
~~~~~~~~~~~

1. ``JSONError``: JSON输入不符合所需格式，例如，输入不是JSON对象，不支持的语言等。
2. ``IOError``: IO和导入处理错误，例如，在提供的源里包含无法解析的URL或哈希值不匹配。
3. ``ParserError``: 源代码不符合语言规则。
4. ``DocstringParsingError``: 注释块中的NatSpec标签无法解析。
5. ``SyntaxError``: 语法错误，例如 ``continue`` 在 ``for`` 循环外部使用。
6. ``DeclarationError``: 无效的，无法解析的或冲突的标识符名称 比如 ``Identifier not found``。
7. ``TypeError``: 类型系统内的错误，例如无效类型转换，无效赋值等。
8. ``UnimplementedFeatureError``: 编译器当前不支持该功能，但预计将在未来的版本中支持。
9. ``InternalCompilerError``: 在编译器中触发的内部错误——应将此报告为一个issue。
10. ``Exception``: 编译期间的未知失败——应将此报告为一个issue。
11. ``CompilerError``: 编译器堆栈的无效使用——应将此报告为一个issue。
12. ``FatalError``: 未正确处理致命错误——应将此报告为一个issue。
13. ``Warning``: 警告，不会停止编译，但应尽可能处理。



.. _compiler-tools:

Compiler tools
**************

solidity-upgrade
----------------

``solidity-upgrade`` can help you to semi-automatically upgrade your contracts
to breaking language changes. While it does not and cannot implement all
required changes for every breaking release, it still supports the ones, that
would need plenty of repetitive manual adjustments otherwise.

.. note::

    ``solidity-upgrade`` carries out a large part of the work, but your
    contracts will most likely need further manual adjustments. We recommend
    using a version control system for your files. This helps reviewing and
    eventually rolling back the changes made.

.. warning::

    ``solidity-upgrade`` is not considered to be complete or free from bugs, so
    please use with care.

How it works
~~~~~~~~~~~~

You can pass (a) Solidity source file(s) to ``solidity-upgrade [files]``. If
these make use of ``import`` statement which refer to files outside the
current source file's directory, you need to specify directories that
are allowed to read and import files from, by passing
``--allow-paths [directory]``. You can ignore missing files by passing
``--ignore-missing``.

``solidity-upgrade`` is based on ``libsolidity`` and can parse, compile and
analyse your source files, and might find applicable source upgrades in them.

Source upgrades are considered to be small textual changes to your source code.
They are applied to an in-memory representation of the source files
given. The corresponding source file is updated by default, but you can pass
``--dry-run`` to simulate to whole upgrade process without writing to any file.

The upgrade process itself has two phases. In the first phase source files are
parsed, and since it is not possible to upgrade source code on that level,
errors are collected and can be logged by passing ``--verbose``. No source
upgrades available at this point.

In the second phase, all sources are compiled and all activated upgrade analysis
modules are run alongside compilation. By default, all available modules are
activated. Please read the documentation on
:ref:`available modules <upgrade-modules>` for further details.


This can result in compilation errors that may
be fixed by source upgrades. If no errors occur, no source upgrades are being
reported and you're done.
If errors occur and some upgrade module reported a source upgrade, the first
reported one gets applied and compilation is triggered again for all given
source files. The previous step is repeated as long as source upgrades are
reported. If errors still occur, you can log them by passing ``--verbose``.
If no errors occur, your contracts are up to date and can be compiled with
the latest version of the compiler.

.. _upgrade-modules:

Available upgrade modules
~~~~~~~~~~~~~~~~~~~~~~~~~~

+-----------------+---------+--------------------------------------------------+
| Module          | Version | Description                                      |
+=================+=========+==================================================+
| ``constructor`` | 0.5.0   | Constructors must now be defined using the       |
|                 |         | ``constructor`` keyword.                         |
+-----------------+---------+--------------------------------------------------+
| ``visibility``  | 0.5.0   | Explicit function visibility is now mandatory,   |
|                 |         | defaults to ``public``.                          |
+-----------------+---------+--------------------------------------------------+
| ``abstract``    | 0.6.0   | The keyword ``abstract`` has to be used if a     |
|                 |         | contract does not implement all its functions.   |
+-----------------+---------+--------------------------------------------------+
| ``virtual``     | 0.6.0   | Functions without implementation outside an      |
|                 |         | interface have to be marked ``virtual``.         |
+-----------------+---------+--------------------------------------------------+
| ``override``    | 0.6.0   | When overriding a function or modifier, the new  |
|                 |         | keyword ``override`` must be used.               |
+-----------------+---------+--------------------------------------------------+

Please read :doc:`0.5.0 release notes <050-breaking-changes>` and
:doc:`0.6.0 release notes <060-breaking-changes>` for further details.

Synopsis
~~~~~~~~

.. code-block:: none

    Usage: solidity-upgrade [options] contract.sol

    Allowed options:
        --help               Show help message and exit.
        --version            Show version and exit.
        --allow-paths path(s)
                             Allow a given path for imports. A list of paths can be
                             supplied by separating them with a comma.
        --ignore-missing     Ignore missing files.
        --modules module(s)  Only activate a specific upgrade module. A list of
                             modules can be supplied by separating them with a comma.
        --dry-run            Apply changes in-memory only and don't write to input
                             file.
        --verbose            Print logs, errors and changes. Shortens output of
                             upgrade patches.
        --unsafe             Accept *unsafe* changes.



Bug Reports / Feature requests
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If you found a bug or if you have a feature request, please
`file an issue <https://github.com/ethereum/solidity/issues/new/choose>`_ on Github.


Example
~~~~~~~

Assume you have the following contracts you want to update declared in ``Source.sol``:

.. code-block:: none

    // This will not compile after 0.5.0
    pragma solidity >0.4.23 <0.5.0;

    contract Updateable {
        function run() public view returns (bool);
        function update() public;
    }

    contract Upgradable {
        function run() public view returns (bool);
        function upgrade();
    }

    contract Source is Updateable, Upgradable {
        function Source() public {}

        function run()
            public
            view
            returns (bool) {}

        function update() {}
        function upgrade() {}
    }


必要的更改
^^^^^^^^^^^^^^^^

To bring the contracts up to date with the current Solidity version, the
following upgrade modules have to be executed: ``constructor``,
``visibility``, ``abstract``, ``override`` and ``virtual``. Please read the
documentation on :ref:`available modules <upgrade-modules>` for further details.

Running the upgrade
^^^^^^^^^^^^^^^^^^^

In this example, all modules needed to upgrade the contracts above,
are available and all of them are activated by default. Therefore you
do not need to specify the ``--modules`` option.

.. code-block:: none

    $ solidity-upgrade Source.sol --dry-run

.. code-block:: none

    Running analysis (and upgrade) on given source files.
    ..............

    After upgrade:

    Found 0 errors.
    Found 0 upgrades.

The above performs a dry-ran upgrade on the given file and logs statistics after all.
In this case, the upgrade was successful and no further adjustments are needed.

Finally, you can run the upgrade and also write to the source file.

.. code-block:: none

    $ solidity-upgrade Source.sol

.. code-block:: none

    Running analysis (and upgrade) on given source files.
    ..............

    After upgrade:

    Found 0 errors.
    Found 0 upgrades.


Review changes
^^^^^^^^^^^^^^

The command above applies all changes as shown below. Please review them carefully.

.. code-block:: solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity >=0.6.0 <0.7.0;

    abstract contract Updateable {
        function run() public view virtual returns (bool);
        function update() public virtual;
    }

    abstract contract Upgradable {
        function run() public view virtual returns (bool);
        function upgrade() public virtual;
    }

    contract Source is Updateable, Upgradable {
        constructor() public {}

        function run()
            public
            view
            override(Updateable,Upgradable)
            returns (bool) {}

        function update() public override {}
        function upgrade() public override {}
    }