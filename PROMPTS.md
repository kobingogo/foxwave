# Foxwave · 狐狸电波：完整制作提示词

这份文档整理了项目技术复盘中的完整提示词，保留总提示词、七个制作阶段、模型实测参考值、各阶段验收要求及纠错提示词，方便按顺序复制使用。提示词正文保持原文，不包含复盘正文及文章素材。

适用于仓库中的 `fox-web.glb` 静态狐狸模型，目标是制作一个单文件、可互动、可合成音乐和录屏分享的网页小产品。换用其他模型时，请重新测量部位坐标。

## 使用顺序

1. 先发送「总提示词」，新对话开始时再次发送。
2. 每次只发送一个阶段，运行并完成验收后再继续下一阶段。
3. 测量阶段需将探针输出反馈给模型；使用同一只狐狸时，可参考文档中的实测值。
4. 出现问题时，发送末尾对应的纠错提示词。

## 1. 为什么要这样写

非顶级模型做这类任务通常卡在：

- **一次写 3000 多行会前后不一致**：变量名漂移、忘了前面定义过的东西。
- **几何直觉弱**：不会自己去测量模型，倾向于"猜坐标"。
- **容易编 API**：three.js 版本差异大，比如 r186 删除了 `PCFSoftShadowMap`，`Clock` 也已弃用。
- **视觉问题无法自检**：说"已完成"，但实际是黑屏。

对策：

- **分阶段**：每阶段一个可运行、可验收的交付，下一阶段在上一阶段的代码上改。
- **把测量变成任务**：先写探针脚本，再用测出来的数字填常量。
- **给公式、给参数、给 GLSL 骨架**，不给"自由发挥"的空间。
- **每阶段附验收清单和已知坑**。

下面是可以直接复制的提示词。用法：**先发"总提示词"，然后每次只发一个阶段**，确认这一阶段跑通了再发下一个。

---

## 2. 总提示词（每次对话开头都带上）

```text
你是一名资深 WebGL / three.js 工程师。我们要把一个静态的 glb 动物模型做成一个可互动、可录屏分享的网页小产品。

【硬性约束】
1. 交付物始终是一个单文件 index.html，没有构建工具，不使用 npm。
2. three.js 固定为 0.186.1，用 import map 从 jsDelivr 加载：
   "three": "https://cdn.jsdelivr.net/npm/three@0.186.1/build/three.module.js"
   "three/addons/": "https://cdn.jsdelivr.net/npm/three@0.186.1/examples/jsm/"
3. 模型文件是同目录下的 fox-web.glb，使用了 Draco 压缩（KHR_draco_mesh_compression），
   必须挂 DRACOLoader，解码器路径：
   https://cdn.jsdelivr.net/npm/three@0.186.1/examples/jsm/libs/draco/gltf/
4. 模型是单网格、单材质（MeshPhysicalMaterial，带 map / normalMap / roughnessMap / metalnessMap），
   没有骨骼，没有 morph target，网格本身是很多不相连的碎块。不允许修改模型文件。
5. 所有"部位"（头、尾、眼、鼻、嘴、耳）都用包围盒归一化坐标 n = (p - box.min) / box.size 表示，
   写成命名常量，不要在代码各处散落魔法数字。
6. three.js r186 的注意事项：
   - 不要用 PCFSoftShadowMap（已移除），用 PCFShadowMap + light.shadow.radius 控制柔和度；
   - 用 THREE.Timer 代替已弃用的 THREE.Clock；
   - glTF 贴图 flipY = false，按 UV 读贴图像素时 y = v * H，不要取反；
   - 自己生成的 CanvasTexture 要设置 flipY = false 和 colorSpace = SRGBColorSpace。
7. 每次输出完整的 index.html，不要只给片段，不要写"其余代码不变"。
8. 代码注释用中文，说明"为什么"，而不是复述代码。

【工作方式】
- 我会分阶段给你任务，一次只做一个阶段。
- 每个阶段结束时，你要列出：本阶段改了什么、我应该如何验证（具体到点哪里、看到什么）、
  已知限制。
- 如果某个数值需要从模型里测量，先给我一段可以粘贴到浏览器控制台运行的探针脚本，
  等我把输出贴回来，再填常量。绝对不要猜坐标。
- 遇到不确定的 API，说明不确定，并给出最保守的写法。
```

---

