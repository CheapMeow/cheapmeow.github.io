---
title: 'C++用于高性能计算的多维数组的不同实现的性能测试'
show_date: true
permalink: /posts/2024/07/cpp_multiple_dimensional_array/
tags:
  - Cpp
  - Mutliple Dimensional Array
  - High Performance Computing
toc: true
toc_sticky: true
author_profile: false
---

开发 CFD 求解器的时候，需要使用三维数组来表示结构网格。我希望这个三维数组的内存是连续的，所以我分配了三维空间大小的线性内存。需要使用三维坐标来访问数组时，人为计算一维索引。这自然就引出了一个问题：这样做会带来性能损失吗？如果有，损失多少？

测试代码见 [https://github.com/CheapMeow/test-mdspan](https://github.com/CheapMeow/test-mdspan)

## 不同的实现

第一个最简单的实现就是，分配一维连续数组，三维坐标转化为一维索引

```cpp
class field3_1dp
{
protected:
    unsigned int Nx, Ny, Nz;

public:
    double* value = nullptr;
    field3_1dp(unsigned int _Nx, unsigned int _Ny, unsigned int _Nz)
        : Nx(_Nx)
        , Ny(_Ny)
        , Nz(_Nz)
    {
        value = new double[Nx * Ny * Nz];

        for (unsigned int i = 0; i < Nx * Ny * Nz; i++)
            value[i] = 0.;
    }

    ~field3_1dp() { delete[] value; }

    double& operator()(unsigned int i, unsigned int j, unsigned int k) { return value[i * Ny * Nz + j * Nz + k]; }

    int SizeX() { return Nx; }
    int SizeY() { return Ny; }
    int SizeZ() { return Nz; }
};
```

为了加速一维索引的计算，可以使用模板来将维度参数固定为字面值

```cpp
template<unsigned int Nx, unsigned int Ny, unsigned int Nz>
class field3_1dp_t
{
public:
    double* value = nullptr;
    field3_1dp_t()
    {
        value = new double[Nx * Ny * Nz];

        for (unsigned int i = 0; i < Nx * Ny * Nz; i++)
            value[i] = 0.;
    }

    ~field3_1dp_t() { delete[] value; }

    double& operator()(unsigned int i, unsigned int j, unsigned int k) { return value[i * Ny * Nz + j * Nz + k]; }
};
```

作为对比，你可以在类的内部分配三维数组

```cpp
class field3_3dp
{
protected:
    unsigned int Nx, Ny, Nz;

public:
    double*** value = nullptr;
    field3_3dp(unsigned int _Nx, unsigned int _Ny, unsigned int _Nz)
        : Nx(_Nx)
        , Ny(_Ny)
        , Nz(_Nz)
    {
        value = allocate_3d_array(Nx, Ny, Nz);
    }

    ~field3_3dp() { deallocate_3d_array(value, Nx, Ny); }

    double& operator()(unsigned int i, unsigned int j, unsigned int k) { return value[i][j][k]; }

    int SizeX() { return Nx; }
    int SizeY() { return Ny; }
    int SizeZ() { return Nz; }
};
```

但是我希望数据是一维连续分布的……这样会方便我传输一个 2d 的 slice

所以如果乘法计算一维索引很费的话，那么也可以创建一个三维数组，然后把指针指向实际的一维内存

```cpp
class field3_map
{
protected:
    unsigned int Nx, Ny, Nz;
    double*      value;
    double***    ptr3d;

public:
    field3_map(unsigned int _Nx, unsigned int _Ny, unsigned int _Nz)
        : Nx(_Nx)
        , Ny(_Ny)
        , Nz(_Nz)
        , value(new double[Nx * Ny * Nz])
    {
        ptr3d = new double**[Nx];
        for (unsigned int i = 0; i < Nx; ++i)
        {
            ptr3d[i] = new double*[Ny];
            for (unsigned int j = 0; j < Ny; ++j)
            {
                ptr3d[i][j] = value + (i * Ny + j) * Nz;
            }
        }
    }

    ~field3_map()
    {
        for (unsigned int i = 0; i < Nx; ++i)
        {
            delete[] ptr3d[i];
        }
        delete[] ptr3d;
        delete[] value;
    }

    double** operator[](unsigned int i) { return ptr3d[i]; }

    int SizeX() { return Nx; }
    int SizeY() { return Ny; }
    int SizeZ() { return Nz; }
};
```

Kokkos 有一个 `mdspan` 实现来达成类似的效果。

```cpp
class field3_mdspan
{
protected:
    unsigned int                                     Nx, Ny, Nz;
    double*                                          value;
    Kokkos::mdspan<double, Kokkos::dextents<int, 3>> m_mdspan;

public:
    field3_mdspan(unsigned int _Nx, unsigned int _Ny, unsigned int _Nz)
        : Nx(_Nx)
        , Ny(_Ny)
        , Nz(_Nz)
        , value(new double[Nx * Ny * Nz])
        , m_mdspan(value, Nx, Ny, Nz)
    {}

    ~field3_mdspan() { delete[] value; }

    double& operator()(int i, int j, int k) { return m_mdspan[i, j, k]; }

    int SizeX() { return Nx; }
    int SizeY() { return Ny; }
    int SizeZ() { return Nz; }
};
```

使用一个常见的对流计算函数来做测试，这是一个例子

因为不同的类的访问可能是 `(i, j, k)` 或者 `[i, j, k]` 或者 `[i][j][k]`，所以我并没有写模板函数，而是每个类都写了对应的函数

```cpp
// Function to calculate convection term u * ∇u + v * ∇v + w * ∇w
void calculate_convection_3dp_raw(double***    u,
                                  double***    v,
                                  double***    w,
                                  double***    conv_u,
                                  double***    conv_v,
                                  double***    conv_w,
                                  unsigned int Nx,
                                  unsigned int Ny,
                                  unsigned int Nz)
{
    for (unsigned int i = 1; i < Nx - 1; ++i)
    {
        for (unsigned int j = 1; j < Ny - 1; ++j)
        {
            for (unsigned int k = 1; k < Nz - 1; ++k)
            {
                double dudx = (u[i + 1][j][k] - u[i - 1][j][k]) / 2.0;
                double dudy = (u[i][j + 1][k] - u[i][j - 1][k]) / 2.0;
                double dudz = (u[i][j][k + 1] - u[i][j][k - 1]) / 2.0;

                double dvdx = (v[i + 1][j][k] - v[i - 1][j][k]) / 2.0;
                double dvdy = (v[i][j + 1][k] - v[i][j - 1][k]) / 2.0;
                double dvdz = (v[i][j][k + 1] - v[i][j][k - 1]) / 2.0;

                double dwdx = (w[i + 1][j][k] - w[i - 1][j][k]) / 2.0;
                double dwdy = (w[i][j + 1][k] - w[i][j - 1][k]) / 2.0;
                double dwdz = (w[i][j][k + 1] - w[i][j][k - 1]) / 2.0;

                conv_u[i][j][k] = u[i][j][k] * dudx + v[i][j][k] * dudy + w[i][j][k] * dudz;
                conv_v[i][j][k] = u[i][j][k] * dvdx + v[i][j][k] * dvdy + w[i][j][k] * dvdz;
                conv_w[i][j][k] = u[i][j][k] * dwdx + v[i][j][k] * dwdy + w[i][j][k] * dwdz;
            }
        }
    }
}
```

计算效率的过程为

```cpp
// Measure the time for 100 iterations of convection calculation
const int num_iterations = 20;
auto      start_time     = std::chrono::high_resolution_clock::now();

for (int iter = 0; iter < num_iterations; ++iter)
{
    calculate_convection_1dp(u, v, w, conv_u, conv_v, conv_w);
    std::cout << "iter = " << iter << std::endl;
}

auto                          end_time     = std::chrono::high_resolution_clock::now();
std::chrono::duration<double> elapsed_time = end_time - start_time;
```

测试结果如下，在 512^3 的数组大小下进行测试，以保证单次执行的粒度

<table>
    <tr>
        <td rowspan="2" align="center">platform</td>
        <td rowspan="2" align="center">complier</td>
        <td colspan="6" align="center">time(s)</td>
    </tr>
    <tr>
        <td align="center">field3_1dp</td>
        <td align="center">field3_1dp_t</td>
        <td align="center">field3_3dp</td>
        <td align="center">field3_r</td>
        <td align="center">field3_map</td>
        <td align="center">field3_mdspan</td>
    </tr>
    <tr>
        <td rowspan="3" align="center">windows</td>
        <td align="center">msvc</td>
        <td align="center">6.1776</td>
        <td align="center">3.916</td>
        <td align="center">1.37877</td>
        <td align="center">1.03</td>
        <td align="center">6.43983</td>
        <td align="center">6.26672</td>
    </tr>
    <tr>
        <td align="center">g++</td>
        <td align="center">5.63062</td>
        <td align="center">4.18603</td>
        <td align="center">1.12186</td>
        <td align="center">1.06282</td>
        <td align="center">5.05098</td>
        <td align="center">5.11213</td>
    </tr>
    <tr>
        <td align="center">clang++</td>
        <td align="center">3.14699</td>
        <td align="center">1.68778</td>
        <td align="center">1.19851</td>
        <td align="center">1.13926</td>
        <td align="center">1.99766</td>
        <td align="center">1.93895</td>
    </tr>
    <tr>
        <td rowspan="2" align="center">linux</td>
        <td align="center">g++</td>
        <td align="center">1.90795</td>
        <td align="center">1.86067</td>
        <td align="center">1.42331</td>
        <td align="center">1.42438</td>
        <td align="center">1.87503</td>
        <td align="center">1.92114</td>
    </tr>
    <tr>
        <td align="center">clang++</td>
        <td align="center">1.03195</td>
        <td align="center">0.96762</td>
        <td align="center">1.10214</td>
        <td align="center">1.0981</td>
        <td align="center">1.04556</td>
        <td align="center">1.0522</td>
    </tr>
</table>

这里并没有给出硬件参数和编译器版本

这说明：

1. 操作系统、编译器类型与性能具有很大关系

2. clang 在 linux 上的表现最好，使得不同实现版本的性能表现趋于一致。这同时也证明了，将三维索引映射到一维内存索引的策略是有意义的

这个测试表明了操作系统、编译器类型对性能具有不可忽视的影响，如果可能的话，可以尝试更改编译器来查看性能差异。例如编译 MPI 程序时也可以更换到 clang 编译。

```shell
mpicxx -cxx=clang++ main.cpp -O3 -I3rdparty/mdspan/include -march=native -o main_mpicxx_warpper
```

<script src="https://utteranc.es/client.js"
        repo="CheapMeow/cheapmeow.github.io"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>