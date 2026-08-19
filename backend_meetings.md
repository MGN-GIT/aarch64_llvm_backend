- [LLVM IR](https://www.youtube.com/watch?v=m8G_S5LwlTo)
- [Optimization Passes in LLVM IR](https://www.youtube.com/watch?v=7GHXDEIMGIY)
- [Cost Modeling (TTI)](https://www.youtube.com/watch?v=uvOiF0RtaGs)
- [GI vs SD](https://www.youtube.com/watch?v=F6GGbYtae3g&list=PL_R5A0lGi1AA4Lv2bBFSwhgDaHvvpVU21&index=3)
- [SelectionDAG](https://youtu.be/nNQ6AF6i5FI)
- [GI Instruction Selection](https://discourse.llvm.org/t/the-state-of-art-at-instruction-selection/57674/3)
- [TableGen — AArch64](https://youtu.be/vkVjIAlzdMw)
- [Instruction Scheduling Model — LLVM](https://www.youtube.com/watch?v=YZHhlmOTG0g)
- [Register Allocation](https://www.youtube.com/watch?v=IK8TMJf3G6U)
- [Object Code Emission](https://www.youtube.com/watch?v=VPyZBi39Ymw&t=236s)
- [Creating an LLVM Backend](https://youtu.be/b53WqCbLEYg)
- [Writing an LLVM Backend — LLVM Documentation](https://llvm.org/docs/WritingAnLLVMBackend.html)
- [Modern C++](https://youtube.com/playlist?list=PLgnQpQtFTOGRM59sr3nSL8BmeMZR9GCIA)
- [Concurrency in C++](https://youtube.com/playlist?list=PLvv0ScY6vfd_ocTP2ZLicgqKnvq50OCXM)

```
                    LLVM IR
                       │
                       ▼
              Target IR optimizations
                       │
                       ▼
              Instruction Selection
          ┌────────────┴────────────┐
          │                         │
     SelectionDAG                GlobalISel
          │                         │
          └────────────┬────────────┘
                       ▼
                   Machine IR
                       │
                       ▼
             ┌───────────────────┐
             │ Pre-RA optimization│
             │ + scheduling      │
             └─────────┬─────────┘
                       │
                       ▼
              Register Allocation
                       │
                       ▼
             ┌───────────────────┐
             │ Post-RA scheduling│
             │ + optimizations   │
             └─────────┬─────────┘
                       │
                       ▼
             Prologue / Epilogue
                       │
                       ▼
                Late MI passes
                       │
                       ▼
                  Assembly


```
