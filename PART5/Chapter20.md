第二十章 实时光线跟踪的纹理LOD策略
Tomas Akenine
摘要
在光栅化中，我们依赖于像素四周的偏导数来过滤纹理，但是在光线跟踪中，我们需要使用其他的方法。在这里我们给出两种方法在光线跟踪中计算纹理LOD。第一种方法是使用射线微分，这是可以获得高质量结果的一般做法，它的缺陷在于有大量的计算以及需要很多射线的存储。第二种方法是使用射锥追踪，并且使用单一的三线性查询，这种方法相较于射线微分方法来说，有更小的内存开销和更少的计算量。我们将介绍如何在DirectX Raytracing（DXR）中实现射线微分，以及如何将它们与G缓冲绑定从而实现元可见性。我们提出一种计算质心微分的新方法，此外，我们给出了之前没有发表的射锥方法的详细细节，并且与Mip层级0的双线性过滤进行了比较。

20.1 简介
Mipmapping[17]是一种避免纹理走样的标准方法，在光栅化中，所有的GPU硬件都支持这种技术。比如OpenGL[7，15]，指定LOD的参数λ为：
式（1）
上式中，(x,y)是像素坐标，函数ρ为：
式（2）
(s,t)是2维纹理查找的纹理坐标，比如通常使用的场景是纹理坐标(∈ [0, 1]2)与纹理的分辨率相乘。如图20-1。这些函数有助于确保采样mipmap的层次结构，使屏幕空间像素映射到大约一个纹素。一般来讲，这些微分量都是由GPU硬件来计算，计算时会使用2×2的像素方格来计算每一个像素之间的微分。注意，公式2并不遵守单一的三线性查找，因为它不会计算出最小的包含块。在包含块中最大一面的可以如此计算：ρ(x, y) = max(| ∂s/∂x| + | ∂s/∂y| , | ∂t/∂x| + | ∂t/∂y|)。OpenGL 允许我们使用比公式2更保守的估计，但是我们并没有发现更好的方法或者实现。因此，GPU纹理使用的很多方法会产生模糊和走样。

图20-1. 在纹理空间，像素的近似面积是一个平行四边形。在公式2中使用该记录法。

在光线跟踪中，需要有计算纹理LOD的方法，并且它可以在递归射线路径时使用。由于像素四边形一般不能用于光线跟踪(眼睛光线除外)，因此需要考虑其他方法。本章描述了两种实时光线跟踪的纹理方法。第一种是射线微分[9]，它使用链式法则推导出能够精确计算纹理面积的表达式，即使对于镜面反射和折射也同样适用。射线微分需要大量的计算，每条射线需要大量的数据，但是它可以提供高质量的纹理过滤。第二种是射线锥，它的成本低于第一中方法，它使用锥来表示射线空间容量，它的大小取决于相交表面的距离，远的话放大，近的话缩小。我们描述了这两种方法在DXR中的实现方法。有关如何在光线跟踪引擎中过滤环境映射查找的信息，请参见第21章。

20.2 背景
通常我们使用分层图像(称为mipmap)在过滤纹理中进行加速[17]。每一个像素大小会映射到纹理空间，并且计算λ值。这个λ值以及当前片段的纹理坐标，用于从mipmap中收集和过滤八个采样点。Heckbert[7,8]研究了各种纹理滤波技术McCormack等人[10]根据之前的方法进行研究提出了一种各向异性采样的方法。Greene和Heckbert[6]提出了椭圆加权平均(EWA)滤波器，该滤波器通常被认为是质量最好、性能合理的方法。EWA在纹理空间中计算椭圆轨迹，并通过高斯权值的查找对mipmap进行采样。EWA既可以用于光栅化，也可以用于光线跟踪。
Ewins等人[5]提出了纹理LOD的各种近似，我们建议读者可以适当参考他们的方法。例如，他们用LOD描述了一个粗略的近似。计算公式如下：
式（3）
其中变量ta和pa分别是纹理空间面积的两倍和屏幕空间三角形面积的两倍。ta，pa的计算如下：
式（4）
其中w×h为纹理分辨率，Ti = (tix, tiy)为每个顶点的二维纹理坐标，Pi = (pix, piy)， i∈{0,1,2}，是屏幕空间三角形的三个顶点。在世界空间中两倍三角形面积的计算如下：
式（5）
其中Pi在世界空间中。我们利用这种设置作为射线锥过滤解决方案的一部分，如果三角形在z=1的平面上，公式3给出像素和纹素之前一对一的映射。在这种情况下，Δ可以看作一种三角形的基础纹理LOD。
Igehy[9]提出了第一种用于光线跟踪的纹理过滤方法。他使用射线微分，通过场景跟踪射线，并应用链式法则来建模反射和折射。计算出来的LOD既可用于常规mipmapping，也可用于各向异性采样mipmapping。射线跟踪的另一种纹理方法是基于锥[1]的。最近，Christensen等人[3]透露他们在电影渲染中也使用了一种光线锥的表示方法来过滤纹理，即类似于20.3.4节所描述的方法。