## 3. 阶段 1：基础查看器

```text
【阶段 1：基础查看器】
做一个能正常显示模型的页面：
1. 加载 glb，加入场景（务必 scene.add(gltf.scene)），水平居中、最低点落到 y=0。
2. 渲染器：antialias、alpha（让 CSS 背景透出来）、ACESFilmic 色调映射、阴影开启。
3. 环境光照：RoomEnvironment + PMREMGenerator 生成环境贴图，scene.environmentIntensity = 0.5。
   一盏投影主光（方向光，shadow.mapSize 2048，shadow.radius 10），一盏冷色轮廓光。
4. 地面：一个径向渐变的 CanvasTexture 做 map 和 alphaMap 的圆盘，接收阴影，depthWrite=false。
5. OrbitControls：开阻尼，禁止平移，不能转到地面以下。
6. 相机自动取景：对包围盒 8 个角点求"刚好装下"的距离，再乘 1.4 留出边距；窗口缩放时保持当前视角，
   只按新宽高比调整距离。
7. 加载中和加载失败都要有提示卡片（失败时说明：需要用 http:// 打开、模型要在同一目录）。
8. 在 window.__fox 暴露调试对象：THREE, scene, camera, renderer, controls, model。

【验收】
- 控制台零报错；模型完整可见，有地面影子。
- 在控制台运行 new THREE.Box3().setFromObject(__fox.model).min.y 约等于 0。
- 宽屏、竖屏（420x900）都能完整看到模型。
```

---

## 4. 阶段 2：测量模型（只写探针，不改页面）

```text
【阶段 2：测量模型】
先不要改 index.html。请给我 4 段控制台探针脚本，我逐个运行后把输出贴回给你：

探针 A：按 y 把模型分 20 层，输出每层顶点的 x 范围和 z 范围（世界坐标），
        用来找脖子位置（身体和头之间宽度突变的那一层）。
探针 B：只取 y 在 [0.03, 0.38]（世界坐标）之间的顶点，在 x-z 平面上画 0.05 米一格的 ASCII 栅格
        （顶点多的格子用 #，少的用 +，有的用 .），用来找尾巴。
探针 C：对 z 归一化 > 0.78、y 归一化在 [0.42, 0.80] 的顶点，读取它对应的贴图像素
        （用 canvas 把 material.map.image 画出来再 getImageData；注意 y = v * H），
        在 x-y 平面上画 0.02 一格的 ASCII 栅格：平均亮度 < 0.15 用 #，< 0.3 用 %，> 0.75 用 o。
        瞳孔会显示成两团 #，鼻子是 %。
探针 D：输出 material 上哪些贴图存在（map、normalMap、roughnessMap、metalnessMap、aoMap……）
        以及 roughness、metalness 的值。

所有输出都用归一化坐标标注行和列。拿到我的输出后，你要给出这些常量：
HEAD_PIVOT、TAIL_PIVOT、EYE_L、EYE_R、NOSE、MOUTH、EAR_L、EAR_R，以及眼睛的半宽和半高。
```

> 参考值（这只狐狸实测）：HEAD_PIVOT (0.38, 0.40, 0.58)，TAIL_PIVOT (0.62, 0.14, 0.30)，EYE_L (0.27, 0.62)，EYE_R (0.495, 0.62)，眼睛半宽 0.062、半高 0.06，NOSE (0.38, 0.545)，MOUTH (0.38, 0.497)，EAR_L (0.22, 0.76)，EAR_R (0.56, 0.76)。如果你用的就是这只狐狸，可以直接把这些值写进提示词，跳过阶段 2。

---

## 5. 阶段 3：无骨骼动画（点头、摇尾、整体律动）

