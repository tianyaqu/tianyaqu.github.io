---
layout: post
title: 分离一个核模板
date: 2014-04-10 12:54:40
categories: 信号与图像
tags: [sperable filter,convolution]
comments: true
---
数字图像处理经常用卷积来作滤波，卷积核的大小的选择对计算量的影响很大，在一个卷积核大小<script type="math/tex">M\times N</script>区域内的运算量为<script type="math/tex">M^ 2\times N^ 2</script>。而我们知道，卷积满足结合律，

<script type="math/tex">I \ast  h = I \ast  (h1\times h2)\ = I \ast  h1 \ast h2\quad </script>

这样，如果能将滤波器（二维）转化成两个一维的乘积形式，计算量就降低到<script type="math/tex">2 \times M \times N</script>，这样会大大简化计算量，尤其是是当图像比较大的时候，计算量的优势就越明显。[这篇文章](http://blog.csdn.net/zddblog/article/details/7450033)给出了不同高斯实现的性能比较。
最典型的应用当属二维高斯滤波了，二维高斯核刚好等同于两个一维高斯的乘积，二维高斯核函数可以用一维的高斯核分别在x轴和y作卷积来代替。

<!--more-->

    % kernel width  6*sigma (-3sigma,3sigma) should be ok
    cutoff = ceil(3*sigma);  
    % 1D gauss kernel.width puls one to make it symentric
    h = fspecial('gaussian',[1,2*cutoff+1],sigma);  
    out = conv2(h,h,I,'same');
    

OK，高斯这个是情况特殊（恩，高斯可是跟上帝掷色子的人）。现在我们有任意一个核函数模板，怎么去分解这个矩阵呢？
这里有两种方法，先说一下简单的方法。它的思路是利用可分离核函数的对称性，任意取出一列Cn，作为分解列向量h1，取对应行Rn，同时为了保证归整，h2 = Rn/K(n,n)作为行向量。不过，可能会遇到除0的情况，所以更普遍意义上是这么实现的：

    % Pick the column with largest values
    [~,I] = max(sum(abs(h))); 
    h1 = h(:,I);
    % Pick the row with largest values
    [~,I] = max(sum(abs(h),2)); 
    h2 = h(I,:)/h1(I);

第二种类方法，充分利用了svd的强大计算能力。svd可以将一个矩阵（核模板只是一个方阵）分解成两个正交矩（这两个矩不正交）夹乘一个对角阵，

<script type="math/tex">A=US{ V }^{ T } </script>

那么，很容易得到 AV = US，又由于V是一个正交矩阵，取任一列作<script type="math/tex">V_i</script>,有

<script type="math/tex">A{V_i}{V_i}^{T}=AE=A </script>

<p>如此，便得到 <script type="math/tex">AV_i</script>与 $V_{i}^T$ 为卷积核的两个一维分量。</p>

    [u,s,v] = svd(h);
    s = diag(s);
    if sum(s > eps('single'))==1
        h1 = u(:,1)*s(1);
        h2 = v(:,1)';
    else
        error('Kernel not separable!')
    end

注：文中的代码来源于 [Cris's Image Analysis Blog](http://www.cb.uu.se/~cris/blog/index.php/archives/288)
ps: 
怎样才能确认该核模板为可分离（seperable）呢？答：秩为1;
这方法是不是降维啊？肯定是。挖个坑，以后补上PCA。