20.3 纹理LOD算法
本节描述用于实时光线跟踪的纹理LOD算法。我们改进了射线锥方法(第20.3.4节)，使其在第一次命中时处理曲率，这大大提高了质量。我们还扩展了与G-buffer一起使用的射线微分，从而提高了性能。此外，本文还提出了一种计算质心微差的新方法。

20.3.1 MIP层级0的双线性滤波
访问纹理的一个简单方法是采样MIP层级0。每个像素使用很多光线就能产生高质量的图像，但是性能会受到影响，因为重复的层级0访问常常导致糟糕的纹理缓存。当每像素只有几束光线时，尤其是在微型化的情况下，质量会受到比较大的影响。这时启用双线性过滤会对质量有一些小的改善。然而，当使用降噪器作为后处理时，有时候产生的模糊的结果，这时候可以使用双线性过滤。

20.3.2 射线微分
假设射线的数学表示为(参见第2章)
式（6）
其中O是射线的原点，ˆd是归一化的射线方向，也就是说ˆd = d /‖d‖。相关的射线微分是由四个向量组成：
式（7）
其中(x, y)为屏幕坐标，相邻像素之间为一个单位。其核心思想是在光线在场景中反弹时，沿着每条路径跟踪光线差。不管光线穿过的介质是什么，对所有沿着路径的交点进行偏微分，并且对入射射线进行微分从而产生出射射线的微分。当索引到一个纹理时，当前的光线微分决定了纹理占用的空间。大多数情况射线微分的方程[9]都可以被采用，但眼睛射线方向的微分需要修改。本文对质心坐标微分计算进行了优化。

20.3.2.1 视方向射线构建
对于w×h屏幕分辨率，坐标(x, y)像素处的非归一化视方向射线d通常在DXR中生成的方式如下：
式（8）
或者可以对上面的设置进行小的修改。上式中，p ∈ [0, 1]2，其中加0.5是获得像素的中心点，也就是说，在DirectX和OpenGL中是一致的，因此c ∈ [−1, 1]。在右手坐标系中，正交摄像机的基向量为{r′, u′, v′}，其中，r′是右向量，u′是上向量，v′是视向量，它指向摄像机的位置。注意，我们在方程8中使用{r, u, v}，这些是相机基向量的缩放版本，即，
式（9）
其中a是屏幕宽高比，f=tan (ω/2)，这里ω是垂直方向的FOV。
对于视方向射线，Igehy[9]计算的射线微分为：
式（10）
其中r-是从一个像素到下一个像素的右向量，u-是向上的向量，从式8推导可得，在本文最终是：
式（11）
以上就是设置眼睛光线的光线微分所需要的全部内容。

20.3.2.2 质心坐标微分计算的优化
三角形上的任何点都可以用质心坐标(u, v)表示为：P0 + ue1 + ve2，，其中e1= P1 - P0，e2 = P2 - P0。当我们获得一个交点，我们需要去计算该交点的微分，也就是说，∂u/∂x, ∂u/∂y, ∂v/∂x, and ∂v/∂y。现在，假设P为空间中的任意一点，g为与三角形平面不平行投影向量。点P=（px,py,pz）可以表示为：
式（12）
s为投影的距离。参见图20-2.

图20-2 质心坐标微分计算的推导过程
这个设置类似于一些射线/三角形相交测试[11]中使用的设置，它可以表示为一个可以使用克莱默规则求解的线性方程组，结果如下：
式（13）
其中k = (e1 × e2) ⋅ g。从这些表达式中，我们可以看出
式（14）
这些表达式在后面的推导中会很有用。接下来，假设交点为，P=O+td（射线方向d不需要归一化），也就是说我们可以计算∂P/∂x：
式（15）
其中q = ∂O/∂x + t(∂d/∂x)。同样可以得出∂P/∂y，其中r = ∂O/∂y + t(∂d/∂y)。使用公式14，15以及链式法则可以得出：
式（16）