```text
【阶段 3：在顶点着色器里做骨骼动画】
用 material.onBeforeCompile 注入 GLSL，不修改几何数据。

1. 结构：把模型放进一个 rig Group，rig 的原点在狐狸脚下（头部枢轴的 x、z，y=0）。
   记录模型网格在"rig 为单位矩阵时"的 matrixWorld 作为 uRest，以及它的逆 uRestInv。
2. uniforms：uRest, uRestInv, uBoxMin, uBoxSize, uHead(vec3: pitch, yaw, roll), uTail(vec2: 横摆, 上翘)。
3. 在 #include <beginnormal_vertex> 之后调用 foxDeform(position, objectNormal)，
   在 #include <begin_vertex> 处用变形后的位置。foxDeform 的步骤：
   a. wp = uRest * p；n = (wp - uBoxMin) / uBoxSize
   b. 尾巴权重 tw = smoothstep(0.60,0.74,n.x) * (1-smoothstep(0.34,0.46,n.z)) * (1-smoothstep(0.36,0.46,n.y))
      （阈值换成阶段 2 测到的）
   c. 头部权重 hw = smoothstep(0.36,0.50,n.y) * (1 - tw)
   d. R = rotY(yaw*hw) * rotX(pitch*hw) * rotZ(roll*hw)；wp = 枢轴 + R*(wp-枢轴)；法线也乘 R
   e. 尾巴：绕尾根旋转，旋转量乘以 reach = clamp(到尾根的水平距离 / (0.32*boxSize.x), 0, 1)
   f. 变回局部：p = uRestInv * wp；法线用 mat3(uRestInv)
   注意 GLSL 的 mat3 构造是按列填的；smoothstep 的 edge0 必须小于 edge1，反向请用 1 - smoothstep。
4. 阴影：新建一个 MeshDepthMaterial(RGBADepthPacking)，打同样的补丁（只处理位置），
   赋给 mesh.customDepthMaterial。
5. customProgramCacheKey 要区分普通材质和深度材质。
6. JS 端写一个 headPoint(restPoint) 函数：用同样的欧拉角顺序（YXZ）计算静止点在当前姿态下的世界坐标。
7. 动作：头部看向鼠标（射线和"头部前方 0.6 米、朝向相机"的平面求交，算出 yaw 和 pitch，并做限幅）；
   没有鼠标时看相机。每隔几秒随机歪一下头。尾巴慢慢晃。用 sin 做呼吸。
   所有目标值都用指数平滑逼近：x += (目标 - x) * (1 - exp(-dt * k))。

【验收】
- 鼠标移动时狐狸转头看鼠标，脖子处没有撕裂，影子跟着动。
- 在控制台运行 __fox.deformUniforms.uHead.value.set(0.3, 0, 0) 能看到明显的低头。
```

---

## 6. 阶段 4：音乐与跟拍

```text
【阶段 4：WebAudio 电台】
1. 写一个 Radio 类，全部用振荡器和噪声合成，不加载任何音频文件。
   信号链：各声部 → bus → 低通"音色"滤波 → out → 压缩器 → AnalyserNode → 输出。
   效果：程序生成脉冲响应的 ConvolverNode 混响（2.6 秒衰减噪声）；带低通的反馈延迟；
         黑胶底噪（循环播放的噪声 buffer，包含稀疏爆点）；
         一个 0.55 Hz 的 LFO 接到所有旋律振荡器的 detune（磁带抖动效果）。
2. 乐器：kick（正弦波 150→44 Hz 指数下滑，0.42 秒衰减）、snare（带通噪声 + 185 Hz 三角波）、
   clap（三连包络）、rim、hat（高通 7600 Hz 噪声）、shaker、FM 电钢琴（调制器频率等于载波，调制指数逐渐衰减）、
   拨弦、贝斯、锯齿 lead、马林巴、钟声、鸟叫。
3. 三个电台用数据描述，每个包含：bpm、swing、音色滤波截止频率、底噪量、抖动量、音量补偿、
   4 小节和弦进行（每小节一个低音音符 + 4 个和弦音，用 MIDI 编号表示）、旋律音阶，
   以及 16 字符的鼓机谱字符串（X=1、x=0.72、o=0.42、.=不发声；hat 谱里 O 表示开镲）。
   参考：Lo-fi 74 BPM, swing 0.22；City Pop 112 BPM；Bossa 92 BPM。
4. 调度：用 setInterval 每 25 ms 把未来 140 ms 内的 16 分音符排进去；
   swing 只推迟奇数位的 16 分音符。
   旋律由 hash(小节 % 4, 步) 决定，这样 4 小节会重复一次，听起来像写好的乐句。
5. 视觉事件：排音符的同时往 events 队列里推 {t, type: kick|snare|note|bar}。
   渲染循环中，当 ctx.currentTime - outputLatency >= t 时消费；
   落后超过 0.25 秒的事件直接丢掉（标签页切回来时会积压）。
6. 跟拍：beat = (currentTime - latency - anchor) / 每拍秒数；
   点头 = pow(0.5 + 0.5*cos(2π*beat), 2.2) * 幅度；每两拍左右摆一次；尾巴按拍子摇；
   kick 事件让 rig 压扁（y 缩小、x 和 z 放大）；耳机材质（如果已经拆出来）的 emissiveIntensity 跟着 kick 闪。
7. 浏览器要求用户手势才能出声：首屏放一个"打开电台"按钮。
8. 换台：带通噪声扫频的电流声，旧台音量压低，新台从第 0 拍进入。

【验收】
- 三个台听起来风格明显不同，没有破音（分析器上峰值 < 1）。
- 狐狸低头的时刻和底鼓对得上。
```

