## 一、实验目的

本实验目的是帮助你理解cache对C语言程序性能的影响。

## 二、报告要求

本报告要求学生解释说明所提交的转置函数是如何针对三种矩阵大小进行了何种优化的，优化效果如何。欢迎你写出你尝试过的所有优化策略，比较它们的效果以及分析效果优劣的原因。

## 三、实现分析

实验规则：

1. 最多使用12个局部变量
2. 不可以使用递归
3. 不可以修改数组A
4. 不能定义任何数组，不能使用malloc函数
5. $s =5,E=1,b=5$，也就是$C=1024$，一个set中有一个块，可以存储**8个int数据**

矩阵转置与

### 1. 针对$32 \times 32 (M=32, N=32)$的矩阵

#### 优化策略

采取**分块**矩阵，每个Block的大小为$B\times B$，为了实现好的优化，要有$2B^2<C/4$，那么合适的$B$取值为8（取32的因子会比较好操作）

这里我一开始尝试最内层循环是这样的：

```c
for(int jj=j;jj<j+8;j++){
    B[ii][jj] = A[jj][ii];
}
```

但是这样评测出来的结果miss数约为350，所以需要继续提升他的局部性，我的到得到了下面的代码:

```c
int temp0,temp1,temp2,temp3,temp4,temp5,temp6,temp7;
for(int i=0;i<M;i+=8){
	for(int j=0;j<N;j+=8){
		for(int k=i;k<i+8;k++){
			temp0 = A[j+0][k];
            temp1 = A[j+1][k];
            temp2 = A[j+2][k];
            temp3 = A[j+3][k];
            temp4 = A[j+4][k];
            temp5 = A[j+5][k];
            temp6 = A[j+6][k];
            temp7 = A[j+7][k];
            B[k][j+0] = temp0;
            B[k][j+1] = temp1;
            B[k][j+2] = temp2;
            B[k][j+3] = temp3;
            B[k][j+4] = temp4;
            B[k][j+5] = temp5;
            B[k][j+6] = temp6;
            B[k][j+7] = temp7;
		}
}
```

#### 优化效果

miss数从1183下降到287：