接下来，我们选择g=d，而且(e2 × d) ⋅ d = 0，我们可以将前面的表达式简化为：
式（17）
现在，我们所求的表达式可以总结为：
式（18）
其中
式（19）
注意，q和r通过公式7的射线微分以及参数t进行表示。此外，由w=1-u-v可得：
式（20）
同理可计算出∂w/∂y。
当(u, v)的微分计算出来，就可以用来计算相应的纹理空间微分，通过公式2可得：
式（21）
其中w×h为纹理分辨率，g1 = (g1x, g1y) = T1−T0, g2 = (g2x, g2y) = T2−T0为相邻顶点纹理坐标的差值。类似地，下一步的反射/折射光线的射线原点O '的微分可计算为：
式（22）
与Igehy之前的实现[9]相比，我们看到使用这种方法会带来更好的效果。

20.3.3 射线微分与G-Buffer
对于实时光线跟踪，常见的是使用视射线光栅化获得G-buffer。当将射线微分[9]与G-buffer相结合时，可以像通常一样创建视射线的射线微分，因为第一次命中没有原始的几何图形，所以该次交互必须使用G-buffer中的内容。在这里，本文提出一个使用G-buffer的方法，假设我们通过法线n和距离t创建了从摄像机发出的第一个命中点(或者世界的位置)。本文描述了如何从G-buffer建立第一个反射射线微分。本文的方法是简单的去读取G-buffer中当前像素的上方和右方两个像素去创建射线微分。当前像素(x,y)，相邻像素的(x+1,y),(x,y+1)的法线和距离t都是从G-buffer读取。我们使用n0:0表示当前的像素，n+1:0表示右方的像素，n0:+1表示上方的像素，其他变量类似。那么，接下来计算这些相邻像素的视射线方向e。这里，我们可以计算出第一次相交原点的射线微分：
式（23）
同理可得∂O/∂y。射线方向微分可以用以下公式计算：
式（24）
其中r是着色函数 reflect（）。同样可得，∂d / ∂y。这里，我们已经计算出所有射线微分的组成部分，
{∂O / ∂x, ∂O / ∂y , ∂dˆ / ∂x, ∂dˆ / ∂y}，然后我们可以从这里开始之后的射线追踪。
上面的方法是高效的，但是会存在周围像素点不在一个表面上的情况。简单的解决方法为，对距离t进行检测，如果│t+1:0 − t0:0│ > ε，其中ε为一个极小数，如果差值为极小数，那么我们可以访问G-buffer中的-1：0点作为代替。在y方向上同样可以应用该方法。这种方法会慢一些，但是在深度不连续的部分会有较好的结果。

20.3.4 射线锥
一种计算纹理LOD的方法是基于跟踪锥。这个和Amanatides[1]提出的方法非常类似，但我们仅仅使用该方法来做纹理LOD，接下来我们会详细的描述实现细节。核心的方法见图20-3。当一个像素的纹理LOD λ计算出后，在GPU中使用三线性mipmap的纹理采样。

图20-3 图中展示了如何通过像素去创建射线锥，并且如何在场景中传输，增长和收缩。假设矩形是有纹理的，而且是完全反射的，我们会在交点处使用锥体的宽和发现来进行纹理查找，并且会将纹理反射到接下来的物体。接下来会介绍如何计算锥体的宽度。

在这个部分，我们使用射线锥来近似推导纹理LOD。本文使用锥体来在屏幕空间的mipmap的近似推导为起点，使用递归光线跟踪来处理反射。理想情况下，我们会对所有表面交点类型做处理，同时我们需要注意如下几个情况，见图20-4。但是不包括双曲抛物面的鞍点。

图20-4 图示为平面（左），凸面（中），凹面（右）的锥体反射。注意，凸面会增加锥体的角度，凹面会减少直到为零，然后开始重新增加。

20.3.4.1 屏幕空间
图20-5展示出如何构建通过一个像素的锥体的几何形状。其中以一个像素为单位的扩展角度为α，d0是从摄像机指向交点的向量，n0是交点的法线。这个锥体会从像素开始追踪，并且在接下来每个相交的表面更新锥体的参数。

图20-5 锥几何体的构建

锥体的宽度会随着距离而增长。在第一个交点处，这个锥体的宽度为：w0 = 2‖d0‖tan(α/2) ≈ α‖d0‖，这里0代表第一个交点的索引值。这个索引值在下面的部分会广泛使用。本文会使用小角度的三角近似，也就是说，tan α ≈ α。在相交表面的投影也会随着-d0和n0角度的变化而变化，使用[-d0,n0]来表示。直观来看，角度越大，这条射线可以看到更多的表面，同时LOD应该增加，也就是说纹素访问应该使用高级别的mipmap。结合近似投影的因子为：
式（25）
其中∣n0ˆ.d0ˆ∣为投影面积的平方根。绝对值是用来以相同的方式处理正面三角形和背面三角形。当(−d0, n0) = 0,这种情况我们仅会依赖距离，随着[−d0, n0]的增长,投影的范围会向无穷大靠近，也就是当(−d0, n0)→π/ 2时。