---

## 7. 阶段 5：表情（最难的一阶段，务必按步骤来）

```text
【阶段 5：表情】
模型没有表情，全部在 shader 里做。闭眼不要用"把眼睛几何挤扁"的方法（试过，会像在坏笑），
按下面的方案做。

A. 眼睛 —— 在片元着色器里"画"眼皮
1. vertex 里计算眼睛局部坐标 e = ((n.x - 眼睛中心x)/眼睛半宽, (n.y - 眼睛中心y)/眼睛半高)，
   传给 varying vec4 vFoxEye = (e.x, e.y, 有效标记, 左右)。
   有效标记 = step(length(e), 1.6)。这一条必须做：跨左右眼分界线的三角形，
   插值会在额头中间造出一只"假眼睛"。
2. uniform vec4 uFace = (左眼闭合, 右眼闭合, 笑眼弧度, 张嘴)，闭合取值 -0.6～1（负数表示瞪大眼）。
3. 在 #include <map_fragment> 之后改 diffuseColor：
   合缝线 L = -0.12 + happy * 0.8 * (1 - e.x²)（happy > 0 拱成 ^，< 0 弯成 ◡）；
   上眼皮 top = mix(1.4, L, close)，下眼皮 bot = mix(-1.4, L, close)；
   覆盖区 = 眼睛椭圆内 且 (e.y > top 或 e.y < bot)，边缘用 smoothstep 柔化；
   覆盖区换成"眼皮毛色"，再乘一点顺毛向的噪声（幅度 ±5%），越靠近合缝线越暗（最暗乘 0.84）；
   在 e.y = top 处画睫毛线：中间粗、两头细，外眼角往上挑一点。
4. 眼皮毛色不要手填：在 JS 里取眼睛外圈 r 在 1.15～1.45 的那一环顶点，读贴图像素，
   排除亮度 < 0.2（眼线）和 > 0.78（白眉毛）的像素，在线性空间里求平均，
   上下眼皮各取一个，作为 uniform 传入。
5. 外眼角漏出来的眼白：覆盖区附近"亮度高且饱和度低"的像素也一并盖掉。
6. 材质：覆盖区 roughnessFactor 混到 0.9，metalnessFactor 混到 0，否则眼皮看起来像塑料。
7. 【关键】这个模型有法线贴图，里面刻着虹膜和高光的轮廓。必须在覆盖区跳过法线贴图，
   否则闭上眼还会照出一圈圈"褶皱"：
   在 #include <normal_fragment_begin> 之后加：vec3 foxGeoNormal = normal;
   在 #include <normal_fragment_maps> 之后加：normal = normalize(mix(normal, foxGeoNormal, 覆盖率));
8. 几何抹平：在 JS 里用眼睛外圈 r 在 1.3～2.1 的那一环（每个网格单元只取 z 最大的点，即最外层表面），
   最小二乘拟合 z = a + b·ex + c·ey + d·ex² + e·ex·ey + f·ey²（6×6 正规方程，带主元的高斯消元）。
   闭眼时（closure > 0.3 开始生效），在 vertex 里把眼窝内的所有层（z 不比曲面低 0.1 以上的都算）
   的 z 混到 "拟合曲面 + 0.005 * max(0, 1 - r²/1.44)"，法线混成曲面的解析法线：
   normalize(-dz/dx, -dz/dy, 1)，注意要把对 e 的导数换算成对世界坐标的导数。

B. 舌头 —— 程序生成网格
1. 沿长度 30 段 × 截面一圈 20 段的管状网格。中心线在 y-z 平面里，
   角度 θ(s) = pitch + curl * s² * (1.25 - 0.25s)，方向 (0, -cosθ, sinθ)，舌面朝向 (0, sinθ, cosθ)。
   θ = 0 表示垂下，θ = π/2 表示朝前。
2. 截面：半宽 0.021 米、半厚 0.0062 米的扁椭圆；舌面（sinφ > 0 的一侧）中间用
   exp(-cos²φ / 0.05) 压出舌沟；舌尖在 s > 0.68 之后按圆弧收口。
3. 顶点色：舌根 #c23a55 → 舌尖 #ff7d93，舌沟和舌底更暗。
   材质：MeshPhysicalMaterial，开 vertexColors，加 clearcoat 和 sheen。
4. 舌头根部放在嘴巴位置往里约 1 厘米处，挂在一个 matrixAutoUpdate = false 的 Group 下，
   这个 Group 的矩阵 = rig.matrixWorld × 平移(枢轴 - rig 原点) × 头部旋转(YXZ) × 平移(-枢轴)。
5. 只在参数变化时重算顶点和 computeVertexNormals()。
6. 张嘴：嘴部区域也传一个 varying，在片元里画一个上沿贴着笑线（嘴角上扬）、下沿为圆弧的深红色开口，
   开口大小跟随 uFace.w。

C. 耳朵：头部区域 y > 0.74、离中线大于 0.09 的部分，绕耳根做 rotZ（外扇）和 rotX（后折）。
   耳机材质用 #define 排除，否则会把头梁掰弯。

D. 表情控制器：每个表情是 {dur, f(u)}，u 从 0 到 1；同一帧多个动作叠加。
   闭眼用 max 合并，头部偏移用加法合并。
   缓动用 bump(u, a, b)：先 smoothstep 进入，最后 smoothstep 退出。
   需要的表情：blink(0.17s)、winkL、winkR、smile、surprise（闭合为负，耳朵竖起）、
   blep（吐舌头：pitch 0.28, curl 0.45, 伸出 0.78, 张嘴 0.55）、
   lick（舔嘴：pitch 0.95, curl 1.7, 左右扫）、
   lickNose（舔鼻子：pitch 1.5, curl 1.5, 伸出 1.3, 头微仰）、
   earL、earR、ears、tail、dizzy。
   自动行为：每 1.8～5.6 秒眨一次眼（25% 概率连眨两下）；闲着时每 6～14 秒随机做一个小动作。

【验收】（请把镜头推到脸部截图核对，不要只看代码）
- 眯眼笑是干净的 ^ ^，眼皮和周围的毛颜色接得上，没有同心圆褶皱，没有漏出的眼白。
- 吐舌头是从张开的嘴里伸出来的扁舌头，有舌沟。
- 舔鼻子时舌尖能碰到鼻子下沿。
```

