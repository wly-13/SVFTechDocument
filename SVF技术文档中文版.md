[SVF官方技术文档](https://github.com/SVF-tools/SVF/wiki/Technical-documentation) 

本文档描述了SVF的详细内部工作，包括内存建模、指针分析和基于其设计的值流构造。您可能还希望参考[本Doxygen手册](http://www.cse.unsw.edu.au/~corg/svf/doxygen/)了解更详细的实现。  

## 1.内存模型  
SVF解析LLVM位代码并创建其内部符号表信息以进行指针分析。我们采纳LLVM的惯例把所有的符号分成两种：ValSym表示寄存器llvm值，它是一个顶级指针；ObjSym表示一个抽象内存对象，它是指针的一个取地址变量。一些对象被特殊标记。比如，一些不能被静态定义的符号被标记为BlackHole，如：LLVM中的UndefValue。常量对象用ConstantObj标记。  
### 1.1抽象内存对象
每个抽象内存对象（MemObj），无论是全局、堆栈或堆对象，都在内存分配站点进行分析。每个对象自己的类型信息被记录在ObjTypeInfo。具有结构类型（StInfo）的对象将使用其所有字段信息（FieldInfo）进行建模，包括每个字段的偏移量和类型。嵌套结构被扁平化。一个结构数组是用它的步幅信息建模的。  
### 1.2程序分配图（PAG）  
SVF将LLVM指令转换为图形表示（PAG）。每个节点（PAGNode）代表一个指针，每个边（PAGEdge）代表两个指针之间的约束。  
#### 1.2.1PAGNode  
![image](https://github.com/svf-tools/SVF/raw/master/images/pagnode.png)  
PAGNode分为两类：ValPN代表指针，ObjPN代表抽象对象。GepObjPN是ObjPN的子类，代表聚合对象的一个字段。GepValPN是ValPN的一个子类，代表了一个引入的虚拟节点，用来在处理外部库调用的时候实现字段敏感，比如，memcpy函数，其中指向结构的字段的指针（llvm值）未显式出现在指令中。RetNode表示过程的唯一返回。VarargNode表示过程的可变参数。  
#### 1.2.2PAGEdge  
![image](https://raw.githubusercontent.com/svf-tools/SVF/master/images/pagedge.png)  
我们将PAGEdge分成以下几类：
- AddrPE 　　　　　　//在内存分配点（ValPN <-- ObjPN)  
- CopyPE  　　　　　　//在PHINode, CastInst 或 SelectInst (ValPN <-- ValPN)  
- StorePE 　　　　　　//在StoreInst (ValPN <-- ValPN)  
- LoadPE 　　　　　　　// 在LoadInst (ValPN <-- ValPN)  
- GepPE  　　　　　　　// 在GetElementPtrInst (ValPN <-- ValPN)
- CallPE 　　　　　　　// 从实参到形参 (ValPN <-- ValPN)  
- RetPE 　　　　　　　// 从RetNode到调用点的返回值 (ValPN <-- ValPN)  
#### 1.2.3PAGBulider 
通过处理以下LLVM指令和表达式，使用PAGBuilder生成PAG： 
- AllocaInst
- PHINode
- StoreInst
- LoadInst
- GetElementPtrInst
- CallInst
- InvokeInst
- ReturnInst
- CastInst
- SelectInst
- IntToPtrInst
- ExtractValueInst
- ConstantExpr
- GlobalValue  
虽然所有其他的LLVM指令都没有被处理，但是它们也可以被转换为PAG中的基本边类型。

### 1.3约束图  
约束图（Constraint Graph）是PAG的副本，用于基于包含的指针分析。在SVF的所有分析中，PAG作为一个基本图来表示整个程序，并且它的边是固定的。但是，在约束解析过程中，ConstraintGraph的边是可以更改的。  
## 2.指针分析  
![image](https://raw.githubusercontent.com/svf-tools/SVF/master/images/pt.png)  
指针分析是所有实现的根类。为了实现不同的指针分析算法，用户可以根据不同的数据结构点选择不同的实现。例如，流敏感和流不敏感的指针分析可以选择BVDataPTAImpl作为基类，它使用位向量作为其核心数据结构，基于指向集将指针映射到位向量。上下文敏感和路径敏感的指针分析要求上下文或路径条件作为每个指针的前修饰符。CondPTAImpl作为基类是一个很好的选择。CondPTAImpl将条件变量映射到它的指向集。  

每种指针分析实现都是通过利用解算器来解算约束对图形表示进行操作。例如，通过在基于包含的约束图（ConsG）上选择合适的求解器来推导传递闭包规则，安德森指针分析算法可以很轻易实现。相似地，在稀疏值流图（VFG）上，可以使用指向传播解算器通过遵循流敏感的强/弱更新规则来实现流敏感分析。  
## 3.值流构造  
![image](https://raw.githubusercontent.com/svf-tools/SVF/master/images/svfg-framework.png)  
### 3.1内存SSA  
在LLVM IR中，顶层变量的def-use链是可读的。寻址对象可以在load语句和stroe语句间接访问到。它们的def-use链分以下几步生成。首先，进行指针分析，生成顶级变量的指向信息。第二，对于load语句p=\*q，对于每一个可能被q指向的变量o用函数μ(o) (LoadMU)来注释，表示变量o的一个潜在使用在该load语句中。相似地，对于store语句\*p=q,对于每一个可能被p指向的变量o用o = χ(o) (StoreCHI)注释，表示变量o在该store语句的潜在定义和使用。调用语句cs用函数 μ(o) (CallMU)和o = χ(o) (CallCHI)注释来捕获变量o的过程间使用和定义。同样地，在过程f入口和退出的地方注释o = χ(o) (EntryCHI) 和 μ(o) (RetMU) ，模拟非局部变量o的参数传递（返回）
PHI语句 o=mssaPhi（o）（MSSAPHI）操作插入控制流连接点，以合并对象o的多个定义。第三，所有的寻址对象被转换成SSA模式，每个μ（o）都被视为o的使用，每个o = χ(o) 都被视为定义和o的使用。  
![image](https://raw.githubusercontent.com/svf-tools/SVF/master/images/mssa-cha.png)  
### 3.2稀疏值流图（SVFG）
给定一个带有μ和χ函数注释的程序，在SSA转换后，通过连接每个SSA变量的def-use链，构造一个SVFG。

程序的过程间稀疏值流图（svfg）是一个有向图，它捕获顶级指针和寻址对象的def-use链。  

#### 3.2.1SVFGNode
SVFG节点是以下节点之一：  
![image](https://raw.githubusercontent.com/svf-tools/SVF/master/images/svfgnode-cha.png)  

（1）语句（PAGEdge）

- AddrPE
- CopyPE
- GepPE
- LoadPE
- StorePE
 
（2）内存区域定义

- FormalIN 　　　　　　// EntryCHI
- FormalOut　　　　　　// RetMU
- ActualIn　　　　　　　// CallMU
- ActualOut 　　　　　　// CallCHI
- MSSAPHI　　　　　　　// MSSAPHI　

（3）参数

- FormalParm 　　　　　 //Formal Parameter
- ActualParm 　　　　　 //Actual Parameter
- FormalRet  　　　　　 // Procedure Return Variable
- ActualRet 　　　　　  //CallSite Return Variable

#### 3.2.2SVFGEdge

SVFG边表示从变量定义到使用的值流依赖性。价值流是下列之一：  
![image](https://raw.githubusercontent.com/svf-tools/SVF/master/images/svfgedge-cha.png)

（1）顶级指针的直接值流  
（2）寻址对象的间接价值流

#### 3.2.3SVFG优化

一些SVFG节点可以进行优化以使图形更紧凑。

可以删除以下节点：ActualParm ActualIn FormalRet FormalOut