如果表达式25的值加倍或者减半，这时我们就应该获取mipmap更高一级或者更低一级。因此这里我们使用log2来计算。因此，对于第一次命中的纹理LOD，即与GPU生成的屏幕空间mipmap类似，为：
式（26）
其中Δ0由世界空间的顶点以及公式3和5得出。这里，Δ0是透过一个像素看到的三角形的基础纹理LOD，也就是说，此处没有任何反射。当三角形位于z = 1时，需要添加这一项来提供一个合理的LOD基数。这种情况考虑了三角形顶点和纹理坐标的变化。例如，如果一个三角形变得两倍大，那么基础LOD需要减1。在公式26中可得，如果距离或者入射角度增加，那么也会导致LOD层级上升。

20.3.4.2 反射
接下来，本文要在上一部分的基础上推广去处理反射的情况。我们对于构建反射的推导可见图20-6，我们会在反射的交点计算宽度w1。注意角β是交点表面的曲率（扩展阅读可见20.3.4.4），它会根据不同的表面交点影响传输角度增长多大或者收缩多大。见图20-4。首先我们可以得出：
式（27）
式（28）



图20-6 左上：纹理LOD反射的几何体构建，摄像机出发的射线在第一个交点处反射，此处构建绿色的和蓝色的共线。反射后的交点是绿色直线上的黑色圆点。右下：绿色蓝色共线部分放大的视图。我们希望能计算出宽度w1。注意，此处表面的传输角度β表明了锥体如何根据交点的曲率放大和缩小，该图示中是个凸面，所以锥体的范围会增大（β >0）。
接下来，我们使用公式27计算得出t′，然后将它代入公式28中，并得到：

式（29）
在最后一步我们用小角度近似tanα≈α。直观地说,这个表达式是有意义的，因为由w0≈α‖d0‖可得，锥体的范围会随着和第一个交点和眼睛距离乘以像素的大小α增加而增加，第二项是由第一个交点和第二个交点决定的，它依赖于距离t1（第一个交点和第二个交点的距离）以及角度α + β。

20.3.4.3 像素扩散角度
这一部分，我们提供一个计算像素扩散角度α的简单方法，也就是说对主要的射线。从相机到像素的角度会随着屏幕而变化，但是我们选择使用一个单一的值作为所有像素的近似值，也就是为了更快的计算速度牺牲了精度。
式（30）
其中ψ是垂直的视野，H是像素的高度。注意，α角是像素在中心位置的角度。
虽然有更精确的方法来计算像素扩散角，但是上面的近似方法会有较好的结果，所以这里会选择该方法。在极端情况下，例如，虚拟现实，可能需要使用一个更复杂的方法，比如漏斗渲染器与眼部跟踪[12],这种情况下希望使用一个更大的α角度。

20.3.4.4 反射中的表面扩散角度
图20-4中展示出各种不同的几何表面的反射，平面，凸面和凹面。另外，图20-6展示出表面扩散角β，这个角在平面情况下是0，凸面情况下是大于0，凹面情况下是小于0。直观的来看，β是根据交点表面的曲率而来的额外的扩散角度。一般情况，在交点处或者平均曲率法线的半径内有两个主要的曲率参与扩散角的计算。我们选择了一种更简单和更快的方法作为替代，只使用一个数字β表示曲率。

如果光栅化的，那么G-buffer可以用来计算表面扩散角。尽管可能还有其他可行的方法，这就是我们在这里采用的方法。片段的法线n和位置P都存储在世界空间中，我们使用ddx和ddy(HLSL语法)来获得它们的微分。点P在x处的微分为∂P/∂X。

图20-7左侧的部分展示第一次计算β涉及到的几何部分。从图中我们可知，
式（31）


图20-7 左：计算ϕ的几何部分。右：视向量v沿着表面法线反射，生成r。如果法线n经过扰动到n‘，那么我们会得到r’。因为[−v, n′] = θ + ϕ/2，那么[r ′, n′] = θ + ϕ/2，也就得到[r, r ′] = ϕ，[n, n′] = ϕ/2。

