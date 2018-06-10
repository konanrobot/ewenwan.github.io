---
layout: post
title: "单目SLAM基础知识"
date: 2018-06-09 
description: "特征点匹配，单应变换，对极几何，本质矩阵"
tag: SLAM 
---   

# 特征点匹配
        在讲解恢复R,T前，稍微提一下特征点匹配的方法。
        常见的有如下两种方式： 
        1. 计算特征点，然后计算特征描述子，通过描述子来进行匹配，优点准确度高，缺点是描述子计算量大。 
        2. 光流法：在第一幅图中检测特征点，使用光流法(Lucas Kanade method)对这些特征点进行跟踪，
           得到这些特征点在第二幅图像中的位置，得到的位置可能和真实特征点所对应的位置有偏差。
           所以通常的做法是对第二幅图也检测特征点，如果检测到的特征点位置和光流法预测的位置靠近，
           那就认为这个特征点和第一幅图中的对应。
           在相邻时刻光照条件几乎不变的条件下（特别是单目slam的情形），
           光流法匹配是个不错的选择，它不需要计算特征描述子，计算量更小。
           
# 单应矩阵H 恢复变换矩阵 R, t 
[单目视觉slam 基础几何知识](https://blog.csdn.net/heyijia0327/article/details/50758944)

```asm
        p2   =  H12 * p1  4对点   A*h = 0 奇异值分解 A 得到 单元矩阵 H ，  T =  K 逆 * H21*K 

        展开成矩阵形式：
        u2         h1  h2  h3        u1
        v2  =      h4  h5  h6    *   v1
        1          h7  h8  h9        1   
        按矩阵乘法展开：
        u2 = (h1*u1 + h2*v1 + h3) /( h7*u1 + h8*v1 + h9)
        v2 = (h4*u1 + h5*v1 + h6) /( h7*u1 + h8*v1 + h9)   
        将分母移到另一边，两边再做减法
        -((h4*u1 + h5*v1 + h6) - ( h7*u1*v2 + h8*v1*v2 + h9*v2))=0  式子为0  左侧加 - 号不变
        h1*u1 + h2*v1 + h3 - ( h7*u1*u2 + h8*v1*u2 + h9*u2)=0  
        写成关于 H的矩阵形式：
        0   0   0   0   -u1  -v1  -1   u1*v2    v1*v2   v2
        u1  v1  1   0    0    0    0   -u1*u2  -v1*u2  -u2  * (h1 h2 h3 h4 h5 h6 h7 h8 h9)转置  = 0
        h1~h9 9个变量一个尺度因子，相当于8个自由变量
        一对点 2个约束
        4对点  8个约束 求解8个变量
        A*h = 0 奇异值分解 A 得到 单元矩阵 H

        cv::SVDecomp(A,w,u,vt,cv::SVD::MODIFY_A | cv::SVD::FULL_UV);// 奇异值分解
        H = vt.row(8).reshape(0, 3);// v的最后一列

        单应矩阵恢复  旋转矩阵 R 和平移向量t
         p2 =  H21 * p1   = H21 * KP   
         p2 = K( RP + t)  = KTP = H21 * KP  
         T =  K 逆 * H21*K      
```
# 本质矩阵F求解 变换矩阵[R t]   p2转置 * F * p1 =  0

[参考](http://frc.ri.cmu.edu/~kaess/vslam_cvpr14/media/VSLAM-Tutorial-CVPR14-A11-VisualOdometry.pdf)

## 基本矩阵的获得
```asm
        空间点 P  两相机 像素点对  p1  p2 两相机 归一化平面上的点对 x1 x2 与P点对应
        p1 = KP 
        p2 = K( RP + t) 
        x1 =  K逆* p1 = P 
        x2 =  K逆* p2 = ( RP + t) = R * x1  + t 
        消去t(同一个变量和自己叉乘得到0向量)
        t 叉乘 x2 = t 叉乘 R * x1
        再消去等式右边
        x2转置 * t 叉乘 x2 = 0 = x2转置 * t 叉乘 R * x1
        得到 ：
        x2转置 * t 叉乘 R * x1 = x2转置 * E * x1 =  0  本质矩阵
        也可以写成：
        p2转置 * K 转置逆 * t 叉乘 R * K逆 * p1   = p2转置 * F * p1 =  0 基本矩阵
```
## 几何知识参考
        对极几何原理：
        两个摄像机的光心C0、C1，三维空间中一点P，在两幅图像中的位置为p0、p1（相当于上面的 x1, x2）。
        像素点 u1，u2
        p0 = inv(K) * u0
        p1 = inv(K) * u1
        
              x0    (u0x - cx) / fx 
        p0 =  y0 =  (u0y - cy) / fy
               1     1
               
              x1    (u1x - cx) / fx 
        p1 =  y1 =  (u1y - cy) / fy
               1     1        
        
        如下图所示：
![](https://img-blog.csdn.net/20160228210635964)

        由于C0、C1、P三点共面，得到： 
![](https://img-blog.csdn.net/20160228213615018)

         这时，由共面得到的向量方程可写成：  
         p0 *(t 叉乘 R * p1)
         其中，
         t是两个摄像机光心的平移量;
         R是从坐标系C1到坐标系C0的旋转变换，
         p1左乘旋转矩阵R的目的是把向量p1从C1坐标系下旋转到C0坐标系下(统一表示)。
         
         一个向量a叉乘一个向量b，
         可以表示为一个反对称矩阵乘以向量b的形式，
         这时由向量a=(a1,a2,a3) 表示的反对称矩阵(skew symmetric matrix)如下: 
         
               0  -a3  a2
         ax =  a3  0  -a1
              -a2  a1   0
![](https://img-blog.csdn.net/20160228220006072)

        所以把括号去掉的话，p0 需要变成 1*3的行向量形式
        才可以与 3*3的 反对称矩阵相乘
        
        p0转置 * t叉乘 * R * P1
        
       我们把  t叉乘 * R  = E 写成
       
       p0转置 * E * P1

![](https://img-blog.csdn.net/20160228221439077)

        本征矩阵E的性质： 
        一个3x3的矩阵是本征矩阵的充要条件是对它奇异值分解后，
        它有两个相等的奇异值，并且第三个奇异值为0。
        牢记这个性质，它在实际求解本征矩阵时有很重要的意义 .
        
       计算本征矩阵E、尺度scale的来由:
       将上述矩阵相乘的形式拆开得到 ：

![](https://img-blog.csdn.net/20160229091018025)
        
        上面这个方程左边进行任意缩放都不会影响方程的解： 
        
       (x0x1 x0y1 x0 y0x1 y0y1 y0 x1 y1 1)*E33(E11/E33 ... E32/E33 1) = 0
       所以E虽然有9个未知数，但是有一个变量E33可以看做是缩放因子，
       因此实际只有8个未知量，这里就是尺度scale的来由，后面会进一步分析这个尺度。
       
       AX=0，x有8个未知量，需要A的秩等于8，所以至少需要8对匹配点,应为有可能有两个约束是可以变成一个的，
       点可能在同一条线上，或者所有点在同一个面上的情况，这时候就存在多解,得到的值可能不对。
       
       对矩阵A进行奇异值SVD分解，可以得到A


## p2转置 * F * p1 =  0 8点对8个约束求解得到F
```asm
        *                    f1   f2    f3      u1
        *   (u2 v2 1)    *   f4   f5    f6  *   v1  = 0  
        *                    f7   f8    f9       1
        按照矩阵乘法展开：
        a1 = f1*u2 + f4*v2 + f7;
        b1 = f2*u2 + f5*v2 + f8;
        c1 = f3*u2 + f6*v2 + f9;
        得到：
        a1*u1+ b1*v1 + c1= 0
        展开：
        f1*u2*u1 + f2*u2*v1 + f3*u2 + f4*v2*u1 + f5*v2*v1 + f6*v2 + f7*u1 + f8*v1 + f9*1 = 0
        写成矩阵形式：
        [u1*u2 v1*u2 u2 u1*v2 v1*v2 v2 u1 v1 1]*[f1 f2 f3 f4 f5 f6 f7 f8 f9]转置 = 0
        f 9个变量，1个尺度因子，相当于8个变量
        一个点对，得到一个约束方程
        需要8个点对，得到8个约束方程，来求解8个变量
        A*f = 0
        所以F虽然有9个未知数，但是有一个变量f9可以看做是缩放因子，
        因此实际只有8个未知量，这里就是尺度scale的来由，后面会进一步分析这个尺度。 
        
        上面这个方程的解就是矩阵A进行SVD分解A=UΣV转置 后，V矩阵是最右边那一列的值f。
        另外如果这些匹配点都在一个平面上那就会出现A的秩小于8的情况，这时会出现多解，会让你计算的E/F可能是错误的。    
        
        A * f = 0 求 f   
        奇异值分解F 基础矩阵 且其秩为2 
        需要再奇异值分解 后 取对角矩阵 秩为2 后在合成F

        cv::Mat u,w,vt;
        cv::SVDecomp(A,w,u,vt,cv::SVD::MODIFY_A | cv::SVD::FULL_UV);// A = w * u * vt
        cv::Mat Fpre = vt.row(8).reshape(0, 3);// F 基础矩阵的秩为2 需要在分解 后 取对角矩阵 秩为2 在合成F
        cv::SVDecomp(Fpre,w,u,vt,cv::SVD::MODIFY_A | cv::SVD::FULL_UV);
        w.at<float>(2)=0;//  基础矩阵的秩为2，重要的约束条件
        F =  u * cv::Mat::diag(w)  * vt;// 在合成F

        从基本矩阵恢复 旋转矩阵R 和 平移向量t
                F =  K转置逆 * E * K逆
        本质矩阵 E  =  K转置 * F * K =  t 叉乘 R

        从本质矩阵恢复 旋转矩阵R 和 平移向量t
        恢复时有四种假设 并验证得到其中一个可行的解
        
        本征矩阵的性质： 
        一个3x3的矩阵是本征矩阵的充要条件是对它奇异值分解后，
        它有两个相等的奇异值，
        并且第三个奇异值为0。
        牢记这个性质，它在实际求解本征矩阵时有很重要的意义。
        
        计算本征矩阵E的八点法，大家也可以去看看wiki的详细说明 
```
[本征矩阵E的八点法](https://en.wikipedia.org/wiki/Eight-point_algorithm)

        有本质矩阵E 恢复 R,t
        从R,T的计算公式中可以看到R,T都有两种情况，
        组合起来R,T有4种组合方式。
        由于一组R,T就决定了摄像机光心坐标系C的位姿，
        所以选择正确R、T的方式就是，把所有特征点的深度计算出来，
        看深度值是不是都大于0，深度都大于0的那组R,T就是正确的。
![](https://img-blog.csdn.net/20160229102903105)


## 以上的解法 在 orbslam2中有很好的代码解法
[orbslam2 单目初始化 ](https://github.com/Ewenwan/MVision/blob/master/vSLAM/oRB_SLAM2/src/Initializer.cc)


<br>
*转载请注明原地址，万有文的博客：[ewenwan.github.io](https://ewenwan.github.io) 谢谢！*