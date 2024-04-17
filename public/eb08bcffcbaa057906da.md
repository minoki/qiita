---
title: Metal Performance Shadersで行列乗算
tags:
  - Objective-C
  - Metal
private: false
updated_at: '2021-02-08T22:18:02+09:00'
id: eb08bcffcbaa057906da
organization_url_name: null
slide: false
ignorePublish: false
---
# Metal Performance Shadersで行列乗算

Metal Performance Shadersというやつを使うと行列演算等をApple製品のGPUで行えるらしいです。というわけでObjective-Cから行列乗算を呼び出してみました。

行列はrow majorのみ対応のようです（column majorのみ対応なcuBLASとは対照的です）。浮動小数点数の精度は単精度と半精度のみで、倍精度には対応していないようです。

```objc
// clang -fobjc-arc mps-test.m -lobjc -framework Metal -framework MetalPerformanceShaders -framework CoreGraphics
#import <Metal/Metal.h>
#import <MetalPerformanceShaders/MetalPerformanceShaders.h>
#include <stdio.h>

int main()
{
    float A[3][3] = {
        {1.0, 2.0, 3.0},
        {2.0, 3.0, 4.0},
        {4.0, 5.0, 6.0},
    };
    float B[3][4] = {
        {1.0, 2.0, 3.0, 2.0},
        {2.0, -2.0, 4.0, 0.0},
        {1.0, 5.0, 6.0, -1.0},
    };
    float C[3][4];
    printf("A: {");
    for (size_t i = 0; i < 3 * 3; ++i) {
        printf("%g, ", (&A[0][0])[i]);
    }
    puts("}");
    for (size_t i = 0; i < 3; ++i) {
        for (size_t j = 0; j < 3; ++j) {
            printf("%g, ", A[i][j]); // 3 * i + j
        }
        puts("");
    }
    puts("---");
    printf("sizeof(B[0]) = %zu\n", sizeof(B[0]));
    printf("B: {");
    for (size_t i = 0; i < 3 * 4; ++i) {
        printf("%g, ", (&B[0][0])[i]);
    }
    puts("}");
    for (size_t j = 0; j < 3; ++j) {
        for (size_t k = 0; k < 4; ++k) {
            printf("%g, ", B[j][k]); // 4 * j + k
        }
        puts("");
    }
    puts("---");
    @autoreleasepool {
        id <MTLDevice> device = MTLCreateSystemDefaultDevice(); // -framework CoreGraphics をしないと nil が返ってくるので注意
        if (device == nil) {
            puts("MTLCreateSystemDefaultDevice failed");
            return 1;
        }
        printf("device name: %s\n", [[device name] UTF8String]);
        if (!MPSSupportsMTLDevice(device)) {
            puts("Metal Performance Shaders does not support this metal device.");
            return 1;
        }
        id <MTLCommandQueue> commandQueue = [device newCommandQueue];

        // バッファーの用意
        id <MTLBuffer> bufA = [device newBufferWithBytes:A length:sizeof(float) * 3 * 3 options:MTLResourceStorageModeManaged]; // CPUからGPUに渡す用
        id <MTLBuffer> bufB = [device newBufferWithBytes:B length:sizeof(float) * 3 * 4 options:MTLResourceStorageModeManaged]; // CPUからGPUに渡す用
        id <MTLBuffer> bufC = [device newBufferWithLength:sizeof(float) * 3 * 4 options:MTLResourceStorageModeShared]; // 計算完了後にGPUからCPUに渡す用

        // 行列の用意
        MPSMatrix *matA = [[MPSMatrix alloc] initWithBuffer:bufA descriptor:[MPSMatrixDescriptor matrixDescriptorWithRows:3 columns:3 rowBytes:sizeof(float) * 3 dataType:MPSDataTypeFloat32]];
        MPSMatrix *matB = [[MPSMatrix alloc] initWithBuffer:bufB descriptor:[MPSMatrixDescriptor matrixDescriptorWithRows:3 columns:4 rowBytes:sizeof(float) * 4 dataType:MPSDataTypeFloat32]];
        MPSMatrix *matC = [[MPSMatrix alloc] initWithBuffer:bufC descriptor:[MPSMatrixDescriptor matrixDescriptorWithRows:3 columns:4 rowBytes:sizeof(float) * 4 dataType:MPSDataTypeFloat32]];

        id <MTLCommandBuffer> commandBuffer = [commandQueue commandBuffer];
        // MPSMatrixMultiplication *mmul = [[MPSMatrixMultiplication alloc] initWithDevice:device transposeLeft:NO transposeRight:NO resultRows:3 resultColumns:4 interiorColumns:3 alpha:1.0 beta:0.0]; // この辺BLASっぽい
        MPSMatrixMultiplication *mmul = [[MPSMatrixMultiplication alloc] initWithDevice:device resultRows:3 resultColumns:4 interiorColumns:3];
        [mmul encodeToCommandBuffer:commandBuffer leftMatrix:matA rightMatrix:matB resultMatrix:matC];
        [commandBuffer commit];
        [commandBuffer waitUntilCompleted];
        // この時点でCPUへの転送が済んでいて、演算結果を [bufC contents] で読めるはず
        memcpy(C, [bufC contents], sizeof(C));
    }
    // 結果の表示
    puts("---");
    printf("C: {");
    for (size_t i = 0; i < 3 * 4; ++i) {
        printf("%g, ", (&C[0][0])[i]);
    }
    puts("}");
    for (size_t i = 0; i < 3; ++i) {
        for (size_t k = 0; k < 4; ++k) {
            printf("%g, ", C[i][k]); // 4 * i + k
        }
        puts("");
    }
}
```