在图20-7的右半部份可以看出，这里法线角度的变化为ϕ/2，而在反射中会增大两倍。也就是说β = 2ϕ。这里为了计算β增加两个常量k1和k2，以及一个符号因子s（下面的公式详述），结果可得β = 2k1sϕ + k2，默认值为k1=1,k2=0。总结如下：
式（32）
β若为正值表示这是凸面,而负值表示这是凹面。注意，这里ϕ一直为正。因此，基于表面的类型，我们可以通过s因子切换β的符号。如下：
式（33）
如果参数大于0，则返回1，否则返回- 1。∂P/∂x和∂n/∂x (y也是一样)计算的基本原理是，当局部几何是凸的时候，∂P/∂x和∂n/∂x (y也是一样)的方向大致相同，当局部几何是凹的时候，∂n/∂x的方向大致相反(负的点积)。注意，有些曲面，例如双曲抛物面，在曲面上的所有点上都是凹的和凸的。在这些情况下，我们发现使用s = 1更好。如果想要有光泽的外观，可以增加k1和k2的值。对平面表面，ϕ为0，这意味着k1没有任何影响，使用k2因子代替。

20.3.4.5 归纳
我们从索引0开始对射线路径上的交点进行枚举，记做i。所以，第一个交点索引为0，第二个为1，以此类推。第i个交点的纹理LOD由以下得出：
式（34）
与公式26类似的，会根据距离和法线计算。参考图20-6，ni是第i个交点的表面法线，di是上一个交点到该交点的距离向量。Δi为第i个交点的基础的三角形LOD。跟之前类似，ˆdi为di的归一化向量。注意，我们在式34中添加了两个绝对值函数，因为计算距离的部分有可能会有负数，比如，凹面的点（见图20-4右侧部分）。法线的绝对值是用来以一致的方式处理背面三角形。

注意，w0 = αt0 = γ0t0以及w1 = αt0 + (α + β0)t1 = w0 + γ1t1，这里我们引入γ0 = α和γ1 = α + β0，其中β0为第一次交点的扩散角度。因此，式34可以递归的处理，见下面的公式，之后我们将会在20.6的部分以伪代码描述。
式（35）
其中γi = γi − 1 + βi − 1。可以参考图20-3。

20.4 实现
我们在Falcor[2]技术的基础上通过directx12和DXR上实现了射线锥和射线微分。我们根据方程34和35计算出λi，将它作为参数传递到SampleLevel()函数进行锥的纹理查找。
由于所有光线共享一个单一的原点，基于此光栅化在基元可视性方面进行了高度优化，所以我们对光线锥和20.3.3节中的光线微分方法使用G-buffer通道。当G-buffer被使用时，光线跟踪从第一个交点就由G-buffer存储。因此，使用本章的方法来获得第一个交点的GPU纹理单元，在之后的计算中使用λ。对于射线锥，βi将会通过G-buffer的微分来计算得到，其会根据第一个交点的曲率估计值β0计算。在我们的视线中，我们会在i>0时使βi = 0。这意味着在第一个命中点之外，所有的交互作用都假定为平面的。这是不正确的，但给出了合理的结果，而第一个交点是最重要的。然而，当对表面进行递归纹理反射时，这种近似会产生误差，如图20-8所示。

图20-8 放大桌面反射的花瓶底座，可以看到在递归反射方面光线锥方法比基于射线微分的方法弱。在射线锥图像的下方，由于我们的视线中假设第一次命中之后的所有表面都是平面，图像中存在大量的走样。
接下来，我们讨论了射线锥方法的精度。每条射线需要发送的数据为2个浮点数，wi和γi。对于β，wi和γi，我们已经尝试了fp32和fp16精度(G-buffer)，结论是16为的精度会有有较好的品质。在双曲抛物面场景中，我们无法直观地检测到任何差异，最大误差为5个像素分量差(大于255)。在应用、纹理和场景几何中，使用fp16是有效的，特别是在需要减少G-buffer存储和ray有效负载时。同样的，计算β使用小角度近似引起的误差(tan(α)≈α)并没有导致视觉效果的差异。通过逐像素图像的差异，我们可以看到另一组错误稀疏地分布在表面上，最大像素的分量偏差为5。这是另一个需要权衡的问题。
可以提前计算出静态模型的每个三角形Δ(公式3)，并将它存储在材质可访问的缓冲区中。然而，我们发现当每一次命中最近的三角形时重新计算Δ一样快。因此，射线锥方法处理动画模型时，该方法在处理每个三角形的多纹理坐标层时没有额外的开销。注意，在公式34中当两个向量的角度接近π/2时，∣niˆ dˆi∣ 的值趋近于+0.0。这并不是一个问题，因为在浮点数的IEEE标准754下，log2 (+0.0) = –inf，这里我们就取λ = inf。反过来，当角度接近π/2时，这里将强制使用三线性查找去访问mipmap的最上层。
我们的射线微分实现很好地遵循了Igehy[9]的描述。然而，除非另外有提到，我们都是使用方程1和2中的λ计算，和20.3.2.1和20.3.2.2中的方法。对于射线微分，每条射线需要12个浮点数用来存储。

