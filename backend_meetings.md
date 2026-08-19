GI VS SD : https://www.youtube.com/watch?v=F6GGbYtae3g&list=PL_R5A0lGi1AA4Lv2bBFSwhgDaHvvpVU21&index=3

GI Instruction Selection: https://discourse.llvm.org/t/the-state-of-art-at-instruction-selection/57674/3
 
Tablegen Aarch64: https://youtu.be/vkVjIAlzdMw?si=C3zruIaXck8tZG3w
 
Instruction scheduling modelling llvm: https://www.youtube.com/watch?v=YZHhlmOTG0g
 
Register Allocation: https://www.youtube.com/watch?v=IK8TMJf3G6U
 
Object Code Emission: https://www.youtube.com/watch?v=VPyZBi39Ymw&t=236s
 
Creating a LLVM Backend: https://youtu.be/b53WqCbLEYg?si=Z8KKigGErzzQhJDO
 
LLVM docs for backend: https://llvm.org/docs/WritingAnLLVMBackend.html

LLVM IR : https://www.youtube.com/watch?v=m8G_S5LwlTo

Optimization Passes in LLVM IR : https://www.youtube.com/watch?v=7GHXDEIMGIY
 
SelectionDAG: https://youtu.be/nNQ6AF6i5FI?si=OBb16bQxxlg--uRD

Modern CPP: https://youtube.com/playlist?list=PLgnQpQtFTOGRM59sr3nSL8BmeMZR9GCIA&si=G852AdIfjcGTzs8W

Concurrency in cpp: https://youtube.com/playlist?list=PLvv0ScY6vfd_ocTP2ZLicgqKnvq50OCXM&si=Xz6RwIiIWx_smAgh

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
