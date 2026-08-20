```

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


```