20.5 对照及结果
我们在本节中使用的方法是:
>>Groundtruth：一个地面实况渲染器（每个像素点1024个采样数的射线追踪）
》Mip0：Mip等级0的双线性过滤器
》RayCones：射线锥方法（章节20.3.4）
》RayDiffs GB：使用G-buffer的射线微分（章节20.3.3）
》RayDiffs RT：我们的实现，使用基于射线微分的射线跟踪[9]
》RayDiffs PBRT：在pbrt渲染器中实现的射线微分[14]
注意，Mip0，RayCones，以及RayDiffs GB都是使用G-buffer作为主可视性，而RayDiffs RT和RayDiffs PBRT使用射线追踪。我们使用NVIDIA RTX 2080 Ti (Turing)，驱动版本416.16，来产生我们性能试验的结果。

为了验证我们对射线微分[9]的实现是正确的，我们将其与pbrt渲染器[14]中的实现进行了比较。为了可视化过滤纹理查找的mip级别的结果，我们使用了一个特殊的彩虹纹理，如图20-9所示。每个mip级别都设置为单一颜色。在图20-10中，我们在一个漫反射的房间中渲染了一个反射双曲抛物面。这意味着房间只显示从眼睛看到的mip层级，而双曲抛物面显示反射的mip层级，这在图20-10有详细说明。值得注意的是，在这些图像中可以看到双曲抛物面的三角形结构。其原因是质心坐标的微分在共享三角形边缘上不是连续的。对于光栅化也是如此，光栅化显示了类似的结构。因此，这种不连续会在递归反射中产生噪点，但它不会在我们视频中呈现的图像中显示出来。


图20-9 根据这张图片选择彩虹纹理中的mip层级颜色，也就是说，最底层的mip层级（层级0）是红颜色，层级1是黄颜色，以此类推。Mip层级6以及大于6的，都是白颜色。


图20-10 mipmap层级的可视化，其中红色为0级，黄色为1级，依此类推，如图20-9所定义。对于视射线，RayCones以及RayDiffs GB使用G-buffer通道，因此，我们使用与pbrt相同的公式在该通道中生成的纹理导数来计算mipmap级别，以便在地板上得到合理的匹配。由于双曲抛物面在所有点上都是凹的和凸的，我们在方程32中使用s = 1。注意，虽然和上图“彩虹”颜色着色并不完全匹配，但是焦点应该放在实际的颜色上。右边的三幅图像匹配得很好，而raycone则略有不同，尤其是在递归反射方面。这种差别是可以预料到的，因为这种方法假定反射在第一次反弹之后都是平面的。

图20-11展示了进一步的效果。我们选择了双曲抛物面(上)和双线性补片(下)，因为它们是鞍面而且是仅处理各向同性，这种情况对射线锥来说是比较困难的。选择半圆，因为当曲率沿着圆柱体的长度为零，并且在另一个方向上像圆一样弯曲时，对于使用射线锥来处理比较困难。因此，与射线微分相比，射线锥有时显得更加模糊。同样明显的是，与其他方法相比，Groundtruth图像的清晰度要高得多，因此过滤纹理还有很多需要改进的地方。这种过度模糊的结果是峰值信噪比(PSNR)和结构相似性指数(SSIM)值相对较差。对于双曲抛物面，也就是图20-11的第一行，Mip0的Groundtruth，Raycone，RayDiffsRT的PSNR对应值，分别为25.0、26.7和31.0 dB。Mip0的PSNR比预期的要低，但是即使对于其他方法，这个数字也是比较低的。这是它们产生更多模糊的原因。另一方面，这是他们产生走样比较少的原因。SSIM相关的参数0.95，0.95，0.97是一样的。