---

## 8. 阶段 6：部位交互、摸头、睡觉、彩蛋

```text
【阶段 6：交互】
1. 区分点击和拖动：位移 > 6px 或时长 > 600ms 不算点击。
2. 反解部位：射线命中点 → rig.worldToLocal → 加上 rig 原点 → 如果 y 归一化 > 0.45，
   用头部旋转的逆矩阵转回静止姿态 → 归一化 → 按以下顺序判定：
   耳机（face.materialIndex === 1）> 鼻子 > 左眼 > 右眼 > 嘴（这四个要求 z > 0.8，用椭圆判定）
   > 左耳 / 右耳 > 尾巴 > 头 > 身体。
3. 反应：鼻子 → boop + 舔鼻子；嘴 → 吐舌头；眼睛 → 对应那只眼睛眨一下；耳朵 → 对应那只耳朵抖一抖；
   尾巴 → 甩尾巴并回头看；头顶 → 眯眼笑；身体 → boop（身体被压扁回弹、冒爱心、
   从当前电台音阶里挑一个音播放）；耳机 → 换台。
4. 摸头：在 canvas 上加一个 capture 阶段的 pointerdown 监听，按在头或耳朵上就设置 controls.enabled = false，
   松手恢复。拖动时累积"摸头值"（按移动距离增加、随时间衰减）。
   摸头值 > 0 时：闭眼 + 笑眼、头往手的方向歪、尾巴快速摇、播放呼噜声
   （低通噪声，用 22 Hz LFO 做幅度调制），值较高时冒爱心。
   鼠标在头上快速来回划（不按键）也算摸。
5. 连点：2.5 秒内点 8 下 → dizzy。
6. 睡觉：暂停状态下 25 秒没人碰 → 眼睛慢慢闭成 ◡、低头、耳朵耷拉，头顶每 1.3 秒冒一个 Z。
   有人点 → 吓一跳醒来（瞪眼，然后连眨两下）。注意鼠标移动只能重置计时，不能把它吵醒。
7. 彩蛋：8 个（戳鼻子、点嘴巴、戳眼睛、点耳朵、点尾巴、摸摸头、叫醒它、连点 8 下），
   用 localStorage 记录，右上角显示 n/8，点开能看到没发现的条目和提示。
8. 右侧放一列表情按钮（眨眼、眯眼笑、吐舌头、舔舔嘴、舔鼻子、吓一跳、抖耳朵、睡觉/叫醒），
   方便录视频时演示。

【验收】
在控制台把每个部位的归一化坐标投影到屏幕上，派发 PointerEvent，打印识别结果，必须全部正确。
```