![image-20221129105802271](https://lbw-img-lbw.oss-cn-beijing.aliyuncs.com/img/image-20221129105802271.png)

#### 原因分析

我第一次尝试得到的代码很容易写出来，因为只要采取了分块的思想，就能写出这样的代码；但是效果不够达到满分，因为矩阵A和矩阵B同一位置的元素是映射到同一个Line上的，也就是说如果直接写循环会造成很多冲突不命中。所以我用8个局部变量一次读8个A中元素，再一口气写到B中，就又能提升命中数

### 2. 针对$64 \times 64 (M=64, N=64)$的矩阵

#### 优化策略

这个矩阵大小是第一个矩阵大小的4倍，我一开始觉得它的优化思路与前一题一样，但是我在对这个矩阵尝试以8为长度分块时，并没有得到很好的优化效果，miss数只比按行转置少一丢丢

那这个时候就要转换优化策略了，但是除了分块之外我个人想不出什么比较好的优化策略。思考一段时间不得之后我就去网上搜索解决的方案，发现一个可行的方案是：**在八分块中再四分块**

那么就遵循这个思路进行优化，先写一个八分块的循环；然后再从八分块中进行一次8分块和两次4分块处理：

```c
		int i, j, x, y;
		int temp0,temp1,temp2,temp3,temp4,temp5,temp6,temp7;
		for (i = 0; i < N; i += 8){
			for (j = 0; j < M; j += 8){
				for (x = i; x < i + 4; ++x){
					temp0 = A[x][j]; 
                    temp1 = A[x][j+1]; 
                    temp2 = A[x][j+2]; 
                    temp3 = A[x][j+3];
					temp4 = A[x][j+4]; 
                    temp5 = A[x][j+5]; 
                    temp6 = A[x][j+6]; 
                    temp7 = A[x][j+7];
					
					B[j][x] = temp0; 
                    B[j+1][x] = temp1; 
                    B[j+2][x] = temp2; 
                    B[j+3][x] = temp3;
					B[j][x+4] = temp4; 
                    B[j+1][x+4] = temp5; 
                    B[j+2][x+4] = temp6; 
                    B[j+3][x+4] = temp7;
				}
				for (y = j; y < j + 4; ++y){
					temp0 = A[i+4][y]; 
                    temp1 = A[i+5][y]; 
                    temp2 = A[i+6][y]; 
                    temp3 = A[i+7][y];
					temp4 = B[y][i+4]; 
                    temp5 = B[y][i+5]; 
                    temp6 = B[y][i+6]; 
                    temp7 = B[y][i+7];
					
					B[y][i+4] = temp0; 
                    B[y][i+5] = temp1; 
                    B[y][i+6] = temp2; 
                    B[y][i+7] = temp3;
					B[y+4][i] = temp4; 
                    B[y+4][i+1] = temp5; 
                    B[y+4][i+2] = temp6; 
                    B[y+4][i+3] = temp7;
				}
				for (x = i + 4; x < i + 8; ++x){
					temp0 = A[x][j+4]; 
                    temp1 = A[x][j+5]; 
                    temp2 = A[x][j+6]; 
                    temp3 = A[x][j+7];
					B[j+4][x] = temp0; 
                    B[j+5][x] = temp1; 
                    B[j+6][x] = temp2; 
                    B[j+7][x] = temp3;
				}
			}
        }
```

#### 优化效果

miss数从4723下降到1179：

![image-20221129105422113](https://lbw-img-lbw.oss-cn-beijing.aliyuncs.com/img/image-20221129105422113.png)

### 3. 针对$61 \times 67 (M=61, N=67)$的矩阵

#### 优化策略

仍然跟前面一样，先尝试以8为长度分块

但是与前面不一样的是，之前以8为长度分块可以正好分完，但是现在不行；所以需要在8分块之后继续进行转置：

```c
int i,j,temp0,temp1,temp2,temp3,temp4,temp5,temp6,temp7;
        // 8分块
		for (j = 0; j < 56; j += 8){
			for (i = 0; i < 64; ++i){
				temp0 = A[i][j];
				temp1 = A[i][j+1];
				temp2 = A[i][j+2];
				temp3 = A[i][j+3];
				temp4 = A[i][j+4];
				temp5 = A[i][j+5];
				temp6 = A[i][j+6];
				temp7 = A[i][j+7];
				
				B[j][i] = temp0;
				B[j+1][i] = temp1;
				B[j+2][i] = temp2;
				B[j+3][i] = temp3;
				B[j+4][i] = temp4;
				B[j+5][i] = temp5;
				B[j+6][i] = temp6;
				B[j+7][i] = temp7;
			}
        }

        // 直接转置 
		for (i = 0; i < N; ++i){
            for (j = 56; j < M; ++j){
				temp0 = A[i][j];
				B[j][i] = temp0;
			}
        }

		for (i = 64; i < N; ++i){
            for (j = 0; j < M; ++j){
				temp0 = A[i][j];
				B[j][i] = temp0;
			}
        }

        for (i = 64; i < N; ++i){
            for (j = 56; j < M; ++j){
				temp0 = A[i][j];
				B[j][i] = temp0;
			}
        }
```

#### 优化效果

miss数从4432降低到1904：

![image-20221129101159125](https://lbw-img-lbw.oss-cn-beijing.aliyuncs.com/img/image-20221129101159125.png)

#### 原因分析

这个矩阵得到优化的矩阵和第一个矩阵一样，都是因为分块。该矩阵其中$56\times64$的部分使用8分块，其他部分直接转置；直接转置造成的miss数会相对多一些，但好在8分块对转置的优化已经足以让这个实验拿到满分了，不需要再进行什么优化。



## 四、实验总结

遇到的困难就是这个实验有点太难了，如果有老师进行一些指导可能会做出来。包括$64\times64$矩阵的优化真的很难想，最后在网上才找到思路

还有就是实验平台真的好卡，有的时候甚至在vim里面写代码都卡，哎。。。