图20-11 使用不同技术的不同类型表面的纹理反射的比较。Groundtruth图像使用每像素1024个样本，并通过访问mipmap level 0对所有纹理进行渲染。Raycone，我们使用了方程33中的函数。
虽然静态图像可以很好的展示过度模糊，但是很难去展示走样的总量。因此，我们的大部分结果都显示在我们随附的视频中(可以在本书的网站上找到)，我们在下一段视频中会提到。我们将用mm:ss来表示视频中特定的时间，其中mm为分钟，ss为秒。
在视频00:05和00:15处，很明显，与Mip0相比，Raycone生成的图像具有更少的走样，正如期望的那样，因为反射对象总是使用层级0级的Mip。在一定距离上，Raycone也有少量的时间走样，但即使是mipmap的GPU栅格化也会造成走样。从00:25处开始，raycone和Mip0都在一个更大的场景中进行，在这个场景中，房间的条纹墙纸为Mip0产生了相当多的走样，而Raycone和RayDiffs RT的表现要好很多。

我们在粉色房间和大的室内进行了性能的测试。所有渲染图的分辨率都是3840×2160像素。为了忽略预热效果和其他随机变量，我们在不测量帧长的情况下，通过1000帧的相机路径渲染场景一次，然后在测量时再次通过相同的相机路径渲染场景。我们对每个场景重复这个过程100次，并收集帧的持续时间。对于Mip0，粉色房间的平均帧时间为3.4 ms，大型室内的平均帧时间为13.4 ms。在图20-12中，显示了Mip0、raycone、RayDiffs GB和RayDiffs RT。粉色房间是一个相当小的场景，其中纹理LOD的复杂计算只占总帧时间的一小部分，而对于大的室内场景(一个大得多的场景)，这种情况更加明显。然而，对于这两个场景，增加纹理LOD计算的时间的趋势是相当明显的：与Raycone相比，RayDiffs GB增加了大约2倍时间，而RayDiffs RT增加了大约3倍。


图20-12 纹理LOD的性能分析：粉红色房间（左），大室内场景（右）。较小的场景(粉色房间)不太容易受到额外过滤器消耗的影响，较大的场景(大型室内)更容易受到额外过滤器消耗的影响。然而，对于这两种场景，相比较Raycones而言，RayDiffs GB的消耗时间增加约为2倍，而RayDiffs RT约为3倍。
本章的目标是通过实现和评估不同的方法，为实时应用程序选择合适的纹理过滤方法提供一些帮助，并说明如何使用G-buffer以提高性能。当使用复杂的反射滤光器对结果进行模糊处理时，或者当积累了许多帧或样本时，建议使用Mip0方法，因为它更快并能有较好的质量。当需要得到较好的过滤反射结果，并且需要最小化射线存储和指令计数时，我们建议使用Raycone。然而，在第一次相交之后，曲率没有被考虑进去，这可能会导致在更深的反射路径中产生走样。在这些情况下，我们推荐使用RayDiffs方法。对于较大的场景，如Pharr所描述[13]，任何类型的纹理过滤都可能有助于性能，因为他们能提供更好的纹理缓存命中率。当对视射线使用光线跟踪时，我们看到使用纹理过滤比使用mip层级0，性能有轻微的改进。未来我们会在更大的场景中进行探索并获取有效的数据结果。

20.6 代码
这一小节，我们展示出当前Raycones方法的伪代码。首先，我们需要一些数据结构：
struct RayCone
{
float width; // Called wi in the text
float spreadAngle; // Called γi in the text
};
struct Ray
{
float3 origin;
float3 direction;
};
struct SurfaceHit
{
float3 position;
float3 normal;
float surfaceSpreadAngle; // Initialized according to Eq. 32
float distance; // Distance to first hit
};

在下一个伪代码中，我们遵循DXR流程来进行光线跟踪。我们提供了一个射线生成模块和一个最近相交模块，但是省略了其他几个在当前上下文中无用的模块。TraceRay函数遍历空间的数据结构，并找到最近的相交点。伪代码处理反射的递归。
void rayGenerationShader(SurfaceHit gbuffer)
{
RayCone firstCone = computeRayConeFromGBuffer(gbuffer);
Ray viewRay = getViewRay(pixel);
Ray reflectedRay = computeReflectedRay(viewRay, gbuffer);
TraceRay(closestHitProgram, reflectedRay, firstCone);
}
RayCone propagate(RayCone cone, float surfaceSpreadAngle, float hitT)
{
RayCone newCone;
newCone.width = cone.spreadAngle * hitT + cone.width;
newCone.spreadAngle = cone.spreadAngle + surfaceSpreadAngle;
return newCone;
}
RayCone computeRayConeFromGBuffer(SurfaceHit gbuffer)
{
RayCone rc;
rc.width = 0; // No width when ray cone starts
rc.spreadAngle = pixelSpreadAngle(pixel); // Eq. 30
// gbuffer.surfaceSpreadAngle holds a value generated by Eq. 32
return propagate(rc, gbuffer.surfaceSpreadAngle, gbuffer.distance);
}
void closestHitShader(Ray ray, SurfaceHit surf, RayCone cone)
{
// Propagate cone to second hit
cone = propagate(cone, 0, hitT); // Using 0 since no curvature
// measure at second hit
float lambda = computeTextureLOD(ray, surf, cone);
float3 filteredColor = textureLookup(lambda);
// use filteredColor for shading here
if (isReflective)
{
Ray reflectedRay = computeReflectedRay(ray, surf);
TraceRay(closestHitProgram, reflectedRay, cone); // Recursion
}
}
float computeTextureLOD(Ray ray, SurfaceHit surf, RayCone cone)
{
// Eq. 34
float lambda = getTriangleLODConstant();
lambda += log2(abs(cone.width));
lambda += 0.5 * log2(texture.width * texture.height);
lambda -= log2(abs(dot(ray.direction, surf.normal)));
return lambda;
}
float getTriangleLODConstant()
{
float P_a = computeTriangleArea(); // Eq. 5
float T_a = computeTextureCoordsArea(); // Eq. 4
return 0.5 * log2(T_a/P_a); // Eq. 3
}