---

## 9. 阶段 7：视觉氛围与录屏导出

```text
【阶段 7：氛围 + 录屏】
1. 每个电台有一套 look：主光、轮廓光、环境光、地面颜色、粒子颜色和上升速度、混合方式、耳机发光强度。
   切换时用 1 - exp(-3.2*dt) 做插值；背景用三层 CSS 渐变交叉淡入。
2. 粒子：180 个点，运动全部写在 ShaderMaterial 的顶点着色器里（用 mod 实现循环上升，sin 做漂移）。
3. 音符和爱心：Sprite 对象池；音符贴图用 canvas 画 ♪♫♬，爱心用贝塞尔曲线画；
   音符的出生点用 headPoint(耳罩位置) 计算。
4. 每小节在地面放一个向外扩散的声波环。
5. 录屏：
   - 单独建一块 1080×1350 的 canvas，每帧在 renderer.render() 之后立刻合成：
     背景渐变 → drawImage(WebGL canvas，按 4:5 居中裁切) → 频率数字、台名、BOOPS、曲名、频谱。
   - 开录时先同步调用一次 render() 再合成一帧（X 用第一帧当封面，不能是空的）。
   - 画面：out.captureStream(30)；声音：analyser 再接一路 MediaStreamDestination；合成一个 MediaStream。
   - mimeType 按顺序试：video/mp4;codecs=avc1.640028,mp4a.40.2 → avc1.4d0028 → avc1.42E01E → video/mp4
     → webm 系列。录成 WebM 时要在弹窗里提示"X 不收 WebM"。
   - 时长选项：10 / 30 / 60 秒和自由录制（最长 140 秒）。码率：≤30 秒 10 Mbps，60 秒 8 Mbps，自由录制 6 Mbps。
     定时录制多录 0.4 秒，保证成片不短于所选时长。
   - 录制中按钮显示 已录/总时长，再点一次提前结束；document.hidden 时 pause()，回来再 resume() 并顺延计时。
   - 录完弹出预览（静音自动播放）、下载、再录一次。

【验收】
- 下载的 MP4 能在 QuickTime 里播放，有声音，分辨率 1080x1350，第一帧有狐狸。
```

---

## 10. 配套的"纠错提示词"（模型交付出问题时用）

```text
画面全黑 / 模型不见了：
  检查是否 scene.add(gltf.scene)；检查 onBeforeCompile 注入的 GLSL 是否编译失败
  （控制台搜 "Shader Error"）；检查 mat3 是否按列构造。

眼皮有褶皱或同心圆：
  先在控制台打印 material.normalMap，如果存在，确认已在覆盖区跳过法线贴图；
  再确认二次曲面拟合的系数是有限值，并且 uEyeOK = 1。

额头中间有一条线：
  vFoxEye 的有效标记没做，或者判断条件写反了。

舌头从脸里穿出来：
  pitch 太大，或者根部没有放在嘴的内侧；舔鼻子时应该先朝前伸（pitch ≈ 1.5）再往上卷（curl ≈ 1.5）。

拖动头部时镜头也在转：
  pointerdown 必须用 { capture: true } 注册，并且在 OrbitControls 处理之前把 controls.enabled 设为 false。

录屏首帧是黑的：
  开录时没有先同步 render() 再合成一帧。
```

---