実行結果（Intel Mac）：

```
A: {1, 2, 3, 2, 3, 4, 4, 5, 6, }
1, 2, 3, 
2, 3, 4, 
4, 5, 6, 
---
sizeof(B[0]) = 16
B: {1, 2, 3, 2, 2, -2, 4, 0, 1, 5, 6, -1, }
1, 2, 3, 2, 
2, -2, 4, 0, 
1, 5, 6, -1, 
---
device name: Intel(R) Iris(TM) Plus Graphics
---
C: {8, 13, 29, -1, 12, 18, 42, 0, 20, 28, 68, 2, }
8, 13, 29, -1, 
12, 18, 42, 0, 
20, 28, 68, 2, 
```

実行結果（Apple M1）：

```
A: {1, 2, 3, 2, 3, 4, 4, 5, 6, }
1, 2, 3, 
2, 3, 4, 
4, 5, 6, 
---
sizeof(B[0]) = 16
B: {1, 2, 3, 2, 2, -2, 4, 0, 1, 5, 6, -1, }
1, 2, 3, 2, 
2, -2, 4, 0, 
1, 5, 6, -1, 
---
device name: Apple M1
---
C: {8, 13, 29, -1, 12, 18, 42, 0, 20, 28, 68, 2, }
8, 13, 29, -1, 
12, 18, 42, 0, 
20, 28, 68, 2, 
```

# おまけ：BLASでの例

この記事及び [HaskellからBLASを叩く](https://qiita.com/mod_poppo/items/625677a12ca42aca22e1) の元ネタになったCBLASを使うコードも供養しておきます。

```c
#include <cblas_openblas.h>
#include <stdio.h>
#include <math.h>

int main(void)
{
    double A[3][3] = {
        {1.0, 2.0, 3.0},
        {2.0, 3.0, 4.0},
        {4.0, 5.0, 6.0},
    };
    double B[3][4] = {
        {1.0, 2.0, 3.0, 2.0},
        {2.0, -2.0, 4.0, 0.0},
        {1.0, 5.0, 6.0, -1.0},
    };
    double C[3][4];
    printf("A: {");
    for (size_t i = 0; i < 3 * 3; ++i) {
        printf("%g, ", (&A[0][0])[i]);
    }
    puts("}");
    for (size_t i = 0; i < 3; ++i) {
        for (size_t j = 0; j < 3; ++j) {
            printf("%g, ", A[i][j]); // 3 * i + j
        }
        puts("");
    }
    puts("---");
    printf("sizeof(B[0]) = %zu\n", sizeof(B[0]));
    printf("B: {");
    for (size_t i = 0; i < 3 * 4; ++i) {
        printf("%g, ", (&B[0][0])[i]);
    }
    puts("}");
    for (size_t j = 0; j < 3; ++j) {
        for (size_t k = 0; k < 4; ++k) {
            printf("%g, ", B[j][k]); // 4 * j + k
        }
        puts("");
    }
    puts("---");
    for (size_t i = 0; i < 3; ++i) {
        for (size_t k = 0; k < 4; ++k) {
            double x = 0.0;
            for (size_t j = 0; j < 3; ++j) {
                x += A[i][j] * B[j][k];
            }
            C[i][k] = x;
        }
    }
    printf("C (naive): {");
    for (size_t i = 0; i < 3 * 4; ++i) {
        printf("%g, ", (&C[0][0])[i]);
    }
    puts("}");
    for (size_t i = 0; i < 3; ++i) {
        for (size_t k = 0; k < 4; ++k) {
            printf("%g, ", C[i][k]); // 4 * i + k
        }
        puts("");
    }
    puts("---");
    for (size_t i = 0; i < 3; ++i) {
        for (size_t k = 0; k < 4; ++k) {
            C[i][k] = NAN;
        }
    }
    cblas_dgemm(CblasRowMajor /* or CblasColMajor */, CblasNoTrans, CblasNoTrans, /* m */ 3, /* n */ 4, /* k */ 3, /* alpha */ 1.0, &A[0][0], /* lda */ 3, &B[0][0], /* ldb */ 4, /* beta */ 0.0, &C[0][0], /* ldc */ 4);
    printf("C (blas): {");
    for (size_t i = 0; i < 3 * 4; ++i) {
        printf("%g, ", (&C[0][0])[i]);
    }
    puts("}");
    for (size_t i = 0; i < 3; ++i) {
        for (size_t k = 0; k < 4; ++k) {
            printf("%g, ", C[i][k]); // 4 * i + k
        }
        puts("");
    }
}
```

実行結果：

```
A: {1, 2, 3, 2, 3, 4, 4, 5, 6, }
1, 2, 3, 
2, 3, 4, 
4, 5, 6, 
---
sizeof(B[0]) = 32
B: {1, 2, 3, 2, 2, -2, 4, 0, 1, 5, 6, -1, }
1, 2, 3, 2, 
2, -2, 4, 0, 
1, 5, 6, -1, 
---
C (naive): {8, 13, 29, -1, 12, 18, 42, 0, 20, 28, 68, 2, }
8, 13, 29, -1, 
12, 18, 42, 0, 
20, 28, 68, 2, 
---
C (blas): {8, 13, 29, -1, 12, 18, 42, 0, 20, 28, 68, 2, }
8, 13, 29, -1, 
12, 18, 42, 0, 
20, 28, 68, 2, 
```