致谢
感谢Jacob Munkberg和Jon Hasselgren的头脑风暴的帮助和建议。

参考文献
[1] Amanatides, J. Ray Tracing with Cones. Computer Graphics (SIGGRAPH) 18, 3 (1984), 129–135.
[2] Benty, N., Yao, K.-H., Foley, T., Kaplanyan, A. S., Lavelle, C., Wyman, C., and Vijay, A. The Falcor
Rendering Framework. https://github.com/NVIDIAGameWorks/Falcor, July 2017.
[3] Christensen, P., Fong, J., Shade, J., Wooten, W., Schubert, B., Kensler, A., Friedman, S.,
Kilpatrick, C., Ramshaw, C., Bannister, M., Rayner, B., Brouillat, J., and Liani, M. RenderMan:
An Advanced Path-Tracing Architecture for Movie Rendering. ACM Transactions on Graphics 37, 3 (2018), 30:1–30:21.
[4] do Carmo, M. P. Differential Geometry of Curves and Surfaces. Prentice Hall Inc., 1976.
[5] Ewins, J. P., Waller, M. D., White, M., and Lister, P. F. MIP-Map Level Selection for Texture
Mapping. IEEE Transactions on Visualization and Computer Graphics 4, 4 (1998), 317–329.
[6] Green, N., and Heckbert, P. S. Creating Raster Omnimax Images from Multiple Perspective Views Using the Elliptical Weighted Average Filter. IEEE Computer Graphics and Applications 6, 6 (1986), 21–27.
[7] Heckbert, P. S. Survey of Texture Mapping. IEEE Computer Graphics and Applications 6, 11 (1986), 56–67.
[8] Heckbert, P. S. Fundamentals of Texture Mapping and Image Warping. Master’s thesis, University of California, Berkeley, 1989.
[9] Igehy, H. Tracing Ray Differentials. In Proceedings of SIGGRAPH (1999), pp. 179–186.
[10] McCormack, J., Perry, R., Farkas, K. I., and Jouppi, N. P. Feline: Fast Elliptical Lines for
Anisotropic Texture Mapping. In Proceedings of SIGGRAPH (1999), pp. 243–250.
[11] Möller, T., and Trumbore, B. Fast, Minimum Storage Ray-Triangle Intersection. Journal of
Graphics Tools 2, 1 (1997), 21–28.
[12] Patney, A., Salvi, M., Kim, J., Kaplanyan, A., Wyman, C., Benty, N., Luebke, D., and Lefohn,
A. Towards Foveated Rendering for Gaze-Tracked Virtual Reality. ACM Transactions on Graphics
35, 6 (2016), 179:1–179:12.
[13] Pharr, M. Swallowing the Elephant (Part 5). Matt Pharr’s blog, https://pharr.org/matt/
blog/2018/07/16/moana-island-pbrt-5.html, July 16 2018.
[14] Pharr, M., Jakob, W., and Humphreys, G. Physically Based Rendering: From Theory to
Implementation, third ed. Morgan Kaufmann, 2016.
[15] Segal, M., and Akeley, K. The OpenGL Graphics System: A Specification (Version 4.5). Khronos Group documentation, 2016.
[16] Voorhies, D., and Foran, J. Reflection Vector Shading Hardware. In Proceedings of SIGGRAPH (1994), pp. 163–166.
[17] Williams, L. Pyramidal Parametrics. Computer Graphics (SIGGRAPH) 17, 3 (1983), 1–11.