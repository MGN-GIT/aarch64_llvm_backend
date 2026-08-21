### Build instructions 

* Configure CMD :

```
cmake -S <Path to llvm src> -B <Path to build out> -G Ninja -DCMAKE_BUILD_TYPE=Debug -DLLVM_TARGETS_TO_BUILD=AArch64 -DLLVM_INCLUDE_TESTS=ON -DLLVM_BUILD_TESTS=ON -DLLVM_INCLUDE_BENCHMARKS=OFF -DLLVM_INCLUDE_EXAMPLES=OFF -DLLVM_INCLUDE_DOCS=OFF -DLLVM_ENABLE_PROJECTS=
```

* Build only core Backend components :
```
cmake --build <Path to build out> --target llc FileCheck count not split-file llvm-config

```

### Running tests

* Change the path to build output bin dir -> Make sure you have the core llc(llvm static compiler), file check, count, not, etc

```
Generates asm directly: llc.exe -mtriple=aarch64-linux-gnu test.mir -o output.s
Generates mir after the pass but stop before asm printer/obj emission:  llc.exe -mtriple=aarch64-linux-gnu -run-pass=<mir pass name> -verify-machineinstrs test.mir -o output.mir
```

### Condition Codes - AArch64


| Condition |          Value | Meaning         | Common use                      |   |         |
| --------- | -------------: | --------------- | ------------------------------- | - | ------- |
| `EQ`      |  `0x0` / **0** | Equal           | `Z == 1`                        |   |         |
| `NE`      |  `0x1` / **1** | Not equal       | `Z == 0`                        |   |         |
| `HS`      |  `0x2` / **2** | Unsigned ≥      | `C == 1`                        |   |         |
| `LO`      |  `0x3` / **3** | Unsigned <      | `C == 0`                        |   |         |
| `MI`      |  `0x4` / **4** | Negative        | `N == 1`                        |   |         |
| `PL`      |  `0x5` / **5** | Positive / zero | `N == 0`                        |   |         |
| `VS`      |  `0x6` / **6** | Overflow        | `V == 1`                        |   |         |
| `VC`      |  `0x7` / **7** | No overflow     | `V == 0`                        |   |         |
| `HI`      |  `0x8` / **8** | Unsigned >      | `C == 1 && Z == 0`              |   |         |
| `LS`      |  `0x9` / **9** | Unsigned ≤      | `C == 0                         |   | Z == 1` |
| `GE`      | `0xA` / **10** | Signed ≥        | `N == V`                        |   |         |
| `LT`      | `0xB` / **11** | Signed <        | `N != V`                        |   |         |
| `GT`      | `0xC` / **12** | Signed >        | `Z == 0 && N == V`              |   |         |
| `LE`      | `0xD` / **13** | Signed ≤        | `Z == 1                         |   | N != V` |
| `AL`      | `0xE` / **14** | Always          | Unconditional                   |   |         |
| `NV`      | `0xF` / **15** | Always          | Encodes `1111`; executes always |   |         |

