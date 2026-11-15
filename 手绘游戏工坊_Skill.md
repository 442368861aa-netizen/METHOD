# 手绘游戏工坊 · 3D 画面风格 Skill

> 用途：把统一的视觉语言应用到用户自己的作品。只提供画面风格、渲染、程序化造型、环境美术和界面美术，不规定作品题材、玩法、角色动作、镜头运动或交互流程。
> 单文件知识包：内嵌离线引擎、视觉核心、字体、样式和静态风格展示。所有运行代码均在本文件，不依赖其他项目。

## 0. 使用范围与开场指令

复制给模型：

```text
请以本文件作为画面风格参考，按照我另行指定的作品需求实现。
保留墨线暖光的配色、分档明暗、墨线、纸纹、雾、天空、水、低面数造型与纸贴式界面品质。
直接提取本文件内嵌 engine.js/core.js，不从 CDN 引入另一套渲染器，不用默认 PBR 加轮廓冒充同一画风。
作品的题材、玩法、角色动作、运镜、任务、输入、存档与声音均由本次需求独立决定，本文件不提供默认方案。
先制作静态视觉品质板，确认各物件和界面属于同一作品，再应用到目标作品。
只报告实际执行的视觉检查；没有运行和查看过的效果明确标记未验证。
```

### 0.1 保留的视觉契约

1. 奶白、浅赭、陶土、草绿构成环境基础，钴蓝、金黄、朱红承担少量强调，暗部可偏紫或冷灰。
2. 主剪影先读得出，再加入结构、识别件和少量表面细节。不能用重复盒子堆数量代替美术设计。
3. 近处结构线清楚，远处墨线与细节随雾减弱。排线贴附物体表面，主要存在于暗部。
4. 统一材质与后处理协议，保持地面、植物、道具、天空、水和界面的配色关系。
5. 界面使用暖纸、墨边、硬影、非对称圆角及少量歪斜；字体层次和强调色需要统一。
6. 场景类型与镜头构图由目标作品决定。保留前中远层次、主体剪影和视觉留白，不预设城市、草原或特定视角。

### 0.2 标准库提取器

本文件保留六个标记块，engine/core为封存视觉资产，adapter与壳是静态展示。提取器默认拒绝覆盖已有输出：

```python
from pathlib import Path
import re, sys
text = Path(sys.argv[1]).read_text(encoding='utf-8')
out = Path(sys.argv[2]); out.mkdir(parents=True, exist_ok=True)
parts = {}
for name in ['engine.js','core.js','fonts.css','ui.css','adapter.js','shell.html.txt']:
    pattern = r'^@@BEGIN:' + re.escape(name) + r'@@\n```[^\n]*\n([\s\S]*?)\n```\n@@END:' + re.escape(name) + r'@@$'
    m = re.search(pattern, text, re.M)
    if not m: raise SystemExit('缺少或损坏的代码块: '+name)
    parts[name] = m.group(1)
html = parts['shell.html.txt'].replace('__CSS__', parts['fonts.css']+'\n'+parts['ui.css'])
js = '\n'.join(parts[n] for n in ['engine.js','core.js','adapter.js'])
html = html.replace('__JS__', js.replace('</script', '<\\/script'))
target = out / '墨线暖光_静态风格展示.html'
with target.open('x', encoding='utf-8') as f: f.write(html)
print(target.resolve())
```

## 1. 来源与证据

墨线暖光的主要视觉分析对象为 Cozy Lisbon。原文件署名 Fable 5.2 与 goncalo_canhoto，提供来源 https://cozylisbon.vercel.app 与 https://github.com/fengyiqicoder/awesome-fable-5.2 。长风的可选美术参考来自用户明确授权的本地 LongWind_长风武侠入口及其直接依赖。来源网址作为署名记录，不是运行依赖。

证据分为：已从源码确认、解释性分析、新增替代实现、未验证。原作心理感受与主观美术相似度不因代码共用而自动成立。原作应用代码的独立商用许可证未确认，本文件不授予额外许可。

## 2. 视觉接口与装配

| 接口 | 用途与边界 |
|---|---|
| COZY.seed(seed) | 程序化美术的可重复随机流，各生成域独立持有 |
| COZY.build() | 链式几何配方；box底面对齐、cbox中心对齐；输入geometry所有权转移 |
| COZY.material(opts) | 公共MRT材质，允许顶点/颜色/明暗/雾阶段注入；color.a是语义标记 |
| COZY.Pipeline(renderer) | 阴影、MRT、描边、bloom、FXAA与纸纹；resize时同步所有缓冲 |
| COZY.Mood(pipe) | 同时更新天空、雾、日光、暗部、水、墨色与bloom |
| COZY.sky()/clouds() | 风格化天空与实体云，依赖公共uniform |
| COZY.water(rect,sd) | rect是岸距贴图域，sd正陆负水；默认几何是60000×60000，需要自行裁定实际范围 |

装配顺序：引擎 → 公共uniform/噪声 → 材质/管线 → 色调/天空/水 → 几何与场景 → 固定展示构图 → 尺寸同步 → 渲染。此核心是单世界共享uniform模型，不保证多个独立时段世界互不影响。

内嵌引擎是通用底座，包含部分底层数学/插值工具；保留它们不代表风格包推荐任何控制或玩法系统。目标工程只需要视觉接口。

## 3. 视觉机制详解

### 3.1 视觉核心：共享材质协议 + 双 MRT + 多阶段合成

基础着色模板与工厂：`et` 保存共享 uniform 对象，`wl()` 将对象引用分发给材质，`rl()` 替换 `VPARS / VPRE / VWORLD / FPARS / FALBEDO / FLIT / FFOG` 注入点。`bi()` 构造有光照实体材质；`Vu()` 是阴影深度材质工厂，`_1()` 缓存默认深度材质；`Ao()` 为透明、预乘颜色及 billboard 类效果提供独立模板。提取时值得保留的是这些明确的阶段接口，而非大段无边界的 shader 字符串。

`p1` 顶点着色器同时给出世界位置、世界法线、视空间法线、正的视深度 `vViewZ=-vp.z`、阴影坐标、物体坐标及实例随机种子。`iscale` 来自实例矩阵三列长度；`vObj=pos*iscale` 让标准尺寸网格在不同实例比例下仍有世界尺度的窗户/瓦片纹理。法线通过矩阵列的平方长度补偿非均匀缩放；此实现假设常规 TRS，不能据此宣称支持任意剪切。

m1 主片元模板 输出：

- `outColor = vec4(col,1)`：已经计算光照、排线和雾的颜色。
- `outGeo = vec4(NV*ink,vViewZ)`：RGB 是乘上描边可见度的视空间法线，A 是线性视深度；**A 不是透明度**。
- `vColor.a` 是材质自定义 flag，可能表示瓦屋顶、树干、玻璃、夜间泛光石材或自发光，不能统一解释为 opacity。

T1 自建渲染管线：可选阴影图 → 场景 `gbuf`（两张附件）→ `b1` 描边至 `inkRT` → `M1` 亮部提取 → `S1` 两轮横/竖模糊 → `w1` 最终 FXAA、bloom、饱和度、纸纹、暗角与 fade。不是 Three.js 默认材质外再套一个通用 OutlinePass。

`gbuf` 颜色附件采用 `Fr=1009`（UnsignedByteType），几何附件改为 `Rr=1016`（HalfFloatType）且 `oi=1003` 最近邻；颜色为 `jt=1006` 线性采样。深度/法线缓冲必须最近邻，避免不同物体边界的深度与法线混合。天空深度 `60000` 也接近半浮点最大有限值，应保留此约定并检查移植平台的精度。

`T1.setSize()` 的内部尺寸是 CSS 宽高 × DPR × scale，bloom 每维为主缓冲的 1/4；不是 1/4 像素总量，而是约 1/16。全屏绘制使用一个大三角形 `[-1,-1],[3,-1],[-1,3]`。默认 bloom `.35`、暗角 `.3`、grain `1`、饱和度 `1.06`、纸色 `(.96,.95,.93)`，亮部阈值 `.82`。`M1` 用四角平均后的最大 RGB 分量过 `smoothstep(threshold,threshold+.18)`，并非亮度阈值。`S1` 权重中心 `.227027`，两组正负采样偏移 `1.3846/3.2308`、权重 `.316216/.070270`；两轮步幅分别为 1 和 2。

### 3.2 描边：逆深度二阶差分比简单深度梯度更关键

b1 是最值得单独封存和测试的算法。

1. 纹理噪声先偏移采样中心，振幅 `2.4*uScale` 像素，然后 `floor(...*uRes)+.5` **重新吸附到精确像素中心**。噪声只依赖屏幕坐标/分辨率，没有时间 uniform；注释中的 wobble 不能误报为随时间抖动。
2. 线宽随机系数 `.75–1.3`；十字半径 `round(uScale*w)`、对角半径 `round(uScale*2.3*w*.7071)`，都至少 1 像素且左右严格对称。源码注释明确指出：不对称采样会把 `1/z` 的一阶斜率漏进二阶差分，导致海面地平线误着墨。
3. 折痕取中心与四邻法线 `1-dot` 的最大值，阈值 `.16–.42`。
4. 轮廓取两组 `1/z` 二阶差分绝对值之和，再除中心 `1/z` 归一化；十字阈值 `.012–.03`、对角阈值 `.03–.07`。在透视投影下，平面逆深度为屏幕坐标的仿射函数，对称二阶差分可避免给平坦斜面整体画线。
5. 每组采样都乘有效法线掩码 `step(.5,dot(n,n))`。因此 `ink` 并非始终线性透明：单位法线乘 ink 后，低于约 `sqrt(.5)=.7071` 就无法通过有效性判定。这一点对树与云的远处描边消失尤其重要。
6. 折痕在 `180–1500` 深度退去，强度还从 `.82` 降至 `.4`（`60–700`）；细轮廓在 `300–2600` 从 1 降至 `.5`，再乘 `1-.9*fogA`；粗轮廓在 `140–1100` 消失并乘 `1-fogB`。最小邻域深度用于轮廓衰减，避免只按背景距离淡化前景边缘。
7. 邻域最小深度达到 `40000` 则走天空分支（边强度 `.3`）；纸刷断裂强度 `.72–1`。默认墨色 `(.09,.07,.12)`，与 `col*uInk*1.6` 混合 `.35`，而非所有边都使用纯黑。

迁移建议：先在固定相机下建立“斜平面、海面地平线、相邻箱体、天空边界”四种最小场景，保持几何缓冲协议和像素中心规则，再调整墨色/粗细。不要改成直接深度 Sobel 后仍宣称保留了原算法。对零/无效深度增加约定时，应同时调整天空与透明输出，避免除零；原作依赖场景覆盖及天空哨兵。

### 3.3 明暗分档、排线、雾与阴影的协同

光照及排线：`ndl=dot(N,uSunDir)`，两个过渡 `0–.05` 和 `.42–.48`，`lit=min(l1,sh)`，`tone=lit*mix(.58,1,l2)`。暗面颜色为 `uShadeTint + uSkyAmb*(.5+.5*N.y)`，背光较深处再乘 `.9–1`（`ndl=-.55–-.45`），最后在 shade 与 sunTint 间按 tone 混合。这是窄过渡的分档光照，不是 PBR 粗糙度、金属度或真实多次散射。

`hatch(p,n,scale)` 贴在**世界表面**：水平面（`abs(n.y)>.6`）用 `p.xz`，侧面用沿表面切线的投影和 `p.y`，再转成 `q=(u+v,u-v)`。蓝噪声通道采样尺度 `(.23,.012)`，`smoothstep(.58,.66)*.8`；第二条交叉排线虽计算了绿色噪声，但结果系数为 `0.0`，**当前只有一组有效排线**。默认 `uHatch=.55`，距离因子写作 `smoothstep(520,120,vViewZ)`，排线只在暗面按 `(1-tone)*.34` 乘暗，最大理论暗化约 `.8*.55*.34=.1496`。不要把排线当统一屏幕条纹；树、水和云明确关闭 hatch。

多处源码用倒序 `smoothstep`（例如 520→120）表达淡出；GLSL 规范对 `edge0>=edge1` 的结果未定义。移植应显式重写为 `1-smoothstep(120,520,z)`，保留意图并重新核验，不应把依赖某驱动的行为视为跨平台保证。

公共雾 Eo：`f=1-exp(-(dist*density)^1.3)`；高度由 `smoothstep(0,500,y)` 将雾强度从 1 混至 `.62`，总量最大 `.96`。`atmo(dir)` 比较视线与太阳的**水平朝向**，正向点积取 `sunPow` 次方后，在 `uFogCol/uFogSun` 间混合。因此雾是有方向的配色，不是统一浅蓝遮罩；材质雾用欧氏距离，描边衰减用视深度且没有完整高度项，两者有关联但不完全同一公式。

阴影采用 T1.updateShadow() 的正交相机，默认半宽 520（完整覆盖宽 1040）、分辨率 2048，near 10 / far 4200。日光方向 y 钳到 `.24`，相机沿光向后置 2200；投影中心按世界 texel 尺寸 `2*shadowSize/shadowRes` 对齐，减小跟随镜头时阴影抖动。片元 `shadowAt()` 做四个偏移 `.8*texel` 的比较采样，均值过 `.3–.7` 平滑，再在阴影图边缘 `.82–1` 淡出。偏移包括接收表面沿法线 `.6` 与深度 bias `.0012`；深度材质还启用 polygonOffset `1.5/3`。

迁移建议：保留“分档明暗 + 紫色暗面 + 少量暗面排线 + 定向雾 + 随距离减弱的墨线”的组合。只复制描边，缺少后面四项，不能复现这套层次设计。

### 3.4 时间色调、天空、水和云：统一配色数据驱动

rr、J1 定义 4.8、6、7.2、10、13.2、16.4、18.3、19.5、20.4、21.6 小时十个关键帧。`J1.apply()` 对相邻帧 smoothstep 后共同更新天空/雾/日光/暗面/环境色/云/海深浅色/泡沫/闪光/墨色/bloom；初始 `t=18.15`，即接近 golden hour。`Rt()` 直接把十六进制 RGB 除以 255，`ft.enabled=false` 关闭颜色管理，渲染器 `outputColorSpace=Ns='srgb-linear'`、`toneMapping=0`。迁移至开启颜色管理的工程须统一调色板和输出传递函数，否则相同十六进制数并不保证相同显示结果。

| 时段 | 雾/朝阳雾 | 深水/浅水 | 阴影 tint | 雾密度 | bloom | 墨色 |
| --- | --- | --- | --- | --- | --- | --- |
| 13.2 midday | `#d0eaf8 / #eaf4f8` | `#2686c6 / #5fd2cc` | `.57,.62,.84` | `.00014` | `.22` | `#1b1529` |
| 18.3 golden hour | `#e6cdc0 / #ffd27c` | `#447fa4 / #86c6b2` | `.52,.49,.73` | `.00021` | `.42` | `#1d1426` |
| 19.5 sunset | `#a68ab4 / #ff8c4c` | `#4a5896 / #a488ae` | `.44,.39,.64` | `.00023` | `.55` | `#190f22` |
| night | `#1d2c5e / #26386e` | `#101c4a / #213266` | `.25,.29,.55` | `.00017` | `.7` | `#05071a` |

`J1` 还把亮部阈值随 night 从 `.82` 降到 `.6`；夜间主光方向来自月亮，阴影强度 `.55`。这些变量应作为一个 mood preset 迁移。自动一天变化（`hourSeconds=52`，不同时间段变速）是表现控制，可用静态时段滑块替代，无需引入飞行系统。

天空 q1() 使用半径 9000、32×16 的球面随相机平移，顶点输出 `p.xyww`；关闭深度测试/写入、renderOrder=-1000。按高度平方根构造地平线→mid→zenith 三色，地平线直接复用 `atmo()`，以保证地面雾与天空衔接。太阳是角半径 `.022–.05` 的平面海报圆盘（低空更大），另加 1.7/2.9 倍半径的软环；月亮是两个角圆相减的新月；星星按 hash 稀疏生成。天空 `outGeo=(0,0,1,60000)`，不要随意改成零法线/零深度。

天空有一层噪声裁纸云带，`uCover` 初始 `.71`，集中在高度 `.035–.42`；**已读应用调用中 `J1.cover` 虽计算，未看到赋给天空 `uCover` 的连线**，所以不能声称 rr 的 cover 值已控制此云带。另有 j1()/Y1() 的实体云：每模板 5–8 个 detail=2 多面球合并，下底压平；四种模板实例化，数量实际是 `4*ceil(e/4)`（移动配置 30 产生 32，桌面 56 产生 56）。云无阴影接收/排线，2200–5200 距离额外融雾并抑制墨线。`qs()` 内靠近实体云时平滑增加 `uFade` 至最多 `.82`，`uPaper` 改为云亮面色；这属于可独立复用的相机穿云表现。

水 W1()/X1()：一张 60000×60000 的水平面，没有几何波浪、场景反射渲染或水下折射。`W1()` 把海岸有符号距离 `vi` 编成 2048 宽单通道纹理 `clamp(.5+distance/400)`，shader 解回 `off=(.5-sample)*400`；离岸 0–70 将浅水混成深水。两个低频噪声切换 `.95–1.04` 色块，Fresnel `(1-max(V.y,0))^5` 以 `.72` 强度借用天空/雾色。

水面 `waveMarks()` 是格子内短抛物弧线，cell `34×15` 与 `90×38`，各自寿命/距离淡出；近中远 glint 用 4、13 单位 sparkle 格及远处噪声，反射方向与太阳/月亮水平朝向在 `.93–.9995` 之间集中成反光带。泡沫沿海岸带、噪声和向岸移动的 `sin(off*.36-time*.9+noise*7)` 组成，环带主要在离岸 6–30 内。`fLit` 覆盖通用分档结果，使用阴影控制海水、泡沫、闪光；仍保留公共雾和几何附件输出。源码“with the wind”注释下波纹漂移实际是固定 `vec2(1.6,.5)`，并未读取 `uWind`。

迁移建议：优先移植平面水的配色、短弧和反光带，以非常低的几何成本建立手绘水意象；不要把它描述为完整物理水模拟。

### 3.5 几何构造器：封存 K1 的小型建模语言

K1 / pe() 是地标与模板共用的链式 DSL。`col()` 设置颜色和 flag；`push(x,y,z,yaw,scale,rx,rz)` 用 YXZ 变换压栈，`pop()` 恢复。`add()` 统一去索引、删 UV、应用局部及栈矩阵、写四分量 color；`build()` 用 `Yu(parts,false)` 合并，计算包围球/盒；`mesh(mat)` 创建网格。

值得保留的方法簇：`box` 底面落在 y=0（先平移高的一半），`cbox` 以中心为原点；`cyl/cone` 同样底对齐；`sph` 实际用 `gu` 即 IcosahedronGeometry，默认 detail=2；`dome/torus/lathe` 控制低分段曲面；`gable/hip/frustum` 用明确三角面构屋顶；`ext/slab/plan` 构有洞挤出轮廓；`span/beam/tube` 构跨线梁和曲线管；`ring` 按圆周放置重复构件；`arcade` 把 8 段近似拱洞写入 holes（pointed 时拱高乘 1.5）。

`fs()` 为三角点生成 position 后算 normals，`Mr()` 把四点拆成两三角。`Yu()/uc()` 是通用 BufferGeometry 合并工具（15190），会检查属性/索引/数组类型一致性；不能把它们误报为游戏独创建模算法。`K1.add()` 对非索引输入直接修改原 geometry，且会删 UV；移植时若模板需复用必须 clone，或明确“所有权转移”。`push/pop` 不校验栈平衡、`beam` 不处理零长度，正式 API 应补边界校验。去索引适合硬边建筑，但会增大顶点量，应避免把整座城市所有小构件无差别展开。

地标通用材质 ng()/lg()：flag `>.9` 全自发光；`>.4` 玻璃按 hash 与 night 点亮；`>.2` 夜间泛光；普通石材按物体高度 4–95 做从下向上的暖光。`lg.place()` 设置地形高度、冻结 matrixAutoUpdate、登记投影者。这是可复用的摆放/夜景协议，地标具体坐标、碰撞及彩蛋应分开。

### 3.6 建筑生成：区域规则、模板实例、shader 立面

U1、Gr 与空间查询 把街区作为带角度的超椭圆区域。`za()` 选择首个命中区域；`Vs()` 生成城市混合强度，`sa()` 查询公园椭圆，`Ua()` 排除地标保留区及道路缓冲。区域样式而非任意撒盒子决定城市节奏：例如 pombaline 地块 11–15、深13、庭院6、固定5层、街宽9，屋顶 mansard `.7` / hip `.3`；old 地块6.5–10、深8.5、2–4层、街宽4.5、角度 jitter `.2`；modern 地块26–44、6–12层、街宽26、全平顶、sparse `.45`。`ll=3.1` 为统一层高。

ig() 使用 `Mi(20261109)` 固定种子。传统区域沿街成对排布并细分地块；现代/expo 使用独立疏松地块。每地块取旋转后四角+中心的地形高度，最低低于 2.5 或高差超过 10（无庭院 old 为13）则拒绝；地基起于最低点-.6。屋顶 gable/hip/mansard/flat 取 tg() 的标准单位几何；坡屋顶檐挑 `.38`，烟囱概率 `.42`，平顶设备概率 `.7`。

每 **1000×1000** 空间格分别积累 walls、四种屋顶、props，再创建 InstancedMesh（原别名 `Ri`）；每实例只有矩阵/颜色差异，计算实例包围球，冻结 matrixAutoUpdate。建筑体/屋顶投影，props 明确不投影。返回 `lots` 给树生成器避让，也写入高度占据网格。loading 文案“twenty-eight thousand roofs”不能当成实测建筑数；真实 count 来自运行生成返回值，静态核对没有测量该生成数量。

Q1 立面 shader 用 `vObj/vDims/vSeed` 在箱体侧面绘制窗户、窗框、墙角石、层间线、铁阳台、店铺雨篷。窗列宽度 2.5–3.4，窗宽占 bay 的 `.42–.56`；大部分立面细节在700–1900退去，azulejo tile 单元 `.7` 且仅 seed>.86，70–260退去。夜窗用稳定 hash 点亮；250–1300以后整块 bay 以 `.42` 强度承接灯色，避免远处小窗只剩难以稳定采样的微小像素。`eg` 屋瓦 shader 以 `.85` 瓦槽、`.62` 行尺度生成纹理，160–520退去；平屋顶走边缘 gravel/parapet 线。

迁移建议：保留“街区参数→地块记录→格子分桶→标准网格实例化→近景shader细节”五段接口。换区域数据即可迁移城市形态；地标位置、机场平整区和原地图整体坐标不必搬入目标作品。不要给每扇窗独立几何或 DOM。

### 3.7 树：四种低模、空间避让与低成本风动

rg()/ag() 四模板为 round、pine（扁宽冠形）、cypress（7段 lathe）、palm（9组双面三角叶）。树干 flag=1、白色树冠 flag=0，顶点阶段用 `mix(instanceColor,white,col.a)` 保留树干本色、让树冠着实例颜色；这说明不能全树直接乘绿色。

`ag()` 固定种子8181，1000网格分桶，每格四种 InstancedMesh。公园按面积给密度：forest 1/430、garden 1/300、formal 1/260、meadow 1/900，pines 为0（不代表所有带松树含义区域都自动密植）。城区再按style密度补树，最后尝试 `9800*i` 个野外候选；候选被岸线、城市、公园、道路、噪声与坡度 `D1>.7` 等过滤，不能把9800当最终数量。

避建筑采用40单位空间 hash，每候选查3×3邻格及 lot 圆半径；这降低了全量两两检测成本，但搜索固定邻域对半径特别大的建筑未必充分，移植到更大地块应让邻域覆盖最大建筑半径。道路两侧每13单位放11高行道树，另有沿岸树列。

树材质关闭排线，树冠噪声斑驳只在 `.9 / 1.08` 间跳变。风动在世界顶点坐标加 `uWind*(sin(ph)*.5+sin(ph*2.3+1)*.25)*.02*swayH`，相位为 `time*1.2 + worldX*.031 + worldZ*.027`；随树高增加摇幅，不需要逐树CPU更新矩阵。描边 `ink *= 1-smoothstep(380,1500,dist)`，并受前述法线有效阈值影响。

注意：`ag()` 的材质虽注入 `vWorld` 风动，却没有设置 `customDepth:true`，阴影使用默认 `Vu()`，因此源码中的树冠彩色通道与阴影轮廓不共享风动。可选择为目标项目启用对应自定义深度顶点变形，以获得一致阴影；不能声称原作已实现这一点。

### 3.8 地形与性能：把可绘制细节烘成底图

地形采样：两条海岸平滑轮廓用 `ku()` Chaikin 两轮平滑；`Wu()` 求有符号距离，`$h()` 用900单位空间桶加速，预计算24单位岸距格；`I1()` 进一步生成12单位高度格，`Fe()` 双线性查询；`R1/C1/P1` 给山丘、峭壁/沙滩影响，`N1()` 平整指定圆/线区域。这些空间查询同时供地形、城市、树、水岸使用，值得封存接口，但原始里斯本点表属于地图数据。

H1()/cc()/V1()：地形用常规网格，核心 step22（移动30），外延间距逐级乘1.27，延伸至约-19000/17000；全四角都低于-6的单元不提交三角形。`cc()` 在 canvas 烘地表颜色与街道/广场/球场/农田等细节，核心纹理与外圈纹理交叠，anisotropy8+mipmap。`V1()` 的11000次农田候选等属于**生成阶段**，不能当每帧开销。`H1` 着色器在900→80近处增加细纸纹，地图边缘外500–3600逐渐强制融雾并去墨线；夜街灯池来自 `G1()` 的1024宽城市掩码。

平台配置和画质：

| 参数 | 移动预设 | 桌面预设 |
| --- | --- | --- |
| DPR上限 | 1.6 | 2 |
| 阴影分辨率/半宽 | 1024 / 380 | 2048 / 520 |
| 树密度/几何detail | .5 / 0 | 1 / 1 |
| props / 地图纹理宽 | false / 2048 | true / 4096 |
| 云请求数 / 地形step | 30 / 30 | 56 / 22 |
| draw距离 | 4300 | 7200 |

渲染器关闭 MSAA（antialias=false）、stencil、alpha，指定 high-performance；末阶段FXAA补抗锯齿。light模式 DPR上限1.25、内部scale=.8、关闭bloom；它没有重建更稀疏城市，也没有关闭阴影，不能称为完整几何LOD。运行分辨率自适应属于目标工程策略，不在本风格包中规定。桌面非light且DPR<1.5另有 `1.5/r` 补偿，以保持内部细节。

qs() 对建筑/树的实例分桶包围球做 `cameraDistance-radius<draw` 可见性筛选，再由渲染器完成常规视锥剔除；地形、水、天空及实体云则显式关闭frustumCulled。阴影中心应按目标场景的展示范围设置。`P2()` 用 `Rc()` 在生成阶段间让出UI线程，未使用worker，不能声称城市生成并行化。矩阵冻结与实例化降低CPU/提交开销，但每帧仍可能渲染阴影、MRT及多次全屏pass，实际预算需实测。

### 3.9 CSS 可迁移的是设计语法

CSS tokens：墨 `#17131f`、纸 `#fbfaf5`、钴蓝 `#1f4fbf`、黄 `#f6c22f`、红 `#e0492f`；标题 Caprasimo，正文手写 Shantell Sans Variable（默认500字重）。tile/button通过3px边框、4–6px硬投影、不同横纵半径的非对称圆角形成纸贴感；按下位移3–4px，动画80ms。40×40内联SVG瓷砖包含蓝色花瓣与黄色点，可作为纹样原则，但字体与资产路径需单独处理。

标题与相册：logo旋转-4°，文字伪元素 `.15em` 描边+硬阴影，瓷砖纹理clip进字形；tagline旋转-1.2°；邮卡14:9照片、斜条邮边、略有不同的rotate；640px断点将3px描边改为2.5px。safe-area用CSS变量进入布局。`prefers-reduced-motion` 将主要transition降到10ms，但area动画仍4.2秒，不宜描述为全部动效都关闭。迁移应提取token、字体层次、纸片边框/硬阴影/轻微歪斜，重新设计目标项目自己的状态与操作，不复制HUD、油门、收集和机场流程。

## 4. 道具与环境美术配方

### 4.1 造型语法：不是随机堆基础体

| 配方层 | 约束 | 花车例 | 机器人例 | 渡船例 |
|---|---|---|---|---|
| 主剪影 | 1—3个大体块；先用纯色验证 | 狭长车身+两只大轮 | 头身比例明显，短腿宽脚 | 中部宽、两端收窄的船壳 |
| 结构 | 看得懂用途；大形不被细节淹没 | 把手、轮轴、花槽 | 关节、手掌、背包 | 舷边、甲板、舵/棚 |
| 识别件 | 1—2个最有语义的部位 | 斜放的铲子、一束高花 | 一只圆眼+小天线 | 钴蓝舱棚+奶白边 |
| 表面 | 面片色差/细条纹优先，微几何按视角筛选 | 木板纹+黄油漆 | 釉面色块+少量螺钉 | 吃水线+木甲板条 |
| 次级动态 | 小振幅、不同相位，影响不大于主动作 | 花轻摇、轮转 | 天线滞后、眼睛看方向 | 船体轻浮、旗摆 |

建议比例并非原作硬参数：主色60—75%、辅色20—30%、强调色5—10%；每件道具最多3—5种主要材质；可以超出，但先在最小配方证明需要。镜头中不足1像素的结构不再堆实体，改shader/色块。轮廓特征必须先于纹样成立。

下面是可直接加入 adapter 的真实配方示例（替代设计），用法 `mesh(makeCart(),x,y,z)`。几何尺度适用于当前小场景，需要随玩法统一缩放。

```javascript
function makeCart(){
  const b=COZY.build();
  b.col('#b66d49').box(3.6,.7,5.2,0,1.1,0);
  for(const side of [-1,1]){
    b.col('#edd8b1').box(.18,1.5,5.3,side*1.7,1.7,0);
    b.col('#3e4a63').torus(.85,.23,side*1.95,.9,1.1,{ry:Math.PI/2,rs:6,ts:12});
    b.col('#e5c451').beam([side*1.5,1.4,-2],[side*1.5,2.1,-4],.17);
  }
  for(let i=0;i<5;i++){
    const x=(i%2-.5)*1.5,z=-1.5+i*.7,h=1.5+(i%3)*.4;
    b.col('#5e865d').cyl(.06,.06,h,x,1.8,z,5);
    b.col(i%2?'#f6c22f':'#d87874').sph(.43,x,1.8+h,z,{detail:1});
  }
  return b;
}
```

### 4.2 区域生成而不是全图均匀撒点

生成分为数据层、候选层、验证层、实例层，禁止一个函数里同时创建所有几何、决定碰撞并写存档。建议区域契约：

```javascript
const region={
 id:'orchard', seed:521, bounds:{x0:-80,x1:80,z0:-60,z1:60},
 palette:['#eed8a9','#dfb49c','#f1e6c9'],
 families:[{kind:'tree',weight:.65},{kind:'bench',weight:.2},{kind:'cart',weight:.15}],
 spacing:7, maxAttempts:1600, density:.008,
 exclusions:[], // 明确保留道路、出生点、交互路径与镜头区
 groundAt:(x,z)=>0, isAllowed:(x,z)=>Math.abs(z)>9
};
```

可直接复用的候选采样器（新增通用实现，真实逻辑而非伪代码）：

```javascript
function sampleRegion(c){
 const rand=COZY.seed(c.seed),pts=[],cells=new Map(),cell=c.spacing;
 const key=(x,z)=>Math.floor(x/cell)+','+Math.floor(z/cell);
 const {x0,x1,z0,z1}=c.bounds;
 if(!(cell>0&&x1>x0&&z1>z0))throw Error('invalid region');
 const target=Math.floor((x1-x0)*(z1-z0)*c.density);
 for(let i=0;i<c.maxAttempts&&pts.length<target;i++){
  const x=rand.range(x0,x1),z=rand.range(z0,z1);
  if(!c.isAllowed(x,z))continue;
  const y=c.groundAt(x,z);if(!Number.isFinite(y))continue;
  let valid=true,ix=Math.floor(x/cell),iz=Math.floor(z/cell);
  for(let a=-1;a<=1;a++)for(let b=-1;b<=1;b++){
   for(const q of cells.get((ix+a)+','+(iz+b))||[])
    if(Math.hypot(x-q.x,z-q.z)<cell)valid=false;
  }
  if(!valid)continue;
  let w=rand()*c.families.reduce((s,f)=>s+f.weight,0),kind=c.families.at(-1).kind;
  for(const f of c.families)if((w-=f.weight)<=0){kind=f.kind;break;}
  const p={id:c.id+':'+pts.length,x,y,z,kind,yaw:rand()*Math.PI*2,scale:rand.range(.85,1.15)};
  pts.push(p);const k=key(x,z);if(!cells.has(k))cells.set(k,[]);cells.get(k).push(p);
 }
 return {points:pts,shortfall:target-pts.length};
}
```

此采样器只保证中心点最小间距，不保证大模型尺寸互不相交。若道具半径差异很大，邻域大小须改为 `ceil((candidateRadius+maxRadius)/cell)`，再做形状净空检查。输出shortfall必须保留，不能为了凑数量无限循环。

## 5. 长风参考：只提取画面风格

以下为用户授权本地长风源码的美术分析，不包含动作、战斗、镜头控制或游戏进程。以可选美术方向使用，不强制所有作品成为武侠草原。

### 5.1 暖逆光与冷环境

源码锚点：`environment-h6XP-N4h.js` 的 golden 参数与环境更新，`sky-DGmABWnU.js` 的天空/地平线。

- golden原始日光色量L=[3.3,2.44,1.55]，天空环境色偏冷，地面反射偏土黄，构成暖亮部与冷暗部。
- 冷雾=[.33,.43,.5]，朝光雾=[1.05,.66,.36]；地平线继承雾色，远山融入空气，而非在天际线贴一层统一白色遮罩。
- 解析绘画云强调团块、顺风条纹与边缘亮度。天空、云、地面亮部和雾同一套配色更新。
- 原作有日光/环境增益、shader校色、HDR与后处理；不能把原始L直接作为墨线核心灯强，也不能照搬曝光数值宣称视觉一致。
- 迁移到墨线暖光时，优先调整Mood与共享暗部/雾色。保留墨线核心的输出协议，不混接未经适配的另一套色调映射。

### 5.2 草野的视觉组织

源码锚点：`layout-Dwvbg8vm.js`、`terrain-field-BoBdMPgY.js`、`terrain-BK7wO48U.js`。

- 大起伏决定远处天际线，中尺度缓坡连接区域，少量小尺度起伏承接草石，避免地面频繁碎抖。
- 一个显著主树、碑、亭或岩露头作为空间参照，周围留出清晰空地；植被不是均匀铺满视野。
- 路径由曲线与宽度控制，视觉上贴合地势；道路附近降低草高/密度，使路线可读但不过分像人工描线。
- 前景草与岩、中景地标、远景山和雾形成层次。远山可用更简单几何和更弱描线，不以近景密度延伸到底。
- 可见地形尺度、地表纹理尺度和实际作品场景尺度分别配置，不从某个半径推断整体规模。

### 5.3 界面美术

源码锚点：`style-DOFPLf1u.js`、`screens-CK7ADvq5.js` 的视觉tokens与墨迹构形。

- 暖纸墨色 #e8d3ad、亮纸 #f3e4c4、朱红 #b8322a、近黑墨 #0c0b09、淡金 #f4cf86 可作为独立的古雅配色方向。
- 大字留白、少量竖排标题、衬线中英文字体层次与小面积朱印形成书卷感；不要复制原作文案、字体资产或具体功能布局。
- 细笔触、轻微不规则边缘和受控墨迹遮罩可增加手工感，文字必须保持清晰。静态纹理优先，避免持续扰动影响阅读。
- 与墨线暖光的纸贴式UI二选一形成清晰主题，或仅用一两个点缀；不要把两套粗细、边框、阴影无规则叠加。

## 6. 画面稳定性与实现注意

1. 水面默认世界大平面，rect只定义岸距贴图。需要按实际水域缩放/平移或替换几何，地面与水面不能共面。展示示例显式限制水域并降低水位。
2. 不用重叠道路盒子堆路口；采用一次并集网格、地表着色或贴图。贴花与表面需有明确深度策略。
3. 描边采样依赖缓冲尺寸，频繁自动改变分辨率会造成线条与纸纹清晰度跳变。先固定尺寸验收，再决定画质策略。
4. 最终纸感shader含随时间变化的颗粒。需要稳定纸面时给finalU.uTime独立固定值，保留纸纹而不冻结其他环境表现。
5. 阴影投影中心保留texel对齐；风动若投影，需要对应的深度变形。不要把材质旗标当透明度。
6. GLSL倒序smoothstep应按等效正序淡出审查；动态边界剩余项仍需实际跨平台检查。
7. 静止时保持同一视图、时间和尺寸，连续渲染应一致；连续帧差只能说明时序稳定性，不能代替肉眼美术检查。

## 7. 静态品质板与视觉验收

内嵌展示仅包含地面、小屋、树、蓝色花器、有限水域、天空/远岛、纸卡与时段按钮。固定构图，无角色控制或目标进度。它用于确认风格组件能同场渲染，不代表任何玩法模板。

| 检查 | 通过条件 |
|---|---|
| 离线完整性 | 单文件打开，无外部资源依赖、脚本或shader错误 |
| 配色 | 环境主色克制，强调色有层级，亮暗面统一 |
| 材质 | 所有实体遵循同一MRT/雾/色调协议，无默认材质混入 |
| 墨线 | 斜面无错误满屏线，结构读得清，远景线减弱 |
| 排线与纸纹 | 排线附着世界表面、主要在暗部；纸纹不抢主体 |
| 阴影 | 投影稳定，无明显自阴影条纹或大面积黑块 |
| 水与地面 | 无共面争抢，水岸和雾色相接 |
| 构图 | 主体剪影、前中远层次和留白清楚；各尺寸无文字越界 |
| 时段 | 固定白昼/暖暮/夜色分别检查，不只修改背景 |
| 性能 | 记录真实设备与采样，不从末次全屏pass统计推断完整成本 |

未运行、未查看的项目如实标记。截图生成不等于主观视觉验收；不继承旧版玩法示例的测试记录。本次新展示的验证应单独记录。

## 8. 封存代码与来源边界

engine/core/fonts保持整理前字节与哈希不变；ui/adapter/shell替换为纯静态风格展示。六个块继续支持第0节提取器。中文使用系统后备字体；内嵌Caprasimo不是中文字库。封存视觉核心不包含原城市完整生成器，前面相应内容是分析方法，不应假称现成API。

### engine.js

@@BEGIN:engine.js@@
```javascript
(function(){let e=document.createElement("link").relList;if(e&&e.supports&&e.supports("modulepreload"))return;for(let r of document.querySelectorAll('link[rel="modulepreload"]'))i(r);new MutationObserver(r=>{for(let a of r)if(a.type==="childList")for(let o of a.addedNodes)o.tagName==="LINK"&&o.rel==="modulepreload"&&i(o)}).observe(document,{childList:!0,subtree:!0});function t(r){let a={};return r.integrity&&(a.integrity=r.integrity),r.referrerPolicy&&(a.referrerPolicy=r.referrerPolicy),a.credentials=r.crossOrigin==="use-credentials"?"include":r.crossOrigin==="anonymous"?"omit":"same-origin",a}function i(r){if(r.ep)return;r.ep=!0;let a=t(r);fetch(r.href,a)}})();var Rs=1e3,Zi=1001,Zn=1002,oi=1003,hd=1004,cd=1005,jt=1006,ud=1007,Xa=1008,Fr=1009,dd=1010,fd=1011,Nc=1012,pd=1013,Vr=1014,Co=1015,Rr=1016,Ic=1017,Dc=1018,vl=1020,md=35902,gd=35899,vd=1021,xd=1022,yo=1023,bo=1026,Cs=1027,Ys=1028,Uc=1029,Ps=1030,Oc=1031,zc=1033,_d=33776,yd=33777,bd=33778,Md=33779,Sd=35840,wd=35841,Td=35842,Ed=35843,Ad=36196,Rd=37492,Cd=37496,Pd=37488,Ld=37489,Nd=37490,Id=37491,Dd=37808,Ud=37809,Od=37810,zd=37811,Fd=37812,Bd=37813,kd=37814,Vd=37815,Gd=37816,Hd=37817,Wd=37818,Xd=37819,qd=37820,jd=37821,Yd=36492,Zd=36494,Jd=36495,Kd=36283,$d=36284,Qd=36285,e0=36286,Ls=2300,Jn=2301,rn=2302,Wl=2303,Xl=2400,ql=2401,jl=2402,t0=3200,qi="srgb",Ns="srgb-linear",Is="linear",Ds="srgb",an=7680,i0=35044,Ci=35048,Po="300 es",ia=2e3;function r0(e){for(let t=e.length-1;t>=0;--t)if(e[t]>=65535)return!0;return!1}function a0(e){return ArrayBuffer.isView(e)&&!(e instanceof DataView)}function Us(e){return document.createElementNS("http://www.w3.org/1999/xhtml",e)}function o0(){let e=Us("canvas");return e.style.display="block",e}var Yl={};function Zl(...e){let t="THREE."+e.shift();console.log(t,...e)}function Fc(e){let t=e[0];if(typeof t=="string"&&t.startsWith("TSL:")){let i=e[1];i&&i.isStackTrace?e[0]+=" "+i.getLocation():e[1]='Stack trace not available. Enable "THREE.Node.captureStackTrace" to capture stack traces.'}return e}function Je(...e){e=Fc(e);let t="THREE."+e.shift();{let i=e[0];i&&i.isStackTrace?console.warn(i.getError(t)):console.warn(t,...e)}}function Qe(...e){e=Fc(e);let t="THREE."+e.shift();{let i=e[0];i&&i.isStackTrace?console.error(i.getError(t)):console.error(t,...e)}}function ys(...e){let t=e.join(" ");t in Yl||(Yl[t]=!0,Je(...e))}function s0(e,t,i){return new Promise(function(r,a){function o(){switch(e.clientWaitSync(t,e.SYNC_FLUSH_COMMANDS_BIT,0)){case e.WAIT_FAILED:a();break;case e.TIMEOUT_EXPIRED:setTimeout(o,i);break;default:r()}}setTimeout(o,i)})}var n0={0:0,2:6,4:4,3:5,1:1,6:2,7:7,5:3},na=class{addEventListener(e,t){this._listeners===void 0&&(this._listeners={});let i=this._listeners;i[e]===void 0&&(i[e]=[]),i[e].indexOf(t)===-1&&i[e].push(t)}hasEventListener(e,t){let i=this._listeners;return i!==void 0&&i[e]!==void 0&&i[e].indexOf(t)!==-1}removeEventListener(e,t){let i=this._listeners;if(i===void 0)return;let r=i[e];if(r!==void 0){let a=r.indexOf(t);a!==-1&&r.splice(a,1)}}dispatchEvent(e){let t=this._listeners;if(t===void 0)return;let i=t[e.type];if(i!==void 0){e.target=this;let r=i.slice(0);for(let a=0,o=r.length;a<o;a++)r[a].call(this,e);e.target=null}}},hi="00.01.02.03.04.05.06.07.08.09.0a.0b.0c.0d.0e.0f.10.11.12.13.14.15.16.17.18.19.1a.1b.1c.1d.1e.1f.20.21.22.23.24.25.26.27.28.29.2a.2b.2c.2d.2e.2f.30.31.32.33.34.35.36.37.38.39.3a.3b.3c.3d.3e.3f.40.41.42.43.44.45.46.47.48.49.4a.4b.4c.4d.4e.4f.50.51.52.53.54.55.56.57.58.59.5a.5b.5c.5d.5e.5f.60.61.62.63.64.65.66.67.68.69.6a.6b.6c.6d.6e.6f.70.71.72.73.74.75.76.77.78.79.7a.7b.7c.7d.7e.7f.80.81.82.83.84.85.86.87.88.89.8a.8b.8c.8d.8e.8f.90.91.92.93.94.95.96.97.98.99.9a.9b.9c.9d.9e.9f.a0.a1.a2.a3.a4.a5.a6.a7.a8.a9.aa.ab.ac.ad.ae.af.b0.b1.b2.b3.b4.b5.b6.b7.b8.b9.ba.bb.bc.bd.be.bf.c0.c1.c2.c3.c4.c5.c6.c7.c8.c9.ca.cb.cc.cd.ce.cf.d0.d1.d2.d3.d4.d5.d6.d7.d8.d9.da.db.dc.dd.de.df.e0.e1.e2.e3.e4.e5.e6.e7.e8.e9.ea.eb.ec.ed.ee.ef.f0.f1.f2.f3.f4.f5.f6.f7.f8.f9.fa.fb.fc.fd.fe.ff".split("."),on=Math.PI/180,Kn=180/Math.PI;function qa(){let e=Math.random()*4294967295|0,t=Math.random()*4294967295|0,i=Math.random()*4294967295|0,r=Math.random()*4294967295|0;return(hi[e&255]+hi[e>>8&255]+hi[e>>16&255]+hi[e>>24&255]+"-"+hi[t&255]+hi[t>>8&255]+"-"+hi[t>>16&15|64]+hi[t>>24&255]+"-"+hi[i&63|128]+hi[i>>8&255]+"-"+hi[i>>16&255]+hi[i>>24&255]+hi[r&255]+hi[r>>8&255]+hi[r>>16&255]+hi[r>>24&255]).toLowerCase()}function ct(e,t,i){return Math.max(t,Math.min(i,e))}function l0(e,t){return(e%t+t)%t}function sn(e,t,i){return(1-i)*e+i*t}function h0(e){return 2**Math.floor(Math.log(e)/Math.LN2)}function xl(e){switch(e.constructor){case Uint32Array:return 4294967295;case Uint16Array:return 65535;case Uint8Array:case Uint8ClampedArray:return 255;case Int32Array:return 2147483647;case Int16Array:return 32767;case Int8Array:return 127;default:return}}function c0(e,t){if(t.constructor===Float32Array)return e;let i=xl(t);if(i===void 0)throw Error("THREE.MathUtils: Invalid component type.");return Math.max(e/i,-1)}function u0(e,t){if(t.constructor===Float32Array)return e;let i=xl(t);if(i===void 0)throw Error("THREE.MathUtils: Invalid component type.");return Math.round(e*i)}var ge=class{static{this.prototype.isVector2=!0}constructor(e=0,t=0){this.x=e,this.y=t}get width(){return this.x}set width(e){this.x=e}get height(){return this.y}set height(e){this.y=e}set(e,t){return this.x=e,this.y=t,this}setScalar(e){return this.x=e,this.y=e,this}setX(e){return this.x=e,this}setY(e){return this.y=e,this}setComponent(e,t){switch(e){case 0:this.x=t;break;case 1:this.y=t;break;default:throw Error("THREE.Vector2: index is out of range: "+e)}return this}getComponent(e){switch(e){case 0:return this.x;case 1:return this.y;default:throw Error("THREE.Vector2: index is out of range: "+e)}}clone(){return new this.constructor(this.x,this.y)}copy(e){return this.x=e.x,this.y=e.y,this}add(e){return this.x+=e.x,this.y+=e.y,this}addScalar(e){return this.x+=e,this.y+=e,this}addVectors(e,t){return this.x=e.x+t.x,this.y=e.y+t.y,this}addScaledVector(e,t){return this.x+=e.x*t,this.y+=e.y*t,this}sub(e){return this.x-=e.x,this.y-=e.y,this}subScalar(e){return this.x-=e,this.y-=e,this}subVectors(e,t){return this.x=e.x-t.x,this.y=e.y-t.y,this}multiply(e){return this.x*=e.x,this.y*=e.y,this}multiplyScalar(e){return this.x*=e,this.y*=e,this}divide(e){return this.x/=e.x,this.y/=e.y,this}divideScalar(e){return this.multiplyScalar(1/e)}applyMatrix3(e){let t=this.x,i=this.y,r=e.elements;return this.x=r[0]*t+r[3]*i+r[6],this.y=r[1]*t+r[4]*i+r[7],this}min(e){return this.x=Math.min(this.x,e.x),this.y=Math.min(this.y,e.y),this}max(e){return this.x=Math.max(this.x,e.x),this.y=Math.max(this.y,e.y),this}clamp(e,t){return this.x=ct(this.x,e.x,t.x),this.y=ct(this.y,e.y,t.y),this}clampScalar(e,t){return this.x=ct(this.x,e,t),this.y=ct(this.y,e,t),this}clampLength(e,t){let i=this.length();return this.divideScalar(i||1).multiplyScalar(ct(i,e,t))}floor(){return this.x=Math.floor(this.x),this.y=Math.floor(this.y),this}ceil(){return this.x=Math.ceil(this.x),this.y=Math.ceil(this.y),this}round(){return this.x=Math.round(this.x),this.y=Math.round(this.y),this}roundToZero(){return this.x=Math.trunc(this.x),this.y=Math.trunc(this.y),this}negate(){return this.x=-this.x,this.y=-this.y,this}dot(e){return this.x*e.x+this.y*e.y}cross(e){return this.x*e.y-this.y*e.x}lengthSq(){return this.x*this.x+this.y*this.y}length(){return Math.sqrt(this.x*this.x+this.y*this.y)}manhattanLength(){return Math.abs(this.x)+Math.abs(this.y)}normalize(){return this.divideScalar(this.length()||1)}angle(){return Math.atan2(-this.y,-this.x)+Math.PI}angleTo(e){let t=Math.sqrt(this.lengthSq()*e.lengthSq());if(t===0)return Math.PI/2;let i=this.dot(e)/t;return Math.acos(ct(i,-1,1))}distanceTo(e){return Math.sqrt(this.distanceToSquared(e))}distanceToSquared(e){let t=this.x-e.x,i=this.y-e.y;return t*t+i*i}manhattanDistanceTo(e){return Math.abs(this.x-e.x)+Math.abs(this.y-e.y)}setLength(e){return this.normalize().multiplyScalar(e)}lerp(e,t){return this.x+=(e.x-this.x)*t,this.y+=(e.y-this.y)*t,this}lerpVectors(e,t,i){return this.x=e.x+(t.x-e.x)*i,this.y=e.y+(t.y-e.y)*i,this}equals(e){return e.x===this.x&&e.y===this.y}fromArray(e,t=0){return this.x=e[t],this.y=e[t+1],this}toArray(e=[],t=0){return e[t]=this.x,e[t+1]=this.y,e}fromBufferAttribute(e,t){return this.x=e.getX(t),this.y=e.getY(t),this}rotateAround(e,t){let i=Math.cos(t),r=Math.sin(t),a=this.x-e.x,o=this.y-e.y;return this.x=a*i-o*r+e.x,this.y=a*r+o*i+e.y,this}random(){return this.x=Math.random(),this.y=Math.random(),this}*[Symbol.iterator](){yield this.x,yield this.y}},ei=class{constructor(e=0,t=0,i=0,r=1){this.isQuaternion=!0,this._x=e,this._y=t,this._z=i,this._w=r}static slerpFlat(e,t,i,r,a,o,s){let n=i[r+0],l=i[r+1],h=i[r+2],c=i[r+3],u=a[o+0],d=a[o+1],f=a[o+2],p=a[o+3];if(c!==p||n!==u||l!==d||h!==f){let x=n*u+l*d+h*f+c*p;x<0&&(u=-u,d=-d,f=-f,p=-p,x=-x);let g=1-s;if(x<.9995){let m=Math.acos(x),y=Math.sin(m);g=Math.sin(g*m)/y,s=Math.sin(s*m)/y,n=n*g+u*s,l=l*g+d*s,h=h*g+f*s,c=c*g+p*s}else{n=n*g+u*s,l=l*g+d*s,h=h*g+f*s,c=c*g+p*s;let m=1/Math.sqrt(n*n+l*l+h*h+c*c);n*=m,l*=m,h*=m,c*=m}}e[t]=n,e[t+1]=l,e[t+2]=h,e[t+3]=c}static multiplyQuaternionsFlat(e,t,i,r,a,o){let s=i[r],n=i[r+1],l=i[r+2],h=i[r+3],c=a[o],u=a[o+1],d=a[o+2],f=a[o+3];return e[t]=s*f+h*c+n*d-l*u,e[t+1]=n*f+h*u+l*c-s*d,e[t+2]=l*f+h*d+s*u-n*c,e[t+3]=h*f-s*c-n*u-l*d,e}get x(){return this._x}set x(e){this._x=e,this._onChangeCallback()}get y(){return this._y}set y(e){this._y=e,this._onChangeCallback()}get z(){return this._z}set z(e){this._z=e,this._onChangeCallback()}get w(){return this._w}set w(e){this._w=e,this._onChangeCallback()}set(e,t,i,r){return this._x=e,this._y=t,this._z=i,this._w=r,this._onChangeCallback(),this}clone(){return new this.constructor(this._x,this._y,this._z,this._w)}copy(e){return this._x=e.x,this._y=e.y,this._z=e.z,this._w=e.w,this._onChangeCallback(),this}setFromEuler(e,t=!0){let i=e._x,r=e._y,a=e._z,o=e._order,s=Math.cos,n=Math.sin,l=s(i/2),h=s(r/2),c=s(a/2),u=n(i/2),d=n(r/2),f=n(a/2);switch(o){case"XYZ":this._x=u*h*c+l*d*f,this._y=l*d*c-u*h*f,this._z=l*h*f+u*d*c,this._w=l*h*c-u*d*f;break;case"YXZ":this._x=u*h*c+l*d*f,this._y=l*d*c-u*h*f,this._z=l*h*f-u*d*c,this._w=l*h*c+u*d*f;break;case"ZXY":this._x=u*h*c-l*d*f,this._y=l*d*c+u*h*f,this._z=l*h*f+u*d*c,this._w=l*h*c-u*d*f;break;case"ZYX":this._x=u*h*c-l*d*f,this._y=l*d*c+u*h*f,this._z=l*h*f-u*d*c,this._w=l*h*c+u*d*f;break;case"YZX":this._x=u*h*c+l*d*f,this._y=l*d*c+u*h*f,this._z=l*h*f-u*d*c,this._w=l*h*c-u*d*f;break;case"XZY":this._x=u*h*c-l*d*f,this._y=l*d*c-u*h*f,this._z=l*h*f+u*d*c,this._w=l*h*c+u*d*f;break;default:Je("Quaternion: .setFromEuler() encountered an unknown order: "+o)}return t===!0&&this._onChangeCallback(),this}setFromAxisAngle(e,t){let i=t/2,r=Math.sin(i);return this._x=e.x*r,this._y=e.y*r,this._z=e.z*r,this._w=Math.cos(i),this._onChangeCallback(),this}setFromRotationMatrix(e){let t=e.elements,i=t[0],r=t[4],a=t[8],o=t[1],s=t[5],n=t[9],l=t[2],h=t[6],c=t[10],u=i+s+c;if(u>0){let d=.5/Math.sqrt(u+1);this._w=.25/d,this._x=(h-n)*d,this._y=(a-l)*d,this._z=(o-r)*d}else if(i>s&&i>c){let d=2*Math.sqrt(1+i-s-c);this._w=(h-n)/d,this._x=.25*d,this._y=(r+o)/d,this._z=(a+l)/d}else if(s>c){let d=2*Math.sqrt(1+s-i-c);this._w=(a-l)/d,this._x=(r+o)/d,this._y=.25*d,this._z=(n+h)/d}else{let d=2*Math.sqrt(1+c-i-s);this._w=(o-r)/d,this._x=(a+l)/d,this._y=(n+h)/d,this._z=.25*d}return this._onChangeCallback(),this}setFromUnitVectors(e,t){let i=e.dot(t)+1;return i<1e-8?(i=0,Math.abs(e.x)>Math.abs(e.z)?(this._x=-e.y,this._y=e.x,this._z=0,this._w=i):(this._x=0,this._y=-e.z,this._z=e.y,this._w=i)):(this._x=e.y*t.z-e.z*t.y,this._y=e.z*t.x-e.x*t.z,this._z=e.x*t.y-e.y*t.x,this._w=i),this.normalize()}angleTo(e){return 2*Math.acos(Math.abs(ct(this.dot(e),-1,1)))}rotateTowards(e,t){let i=this.angleTo(e);if(i===0)return this;let r=Math.min(1,t/i);return this.slerp(e,r),this}identity(){return this.set(0,0,0,1)}invert(){return this.conjugate()}conjugate(){return this._x*=-1,this._y*=-1,this._z*=-1,this._onChangeCallback(),this}dot(e){return this._x*e._x+this._y*e._y+this._z*e._z+this._w*e._w}lengthSq(){return this._x*this._x+this._y*this._y+this._z*this._z+this._w*this._w}length(){return Math.sqrt(this._x*this._x+this._y*this._y+this._z*this._z+this._w*this._w)}normalize(){let e=this.length();return e===0?(this._x=0,this._y=0,this._z=0,this._w=1):(e=1/e,this._x*=e,this._y*=e,this._z*=e,this._w*=e),this._onChangeCallback(),this}multiply(e){return this.multiplyQuaternions(this,e)}premultiply(e){return this.multiplyQuaternions(e,this)}multiplyQuaternions(e,t){let i=e._x,r=e._y,a=e._z,o=e._w,s=t._x,n=t._y,l=t._z,h=t._w;return this._x=i*h+o*s+r*l-a*n,this._y=r*h+o*n+a*s-i*l,this._z=a*h+o*l+i*n-r*s,this._w=o*h-i*s-r*n-a*l,this._onChangeCallback(),this}slerp(e,t){let i=e._x,r=e._y,a=e._z,o=e._w,s=this.dot(e);s<0&&(i=-i,r=-r,a=-a,o=-o,s=-s);let n=1-t;if(s<.9995){let l=Math.acos(s),h=Math.sin(l);n=Math.sin(n*l)/h,t=Math.sin(t*l)/h,this._x=this._x*n+i*t,this._y=this._y*n+r*t,this._z=this._z*n+a*t,this._w=this._w*n+o*t,this._onChangeCallback()}else this._x=this._x*n+i*t,this._y=this._y*n+r*t,this._z=this._z*n+a*t,this._w=this._w*n+o*t,this.normalize();return this}slerpQuaternions(e,t,i){return this.copy(e).slerp(t,i)}random(){let e=2*Math.PI*Math.random(),t=2*Math.PI*Math.random(),i=Math.random(),r=Math.sqrt(1-i),a=Math.sqrt(i);return this.set(r*Math.sin(e),r*Math.cos(e),a*Math.sin(t),a*Math.cos(t))}equals(e){return e._x===this._x&&e._y===this._y&&e._z===this._z&&e._w===this._w}fromArray(e,t=0){return this._x=e[t],this._y=e[t+1],this._z=e[t+2],this._w=e[t+3],this._onChangeCallback(),this}toArray(e=[],t=0){return e[t]=this._x,e[t+1]=this._y,e[t+2]=this._z,e[t+3]=this._w,e}fromBufferAttribute(e,t){return this._x=e.getX(t),this._y=e.getY(t),this._z=e.getZ(t),this._w=e.getW(t),this._onChangeCallback(),this}toJSON(){return this.toArray()}_onChange(e){return this._onChangeCallback=e,this}_onChangeCallback(){}*[Symbol.iterator](){yield this._x,yield this._y,yield this._z,yield this._w}},I=class{static{this.prototype.isVector3=!0}constructor(e=0,t=0,i=0){this.x=e,this.y=t,this.z=i}set(e,t,i){return i===void 0&&(i=this.z),this.x=e,this.y=t,this.z=i,this}setScalar(e){return this.x=e,this.y=e,this.z=e,this}setX(e){return this.x=e,this}setY(e){return this.y=e,this}setZ(e){return this.z=e,this}setComponent(e,t){switch(e){case 0:this.x=t;break;case 1:this.y=t;break;case 2:this.z=t;break;default:throw Error("THREE.Vector3: index is out of range: "+e)}return this}getComponent(e){switch(e){case 0:return this.x;case 1:return this.y;case 2:return this.z;default:throw Error("THREE.Vector3: index is out of range: "+e)}}clone(){return new this.constructor(this.x,this.y,this.z)}copy(e){return this.x=e.x,this.y=e.y,this.z=e.z,this}add(e){return this.x+=e.x,this.y+=e.y,this.z+=e.z,this}addScalar(e){return this.x+=e,this.y+=e,this.z+=e,this}addVectors(e,t){return this.x=e.x+t.x,this.y=e.y+t.y,this.z=e.z+t.z,this}addScaledVector(e,t){return this.x+=e.x*t,this.y+=e.y*t,this.z+=e.z*t,this}sub(e){return this.x-=e.x,this.y-=e.y,this.z-=e.z,this}subScalar(e){return this.x-=e,this.y-=e,this.z-=e,this}subVectors(e,t){return this.x=e.x-t.x,this.y=e.y-t.y,this.z=e.z-t.z,this}multiply(e){return this.x*=e.x,this.y*=e.y,this.z*=e.z,this}multiplyScalar(e){return this.x*=e,this.y*=e,this.z*=e,this}multiplyVectors(e,t){return this.x=e.x*t.x,this.y=e.y*t.y,this.z=e.z*t.z,this}applyEuler(e){return this.applyQuaternion(Jl.setFromEuler(e))}applyAxisAngle(e,t){return this.applyQuaternion(Jl.setFromAxisAngle(e,t))}applyMatrix3(e){let t=this.x,i=this.y,r=this.z,a=e.elements;return this.x=a[0]*t+a[3]*i+a[6]*r,this.y=a[1]*t+a[4]*i+a[7]*r,this.z=a[2]*t+a[5]*i+a[8]*r,this}applyNormalMatrix(e){return this.applyMatrix3(e).normalize()}applyMatrix4(e){let t=this.x,i=this.y,r=this.z,a=e.elements,o=1/(a[3]*t+a[7]*i+a[11]*r+a[15]);return this.x=(a[0]*t+a[4]*i+a[8]*r+a[12])*o,this.y=(a[1]*t+a[5]*i+a[9]*r+a[13])*o,this.z=(a[2]*t+a[6]*i+a[10]*r+a[14])*o,this}applyQuaternion(e){let t=this.x,i=this.y,r=this.z,a=e.x,o=e.y,s=e.z,n=e.w,l=2*(o*r-s*i),h=2*(s*t-a*r),c=2*(a*i-o*t);return this.x=t+n*l+o*c-s*h,this.y=i+n*h+s*l-a*c,this.z=r+n*c+a*h-o*l,this}project(e){return this.applyMatrix4(e.matrixWorldInverse).applyMatrix4(e.projectionMatrix)}unproject(e){return this.applyMatrix4(e.projectionMatrixInverse).applyMatrix4(e.matrixWorld)}transformDirection(e){let t=this.x,i=this.y,r=this.z,a=e.elements;return this.x=a[0]*t+a[4]*i+a[8]*r,this.y=a[1]*t+a[5]*i+a[9]*r,this.z=a[2]*t+a[6]*i+a[10]*r,this.normalize()}divide(e){return this.x/=e.x,this.y/=e.y,this.z/=e.z,this}divideScalar(e){return this.multiplyScalar(1/e)}min(e){return this.x=Math.min(this.x,e.x),this.y=Math.min(this.y,e.y),this.z=Math.min(this.z,e.z),this}max(e){return this.x=Math.max(this.x,e.x),this.y=Math.max(this.y,e.y),this.z=Math.max(this.z,e.z),this}clamp(e,t){return this.x=ct(this.x,e.x,t.x),this.y=ct(this.y,e.y,t.y),this.z=ct(this.z,e.z,t.z),this}clampScalar(e,t){return this.x=ct(this.x,e,t),this.y=ct(this.y,e,t),this.z=ct(this.z,e,t),this}clampLength(e,t){let i=this.length();return this.divideScalar(i||1).multiplyScalar(ct(i,e,t))}floor(){return this.x=Math.floor(this.x),this.y=Math.floor(this.y),this.z=Math.floor(this.z),this}ceil(){return this.x=Math.ceil(this.x),this.y=Math.ceil(this.y),this.z=Math.ceil(this.z),this}round(){return this.x=Math.round(this.x),this.y=Math.round(this.y),this.z=Math.round(this.z),this}roundToZero(){return this.x=Math.trunc(this.x),this.y=Math.trunc(this.y),this.z=Math.trunc(this.z),this}negate(){return this.x=-this.x,this.y=-this.y,this.z=-this.z,this}dot(e){return this.x*e.x+this.y*e.y+this.z*e.z}lengthSq(){return this.x*this.x+this.y*this.y+this.z*this.z}length(){return Math.sqrt(this.x*this.x+this.y*this.y+this.z*this.z)}manhattanLength(){return Math.abs(this.x)+Math.abs(this.y)+Math.abs(this.z)}normalize(){return this.divideScalar(this.length()||1)}setLength(e){return this.normalize().multiplyScalar(e)}lerp(e,t){return this.x+=(e.x-this.x)*t,this.y+=(e.y-this.y)*t,this.z+=(e.z-this.z)*t,this}lerpVectors(e,t,i){return this.x=e.x+(t.x-e.x)*i,this.y=e.y+(t.y-e.y)*i,this.z=e.z+(t.z-e.z)*i,this}cross(e){return this.crossVectors(this,e)}crossVectors(e,t){let i=e.x,r=e.y,a=e.z,o=t.x,s=t.y,n=t.z;return this.x=r*n-a*s,this.y=a*o-i*n,this.z=i*s-r*o,this}projectOnVector(e){let t=e.lengthSq();if(t===0)return this.set(0,0,0);let i=e.dot(this)/t;return this.copy(e).multiplyScalar(i)}projectOnPlane(e){return nn.copy(this).projectOnVector(e),this.sub(nn)}reflect(e){return this.sub(nn.copy(e).multiplyScalar(2*this.dot(e)))}angleTo(e){let t=Math.sqrt(this.lengthSq()*e.lengthSq());if(t===0)return Math.PI/2;let i=this.dot(e)/t;return Math.acos(ct(i,-1,1))}distanceTo(e){return Math.sqrt(this.distanceToSquared(e))}distanceToSquared(e){let t=this.x-e.x,i=this.y-e.y,r=this.z-e.z;return t*t+i*i+r*r}manhattanDistanceTo(e){return Math.abs(this.x-e.x)+Math.abs(this.y-e.y)+Math.abs(this.z-e.z)}setFromSpherical(e){return this.setFromSphericalCoords(e.radius,e.phi,e.theta)}setFromSphericalCoords(e,t,i){let r=Math.sin(t)*e;return this.x=r*Math.sin(i),this.y=Math.cos(t)*e,this.z=r*Math.cos(i),this}setFromCylindrical(e){return this.setFromCylindricalCoords(e.radius,e.theta,e.y)}setFromCylindricalCoords(e,t,i){return this.x=e*Math.sin(t),this.y=i,this.z=e*Math.cos(t),this}setFromMatrixPosition(e){let t=e.elements;return this.x=t[12],this.y=t[13],this.z=t[14],this}setFromMatrixScale(e){let t=this.setFromMatrixColumn(e,0).length(),i=this.setFromMatrixColumn(e,1).length(),r=this.setFromMatrixColumn(e,2).length();return this.x=t,this.y=i,this.z=r,this}setFromMatrixColumn(e,t){return this.fromArray(e.elements,t*4)}setFromMatrix3Column(e,t){return this.fromArray(e.elements,t*3)}setFromEuler(e){return this.x=e._x,this.y=e._y,this.z=e._z,this}setFromColor(e){return this.x=e.r,this.y=e.g,this.z=e.b,this}equals(e){return e.x===this.x&&e.y===this.y&&e.z===this.z}fromArray(e,t=0){return this.x=e[t],this.y=e[t+1],this.z=e[t+2],this}toArray(e=[],t=0){return e[t]=this.x,e[t+1]=this.y,e[t+2]=this.z,e}fromBufferAttribute(e,t){return this.x=e.getX(t),this.y=e.getY(t),this.z=e.getZ(t),this}random(){return this.x=Math.random(),this.y=Math.random(),this.z=Math.random(),this}randomDirection(){let e=Math.random()*Math.PI*2,t=Math.random()*2-1,i=Math.sqrt(1-t*t);return this.x=i*Math.cos(e),this.y=t,this.z=i*Math.sin(e),this}*[Symbol.iterator](){yield this.x,yield this.y,yield this.z}},nn=new I,Jl=new ei,it=class{static{this.prototype.isMatrix3=!0}constructor(e,t,i,r,a,o,s,n,l){this.elements=[1,0,0,0,1,0,0,0,1],e!==void 0&&this.set(e,t,i,r,a,o,s,n,l)}set(e,t,i,r,a,o,s,n,l){let h=this.elements;return h[0]=e,h[1]=r,h[2]=s,h[3]=t,h[4]=a,h[5]=n,h[6]=i,h[7]=o,h[8]=l,this}identity(){return this.set(1,0,0,0,1,0,0,0,1),this}copy(e){let t=this.elements,i=e.elements;return t[0]=i[0],t[1]=i[1],t[2]=i[2],t[3]=i[3],t[4]=i[4],t[5]=i[5],t[6]=i[6],t[7]=i[7],t[8]=i[8],this}extractBasis(e,t,i){return e.setFromMatrix3Column(this,0),t.setFromMatrix3Column(this,1),i.setFromMatrix3Column(this,2),this}setFromMatrix4(e){let t=e.elements;return this.set(t[0],t[4],t[8],t[1],t[5],t[9],t[2],t[6],t[10]),this}multiply(e){return this.multiplyMatrices(this,e)}premultiply(e){return this.multiplyMatrices(e,this)}multiplyMatrices(e,t){let i=e.elements,r=t.elements,a=this.elements,o=i[0],s=i[3],n=i[6],l=i[1],h=i[4],c=i[7],u=i[2],d=i[5],f=i[8],p=r[0],x=r[3],g=r[6],m=r[1],y=r[4],M=r[7],b=r[2],E=r[5],T=r[8];return a[0]=o*p+s*m+n*b,a[3]=o*x+s*y+n*E,a[6]=o*g+s*M+n*T,a[1]=l*p+h*m+c*b,a[4]=l*x+h*y+c*E,a[7]=l*g+h*M+c*T,a[2]=u*p+d*m+f*b,a[5]=u*x+d*y+f*E,a[8]=u*g+d*M+f*T,this}multiplyScalar(e){let t=this.elements;return t[0]*=e,t[3]*=e,t[6]*=e,t[1]*=e,t[4]*=e,t[7]*=e,t[2]*=e,t[5]*=e,t[8]*=e,this}determinant(){let e=this.elements,t=e[0],i=e[1],r=e[2],a=e[3],o=e[4],s=e[5],n=e[6],l=e[7],h=e[8];return t*o*h-t*s*l-i*a*h+i*s*n+r*a*l-r*o*n}invert(){let e=this.elements,t=e[0],i=e[1],r=e[2],a=e[3],o=e[4],s=e[5],n=e[6],l=e[7],h=e[8],c=h*o-s*l,u=s*n-h*a,d=l*a-o*n,f=t*c+i*u+r*d;if(f===0)return this.set(0,0,0,0,0,0,0,0,0);let p=1/f;return e[0]=c*p,e[1]=(r*l-h*i)*p,e[2]=(s*i-r*o)*p,e[3]=u*p,e[4]=(h*t-r*n)*p,e[5]=(r*a-s*t)*p,e[6]=d*p,e[7]=(i*n-l*t)*p,e[8]=(o*t-i*a)*p,this}transpose(){let e,t=this.elements;return e=t[1],t[1]=t[3],t[3]=e,e=t[2],t[2]=t[6],t[6]=e,e=t[5],t[5]=t[7],t[7]=e,this}getNormalMatrix(e){return this.setFromMatrix4(e).invert().transpose()}transposeIntoArray(e){let t=this.elements;return e[0]=t[0],e[1]=t[3],e[2]=t[6],e[3]=t[1],e[4]=t[4],e[5]=t[7],e[6]=t[2],e[7]=t[5],e[8]=t[8],this}setUvTransform(e,t,i,r,a,o,s){let n=Math.cos(a),l=Math.sin(a);return this.set(i*n,i*l,-i*(n*o+l*s)+o+e,-r*l,r*n,-r*(-l*o+n*s)+s+t,0,0,1),this}scale(e,t){return ys("Matrix3: .scale() is deprecated. Use .makeScale() instead."),this.premultiply(ln.makeScale(e,t)),this}rotate(e){return ys("Matrix3: .rotate() is deprecated. Use .makeRotation() instead."),this.premultiply(ln.makeRotation(-e)),this}translate(e,t){return ys("Matrix3: .translate() is deprecated. Use .makeTranslation() instead."),this.premultiply(ln.makeTranslation(e,t)),this}makeTranslation(e,t){return e.isVector2?this.set(1,0,e.x,0,1,e.y,0,0,1):this.set(1,0,e,0,1,t,0,0,1),this}makeRotation(e){let t=Math.cos(e),i=Math.sin(e);return this.set(t,-i,0,i,t,0,0,0,1),this}makeScale(e,t){return this.set(e,0,0,0,t,0,0,0,1),this}equals(e){let t=this.elements,i=e.elements;for(let r=0;r<9;r++)if(t[r]!==i[r])return!1;return!0}fromArray(e,t=0){for(let i=0;i<9;i++)this.elements[i]=e[i+t];return this}toArray(e=[],t=0){let i=this.elements;return e[t]=i[0],e[t+1]=i[1],e[t+2]=i[2],e[t+3]=i[3],e[t+4]=i[4],e[t+5]=i[5],e[t+6]=i[6],e[t+7]=i[7],e[t+8]=i[8],e}clone(){return new this.constructor().fromArray(this.elements)}},ln=new it,Kl=new it().set(.4123908,.3575843,.1804808,.212639,.7151687,.0721923,.0193308,.1191948,.9505322),$l=new it().set(3.2409699,-1.5373832,-.4986108,-.9692436,1.8759675,.0415551,.0556301,-.203977,1.0569715);function d0(){let e={enabled:!0,workingColorSpace:Ns,spaces:{},convert:function(a,o,s){return this.enabled===!1||o===s||!o||!s||(this.spaces[o].transfer==="srgb"&&(a.r=Er(a.r),a.g=Er(a.g),a.b=Er(a.b)),this.spaces[o].primaries!==this.spaces[s].primaries&&(a.applyMatrix3(this.spaces[o].toXYZ),a.applyMatrix3(this.spaces[s].fromXYZ)),this.spaces[s].transfer==="srgb"&&(a.r=Oa(a.r),a.g=Oa(a.g),a.b=Oa(a.b))),a},workingToColorSpace:function(a,o){return this.convert(a,this.workingColorSpace,o)},colorSpaceToWorking:function(a,o){return this.convert(a,o,this.workingColorSpace)},getPrimaries:function(a){return this.spaces[a].primaries},getTransfer:function(a){return a===""?Is:this.spaces[a].transfer},getToneMappingMode:function(a){return this.spaces[a].outputColorSpaceConfig.toneMappingMode||"standard"},getLuminanceCoefficients:function(a,o=this.workingColorSpace){return a.fromArray(this.spaces[o].luminanceCoefficients)},define:function(a){Object.assign(this.spaces,a)},_getMatrix:function(a,o,s){return a.copy(this.spaces[s].fromXYZ).multiply(this.spaces[o].toXYZ)},_getDrawingBufferColorSpace:function(a){return this.spaces[a].outputColorSpaceConfig.drawingBufferColorSpace},_getUnpackColorSpace:function(a=this.workingColorSpace){return this.spaces[a].workingColorSpaceConfig.unpackColorSpace}},t=[.64,.33,.3,.6,.15,.06],i=[.2126,.7152,.0722],r=[.3127,.329];return e.define({[Ns]:{primaries:t,whitePoint:r,transfer:Is,toXYZ:Kl,fromXYZ:$l,luminanceCoefficients:i,workingColorSpaceConfig:{unpackColorSpace:qi},outputColorSpaceConfig:{drawingBufferColorSpace:qi}},[qi]:{primaries:t,whitePoint:r,transfer:Ds,toXYZ:Kl,fromXYZ:$l,luminanceCoefficients:i,outputColorSpaceConfig:{drawingBufferColorSpace:qi}}}),e}var ft=d0();function Er(e){return e<.04045?e*.0773993808:(e*.9478672986+.0521327014)**2.4}function Oa(e){return e<.0031308?e*12.92:1.055*e**.41666-.055}var ha,f0=class{static getDataURL(e,t="image/png"){if(/^data:/i.test(e.src)||typeof HTMLCanvasElement>"u")return e.src;let i;if(e instanceof HTMLCanvasElement)i=e;else{ha===void 0&&(ha=Us("canvas")),ha.width=e.width,ha.height=e.height;let r=ha.getContext("2d");e instanceof ImageData?r.putImageData(e,0,0):r.drawImage(e,0,0,e.width,e.height),i=ha}return i.toDataURL(t)}static sRGBToLinear(e){if(typeof HTMLImageElement<"u"&&e instanceof HTMLImageElement||typeof HTMLCanvasElement<"u"&&e instanceof HTMLCanvasElement||typeof ImageBitmap<"u"&&e instanceof ImageBitmap){let t=Us("canvas");t.width=e.width,t.height=e.height;let i=t.getContext("2d");i.drawImage(e,0,0,e.width,e.height);let r=i.getImageData(0,0,e.width,e.height),a=r.data;for(let o=0;o<a.length;o++)a[o]=Er(a[o]/255)*255;return i.putImageData(r,0,0),t}if(e.data){let t=e.data.slice(0);for(let i=0;i<t.length;i++)t instanceof Uint8Array||t instanceof Uint8ClampedArray?t[i]=Math.floor(Er(t[i]/255)*255):t[i]=Er(t[i]);return{data:t,width:e.width,height:e.height}}return Je("ImageUtils.sRGBToLinear(): Unsupported image type. No color space conversion applied."),e}},p0=0,_l=class{constructor(e=null){this.isTextureSource=!0,Object.defineProperty(this,"id",{value:p0++}),this.uuid=qa(),this.data=e,this.dataReady=!0,this.version=0}getSize(e){let t=this.data;return typeof HTMLVideoElement<"u"&&t instanceof HTMLVideoElement?e.set(t.videoWidth,t.videoHeight,0):typeof VideoFrame<"u"&&t instanceof VideoFrame?e.set(t.displayWidth,t.displayHeight,0):t===null?e.set(0,0,0):e.set(t.width||0,t.height||0,t.depth||0),e}set needsUpdate(e){e===!0&&this.version++}toJSON(e){let t=e===void 0||typeof e=="string";if(!t&&e.images[this.uuid]!==void 0)return e.images[this.uuid];let i={uuid:this.uuid,url:""},r=this.data;if(r!==null){let a;if(Array.isArray(r)){a=[];for(let o=0,s=r.length;o<s;o++)r[o].isDataTexture?a.push(hn(r[o].image)):a.push(hn(r[o]))}else a=hn(r);i.url=a}return t||(e.images[this.uuid]=i),i}};function hn(e){return typeof HTMLImageElement<"u"&&e instanceof HTMLImageElement||typeof HTMLCanvasElement<"u"&&e instanceof HTMLCanvasElement||typeof ImageBitmap<"u"&&e instanceof ImageBitmap?f0.getDataURL(e):e.data?{data:Array.from(e.data),width:e.width,height:e.height,type:e.data.constructor.name}:(Je("Texture: Unable to serialize Texture."),{})}var m0=0,cn=new I,zi=class bs extends na{constructor(t=bs.DEFAULT_IMAGE,i=bs.DEFAULT_MAPPING,r=Zi,a=Zi,o=jt,s=Xa,n=yo,l=Fr,h=bs.DEFAULT_ANISOTROPY,c=""){super(),this.isTexture=!0,Object.defineProperty(this,"id",{value:m0++}),this.uuid=qa(),this.name="",this.source=new _l(t),this.mipmaps=[],this.mapping=i,this.channel=0,this.wrapS=r,this.wrapT=a,this.magFilter=o,this.minFilter=s,this.anisotropy=h,this.format=n,this.internalFormat=null,this.type=l,this.offset=new ge(0,0),this.repeat=new ge(1,1),this.center=new ge(0,0),this.rotation=0,this.matrixAutoUpdate=!0,this.matrix=new it,this.generateMipmaps=!0,this.mipmapsAutoUpdate=!0,this.premultiplyAlpha=!1,this.flipY=!0,this.unpackAlignment=4,this.colorSpace=c,this.userData={},this.updateRanges=[],this.version=0,this.onUpdate=null,this.renderTarget=null,this.isRenderTargetTexture=!1,this.isArrayTexture=!!(t&&t.depth&&t.depth>1),this.pmremVersion=0,this.isPMREMTexture=!1,this.normalized=!1}get width(){return this.source.getSize(cn).x}get height(){return this.source.getSize(cn).y}get depth(){return this.source.getSize(cn).z}get image(){return this.source.data}set image(t){this.source.data=t}updateMatrix(){this.matrix.setUvTransform(this.offset.x,this.offset.y,this.repeat.x,this.repeat.y,this.rotation,this.center.x,this.center.y)}addUpdateRange(t,i){this.updateRanges.push({start:t,count:i})}clearUpdateRanges(){this.updateRanges.length=0}clone(){return new this.constructor().copy(this)}copy(t){return this.name=t.name,this.source=t.source,this.mipmaps=t.mipmaps.slice(0),this.mapping=t.mapping,this.channel=t.channel,this.wrapS=t.wrapS,this.wrapT=t.wrapT,this.magFilter=t.magFilter,this.minFilter=t.minFilter,this.anisotropy=t.anisotropy,this.format=t.format,this.internalFormat=t.internalFormat,this.type=t.type,this.normalized=t.normalized,this.offset.copy(t.offset),this.repeat.copy(t.repeat),this.center.copy(t.center),this.rotation=t.rotation,this.matrixAutoUpdate=t.matrixAutoUpdate,this.matrix.copy(t.matrix),this.generateMipmaps=t.generateMipmaps,this.mipmapsAutoUpdate=t.mipmapsAutoUpdate,this.premultiplyAlpha=t.premultiplyAlpha,this.flipY=t.flipY,this.unpackAlignment=t.unpackAlignment,this.colorSpace=t.colorSpace,this.renderTarget=t.renderTarget,this.isRenderTargetTexture=t.isRenderTargetTexture,this.isPMREMTexture=t.isPMREMTexture,this.isArrayTexture=t.isArrayTexture,this.userData=JSON.parse(JSON.stringify(t.userData)),this.needsUpdate=!0,this}setValues(t){for(let i in t){let r=t[i];if(r===void 0){Je(`Texture.setValues(): parameter '${i}' has value of undefined.`);continue}let a=this[i];a===void 0?Je(`Texture.setValues(): property '${i}' does not exist.`):a&&r&&a.isVector2&&r.isVector2||a&&r&&a.isVector3&&r.isVector3||a&&r&&a.isMatrix3&&r.isMatrix3?a.copy(r):this[i]=r}}toJSON(t){let i=t===void 0||typeof t=="string";if(!i&&t.textures[this.uuid]!==void 0)return t.textures[this.uuid];let r={metadata:{version:4.7,type:"Texture",generator:"Texture.toJSON"},uuid:this.uuid,name:this.name,image:this.source.toJSON(t).uuid,mapping:this.mapping,channel:this.channel,repeat:[this.repeat.x,this.repeat.y],offset:[this.offset.x,this.offset.y],center:[this.center.x,this.center.y],rotation:this.rotation,wrap:[this.wrapS,this.wrapT],format:this.format,internalFormat:this.internalFormat,type:this.type,normalized:this.normalized,colorSpace:this.colorSpace,minFilter:this.minFilter,magFilter:this.magFilter,anisotropy:this.anisotropy,flipY:this.flipY,generateMipmaps:this.generateMipmaps,mipmapsAutoUpdate:this.mipmapsAutoUpdate,premultiplyAlpha:this.premultiplyAlpha,unpackAlignment:this.unpackAlignment};return Object.keys(this.userData).length>0&&(r.userData=this.userData),i||(t.textures[this.uuid]=r),r}dispose(){this.dispatchEvent({type:"dispose"})}transformUv(t){if(this.mapping!==300)return t;if(t.applyMatrix3(this.matrix),t.x<0||t.x>1)switch(this.wrapS){case Rs:t.x-=Math.floor(t.x);break;case Zi:t.x=t.x<0?0:1;break;case Zn:Math.abs(Math.floor(t.x)%2)===1?t.x=Math.ceil(t.x)-t.x:t.x-=Math.floor(t.x)}if(t.y<0||t.y>1)switch(this.wrapT){case Rs:t.y-=Math.floor(t.y);break;case Zi:t.y=t.y<0?0:1;break;case Zn:Math.abs(Math.floor(t.y)%2)===1?t.y=Math.ceil(t.y)-t.y:t.y-=Math.floor(t.y)}return this.flipY&&(t.y=1-t.y),t}set needsUpdate(t){t===!0&&(this.version++,this.source.needsUpdate=!0)}set needsPMREMUpdate(t){t===!0&&this.pmremVersion++}};zi.DEFAULT_IMAGE=null,zi.DEFAULT_MAPPING=300,zi.DEFAULT_ANISOTROPY=1;var Ft=class{static{this.prototype.isVector4=!0}constructor(e=0,t=0,i=0,r=1){this.x=e,this.y=t,this.z=i,this.w=r}get width(){return this.z}set width(e){this.z=e}get height(){return this.w}set height(e){this.w=e}set(e,t,i,r){return this.x=e,this.y=t,this.z=i,this.w=r,this}setScalar(e){return this.x=e,this.y=e,this.z=e,this.w=e,this}setX(e){return this.x=e,this}setY(e){return this.y=e,this}setZ(e){return this.z=e,this}setW(e){return this.w=e,this}setComponent(e,t){switch(e){case 0:this.x=t;break;case 1:this.y=t;break;case 2:this.z=t;break;case 3:this.w=t;break;default:throw Error("THREE.Vector4: index is out of range: "+e)}return this}getComponent(e){switch(e){case 0:return this.x;case 1:return this.y;case 2:return this.z;case 3:return this.w;default:throw Error("THREE.Vector4: index is out of range: "+e)}}clone(){return new this.constructor(this.x,this.y,this.z,this.w)}copy(e){return this.x=e.x,this.y=e.y,this.z=e.z,this.w=e.w===void 0?1:e.w,this}add(e){return this.x+=e.x,this.y+=e.y,this.z+=e.z,this.w+=e.w,this}addScalar(e){return this.x+=e,this.y+=e,this.z+=e,this.w+=e,this}addVectors(e,t){return this.x=e.x+t.x,this.y=e.y+t.y,this.z=e.z+t.z,this.w=e.w+t.w,this}addScaledVector(e,t){return this.x+=e.x*t,this.y+=e.y*t,this.z+=e.z*t,this.w+=e.w*t,this}sub(e){return this.x-=e.x,this.y-=e.y,this.z-=e.z,this.w-=e.w,this}subScalar(e){return this.x-=e,this.y-=e,this.z-=e,this.w-=e,this}subVectors(e,t){return this.x=e.x-t.x,this.y=e.y-t.y,this.z=e.z-t.z,this.w=e.w-t.w,this}multiply(e){return this.x*=e.x,this.y*=e.y,this.z*=e.z,this.w*=e.w,this}multiplyScalar(e){return this.x*=e,this.y*=e,this.z*=e,this.w*=e,this}applyMatrix4(e){let t=this.x,i=this.y,r=this.z,a=this.w,o=e.elements;return this.x=o[0]*t+o[4]*i+o[8]*r+o[12]*a,this.y=o[1]*t+o[5]*i+o[9]*r+o[13]*a,this.z=o[2]*t+o[6]*i+o[10]*r+o[14]*a,this.w=o[3]*t+o[7]*i+o[11]*r+o[15]*a,this}divide(e){return this.x/=e.x,this.y/=e.y,this.z/=e.z,this.w/=e.w,this}divideScalar(e){return this.multiplyScalar(1/e)}setAxisAngleFromQuaternion(e){this.w=2*Math.acos(e.w);let t=Math.sqrt(1-e.w*e.w);return t<1e-4?(this.x=1,this.y=0,this.z=0):(this.x=e.x/t,this.y=e.y/t,this.z=e.z/t),this}setAxisAngleFromRotationMatrix(e){let t,i,r,a,o=.01,s=.1,n=e.elements,l=n[0],h=n[4],c=n[8],u=n[1],d=n[5],f=n[9],p=n[2],x=n[6],g=n[10];if(Math.abs(h-u)<o&&Math.abs(c-p)<o&&Math.abs(f-x)<o){if(Math.abs(h+u)<s&&Math.abs(c+p)<s&&Math.abs(f+x)<s&&Math.abs(l+d+g-3)<s)return this.set(1,0,0,0),this;t=Math.PI;let y=(l+1)/2,M=(d+1)/2,b=(g+1)/2,E=(h+u)/4,T=(c+p)/4,v=(f+x)/4;return y>M&&y>b?y<o?(i=0,r=.707106781,a=.707106781):(i=Math.sqrt(y),r=E/i,a=T/i):M>b?M<o?(i=.707106781,r=0,a=.707106781):(r=Math.sqrt(M),i=E/r,a=v/r):b<o?(i=.707106781,r=.707106781,a=0):(a=Math.sqrt(b),i=T/a,r=v/a),this.set(i,r,a,t),this}let m=Math.sqrt((x-f)*(x-f)+(c-p)*(c-p)+(u-h)*(u-h));return Math.abs(m)<.001&&(m=1),this.x=(x-f)/m,this.y=(c-p)/m,this.z=(u-h)/m,this.w=Math.acos((l+d+g-1)/2),this}setFromMatrixPosition(e){let t=e.elements;return this.x=t[12],this.y=t[13],this.z=t[14],this.w=t[15],this}min(e){return this.x=Math.min(this.x,e.x),this.y=Math.min(this.y,e.y),this.z=Math.min(this.z,e.z),this.w=Math.min(this.w,e.w),this}max(e){return this.x=Math.max(this.x,e.x),this.y=Math.max(this.y,e.y),this.z=Math.max(this.z,e.z),this.w=Math.max(this.w,e.w),this}clamp(e,t){return this.x=ct(this.x,e.x,t.x),this.y=ct(this.y,e.y,t.y),this.z=ct(this.z,e.z,t.z),this.w=ct(this.w,e.w,t.w),this}clampScalar(e,t){return this.x=ct(this.x,e,t),this.y=ct(this.y,e,t),this.z=ct(this.z,e,t),this.w=ct(this.w,e,t),this}clampLength(e,t){let i=this.length();return this.divideScalar(i||1).multiplyScalar(ct(i,e,t))}floor(){return this.x=Math.floor(this.x),this.y=Math.floor(this.y),this.z=Math.floor(this.z),this.w=Math.floor(this.w),this}ceil(){return this.x=Math.ceil(this.x),this.y=Math.ceil(this.y),this.z=Math.ceil(this.z),this.w=Math.ceil(this.w),this}round(){return this.x=Math.round(this.x),this.y=Math.round(this.y),this.z=Math.round(this.z),this.w=Math.round(this.w),this}roundToZero(){return this.x=Math.trunc(this.x),this.y=Math.trunc(this.y),this.z=Math.trunc(this.z),this.w=Math.trunc(this.w),this}negate(){return this.x=-this.x,this.y=-this.y,this.z=-this.z,this.w=-this.w,this}dot(e){return this.x*e.x+this.y*e.y+this.z*e.z+this.w*e.w}lengthSq(){return this.x*this.x+this.y*this.y+this.z*this.z+this.w*this.w}length(){return Math.sqrt(this.x*this.x+this.y*this.y+this.z*this.z+this.w*this.w)}manhattanLength(){return Math.abs(this.x)+Math.abs(this.y)+Math.abs(this.z)+Math.abs(this.w)}normalize(){return this.divideScalar(this.length()||1)}setLength(e){return this.normalize().multiplyScalar(e)}lerp(e,t){return this.x+=(e.x-this.x)*t,this.y+=(e.y-this.y)*t,this.z+=(e.z-this.z)*t,this.w+=(e.w-this.w)*t,this}lerpVectors(e,t,i){return this.x=e.x+(t.x-e.x)*i,this.y=e.y+(t.y-e.y)*i,this.z=e.z+(t.z-e.z)*i,this.w=e.w+(t.w-e.w)*i,this}equals(e){return e.x===this.x&&e.y===this.y&&e.z===this.z&&e.w===this.w}fromArray(e,t=0){return this.x=e[t],this.y=e[t+1],this.z=e[t+2],this.w=e[t+3],this}toArray(e=[],t=0){return e[t]=this.x,e[t+1]=this.y,e[t+2]=this.z,e[t+3]=this.w,e}fromBufferAttribute(e,t){return this.x=e.getX(t),this.y=e.getY(t),this.z=e.getZ(t),this.w=e.getW(t),this}random(){return this.x=Math.random(),this.y=Math.random(),this.z=Math.random(),this.w=Math.random(),this}*[Symbol.iterator](){yield this.x,yield this.y,yield this.z,yield this.w}},g0=class extends na{constructor(e=1,t=1,i={}){super(),i=Object.assign({generateMipmaps:!1,internalFormat:null,minFilter:jt,depthBuffer:!0,stencilBuffer:!1,resolveColorBuffer:!0,resolveDepthBuffer:!0,resolveStencilBuffer:!0,storeMultisampledColorBuffer:!0,storeMultisampledDepthBuffer:!0,storeMultisampledStencilBuffer:!0,depthTexture:null,samples:0,count:1,depth:1,multiview:!1,useArrayDepthTexture:!1},i),this.isRenderTarget=!0,this.width=e,this.height=t,this.depth=i.depth,this.scissor=new Ft(0,0,e,t),this.scissorTest=!1,this.viewport=new Ft(0,0,e,t),this.textures=[];let r=i.count;for(let a=0;a<r;a++)this.textures[a]=new zi({width:e,height:t,depth:i.depth}),this.textures[a].isRenderTargetTexture=!0,this.textures[a].renderTarget=this;this._setTextureOptions(i),this.depthBuffer=i.depthBuffer,this.stencilBuffer=i.stencilBuffer,this.resolveColorBuffer=i.resolveColorBuffer,this.resolveDepthBuffer=i.resolveDepthBuffer,this.resolveStencilBuffer=i.resolveStencilBuffer,this.storeMultisampledColorBuffer=i.storeMultisampledColorBuffer,this.storeMultisampledDepthBuffer=i.storeMultisampledDepthBuffer,this.storeMultisampledStencilBuffer=i.storeMultisampledStencilBuffer,this._depthTexture=null,this.depthTexture=i.depthTexture,this.samples=i.samples,this.multiview=i.multiview,this.useArrayDepthTexture=i.useArrayDepthTexture}_setTextureOptions(e={}){let t={minFilter:jt,generateMipmaps:!1,flipY:!1,internalFormat:null};e.mapping!==void 0&&(t.mapping=e.mapping),e.wrapS!==void 0&&(t.wrapS=e.wrapS),e.wrapT!==void 0&&(t.wrapT=e.wrapT),e.wrapR!==void 0&&(t.wrapR=e.wrapR),e.magFilter!==void 0&&(t.magFilter=e.magFilter),e.minFilter!==void 0&&(t.minFilter=e.minFilter),e.format!==void 0&&(t.format=e.format),e.type!==void 0&&(t.type=e.type),e.anisotropy!==void 0&&(t.anisotropy=e.anisotropy),e.colorSpace!==void 0&&(t.colorSpace=e.colorSpace),e.flipY!==void 0&&(t.flipY=e.flipY),e.generateMipmaps!==void 0&&(t.generateMipmaps=e.generateMipmaps),e.mipmapsAutoUpdate!==void 0&&(t.mipmapsAutoUpdate=e.mipmapsAutoUpdate),e.internalFormat!==void 0&&(t.internalFormat=e.internalFormat);for(let i=0;i<this.textures.length;i++)this.textures[i].setValues(t)}get texture(){return this.textures[0]}set texture(e){this.textures[0]=e}set depthTexture(e){this._depthTexture!==null&&this._depthTexture.renderTarget===this&&(this._depthTexture.renderTarget=null),e!==null&&e.renderTarget===null&&(e.renderTarget=this),this._depthTexture=e}get depthTexture(){return this._depthTexture}setSize(e,t,i=1){if(this.width!==e||this.height!==t||this.depth!==i){this.width=e,this.height=t,this.depth=i;for(let r=0,a=this.textures.length;r<a;r++)this.textures[r].image.width=e,this.textures[r].image.height=t,this.textures[r].image.depth=i,this.textures[r].isData3DTexture!==!0&&(this.textures[r].isArrayTexture=this.textures[r].image.depth>1);this.dispose()}this.viewport.set(0,0,e,t),this.scissor.set(0,0,e,t)}clone(){return new this.constructor().copy(this)}copy(e){this.width=e.width,this.height=e.height,this.depth=e.depth,this.scissor.copy(e.scissor),this.scissorTest=e.scissorTest,this.viewport.copy(e.viewport),this.textures.length=0;for(let t=0,i=e.textures.length;t<i;t++){this.textures[t]=e.textures[t].clone(),this.textures[t].isRenderTargetTexture=!0,this.textures[t].renderTarget=this;let r=Object.assign({},e.textures[t].image);this.textures[t].source=new _l(r)}if(this.depthBuffer=e.depthBuffer,this.stencilBuffer=e.stencilBuffer,this.resolveColorBuffer=e.resolveColorBuffer,this.resolveDepthBuffer=e.resolveDepthBuffer,this.resolveStencilBuffer=e.resolveStencilBuffer,this.storeMultisampledColorBuffer=e.storeMultisampledColorBuffer,this.storeMultisampledDepthBuffer=e.storeMultisampledDepthBuffer,this.storeMultisampledStencilBuffer=e.storeMultisampledStencilBuffer,e.depthTexture!==null)if(e.depthTexture.renderTarget===e){let t=e.depthTexture.clone();t.renderTarget=null,this.depthTexture=t}else this.depthTexture=e.depthTexture;else this.depthTexture=null;return this.samples=e.samples,this.multiview=e.multiview,this.useArrayDepthTexture=e.useArrayDepthTexture,this}dispose(){this.dispatchEvent({type:"dispose"})}},gi=class extends g0{constructor(e=1,t=1,i={}){super(e,t,i),this.isWebGLRenderTarget=!0}},Bc=class extends zi{constructor(e=null,t=1,i=1,r=1){super(null),this.isDataArrayTexture=!0,this.image={data:e,width:t,height:i,depth:r},this.magFilter=oi,this.minFilter=oi,this.wrapR=Zi,this.generateMipmaps=!1,this.flipY=!1,this.unpackAlignment=1,this.layerUpdates=new Set}copy(e){return super.copy(e),this.wrapR=e.wrapR,this}addLayerUpdate(e){this.layerUpdates.add(e)}clearLayerUpdates(){this.layerUpdates.clear()}},v0=class extends zi{constructor(e=null,t=1,i=1,r=1){super(null),this.isData3DTexture=!0,this.image={data:e,width:t,height:i,depth:r},this.magFilter=oi,this.minFilter=oi,this.wrapR=Zi,this.generateMipmaps=!1,this.flipY=!1,this.unpackAlignment=1}copy(e){return super.copy(e),this.wrapR=e.wrapR,this}},gt=class kc{static{this.prototype.isMatrix4=!0}constructor(t,i,r,a,o,s,n,l,h,c,u,d,f,p,x,g){this.elements=[1,0,0,0,0,1,0,0,0,0,1,0,0,0,0,1],t!==void 0&&this.set(t,i,r,a,o,s,n,l,h,c,u,d,f,p,x,g)}set(t,i,r,a,o,s,n,l,h,c,u,d,f,p,x,g){let m=this.elements;return m[0]=t,m[4]=i,m[8]=r,m[12]=a,m[1]=o,m[5]=s,m[9]=n,m[13]=l,m[2]=h,m[6]=c,m[10]=u,m[14]=d,m[3]=f,m[7]=p,m[11]=x,m[15]=g,this}identity(){return this.set(1,0,0,0,0,1,0,0,0,0,1,0,0,0,0,1),this}clone(){return new kc().fromArray(this.elements)}copy(t){let i=this.elements,r=t.elements;return i[0]=r[0],i[1]=r[1],i[2]=r[2],i[3]=r[3],i[4]=r[4],i[5]=r[5],i[6]=r[6],i[7]=r[7],i[8]=r[8],i[9]=r[9],i[10]=r[10],i[11]=r[11],i[12]=r[12],i[13]=r[13],i[14]=r[14],i[15]=r[15],this}copyPosition(t){let i=this.elements,r=t.elements;return i[12]=r[12],i[13]=r[13],i[14]=r[14],this}setFromMatrix3(t){let i=t.elements;return this.set(i[0],i[3],i[6],0,i[1],i[4],i[7],0,i[2],i[5],i[8],0,0,0,0,1),this}extractBasis(t,i,r){return this.determinantAffine()===0?(t.set(1,0,0),i.set(0,1,0),r.set(0,0,1),this):(t.setFromMatrixColumn(this,0),i.setFromMatrixColumn(this,1),r.setFromMatrixColumn(this,2),this)}makeBasis(t,i,r){return this.set(t.x,i.x,r.x,0,t.y,i.y,r.y,0,t.z,i.z,r.z,0,0,0,0,1),this}extractRotation(t){if(t.determinantAffine()===0)return this.identity();let i=this.elements,r=t.elements,a=1/ca.setFromMatrixColumn(t,0).length(),o=1/ca.setFromMatrixColumn(t,1).length(),s=1/ca.setFromMatrixColumn(t,2).length();return i[0]=r[0]*a,i[1]=r[1]*a,i[2]=r[2]*a,i[3]=0,i[4]=r[4]*o,i[5]=r[5]*o,i[6]=r[6]*o,i[7]=0,i[8]=r[8]*s,i[9]=r[9]*s,i[10]=r[10]*s,i[11]=0,i[12]=0,i[13]=0,i[14]=0,i[15]=1,this}makeRotationFromEuler(t){let i=this.elements,r=t.x,a=t.y,o=t.z,s=Math.cos(r),n=Math.sin(r),l=Math.cos(a),h=Math.sin(a),c=Math.cos(o),u=Math.sin(o);if(t.order==="XYZ"){let d=s*c,f=s*u,p=n*c,x=n*u;i[0]=l*c,i[4]=-l*u,i[8]=h,i[1]=f+p*h,i[5]=d-x*h,i[9]=-n*l,i[2]=x-d*h,i[6]=p+f*h,i[10]=s*l}else if(t.order==="YXZ"){let d=l*c,f=l*u,p=h*c,x=h*u;i[0]=d+x*n,i[4]=p*n-f,i[8]=s*h,i[1]=s*u,i[5]=s*c,i[9]=-n,i[2]=f*n-p,i[6]=x+d*n,i[10]=s*l}else if(t.order==="ZXY"){let d=l*c,f=l*u,p=h*c,x=h*u;i[0]=d-x*n,i[4]=-s*u,i[8]=p+f*n,i[1]=f+p*n,i[5]=s*c,i[9]=x-d*n,i[2]=-s*h,i[6]=n,i[10]=s*l}else if(t.order==="ZYX"){let d=s*c,f=s*u,p=n*c,x=n*u;i[0]=l*c,i[4]=p*h-f,i[8]=d*h+x,i[1]=l*u,i[5]=x*h+d,i[9]=f*h-p,i[2]=-h,i[6]=n*l,i[10]=s*l}else if(t.order==="YZX"){let d=s*l,f=s*h,p=n*l,x=n*h;i[0]=l*c,i[4]=x-d*u,i[8]=p*u+f,i[1]=u,i[5]=s*c,i[9]=-n*c,i[2]=-h*c,i[6]=f*u+p,i[10]=d-x*u}else if(t.order==="XZY"){let d=s*l,f=s*h,p=n*l,x=n*h;i[0]=l*c,i[4]=-u,i[8]=h*c,i[1]=d*u+x,i[5]=s*c,i[9]=f*u-p,i[2]=p*u-f,i[6]=n*c,i[10]=x*u+d}return i[3]=0,i[7]=0,i[11]=0,i[12]=0,i[13]=0,i[14]=0,i[15]=1,this}makeRotationFromQuaternion(t){return this.compose(x0,t,_0)}lookAt(t,i,r){let a=this.elements;return wi.subVectors(t,i),wi.lengthSq()===0&&(wi.z=1),wi.normalize(),Lr.crossVectors(r,wi),Lr.lengthSq()===0&&(Math.abs(r.z)===1?wi.x+=1e-4:wi.z+=1e-4,wi.normalize(),Lr.crossVectors(r,wi)),Lr.normalize(),zo.crossVectors(wi,Lr),a[0]=Lr.x,a[4]=zo.x,a[8]=wi.x,a[1]=Lr.y,a[5]=zo.y,a[9]=wi.y,a[2]=Lr.z,a[6]=zo.z,a[10]=wi.z,this}multiply(t){return this.multiplyMatrices(this,t)}premultiply(t){return this.multiplyMatrices(t,this)}multiplyMatrices(t,i){let r=t.elements,a=i.elements,o=this.elements,s=r[0],n=r[4],l=r[8],h=r[12],c=r[1],u=r[5],d=r[9],f=r[13],p=r[2],x=r[6],g=r[10],m=r[14],y=r[3],M=r[7],b=r[11],E=r[15],T=a[0],v=a[4],_=a[8],A=a[12],C=a[1],R=a[5],L=a[9],G=a[13],P=a[2],z=a[6],H=a[10],D=a[14],ae=a[3],j=a[7],Y=a[11],ie=a[15];return o[0]=s*T+n*C+l*P+h*ae,o[4]=s*v+n*R+l*z+h*j,o[8]=s*_+n*L+l*H+h*Y,o[12]=s*A+n*G+l*D+h*ie,o[1]=c*T+u*C+d*P+f*ae,o[5]=c*v+u*R+d*z+f*j,o[9]=c*_+u*L+d*H+f*Y,o[13]=c*A+u*G+d*D+f*ie,o[2]=p*T+x*C+g*P+m*ae,o[6]=p*v+x*R+g*z+m*j,o[10]=p*_+x*L+g*H+m*Y,o[14]=p*A+x*G+g*D+m*ie,o[3]=y*T+M*C+b*P+E*ae,o[7]=y*v+M*R+b*z+E*j,o[11]=y*_+M*L+b*H+E*Y,o[15]=y*A+M*G+b*D+E*ie,this}multiplyScalar(t){let i=this.elements;return i[0]*=t,i[4]*=t,i[8]*=t,i[12]*=t,i[1]*=t,i[5]*=t,i[9]*=t,i[13]*=t,i[2]*=t,i[6]*=t,i[10]*=t,i[14]*=t,i[3]*=t,i[7]*=t,i[11]*=t,i[15]*=t,this}determinant(){let t=this.elements,i=t[0],r=t[4],a=t[8],o=t[12],s=t[1],n=t[5],l=t[9],h=t[13],c=t[2],u=t[6],d=t[10],f=t[14],p=t[3],x=t[7],g=t[11],m=t[15],y=l*f-h*d,M=n*f-h*u,b=n*d-l*u,E=s*f-h*c,T=s*d-l*c,v=s*u-n*c;return i*(x*y-g*M+m*b)-r*(p*y-g*E+m*T)+a*(p*M-x*E+m*v)-o*(p*b-x*T+g*v)}determinantAffine(){let t=this.elements,i=t[0],r=t[4],a=t[8],o=t[1],s=t[5],n=t[9],l=t[2],h=t[6],c=t[10];return i*(s*c-n*h)-r*(o*c-n*l)+a*(o*h-s*l)}transpose(){let t=this.elements,i;return i=t[1],t[1]=t[4],t[4]=i,i=t[2],t[2]=t[8],t[8]=i,i=t[6],t[6]=t[9],t[9]=i,i=t[3],t[3]=t[12],t[12]=i,i=t[7],t[7]=t[13],t[13]=i,i=t[11],t[11]=t[14],t[14]=i,this}setPosition(t,i,r){let a=this.elements;return t.isVector3?(a[12]=t.x,a[13]=t.y,a[14]=t.z):(a[12]=t,a[13]=i,a[14]=r),this}invert(){let t=this.elements,i=t[0],r=t[1],a=t[2],o=t[3],s=t[4],n=t[5],l=t[6],h=t[7],c=t[8],u=t[9],d=t[10],f=t[11],p=t[12],x=t[13],g=t[14],m=t[15],y=i*n-r*s,M=i*l-a*s,b=i*h-o*s,E=r*l-a*n,T=r*h-o*n,v=a*h-o*l,_=c*x-u*p,A=c*g-d*p,C=c*m-f*p,R=u*g-d*x,L=u*m-f*x,G=d*m-f*g,P=y*G-M*L+b*R+E*C-T*A+v*_;if(P===0)return this.set(0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0);let z=1/P;return t[0]=(n*G-l*L+h*R)*z,t[1]=(a*L-r*G-o*R)*z,t[2]=(x*v-g*T+m*E)*z,t[3]=(d*T-u*v-f*E)*z,t[4]=(l*C-s*G-h*A)*z,t[5]=(i*G-a*C+o*A)*z,t[6]=(g*b-p*v-m*M)*z,t[7]=(c*v-d*b+f*M)*z,t[8]=(s*L-n*C+h*_)*z,t[9]=(r*C-i*L-o*_)*z,t[10]=(p*T-x*b+m*y)*z,t[11]=(u*b-c*T-f*y)*z,t[12]=(n*A-s*R-l*_)*z,t[13]=(i*R-r*A+a*_)*z,t[14]=(x*M-p*E-g*y)*z,t[15]=(c*E-u*M+d*y)*z,this}scale(t){let i=this.elements,r=t.x,a=t.y,o=t.z;return i[0]*=r,i[4]*=a,i[8]*=o,i[1]*=r,i[5]*=a,i[9]*=o,i[2]*=r,i[6]*=a,i[10]*=o,i[3]*=r,i[7]*=a,i[11]*=o,this}getMaxScaleOnAxis(){let t=this.elements,i=t[0]*t[0]+t[1]*t[1]+t[2]*t[2],r=t[4]*t[4]+t[5]*t[5]+t[6]*t[6],a=t[8]*t[8]+t[9]*t[9]+t[10]*t[10];return Math.sqrt(Math.max(i,r,a))}makeTranslation(t,i,r){return t.isVector3?this.set(1,0,0,t.x,0,1,0,t.y,0,0,1,t.z,0,0,0,1):this.set(1,0,0,t,0,1,0,i,0,0,1,r,0,0,0,1),this}makeRotationX(t){let i=Math.cos(t),r=Math.sin(t);return this.set(1,0,0,0,0,i,-r,0,0,r,i,0,0,0,0,1),this}makeRotationY(t){let i=Math.cos(t),r=Math.sin(t);return this.set(i,0,r,0,0,1,0,0,-r,0,i,0,0,0,0,1),this}makeRotationZ(t){let i=Math.cos(t),r=Math.sin(t);return this.set(i,-r,0,0,r,i,0,0,0,0,1,0,0,0,0,1),this}makeRotationAxis(t,i){let r=Math.cos(i),a=Math.sin(i),o=1-r,s=t.x,n=t.y,l=t.z,h=o*s,c=o*n;return this.set(h*s+r,h*n-a*l,h*l+a*n,0,h*n+a*l,c*n+r,c*l-a*s,0,h*l-a*n,c*l+a*s,o*l*l+r,0,0,0,0,1),this}makeScale(t,i,r){return this.set(t,0,0,0,0,i,0,0,0,0,r,0,0,0,0,1),this}makeShear(t,i,r,a,o,s){return this.set(1,r,o,0,t,1,s,0,i,a,1,0,0,0,0,1),this}compose(t,i,r){let a=this.elements,o=i._x,s=i._y,n=i._z,l=i._w,h=o+o,c=s+s,u=n+n,d=o*h,f=o*c,p=o*u,x=s*c,g=s*u,m=n*u,y=l*h,M=l*c,b=l*u,E=r.x,T=r.y,v=r.z;return a[0]=(1-(x+m))*E,a[1]=(f+b)*E,a[2]=(p-M)*E,a[3]=0,a[4]=(f-b)*T,a[5]=(1-(d+m))*T,a[6]=(g+y)*T,a[7]=0,a[8]=(p+M)*v,a[9]=(g-y)*v,a[10]=(1-(d+x))*v,a[11]=0,a[12]=t.x,a[13]=t.y,a[14]=t.z,a[15]=1,this}decompose(t,i,r){let a=this.elements;t.x=a[12],t.y=a[13],t.z=a[14];let o=this.determinantAffine();if(o===0)return r.set(1,1,1),i.identity(),this;let s=ca.set(a[0],a[1],a[2]).length(),n=ca.set(a[4],a[5],a[6]).length(),l=ca.set(a[8],a[9],a[10]).length();o<0&&(s=-s),Bi.copy(this);let h=1/s,c=1/n,u=1/l;return Bi.elements[0]*=h,Bi.elements[1]*=h,Bi.elements[2]*=h,Bi.elements[4]*=c,Bi.elements[5]*=c,Bi.elements[6]*=c,Bi.elements[8]*=u,Bi.elements[9]*=u,Bi.elements[10]*=u,i.setFromRotationMatrix(Bi),r.x=s,r.y=n,r.z=l,this}makePerspective(t,i,r,a,o,s,n=ia,l=!1){let h=this.elements,c=2*o/(i-t),u=2*o/(r-a),d=(i+t)/(i-t),f=(r+a)/(r-a),p,x;if(l)p=o/(s-o),x=s*o/(s-o);else if(n===2e3)p=-(s+o)/(s-o),x=-2*s*o/(s-o);else if(n===2001)p=-s/(s-o),x=-s*o/(s-o);else throw Error("THREE.Matrix4.makePerspective(): Invalid coordinate system: "+n);return h[0]=c,h[4]=0,h[8]=d,h[12]=0,h[1]=0,h[5]=u,h[9]=f,h[13]=0,h[2]=0,h[6]=0,h[10]=p,h[14]=x,h[3]=0,h[7]=0,h[11]=-1,h[15]=0,this}makeOrthographic(t,i,r,a,o,s,n=ia,l=!1){let h=this.elements,c=2/(i-t),u=2/(r-a),d=-(i+t)/(i-t),f=-(r+a)/(r-a),p,x;if(l)p=1/(s-o),x=s/(s-o);else if(n===2e3)p=-2/(s-o),x=-(s+o)/(s-o);else if(n===2001)p=-1/(s-o),x=-o/(s-o);else throw Error("THREE.Matrix4.makeOrthographic(): Invalid coordinate system: "+n);return h[0]=c,h[4]=0,h[8]=0,h[12]=d,h[1]=0,h[5]=u,h[9]=0,h[13]=f,h[2]=0,h[6]=0,h[10]=p,h[14]=x,h[3]=0,h[7]=0,h[11]=0,h[15]=1,this}equals(t){let i=this.elements,r=t.elements;for(let a=0;a<16;a++)if(i[a]!==r[a])return!1;return!0}fromArray(t,i=0){for(let r=0;r<16;r++)this.elements[r]=t[r+i];return this}toArray(t=[],i=0){let r=this.elements;return t[i]=r[0],t[i+1]=r[1],t[i+2]=r[2],t[i+3]=r[3],t[i+4]=r[4],t[i+5]=r[5],t[i+6]=r[6],t[i+7]=r[7],t[i+8]=r[8],t[i+9]=r[9],t[i+10]=r[10],t[i+11]=r[11],t[i+12]=r[12],t[i+13]=r[13],t[i+14]=r[14],t[i+15]=r[15],t}},ca=new I,Bi=new gt,x0=new I(0,0,0),_0=new I(1,1,1),Lr=new I,zo=new I,wi=new I,Ql=new gt,eh=new ei,Cr=class Vc{constructor(t=0,i=0,r=0,a=Vc.DEFAULT_ORDER){this.isEuler=!0,this._x=t,this._y=i,this._z=r,this._order=a}get x(){return this._x}set x(t){this._x=t,this._onChangeCallback()}get y(){return this._y}set y(t){this._y=t,this._onChangeCallback()}get z(){return this._z}set z(t){this._z=t,this._onChangeCallback()}get order(){return this._order}set order(t){this._order=t,this._onChangeCallback()}set(t,i,r,a=this._order){return this._x=t,this._y=i,this._z=r,this._order=a,this._onChangeCallback(),this}clone(){return new this.constructor(this._x,this._y,this._z,this._order)}copy(t){return this._x=t._x,this._y=t._y,this._z=t._z,this._order=t._order,this._onChangeCallback(),this}setFromRotationMatrix(t,i=this._order,r=!0){let a=t.elements,o=a[0],s=a[4],n=a[8],l=a[1],h=a[5],c=a[9],u=a[2],d=a[6],f=a[10];switch(i){case"XYZ":this._y=Math.asin(ct(n,-1,1)),Math.abs(n)<.9999999?(this._x=Math.atan2(-c,f),this._z=Math.atan2(-s,o)):(this._x=Math.atan2(d,h),this._z=0);break;case"YXZ":this._x=Math.asin(-ct(c,-1,1)),Math.abs(c)<.9999999?(this._y=Math.atan2(n,f),this._z=Math.atan2(l,h)):(this._y=Math.atan2(-u,o),this._z=0);break;case"ZXY":this._x=Math.asin(ct(d,-1,1)),Math.abs(d)<.9999999?(this._y=Math.atan2(-u,f),this._z=Math.atan2(-s,h)):(this._y=0,this._z=Math.atan2(l,o));break;case"ZYX":this._y=Math.asin(-ct(u,-1,1)),Math.abs(u)<.9999999?(this._x=Math.atan2(d,f),this._z=Math.atan2(l,o)):(this._x=0,this._z=Math.atan2(-s,h));break;case"YZX":this._z=Math.asin(ct(l,-1,1)),Math.abs(l)<.9999999?(this._x=Math.atan2(-c,h),this._y=Math.atan2(-u,o)):(this._x=0,this._y=Math.atan2(n,f));break;case"XZY":this._z=Math.asin(-ct(s,-1,1)),Math.abs(s)<.9999999?(this._x=Math.atan2(d,h),this._y=Math.atan2(n,o)):(this._x=Math.atan2(-c,f),this._y=0);break;default:Je("Euler: .setFromRotationMatrix() encountered an unknown order: "+i)}return this._order=i,r===!0&&this._onChangeCallback(),this}setFromQuaternion(t,i,r){return Ql.makeRotationFromQuaternion(t),this.setFromRotationMatrix(Ql,i,r)}setFromVector3(t,i=this._order){return this.set(t.x,t.y,t.z,i)}reorder(t){return eh.setFromEuler(this),this.setFromQuaternion(eh,t)}equals(t){return t._x===this._x&&t._y===this._y&&t._z===this._z&&t._order===this._order}fromArray(t){return this._x=t[0],this._y=t[1],this._z=t[2],t[3]!==void 0&&(this._order=t[3]),this._onChangeCallback(),this}toArray(t=[],i=0){return t[i]=this._x,t[i+1]=this._y,t[i+2]=this._z,t[i+3]=this._order,t}_onChange(t){return this._onChangeCallback=t,this}_onChangeCallback(){}*[Symbol.iterator](){yield this._x,yield this._y,yield this._z,yield this._order}};Cr.DEFAULT_ORDER="XYZ";var Gc=class{constructor(){this.mask=1}set(e){this.mask=1<<e>>>0}enable(e){this.mask|=1<<e}enableAll(){this.mask=-1}toggle(e){this.mask^=1<<e}disable(e){this.mask&=~(1<<e)}disableAll(){this.mask=0}test(e){return(this.mask&e.mask)!==0}isEnabled(e){return!!(this.mask&1<<e)}},y0=0,th=new I,ua=new ei,vr=new gt,Fo=new I,Za=new I,b0=new I,M0=new ei,ih=new I(1,0,0),rh=new I(0,1,0),ah=new I(0,0,1),oh={type:"added"},S0={type:"removed"},da={type:"childadded",child:null},un={type:"childremoved",child:null},Ar=class Ms extends na{constructor(){super(),this.isObject3D=!0,Object.defineProperty(this,"id",{value:y0++}),this.uuid=qa(),this.name="",this.type="Object3D",this.parent=null,this.children=[],this.up=Ms.DEFAULT_UP.clone();let t=new I,i=new Cr,r=new ei,a=new I(1,1,1);function o(){r.setFromEuler(i,!1)}function s(){i.setFromQuaternion(r,void 0,!1)}i._onChange(o),r._onChange(s),Object.defineProperties(this,{position:{configurable:!0,enumerable:!0,value:t},rotation:{configurable:!0,enumerable:!0,value:i},quaternion:{configurable:!0,enumerable:!0,value:r},scale:{configurable:!0,enumerable:!0,value:a},modelViewMatrix:{value:new gt},normalMatrix:{value:new it}}),this.matrix=new gt,this.matrixWorld=new gt,this.matrixAutoUpdate=Ms.DEFAULT_MATRIX_AUTO_UPDATE,this.matrixWorldAutoUpdate=Ms.DEFAULT_MATRIX_WORLD_AUTO_UPDATE,this.matrixWorldNeedsUpdate=!1,this.layers=new Gc,this.visible=!0,this.castShadow=!1,this.receiveShadow=!1,this.frustumCulled=!0,this.renderOrder=0,this.animations=[],this.customDepthMaterial=void 0,this.customDistanceMaterial=void 0,this.static=!1,this.userData={},this.pivot=null}onBeforeShadow(){}onAfterShadow(){}onBeforeRender(){}onAfterRender(){}applyMatrix4(t){this.matrixAutoUpdate&&this.updateMatrix(),this.matrix.premultiply(t),this.matrix.decompose(this.position,this.quaternion,this.scale),this.pivot!==null&&this.position.copy(this.pivot).applyMatrix4(this.matrix).sub(this.pivot)}applyQuaternion(t){return this.quaternion.premultiply(t),this}setRotationFromAxisAngle(t,i){this.quaternion.setFromAxisAngle(t,i)}setRotationFromEuler(t){this.quaternion.setFromEuler(t,!0)}setRotationFromMatrix(t){this.quaternion.setFromRotationMatrix(t)}setRotationFromQuaternion(t){this.quaternion.copy(t)}rotateOnAxis(t,i){return ua.setFromAxisAngle(t,i),this.quaternion.multiply(ua),this}rotateOnWorldAxis(t,i){return ua.setFromAxisAngle(t,i),this.quaternion.premultiply(ua),this}rotateX(t){return this.rotateOnAxis(ih,t)}rotateY(t){return this.rotateOnAxis(rh,t)}rotateZ(t){return this.rotateOnAxis(ah,t)}translateOnAxis(t,i){return th.copy(t).applyQuaternion(this.quaternion),this.position.add(th.multiplyScalar(i)),this}translateX(t){return this.translateOnAxis(ih,t)}translateY(t){return this.translateOnAxis(rh,t)}translateZ(t){return this.translateOnAxis(ah,t)}localToWorld(t){return this.updateWorldMatrix(!0,!1),t.applyMatrix4(this.matrixWorld)}worldToLocal(t){return this.updateWorldMatrix(!0,!1),t.applyMatrix4(vr.copy(this.matrixWorld).invert())}lookAt(t,i,r){t.isVector3?Fo.copy(t):Fo.set(t,i,r);let a=this.parent;this.updateWorldMatrix(!0,!1),Za.setFromMatrixPosition(this.matrixWorld),this.isCamera||this.isLight?vr.lookAt(Za,Fo,this.up):vr.lookAt(Fo,Za,this.up),this.quaternion.setFromRotationMatrix(vr),a&&(vr.extractRotation(a.matrixWorld),ua.setFromRotationMatrix(vr),this.quaternion.premultiply(ua.invert()))}add(t){if(arguments.length>1){for(let i=0;i<arguments.length;i++)this.add(arguments[i]);return this}return t===this?(Qe("Object3D.add: object can't be added as a child of itself.",t),this):(t&&t.isObject3D?(t.removeFromParent(),t.parent=this,this.children.push(t),t.dispatchEvent(oh),da.child=t,this.dispatchEvent(da),da.child=null):Qe("Object3D.add: object not an instance of THREE.Object3D.",t),this)}remove(t){if(arguments.length>1){for(let r=0;r<arguments.length;r++)this.remove(arguments[r]);return this}let i=this.children.indexOf(t);return i!==-1&&(t.parent=null,this.children.splice(i,1),t.dispatchEvent(S0),un.child=t,this.dispatchEvent(un),un.child=null),this}removeFromParent(){let t=this.parent;return t!==null&&t.remove(this),this}clear(){return this.remove(...this.children)}attach(t){return this.updateWorldMatrix(!0,!1),vr.copy(this.matrixWorld).invert(),t.parent!==null&&(t.parent.updateWorldMatrix(!0,!1),vr.multiply(t.parent.matrixWorld)),t.applyMatrix4(vr),t.removeFromParent(),t.parent=this,this.children.push(t),t.updateWorldMatrix(!1,!0),t.dispatchEvent(oh),da.child=t,this.dispatchEvent(da),da.child=null,this}getObjectById(t){return this.getObjectByProperty("id",t)}getObjectByName(t){return this.getObjectByProperty("name",t)}getObjectByProperty(t,i){if(this[t]===i)return this;for(let r=0,a=this.children.length;r<a;r++){let o=this.children[r].getObjectByProperty(t,i);if(o!==void 0)return o}}getObjectsByProperty(t,i,r=[]){this[t]===i&&r.push(this);let a=this.children;for(let o=0,s=a.length;o<s;o++)a[o].getObjectsByProperty(t,i,r);return r}getWorldPosition(t){return this.updateWorldMatrix(!0,!1),t.setFromMatrixPosition(this.matrixWorld)}getWorldQuaternion(t){return this.updateWorldMatrix(!0,!1),this.matrixWorld.decompose(Za,t,b0),t}getWorldScale(t){return this.updateWorldMatrix(!0,!1),this.matrixWorld.decompose(Za,M0,t),t}getWorldDirection(t){this.updateWorldMatrix(!0,!1);let i=this.matrixWorld.elements;return t.set(i[8],i[9],i[10]).normalize()}raycast(){}intersectsFrustum(){return!0}traverse(t){t(this);let i=this.children;for(let r=0,a=i.length;r<a;r++)i[r].traverse(t)}traverseVisible(t){if(this.visible===!1)return;t(this);let i=this.children;for(let r=0,a=i.length;r<a;r++)i[r].traverseVisible(t)}traverseAncestors(t){let i=this.parent;i!==null&&(t(i),i.traverseAncestors(t))}updateMatrix(){this.matrix.compose(this.position,this.quaternion,this.scale);let t=this.pivot;if(t!==null){let i=t.x,r=t.y,a=t.z,o=this.matrix.elements;o[12]+=i-o[0]*i-o[4]*r-o[8]*a,o[13]+=r-o[1]*i-o[5]*r-o[9]*a,o[14]+=a-o[2]*i-o[6]*r-o[10]*a}this.matrixWorldNeedsUpdate=!0}updateMatrixWorld(t){this.matrixAutoUpdate&&this.updateMatrix(),(this.matrixWorldNeedsUpdate||t)&&(this.matrixWorldAutoUpdate===!0&&(this.parent===null?this.matrixWorld.copy(this.matrix):this.matrixWorld.multiplyMatrices(this.parent.matrixWorld,this.matrix)),this.matrixWorldNeedsUpdate=!1,t=!0);let i=this.children;for(let r=0,a=i.length;r<a;r++)i[r].updateMatrixWorld(t)}updateWorldMatrix(t,i,r=!1){let a=this.parent;if(t===!0&&a!==null&&a.updateWorldMatrix(!0,!1),this.matrixAutoUpdate&&this.updateMatrix(),(this.matrixWorldNeedsUpdate||r)&&(this.matrixWorldAutoUpdate===!0&&(this.parent===null?this.matrixWorld.copy(this.matrix):this.matrixWorld.multiplyMatrices(this.parent.matrixWorld,this.matrix)),this.matrixWorldNeedsUpdate=!1,r=!0),i===!0){let o=this.children;for(let s=0,n=o.length;s<n;s++)o[s].updateWorldMatrix(!1,!0,r)}}toJSON(t){let i=t===void 0||typeof t=="string",r={};i&&(t={geometries:{},materials:{},textures:{},images:{},shapes:{},skeletons:{},animations:{},nodes:{}},r.metadata={version:4.7,type:"Object",generator:"Object3D.toJSON"});let a={};a.uuid=this.uuid,a.type=this.type,a.name=this.name,a.castShadow=this.castShadow,a.receiveShadow=this.receiveShadow,a.visible=this.visible,a.frustumCulled=this.frustumCulled,a.renderOrder=this.renderOrder,a.static=this.static,a.matrixAutoUpdate=this.matrixAutoUpdate,Object.keys(this.userData).length>0&&(a.userData=this.userData),a.layers=this.layers.mask,a.matrix=this.matrix.toArray(),a.up=this.up.toArray(),this.pivot!==null&&(a.pivot=this.pivot.toArray()),this.morphTargetDictionary!==void 0&&(a.morphTargetDictionary=Object.assign({},this.morphTargetDictionary)),this.morphTargetInfluences!==void 0&&(a.morphTargetInfluences=this.morphTargetInfluences.slice()),this.isInstancedMesh&&(a.type="InstancedMesh",a.count=this.count,a.instanceMatrix=this.instanceMatrix.toJSON(),this.instanceColor!==null&&(a.instanceColor=this.instanceColor.toJSON())),this.isBatchedMesh&&(a.type="BatchedMesh",a.perObjectFrustumCulled=this.perObjectFrustumCulled,a.sortObjects=this.sortObjects,a.drawRanges=this._drawRanges,a.reservedRanges=this._reservedRanges,a.geometryInfo=this._geometryInfo.map(n=>({...n,boundingBox:n.boundingBox?n.boundingBox.toJSON():void 0,boundingSphere:n.boundingSphere?n.boundingSphere.toJSON():void 0})),a.instanceInfo=this._instanceInfo.map(n=>({...n})),a.availableInstanceIds=this._availableInstanceIds.slice(),a.availableGeometryIds=this._availableGeometryIds.slice(),a.nextIndexStart=this._nextIndexStart,a.nextVertexStart=this._nextVertexStart,a.geometryCount=this._geometryCount,a.maxInstanceCount=this._maxInstanceCount,a.maxVertexCount=this._maxVertexCount,a.maxIndexCount=this._maxIndexCount,a.geometryInitialized=this._geometryInitialized,a.matricesTexture=this._matricesTexture.toJSON(t),a.indirectTexture=this._indirectTexture.toJSON(t),this._colorsTexture!==null&&(a.colorsTexture=this._colorsTexture.toJSON(t)),this.boundingSphere!==null&&(a.boundingSphere=this.boundingSphere.toJSON()),this.boundingBox!==null&&(a.boundingBox=this.boundingBox.toJSON()));function o(n,l){return n[l.uuid]===void 0&&(n[l.uuid]=l.toJSON(t)),l.uuid}if(this.isScene)this.background&&(this.background.isColor?a.background=this.background.toJSON():this.background.isTexture&&(a.background=this.background.toJSON(t).uuid)),this.environment&&this.environment.isTexture&&this.environment.isRenderTargetTexture!==!0&&(a.environment=this.environment.toJSON(t).uuid);else if(this.isMesh||this.isLine||this.isPoints){a.geometry=o(t.geometries,this.geometry);let n=this.geometry.parameters;if(n!==void 0&&n.shapes!==void 0){let l=n.shapes;if(Array.isArray(l))for(let h=0,c=l.length;h<c;h++){let u=l[h];o(t.shapes,u)}else o(t.shapes,l)}}if(this.isSkinnedMesh&&(a.bindMode=this.bindMode,a.bindMatrix=this.bindMatrix.toArray(),this.skeleton!==void 0&&(o(t.skeletons,this.skeleton),a.skeleton=this.skeleton.uuid)),this.material!==void 0)if(Array.isArray(this.material)){let n=[];for(let l=0,h=this.material.length;l<h;l++)n.push(o(t.materials,this.material[l]));a.material=n}else a.material=o(t.materials,this.material);if(this.children.length>0){a.children=[];for(let n=0;n<this.children.length;n++)a.children.push(this.children[n].toJSON(t).object)}if(this.animations.length>0){a.animations=[];for(let n=0;n<this.animations.length;n++){let l=this.animations[n];a.animations.push(o(t.animations,l))}}if(i){let n=s(t.geometries),l=s(t.materials),h=s(t.textures),c=s(t.images),u=s(t.shapes),d=s(t.skeletons),f=s(t.animations),p=s(t.nodes);n.length>0&&(r.geometries=n),l.length>0&&(r.materials=l),h.length>0&&(r.textures=h),c.length>0&&(r.images=c),u.length>0&&(r.shapes=u),d.length>0&&(r.skeletons=d),f.length>0&&(r.animations=f),p.length>0&&(r.nodes=p)}return r.object=a,r;function s(n){let l=[];for(let h in n){let c=n[h];delete c.metadata,l.push(c)}return l}}clone(t){return new this.constructor().copy(this,t)}copy(t,i=!0){if(this.name=t.name,this.up.copy(t.up),this.position.copy(t.position),this.rotation.order=t.rotation.order,this.quaternion.copy(t.quaternion),this.scale.copy(t.scale),this.pivot=t.pivot===null?null:t.pivot.clone(),this.matrix.copy(t.matrix),this.matrixWorld.copy(t.matrixWorld),this.matrixAutoUpdate=t.matrixAutoUpdate,this.matrixWorldAutoUpdate=t.matrixWorldAutoUpdate,this.matrixWorldNeedsUpdate=t.matrixWorldNeedsUpdate,this.layers.mask=t.layers.mask,this.visible=t.visible,this.castShadow=t.castShadow,this.receiveShadow=t.receiveShadow,this.frustumCulled=t.frustumCulled,this.renderOrder=t.renderOrder,this.static=t.static,this.animations=t.animations.slice(),this.userData=JSON.parse(JSON.stringify(t.userData)),i===!0)for(let r=0;r<t.children.length;r++){let a=t.children[r];this.add(a.clone())}return this}dispose(){this.dispatchEvent({type:"dispose"})}};Ar.DEFAULT_UP=new I(0,1,0),Ar.DEFAULT_MATRIX_AUTO_UPDATE=!0,Ar.DEFAULT_MATRIX_WORLD_AUTO_UPDATE=!0;var Ji=class extends Ar{constructor(){super(),this.isGroup=!0,this.type="Group"}},w0={type:"move"},dn=class{constructor(){this._targetRay=null,this._grip=null,this._hand=null}getHandSpace(){return this._hand===null&&(this._hand=new Ji,this._hand.matrixAutoUpdate=!1,this._hand.visible=!1,this._hand.joints={},this._hand.inputState={pinching:!1}),this._hand}getTargetRaySpace(){return this._targetRay===null&&(this._targetRay=new Ji,this._targetRay.matrixAutoUpdate=!1,this._targetRay.visible=!1,this._targetRay.hasLinearVelocity=!1,this._targetRay.linearVelocity=new I,this._targetRay.hasAngularVelocity=!1,this._targetRay.angularVelocity=new I),this._targetRay}getGripSpace(){return this._grip===null&&(this._grip=new Ji,this._grip.matrixAutoUpdate=!1,this._grip.visible=!1,this._grip.hasLinearVelocity=!1,this._grip.linearVelocity=new I,this._grip.hasAngularVelocity=!1,this._grip.angularVelocity=new I,this._grip.eventsEnabled=!1),this._grip}dispatchEvent(e){return this._targetRay!==null&&this._targetRay.dispatchEvent(e),this._grip!==null&&this._grip.dispatchEvent(e),this._hand!==null&&this._hand.dispatchEvent(e),this}connect(e){if(e&&e.hand){let t=this._hand;if(t)for(let i of e.hand.values())this._getHandJoint(t,i)}return this.dispatchEvent({type:"connected",data:e}),this}disconnect(e){return this.dispatchEvent({type:"disconnected",data:e}),this._targetRay!==null&&(this._targetRay.visible=!1),this._grip!==null&&(this._grip.visible=!1),this._hand!==null&&(this._hand.visible=!1),this}update(e,t,i){let r=null,a=null,o=null,s=this._targetRay,n=this._grip,l=this._hand;if(e&&t.session.visibilityState!=="visible-blurred"){if(l&&e.hand){o=!0;for(let d of e.hand.values()){let f=t.getJointPose(d,i),p=this._getHandJoint(l,d);f!==null&&(p.matrix.fromArray(f.transform.matrix),p.matrix.decompose(p.position,p.quaternion,p.scale),p.matrixWorldNeedsUpdate=!0,p.jointRadius=f.radius),p.visible=f!==null}let h=l.joints["index-finger-tip"],c=l.joints["thumb-tip"],u=h.position.distanceTo(c.position);l.inputState.pinching&&u>.025?(l.inputState.pinching=!1,this.dispatchEvent({type:"pinchend",handedness:e.handedness,target:this})):!l.inputState.pinching&&u<=.015&&(l.inputState.pinching=!0,this.dispatchEvent({type:"pinchstart",handedness:e.handedness,target:this}))}else n!==null&&e.gripSpace&&(a=t.getPose(e.gripSpace,i),a!==null&&(n.matrix.fromArray(a.transform.matrix),n.matrix.decompose(n.position,n.quaternion,n.scale),n.matrixWorldNeedsUpdate=!0,a.linearVelocity?(n.hasLinearVelocity=!0,n.linearVelocity.copy(a.linearVelocity)):n.hasLinearVelocity=!1,a.angularVelocity?(n.hasAngularVelocity=!0,n.angularVelocity.copy(a.angularVelocity)):n.hasAngularVelocity=!1,n.eventsEnabled&&n.dispatchEvent({type:"gripUpdated",data:e,target:this})));s!==null&&(r=t.getPose(e.targetRaySpace,i),r===null&&a!==null&&(r=a),r!==null&&(s.matrix.fromArray(r.transform.matrix),s.matrix.decompose(s.position,s.quaternion,s.scale),s.matrixWorldNeedsUpdate=!0,r.linearVelocity?(s.hasLinearVelocity=!0,s.linearVelocity.copy(r.linearVelocity)):s.hasLinearVelocity=!1,r.angularVelocity?(s.hasAngularVelocity=!0,s.angularVelocity.copy(r.angularVelocity)):s.hasAngularVelocity=!1,this.dispatchEvent(w0)))}return s!==null&&(s.visible=r!==null),n!==null&&(n.visible=a!==null),l!==null&&(l.visible=o!==null),this}_getHandJoint(e,t){if(e.joints[t.jointName]===void 0){let i=new Ji;i.matrixAutoUpdate=!1,i.visible=!1,e.joints[t.jointName]=i,e.add(i)}return e.joints[t.jointName]}},Hc={aliceblue:15792383,antiquewhite:16444375,aqua:65535,aquamarine:8388564,azure:15794175,beige:16119260,bisque:16770244,black:0,blanchedalmond:16772045,blue:255,blueviolet:9055202,brown:10824234,burlywood:14596231,cadetblue:6266528,chartreuse:8388352,chocolate:13789470,coral:16744272,cornflowerblue:6591981,cornsilk:16775388,crimson:14423100,cyan:65535,darkblue:139,darkcyan:35723,darkgoldenrod:12092939,darkgray:11119017,darkgreen:25600,darkgrey:11119017,darkkhaki:12433259,darkmagenta:9109643,darkolivegreen:5597999,darkorange:16747520,darkorchid:10040012,darkred:9109504,darksalmon:15308410,darkseagreen:9419919,darkslateblue:4734347,darkslategray:3100495,darkslategrey:3100495,darkturquoise:52945,darkviolet:9699539,deeppink:16716947,deepskyblue:49151,dimgray:6908265,dimgrey:6908265,dodgerblue:2003199,firebrick:11674146,floralwhite:16775920,forestgreen:2263842,fuchsia:16711935,gainsboro:14474460,ghostwhite:16316671,gold:16766720,goldenrod:14329120,gray:8421504,green:32768,greenyellow:11403055,grey:8421504,honeydew:15794160,hotpink:16738740,indianred:13458524,indigo:4915330,ivory:16777200,khaki:15787660,lavender:15132410,lavenderblush:16773365,lawngreen:8190976,lemonchiffon:16775885,lightblue:11393254,lightcoral:15761536,lightcyan:14745599,lightgoldenrodyellow:16448210,lightgray:13882323,lightgreen:9498256,lightgrey:13882323,lightpink:16758465,lightsalmon:16752762,lightseagreen:2142890,lightskyblue:8900346,lightslategray:7833753,lightslategrey:7833753,lightsteelblue:11584734,lightyellow:16777184,lime:65280,limegreen:3329330,linen:16445670,magenta:16711935,maroon:8388608,mediumaquamarine:6737322,mediumblue:205,mediumorchid:12211667,mediumpurple:9662683,mediumseagreen:3978097,mediumslateblue:8087790,mediumspringgreen:64154,mediumturquoise:4772300,mediumvioletred:13047173,midnightblue:1644912,mintcream:16121850,mistyrose:16770273,moccasin:16770229,navajowhite:16768685,navy:128,oldlace:16643558,olive:8421376,olivedrab:7048739,orange:16753920,orangered:16729344,orchid:14315734,palegoldenrod:15657130,palegreen:10025880,paleturquoise:11529966,palevioletred:14381203,papayawhip:16773077,peachpuff:16767673,peru:13468991,pink:16761035,plum:14524637,powderblue:11591910,purple:8388736,rebeccapurple:6697881,red:16711680,rosybrown:12357519,royalblue:4286945,saddlebrown:9127187,salmon:16416882,sandybrown:16032864,seagreen:3050327,seashell:16774638,sienna:10506797,silver:12632256,skyblue:8900331,slateblue:6970061,slategray:7372944,slategrey:7372944,snow:16775930,springgreen:65407,steelblue:4620980,tan:13808780,teal:32896,thistle:14204888,tomato:16737095,turquoise:4251856,violet:15631086,wheat:16113331,white:16777215,whitesmoke:16119285,yellow:16776960,yellowgreen:10145074},Nr={h:0,s:0,l:0},Bo={h:0,s:0,l:0};function fn(e,t,i){return i<0&&(i+=1),i>1&&--i,i<1/6?e+(t-e)*6*i:i<1/2?t:i<2/3?e+(t-e)*6*(2/3-i):e}var dt=class{constructor(e,t,i){return this.isColor=!0,this.r=1,this.g=1,this.b=1,this.set(e,t,i)}set(e,t,i){if(t===void 0&&i===void 0){let r=e;r&&r.isColor?this.copy(r):typeof r=="number"?this.setHex(r):typeof r=="string"&&this.setStyle(r)}else this.setRGB(e,t,i);return this}setScalar(e){return this.r=e,this.g=e,this.b=e,this}setHex(e,t=qi){return e=Math.floor(e),this.r=(e>>16&255)/255,this.g=(e>>8&255)/255,this.b=(e&255)/255,ft.colorSpaceToWorking(this,t),this}setRGB(e,t,i,r=ft.workingColorSpace){return this.r=e,this.g=t,this.b=i,ft.colorSpaceToWorking(this,r),this}setHSL(e,t,i,r=ft.workingColorSpace){if(e=l0(e,1),t=ct(t,0,1),i=ct(i,0,1),t===0)this.r=this.g=this.b=i;else{let a=i<=.5?i*(1+t):i+t-i*t,o=2*i-a;this.r=fn(o,a,e+1/3),this.g=fn(o,a,e),this.b=fn(o,a,e-1/3)}return ft.colorSpaceToWorking(this,r),this}setStyle(e,t=qi){function i(a){a!==void 0&&parseFloat(a)<1&&Je("Color: Alpha component of "+e+" will be ignored.")}let r;if(r=/^(\w+)\(([^\)]*)\)/.exec(e)){let a,o=r[1],s=r[2];switch(o){case"rgb":case"rgba":if(a=/^\s*(\d+)\s*,\s*(\d+)\s*,\s*(\d+)\s*(?:,\s*(\d+(?:\.\d+)?|\.\d+)\s*)?$/.exec(s))return i(a[4]),this.setRGB(Math.min(255,parseInt(a[1],10))/255,Math.min(255,parseInt(a[2],10))/255,Math.min(255,parseInt(a[3],10))/255,t);if(a=/^\s*(\d+(?:\.\d+)?|\.\d+)\%\s*,\s*(\d+(?:\.\d+)?|\.\d+)\%\s*,\s*(\d+(?:\.\d+)?|\.\d+)\%\s*(?:,\s*(\d+(?:\.\d+)?|\.\d+)\s*)?$/.exec(s))return i(a[4]),this.setRGB(Math.min(100,parseFloat(a[1]))/100,Math.min(100,parseFloat(a[2]))/100,Math.min(100,parseFloat(a[3]))/100,t);break;case"hsl":case"hsla":if(a=/^\s*([+-]?(?:\d+(?:\.\d+)?|\.\d+))\s*,\s*(\d+(?:\.\d+)?|\.\d+)\%\s*,\s*(\d+(?:\.\d+)?|\.\d+)\%\s*(?:,\s*(\d+(?:\.\d+)?|\.\d+)\s*)?$/.exec(s))return i(a[4]),this.setHSL(parseFloat(a[1])/360,parseFloat(a[2])/100,parseFloat(a[3])/100,t);break;default:Je("Color: Unknown color model "+e)}}else if(r=/^\#([A-Fa-f\d]+)$/.exec(e)){let a=r[1],o=a.length;if(o===3)return this.setRGB(parseInt(a.charAt(0),16)/15,parseInt(a.charAt(1),16)/15,parseInt(a.charAt(2),16)/15,t);if(o===6)return this.setHex(parseInt(a,16),t);Je("Color: Invalid hex color "+e)}else if(e&&e.length>0)return this.setColorName(e,t);return this}setColorName(e,t=qi){let i=Hc[e.toLowerCase()];return i===void 0?Je("Color: Unknown color "+e):this.setHex(i,t),this}clone(){return new this.constructor(this.r,this.g,this.b)}copy(e){return this.r=e.r,this.g=e.g,this.b=e.b,this}copySRGBToLinear(e){return this.r=Er(e.r),this.g=Er(e.g),this.b=Er(e.b),this}copyLinearToSRGB(e){return this.r=Oa(e.r),this.g=Oa(e.g),this.b=Oa(e.b),this}convertSRGBToLinear(){return this.copySRGBToLinear(this),this}convertLinearToSRGB(){return this.copyLinearToSRGB(this),this}getHex(e=qi){return ft.workingToColorSpace(ci.copy(this),e),Math.round(ct(ci.r*255,0,255))*65536+Math.round(ct(ci.g*255,0,255))*256+Math.round(ct(ci.b*255,0,255))}getHexString(e=qi){return("000000"+this.getHex(e).toString(16)).slice(-6)}getHSL(e,t=ft.workingColorSpace){ft.workingToColorSpace(ci.copy(this),t);let i=ci.r,r=ci.g,a=ci.b,o=Math.max(i,r,a),s=Math.min(i,r,a),n,l,h=(s+o)/2;if(s===o)n=0,l=0;else{let c=o-s;switch(l=h<=.5?c/(o+s):c/(2-o-s),o){case i:n=(r-a)/c+(r<a?6:0);break;case r:n=(a-i)/c+2;break;case a:n=(i-r)/c+4}n/=6}return e.h=n,e.s=l,e.l=h,e}getRGB(e,t=ft.workingColorSpace){return ft.workingToColorSpace(ci.copy(this),t),e.r=ci.r,e.g=ci.g,e.b=ci.b,e}getStyle(e=qi){ft.workingToColorSpace(ci.copy(this),e);let t=ci.r,i=ci.g,r=ci.b;return e==="srgb"?`rgb(${Math.round(t*255)},${Math.round(i*255)},${Math.round(r*255)})`:`color(${e} ${t.toFixed(3)} ${i.toFixed(3)} ${r.toFixed(3)})`}offsetHSL(e,t,i){return this.getHSL(Nr),this.setHSL(Nr.h+e,Nr.s+t,Nr.l+i)}add(e){return this.r+=e.r,this.g+=e.g,this.b+=e.b,this}addColors(e,t){return this.r=e.r+t.r,this.g=e.g+t.g,this.b=e.b+t.b,this}addScalar(e){return this.r+=e,this.g+=e,this.b+=e,this}sub(e){return this.r=Math.max(0,this.r-e.r),this.g=Math.max(0,this.g-e.g),this.b=Math.max(0,this.b-e.b),this}multiply(e){return this.r*=e.r,this.g*=e.g,this.b*=e.b,this}multiplyScalar(e){return this.r*=e,this.g*=e,this.b*=e,this}lerp(e,t){return this.r+=(e.r-this.r)*t,this.g+=(e.g-this.g)*t,this.b+=(e.b-this.b)*t,this}lerpColors(e,t,i){return this.r=e.r+(t.r-e.r)*i,this.g=e.g+(t.g-e.g)*i,this.b=e.b+(t.b-e.b)*i,this}lerpHSL(e,t){this.getHSL(Nr),e.getHSL(Bo);let i=sn(Nr.h,Bo.h,t),r=sn(Nr.s,Bo.s,t),a=sn(Nr.l,Bo.l,t);return this.setHSL(i,r,a),this}setFromVector3(e){return this.r=e.x,this.g=e.y,this.b=e.z,this}applyMatrix3(e){let t=this.r,i=this.g,r=this.b,a=e.elements;return this.r=a[0]*t+a[3]*i+a[6]*r,this.g=a[1]*t+a[4]*i+a[7]*r,this.b=a[2]*t+a[5]*i+a[8]*r,this}equals(e){return e.r===this.r&&e.g===this.g&&e.b===this.b}fromArray(e,t=0){return this.r=e[t],this.g=e[t+1],this.b=e[t+2],this}toArray(e=[],t=0){return e[t]=this.r,e[t+1]=this.g,e[t+2]=this.b,e}fromBufferAttribute(e,t){return this.r=e.getX(t),this.g=e.getY(t),this.b=e.getZ(t),this}toJSON(){return this.getHex()}*[Symbol.iterator](){yield this.r,yield this.g,yield this.b}},ci=new dt;dt.NAMES=Hc;var Wc=class extends Ar{constructor(){super(),this.isScene=!0,this.type="Scene",this.background=null,this.environment=null,this.fog=null,this.backgroundBlurriness=0,this.backgroundIntensity=1,this.backgroundRotation=new Cr,this.environmentIntensity=1,this.environmentRotation=new Cr,this.overrideMaterial=null,typeof __THREE_DEVTOOLS__<"u"&&__THREE_DEVTOOLS__.dispatchEvent(new CustomEvent("observe",{detail:this}))}copy(e,t){return super.copy(e,t),this.background=e.background===null?null:e.background.clone(),this.environment=e.environment===null?null:e.environment.clone(),this.fog=e.fog===null?null:e.fog.clone(),this.backgroundBlurriness=e.backgroundBlurriness,this.backgroundIntensity=e.backgroundIntensity,this.backgroundRotation.copy(e.backgroundRotation),this.environmentIntensity=e.environmentIntensity,this.environmentRotation.copy(e.environmentRotation),this.overrideMaterial=e.overrideMaterial===null?null:e.overrideMaterial.clone(),this.matrixAutoUpdate=e.matrixAutoUpdate,this}toJSON(e){let t=super.toJSON(e);return this.fog!==null&&(t.object.fog=this.fog.toJSON()),t.object.backgroundBlurriness=this.backgroundBlurriness,t.object.backgroundIntensity=this.backgroundIntensity,t.object.backgroundRotation=this.backgroundRotation.toArray(),t.object.environmentIntensity=this.environmentIntensity,t.object.environmentRotation=this.environmentRotation.toArray(),t}},ki=new I,xr=new I,pn=new I,_r=new I,fa=new I,pa=new I,sh=new I,mn=new I,gn=new I,vn=new I,xn=new Ft,_n=new Ft,yn=new Ft,Ja=class Ca{constructor(t=new I,i=new I,r=new I){this.a=t,this.b=i,this.c=r}static getNormal(t,i,r,a){a.subVectors(r,i),ki.subVectors(t,i),a.cross(ki);let o=a.lengthSq();return o>0?a.multiplyScalar(1/Math.sqrt(o)):a.set(0,0,0)}static getBarycoord(t,i,r,a,o){ki.subVectors(a,i),xr.subVectors(r,i),pn.subVectors(t,i);let s=ki.dot(ki),n=ki.dot(xr),l=ki.dot(pn),h=xr.dot(xr),c=xr.dot(pn),u=s*h-n*n;if(u===0)return o.set(0,0,0),null;let d=1/u,f=(h*l-n*c)*d,p=(s*c-n*l)*d;return o.set(1-f-p,p,f)}static containsPoint(t,i,r,a){return this.getBarycoord(t,i,r,a,_r)!==null&&_r.x>=0&&_r.y>=0&&_r.x+_r.y<=1}static getInterpolation(t,i,r,a,o,s,n,l){return this.getBarycoord(t,i,r,a,_r)===null?(l.x=0,l.y=0,"z"in l&&(l.z=0),"w"in l&&(l.w=0),null):(l.setScalar(0),l.addScaledVector(o,_r.x),l.addScaledVector(s,_r.y),l.addScaledVector(n,_r.z),l)}static getInterpolatedAttribute(t,i,r,a,o,s){return xn.setScalar(0),_n.setScalar(0),yn.setScalar(0),xn.fromBufferAttribute(t,i),_n.fromBufferAttribute(t,r),yn.fromBufferAttribute(t,a),s.setScalar(0),s.addScaledVector(xn,o.x),s.addScaledVector(_n,o.y),s.addScaledVector(yn,o.z),s}static isFrontFacing(t,i,r,a){return ki.subVectors(r,i),xr.subVectors(t,i),ki.cross(xr).dot(a)<0}set(t,i,r){return this.a.copy(t),this.b.copy(i),this.c.copy(r),this}setFromPointsAndIndices(t,i,r,a){return this.a.copy(t[i]),this.b.copy(t[r]),this.c.copy(t[a]),this}setFromAttributeAndIndices(t,i,r,a){return this.a.fromBufferAttribute(t,i),this.b.fromBufferAttribute(t,r),this.c.fromBufferAttribute(t,a),this}clone(){return new this.constructor().copy(this)}copy(t){return this.a.copy(t.a),this.b.copy(t.b),this.c.copy(t.c),this}getArea(){return ki.subVectors(this.c,this.b),xr.subVectors(this.a,this.b),ki.cross(xr).length()*.5}getMidpoint(t){return t.addVectors(this.a,this.b).add(this.c).multiplyScalar(1/3)}getNormal(t){return Ca.getNormal(this.a,this.b,this.c,t)}getPlane(t){return t.setFromCoplanarPoints(this.a,this.b,this.c)}getBarycoord(t,i){return Ca.getBarycoord(t,this.a,this.b,this.c,i)}getInterpolation(t,i,r,a,o){return Ca.getInterpolation(t,this.a,this.b,this.c,i,r,a,o)}containsPoint(t){return Ca.containsPoint(t,this.a,this.b,this.c)}isFrontFacing(t){return Ca.isFrontFacing(this.a,this.b,this.c,t)}intersectsBox(t){return t.intersectsTriangle(this)}closestPointToPoint(t,i){let r=this.a,a=this.b,o=this.c,s,n;fa.subVectors(a,r),pa.subVectors(o,r),mn.subVectors(t,r);let l=fa.dot(mn),h=pa.dot(mn);if(l<=0&&h<=0)return i.copy(r);gn.subVectors(t,a);let c=fa.dot(gn),u=pa.dot(gn);if(c>=0&&u<=c)return i.copy(a);let d=l*u-c*h;if(d<=0&&l>=0&&c<=0&&l-c>0)return s=l/(l-c),i.copy(r).addScaledVector(fa,s);vn.subVectors(t,o);let f=fa.dot(vn),p=pa.dot(vn);if(p>=0&&f<=p)return i.copy(o);let x=f*h-l*p;if(x<=0&&h>=0&&p<=0)return n=h/(h-p),i.copy(r).addScaledVector(pa,n);let g=c*p-f*u;if(g<=0&&u-c>=0&&f-p>=0)return sh.subVectors(o,a),n=(u-c)/(u-c+(f-p)),i.copy(a).addScaledVector(sh,n);let m=1/(g+x+d);return s=x*m,n=d*m,i.copy(r).addScaledVector(fa,s).addScaledVector(pa,n)}equals(t){return t.a.equals(this.a)&&t.b.equals(this.b)&&t.c.equals(this.c)}},la=class{constructor(e=new I(1/0,1/0,1/0),t=new I(-1/0,-1/0,-1/0)){this.isBox3=!0,this.min=e,this.max=t}set(e,t){return this.min.copy(e),this.max.copy(t),this}setFromArray(e){this.makeEmpty();for(let t=0,i=e.length;t<i;t+=3)this.expandByPoint(Vi.fromArray(e,t));return this}setFromBufferAttribute(e){this.makeEmpty();for(let t=0,i=e.count;t<i;t++)this.expandByPoint(Vi.fromBufferAttribute(e,t));return this}setFromPoints(e){this.makeEmpty();for(let t=0,i=e.length;t<i;t++)this.expandByPoint(e[t]);return this}setFromCenterAndSize(e,t){let i=Vi.copy(t).multiplyScalar(.5);return this.min.copy(e).sub(i),this.max.copy(e).add(i),this}setFromObject(e,t=!1){return this.makeEmpty(),this.expandByObject(e,t)}clone(){return new this.constructor().copy(this)}copy(e){return this.min.copy(e.min),this.max.copy(e.max),this}makeEmpty(){return this.min.x=this.min.y=this.min.z=1/0,this.max.x=this.max.y=this.max.z=-1/0,this}isEmpty(){return this.max.x<this.min.x||this.max.y<this.min.y||this.max.z<this.min.z}getCenter(e){return this.isEmpty()?e.set(0,0,0):e.addVectors(this.min,this.max).multiplyScalar(.5)}getSize(e){return this.isEmpty()?e.set(0,0,0):e.subVectors(this.max,this.min)}expandByPoint(e){return this.min.min(e),this.max.max(e),this}expandByVector(e){return this.min.sub(e),this.max.add(e),this}expandByScalar(e){return this.min.addScalar(-e),this.max.addScalar(e),this}expandByObject(e,t=!1){e.updateWorldMatrix(!1,!1);let i=e.geometry;if(i!==void 0){let a=i.getAttribute("position");if(t===!0&&a!==void 0&&e.isInstancedMesh!==!0&&e.isBatchedMesh!==!0)for(let o=0,s=a.count;o<s;o++)e.isMesh===!0?e.getVertexPosition(o,Vi):Vi.fromBufferAttribute(a,o),Vi.applyMatrix4(e.matrixWorld),this.expandByPoint(Vi);else e.boundingBox===void 0?(i.boundingBox===null&&i.computeBoundingBox(),ko.copy(i.boundingBox)):(e.boundingBox===null&&e.computeBoundingBox(),ko.copy(e.boundingBox)),ko.applyMatrix4(e.matrixWorld),this.union(ko)}let r=e.children;for(let a=0,o=r.length;a<o;a++)this.expandByObject(r[a],t);return this}containsPoint(e){return e.x>=this.min.x&&e.x<=this.max.x&&e.y>=this.min.y&&e.y<=this.max.y&&e.z>=this.min.z&&e.z<=this.max.z}containsBox(e){return this.min.x<=e.min.x&&e.max.x<=this.max.x&&this.min.y<=e.min.y&&e.max.y<=this.max.y&&this.min.z<=e.min.z&&e.max.z<=this.max.z}getParameter(e,t){return t.set((e.x-this.min.x)/(this.max.x-this.min.x),(e.y-this.min.y)/(this.max.y-this.min.y),(e.z-this.min.z)/(this.max.z-this.min.z))}intersectsBox(e){return e.max.x>=this.min.x&&e.min.x<=this.max.x&&e.max.y>=this.min.y&&e.min.y<=this.max.y&&e.max.z>=this.min.z&&e.min.z<=this.max.z}intersectsSphere(e){return this.clampPoint(e.center,Vi),Vi.distanceToSquared(e.center)<=e.radius*e.radius}intersectsPlane(e){let t,i;return e.normal.x>0?(t=e.normal.x*this.min.x,i=e.normal.x*this.max.x):(t=e.normal.x*this.max.x,i=e.normal.x*this.min.x),e.normal.y>0?(t+=e.normal.y*this.min.y,i+=e.normal.y*this.max.y):(t+=e.normal.y*this.max.y,i+=e.normal.y*this.min.y),e.normal.z>0?(t+=e.normal.z*this.min.z,i+=e.normal.z*this.max.z):(t+=e.normal.z*this.max.z,i+=e.normal.z*this.min.z),t<=-e.constant&&i>=-e.constant}intersectsTriangle(e){if(this.isEmpty())return!1;this.getCenter(Ka),Vo.subVectors(this.max,Ka),ma.subVectors(e.a,Ka),ga.subVectors(e.b,Ka),va.subVectors(e.c,Ka),Ir.subVectors(ga,ma),Dr.subVectors(va,ga),qr.subVectors(ma,va);let t=[0,-Ir.z,Ir.y,0,-Dr.z,Dr.y,0,-qr.z,qr.y,Ir.z,0,-Ir.x,Dr.z,0,-Dr.x,qr.z,0,-qr.x,-Ir.y,Ir.x,0,-Dr.y,Dr.x,0,-qr.y,qr.x,0];return!bn(t,ma,ga,va,Vo)||(t=[1,0,0,0,1,0,0,0,1],!bn(t,ma,ga,va,Vo))?!1:(Go.crossVectors(Ir,Dr),t=[Go.x,Go.y,Go.z],bn(t,ma,ga,va,Vo))}clampPoint(e,t){return t.copy(e).clamp(this.min,this.max)}distanceToPoint(e){return this.clampPoint(e,Vi).distanceTo(e)}getBoundingSphere(e){return this.isEmpty()?e.makeEmpty():(this.getCenter(e.center),e.radius=this.getSize(Vi).length()*.5),e}intersect(e){return this.min.max(e.min),this.max.min(e.max),this.isEmpty()&&this.makeEmpty(),this}union(e){return this.min.min(e.min),this.max.max(e.max),this}applyMatrix4(e){return this.isEmpty()||(yr[0].set(this.min.x,this.min.y,this.min.z).applyMatrix4(e),yr[1].set(this.min.x,this.min.y,this.max.z).applyMatrix4(e),yr[2].set(this.min.x,this.max.y,this.min.z).applyMatrix4(e),yr[3].set(this.min.x,this.max.y,this.max.z).applyMatrix4(e),yr[4].set(this.max.x,this.min.y,this.min.z).applyMatrix4(e),yr[5].set(this.max.x,this.min.y,this.max.z).applyMatrix4(e),yr[6].set(this.max.x,this.max.y,this.min.z).applyMatrix4(e),yr[7].set(this.max.x,this.max.y,this.max.z).applyMatrix4(e),this.setFromPoints(yr)),this}translate(e){return this.min.add(e),this.max.add(e),this}equals(e){return e.min.equals(this.min)&&e.max.equals(this.max)}toJSON(){return{min:this.min.toArray(),max:this.max.toArray()}}fromJSON(e){return this.min.fromArray(e.min),this.max.fromArray(e.max),this}},yr=[new I,new I,new I,new I,new I,new I,new I,new I],Vi=new I,ko=new la,ma=new I,ga=new I,va=new I,Ir=new I,Dr=new I,qr=new I,Ka=new I,Vo=new I,Go=new I,jr=new I;function bn(e,t,i,r,a){for(let o=0,s=e.length-3;o<=s;o+=3){jr.fromArray(e,o);let n=a.x*Math.abs(jr.x)+a.y*Math.abs(jr.y)+a.z*Math.abs(jr.z),l=t.dot(jr),h=i.dot(jr),c=r.dot(jr);if(Math.max(-Math.max(l,h,c),Math.min(l,h,c))>n)return!1}return!0}var Zt=new I,Ho=new ge,T0=0,si=class extends na{constructor(e,t,i=!1){if(super(),Array.isArray(e))throw TypeError("THREE.BufferAttribute: array should be a Typed Array.");this.isBufferAttribute=!0,Object.defineProperty(this,"id",{value:T0++}),this.name="",this.array=e,this.itemSize=t,this.count=e===void 0?0:e.length/t,this.normalized=i,this.usage=i0,this.updateRanges=[],this.gpuType=Co,this.version=0,this._normalizationScale=e===void 0?void 0:xl(e)}onUploadCallback(){}set needsUpdate(e){e===!0&&this.version++}setUsage(e){return this.usage=e,this}addUpdateRange(e,t){this.updateRanges.push({start:e,count:t})}clearUpdateRanges(){this.updateRanges.length=0}copy(e){return this.name=e.name,this.array=new e.array.constructor(e.array),this.itemSize=e.itemSize,this.count=e.count,this.normalized=e.normalized,this.usage=e.usage,this.gpuType=e.gpuType,this._normalizationScale=e._normalizationScale,this}copyAt(e,t,i){e*=this.itemSize,i*=t.itemSize;for(let r=0,a=this.itemSize;r<a;r++)this.array[e+r]=t.array[i+r];return this}copyArray(e){return this.array.set(e),this}applyMatrix3(e){if(this.itemSize===2)for(let t=0,i=this.count;t<i;t++)Ho.fromBufferAttribute(this,t),Ho.applyMatrix3(e),this.setXY(t,Ho.x,Ho.y);else if(this.itemSize===3)for(let t=0,i=this.count;t<i;t++)Zt.fromBufferAttribute(this,t),Zt.applyMatrix3(e),this.setXYZ(t,Zt.x,Zt.y,Zt.z);return this}applyMatrix4(e){for(let t=0,i=this.count;t<i;t++)Zt.fromBufferAttribute(this,t),Zt.applyMatrix4(e),this.setXYZ(t,Zt.x,Zt.y,Zt.z);return this}applyNormalMatrix(e){for(let t=0,i=this.count;t<i;t++)Zt.fromBufferAttribute(this,t),Zt.applyNormalMatrix(e),this.setXYZ(t,Zt.x,Zt.y,Zt.z);return this}transformDirection(e){for(let t=0,i=this.count;t<i;t++)Zt.fromBufferAttribute(this,t),Zt.transformDirection(e),this.setXYZ(t,Zt.x,Zt.y,Zt.z);return this}set(e,t=0){return this.array.set(e,t),this}getComponent(e,t){let i=this.array[e*this.itemSize+t];return this.normalized&&(i=this._denormalize(i)),i}setComponent(e,t,i){return this.normalized&&(i=this._normalize(i)),this.array[e*this.itemSize+t]=i,this}getX(e){let t=this.array[e*this.itemSize];return this.normalized&&(t=this._denormalize(t)),t}setX(e,t){return this.normalized&&(t=this._normalize(t)),this.array[e*this.itemSize]=t,this}getY(e){let t=this.array[e*this.itemSize+1];return this.normalized&&(t=this._denormalize(t)),t}setY(e,t){return this.normalized&&(t=this._normalize(t)),this.array[e*this.itemSize+1]=t,this}getZ(e){let t=this.array[e*this.itemSize+2];return this.normalized&&(t=this._denormalize(t)),t}setZ(e,t){return this.normalized&&(t=this._normalize(t)),this.array[e*this.itemSize+2]=t,this}getW(e){let t=this.array[e*this.itemSize+3];return this.normalized&&(t=this._denormalize(t)),t}setW(e,t){return this.normalized&&(t=this._normalize(t)),this.array[e*this.itemSize+3]=t,this}setXY(e,t,i){return e*=this.itemSize,this.normalized&&(t=this._normalize(t),i=this._normalize(i)),this.array[e+0]=t,this.array[e+1]=i,this}setXYZ(e,t,i,r){return e*=this.itemSize,this.normalized&&(t=this._normalize(t),i=this._normalize(i),r=this._normalize(r)),this.array[e+0]=t,this.array[e+1]=i,this.array[e+2]=r,this}setXYZW(e,t,i,r,a){return e*=this.itemSize,this.normalized&&(t=this._normalize(t),i=this._normalize(i),r=this._normalize(r),a=this._normalize(a)),this.array[e+0]=t,this.array[e+1]=i,this.array[e+2]=r,this.array[e+3]=a,this}onUpload(e){return this.onUploadCallback=e,this}clone(){return new this.constructor(this.array,this.itemSize).copy(this)}toJSON(){let e={itemSize:this.itemSize,type:this.array.constructor.name,array:Array.from(this.array),normalized:this.normalized};return e.name=this.name,e.usage=this.usage,e.gpuType=this.gpuType,e}dispose(){this.dispatchEvent({type:"dispose"})}_denormalize(e){let t=this._normalizationScale;return t===void 0?c0(e,this.array):Math.max(e/t,-1)}_normalize(e){let t=this._normalizationScale;return t===void 0?u0(e,this.array):Math.round(e*t)}},Xc=class extends si{constructor(e,t,i){super(new Uint16Array(e),t,i)}},qc=class extends si{constructor(e,t,i){super(new Uint32Array(e),t,i)}},wt=class extends si{constructor(e,t,i){super(new Float32Array(e),t,i)}},E0=new la,$a=new I,Mn=new I,Lo=class{constructor(e=new I,t=-1){this.isSphere=!0,this.center=e,this.radius=t}set(e,t){return this.center.copy(e),this.radius=t,this}setFromPoints(e,t){let i=this.center;t===void 0?E0.setFromPoints(e).getCenter(i):i.copy(t);let r=0;for(let a=0,o=e.length;a<o;a++)r=Math.max(r,i.distanceToSquared(e[a]));return this.radius=Math.sqrt(r),this}copy(e){return this.center.copy(e.center),this.radius=e.radius,this}isEmpty(){return this.radius<0}makeEmpty(){return this.center.set(0,0,0),this.radius=-1,this}containsPoint(e){return e.distanceToSquared(this.center)<=this.radius*this.radius}distanceToPoint(e){return e.distanceTo(this.center)-this.radius}intersectsSphere(e){let t=this.radius+e.radius;return e.center.distanceToSquared(this.center)<=t*t}intersectsBox(e){return e.intersectsSphere(this)}intersectsPlane(e){return Math.abs(e.distanceToPoint(this.center))<=this.radius}clampPoint(e,t){let i=this.center.distanceToSquared(e);return t.copy(e),i>this.radius*this.radius&&(t.sub(this.center).normalize(),t.multiplyScalar(this.radius).add(this.center)),t}getBoundingBox(e){return this.isEmpty()?(e.makeEmpty(),e):(e.set(this.center,this.center),e.expandByScalar(this.radius),e)}applyMatrix4(e){return this.center.applyMatrix4(e),this.radius*=e.getMaxScaleOnAxis(),this}translate(e){return this.center.add(e),this}expandByPoint(e){if(this.isEmpty())return this.center.copy(e),this.radius=0,this;$a.subVectors(e,this.center);let t=$a.lengthSq();if(t>this.radius*this.radius){let i=Math.sqrt(t),r=(i-this.radius)*.5;this.center.addScaledVector($a,r/i),this.radius+=r}return this}union(e){return e.isEmpty()?this:this.isEmpty()?(this.copy(e),this):(this.center.equals(e.center)===!0?this.radius=Math.max(this.radius,e.radius):(Mn.subVectors(e.center,this.center).setLength(e.radius),this.expandByPoint($a.copy(e.center).add(Mn)),this.expandByPoint($a.copy(e.center).sub(Mn))),this)}equals(e){return e.center.equals(this.center)&&e.radius===this.radius}clone(){return new this.constructor().copy(this)}toJSON(){return{radius:this.radius,center:this.center.toArray()}}fromJSON(e){return this.radius=e.radius,this.center.fromArray(e.center),this}},A0=0,Li=new gt,Sn=new Ar,xa=new I,Ti=new la,Qa=new la,$t=new I,ti=class jc extends na{constructor(){super(),this.isBufferGeometry=!0,Object.defineProperty(this,"id",{value:A0++}),this.uuid=qa(),this.name="",this.type="BufferGeometry",this.index=null,this.indirect=null,this.indirectOffset=0,this.attributes={},this.morphAttributes={},this.morphTargetsRelative=!1,this.groups=[],this.boundingBox=null,this.boundingSphere=null,this.drawRange={start:0,count:1/0},this.userData={},this._transformed=!1}getIndex(){return this.index}setIndex(t){return this.index=Array.isArray(t)?new(r0(t)?qc:Xc)(t,1):t,this}setIndirect(t,i=0){return this.indirect=t,this.indirectOffset=i,this}getIndirect(){return this.indirect}getAttribute(t){return this.attributes[t]}setAttribute(t,i){return this.attributes[t]=i,this}deleteAttribute(t){return delete this.attributes[t],this}hasAttribute(t){return this.attributes[t]!==void 0}addGroup(t,i,r=0){this.groups.push({start:t,count:i,materialIndex:r})}clearGroups(){this.groups=[]}setDrawRange(t,i){this.drawRange.start=t,this.drawRange.count=i}applyMatrix4(t){let i=this.attributes.position;i!==void 0&&(i.applyMatrix4(t),i.needsUpdate=!0);let r=this.attributes.normal;if(r!==void 0){let o=new it().getNormalMatrix(t);r.applyNormalMatrix(o),r.needsUpdate=!0}let a=this.attributes.tangent;return a!==void 0&&(a.transformDirection(t),a.needsUpdate=!0),this.boundingBox!==null&&this.computeBoundingBox(),this.boundingSphere!==null&&this.computeBoundingSphere(),this._transformed=!0,this}applyQuaternion(t){return Li.makeRotationFromQuaternion(t),this.applyMatrix4(Li),this}rotateX(t){return Li.makeRotationX(t),this.applyMatrix4(Li),this}rotateY(t){return Li.makeRotationY(t),this.applyMatrix4(Li),this}rotateZ(t){return Li.makeRotationZ(t),this.applyMatrix4(Li),this}translate(t,i,r){return Li.makeTranslation(t,i,r),this.applyMatrix4(Li),this}scale(t,i,r){return Li.makeScale(t,i,r),this.applyMatrix4(Li),this}lookAt(t){return Sn.lookAt(t),Sn.updateMatrix(),this.applyMatrix4(Sn.matrix),this}center(){return this.computeBoundingBox(),this.boundingBox.getCenter(xa).negate(),this.translate(xa.x,xa.y,xa.z),this}setFromPoints(t){let i=this.getAttribute("position");if(i===void 0){let r=[];for(let a=0,o=t.length;a<o;a++){let s=t[a];r.push(s.x,s.y,s.z||0)}this.setAttribute("position",new wt(r,3))}else{let r=Math.min(t.length,i.count);for(let a=0;a<r;a++){let o=t[a];i.setXYZ(a,o.x,o.y,o.z||0)}t.length>i.count&&Je("BufferGeometry: Buffer size too small for points data. Use .dispose() and create a new geometry."),i.needsUpdate=!0}return this}computeBoundingBox(){this.boundingBox===null&&(this.boundingBox=new la);let t=this.attributes.position,i=this.morphAttributes.position;if(t&&t.isGLBufferAttribute)Qe("BufferGeometry.computeBoundingBox(): GLBufferAttribute requires a manual bounding box.",this),this.boundingBox.set(new I(-1/0,-1/0,-1/0),new I(1/0,1/0,1/0));else{if(t!==void 0){if(this.boundingBox.setFromBufferAttribute(t),i)for(let r=0,a=i.length;r<a;r++){let o=i[r];Ti.setFromBufferAttribute(o),this.morphTargetsRelative?($t.addVectors(this.boundingBox.min,Ti.min),this.boundingBox.expandByPoint($t),$t.addVectors(this.boundingBox.max,Ti.max),this.boundingBox.expandByPoint($t)):(this.boundingBox.expandByPoint(Ti.min),this.boundingBox.expandByPoint(Ti.max))}}else this.boundingBox.makeEmpty();(isNaN(this.boundingBox.min.x)||isNaN(this.boundingBox.min.y)||isNaN(this.boundingBox.min.z))&&Qe('BufferGeometry.computeBoundingBox(): Computed min/max have NaN values. The "position" attribute is likely to have NaN values.',this)}}computeBoundingSphere(){this.boundingSphere===null&&(this.boundingSphere=new Lo);let t=this.attributes.position,i=this.morphAttributes.position;if(t&&t.isGLBufferAttribute)Qe("BufferGeometry.computeBoundingSphere(): GLBufferAttribute requires a manual bounding sphere.",this),this.boundingSphere.set(new I,1/0);else if(t){let r=this.boundingSphere.center;if(Ti.setFromBufferAttribute(t),i)for(let o=0,s=i.length;o<s;o++){let n=i[o];Qa.setFromBufferAttribute(n),this.morphTargetsRelative?($t.addVectors(Ti.min,Qa.min),Ti.expandByPoint($t),$t.addVectors(Ti.max,Qa.max),Ti.expandByPoint($t)):(Ti.expandByPoint(Qa.min),Ti.expandByPoint(Qa.max))}Ti.getCenter(r);let a=0;for(let o=0,s=t.count;o<s;o++)$t.fromBufferAttribute(t,o),a=Math.max(a,r.distanceToSquared($t));if(i)for(let o=0,s=i.length;o<s;o++){let n=i[o],l=this.morphTargetsRelative;for(let h=0,c=n.count;h<c;h++)$t.fromBufferAttribute(n,h),l&&(xa.fromBufferAttribute(t,h),$t.add(xa)),a=Math.max(a,r.distanceToSquared($t))}this.boundingSphere.radius=Math.sqrt(a),isNaN(this.boundingSphere.radius)&&Qe('BufferGeometry.computeBoundingSphere(): Computed radius is NaN. The "position" attribute is likely to have NaN values.',this)}}computeTangents(){let t=this.index,i=this.attributes;if(t===null||i.position===void 0||i.normal===void 0||i.uv===void 0){Qe("BufferGeometry: .computeTangents() failed. Missing required attributes (index, position, normal or uv)");return}let r=i.position,a=i.normal,o=i.uv,s=this.getAttribute("tangent");(s===void 0||s.count!==r.count)&&(s=new si(new Float32Array(4*r.count),4),this.setAttribute("tangent",s));let n=[],l=[],h=new Uint8Array(r.count);for(let A=0;A<r.count;A++)n[A]=new I,l[A]=new I;let c=new I,u=new I,d=new I,f=new ge,p=new ge,x=new ge,g=new I,m=new I;function y(A,C,R){h[A]=h[C]=h[R]=1,c.fromBufferAttribute(r,A),u.fromBufferAttribute(r,C),d.fromBufferAttribute(r,R),f.fromBufferAttribute(o,A),p.fromBufferAttribute(o,C),x.fromBufferAttribute(o,R),u.sub(c),d.sub(c),p.sub(f),x.sub(f);let L=1/(p.x*x.y-x.x*p.y);isFinite(L)&&(g.copy(u).multiplyScalar(x.y).addScaledVector(d,-p.y).multiplyScalar(L),m.copy(d).multiplyScalar(p.x).addScaledVector(u,-x.x).multiplyScalar(L),n[A].add(g),n[C].add(g),n[R].add(g),l[A].add(m),l[C].add(m),l[R].add(m))}let M=this.groups;M.length===0&&(M=[{start:0,count:t.count}]);for(let A=0,C=M.length;A<C;++A){let R=M[A],L=R.start,G=R.count;for(let P=L,z=L+G;P<z;P+=3)y(t.getX(P+0),t.getX(P+1),t.getX(P+2))}let b=new I,E=new I,T=new I,v=new I;function _(A){T.fromBufferAttribute(a,A),v.copy(T);let C=n[A];b.copy(C),b.sub(T.multiplyScalar(T.dot(C))).normalize(),E.crossVectors(v,C);let R=E.dot(l[A])<0?-1:1;s.setXYZW(A,b.x,b.y,b.z,R)}for(let A=0,C=r.count;A<C;++A)h[A]&&_(A);this._transformed=!0}computeVertexNormals(){let t=this.index,i=this.getAttribute("position");if(i!==void 0){let r=this.getAttribute("normal");if(r===void 0||r.count!==i.count)r=new si(new Float32Array(i.count*3),3),this.setAttribute("normal",r);else for(let d=0,f=r.count;d<f;d++)r.setXYZ(d,0,0,0);let a=new I,o=new I,s=new I,n=new I,l=new I,h=new I,c=new I,u=new I;if(t)for(let d=0,f=t.count;d<f;d+=3){let p=t.getX(d+0),x=t.getX(d+1),g=t.getX(d+2);a.fromBufferAttribute(i,p),o.fromBufferAttribute(i,x),s.fromBufferAttribute(i,g),c.subVectors(s,o),u.subVectors(a,o),c.cross(u),n.fromBufferAttribute(r,p),l.fromBufferAttribute(r,x),h.fromBufferAttribute(r,g),n.add(c),l.add(c),h.add(c),r.setXYZ(p,n.x,n.y,n.z),r.setXYZ(x,l.x,l.y,l.z),r.setXYZ(g,h.x,h.y,h.z)}else for(let d=0,f=i.count;d<f;d+=3)a.fromBufferAttribute(i,d+0),o.fromBufferAttribute(i,d+1),s.fromBufferAttribute(i,d+2),c.subVectors(s,o),u.subVectors(a,o),c.cross(u),r.setXYZ(d+0,c.x,c.y,c.z),r.setXYZ(d+1,c.x,c.y,c.z),r.setXYZ(d+2,c.x,c.y,c.z);this.normalizeNormals(),r.needsUpdate=!0}}normalizeNormals(){let t=this.attributes.normal;for(let i=0,r=t.count;i<r;i++)$t.fromBufferAttribute(t,i),$t.normalize(),t.setXYZ(i,$t.x,$t.y,$t.z)}toNonIndexed(){function t(n,l){let h=n.array,c=n.itemSize,u=n.normalized,d=new h.constructor(l.length*c),f=0,p=0;for(let x=0,g=l.length;x<g;x++){f=n.isInterleavedBufferAttribute?l[x]*n.data.stride+n.offset:l[x]*c;for(let m=0;m<c;m++)d[p++]=h[f++]}return new si(d,c,u)}if(this.index===null)return Je("BufferGeometry.toNonIndexed(): BufferGeometry is already non-indexed."),this;let i=new jc,r=this.index.array,a=this.attributes;for(let n in a){let l=a[n],h=t(l,r);i.setAttribute(n,h)}let o=this.morphAttributes;for(let n in o){let l=[],h=o[n];for(let c=0,u=h.length;c<u;c++){let d=h[c],f=t(d,r);l.push(f)}i.morphAttributes[n]=l}i.morphTargetsRelative=this.morphTargetsRelative;let s=this.groups;for(let n=0,l=s.length;n<l;n++){let h=s[n];i.addGroup(h.start,h.count,h.materialIndex)}return i}toJSON(){let t={metadata:{version:4.7,type:"BufferGeometry",generator:"BufferGeometry.toJSON"}};if(t.uuid=this.uuid,t.type=this.parameters!==void 0&&this._transformed===!0?"BufferGeometry":this.type,t.name=this.name,Object.keys(this.userData).length>0&&(t.userData=this.userData),this.parameters!==void 0&&this._transformed!==!0){let l=this.parameters;for(let h in l)l[h]!==void 0&&(t[h]=l[h]);return t}t.data={attributes:{}};let i=this.index;i!==null&&(t.data.index={type:i.array.constructor.name,array:Array.prototype.slice.call(i.array)});let r=this.attributes;for(let l in r){let h=r[l];t.data.attributes[l]=h.toJSON(t.data)}let a={},o=!1;for(let l in this.morphAttributes){let h=this.morphAttributes[l],c=[];for(let u=0,d=h.length;u<d;u++){let f=h[u];c.push(f.toJSON(t.data))}c.length>0&&(a[l]=c,o=!0)}o&&(t.data.morphAttributes=a,t.data.morphTargetsRelative=this.morphTargetsRelative);let s=this.groups;s.length>0&&(t.data.groups=JSON.parse(JSON.stringify(s)));let n=this.boundingSphere;return n!==null&&(t.data.boundingSphere=n.toJSON()),t}clone(){return new this.constructor().copy(this)}copy(t){this.index=null,this.attributes={},this.morphAttributes={},this.groups=[],this.boundingBox=null,this.boundingSphere=null;let i={};this.name=t.name;let r=t.index;r!==null&&this.setIndex(r.clone());let a=t.attributes;for(let h in a){let c=a[h];this.setAttribute(h,c.clone(i))}let o=t.morphAttributes;for(let h in o){let c=[],u=o[h];for(let d=0,f=u.length;d<f;d++)c.push(u[d].clone(i));this.morphAttributes[h]=c}this.morphTargetsRelative=t.morphTargetsRelative;let s=t.groups;for(let h=0,c=s.length;h<c;h++){let u=s[h];this.addGroup(u.start,u.count,u.materialIndex)}let n=t.boundingBox;n!==null&&(this.boundingBox=n.clone());let l=t.boundingSphere;return l!==null&&(this.boundingSphere=l.clone()),this.drawRange.start=t.drawRange.start,this.drawRange.count=t.drawRange.count,this.userData=t.userData,this._transformed=t._transformed,this}dispose(){this.dispatchEvent({type:"dispose"})}},wn=new I,R0=new I,C0=new it,Or=class{constructor(e=new I(1,0,0),t=0){this.isPlane=!0,this.normal=e,this.constant=t}set(e,t){return this.normal.copy(e),this.constant=t,this}setComponents(e,t,i,r){return this.normal.set(e,t,i),this.constant=r,this}setFromNormalAndCoplanarPoint(e,t){return this.normal.copy(e),this.constant=-t.dot(this.normal),this}setFromCoplanarPoints(e,t,i){let r=wn.subVectors(i,t).cross(R0.subVectors(e,t)).normalize();return this.setFromNormalAndCoplanarPoint(r,e),this}copy(e){return this.normal.copy(e.normal),this.constant=e.constant,this}normalize(){let e=1/this.normal.length();return this.normal.multiplyScalar(e),this.constant*=e,this}negate(){return this.constant*=-1,this.normal.negate(),this}distanceToPoint(e){return this.normal.dot(e)+this.constant}distanceToSphere(e){return this.distanceToPoint(e.center)-e.radius}projectPoint(e,t){return t.copy(e).addScaledVector(this.normal,-this.distanceToPoint(e))}intersectLine(e,t,i=!0){let r=e.delta(wn),a=this.normal.dot(r);if(a===0)return this.distanceToPoint(e.start)===0?t.copy(e.start):null;let o=-(e.start.dot(this.normal)+this.constant)/a;return i===!0&&(o<0||o>1)?null:t.copy(e.start).addScaledVector(r,o)}intersectsLine(e){let t=this.distanceToPoint(e.start),i=this.distanceToPoint(e.end);return t<0&&i>0||i<0&&t>0}intersectsBox(e){return e.intersectsPlane(this)}intersectsSphere(e){return e.intersectsPlane(this)}coplanarPoint(e){return e.copy(this.normal).multiplyScalar(-this.constant)}applyMatrix4(e,t){let i=t||C0.getNormalMatrix(e),r=this.coplanarPoint(wn).applyMatrix4(e),a=this.normal.applyMatrix3(i).normalize();return this.constant=-r.dot(a),this}translate(e){return this.constant-=e.dot(this.normal),this}equals(e){return e.normal.equals(this.normal)&&e.constant===this.constant}clone(){return new this.constructor().copy(this)}toJSON(){return{normal:this.normal.toArray(),constant:this.constant}}fromJSON(e){return this.normal.fromArray(e.normal),this.constant=e.constant,this}},P0=0,Zs=class extends na{constructor(){super(),this.isMaterial=!0,Object.defineProperty(this,"id",{value:P0++}),this.uuid=qa(),this.name="",this.type="Material",this.blending=1,this.side=0,this.vertexColors=!1,this.opacity=1,this.transparent=!1,this.alphaHash=!1,this.blendSrc=204,this.blendDst=205,this.blendEquation=100,this.blendSrcAlpha=null,this.blendDstAlpha=null,this.blendEquationAlpha=null,this.blendColor=new dt(0,0,0),this.blendAlpha=0,this.depthFunc=3,this.depthTest=!0,this.depthWrite=!0,this.stencilWriteMask=255,this.stencilFunc=519,this.stencilRef=0,this.stencilFuncMask=255,this.stencilFail=an,this.stencilZFail=an,this.stencilZPass=an,this.stencilWrite=!1,this.clippingPlanes=null,this.clipIntersection=!1,this.clipShadows=!1,this.shadowSide=null,this.colorWrite=!0,this.precision=null,this.polygonOffset=!1,this.polygonOffsetFactor=0,this.polygonOffsetUnits=0,this.dithering=!1,this.alphaToCoverage=!1,this.premultipliedAlpha=!1,this.forceSinglePass=!1,this.allowOverride=!0,this.visible=!0,this.toneMapped=!0,this.userData={},this.version=0,this._alphaTest=0}get alphaTest(){return this._alphaTest}set alphaTest(e){this._alphaTest>0!=e>0&&this.version++,this._alphaTest=e}onBeforeRender(){}onBeforeCompile(){}customProgramCacheKey(){return this.onBeforeCompile.toString()}setValues(e){if(e!==void 0)for(let t in e){let i=e[t];if(i===void 0){Je(`Material: parameter '${t}' has value of undefined.`);continue}let r=this[t];r===void 0?Je(`Material: '${t}' is not a property of THREE.${this.type}.`):r&&r.isColor?r.set(i):r&&r.isVector2&&i&&i.isVector2||r&&r.isEuler&&i&&i.isEuler||r&&r.isVector3&&i&&i.isVector3?r.copy(i):this[t]=i}}toJSON(e){let t=e===void 0||typeof e=="string";t&&(e={textures:{},images:{}});let i={metadata:{version:4.7,type:"Material",generator:"Material.toJSON"}};i.uuid=this.uuid,i.type=this.type,i.blending=this.blending,i.side=this.side,i.shadowSide=this.shadowSide,i.vertexColors=this.vertexColors,i.opacity=this.opacity,i.transparent=this.transparent,i.blendSrc=this.blendSrc,i.blendDst=this.blendDst,i.blendEquation=this.blendEquation,i.blendSrcAlpha=this.blendSrcAlpha,i.blendDstAlpha=this.blendDstAlpha,i.blendEquationAlpha=this.blendEquationAlpha,i.blendColor=this.blendColor.getHex(),i.blendAlpha=this.blendAlpha,i.depthFunc=this.depthFunc,i.depthTest=this.depthTest,i.depthWrite=this.depthWrite,i.colorWrite=this.colorWrite,i.clipIntersection=this.clipIntersection,i.clipShadows=this.clipShadows,i.stencilWriteMask=this.stencilWriteMask,i.stencilFunc=this.stencilFunc,i.stencilRef=this.stencilRef,i.stencilFuncMask=this.stencilFuncMask,i.stencilFail=this.stencilFail,i.stencilZFail=this.stencilZFail,i.stencilZPass=this.stencilZPass,i.stencilWrite=this.stencilWrite,i.polygonOffset=this.polygonOffset,i.polygonOffsetFactor=this.polygonOffsetFactor,i.polygonOffsetUnits=this.polygonOffsetUnits,i.dithering=this.dithering,i.alphaTest=this.alphaTest,i.alphaHash=this.alphaHash,i.alphaToCoverage=this.alphaToCoverage,i.premultipliedAlpha=this.premultipliedAlpha,i.forceSinglePass=this.forceSinglePass,i.allowOverride=this.allowOverride,i.visible=this.visible,i.toneMapped=this.toneMapped,i.name=this.name,this.color&&this.color.isColor&&(i.color=this.color.getHex()),this.roughness!==void 0&&(i.roughness=this.roughness),this.metalness!==void 0&&(i.metalness=this.metalness),this.sheen!==void 0&&(i.sheen=this.sheen),this.sheenColor&&this.sheenColor.isColor&&(i.sheenColor=this.sheenColor.getHex()),this.sheenRoughness!==void 0&&(i.sheenRoughness=this.sheenRoughness),this.emissive&&this.emissive.isColor&&(i.emissive=this.emissive.getHex()),this.emissiveIntensity!==void 0&&(i.emissiveIntensity=this.emissiveIntensity),this.specular&&this.specular.isColor&&(i.specular=this.specular.getHex()),this.specularIntensity!==void 0&&(i.specularIntensity=this.specularIntensity),this.specularColor&&this.specularColor.isColor&&(i.specularColor=this.specularColor.getHex()),this.shininess!==void 0&&(i.shininess=this.shininess),this.clearcoat!==void 0&&(i.clearcoat=this.clearcoat),this.clearcoatRoughness!==void 0&&(i.clearcoatRoughness=this.clearcoatRoughness),this.diffuseRoughness!==void 0&&(i.diffuseRoughness=this.diffuseRoughness),this.diffuseRoughnessMap&&this.diffuseRoughnessMap.isTexture&&(i.diffuseRoughnessMap=this.diffuseRoughnessMap.toJSON(e).uuid),this.clearcoatMap&&this.clearcoatMap.isTexture&&(i.clearcoatMap=this.clearcoatMap.toJSON(e).uuid),this.clearcoatRoughnessMap&&this.clearcoatRoughnessMap.isTexture&&(i.clearcoatRoughnessMap=this.clearcoatRoughnessMap.toJSON(e).uuid),this.clearcoatNormalMap&&this.clearcoatNormalMap.isTexture&&(i.clearcoatNormalMap=this.clearcoatNormalMap.toJSON(e).uuid,i.clearcoatNormalScale=this.clearcoatNormalScale.toArray()),this.sheenColorMap&&this.sheenColorMap.isTexture&&(i.sheenColorMap=this.sheenColorMap.toJSON(e).uuid),this.sheenRoughnessMap&&this.sheenRoughnessMap.isTexture&&(i.sheenRoughnessMap=this.sheenRoughnessMap.toJSON(e).uuid),this.dispersion!==void 0&&(i.dispersion=this.dispersion),this.retroreflectivity!==void 0&&(i.retroreflectivity=this.retroreflectivity),this.iridescence!==void 0&&(i.iridescence=this.iridescence),this.iridescenceIOR!==void 0&&(i.iridescenceIOR=this.iridescenceIOR),this.iridescenceThicknessMinimum!==void 0&&(i.iridescenceThicknessMinimum=this.iridescenceThicknessMinimum),this.iridescenceThicknessMaximum!==void 0&&(i.iridescenceThicknessMaximum=this.iridescenceThicknessMaximum),this.iridescenceMap&&this.iridescenceMap.isTexture&&(i.iridescenceMap=this.iridescenceMap.toJSON(e).uuid),this.iridescenceThicknessMap&&this.iridescenceThicknessMap.isTexture&&(i.iridescenceThicknessMap=this.iridescenceThicknessMap.toJSON(e).uuid),this.anisotropy!==void 0&&(i.anisotropy=this.anisotropy),this.anisotropyRotation!==void 0&&(i.anisotropyRotation=this.anisotropyRotation),this.anisotropyMap&&this.anisotropyMap.isTexture&&(i.anisotropyMap=this.anisotropyMap.toJSON(e).uuid),this.map&&this.map.isTexture&&(i.map=this.map.toJSON(e).uuid),this.matcap&&this.matcap.isTexture&&(i.matcap=this.matcap.toJSON(e).uuid),this.alphaMap&&this.alphaMap.isTexture&&(i.alphaMap=this.alphaMap.toJSON(e).uuid),this.lightMap&&this.lightMap.isTexture&&(i.lightMap=this.lightMap.toJSON(e).uuid,i.lightMapIntensity=this.lightMapIntensity),this.aoMap&&this.aoMap.isTexture&&(i.aoMap=this.aoMap.toJSON(e).uuid,i.aoMapIntensity=this.aoMapIntensity),this.bumpMap&&this.bumpMap.isTexture&&(i.bumpMap=this.bumpMap.toJSON(e).uuid,i.bumpScale=this.bumpScale),this.normalMap&&this.normalMap.isTexture&&(i.normalMap=this.normalMap.toJSON(e).uuid,i.normalMapType=this.normalMapType,i.normalScale=this.normalScale.toArray()),this.displacementMap&&this.displacementMap.isTexture&&(i.displacementMap=this.displacementMap.toJSON(e).uuid,i.displacementScale=this.displacementScale,i.displacementBias=this.displacementBias),this.roughnessMap&&this.roughnessMap.isTexture&&(i.roughnessMap=this.roughnessMap.toJSON(e).uuid),this.metalnessMap&&this.metalnessMap.isTexture&&(i.metalnessMap=this.metalnessMap.toJSON(e).uuid),this.emissiveMap&&this.emissiveMap.isTexture&&(i.emissiveMap=this.emissiveMap.toJSON(e).uuid),this.specularMap&&this.specularMap.isTexture&&(i.specularMap=this.specularMap.toJSON(e).uuid),this.specularIntensityMap&&this.specularIntensityMap.isTexture&&(i.specularIntensityMap=this.specularIntensityMap.toJSON(e).uuid),this.specularColorMap&&this.specularColorMap.isTexture&&(i.specularColorMap=this.specularColorMap.toJSON(e).uuid),this.envMap&&this.envMap.isTexture&&(i.envMap=this.envMap.toJSON(e).uuid,this.combine!==void 0&&(i.combine=this.combine)),this.envMapRotation!==void 0&&(i.envMapRotation=this.envMapRotation.toArray()),this.envMapIntensity!==void 0&&(i.envMapIntensity=this.envMapIntensity),this.reflectivity!==void 0&&(i.reflectivity=this.reflectivity),this.refractionRatio!==void 0&&(i.refractionRatio=this.refractionRatio),this.gradientMap&&this.gradientMap.isTexture&&(i.gradientMap=this.gradientMap.toJSON(e).uuid),this.transmission!==void 0&&(i.transmission=this.transmission),this.transmissionMap&&this.transmissionMap.isTexture&&(i.transmissionMap=this.transmissionMap.toJSON(e).uuid),this.thickness!==void 0&&(i.thickness=this.thickness),this.thicknessMap&&this.thicknessMap.isTexture&&(i.thicknessMap=this.thicknessMap.toJSON(e).uuid),this.attenuationDistance!==void 0&&(i.attenuationDistance=this.attenuationDistance),this.attenuationColor!==void 0&&(i.attenuationColor=this.attenuationColor.getHex()),this.size!==void 0&&(i.size=this.size),this.sizeAttenuation!==void 0&&(i.sizeAttenuation=this.sizeAttenuation),Array.isArray(this.clippingPlanes)&&this.clippingPlanes.length>0&&(i.clippingPlanes=this.clippingPlanes.map(a=>a.toJSON())),this.rotation!==void 0&&(i.rotation=this.rotation),this.depthPacking!==void 0&&(i.depthPacking=this.depthPacking),this.linewidth!==void 0&&(i.linewidth=this.linewidth),this.linecap!==void 0&&(i.linecap=this.linecap),this.linejoin!==void 0&&(i.linejoin=this.linejoin),this.dashSize!==void 0&&(i.dashSize=this.dashSize),this.gapSize!==void 0&&(i.gapSize=this.gapSize),this.scale!==void 0&&(i.scale=this.scale),this.wireframe!==void 0&&(i.wireframe=this.wireframe),this.wireframeLinewidth!==void 0&&(i.wireframeLinewidth=this.wireframeLinewidth),this.wireframeLinecap!==void 0&&(i.wireframeLinecap=this.wireframeLinecap),this.wireframeLinejoin!==void 0&&(i.wireframeLinejoin=this.wireframeLinejoin),this.flatShading!==void 0&&(i.flatShading=this.flatShading),this.fog!==void 0&&(i.fog=this.fog),Object.keys(this.userData).length>0&&(i.userData=this.userData);function r(a){let o=[];for(let s in a){let n=a[s];delete n.metadata,o.push(n)}return o}if(t){let a=r(e.textures),o=r(e.images);a.length>0&&(i.textures=a),o.length>0&&(i.images=o)}return i}fromJSON(e,t){if(e.uuid!==void 0&&(this.uuid=e.uuid),e.name!==void 0&&(this.name=e.name),e.color!==void 0&&this.color!==void 0&&this.color.setHex(e.color),e.roughness!==void 0&&(this.roughness=e.roughness),e.metalness!==void 0&&(this.metalness=e.metalness),e.sheen!==void 0&&(this.sheen=e.sheen),e.sheenColor!==void 0&&(this.sheenColor=new dt().setHex(e.sheenColor)),e.sheenRoughness!==void 0&&(this.sheenRoughness=e.sheenRoughness),e.emissive!==void 0&&this.emissive!==void 0&&this.emissive.setHex(e.emissive),e.specular!==void 0&&this.specular!==void 0&&this.specular.setHex(e.specular),e.specularIntensity!==void 0&&(this.specularIntensity=e.specularIntensity),e.specularColor!==void 0&&this.specularColor!==void 0&&this.specularColor.setHex(e.specularColor),e.shininess!==void 0&&(this.shininess=e.shininess),e.clearcoat!==void 0&&(this.clearcoat=e.clearcoat),e.clearcoatRoughness!==void 0&&(this.clearcoatRoughness=e.clearcoatRoughness),e.diffuseRoughness!==void 0&&(this.diffuseRoughness=e.diffuseRoughness),e.dispersion!==void 0&&(this.dispersion=e.dispersion),e.retroreflectivity!==void 0&&(this.retroreflectivity=e.retroreflectivity),e.iridescence!==void 0&&(this.iridescence=e.iridescence),e.iridescenceIOR!==void 0&&(this.iridescenceIOR=e.iridescenceIOR),e.iridescenceThicknessMinimum!==void 0&&(this.iridescenceThicknessMinimum=e.iridescenceThicknessMinimum),e.iridescenceThicknessMaximum!==void 0&&(this.iridescenceThicknessMaximum=e.iridescenceThicknessMaximum),e.iridescenceThicknessRange!==void 0&&(this.iridescenceThicknessMinimum=e.iridescenceThicknessRange[0],this.iridescenceThicknessMaximum=e.iridescenceThicknessRange[1]),e.transmission!==void 0&&(this.transmission=e.transmission),e.thickness!==void 0&&(this.thickness=e.thickness),e.attenuationDistance!==void 0&&(this.attenuationDistance=e.attenuationDistance),e.attenuationColor!==void 0&&this.attenuationColor!==void 0&&this.attenuationColor.setHex(e.attenuationColor),e.anisotropy!==void 0&&(this.anisotropy=e.anisotropy),e.anisotropyRotation!==void 0&&(this.anisotropyRotation=e.anisotropyRotation),e.fog!==void 0&&(this.fog=e.fog),e.flatShading!==void 0&&(this.flatShading=e.flatShading),e.blending!==void 0&&(this.blending=e.blending),e.combine!==void 0&&(this.combine=e.combine),e.side!==void 0&&(this.side=e.side),e.shadowSide!==void 0&&(this.shadowSide=e.shadowSide),e.opacity!==void 0&&(this.opacity=e.opacity),e.transparent!==void 0&&(this.transparent=e.transparent),e.alphaTest!==void 0&&(this.alphaTest=e.alphaTest),e.alphaHash!==void 0&&(this.alphaHash=e.alphaHash),e.depthFunc!==void 0&&(this.depthFunc=e.depthFunc),e.depthTest!==void 0&&(this.depthTest=e.depthTest),e.depthWrite!==void 0&&(this.depthWrite=e.depthWrite),e.colorWrite!==void 0&&(this.colorWrite=e.colorWrite),e.clippingPlanes!==void 0&&(this.clippingPlanes=e.clippingPlanes.map(i=>new Or().fromJSON(i))),e.clipIntersection!==void 0&&(this.clipIntersection=e.clipIntersection),e.clipShadows!==void 0&&(this.clipShadows=e.clipShadows),e.depthPacking!==void 0&&(this.depthPacking=e.depthPacking),e.blendSrc!==void 0&&(this.blendSrc=e.blendSrc),e.blendDst!==void 0&&(this.blendDst=e.blendDst),e.blendEquation!==void 0&&(this.blendEquation=e.blendEquation),e.blendSrcAlpha!==void 0&&(this.blendSrcAlpha=e.blendSrcAlpha),e.blendDstAlpha!==void 0&&(this.blendDstAlpha=e.blendDstAlpha),e.blendEquationAlpha!==void 0&&(this.blendEquationAlpha=e.blendEquationAlpha),e.blendColor!==void 0&&this.blendColor!==void 0&&this.blendColor.setHex(e.blendColor),e.blendAlpha!==void 0&&(this.blendAlpha=e.blendAlpha),e.stencilWriteMask!==void 0&&(this.stencilWriteMask=e.stencilWriteMask),e.stencilFunc!==void 0&&(this.stencilFunc=e.stencilFunc),e.stencilRef!==void 0&&(this.stencilRef=e.stencilRef),e.stencilFuncMask!==void 0&&(this.stencilFuncMask=e.stencilFuncMask),e.stencilFail!==void 0&&(this.stencilFail=e.stencilFail),e.stencilZFail!==void 0&&(this.stencilZFail=e.stencilZFail),e.stencilZPass!==void 0&&(this.stencilZPass=e.stencilZPass),e.stencilWrite!==void 0&&(this.stencilWrite=e.stencilWrite),e.wireframe!==void 0&&(this.wireframe=e.wireframe),e.wireframeLinewidth!==void 0&&(this.wireframeLinewidth=e.wireframeLinewidth),e.wireframeLinecap!==void 0&&(this.wireframeLinecap=e.wireframeLinecap),e.wireframeLinejoin!==void 0&&(this.wireframeLinejoin=e.wireframeLinejoin),e.rotation!==void 0&&(this.rotation=e.rotation),e.linewidth!==void 0&&(this.linewidth=e.linewidth),e.linecap!==void 0&&(this.linecap=e.linecap),e.linejoin!==void 0&&(this.linejoin=e.linejoin),e.dashSize!==void 0&&(this.dashSize=e.dashSize),e.gapSize!==void 0&&(this.gapSize=e.gapSize),e.scale!==void 0&&(this.scale=e.scale),e.polygonOffset!==void 0&&(this.polygonOffset=e.polygonOffset),e.polygonOffsetFactor!==void 0&&(this.polygonOffsetFactor=e.polygonOffsetFactor),e.polygonOffsetUnits!==void 0&&(this.polygonOffsetUnits=e.polygonOffsetUnits),e.dithering!==void 0&&(this.dithering=e.dithering),e.alphaToCoverage!==void 0&&(this.alphaToCoverage=e.alphaToCoverage),e.premultipliedAlpha!==void 0&&(this.premultipliedAlpha=e.premultipliedAlpha),e.forceSinglePass!==void 0&&(this.forceSinglePass=e.forceSinglePass),e.allowOverride!==void 0&&(this.allowOverride=e.allowOverride),e.visible!==void 0&&(this.visible=e.visible),e.toneMapped!==void 0&&(this.toneMapped=e.toneMapped),e.userData!==void 0&&(this.userData=e.userData),e.vertexColors!==void 0&&(this.vertexColors=typeof e.vertexColors=="number"?e.vertexColors>0:e.vertexColors),e.size!==void 0&&(this.size=e.size),e.sizeAttenuation!==void 0&&(this.sizeAttenuation=e.sizeAttenuation),e.map!==void 0&&(this.map=t[e.map]||null),e.matcap!==void 0&&(this.matcap=t[e.matcap]||null),e.alphaMap!==void 0&&(this.alphaMap=t[e.alphaMap]||null),e.bumpMap!==void 0&&(this.bumpMap=t[e.bumpMap]||null),e.bumpScale!==void 0&&(this.bumpScale=e.bumpScale),e.normalMap!==void 0&&(this.normalMap=t[e.normalMap]||null),e.normalMapType!==void 0&&(this.normalMapType=e.normalMapType),e.normalScale!==void 0){let i=e.normalScale;Array.isArray(i)===!1&&(i=[i,i]),this.normalScale=new ge().fromArray(i)}return e.displacementMap!==void 0&&(this.displacementMap=t[e.displacementMap]||null),e.displacementScale!==void 0&&(this.displacementScale=e.displacementScale),e.displacementBias!==void 0&&(this.displacementBias=e.displacementBias),e.roughnessMap!==void 0&&(this.roughnessMap=t[e.roughnessMap]||null),e.metalnessMap!==void 0&&(this.metalnessMap=t[e.metalnessMap]||null),e.emissiveMap!==void 0&&(this.emissiveMap=t[e.emissiveMap]||null),e.emissiveIntensity!==void 0&&(this.emissiveIntensity=e.emissiveIntensity),e.specularMap!==void 0&&(this.specularMap=t[e.specularMap]||null),e.specularIntensityMap!==void 0&&(this.specularIntensityMap=t[e.specularIntensityMap]||null),e.specularColorMap!==void 0&&(this.specularColorMap=t[e.specularColorMap]||null),e.envMap!==void 0&&(this.envMap=t[e.envMap]||null),e.envMapRotation!==void 0&&this.envMapRotation.fromArray(e.envMapRotation),e.envMapIntensity!==void 0&&(this.envMapIntensity=e.envMapIntensity),e.reflectivity!==void 0&&(this.reflectivity=e.reflectivity),e.refractionRatio!==void 0&&(this.refractionRatio=e.refractionRatio),e.lightMap!==void 0&&(this.lightMap=t[e.lightMap]||null),e.lightMapIntensity!==void 0&&(this.lightMapIntensity=e.lightMapIntensity),e.aoMap!==void 0&&(this.aoMap=t[e.aoMap]||null),e.aoMapIntensity!==void 0&&(this.aoMapIntensity=e.aoMapIntensity),e.gradientMap!==void 0&&(this.gradientMap=t[e.gradientMap]||null),e.clearcoatMap!==void 0&&(this.clearcoatMap=t[e.clearcoatMap]||null),e.clearcoatRoughnessMap!==void 0&&(this.clearcoatRoughnessMap=t[e.clearcoatRoughnessMap]||null),e.clearcoatNormalMap!==void 0&&(this.clearcoatNormalMap=t[e.clearcoatNormalMap]||null),e.clearcoatNormalScale!==void 0&&(this.clearcoatNormalScale=new ge().fromArray(e.clearcoatNormalScale)),e.diffuseRoughnessMap!==void 0&&(this.diffuseRoughnessMap=t[e.diffuseRoughnessMap]||null),e.iridescenceMap!==void 0&&(this.iridescenceMap=t[e.iridescenceMap]||null),e.iridescenceThicknessMap!==void 0&&(this.iridescenceThicknessMap=t[e.iridescenceThicknessMap]||null),e.transmissionMap!==void 0&&(this.transmissionMap=t[e.transmissionMap]||null),e.thicknessMap!==void 0&&(this.thicknessMap=t[e.thicknessMap]||null),e.anisotropyMap!==void 0&&(this.anisotropyMap=t[e.anisotropyMap]||null),e.sheenColorMap!==void 0&&(this.sheenColorMap=t[e.sheenColorMap]||null),e.sheenRoughnessMap!==void 0&&(this.sheenRoughnessMap=t[e.sheenRoughnessMap]||null),this}clone(){return new this.constructor().copy(this)}copy(e){this.name=e.name,this.blending=e.blending,this.side=e.side,this.vertexColors=e.vertexColors,this.opacity=e.opacity,this.transparent=e.transparent,this.blendSrc=e.blendSrc,this.blendDst=e.blendDst,this.blendEquation=e.blendEquation,this.blendSrcAlpha=e.blendSrcAlpha,this.blendDstAlpha=e.blendDstAlpha,this.blendEquationAlpha=e.blendEquationAlpha,this.blendColor.copy(e.blendColor),this.blendAlpha=e.blendAlpha,this.depthFunc=e.depthFunc,this.depthTest=e.depthTest,this.depthWrite=e.depthWrite,this.stencilWriteMask=e.stencilWriteMask,this.stencilFunc=e.stencilFunc,this.stencilRef=e.stencilRef,this.stencilFuncMask=e.stencilFuncMask,this.stencilFail=e.stencilFail,this.stencilZFail=e.stencilZFail,this.stencilZPass=e.stencilZPass,this.stencilWrite=e.stencilWrite;let t=e.clippingPlanes,i=null;if(t!==null){let r=t.length;i=Array(r);for(let a=0;a!==r;++a)i[a]=t[a].clone()}return this.clippingPlanes=i,this.clipIntersection=e.clipIntersection,this.clipShadows=e.clipShadows,this.shadowSide=e.shadowSide,this.colorWrite=e.colorWrite,this.precision=e.precision,this.polygonOffset=e.polygonOffset,this.polygonOffsetFactor=e.polygonOffsetFactor,this.polygonOffsetUnits=e.polygonOffsetUnits,this.dithering=e.dithering,this.alphaTest=e.alphaTest,this.alphaHash=e.alphaHash,this.alphaToCoverage=e.alphaToCoverage,this.premultipliedAlpha=e.premultipliedAlpha,this.forceSinglePass=e.forceSinglePass,this.allowOverride=e.allowOverride,this.visible=e.visible,this.toneMapped=e.toneMapped,this.userData=JSON.parse(JSON.stringify(e.userData)),this}dispose(){this.dispatchEvent({type:"dispose"})}set needsUpdate(e){e===!0&&this.version++}},br=new I,Tn=new I,Wo=new I,Xo=new I,L0=class{constructor(e=new I,t=new I(0,0,-1)){this.origin=e,this.direction=t}set(e,t){return this.origin.copy(e),this.direction.copy(t),this}copy(e){return this.origin.copy(e.origin),this.direction.copy(e.direction),this}at(e,t){return t.copy(this.origin).addScaledVector(this.direction,e)}lookAt(e){return this.direction.copy(e).sub(this.origin).normalize(),this}recast(e){return this.origin.copy(this.at(e,br)),this}closestPointToPoint(e,t){t.subVectors(e,this.origin);let i=t.dot(this.direction);return i<0?t.copy(this.origin):t.copy(this.origin).addScaledVector(this.direction,i)}distanceToPoint(e){return Math.sqrt(this.distanceSqToPoint(e))}distanceSqToPoint(e){let t=br.subVectors(e,this.origin).dot(this.direction);return t<0?this.origin.distanceToSquared(e):(br.copy(this.origin).addScaledVector(this.direction,t),br.distanceToSquared(e))}distanceSqToSegment(e,t,i,r){Tn.copy(e).add(t).multiplyScalar(.5),Wo.copy(t).sub(e).normalize(),Xo.copy(this.origin).sub(Tn);let a=e.distanceTo(t)*.5,o=-this.direction.dot(Wo),s=Xo.dot(this.direction),n=-Xo.dot(Wo),l=Xo.lengthSq(),h=Math.abs(1-o*o),c,u,d,f;if(h>0)if(c=o*n-s,u=o*s-n,f=a*h,c>=0)if(u>=-f)if(u<=f){let p=1/h;c*=p,u*=p,d=c*(c+o*u+2*s)+u*(o*c+u+2*n)+l}else u=a,c=Math.max(0,-(o*u+s)),d=-c*c+u*(u+2*n)+l;else u=-a,c=Math.max(0,-(o*u+s)),d=-c*c+u*(u+2*n)+l;else u<=-f?(c=Math.max(0,-(-o*a+s)),u=c>0?-a:Math.min(Math.max(-a,-n),a),d=-c*c+u*(u+2*n)+l):u<=f?(c=0,u=Math.min(Math.max(-a,-n),a),d=u*(u+2*n)+l):(c=Math.max(0,-(o*a+s)),u=c>0?a:Math.min(Math.max(-a,-n),a),d=-c*c+u*(u+2*n)+l);else u=o>0?-a:a,c=Math.max(0,-(o*u+s)),d=-c*c+u*(u+2*n)+l;return i&&i.copy(this.origin).addScaledVector(this.direction,c),r&&r.copy(Tn).addScaledVector(Wo,u),d}intersectSphere(e,t){if(e.radius<0)return null;br.subVectors(e.center,this.origin);let i=br.dot(this.direction),r=br.dot(br)-i*i,a=e.radius*e.radius;if(r>a)return null;let o=Math.sqrt(a-r),s=i-o,n=i+o;return n<0?null:s<0?this.at(n,t):this.at(s,t)}intersectsSphere(e){return e.radius<0?!1:this.distanceSqToPoint(e.center)<=e.radius*e.radius}distanceToPlane(e){let t=e.normal.dot(this.direction);if(t===0)return e.distanceToPoint(this.origin)===0?0:null;let i=-(this.origin.dot(e.normal)+e.constant)/t;return i>=0?i:null}intersectPlane(e,t){let i=this.distanceToPlane(e);return i===null?null:this.at(i,t)}intersectsPlane(e){let t=e.distanceToPoint(this.origin);return t===0||e.normal.dot(this.direction)*t<0}intersectBox(e,t){let i,r,a,o,s,n,l=1/this.direction.x,h=1/this.direction.y,c=1/this.direction.z,u=this.origin;return l>=0?(i=(e.min.x-u.x)*l,r=(e.max.x-u.x)*l):(i=(e.max.x-u.x)*l,r=(e.min.x-u.x)*l),h>=0?(a=(e.min.y-u.y)*h,o=(e.max.y-u.y)*h):(a=(e.max.y-u.y)*h,o=(e.min.y-u.y)*h),i>o||a>r||((a>i||isNaN(i))&&(i=a),(o<r||isNaN(r))&&(r=o),c>=0?(s=(e.min.z-u.z)*c,n=(e.max.z-u.z)*c):(s=(e.max.z-u.z)*c,n=(e.min.z-u.z)*c),i>n||s>r)||((s>i||i!==i)&&(i=s),(n<r||r!==r)&&(r=n),r<0)?null:this.at(i>=0?i:r,t)}intersectsBox(e){return this.intersectBox(e,br)!==null}intersectTriangle(e,t,i,r,a){let o=this.origin,s=this.direction,n=s.x,l=s.y,h=s.z,c=e.x-o.x,u=e.y-o.y,d=e.z-o.z,f=t.x-o.x,p=t.y-o.y,x=t.z-o.z,g=i.x-o.x,m=i.y-o.y,y=i.z-o.z,M=Math.abs(n),b=Math.abs(l),E=Math.abs(h),T,v,_,A,C,R,L,G,P,z,H,D;if(M>=b&&M>=E?(_=n,R=c,P=f,D=g,n>=0?(T=l,v=h,A=u,C=d,L=p,G=x,z=m,H=y):(T=h,v=l,A=d,C=u,L=x,G=p,z=y,H=m)):b>=E?(_=l,R=u,P=p,D=m,l>=0?(T=h,v=n,A=d,C=c,L=x,G=f,z=y,H=g):(T=n,v=h,A=c,C=d,L=f,G=x,z=g,H=y)):(_=h,R=d,P=x,D=y,h>=0?(T=n,v=l,A=c,C=u,L=f,G=p,z=g,H=m):(T=l,v=n,A=u,C=c,L=p,G=f,z=m,H=g)),_===0)return null;let ae=T/_,j=v/_,Y=1/_,ie=A-ae*R,Se=C-j*R,K=L-ae*P,re=G-j*P,xe=z-ae*D,Ne=H-j*D,Me=xe*re-Ne*K,Q=ie*Ne-Se*xe,ke=K*Se-re*ie;if(r){if(Me<0||Q<0||ke<0)return null}else if((Me<0||Q<0||ke<0)&&(Me>0||Q>0||ke>0))return null;let $e=Me+Q+ke;if($e===0)return null;let De=Y*(Me*R+Q*P+ke*D);return($e>0?De<0:De>0)?null:this.at(De/$e,a)}applyMatrix4(e){return this.origin.applyMatrix4(e),this.direction.transformDirection(e),this}equals(e){return e.origin.equals(this.origin)&&e.direction.equals(this.direction)}clone(){return new this.constructor().copy(this)}},N0=class extends Zs{constructor(e){super(),this.isMeshBasicMaterial=!0,this.type="MeshBasicMaterial",this.color=new dt(16777215),this.map=null,this.lightMap=null,this.lightMapIntensity=1,this.aoMap=null,this.aoMapIntensity=1,this.specularMap=null,this.alphaMap=null,this.envMap=null,this.envMapRotation=new Cr,this.combine=0,this.reflectivity=1,this.refractionRatio=.98,this.wireframe=!1,this.wireframeLinewidth=1,this.wireframeLinecap="round",this.wireframeLinejoin="round",this.fog=!0,this.setValues(e)}copy(e){return super.copy(e),this.color.copy(e.color),this.map=e.map,this.lightMap=e.lightMap,this.lightMapIntensity=e.lightMapIntensity,this.aoMap=e.aoMap,this.aoMapIntensity=e.aoMapIntensity,this.specularMap=e.specularMap,this.alphaMap=e.alphaMap,this.envMap=e.envMap,this.envMapRotation.copy(e.envMapRotation),this.combine=e.combine,this.reflectivity=e.reflectivity,this.refractionRatio=e.refractionRatio,this.wireframe=e.wireframe,this.wireframeLinewidth=e.wireframeLinewidth,this.wireframeLinecap=e.wireframeLinecap,this.wireframeLinejoin=e.wireframeLinejoin,this.fog=e.fog,this}},nh=new gt,Yr=new L0,qo=new Lo,lh=new I,jo=new I,Yo=new I,Zo=new I,En=new I,Jo=new I,hh=new I,Ko=new I,Nt=class extends Ar{constructor(e=new ti,t=new N0){super(),this.isMesh=!0,this.type="Mesh",this.geometry=e,this.material=t,this.morphTargetDictionary=void 0,this.morphTargetInfluences=void 0,this.count=1,this.updateMorphTargets()}copy(e,t){return super.copy(e,t),this.morphTargetInfluences=e.morphTargetInfluences===void 0?void 0:e.morphTargetInfluences.slice(),this.morphTargetDictionary=e.morphTargetDictionary===void 0?void 0:Object.assign({},e.morphTargetDictionary),this.material=Array.isArray(e.material)?e.material.slice():e.material,this.geometry=e.geometry,this}updateMorphTargets(){let e=this.geometry.morphAttributes,t=Object.keys(e);if(t.length>0){let i=e[t[0]];if(i!==void 0){this.morphTargetInfluences=[],this.morphTargetDictionary={};for(let r=0,a=i.length;r<a;r++){let o=i[r].name||String(r);this.morphTargetInfluences.push(0),this.morphTargetDictionary[o]=r}}}}getVertexPosition(e,t){let i=this.geometry,r=i.attributes.position,a=i.morphAttributes.position,o=i.morphTargetsRelative;t.fromBufferAttribute(r,e);let s=this.morphTargetInfluences;if(a&&s){Jo.set(0,0,0);for(let n=0,l=a.length;n<l;n++){let h=s[n],c=a[n];h!==0&&(En.fromBufferAttribute(c,e),o?Jo.addScaledVector(En,h):Jo.addScaledVector(En.sub(t),h))}t.add(Jo)}return t}intersectsFrustum(e){return e.intersectsObject(this)}raycast(e,t){let i=this.geometry,r=this.material,a=this.matrixWorld;r!==void 0&&(i.boundingSphere===null&&i.computeBoundingSphere(),qo.copy(i.boundingSphere),qo.applyMatrix4(a),Yr.copy(e.ray).recast(e.near),qo.containsPoint(Yr.origin)===!1&&(Yr.intersectSphere(qo,lh)===null||Yr.origin.distanceToSquared(lh)>(e.far-e.near)**2)||(nh.copy(a).invert(),Yr.copy(e.ray).applyMatrix4(nh),(i.boundingBox===null||Yr.intersectsBox(i.boundingBox)!==!1)&&this._computeIntersections(e,t,Yr)))}_computeIntersections(e,t,i){let r,a=this.geometry,o=this.material,s=a.index,n=a.attributes.position,l=a.attributes.uv,h=a.attributes.uv1,c=a.attributes.normal,u=a.groups,d=a.drawRange;if(s!==null)if(Array.isArray(o))for(let f=0,p=u.length;f<p;f++){let x=u[f],g=o[x.materialIndex],m=Math.max(x.start,d.start),y=Math.min(s.count,Math.min(x.start+x.count,d.start+d.count));for(let M=m,b=y;M<b;M+=3){let E=s.getX(M),T=s.getX(M+1),v=s.getX(M+2);r=$o(this,g,e,i,l,h,c,E,T,v),r&&(r.faceIndex=Math.floor(M/3),r.face.materialIndex=x.materialIndex,t.push(r))}}else{let f=Math.max(0,d.start),p=Math.min(s.count,d.start+d.count);for(let x=f,g=p;x<g;x+=3){let m=s.getX(x),y=s.getX(x+1),M=s.getX(x+2);r=$o(this,o,e,i,l,h,c,m,y,M),r&&(r.faceIndex=Math.floor(x/3),t.push(r))}}else if(n!==void 0)if(Array.isArray(o))for(let f=0,p=u.length;f<p;f++){let x=u[f],g=o[x.materialIndex],m=Math.max(x.start,d.start),y=Math.min(n.count,Math.min(x.start+x.count,d.start+d.count));for(let M=m,b=y;M<b;M+=3){let E=M,T=M+1,v=M+2;r=$o(this,g,e,i,l,h,c,E,T,v),r&&(r.faceIndex=Math.floor(M/3),r.face.materialIndex=x.materialIndex,t.push(r))}}else{let f=Math.max(0,d.start),p=Math.min(n.count,d.start+d.count);for(let x=f,g=p;x<g;x+=3){let m=x,y=x+1,M=x+2;r=$o(this,o,e,i,l,h,c,m,y,M),r&&(r.faceIndex=Math.floor(x/3),t.push(r))}}}};function I0(e,t,i,r,a,o,s,n){let l;if(l=t.side===1?r.intersectTriangle(s,o,a,!0,n):r.intersectTriangle(a,o,s,t.side===0,n),l===null)return null;Ko.copy(n),Ko.applyMatrix4(e.matrixWorld);let h=i.ray.origin.distanceTo(Ko);return h<i.near||h>i.far?null:{distance:h,point:Ko.clone(),object:e}}function $o(e,t,i,r,a,o,s,n,l,h){e.getVertexPosition(n,jo),e.getVertexPosition(l,Yo),e.getVertexPosition(h,Zo);let c=I0(e,t,i,r,jo,Yo,Zo,hh);if(c){let u=new I;Ja.getBarycoord(hh,jo,Yo,Zo,u),a&&(c.uv=Ja.getInterpolatedAttribute(a,n,l,h,u,new ge)),o&&(c.uv1=Ja.getInterpolatedAttribute(o,n,l,h,u,new ge)),s&&(c.normal=Ja.getInterpolatedAttribute(s,n,l,h,u,new I),c.normal.dot(r.direction)>0&&c.normal.multiplyScalar(-1));let d={a:n,b:l,c:h,normal:new I,materialIndex:0};Ja.getNormal(jo,Yo,Zo,d.normal),c.face=d,c.barycoord=u}return c}var No=class extends zi{constructor(e=null,t=1,i=1,r,a,o,s,n,l=oi,h=oi,c,u){super(null,o,s,n,l,h,r,a,c,u),this.isDataTexture=!0,this.image={data:e,width:t,height:i},this.generateMipmaps=!1,this.flipY=!1,this.unpackAlignment=1}},ch=class extends si{constructor(e,t,i,r=1){super(e,t,i),this.isInstancedBufferAttribute=!0,this.meshPerAttribute=r}copy(e){return super.copy(e),this.meshPerAttribute=e.meshPerAttribute,this}toJSON(){let e=super.toJSON();return e.meshPerAttribute=this.meshPerAttribute,e.isInstancedBufferAttribute=!0,e}},_a=new gt,uh=new gt,Qo=[],dh=new la,D0=new gt,eo=new Nt,to=new Lo,Ri=class extends Nt{constructor(e,t,i){super(e,t),this.isInstancedMesh=!0,this.instanceMatrix=new ch(new Float32Array(i*16),16),this.instanceColor=null,this.morphTexture=null,this.count=i,this.boundingBox=null,this.boundingSphere=null;for(let r=0;r<i;r++)this.setMatrixAt(r,D0)}computeBoundingBox(){let e=this.geometry,t=this.count;this.boundingBox===null&&(this.boundingBox=new la),e.boundingBox===null&&e.computeBoundingBox(),this.boundingBox.makeEmpty();for(let i=0;i<t;i++)this.getMatrixAt(i,_a),dh.copy(e.boundingBox).applyMatrix4(_a),this.boundingBox.union(dh)}computeBoundingSphere(){let e=this.geometry,t=this.count;this.boundingSphere===null&&(this.boundingSphere=new Lo),e.boundingSphere===null&&e.computeBoundingSphere(),this.boundingSphere.makeEmpty();for(let i=0;i<t;i++)this.getMatrixAt(i,_a),to.copy(e.boundingSphere).applyMatrix4(_a),this.boundingSphere.union(to)}copy(e,t){return super.copy(e,t),this.instanceMatrix.copy(e.instanceMatrix),this.morphTexture=e.morphTexture===null?null:e.morphTexture.clone(),this.instanceColor=e.instanceColor===null?null:e.instanceColor.clone(),this.count=e.count,this.boundingBox=e.boundingBox===null?null:e.boundingBox.clone(),this.boundingSphere=e.boundingSphere===null?null:e.boundingSphere.clone(),this}getColorAt(e,t){return this.instanceColor===null?t.setRGB(1,1,1):t.fromArray(this.instanceColor.array,e*3)}getMatrixAt(e,t){return t.fromArray(this.instanceMatrix.array,e*16)}getMorphAt(e,t){let i=t.morphTargetInfluences,r=this.morphTexture.source.data.data,a=e*(i.length+1)+1;for(let o=0;o<i.length;o++)i[o]=r[a+o]}raycast(e,t){let i=this.matrixWorld,r=this.count;if(eo.geometry=this.geometry,eo.material=this.material,eo.material!==void 0&&(this.boundingSphere===null&&this.computeBoundingSphere(),to.copy(this.boundingSphere),to.applyMatrix4(i),e.ray.intersectsSphere(to)!==!1))for(let a=0;a<r;a++){this.getMatrixAt(a,_a),uh.multiplyMatrices(i,_a),eo.matrixWorld=uh,eo.raycast(e,Qo);for(let o=0,s=Qo.length;o<s;o++){let n=Qo[o];n.instanceId=a,n.object=this,t.push(n)}Qo.length=0}}setColorAt(e,t){return this.instanceColor===null&&(this.instanceColor=new ch(new Float32Array(this.instanceMatrix.count*3).fill(1),3)),t.toArray(this.instanceColor.array,e*3),this}setMatrixAt(e,t){return t.toArray(this.instanceMatrix.array,e*16),this}setMorphAt(e,t){let i=t.morphTargetInfluences,r=i.length+1;this.morphTexture===null&&(this.morphTexture=new No(new Float32Array(r*this.count),r,this.count,Ys,Co));let a=this.morphTexture.source.data.data,o=0;for(let l=0;l<i.length;l++)o+=i[l];let s=this.geometry.morphTargetsRelative?1:1-o,n=r*e;return a[n]=s,a.set(i,n+1),this}updateMorphTargets(){}dispose(){super.dispose(),this.instanceMatrix.dispose(),this.instanceColor!==null&&this.instanceColor.dispose(),this.morphTexture!==null&&(this.morphTexture.dispose(),this.morphTexture=null)}},Zr=new Lo,U0=new ge(.5,.5),es=new I,Os=class{constructor(e=new Or,t=new Or,i=new Or,r=new Or,a=new Or,o=new Or){this.planes=[e,t,i,r,a,o]}set(e,t,i,r,a,o){let s=this.planes;return s[0].copy(e),s[1].copy(t),s[2].copy(i),s[3].copy(r),s[4].copy(a),s[5].copy(o),this}copy(e){let t=this.planes;for(let i=0;i<6;i++)t[i].copy(e.planes[i]);return this}setFromProjectionMatrix(e,t=ia,i=!1){let r=this.planes,a=e.elements,o=a[0],s=a[1],n=a[2],l=a[3],h=a[4],c=a[5],u=a[6],d=a[7],f=a[8],p=a[9],x=a[10],g=a[11],m=a[12],y=a[13],M=a[14],b=a[15];if(r[0].setComponents(l-o,d-h,g-f,b-m).normalize(),r[1].setComponents(l+o,d+h,g+f,b+m).normalize(),r[2].setComponents(l+s,d+c,g+p,b+y).normalize(),r[3].setComponents(l-s,d-c,g-p,b-y).normalize(),i)r[4].setComponents(n,u,x,M).normalize(),r[5].setComponents(l-n,d-u,g-x,b-M).normalize();else if(r[4].setComponents(l-n,d-u,g-x,b-M).normalize(),t===2e3)r[5].setComponents(l+n,d+u,g+x,b+M).normalize();else if(t===2001)r[5].setComponents(n,u,x,M).normalize();else throw Error("THREE.Frustum.setFromProjectionMatrix(): Invalid coordinate system: "+t);return this}intersectsObject(e){if(e.boundingSphere!==void 0)e.boundingSphere===null&&e.computeBoundingSphere(),Zr.copy(e.boundingSphere).applyMatrix4(e.matrixWorld);else{let t=e.geometry;t.boundingSphere===null&&t.computeBoundingSphere(),Zr.copy(t.boundingSphere).applyMatrix4(e.matrixWorld)}return this.intersectsSphere(Zr)}intersectsSprite(e){return Zr.center.set(0,0,0),Zr.radius=.7071067811865476+U0.distanceTo(e.center),Zr.applyMatrix4(e.matrixWorld),this.intersectsSphere(Zr)}intersectsSphere(e){let t=this.planes,i=e.center,r=-e.radius;for(let a=0;a<6;a++)if(t[a].distanceToPoint(i)<r)return!1;return!0}intersectsBox(e){let t=this.planes;for(let i=0;i<6;i++){let r=t[i];if(es.x=r.normal.x>0?e.max.x:e.min.x,es.y=r.normal.y>0?e.max.y:e.min.y,es.z=r.normal.z>0?e.max.z:e.min.z,r.distanceToPoint(es)<0)return!1}return!0}containsPoint(e){let t=this.planes;for(let i=0;i<6;i++)if(t[i].distanceToPoint(e)<0)return!1;return!0}clone(){return new this.constructor().copy(this)}},fh=new gt,O0=class Yc{constructor(){this.coordinateSystem=ia,this._frustums=[],this._count=0}setFromArrayCamera(t){let i=t.cameras,r=this._frustums;for(let a=0;a<i.length;a++){let o=i[a];fh.multiplyMatrices(o.projectionMatrix,o.matrixWorldInverse),r[a]===void 0&&(r[a]=new Os),r[a].setFromProjectionMatrix(fh,o.coordinateSystem,o.reversedDepth)}return this._count=i.length,this}getFrustum(t){return this._frustums[t]}intersectsObject(t){let i=this._frustums;for(let r=0;r<this._count;r++)if(i[r].intersectsObject(t))return!0;return!1}intersectsSprite(t){let i=this._frustums;for(let r=0;r<this._count;r++)if(i[r].intersectsSprite(t))return!0;return!1}intersectsSphere(t){let i=this._frustums;for(let r=0;r<this._count;r++)if(i[r].intersectsSphere(t))return!0;return!1}intersectsBox(t){let i=this._frustums;for(let r=0;r<this._count;r++)if(i[r].intersectsBox(t))return!0;return!1}containsPoint(t){let i=this._frustums;for(let r=0;r<this._count;r++)if(i[r].containsPoint(t))return!0;return!1}copy(t){this.coordinateSystem=t.coordinateSystem;let i=this._frustums,r=t._frustums;for(let a=0;a<t._count;a++)i[a]===void 0&&(i[a]=new Os),i[a].copy(r[a]);return this._count=t._count,this}clone(){return new Yc().copy(this)}},Zc=class extends zi{constructor(e=[],t=301,i,r,a,o,s,n,l,h){super(e,t,i,r,a,o,s,n,l,h),this.isCubeTexture=!0,this.flipY=!1}get images(){return this.image}set images(e){this.image=e}},z0=class extends zi{constructor(e,t,i,r,a,o,s,n,l){super(e,t,i,r,a,o,s,n,l),this.isCanvasTexture=!0,this.needsUpdate=!0}},ra=class extends zi{constructor(e,t,i=Vr,r,a,o,s=oi,n=oi,l,h=bo,c=1){if(h!==1026&&h!==1027)throw Error("THREE.DepthTexture: format must be either THREE.DepthFormat or THREE.DepthStencilFormat");super({width:e,height:t,depth:c},r,a,o,s,n,h,i,l),this.isDepthTexture=!0,this.flipY=!1,this.generateMipmaps=!1,this.compareFunction=null}copy(e){return super.copy(e),this.source=new _l(Object.assign({},e.image)),this.compareFunction=e.compareFunction,this}toJSON(e){let t=super.toJSON(e);return t.compareFunction=this.compareFunction,t}},Jc=class extends ra{constructor(e,t=Vr,i=301,r,a,o=oi,s=oi,n,l=bo){let h={width:e,height:e,depth:1},c=[h,h,h,h,h,h];super(e,e,t,i,r,a,o,s,n,l),this.image=c,this.isCubeDepthTexture=!0,this.isCubeTexture=!0}get images(){return this.image}set images(e){this.image=e}},Kc=class extends zi{constructor(e=null){super(),this.sourceTexture=e,this.isExternalTexture=!0}copy(e){return super.copy(e),this.sourceTexture=e.sourceTexture,this}},wr=class $c extends ti{constructor(t=1,i=1,r=1,a=1,o=1,s=1){super(),this.type="BoxGeometry",this.parameters={width:t,height:i,depth:r,widthSegments:a,heightSegments:o,depthSegments:s};let n=this;a=Math.floor(a),o=Math.floor(o),s=Math.floor(s);let l=[],h=[],c=[],u=[],d=0,f=0;p("z","y","x",-1,-1,r,i,t,s,o,0),p("z","y","x",1,-1,r,i,-t,s,o,1),p("x","z","y",1,1,t,r,i,a,s,2),p("x","z","y",1,-1,t,r,-i,a,s,3),p("x","y","z",1,-1,t,i,r,a,o,4),p("x","y","z",-1,-1,t,i,-r,a,o,5),this.setIndex(l),this.setAttribute("position",new wt(h,3)),this.setAttribute("normal",new wt(c,3)),this.setAttribute("uv",new wt(u,2));function p(x,g,m,y,M,b,E,T,v,_,A){let C=b/v,R=E/_,L=b/2,G=E/2,P=T/2,z=v+1,H=_+1,D=0,ae=0,j=new I;for(let Y=0;Y<H;Y++){let ie=Y*R-G;for(let Se=0;Se<z;Se++)j[x]=(Se*C-L)*y,j[g]=ie*M,j[m]=P,h.push(j.x,j.y,j.z),j[x]=0,j[g]=0,j[m]=T>0?1:-1,c.push(j.x,j.y,j.z),u.push(Se/v),u.push(1-Y/_),D+=1}for(let Y=0;Y<_;Y++)for(let ie=0;ie<v;ie++){let Se=d+ie+z*Y,K=d+ie+z*(Y+1),re=d+(ie+1)+z*(Y+1),xe=d+(ie+1)+z*Y;l.push(Se,K,xe),l.push(K,re,xe),ae+=6}n.addGroup(f,ae,A),f+=ae,d+=D}}copy(t){return super.copy(t),this.parameters=Object.assign({},t.parameters),this}static fromJSON(t){return new $c(t.width,t.height,t.depth,t.widthSegments,t.heightSegments,t.depthSegments)}},F0=class Qc extends ti{constructor(t=1,i=32,r=0,a=Math.PI*2){super(),this.type="CircleGeometry",this.parameters={radius:t,segments:i,thetaStart:r,thetaLength:a},i=Math.max(3,i);let o=[],s=[],n=[],l=[],h=new I,c=new ge;s.push(0,0,0),n.push(0,0,1),l.push(.5,.5);for(let u=0,d=3;u<=i;u++,d+=3){let f=r+u/i*a;h.x=t*Math.cos(f),h.y=t*Math.sin(f),s.push(h.x,h.y,h.z),n.push(0,0,1),c.x=(s[d]/t+1)/2,c.y=(s[d+1]/t+1)/2,l.push(c.x,c.y)}for(let u=1;u<=i;u++)o.push(u,u+1,0);this.setIndex(o),this.setAttribute("position",new wt(s,3)),this.setAttribute("normal",new wt(n,3)),this.setAttribute("uv",new wt(l,2))}copy(t){return super.copy(t),this.parameters=Object.assign({},t.parameters),this}static fromJSON(t){return new Qc(t.radius,t.segments,t.thetaStart,t.thetaLength)}},eu=class tu extends ti{constructor(t=1,i=1,r=1,a=32,o=1,s=!1,n=0,l=Math.PI*2){super(),this.type="CylinderGeometry",this.parameters={radiusTop:t,radiusBottom:i,height:r,radialSegments:a,heightSegments:o,openEnded:s,thetaStart:n,thetaLength:l};let h=this;a=Math.floor(a),o=Math.floor(o);let c=[],u=[],d=[],f=[],p=0,x=[],g=r/2,m=0;y(),s===!1&&(t>0&&M(!0),i>0&&M(!1)),this.setIndex(c),this.setAttribute("position",new wt(u,3)),this.setAttribute("normal",new wt(d,3)),this.setAttribute("uv",new wt(f,2));function y(){let b=new I,E=new I,T=0,v=(i-t)/r;for(let _=0;_<=o;_++){let A=[],C=_/o,R=C*(i-t)+t;for(let L=0;L<=a;L++){let G=L/a,P=G*l+n,z=Math.sin(P),H=Math.cos(P);E.x=R*z,E.y=-C*r+g,E.z=R*H,u.push(E.x,E.y,E.z),b.set(z,v,H).normalize(),d.push(b.x,b.y,b.z),f.push(G,1-C),A.push(p++)}x.push(A)}for(let _=0;_<a;_++)for(let A=0;A<o;A++){let C=x[A][_],R=x[A+1][_],L=x[A+1][_+1],G=x[A][_+1];(t>0||A!==0)&&(c.push(C,R,G),T+=3),(i>0||A!==o-1)&&(c.push(R,L,G),T+=3)}h.addGroup(m,T,0),m+=T}function M(b){let E=p,T=new ge,v=new I,_=0,A=b===!0?t:i,C=b===!0?1:-1;for(let L=1;L<=a;L++)u.push(0,g*C,0),d.push(0,C,0),f.push(.5,.5),p++;let R=p;for(let L=0;L<=a;L++){let G=L/a*l+n,P=Math.cos(G),z=Math.sin(G);v.x=A*z,v.y=g*C,v.z=A*P,u.push(v.x,v.y,v.z),d.push(0,C,0),T.x=P*.5+.5,T.y=z*.5*C+.5,f.push(T.x,T.y),p++}for(let L=0;L<a;L++){let G=E+L,P=R+L;b===!0?c.push(P,P+1,G):c.push(P+1,P,G),_+=3}h.addGroup(m,_,b===!0?1:2),m+=_}}copy(t){return super.copy(t),this.parameters=Object.assign({},t.parameters),this}static fromJSON(t){return new tu(t.radiusTop,t.radiusBottom,t.height,t.radialSegments,t.heightSegments,t.openEnded,t.thetaStart,t.thetaLength)}},B0=class iu extends eu{constructor(t=1,i=1,r=32,a=1,o=!1,s=0,n=Math.PI*2){super(0,t,i,r,a,o,s,n),this.type="ConeGeometry",this.parameters={radius:t,height:i,radialSegments:r,heightSegments:a,openEnded:o,thetaStart:s,thetaLength:n}}static fromJSON(t){return new iu(t.radius,t.height,t.radialSegments,t.heightSegments,t.openEnded,t.thetaStart,t.thetaLength)}},k0=class ru extends ti{constructor(t=[],i=[],r=1,a=0){super(),this.type="PolyhedronGeometry",this.parameters={vertices:t,indices:i,radius:r,detail:a};let o=[],s=[];n(a),h(r),c(),this.setAttribute("position",new wt(o,3)),this.setAttribute("normal",new wt(o.slice(),3)),this.setAttribute("uv",new wt(s,2)),a===0?this.computeVertexNormals():this.normalizeNormals();function n(y){let M=new I,b=new I,E=new I;for(let T=0;T<i.length;T+=3)f(i[T+0],M),f(i[T+1],b),f(i[T+2],E),l(M,b,E,y)}function l(y,M,b,E){let T=E+1,v=[];for(let _=0;_<=T;_++){v[_]=[];let A=y.clone().lerp(b,_/T),C=M.clone().lerp(b,_/T),R=T-_;for(let L=0;L<=R;L++)L===0&&_===T?v[_][L]=A:v[_][L]=A.clone().lerp(C,L/R)}for(let _=0;_<T;_++)for(let A=0;A<2*(T-_)-1;A++){let C=Math.floor(A/2);A%2==0?(d(v[_][C+1]),d(v[_+1][C]),d(v[_][C])):(d(v[_][C+1]),d(v[_+1][C+1]),d(v[_+1][C]))}}function h(y){let M=new I;for(let b=0;b<o.length;b+=3)M.x=o[b+0],M.y=o[b+1],M.z=o[b+2],M.normalize().multiplyScalar(y),o[b+0]=M.x,o[b+1]=M.y,o[b+2]=M.z}function c(){let y=new I;for(let M=0;M<o.length;M+=3){y.x=o[M+0],y.y=o[M+1],y.z=o[M+2];let b=g(y)/2/Math.PI+.5,E=m(y)/Math.PI+.5;s.push(b,1-E)}p(),u()}function u(){for(let y=0;y<s.length;y+=6){let M=s[y+0],b=s[y+2],E=s[y+4];Math.max(M,b,E)>.9&&Math.min(M,b,E)<.1&&(M<.2&&(s[y+0]+=1),b<.2&&(s[y+2]+=1),E<.2&&(s[y+4]+=1))}}function d(y){o.push(y.x,y.y,y.z)}function f(y,M){let b=y*3;M.x=t[b+0],M.y=t[b+1],M.z=t[b+2]}function p(){let y=new I,M=new I,b=new I,E=new I,T=new ge,v=new ge,_=new ge;for(let A=0,C=0;A<o.length;A+=9,C+=6){y.set(o[A+0],o[A+1],o[A+2]),M.set(o[A+3],o[A+4],o[A+5]),b.set(o[A+6],o[A+7],o[A+8]),T.set(s[C+0],s[C+1]),v.set(s[C+2],s[C+3]),_.set(s[C+4],s[C+5]),E.copy(y).add(M).add(b).divideScalar(3);let R=g(E);x(T,C+0,y,R),x(v,C+2,M,R),x(_,C+4,b,R)}}function x(y,M,b,E){E<0&&y.x===1&&(s[M]=y.x-1),b.x===0&&b.z===0&&(s[M]=E/2/Math.PI+.5)}function g(y){return Math.atan2(y.z,-y.x)}function m(y){return Math.atan2(-y.y,Math.sqrt(y.x*y.x+y.z*y.z))}}copy(t){return super.copy(t),this.parameters=Object.assign({},t.parameters),this}static fromJSON(t){return new ru(t.vertices,t.indices,t.radius,t.detail)}},mr=class{constructor(){this.type="Curve",this.arcLengthDivisions=200,this.needsUpdate=!1,this.cacheArcLengths=null}getPoint(){Je("Curve: .getPoint() not implemented.")}getPointAt(e,t){let i=this.getUtoTmapping(e);return this.getPoint(i,t)}getPoints(e=5){let t=[];for(let i=0;i<=e;i++)t.push(this.getPoint(i/e));return t}getSpacedPoints(e=5){let t=[];for(let i=0;i<=e;i++)t.push(this.getPointAt(i/e));return t}getLength(){let e=this.getLengths();return e[e.length-1]}getLengths(e=this.arcLengthDivisions){if(this.cacheArcLengths&&this.cacheArcLengths.length===e+1&&!this.needsUpdate)return this.cacheArcLengths;this.needsUpdate=!1;let t=[],i,r=this.getPoint(0),a=0;t.push(0);for(let o=1;o<=e;o++)i=this.getPoint(o/e),a+=i.distanceTo(r),t.push(a),r=i;return this.cacheArcLengths=t,t}updateArcLengths(){this.needsUpdate=!0,this.getLengths()}getUtoTmapping(e,t=null){let i=this.getLengths(),r=0,a=i.length,o;o=t||e*i[a-1];let s=0,n=a-1,l;for(;s<=n;)if(r=Math.floor(s+(n-s)/2),l=i[r]-o,l<0)s=r+1;else if(l>0)n=r-1;else{n=r;break}if(r=n,i[r]===o)return r/(a-1);let h=i[r],c=i[r+1]-h,u=(o-h)/c;return(r+u)/(a-1)}getTangent(e,t){let i=1e-4,r=e-i,a=e+i;r<0&&(r=0),a>1&&(a=1);let o=this.getPoint(r),s=this.getPoint(a),n=t||(o.isVector2?new ge:new I);return n.copy(s).sub(o).normalize(),n}getTangentAt(e,t){let i=this.getUtoTmapping(e);return this.getTangent(i,t)}computeFrenetFrames(e,t=!1){let i=new I,r=[],a=[],o=[],s=new I,n=new gt;for(let d=0;d<=e;d++){let f=d/e;r[d]=this.getTangentAt(f,new I)}a[0]=new I,o[0]=new I;let l=Number.MAX_VALUE,h=Math.abs(r[0].x),c=Math.abs(r[0].y),u=Math.abs(r[0].z);h<=l&&(l=h,i.set(1,0,0)),c<=l&&(l=c,i.set(0,1,0)),u<=l&&i.set(0,0,1),s.crossVectors(r[0],i).normalize(),a[0].crossVectors(r[0],s),o[0].crossVectors(r[0],a[0]);for(let d=1;d<=e;d++){if(a[d]=a[d-1].clone(),o[d]=o[d-1].clone(),s.crossVectors(r[d-1],r[d]),s.length()>2**-52){s.normalize();let f=Math.acos(ct(r[d-1].dot(r[d]),-1,1));a[d].applyMatrix4(n.makeRotationAxis(s,f))}o[d].crossVectors(r[d],a[d])}if(t===!0){let d=Math.acos(ct(a[0].dot(a[e]),-1,1));d/=e,r[0].dot(s.crossVectors(a[0],a[e]))>0&&(d=-d);for(let f=1;f<=e;f++)a[f].applyMatrix4(n.makeRotationAxis(r[f],d*f)),o[f].crossVectors(r[f],a[f])}return{tangents:r,normals:a,binormals:o}}clone(){return new this.constructor().copy(this)}copy(e){return this.arcLengthDivisions=e.arcLengthDivisions,this.cacheArcLengths=null,this}toJSON(){let e={metadata:{version:4.7,type:"Curve",generator:"Curve.toJSON"}};return e.arcLengthDivisions=this.arcLengthDivisions,e.type=this.type,e}fromJSON(e){return this.arcLengthDivisions=e.arcLengthDivisions,this.cacheArcLengths=null,this}},yl=class extends mr{constructor(e=0,t=0,i=1,r=1,a=0,o=Math.PI*2,s=!1,n=0){super(),this.isEllipseCurve=!0,this.type="EllipseCurve",this.aX=e,this.aY=t,this.xRadius=i,this.yRadius=r,this.aStartAngle=a,this.aEndAngle=o,this.aClockwise=s,this.aRotation=n}getPoint(e,t=new ge){let i=t,r=Math.PI*2,a=this.aEndAngle-this.aStartAngle,o=Math.abs(a)<2**-52;for(;a<0;)a+=r;for(;a>r;)a-=r;a<2**-52&&(a=o?0:r),this.aClockwise===!0&&!o&&(a===r?a=-r:a-=r);let s=this.aStartAngle+e*a,n=this.aX+this.xRadius*Math.cos(s),l=this.aY+this.yRadius*Math.sin(s);if(this.aRotation!==0){let h=Math.cos(this.aRotation),c=Math.sin(this.aRotation),u=n-this.aX,d=l-this.aY;n=u*h-d*c+this.aX,l=u*c+d*h+this.aY}return i.set(n,l)}copy(e){return super.copy(e),this.aX=e.aX,this.aY=e.aY,this.xRadius=e.xRadius,this.yRadius=e.yRadius,this.aStartAngle=e.aStartAngle,this.aEndAngle=e.aEndAngle,this.aClockwise=e.aClockwise,this.aRotation=e.aRotation,this}toJSON(){let e=super.toJSON();return e.aX=this.aX,e.aY=this.aY,e.xRadius=this.xRadius,e.yRadius=this.yRadius,e.aStartAngle=this.aStartAngle,e.aEndAngle=this.aEndAngle,e.aClockwise=this.aClockwise,e.aRotation=this.aRotation,e}fromJSON(e){return super.fromJSON(e),this.aX=e.aX,this.aY=e.aY,this.xRadius=e.xRadius,this.yRadius=e.yRadius,this.aStartAngle=e.aStartAngle,this.aEndAngle=e.aEndAngle,this.aClockwise=e.aClockwise,this.aRotation=e.aRotation,this}},V0=class extends yl{constructor(e,t,i,r,a,o){super(e,t,i,i,r,a,o),this.isArcCurve=!0,this.type="ArcCurve"}};function bl(){let e=0,t=0,i=0,r=0;function a(o,s,n,l){e=o,t=n,i=-3*o+3*s-2*n-l,r=2*o-2*s+n+l}return{initCatmullRom:function(o,s,n,l,h){a(s,n,h*(n-o),h*(l-s))},initNonuniformCatmullRom:function(o,s,n,l,h,c,u){let d=(s-o)/h-(n-o)/(h+c)+(n-s)/c,f=(n-s)/c-(l-s)/(c+u)+(l-n)/u;d*=c,f*=c,a(s,n,d,f)},calc:function(o){let s=o*o,n=s*o;return e+t*o+i*s+r*n}}}var ph=new I,mh=new I,An=new bl,Rn=new bl,Cn=new bl,au=class extends mr{constructor(e=[],t=!1,i="centripetal",r=.5){super(),this.isCatmullRomCurve3=!0,this.type="CatmullRomCurve3",this.points=e,this.closed=t,this.curveType=i,this.tension=r}getPoint(e,t=new I){let i=t,r=this.points,a=r.length,o=(a-+!this.closed)*e,s=Math.floor(o),n=o-s;this.closed?s+=s>0?0:(Math.floor(Math.abs(s)/a)+1)*a:n===0&&s===a-1&&(s=a-2,n=1);let l,h;this.closed||s>0?l=r[(s-1)%a]:(mh.subVectors(r[0],r[1]).add(r[0]),l=mh);let c=r[s%a],u=r[(s+1)%a];if(this.closed||s+2<a?h=r[(s+2)%a]:(ph.subVectors(r[a-1],r[a-2]).add(r[a-1]),h=ph),this.curveType==="centripetal"||this.curveType==="chordal"){let d=this.curveType==="chordal"?.5:.25,f=l.distanceToSquared(c)**+d,p=c.distanceToSquared(u)**+d,x=u.distanceToSquared(h)**+d;p<1e-4&&(p=1),f<1e-4&&(f=p),x<1e-4&&(x=p),An.initNonuniformCatmullRom(l.x,c.x,u.x,h.x,f,p,x),Rn.initNonuniformCatmullRom(l.y,c.y,u.y,h.y,f,p,x),Cn.initNonuniformCatmullRom(l.z,c.z,u.z,h.z,f,p,x)}else this.curveType==="catmullrom"&&(An.initCatmullRom(l.x,c.x,u.x,h.x,this.tension),Rn.initCatmullRom(l.y,c.y,u.y,h.y,this.tension),Cn.initCatmullRom(l.z,c.z,u.z,h.z,this.tension));return i.set(An.calc(n),Rn.calc(n),Cn.calc(n)),i}copy(e){super.copy(e),this.points=[];for(let t=0,i=e.points.length;t<i;t++){let r=e.points[t];this.points.push(r.clone())}return this.closed=e.closed,this.curveType=e.curveType,this.tension=e.tension,this}toJSON(){let e=super.toJSON();e.points=[];for(let t=0,i=this.points.length;t<i;t++){let r=this.points[t];e.points.push(r.toArray())}return e.closed=this.closed,e.curveType=this.curveType,e.tension=this.tension,e}fromJSON(e){super.fromJSON(e),this.points=[];for(let t=0,i=e.points.length;t<i;t++){let r=e.points[t];this.points.push(new I().fromArray(r))}return this.closed=e.closed,this.curveType=e.curveType,this.tension=e.tension,this}};function gh(e,t,i,r,a){let o=(r-t)*.5,s=(a-i)*.5,n=e*e,l=e*n;return(2*i-2*r+o+s)*l+(-3*i+3*r-2*o-s)*n+o*e+i}function G0(e,t){let i=1-e;return i*i*t}function H0(e,t){return 2*(1-e)*e*t}function W0(e,t){return e*e*t}function uo(e,t,i,r){return G0(e,t)+H0(e,i)+W0(e,r)}function X0(e,t){let i=1-e;return i*i*i*t}function q0(e,t){let i=1-e;return 3*i*i*e*t}function j0(e,t){return 3*(1-e)*e*e*t}function Y0(e,t){return e*e*e*t}function fo(e,t,i,r,a){return X0(e,t)+q0(e,i)+j0(e,r)+Y0(e,a)}var ou=class extends mr{constructor(e=new ge,t=new ge,i=new ge,r=new ge){super(),this.isCubicBezierCurve=!0,this.type="CubicBezierCurve",this.v0=e,this.v1=t,this.v2=i,this.v3=r}getPoint(e,t=new ge){let i=t,r=this.v0,a=this.v1,o=this.v2,s=this.v3;return i.set(fo(e,r.x,a.x,o.x,s.x),fo(e,r.y,a.y,o.y,s.y)),i}copy(e){return super.copy(e),this.v0.copy(e.v0),this.v1.copy(e.v1),this.v2.copy(e.v2),this.v3.copy(e.v3),this}toJSON(){let e=super.toJSON();return e.v0=this.v0.toArray(),e.v1=this.v1.toArray(),e.v2=this.v2.toArray(),e.v3=this.v3.toArray(),e}fromJSON(e){return super.fromJSON(e),this.v0.fromArray(e.v0),this.v1.fromArray(e.v1),this.v2.fromArray(e.v2),this.v3.fromArray(e.v3),this}},Z0=class extends mr{constructor(e=new I,t=new I,i=new I,r=new I){super(),this.isCubicBezierCurve3=!0,this.type="CubicBezierCurve3",this.v0=e,this.v1=t,this.v2=i,this.v3=r}getPoint(e,t=new I){let i=t,r=this.v0,a=this.v1,o=this.v2,s=this.v3;return i.set(fo(e,r.x,a.x,o.x,s.x),fo(e,r.y,a.y,o.y,s.y),fo(e,r.z,a.z,o.z,s.z)),i}copy(e){return super.copy(e),this.v0.copy(e.v0),this.v1.copy(e.v1),this.v2.copy(e.v2),this.v3.copy(e.v3),this}toJSON(){let e=super.toJSON();return e.v0=this.v0.toArray(),e.v1=this.v1.toArray(),e.v2=this.v2.toArray(),e.v3=this.v3.toArray(),e}fromJSON(e){return super.fromJSON(e),this.v0.fromArray(e.v0),this.v1.fromArray(e.v1),this.v2.fromArray(e.v2),this.v3.fromArray(e.v3),this}},su=class extends mr{constructor(e=new ge,t=new ge){super(),this.isLineCurve=!0,this.type="LineCurve",this.v1=e,this.v2=t}getPoint(e,t=new ge){let i=t;return e===1?i.copy(this.v2):(i.copy(this.v2).sub(this.v1),i.multiplyScalar(e).add(this.v1)),i}getPointAt(e,t){return this.getPoint(e,t)}getTangent(e,t=new ge){return t.subVectors(this.v2,this.v1).normalize()}getTangentAt(e,t){return this.getTangent(e,t)}copy(e){return super.copy(e),this.v1.copy(e.v1),this.v2.copy(e.v2),this}toJSON(){let e=super.toJSON();return e.v1=this.v1.toArray(),e.v2=this.v2.toArray(),e}fromJSON(e){return super.fromJSON(e),this.v1.fromArray(e.v1),this.v2.fromArray(e.v2),this}},J0=class extends mr{constructor(e=new I,t=new I){super(),this.isLineCurve3=!0,this.type="LineCurve3",this.v1=e,this.v2=t}getPoint(e,t=new I){let i=t;return e===1?i.copy(this.v2):(i.copy(this.v2).sub(this.v1),i.multiplyScalar(e).add(this.v1)),i}getPointAt(e,t){return this.getPoint(e,t)}getTangent(e,t=new I){return t.subVectors(this.v2,this.v1).normalize()}getTangentAt(e,t){return this.getTangent(e,t)}copy(e){return super.copy(e),this.v1.copy(e.v1),this.v2.copy(e.v2),this}toJSON(){let e=super.toJSON();return e.v1=this.v1.toArray(),e.v2=this.v2.toArray(),e}fromJSON(e){return super.fromJSON(e),this.v1.fromArray(e.v1),this.v2.fromArray(e.v2),this}},nu=class extends mr{constructor(e=new ge,t=new ge,i=new ge){super(),this.isQuadraticBezierCurve=!0,this.type="QuadraticBezierCurve",this.v0=e,this.v1=t,this.v2=i}getPoint(e,t=new ge){let i=t,r=this.v0,a=this.v1,o=this.v2;return i.set(uo(e,r.x,a.x,o.x),uo(e,r.y,a.y,o.y)),i}copy(e){return super.copy(e),this.v0.copy(e.v0),this.v1.copy(e.v1),this.v2.copy(e.v2),this}toJSON(){let e=super.toJSON();return e.v0=this.v0.toArray(),e.v1=this.v1.toArray(),e.v2=this.v2.toArray(),e}fromJSON(e){return super.fromJSON(e),this.v0.fromArray(e.v0),this.v1.fromArray(e.v1),this.v2.fromArray(e.v2),this}},lu=class extends mr{constructor(e=new I,t=new I,i=new I){super(),this.isQuadraticBezierCurve3=!0,this.type="QuadraticBezierCurve3",this.v0=e,this.v1=t,this.v2=i}getPoint(e,t=new I){let i=t,r=this.v0,a=this.v1,o=this.v2;return i.set(uo(e,r.x,a.x,o.x),uo(e,r.y,a.y,o.y),uo(e,r.z,a.z,o.z)),i}copy(e){return super.copy(e),this.v0.copy(e.v0),this.v1.copy(e.v1),this.v2.copy(e.v2),this}toJSON(){let e=super.toJSON();return e.v0=this.v0.toArray(),e.v1=this.v1.toArray(),e.v2=this.v2.toArray(),e}fromJSON(e){return super.fromJSON(e),this.v0.fromArray(e.v0),this.v1.fromArray(e.v1),this.v2.fromArray(e.v2),this}},hu=class extends mr{constructor(e=[]){super(),this.isSplineCurve=!0,this.type="SplineCurve",this.points=e}getPoint(e,t=new ge){let i=t,r=this.points,a=(r.length-1)*e,o=Math.floor(a),s=a-o,n=r[o===0?o:o-1],l=r[o],h=r[o>r.length-2?r.length-1:o+1],c=r[o>r.length-3?r.length-1:o+2];return i.set(gh(s,n.x,l.x,h.x,c.x),gh(s,n.y,l.y,h.y,c.y)),i}copy(e){super.copy(e),this.points=[];for(let t=0,i=e.points.length;t<i;t++){let r=e.points[t];this.points.push(r.clone())}return this}toJSON(){let e=super.toJSON();e.points=[];for(let t=0,i=this.points.length;t<i;t++){let r=this.points[t];e.points.push(r.toArray())}return e}fromJSON(e){super.fromJSON(e),this.points=[];for(let t=0,i=e.points.length;t<i;t++){let r=e.points[t];this.points.push(new ge().fromArray(r))}return this}},zs=Object.freeze({__proto__:null,ArcCurve:V0,CatmullRomCurve3:au,CubicBezierCurve:ou,CubicBezierCurve3:Z0,EllipseCurve:yl,LineCurve:su,LineCurve3:J0,QuadraticBezierCurve:nu,QuadraticBezierCurve3:lu,SplineCurve:hu}),K0=class extends mr{constructor(){super(),this.type="CurvePath",this.curves=[],this.autoClose=!1,this.cacheLengths=null}add(e){this.curves.push(e)}closePath(){let e=this.curves[0].getPoint(0),t=this.curves[this.curves.length-1].getPoint(1);if(!e.equals(t)){let i=e.isVector2===!0?"LineCurve":"LineCurve3";this.curves.push(new zs[i](t,e))}return this}getPoint(e,t){let i=e*this.getLength(),r=this.getCurveLengths(),a=0;for(;a<r.length;){if(r[a]>=i){let o=r[a]-i,s=this.curves[a],n=s.getLength(),l=n===0?0:1-o/n;return s.getPointAt(l,t)}a++}return null}getLength(){let e=this.getCurveLengths();return e[e.length-1]}updateArcLengths(){this.needsUpdate=!0,this.cacheLengths=null,this.getCurveLengths()}getCurveLengths(){if(this.cacheLengths&&this.cacheLengths.length===this.curves.length)return this.cacheLengths;let e=[],t=0;for(let i=0,r=this.curves.length;i<r;i++)t+=this.curves[i].getLength(),e.push(t);return this.cacheLengths=e,e}getSpacedPoints(e=40){let t=[];for(let i=0;i<=e;i++)t.push(this.getPoint(i/e));return this.autoClose&&t.push(t[0]),t}getPoints(e=12){let t=[],i;for(let r=0,a=this.curves;r<a.length;r++){let o=a[r],s=o.isEllipseCurve?e*2:o.isLineCurve||o.isLineCurve3?1:o.isSplineCurve?e*o.points.length:e,n=o.getPoints(s);for(let l=0;l<n.length;l++){let h=n[l];i&&i.equals(h)||(t.push(h),i=h)}}return this.autoClose&&t.length>1&&!t[t.length-1].equals(t[0])&&t.push(t[0]),t}copy(e){super.copy(e),this.curves=[];for(let t=0,i=e.curves.length;t<i;t++){let r=e.curves[t];this.curves.push(r.clone())}return this.autoClose=e.autoClose,this.cacheLengths=null,this}toJSON(){let e=super.toJSON();e.autoClose=this.autoClose,e.curves=[];for(let t=0,i=this.curves.length;t<i;t++){let r=this.curves[t];e.curves.push(r.toJSON())}return e}fromJSON(e){super.fromJSON(e),this.autoClose=e.autoClose,this.curves=[];for(let t=0,i=e.curves.length;t<i;t++){let r=e.curves[t];this.curves.push(new zs[r.type]().fromJSON(r))}return this.cacheLengths=null,this}},Fs=class extends K0{constructor(e){super(),this.type="Path",this.currentPoint=new ge,e&&this.setFromPoints(e)}setFromPoints(e){this.moveTo(e[0].x,e[0].y);for(let t=1,i=e.length;t<i;t++)this.lineTo(e[t].x,e[t].y);return this}moveTo(e,t){return this.currentPoint.set(e,t),this}lineTo(e,t){let i=new su(this.currentPoint.clone(),new ge(e,t));return this.curves.push(i),this.currentPoint.set(e,t),this}quadraticCurveTo(e,t,i,r){let a=new nu(this.currentPoint.clone(),new ge(e,t),new ge(i,r));return this.curves.push(a),this.currentPoint.set(i,r),this}bezierCurveTo(e,t,i,r,a,o){let s=new ou(this.currentPoint.clone(),new ge(e,t),new ge(i,r),new ge(a,o));return this.curves.push(s),this.currentPoint.set(a,o),this}splineThru(e){let t=new hu([this.currentPoint.clone()].concat(e));return this.curves.push(t),this.currentPoint.copy(e[e.length-1]),this}arc(e,t,i,r,a,o){let s=this.currentPoint.x,n=this.currentPoint.y;return this.absarc(e+s,t+n,i,r,a,o),this}absarc(e,t,i,r,a,o){return this.absellipse(e,t,i,i,r,a,o),this}ellipse(e,t,i,r,a,o,s,n){let l=this.currentPoint.x,h=this.currentPoint.y;return this.absellipse(e+l,t+h,i,r,a,o,s,n),this}absellipse(e,t,i,r,a,o,s,n){let l=new yl(e,t,i,r,a,o,s,n);if(this.curves.length>0){let c=l.getPoint(0);c.equals(this.currentPoint)||this.lineTo(c.x,c.y)}this.curves.push(l);let h=l.getPoint(1);return this.currentPoint.copy(h),this}copy(e){return super.copy(e),this.currentPoint.copy(e.currentPoint),this}toJSON(){let e=super.toJSON();return e.currentPoint=this.currentPoint.toArray(),e}fromJSON(e){return super.fromJSON(e),this.currentPoint.fromArray(e.currentPoint),this}},Ss=class extends Fs{constructor(e){super(e),this.uuid=qa(),this.type="Shape",this.holes=[]}getPointsHoles(e){let t=[];for(let i=0,r=this.holes.length;i<r;i++)t[i]=this.holes[i].getPoints(e);return t}extractPoints(e){return{shape:this.getPoints(e),holes:this.getPointsHoles(e)}}copy(e){super.copy(e),this.holes=[];for(let t=0,i=e.holes.length;t<i;t++){let r=e.holes[t];this.holes.push(r.clone())}return this}toJSON(){let e=super.toJSON();e.uuid=this.uuid,e.holes=[];for(let t=0,i=this.holes.length;t<i;t++){let r=this.holes[t];e.holes.push(r.toJSON())}return e}fromJSON(e){super.fromJSON(e),this.uuid=e.uuid,this.holes=[];for(let t=0,i=e.holes.length;t<i;t++){let r=e.holes[t];this.holes.push(new Fs().fromJSON(r))}return this}};function $0(e,t,i=2){let r=t&&t.length,a=r?t[0]*i:e.length,o=cu(e,0,a,i,!0),s=[];if(!o||o.next===o.prev)return s;let n,l,h;if(r&&(o=af(e,t,o,i)),e.length>80*i){n=e[0],l=e[1];let c=n,u=l;for(let d=i;d<a;d+=i){let f=e[d],p=e[d+1];f<n&&(n=f),p<l&&(l=p),f>c&&(c=f),p>u&&(u=p)}h=Math.max(c-n,u-l),h=h===0?0:32767/h}return Mo(o,s,i,n,l,h,0),s}function cu(e,t,i,r,a){let o;if(a===mf(e,t,i,r)>0)for(let s=t;s<i;s+=r)o=vh(s/r|0,e[s],e[s+1],o);else for(let s=i-r;s>=t;s-=r)o=vh(s/r|0,e[s],e[s+1],o);return o&&Fa(o,o.next)&&(wo(o),o=o.next),o}function aa(e,t){if(!e)return e;t||=e;let i=e,r;do if(r=!1,!i.steiner&&(Fa(i,i.next)||Vt(i.prev,i,i.next)===0)){if(wo(i),i=t=i.prev,i===i.next)break;r=!0}else i=i.next;while(r||i!==t);return t}function Mo(e,t,i,r,a,o,s){if(!e)return;!s&&o&&hf(e,r,a,o);let n=e;for(;e.prev!==e.next;){let l=e.prev,h=e.next;if(o?ef(e,r,a,o):Q0(e))t.push(l.i,e.i,h.i),wo(e),e=h.next,n=h.next;else if(e=h,e===n){s?s===1?(e=tf(aa(e),t),Mo(e,t,i,r,a,o,2)):s===2&&rf(e,t,i,r,a,o):Mo(aa(e),t,i,r,a,o,1);break}}}function Q0(e){let t=e.prev,i=e,r=e.next;if(Vt(t,i,r)>=0)return!1;let a=t.x,o=i.x,s=r.x,n=t.y,l=i.y,h=r.y,c=Math.min(a,o,s),u=Math.min(n,l,h),d=Math.max(a,o,s),f=Math.max(n,l,h),p=r.next;for(;p!==t;){if(p.x>=c&&p.x<=d&&p.y>=u&&p.y<=f&&ho(a,n,o,l,s,h,p.x,p.y)&&Vt(p.prev,p,p.next)>=0)return!1;p=p.next}return!0}function ef(e,t,i,r){let a=e.prev,o=e,s=e.next;if(Vt(a,o,s)>=0)return!1;let n=a.x,l=o.x,h=s.x,c=a.y,u=o.y,d=s.y,f=Math.min(n,l,h),p=Math.min(c,u,d),x=Math.max(n,l,h),g=Math.max(c,u,d),m=$n(f,p,t,i,r),y=$n(x,g,t,i,r),M=e.prevZ,b=e.nextZ;for(;M&&M.z>=m&&b&&b.z<=y;){if(M.x>=f&&M.x<=x&&M.y>=p&&M.y<=g&&M!==a&&M!==s&&ho(n,c,l,u,h,d,M.x,M.y)&&Vt(M.prev,M,M.next)>=0||(M=M.prevZ,b.x>=f&&b.x<=x&&b.y>=p&&b.y<=g&&b!==a&&b!==s&&ho(n,c,l,u,h,d,b.x,b.y)&&Vt(b.prev,b,b.next)>=0))return!1;b=b.nextZ}for(;M&&M.z>=m;){if(M.x>=f&&M.x<=x&&M.y>=p&&M.y<=g&&M!==a&&M!==s&&ho(n,c,l,u,h,d,M.x,M.y)&&Vt(M.prev,M,M.next)>=0)return!1;M=M.prevZ}for(;b&&b.z<=y;){if(b.x>=f&&b.x<=x&&b.y>=p&&b.y<=g&&b!==a&&b!==s&&ho(n,c,l,u,h,d,b.x,b.y)&&Vt(b.prev,b,b.next)>=0)return!1;b=b.nextZ}return!0}function tf(e,t){let i=e;do{let r=i.prev,a=i.next.next;!Fa(r,a)&&du(r,i,i.next,a)&&So(r,a)&&So(a,r)&&(t.push(r.i,i.i,a.i),wo(i),wo(i.next),i=e=a),i=i.next}while(i!==e);return aa(i)}function rf(e,t,i,r,a,o){let s=e;do{let n=s.next.next;for(;n!==s.prev;){if(s.i!==n.i&&df(s,n)){let l=fu(s,n);s=aa(s,s.next),l=aa(l,l.next),Mo(s,t,i,r,a,o,0),Mo(l,t,i,r,a,o,0);return}n=n.next}s=s.next}while(s!==e)}function af(e,t,i,r){let a=[];for(let o=0,s=t.length;o<s;o++){let n=cu(e,t[o]*r,o<s-1?t[o+1]*r:e.length,r,!1);n===n.next&&(n.steiner=!0),a.push(uf(n))}a.sort(of);for(let o=0;o<a.length;o++)i=sf(a[o],i);return i}function of(e,t){let i=e.x-t.x;return i===0&&(i=e.y-t.y,i===0&&(i=(e.next.y-e.y)/(e.next.x-e.x)-(t.next.y-t.y)/(t.next.x-t.x))),i}function sf(e,t){let i=nf(e,t);if(!i)return t;let r=fu(i,e);return aa(r,r.next),aa(i,i.next)}function nf(e,t){let i=t,r=e.x,a=e.y,o=-1/0,s;if(Fa(e,i))return i;do{if(Fa(e,i.next))return i.next;if(a<=i.y&&a>=i.next.y&&i.next.y!==i.y){let u=i.x+(a-i.y)*(i.next.x-i.x)/(i.next.y-i.y);if(u<=r&&u>o&&(o=u,s=i.x<i.next.x?i:i.next,u===r))return s}i=i.next}while(i!==t);if(!s)return null;let n=s,l=s.x,h=s.y,c=1/0;i=s;do{if(r>=i.x&&i.x>=l&&r!==i.x&&uu(a<h?r:o,a,l,h,a<h?o:r,a,i.x,i.y)){let u=Math.abs(a-i.y)/(r-i.x);So(i,e)&&(u<c||u===c&&(i.x>s.x||i.x===s.x&&lf(s,i)))&&(s=i,c=u)}i=i.next}while(i!==n);return s}function lf(e,t){return Vt(e.prev,e,t.prev)<0&&Vt(t.next,e,e.next)<0}function hf(e,t,i,r){let a=e;do a.z===0&&(a.z=$n(a.x,a.y,t,i,r)),a.prevZ=a.prev,a.nextZ=a.next,a=a.next;while(a!==e);a.prevZ.nextZ=null,a.prevZ=null,cf(a)}function cf(e){let t,i=1;do{let r=e,a;e=null;let o=null;for(t=0;r;){t++;let s=r,n=0;for(let h=0;h<i&&(n++,s=s.nextZ,s);h++);let l=i;for(;n>0||l>0&&s;)n!==0&&(l===0||!s||r.z<=s.z)?(a=r,r=r.nextZ,n--):(a=s,s=s.nextZ,l--),o?o.nextZ=a:e=a,a.prevZ=o,o=a;r=s}o.nextZ=null,i*=2}while(t>1);return e}function $n(e,t,i,r,a){return e=(e-i)*a|0,t=(t-r)*a|0,e=(e|e<<8)&16711935,e=(e|e<<4)&252645135,e=(e|e<<2)&858993459,e=(e|e<<1)&1431655765,t=(t|t<<8)&16711935,t=(t|t<<4)&252645135,t=(t|t<<2)&858993459,t=(t|t<<1)&1431655765,e|t<<1}function uf(e){let t=e,i=e;do(t.x<i.x||t.x===i.x&&t.y<i.y)&&(i=t),t=t.next;while(t!==e);return i}function uu(e,t,i,r,a,o,s,n){return(a-s)*(t-n)>=(e-s)*(o-n)&&(e-s)*(r-n)>=(i-s)*(t-n)&&(i-s)*(o-n)>=(a-s)*(r-n)}function ho(e,t,i,r,a,o,s,n){return(e!==s||t!==n)&&uu(e,t,i,r,a,o,s,n)}function df(e,t){return e.next.i!==t.i&&e.prev.i!==t.i&&!ff(e,t)&&(So(e,t)&&So(t,e)&&pf(e,t)&&(Vt(e.prev,e,t.prev)||Vt(e,t.prev,t))||Fa(e,t)&&Vt(e.prev,e,e.next)>0&&Vt(t.prev,t,t.next)>0)}function Vt(e,t,i){return(t.y-e.y)*(i.x-t.x)-(t.x-e.x)*(i.y-t.y)}function Fa(e,t){return e.x===t.x&&e.y===t.y}function du(e,t,i,r){let a=is(Vt(e,t,i)),o=is(Vt(e,t,r)),s=is(Vt(i,r,e)),n=is(Vt(i,r,t));return!!(a!==o&&s!==n||a===0&&ts(e,i,t)||o===0&&ts(e,r,t)||s===0&&ts(i,e,r)||n===0&&ts(i,t,r))}function ts(e,t,i){return t.x<=Math.max(e.x,i.x)&&t.x>=Math.min(e.x,i.x)&&t.y<=Math.max(e.y,i.y)&&t.y>=Math.min(e.y,i.y)}function is(e){return e>0?1:e<0?-1:0}function ff(e,t){let i=e;do{if(i.i!==e.i&&i.next.i!==e.i&&i.i!==t.i&&i.next.i!==t.i&&du(i,i.next,e,t))return!0;i=i.next}while(i!==e);return!1}function So(e,t){return Vt(e.prev,e,e.next)<0?Vt(e,t,e.next)>=0&&Vt(e,e.prev,t)>=0:Vt(e,t,e.prev)<0||Vt(e,e.next,t)<0}function pf(e,t){let i=e,r=!1,a=(e.x+t.x)/2,o=(e.y+t.y)/2;do i.y>o!=i.next.y>o&&i.next.y!==i.y&&a<(i.next.x-i.x)*(o-i.y)/(i.next.y-i.y)+i.x&&(r=!r),i=i.next;while(i!==e);return r}function fu(e,t){let i=Qn(e.i,e.x,e.y),r=Qn(t.i,t.x,t.y),a=e.next,o=t.prev;return e.next=t,t.prev=e,i.next=a,a.prev=i,r.next=i,i.prev=r,o.next=r,r.prev=o,r}function vh(e,t,i,r){let a=Qn(e,t,i);return r?(a.next=r.next,a.prev=r,r.next.prev=a,r.next=a):(a.prev=a,a.next=a),a}function wo(e){e.next.prev=e.prev,e.prev.next=e.next,e.prevZ&&(e.prevZ.nextZ=e.nextZ),e.nextZ&&(e.nextZ.prevZ=e.prevZ)}function Qn(e,t,i){return{i:e,x:t,y:i,prev:null,next:null,z:0,prevZ:null,nextZ:null,steiner:!1}}function mf(e,t,i,r){let a=0;for(let o=t,s=i-r;o<i;o+=r)a+=(e[s]-e[o])*(e[o+1]+e[s+1]),s=o;return a}var gf=class{static triangulate(e,t,i=2){return $0(e,t,i)}},rs=class pu{static area(t){let i=t.length,r=0;for(let a=i-1,o=0;o<i;a=o++)r+=t[a].x*t[o].y-t[o].x*t[a].y;return r*.5}static isClockWise(t){return pu.area(t)<0}static triangulateShape(t,i){let r=[],a=[],o=[];xh(t),_h(r,t);let s=t.length;i.forEach(xh);for(let l=0;l<i.length;l++)a.push(s),s+=i[l].length,_h(r,i[l]);let n=gf.triangulate(r,a);for(let l=0;l<n.length;l+=3)o.push(n.slice(l,l+3));return o}};function xh(e){let t=e.length;t>2&&e[t-1].equals(e[0])&&e.pop()}function _h(e,t){for(let i=0;i<t.length;i++)e.push(t[i].x),e.push(t[i].y)}var Pn=class mu extends ti{constructor(t=new Ss([new ge(.5,.5),new ge(-.5,.5),new ge(-.5,-.5),new ge(.5,-.5)]),i={}){super(),this.type="ExtrudeGeometry",this.parameters={shapes:t,options:i},t=Array.isArray(t)?t:[t];let r=this,a=[],o=[];for(let n=0,l=t.length;n<l;n++){let h=t[n];s(h)}this.setAttribute("position",new wt(a,3)),this.setAttribute("uv",new wt(o,2)),this.computeVertexNormals();function s(n){let l=[],h=i.curveSegments===void 0?12:i.curveSegments,c=i.steps===void 0?1:i.steps,u=i.depth===void 0?1:i.depth,d=i.bevelEnabled===void 0||i.bevelEnabled,f=i.bevelThickness===void 0?.2:i.bevelThickness,p=i.bevelSize===void 0?f-.1:i.bevelSize,x=i.bevelOffset===void 0?0:i.bevelOffset,g=i.bevelSegments===void 0?3:i.bevelSegments,m=i.extrudePath,y=i.UVGenerator===void 0?vf:i.UVGenerator,M,b=!1,E,T,v,_;if(m){M=m.getSpacedPoints(c),b=!0,d=!1;let oe=m.isCatmullRomCurve3?m.closed:!1;E=m.computeFrenetFrames(c,oe),T=new I,v=new I,_=new I}d||(g=0,f=0,p=0,x=0);let A=n.extractPoints(h),C=A.shape,R=A.holes;if(!rs.isClockWise(C)){C=C.reverse();for(let oe=0,he=R.length;oe<he;oe++){let de=R[oe];rs.isClockWise(de)&&(R[oe]=de.reverse())}}function L(oe){let he=oe[0];for(let de=1;de<=oe.length;de++){let Ue=de%oe.length,Ie=oe[Ue],F=Ie.x-he.x,te=Ie.y-he.y,$=F*F+te*te,Te=Math.max(Math.abs(Ie.x),Math.abs(Ie.y),Math.abs(he.x),Math.abs(he.y));$<=10000000000000001e-36*Te*Te?(oe.splice(Ue,1),de--):he=Ie}}L(C),R.forEach(L);let G=R.length,P=C;for(let oe=0;oe<G;oe++){let he=R[oe];C=C.concat(he)}function z(oe,he,de){return he||Qe("ExtrudeGeometry: vec does not exist"),oe.clone().addScaledVector(he,de)}let H=C.length;function D(oe,he,de){let Ue,Ie,F,te=oe.x-he.x,$=oe.y-he.y,Te=de.x-oe.x,Be=de.y-oe.y,U=te*te+$*$,ut=te*Be-$*Te;if(Math.abs(ut)>2**-52){let B=Math.sqrt(U),Ce=Math.sqrt(Te*Te+Be*Be),ne=he.x-$/B,lt=he.y+te/B,Ge=de.x-Be/Ce,je=de.y+Te/Ce,At=((Ge-ne)*Be-(je-lt)*Te)/(te*Be-$*Te);Ue=ne+te*At-oe.x,Ie=lt+$*At-oe.y;let N=Ue*Ue+Ie*Ie;if(N<=2)return new ge(Ue,Ie);F=Math.sqrt(N/2)}else{let B=!1;te>2**-52?Te>2**-52&&(B=!0):te<-(2**-52)?Te<-(2**-52)&&(B=!0):Math.sign($)===Math.sign(Be)&&(B=!0),B?(Ue=-$,Ie=te,F=Math.sqrt(U)):(Ue=te,Ie=$,F=Math.sqrt(U/2))}return new ge(Ue/F,Ie/F)}let ae=[];for(let oe=0,he=P.length,de=he-1,Ue=oe+1;oe<he;oe++,de++,Ue++)de===he&&(de=0),Ue===he&&(Ue=0),ae[oe]=D(P[oe],P[de],P[Ue]);let j=[],Y,ie=ae.concat();for(let oe=0,he=G;oe<he;oe++){let de=R[oe];Y=[];for(let Ue=0,Ie=de.length,F=Ie-1,te=Ue+1;Ue<Ie;Ue++,F++,te++)F===Ie&&(F=0),te===Ie&&(te=0),Y[Ue]=D(de[Ue],de[F],de[te]);j.push(Y),ie=ie.concat(Y)}let Se;if(g===0)Se=rs.triangulateShape(P,R);else{let oe=[],he=[];for(let de=0;de<g;de++){let Ue=de/g,Ie=f*Math.cos(Ue*Math.PI/2),F=p*Math.sin(Ue*Math.PI/2)+x;for(let te=0,$=P.length;te<$;te++){let Te=z(P[te],ae[te],F);Q(Te.x,Te.y,-Ie),Ue===0&&oe.push(Te)}for(let te=0,$=G;te<$;te++){let Te=R[te];Y=j[te];let Be=[];for(let U=0,ut=Te.length;U<ut;U++){let B=z(Te[U],Y[U],F);Q(B.x,B.y,-Ie),Ue===0&&Be.push(B)}Ue===0&&he.push(Be)}}Se=rs.triangulateShape(oe,he)}let K=Se.length,re=p+x;for(let oe=0;oe<H;oe++){let he=d?z(C[oe],ie[oe],re):C[oe];b?(v.copy(E.normals[0]).multiplyScalar(he.x),T.copy(E.binormals[0]).multiplyScalar(he.y),_.copy(M[0]).add(v).add(T),Q(_.x,_.y,_.z)):Q(he.x,he.y,0)}for(let oe=1;oe<=c;oe++)for(let he=0;he<H;he++){let de=d?z(C[he],ie[he],re):C[he];b?(v.copy(E.normals[oe]).multiplyScalar(de.x),T.copy(E.binormals[oe]).multiplyScalar(de.y),_.copy(M[oe]).add(v).add(T),Q(_.x,_.y,_.z)):Q(de.x,de.y,u/c*oe)}for(let oe=g-1;oe>=0;oe--){let he=oe/g,de=f*Math.cos(he*Math.PI/2),Ue=p*Math.sin(he*Math.PI/2)+x;for(let Ie=0,F=P.length;Ie<F;Ie++){let te=z(P[Ie],ae[Ie],Ue);Q(te.x,te.y,u+de)}for(let Ie=0,F=R.length;Ie<F;Ie++){let te=R[Ie];Y=j[Ie];for(let $=0,Te=te.length;$<Te;$++){let Be=z(te[$],Y[$],Ue);b?Q(Be.x,Be.y+M[c-1].y,M[c-1].x+de):Q(Be.x,Be.y,u+de)}}}xe(),Ne();function xe(){let oe=a.length/3;if(d){let he=0,de=H*he;for(let Ue=0;Ue<K;Ue++){let Ie=Se[Ue];ke(Ie[2]+de,Ie[1]+de,Ie[0]+de)}he=c+g*2,de=H*he;for(let Ue=0;Ue<K;Ue++){let Ie=Se[Ue];ke(Ie[0]+de,Ie[1]+de,Ie[2]+de)}}else{for(let he=0;he<K;he++){let de=Se[he];ke(de[2],de[1],de[0])}for(let he=0;he<K;he++){let de=Se[he];ke(de[0]+H*c,de[1]+H*c,de[2]+H*c)}}r.addGroup(oe,a.length/3-oe,0)}function Ne(){let oe=a.length/3,he=0;Me(P,he),he+=P.length;for(let de=0,Ue=R.length;de<Ue;de++){let Ie=R[de];Me(Ie,he),he+=Ie.length}r.addGroup(oe,a.length/3-oe,1)}function Me(oe,he){let de=oe.length;for(;--de>=0;){let Ue=de,Ie=de-1;Ie<0&&(Ie=oe.length-1);for(let F=0,te=c+g*2;F<te;F++){let $=H*F,Te=H*(F+1);$e(he+Ue+$,he+Ie+$,he+Ie+Te,he+Ue+Te)}}}function Q(oe,he,de){l.push(oe),l.push(he),l.push(de)}function ke(oe,he,de){De(oe),De(he),De(de);let Ue=a.length/3,Ie=y.generateTopUV(r,a,Ue-3,Ue-2,Ue-1);nt(Ie[0]),nt(Ie[1]),nt(Ie[2])}function $e(oe,he,de,Ue){De(oe),De(he),De(Ue),De(he),De(de),De(Ue);let Ie=a.length/3,F=y.generateSideWallUV(r,a,Ie-6,Ie-3,Ie-2,Ie-1);nt(F[0]),nt(F[1]),nt(F[3]),nt(F[1]),nt(F[2]),nt(F[3])}function De(oe){a.push(l[oe*3+0]),a.push(l[oe*3+1]),a.push(l[oe*3+2])}function nt(oe){o.push(oe.x),o.push(oe.y)}}}copy(t){return super.copy(t),this.parameters=Object.assign({},t.parameters),this}toJSON(){let t=super.toJSON(),i=this.parameters.shapes,r=this.parameters.options;return xf(i,r,t)}static fromJSON(t,i){let r=[];for(let o=0,s=t.shapes.length;o<s;o++){let n=i[t.shapes[o]];r.push(n)}let a=t.options.extrudePath;return a!==void 0&&(t.options.extrudePath=new zs[a.type]().fromJSON(a)),new mu(r,t.options)}},vf={generateTopUV:function(e,t,i,r,a){let o=t[i*3],s=t[i*3+1],n=t[r*3],l=t[r*3+1],h=t[a*3],c=t[a*3+1];return[new ge(o,s),new ge(n,l),new ge(h,c)]},generateSideWallUV:function(e,t,i,r,a,o){let s=t[i*3],n=t[i*3+1],l=t[i*3+2],h=t[r*3],c=t[r*3+1],u=t[r*3+2],d=t[a*3],f=t[a*3+1],p=t[a*3+2],x=t[o*3],g=t[o*3+1],m=t[o*3+2];return Math.abs(n-c)<Math.abs(s-h)?[new ge(s,1-l),new ge(h,1-u),new ge(d,1-p),new ge(x,1-m)]:[new ge(n,1-l),new ge(c,1-u),new ge(f,1-p),new ge(g,1-m)]}};function xf(e,t,i){if(i.shapes=[],Array.isArray(e))for(let r=0,a=e.length;r<a;r++){let o=e[r];i.shapes.push(o.uuid)}else i.shapes.push(e.uuid);return i.options=Object.assign({},t),t.extrudePath!==void 0&&(i.options.extrudePath=t.extrudePath.toJSON()),i}var gu=class vu extends k0{constructor(t=1,i=0){let r=(1+Math.sqrt(5))/2,a=[-1,r,0,1,r,0,-1,-r,0,1,-r,0,0,-1,r,0,1,r,0,-1,-r,0,1,-r,r,0,-1,r,0,1,-r,0,-1,-r,0,1];super(a,[0,11,5,0,5,1,0,1,7,0,7,10,0,10,11,1,5,9,5,11,4,11,10,2,10,7,6,7,1,8,3,9,4,3,4,2,3,2,6,3,6,8,3,8,9,4,9,5,2,4,11,6,2,10,8,6,7,9,8,1],t,i),this.type="IcosahedronGeometry",this.parameters={radius:t,detail:i}}static fromJSON(t){return new vu(t.radius,t.detail)}},_f=class xu extends ti{constructor(t=[new ge(0,-.5),new ge(.5,0),new ge(0,.5)],i=12,r=0,a=Math.PI*2){super(),this.type="LatheGeometry",this.parameters={points:t,segments:i,phiStart:r,phiLength:a},i=Math.floor(i),a=ct(a,0,Math.PI*2);let o=[],s=[],n=[],l=[],h=[],c=1/i,u=new I,d=new ge,f=new I,p=new I,x=new I,g=0,m=0;for(let y=0;y<=t.length-1;y++)switch(y){case 0:g=t[y+1].x-t[y].x,m=t[y+1].y-t[y].y,f.x=m*1,f.y=-g,f.z=m*0,x.copy(f),f.normalize(),l.push(f.x,f.y,f.z);break;case t.length-1:x.normalize(),l.push(x.x,x.y,x.z);break;default:g=t[y+1].x-t[y].x,m=t[y+1].y-t[y].y,f.x=m*1,f.y=-g,f.z=m*0,p.copy(f),f.x+=x.x,f.y+=x.y,f.z+=x.z,f.normalize(),l.push(f.x,f.y,f.z),x.copy(p)}for(let y=0;y<=i;y++){let M=r+y*c*a,b=Math.sin(M),E=Math.cos(M);for(let T=0;T<=t.length-1;T++){u.x=t[T].x*b,u.y=t[T].y,u.z=t[T].x*E,s.push(u.x,u.y,u.z),d.x=y/i,d.y=T/(t.length-1),n.push(d.x,d.y);let v=l[3*T+0]*b,_=l[3*T+1],A=l[3*T+0]*E;h.push(v,_,A)}}for(let y=0;y<i;y++)for(let M=0;M<t.length-1;M++){let b=M+y*t.length,E=b,T=b+t.length,v=b+t.length+1,_=b+1;o.push(E,T,_),o.push(v,_,T)}this.setIndex(o),this.setAttribute("position",new wt(s,3)),this.setAttribute("uv",new wt(n,2)),this.setAttribute("normal",new wt(h,3))}copy(t){return super.copy(t),this.parameters=Object.assign({},t.parameters),this}static fromJSON(t){return new xu(t.points,t.segments,t.phiStart,t.phiLength)}},Br=class _u extends ti{constructor(t=1,i=1,r=1,a=1){super(),this.type="PlaneGeometry",this.parameters={width:t,height:i,widthSegments:r,heightSegments:a};let o=t/2,s=i/2,n=Math.floor(r),l=Math.floor(a),h=n+1,c=l+1,u=t/n,d=i/l,f=[],p=[],x=[],g=[];for(let m=0;m<c;m++){let y=m*d-s;for(let M=0;M<h;M++){let b=M*u-o;p.push(b,-y,0),x.push(0,0,1),g.push(M/n),g.push(1-m/l)}}for(let m=0;m<l;m++)for(let y=0;y<n;y++){let M=y+h*m,b=y+h*(m+1),E=y+1+h*(m+1),T=y+1+h*m;f.push(M,b,T),f.push(b,E,T)}this.setIndex(f),this.setAttribute("position",new wt(p,3)),this.setAttribute("normal",new wt(x,3)),this.setAttribute("uv",new wt(g,2))}copy(t){return super.copy(t),this.parameters=Object.assign({},t.parameters),this}static fromJSON(t){return new _u(t.width,t.height,t.widthSegments,t.heightSegments)}},yu=class bu extends ti{constructor(t=1,i=32,r=16,a=0,o=Math.PI*2,s=0,n=Math.PI){super(),this.type="SphereGeometry",this.parameters={radius:t,widthSegments:i,heightSegments:r,phiStart:a,phiLength:o,thetaStart:s,thetaLength:n},i=Math.max(3,Math.floor(i)),r=Math.max(2,Math.floor(r));let l=Math.min(s+n,Math.PI),h=0,c=[],u=new I,d=new I,f=[],p=[],x=[],g=[];for(let m=0;m<=r;m++){let y=[],M=m/r,b=s+M*n,E=t*Math.cos(b),T=Math.sqrt(t*t-E*E),v=0;m===0&&s===0?v=.5/i:m===r&&l===Math.PI&&(v=-.5/i);for(let _=0;_<=i;_++){let A=_/i,C=a+A*o;u.x=-T*Math.cos(C),u.y=E,u.z=T*Math.sin(C),p.push(u.x,u.y,u.z),d.copy(u).normalize(),x.push(d.x,d.y,d.z),g.push(A+v,1-M),y.push(h++)}c.push(y)}for(let m=0;m<r;m++)for(let y=0;y<i;y++){let M=c[m][y+1],b=c[m][y],E=c[m+1][y],T=c[m+1][y+1];(m!==0||s>0)&&f.push(M,b,T),(m!==r-1||l<Math.PI)&&f.push(b,E,T)}this.setIndex(f),this.setAttribute("position",new wt(p,3)),this.setAttribute("normal",new wt(x,3)),this.setAttribute("uv",new wt(g,2))}copy(t){return super.copy(t),this.parameters=Object.assign({},t.parameters),this}static fromJSON(t){return new bu(t.radius,t.widthSegments,t.heightSegments,t.phiStart,t.phiLength,t.thetaStart,t.thetaLength)}},yf=class Mu extends ti{constructor(t=1,i=.4,r=12,a=48,o=Math.PI*2,s=0,n=Math.PI*2){super(),this.type="TorusGeometry",this.parameters={radius:t,tube:i,radialSegments:r,tubularSegments:a,arc:o,thetaStart:s,thetaLength:n},r=Math.floor(r),a=Math.floor(a);let l=[],h=[],c=[],u=[],d=new I,f=new I,p=new I;for(let x=0;x<=r;x++){let g=s+x/r*n;for(let m=0;m<=a;m++){let y=m/a*o;f.x=(t+i*Math.cos(g))*Math.cos(y),f.y=(t+i*Math.cos(g))*Math.sin(y),f.z=i*Math.sin(g),h.push(f.x,f.y,f.z),d.x=t*Math.cos(y),d.y=t*Math.sin(y),p.subVectors(f,d).normalize(),c.push(p.x,p.y,p.z),u.push(m/a),u.push(x/r)}}for(let x=1;x<=r;x++)for(let g=1;g<=a;g++){let m=(a+1)*x+g-1,y=(a+1)*(x-1)+g-1,M=(a+1)*(x-1)+g,b=(a+1)*x+g;l.push(m,y,b),l.push(y,M,b)}this.setIndex(l),this.setAttribute("position",new wt(h,3)),this.setAttribute("normal",new wt(c,3)),this.setAttribute("uv",new wt(u,2))}copy(t){return super.copy(t),this.parameters=Object.assign({},t.parameters),this}static fromJSON(t){return new Mu(t.radius,t.tube,t.radialSegments,t.tubularSegments,t.arc,t.thetaStart,t.thetaLength)}},bf=class Su extends ti{constructor(t=new lu(new I(-1,-1,0),new I(-1,1,0),new I(1,1,0)),i=64,r=1,a=8,o=!1){super(),this.type="TubeGeometry",this.parameters={path:t,tubularSegments:i,radius:r,radialSegments:a,closed:o};let s=t.computeFrenetFrames(i,o);this.tangents=s.tangents,this.normals=s.normals,this.binormals=s.binormals;let n=new I,l=new I,h=new ge,c=new I,u=[],d=[],f=[],p=[];x(),this.setIndex(p),this.setAttribute("position",new wt(u,3)),this.setAttribute("normal",new wt(d,3)),this.setAttribute("uv",new wt(f,2));function x(){for(let M=0;M<i;M++)g(M);g(o===!1?i:0),y(),m()}function g(M){c=t.getPointAt(M/i,c);let b=s.normals[M],E=s.binormals[M];for(let T=0;T<=a;T++){let v=T/a*Math.PI*2,_=Math.sin(v),A=-Math.cos(v);l.x=A*b.x+_*E.x,l.y=A*b.y+_*E.y,l.z=A*b.z+_*E.z,l.normalize(),d.push(l.x,l.y,l.z),n.x=c.x+r*l.x,n.y=c.y+r*l.y,n.z=c.z+r*l.z,u.push(n.x,n.y,n.z)}}function m(){for(let M=1;M<=i;M++)for(let b=1;b<=a;b++){let E=(a+1)*(M-1)+(b-1),T=(a+1)*M+(b-1),v=(a+1)*M+b,_=(a+1)*(M-1)+b;p.push(E,T,_),p.push(T,v,_)}}function y(){for(let M=0;M<=i;M++)for(let b=0;b<=a;b++)h.x=M/i,h.y=b/a,f.push(h.x,h.y)}}copy(t){return super.copy(t),this.parameters=Object.assign({},t.parameters),this}toJSON(){let t=super.toJSON();return t.path=this.parameters.path.toJSON(),t}static fromJSON(t){return new Su(new zs[t.path.type]().fromJSON(t.path),t.tubularSegments,t.radius,t.radialSegments,t.closed)}};function Ba(e){let t={};for(let i in e){t[i]={};for(let r in e[i]){let a=e[i][r];if(yh(a))a.isRenderTargetTexture?(Je("UniformsUtils: Textures of render targets cannot be cloned via cloneUniforms() or mergeUniforms()."),t[i][r]=null):t[i][r]=a.clone();else if(Array.isArray(a))if(yh(a[0])){let o=[];for(let s=0,n=a.length;s<n;s++)o[s]=a[s].clone();t[i][r]=o}else t[i][r]=a.slice();else t[i][r]=a}}return t}function mi(e){let t={};for(let i=0;i<e.length;i++){let r=Ba(e[i]);for(let a in r)t[a]=r[a]}return t}function yh(e){return e&&(e.isColor||e.isMatrix3||e.isMatrix4||e.isVector2||e.isVector3||e.isVector4||e.isTexture||e.isQuaternion)}function Mf(e){let t=[];for(let i=0;i<e.length;i++)t.push(e[i].clone());return t}function wu(e){let t=e.getRenderTarget();return t===null?e.outputColorSpace:t.isXRRenderTarget===!0?t.texture.colorSpace:ft.workingColorSpace}var Sf={clone:Ba,merge:mi},wf=`void main() {
	gl_Position = projectionMatrix * modelViewMatrix * vec4( position, 1.0 );
}`,Tf=`void main() {
	gl_FragColor = vec4( 1.0, 0.0, 0.0, 1.0 );
}`,$i=class extends Zs{constructor(e){super(),this.isShaderMaterial=!0,this.type="ShaderMaterial",this.defines={},this.uniforms={},this.uniformsGroups=[],this.vertexShader=wf,this.fragmentShader=Tf,this.linewidth=1,this.wireframe=!1,this.wireframeLinewidth=1,this.fog=!1,this.lights=!1,this.clipping=!1,this.forceSinglePass=!0,this.extensions={clipCullDistance:!1,multiDraw:!1},this.defaultAttributeValues={color:[1,1,1],uv:[0,0],uv1:[0,0]},this.index0AttributeName=void 0,this.uniformsNeedUpdate=!1,this.glslVersion=null,e!==void 0&&this.setValues(e)}copy(e){return super.copy(e),this.fragmentShader=e.fragmentShader,this.vertexShader=e.vertexShader,this.uniforms=Ba(e.uniforms),this.uniformsGroups=Mf(e.uniformsGroups),this.defines=Object.assign({},e.defines),this.wireframe=e.wireframe,this.wireframeLinewidth=e.wireframeLinewidth,this.fog=e.fog,this.lights=e.lights,this.clipping=e.clipping,this.extensions=Object.assign({},e.extensions),this.glslVersion=e.glslVersion,this.defaultAttributeValues=Object.assign({},e.defaultAttributeValues),this.index0AttributeName=e.index0AttributeName,this.uniformsNeedUpdate=e.uniformsNeedUpdate,this}toJSON(e){let t=super.toJSON(e);t.glslVersion=this.glslVersion,t.uniforms={};for(let r in this.uniforms){let a=this.uniforms[r].value;a&&a.isTexture?t.uniforms[r]={type:"t",value:a.toJSON(e).uuid}:a&&a.isColor?t.uniforms[r]={type:"c",value:a.getHex()}:a&&a.isVector2?t.uniforms[r]={type:"v2",value:a.toArray()}:a&&a.isVector3?t.uniforms[r]={type:"v3",value:a.toArray()}:a&&a.isVector4?t.uniforms[r]={type:"v4",value:a.toArray()}:a&&a.isMatrix3?t.uniforms[r]={type:"m3",value:a.toArray()}:a&&a.isMatrix4?t.uniforms[r]={type:"m4",value:a.toArray()}:t.uniforms[r]={value:a}}Object.keys(this.defines).length>0&&(t.defines=this.defines),t.vertexShader=this.vertexShader,t.fragmentShader=this.fragmentShader,t.lights=this.lights,t.clipping=this.clipping;let i={};for(let r in this.extensions)this.extensions[r]===!0&&(i[r]=!0);return Object.keys(i).length>0&&(t.extensions=i),t}fromJSON(e,t){if(super.fromJSON(e,t),e.uniforms!==void 0)for(let i in e.uniforms){let r=e.uniforms[i];switch(this.uniforms[i]={},r.type){case"t":this.uniforms[i].value=t[r.value]||null;break;case"c":this.uniforms[i].value=new dt().setHex(r.value);break;case"v2":this.uniforms[i].value=new ge().fromArray(r.value);break;case"v3":this.uniforms[i].value=new I().fromArray(r.value);break;case"v4":this.uniforms[i].value=new Ft().fromArray(r.value);break;case"m3":this.uniforms[i].value=new it().fromArray(r.value);break;case"m4":this.uniforms[i].value=new gt().fromArray(r.value);break;default:this.uniforms[i].value=r.value}}if(e.defines!==void 0&&(this.defines=e.defines),e.vertexShader!==void 0&&(this.vertexShader=e.vertexShader),e.fragmentShader!==void 0&&(this.fragmentShader=e.fragmentShader),e.glslVersion!==void 0&&(this.glslVersion=e.glslVersion),e.extensions!==void 0)for(let i in e.extensions)this.extensions[i]=e.extensions[i];return e.lights!==void 0&&(this.lights=e.lights),e.clipping!==void 0&&(this.clipping=e.clipping),this}},Tu=class extends $i{constructor(e){super(e),this.isRawShaderMaterial=!0,this.type="RawShaderMaterial"}},Ef=class extends Zs{constructor(e){super(),this.isMeshDepthMaterial=!0,this.type="MeshDepthMaterial",this.depthPacking=t0,this.map=null,this.alphaMap=null,this.displacementMap=null,this.displacementScale=1,this.displacementBias=0,this.wireframe=!1,this.wireframeLinewidth=1,this.setValues(e)}copy(e){return super.copy(e),this.depthPacking=e.depthPacking,this.map=e.map,this.alphaMap=e.alphaMap,this.displacementMap=e.displacementMap,this.displacementScale=e.displacementScale,this.displacementBias=e.displacementBias,this.wireframe=e.wireframe,this.wireframeLinewidth=e.wireframeLinewidth,this}},Af=class extends Zs{constructor(e){super(),this.isMeshDistanceMaterial=!0,this.type="MeshDistanceMaterial",this.map=null,this.alphaMap=null,this.displacementMap=null,this.displacementScale=1,this.displacementBias=0,this.setValues(e)}copy(e){return super.copy(e),this.map=e.map,this.alphaMap=e.alphaMap,this.displacementMap=e.displacementMap,this.displacementScale=e.displacementScale,this.displacementBias=e.displacementBias,this}};function ya(e,t){return!e||e.constructor===t?e:typeof t.BYTES_PER_ELEMENT=="number"?new t(e):Array.prototype.slice.call(e)}function as(e){return e!==void 0&&e.inTangents!==void 0&&e.outTangents!==void 0}var Io=class{constructor(e,t,i,r){this.parameterPositions=e,this._cachedIndex=0,this.resultBuffer=r===void 0?new t.constructor(i):r,this.sampleValues=t,this.valueSize=i,this.settings=null,this.DefaultSettings_={}}evaluate(e){let t=this.parameterPositions,i=this._cachedIndex,r=t[i],a=t[i-1];i:{e:{let o;t:{r:if(!(e<r)){for(let s=i+2;;){if(r===void 0){if(e<a)break r;return i=t.length,this._cachedIndex=i,this.copySampleValue_(i-1)}if(i===s)break;if(a=r,r=t[++i],e<r)break e}o=t.length;break t}if(!(e>=a)){let s=t[1];e<s&&(i=2,a=s);for(let n=i-2;;){if(a===void 0)return this._cachedIndex=0,this.copySampleValue_(0);if(i===n)break;if(r=a,a=t[--i-1],e>=a)break e}o=i,i=0;break t}break i}for(;i<o;){let s=i+o>>>1;e<t[s]?o=s:i=s+1}if(r=t[i],a=t[i-1],a===void 0)return this._cachedIndex=0,this.copySampleValue_(0);if(r===void 0)return i=t.length,this._cachedIndex=i,this.copySampleValue_(i-1)}this._cachedIndex=i,this.intervalChanged_(i,a,r)}return this.interpolate_(i,a,e,r)}getSettings_(){return this.settings||this.DefaultSettings_}copySampleValue_(e){let t=this.resultBuffer,i=this.sampleValues,r=this.valueSize,a=e*r;for(let o=0;o!==r;++o)t[o]=i[a+o];return t}interpolate_(){throw Error("THREE.Interpolant: Call to abstract method.")}intervalChanged_(){}},Rf=class extends Io{constructor(e,t,i,r){super(e,t,i,r),this._weightPrev=-0,this._offsetPrev=-0,this._weightNext=-0,this._offsetNext=-0,this.DefaultSettings_={endingStart:Xl,endingEnd:Xl}}intervalChanged_(e,t,i){let r=this.parameterPositions,a=e-2,o=e+1,s=r[a],n=r[o];if(s===void 0)switch(this.getSettings_().endingStart){case ql:a=e,s=2*t-i;break;case jl:a=r.length-2,s=t+r[a]-r[a+1];break;default:a=e,s=i}if(n===void 0)switch(this.getSettings_().endingEnd){case ql:o=e,n=2*i-t;break;case jl:o=1,n=i+r[1]-r[0];break;default:o=e-1,n=t}let l=(i-t)*.5,h=this.valueSize;this._weightPrev=l/(t-s),this._weightNext=l/(n-i),this._offsetPrev=a*h,this._offsetNext=o*h}interpolate_(e,t,i,r){let a=this.resultBuffer,o=this.sampleValues,s=this.valueSize,n=e*s,l=n-s,h=this._offsetPrev,c=this._offsetNext,u=this._weightPrev,d=this._weightNext,f=(i-t)/(r-t),p=f*f,x=p*f,g=-u*x+2*u*p-u*f,m=(1+u)*x+(-1.5-2*u)*p+(-.5+u)*f+1,y=(-1-d)*x+(1.5+d)*p+.5*f,M=d*x-d*p;for(let b=0;b!==s;++b)a[b]=g*o[h+b]+m*o[l+b]+y*o[n+b]+M*o[c+b];return a}},Cf=class extends Io{constructor(e,t,i,r){super(e,t,i,r)}interpolate_(e,t,i,r){let a=this.resultBuffer,o=this.sampleValues,s=this.valueSize,n=e*s,l=n-s,h=(i-t)/(r-t),c=1-h;for(let u=0;u!==s;++u)a[u]=o[l+u]*c+o[n+u]*h;return a}},Pf=class extends Io{constructor(e,t,i,r){super(e,t,i,r)}interpolate_(e){return this.copySampleValue_(e-1)}},Lf=class extends Io{interpolate_(e,t,i,r){let a=this.resultBuffer,o=this.sampleValues,s=this.valueSize,n=e*s,l=n-s,h=this.inTangents,c=this.outTangents;if(!h||!c){let f=(i-t)/(r-t),p=1-f;for(let x=0;x!==s;++x)a[x]=o[l+x]*p+o[n+x]*f;return a}let u=s*2,d=e-1;for(let f=0;f!==s;++f){let p=o[l+f],x=o[n+f],g=d*u+f*2,m=c[g],y=c[g+1],M=e*u+f*2,b=h[M],E=h[M+1],T=If(i,t,m,b,r);a[f]=Eu(T,p,y,E,x)}return a}};function Eu(e,t,i,r,a){let o=1-e;return o*o*o*t+3*o*o*e*i+3*o*e*e*r+e*e*e*a}function Nf(e,t,i,r,a){let o=1-e;return 3*o*o*(i-t)+6*o*e*(r-i)+3*e*e*(a-r)}function If(e,t,i,r,a){let o=(e-t)/(a-t);for(let s=0;s<8;s++){let n=Eu(o,t,i,r,a)-e;if(Math.abs(n)<1e-10)break;let l=Nf(o,t,i,r,a);if(Math.abs(l)<1e-10)break;o=Math.max(0,Math.min(1,o-n/l))}return o}var lr=class{constructor(e,t,i,r){if(e===void 0)throw Error("THREE.KeyframeTrack: track name is undefined");if(t===void 0||t.length===0)throw Error("THREE.KeyframeTrack: no keyframes in track named "+e);this.name=e,this.times=ya(t,this.TimeBufferType),this.values=ya(i,this.ValueBufferType),this.setInterpolation(r||this.DefaultInterpolation)}static toJSON(e){let t=e.constructor,i;if(t.toJSON!==this.toJSON)i=t.toJSON(e);else{i={name:e.name,times:ya(e.times,Array),values:ya(e.values,Array)};let r=e.getInterpolation();r!==e.DefaultInterpolation&&(i.interpolation=r),as(e.settings)&&(i.settings={inTangents:ya(e.settings.inTangents,Array),outTangents:ya(e.settings.outTangents,Array)})}return i.type=e.ValueTypeName,i}InterpolantFactoryMethodDiscrete(e){return new Pf(this.times,this.values,this.getValueSize(),e)}InterpolantFactoryMethodLinear(e){return new Cf(this.times,this.values,this.getValueSize(),e)}InterpolantFactoryMethodSmooth(e){return new Rf(this.times,this.values,this.getValueSize(),e)}InterpolantFactoryMethodBezier(e){let t=new Lf(this.times,this.values,this.getValueSize(),e);return this.settings&&(t.inTangents=this.settings.inTangents,t.outTangents=this.settings.outTangents),t}setInterpolation(e){let t;switch(e){case Ls:t=this.InterpolantFactoryMethodDiscrete;break;case Jn:t=this.InterpolantFactoryMethodLinear;break;case rn:t=this.InterpolantFactoryMethodSmooth;break;case Wl:t=this.InterpolantFactoryMethodBezier}if(t===void 0){let i="unsupported interpolation for "+this.ValueTypeName+" keyframe track named "+this.name;if(this.createInterpolant===void 0)if(e!==this.DefaultInterpolation)this.setInterpolation(this.DefaultInterpolation);else throw Error(i);return Je("KeyframeTrack:",i),this}return this.createInterpolant=t,this}getInterpolation(){switch(this.createInterpolant){case this.InterpolantFactoryMethodDiscrete:return Ls;case this.InterpolantFactoryMethodLinear:return Jn;case this.InterpolantFactoryMethodSmooth:return rn;case this.InterpolantFactoryMethodBezier:return Wl}}getValueSize(){return this.values.length/this.times.length}shift(e){if(e!==0){let t=this.times;for(let i=0,r=t.length;i!==r;++i)t[i]+=e;as(this.settings)&&(bh(this.settings.inTangents,e),bh(this.settings.outTangents,e))}return this}scale(e){if(e!==1){let t=this.times;for(let i=0,r=t.length;i!==r;++i)t[i]*=e;as(this.settings)&&(Mh(this.settings.inTangents,e),Mh(this.settings.outTangents,e))}return this}trim(e,t){let i=this.times,r=i.length,a=0,o=r-1;for(;a!==r&&i[a]<e;)++a;for(;o!==-1&&i[o]>t;)--o;if(++o,a!==0||o!==r){a>=o&&(o=Math.max(o,1),a=o-1);let s=this.getValueSize();this.times=i.slice(a,o),this.values=this.values.slice(a*s,o*s)}return this}validate(){let e=!0,t=this.getValueSize();t-Math.floor(t)!==0&&(Qe("KeyframeTrack: Invalid value size in track.",this),e=!1);let i=this.times,r=this.values,a=i.length;a===0&&(Qe("KeyframeTrack: Track is empty.",this),e=!1);let o=null;for(let s=0;s!==a;s++){let n=i[s];if(typeof n=="number"&&isNaN(n)){Qe("KeyframeTrack: Time is not a valid number.",this,s,n),e=!1;break}if(o!==null&&o>n){Qe("KeyframeTrack: Out of order keys.",this,s,n,o),e=!1;break}o=n}if(r!==void 0&&a0(r))for(let s=0,n=r.length;s!==n;++s){let l=r[s];if(isNaN(l)){Qe("KeyframeTrack: Value is not a valid number.",this,s,l),e=!1;break}}return e}optimize(){let e=this.times.slice(),t=this.values.slice(),i=this.getValueSize(),r=this.getInterpolation()===rn,a=e.length-1,o=1;for(let s=1;s<a;++s){let n=!1,l=e[s];if(l!==e[s+1]&&(s!==1||l!==e[0]))if(r)n=!0;else{let h=s*i,c=h-i,u=h+i;for(let d=0;d!==i;++d){let f=t[h+d];if(f!==t[c+d]||f!==t[u+d]){n=!0;break}}}if(n){if(s!==o){e[o]=e[s];let h=s*i,c=o*i;for(let u=0;u!==i;++u)t[c+u]=t[h+u]}++o}}if(a>0){e[o]=e[a];for(let s=a*i,n=o*i,l=0;l!==i;++l)t[n+l]=t[s+l];++o}return o===e.length?(this.times=e,this.values=t):(this.times=e.slice(0,o),this.values=t.slice(0,o*i)),this}clone(){let e=this.times.slice(),t=this.values.slice(),i=this.constructor,r=new i(this.name,e,t);return r.createInterpolant=this.createInterpolant,as(this.settings)&&(r.settings={inTangents:this.settings.inTangents.slice(),outTangents:this.settings.outTangents.slice()}),r}};function bh(e,t){for(let i=0,r=e.length;i!==r;i+=2)e[i]+=t}function Mh(e,t){for(let i=0,r=e.length;i!==r;i+=2)e[i]*=t}lr.prototype.ValueTypeName="",lr.prototype.TimeBufferType=Float32Array,lr.prototype.ValueBufferType=Float32Array,lr.prototype.DefaultInterpolation=Jn;var io=class extends lr{constructor(e,t,i){super(e,t,i)}};io.prototype.ValueTypeName="bool",io.prototype.ValueBufferType=Array,io.prototype.DefaultInterpolation=Ls,io.prototype.InterpolantFactoryMethodLinear=void 0,io.prototype.InterpolantFactoryMethodSmooth=void 0;var Df=class extends lr{constructor(e,t,i,r){super(e,t,i,r)}};Df.prototype.ValueTypeName="color";var Uf=class extends lr{constructor(e,t,i,r){super(e,t,i,r)}};Uf.prototype.ValueTypeName="number";var Of=class extends Io{constructor(e,t,i,r){super(e,t,i,r)}interpolate_(e,t,i,r){let a=this.resultBuffer,o=this.sampleValues,s=this.valueSize,n=(i-t)/(r-t),l=e*s;for(let h=l+s;l!==h;l+=4)ei.slerpFlat(a,0,o,l-s,o,l,n);return a}},Sh=class extends lr{constructor(e,t,i,r){super(e,t,i,r)}InterpolantFactoryMethodLinear(e){return new Of(this.times,this.values,this.getValueSize(),e)}};Sh.prototype.ValueTypeName="quaternion",Sh.prototype.InterpolantFactoryMethodSmooth=void 0;var ro=class extends lr{constructor(e,t,i){super(e,t,i)}};ro.prototype.ValueTypeName="string",ro.prototype.ValueBufferType=Array,ro.prototype.DefaultInterpolation=Ls,ro.prototype.InterpolantFactoryMethodLinear=void 0,ro.prototype.InterpolantFactoryMethodSmooth=void 0;var zf=class extends lr{constructor(e,t,i,r){super(e,t,i,r)}};zf.prototype.ValueTypeName="vector";var os=new I,ss=new ei,er=new I,Au=class extends Ar{constructor(){super(),this.isCamera=!0,this.type="Camera",this.matrixWorldInverse=new gt,this.projectionMatrix=new gt,this.projectionMatrixInverse=new gt,this.coordinateSystem=ia,this._reversedDepth=!1}get reversedDepth(){return this._reversedDepth}copy(e,t){return super.copy(e,t),this.matrixWorldInverse.copy(e.matrixWorldInverse),this.projectionMatrix.copy(e.projectionMatrix),this.projectionMatrixInverse.copy(e.projectionMatrixInverse),this.coordinateSystem=e.coordinateSystem,this}getWorldDirection(e){return super.getWorldDirection(e).negate()}updateMatrixWorld(e){super.updateMatrixWorld(e),this.matrixWorld.decompose(os,ss,er),er.x===1&&er.y===1&&er.z===1?this.matrixWorldInverse.copy(this.matrixWorld).invert():this.matrixWorldInverse.compose(os,ss,er.set(1,1,1)).invert()}updateWorldMatrix(e,t,i=!1){super.updateWorldMatrix(e,t,i),this.matrixWorld.decompose(os,ss,er),er.x===1&&er.y===1&&er.z===1?this.matrixWorldInverse.copy(this.matrixWorld).invert():this.matrixWorldInverse.compose(os,ss,er.set(1,1,1)).invert()}clone(){return new this.constructor().copy(this)}},Ur=new I,wh=new ge,Th=new ge,ji=class extends Au{constructor(e=50,t=1,i=.1,r=2e3){super(),this.isPerspectiveCamera=!0,this.type="PerspectiveCamera",this.fov=e,this.zoom=1,this.near=i,this.far=r,this.focus=10,this.aspect=t,this.view=null,this.filmGauge=35,this.filmOffset=0,this.updateProjectionMatrix()}copy(e,t){return super.copy(e,t),this.fov=e.fov,this.zoom=e.zoom,this.near=e.near,this.far=e.far,this.focus=e.focus,this.aspect=e.aspect,this.view=e.view===null?null:Object.assign({},e.view),this.filmGauge=e.filmGauge,this.filmOffset=e.filmOffset,this}setFocalLength(e){let t=.5*this.getFilmHeight()/e;this.fov=Kn*2*Math.atan(t),this.updateProjectionMatrix()}getFocalLength(){let e=Math.tan(on*.5*this.fov);return .5*this.getFilmHeight()/e}getEffectiveFOV(){return Kn*2*Math.atan(Math.tan(on*.5*this.fov)/this.zoom)}getFilmWidth(){return this.filmGauge*Math.min(this.aspect,1)}getFilmHeight(){return this.filmGauge/Math.max(this.aspect,1)}getViewBounds(e,t,i){Ur.set(-1,-1,.5).applyMatrix4(this.projectionMatrixInverse),t.set(Ur.x,Ur.y).multiplyScalar(-e/Ur.z),Ur.set(1,1,.5).applyMatrix4(this.projectionMatrixInverse),i.set(Ur.x,Ur.y).multiplyScalar(-e/Ur.z)}getViewSize(e,t){return this.getViewBounds(e,wh,Th),t.subVectors(Th,wh)}setViewOffset(e,t,i,r,a,o){this.aspect=e/t,this.view===null&&(this.view={enabled:!0,fullWidth:1,fullHeight:1,offsetX:0,offsetY:0,width:1,height:1}),this.view.enabled=!0,this.view.fullWidth=e,this.view.fullHeight=t,this.view.offsetX=i,this.view.offsetY=r,this.view.width=a,this.view.height=o,this.updateProjectionMatrix()}clearViewOffset(){this.view!==null&&(this.view.enabled=!1),this.updateProjectionMatrix()}updateProjectionMatrix(){let e=this.near,t=e*Math.tan(on*.5*this.fov)/this.zoom,i=2*t,r=this.aspect*i,a=-.5*r,o=this.view;if(this.view!==null&&this.view.enabled){let n=o.fullWidth,l=o.fullHeight;a+=o.offsetX*r/n,t-=o.offsetY*i/l,r*=o.width/n,i*=o.height/l}let s=this.filmOffset;s!==0&&(a+=e*s/this.getFilmWidth()),this.projectionMatrix.makePerspective(a,a+r,t,t-i,e,this.far,this.coordinateSystem,this.reversedDepth),this.projectionMatrixInverse.copy(this.projectionMatrix).invert()}toJSON(e){let t=super.toJSON(e);return t.object.fov=this.fov,t.object.zoom=this.zoom,t.object.near=this.near,t.object.far=this.far,t.object.focus=this.focus,t.object.aspect=this.aspect,this.view!==null&&(t.object.view=Object.assign({},this.view)),t.object.filmGauge=this.filmGauge,t.object.filmOffset=this.filmOffset,t}},el=class extends Au{constructor(e=-1,t=1,i=1,r=-1,a=.1,o=2e3){super(),this.isOrthographicCamera=!0,this.type="OrthographicCamera",this.zoom=1,this.view=null,this.left=e,this.right=t,this.top=i,this.bottom=r,this.near=a,this.far=o,this.updateProjectionMatrix()}copy(e,t){return super.copy(e,t),this.left=e.left,this.right=e.right,this.top=e.top,this.bottom=e.bottom,this.near=e.near,this.far=e.far,this.zoom=e.zoom,this.view=e.view===null?null:Object.assign({},e.view),this}setViewOffset(e,t,i,r,a,o){this.view===null&&(this.view={enabled:!0,fullWidth:1,fullHeight:1,offsetX:0,offsetY:0,width:1,height:1}),this.view.enabled=!0,this.view.fullWidth=e,this.view.fullHeight=t,this.view.offsetX=i,this.view.offsetY=r,this.view.width=a,this.view.height=o,this.updateProjectionMatrix()}clearViewOffset(){this.view!==null&&(this.view.enabled=!1),this.updateProjectionMatrix()}updateProjectionMatrix(){let e=(this.right-this.left)/(2*this.zoom),t=(this.top-this.bottom)/(2*this.zoom),i=(this.right+this.left)/2,r=(this.top+this.bottom)/2,a=i-e,o=i+e,s=r+t,n=r-t;if(this.view!==null&&this.view.enabled){let l=(this.right-this.left)/this.view.fullWidth/this.zoom,h=(this.top-this.bottom)/this.view.fullHeight/this.zoom;a+=l*this.view.offsetX,o=a+l*this.view.width,s-=h*this.view.offsetY,n=s-h*this.view.height}this.projectionMatrix.makeOrthographic(a,o,s,n,this.near,this.far,this.coordinateSystem,this.reversedDepth),this.projectionMatrixInverse.copy(this.projectionMatrix).invert()}toJSON(e){let t=super.toJSON(e);return t.object.zoom=this.zoom,t.object.left=this.left,t.object.right=this.right,t.object.top=this.top,t.object.bottom=this.bottom,t.object.near=this.near,t.object.far=this.far,this.view!==null&&(t.object.view=Object.assign({},this.view)),t}},ba=-90,Ma=1,tl=class extends Ar{constructor(e,t,i){super(),this.type="CubeCamera",this.renderTarget=i,this.coordinateSystem=null,this.activeMipmapLevel=0;let r=new ji(ba,Ma,e,t);r.layers=this.layers,this.add(r);let a=new ji(ba,Ma,e,t);a.layers=this.layers,this.add(a);let o=new ji(ba,Ma,e,t);o.layers=this.layers,this.add(o);let s=new ji(ba,Ma,e,t);s.layers=this.layers,this.add(s);let n=new ji(ba,Ma,e,t);n.layers=this.layers,this.add(n);let l=new ji(ba,Ma,e,t);l.layers=this.layers,this.add(l)}updateCoordinateSystem(){let e=this.coordinateSystem,t=this.children.concat(),[i,r,a,o,s,n]=t;for(let l of t)this.remove(l);if(e===2e3)i.up.set(0,1,0),i.lookAt(1,0,0),r.up.set(0,1,0),r.lookAt(-1,0,0),a.up.set(0,0,-1),a.lookAt(0,1,0),o.up.set(0,0,1),o.lookAt(0,-1,0),s.up.set(0,1,0),s.lookAt(0,0,1),n.up.set(0,1,0),n.lookAt(0,0,-1);else if(e===2001)i.up.set(0,-1,0),i.lookAt(-1,0,0),r.up.set(0,-1,0),r.lookAt(1,0,0),a.up.set(0,0,1),a.lookAt(0,1,0),o.up.set(0,0,-1),o.lookAt(0,-1,0),s.up.set(0,-1,0),s.lookAt(0,0,1),n.up.set(0,-1,0),n.lookAt(0,0,-1);else throw Error("THREE.CubeCamera.updateCoordinateSystem(): Invalid coordinate system: "+e);for(let l of t)this.add(l),l.updateMatrixWorld()}update(e,t){this.parent===null&&this.updateMatrixWorld();let{renderTarget:i,activeMipmapLevel:r}=this;this.coordinateSystem!==e.coordinateSystem&&(this.coordinateSystem=e.coordinateSystem,this.updateCoordinateSystem());let[a,o,s,n,l,h]=this.children,c=e.getRenderTarget(),u=e.getActiveCubeFace(),d=e.getActiveMipmapLevel(),f=e.xr.enabled;e.xr.enabled=!1;let p=i.texture.mipmapsAutoUpdate;i.texture.mipmapsAutoUpdate=!1;let x=!1;x=e.isWebGLRenderer===!0?e.state.buffers.depth.getReversed():e.reversedDepthBuffer,e.setRenderTarget(i,0,r),x&&e.autoClear===!1&&e.clearDepth(),e.render(t,a),e.setRenderTarget(i,1,r),x&&e.autoClear===!1&&e.clearDepth(),e.render(t,o),e.setRenderTarget(i,2,r),x&&e.autoClear===!1&&e.clearDepth(),e.render(t,s),e.setRenderTarget(i,3,r),x&&e.autoClear===!1&&e.clearDepth(),e.render(t,n),e.setRenderTarget(i,4,r),x&&e.autoClear===!1&&e.clearDepth(),e.render(t,l),i.texture.mipmapsAutoUpdate=p,e.setRenderTarget(i,5,r),x&&e.autoClear===!1&&e.clearDepth(),e.render(t,h),e.setRenderTarget(c,u,d),e.xr.enabled=f,i.texture.needsPMREMUpdate=!0}},Ff=class extends ji{constructor(e=[]){super(),this.isArrayCamera=!0,this.isMultiViewCamera=!1,this.cameras=e,this.perCameraFrustumCulled=!1}},Bf="\\[\\]\\.:\\/",kf=RegExp("[\\[\\]\\.:\\/]","g"),Ml="[^\\[\\]\\.:\\/]",Vf="[^"+Bf.replace("\\.","")+"]",Gf="((?:WC+[\\/:])*)".replace("WC",Ml),Hf="(WCOD+)?".replace("WCOD",Vf),Wf="(?:\\.(WC+)(?:\\[(.+)\\])?)?".replace("WC",Ml),Xf="\\.(WC+)(?:\\[(.+)\\])?".replace("WC",Ml),qf=RegExp("^"+Gf+Hf+Wf+Xf+"$"),jf=["material","materials","bones","map"],Yf=class{constructor(e,t,i){let r=i||Ht.parseTrackName(t);this._targetGroup=e,this._bindings=e.subscribe_(t,r)}getValue(e,t){this.bind();let i=this._targetGroup.nCachedObjects_,r=this._bindings[i];r!==void 0&&r.getValue(e,t)}setValue(e,t){let i=this._bindings;for(let r=this._targetGroup.nCachedObjects_,a=i.length;r!==a;++r)i[r].setValue(e,t)}bind(){let e=this._bindings;for(let t=this._targetGroup.nCachedObjects_,i=e.length;t!==i;++t)e[t].bind()}unbind(){let e=this._bindings;for(let t=this._targetGroup.nCachedObjects_,i=e.length;t!==i;++t)e[t].unbind()}},Ht=class Pa{constructor(t,i,r){this.path=i,this.parsedPath=r||Pa.parseTrackName(i),this.node=Pa.findNode(t,this.parsedPath.nodeName),this.rootNode=t,this.getValue=this._getValue_unbound,this.setValue=this._setValue_unbound}static create(t,i,r){return t&&t.isAnimationObjectGroup?new Pa.Composite(t,i,r):new Pa(t,i,r)}static sanitizeNodeName(t){return t.replace(/\s/g,"_").replace(kf,"")}static parseTrackName(t){let i=qf.exec(t);if(i===null)throw Error("THREE.PropertyBinding: Cannot parse trackName: "+t);let r={nodeName:i[2],objectName:i[3],objectIndex:i[4],propertyName:i[5],propertyIndex:i[6]},a=r.nodeName&&r.nodeName.lastIndexOf(".");if(a!==void 0&&a!==-1){let o=r.nodeName.substring(a+1);jf.indexOf(o)!==-1&&(r.nodeName=r.nodeName.substring(0,a),r.objectName=o)}if(r.propertyName===null||r.propertyName.length===0)throw Error("THREE.PropertyBinding: can not parse propertyName from trackName: "+t);return r}static findNode(t,i){if(i===void 0||i===""||i==="."||i===-1||i===t.name||i===t.uuid)return t;if(t.skeleton){let r=t.skeleton.getBoneByName(i);if(r!==void 0)return r}if(t.children){let r=function(o){for(let s=0;s<o.length;s++){let n=o[s];if(n.name===i||n.uuid===i)return n;let l=r(n.children);if(l)return l}return null},a=r(t.children);if(a)return a}return null}_getValue_unavailable(){}_setValue_unavailable(){}_getValue_direct(t,i){t[i]=this.targetObject[this.propertyName]}_getValue_array(t,i){let r=this.resolvedProperty;for(let a=0,o=r.length;a!==o;++a)t[i++]=r[a]}_getValue_arrayElement(t,i){t[i]=this.resolvedProperty[this.propertyIndex]}_getValue_toArray(t,i){this.resolvedProperty.toArray(t,i)}_setValue_direct(t,i){this.targetObject[this.propertyName]=t[i]}_setValue_direct_setNeedsUpdate(t,i){this.targetObject[this.propertyName]=t[i],this.targetObject.needsUpdate=!0}_setValue_direct_setMatrixWorldNeedsUpdate(t,i){this.targetObject[this.propertyName]=t[i],this.targetObject.matrixWorldNeedsUpdate=!0}_setValue_array(t,i){let r=this.resolvedProperty;for(let a=0,o=r.length;a!==o;++a)r[a]=t[i++]}_setValue_array_setNeedsUpdate(t,i){let r=this.resolvedProperty;for(let a=0,o=r.length;a!==o;++a)r[a]=t[i++];this.targetObject.needsUpdate=!0}_setValue_array_setMatrixWorldNeedsUpdate(t,i){let r=this.resolvedProperty;for(let a=0,o=r.length;a!==o;++a)r[a]=t[i++];this.targetObject.matrixWorldNeedsUpdate=!0}_setValue_arrayElement(t,i){this.resolvedProperty[this.propertyIndex]=t[i]}_setValue_arrayElement_setNeedsUpdate(t,i){this.resolvedProperty[this.propertyIndex]=t[i],this.targetObject.needsUpdate=!0}_setValue_arrayElement_setMatrixWorldNeedsUpdate(t,i){this.resolvedProperty[this.propertyIndex]=t[i],this.targetObject.matrixWorldNeedsUpdate=!0}_setValue_fromArray(t,i){this.resolvedProperty.fromArray(t,i)}_setValue_fromArray_setNeedsUpdate(t,i){this.resolvedProperty.fromArray(t,i),this.targetObject.needsUpdate=!0}_setValue_fromArray_setMatrixWorldNeedsUpdate(t,i){this.resolvedProperty.fromArray(t,i),this.targetObject.matrixWorldNeedsUpdate=!0}_getValue_unbound(t,i){this.bind(),this.getValue(t,i)}_setValue_unbound(t,i){this.bind(),this.setValue(t,i)}bind(){let t=this.node,i=this.parsedPath,r=i.objectName,a=i.propertyName,o=i.propertyIndex;if(t||(t=Pa.findNode(this.rootNode,i.nodeName),this.node=t),this.getValue=this._getValue_unavailable,this.setValue=this._setValue_unavailable,!t){Je("PropertyBinding: No target node found for track: "+this.path+".");return}if(r){let h=i.objectIndex;switch(r){case"materials":if(!t.material){Qe("PropertyBinding: Can not bind to material as node does not have a material.",this);return}if(!t.material.materials){Qe("PropertyBinding: Can not bind to material.materials as node.material does not have a materials array.",this);return}t=t.material.materials;break;case"bones":if(!t.skeleton){Qe("PropertyBinding: Can not bind to bones as node does not have a skeleton.",this);return}t=t.skeleton.bones;for(let c=0;c<t.length;c++)if(t[c].name===h){h=c;break}break;case"map":if("map"in t){t=t.map;break}if(!t.material){Qe("PropertyBinding: Can not bind to material as node does not have a material.",this);return}if(!t.material.map){Qe("PropertyBinding: Can not bind to material.map as node.material does not have a map.",this);return}t=t.material.map;break;default:if(t[r]===void 0){Qe("PropertyBinding: Can not bind to objectName of node undefined.",this);return}t=t[r]}if(h!==void 0){if(t[h]===void 0){Qe("PropertyBinding: Trying to bind to objectIndex of objectName, but is undefined.",this,t);return}t=t[h]}}let s=t[a];if(s===void 0){let h=i.nodeName;Qe("PropertyBinding: Trying to update property for track: "+h+"."+a+" but it wasn't found.",t);return}let n=this.Versioning.None;this.targetObject=t,t.isMaterial===!0?n=this.Versioning.NeedsUpdate:t.isObject3D===!0&&(n=this.Versioning.MatrixWorldNeedsUpdate);let l=this.BindingType.Direct;if(o!==void 0){if(a==="morphTargetInfluences"){if(!t.geometry){Qe("PropertyBinding: Can not bind to morphTargetInfluences because node does not have a geometry.",this);return}if(!t.geometry.morphAttributes){Qe("PropertyBinding: Can not bind to morphTargetInfluences because node does not have a geometry.morphAttributes.",this);return}t.morphTargetDictionary[o]!==void 0&&(o=t.morphTargetDictionary[o])}l=this.BindingType.ArrayElement,this.resolvedProperty=s,this.propertyIndex=o}else s.fromArray!==void 0&&s.toArray!==void 0?(l=this.BindingType.HasFromToArray,this.resolvedProperty=s):Array.isArray(s)?(l=this.BindingType.EntireArray,this.resolvedProperty=s):this.propertyName=a;this.getValue=this.GetterByBindingType[l],this.setValue=this.SetterByBindingTypeAndVersioning[l][n]}unbind(){this.node=null,this.getValue=this._getValue_unbound,this.setValue=this._setValue_unbound}};Ht.Composite=Yf,Ht.prototype.BindingType={Direct:0,EntireArray:1,ArrayElement:2,HasFromToArray:3},Ht.prototype.Versioning={None:0,NeedsUpdate:1,MatrixWorldNeedsUpdate:2},Ht.prototype.GetterByBindingType=[Ht.prototype._getValue_direct,Ht.prototype._getValue_array,Ht.prototype._getValue_arrayElement,Ht.prototype._getValue_toArray],Ht.prototype.SetterByBindingTypeAndVersioning=[[Ht.prototype._setValue_direct,Ht.prototype._setValue_direct_setNeedsUpdate,Ht.prototype._setValue_direct_setMatrixWorldNeedsUpdate],[Ht.prototype._setValue_array,Ht.prototype._setValue_array_setNeedsUpdate,Ht.prototype._setValue_array_setMatrixWorldNeedsUpdate],[Ht.prototype._setValue_arrayElement,Ht.prototype._setValue_arrayElement_setNeedsUpdate,Ht.prototype._setValue_arrayElement_setMatrixWorldNeedsUpdate],[Ht.prototype._setValue_fromArray,Ht.prototype._setValue_fromArray_setNeedsUpdate,Ht.prototype._setValue_fromArray_setMatrixWorldNeedsUpdate]],class{static{this.prototype.isMatrix2=!0}constructor(e,t,i,r){this.elements=[1,0,0,1],e!==void 0&&this.set(e,t,i,r)}identity(){return this.set(1,0,0,1),this}fromArray(e,t=0){for(let i=0;i<4;i++)this.elements[i]=e[i+t];return this}set(e,t,i,r){let a=this.elements;return a[0]=e,a[2]=t,a[1]=i,a[3]=r,this}};function Eh(e,t,i,r){let a=Zf(r);switch(i){case vd:return e*t;case Ys:return e*t/a.components*a.byteLength;case Uc:return e*t/a.components*a.byteLength;case Ps:return e*t*2/a.components*a.byteLength;case Oc:return e*t*2/a.components*a.byteLength;case xd:return e*t*3/a.components*a.byteLength;case yo:return e*t*4/a.components*a.byteLength;case zc:return e*t*4/a.components*a.byteLength;case _d:case yd:return Math.floor((e+3)/4)*Math.floor((t+3)/4)*8;case bd:case Md:return Math.floor((e+3)/4)*Math.floor((t+3)/4)*16;case wd:case Ed:return Math.max(e,16)*Math.max(t,8)/4;case Sd:case Td:return Math.max(e,8)*Math.max(t,8)/2;case Ad:case Rd:case Pd:case Ld:return Math.floor((e+3)/4)*Math.floor((t+3)/4)*8;case Cd:case Nd:case Id:return Math.floor((e+3)/4)*Math.floor((t+3)/4)*16;case Dd:return Math.floor((e+3)/4)*Math.floor((t+3)/4)*16;case Ud:return Math.floor((e+4)/5)*Math.floor((t+3)/4)*16;case Od:return Math.floor((e+4)/5)*Math.floor((t+4)/5)*16;case zd:return Math.floor((e+5)/6)*Math.floor((t+4)/5)*16;case Fd:return Math.floor((e+5)/6)*Math.floor((t+5)/6)*16;case Bd:return Math.floor((e+7)/8)*Math.floor((t+4)/5)*16;case kd:return Math.floor((e+7)/8)*Math.floor((t+5)/6)*16;case Vd:return Math.floor((e+7)/8)*Math.floor((t+7)/8)*16;case Gd:return Math.floor((e+9)/10)*Math.floor((t+4)/5)*16;case Hd:return Math.floor((e+9)/10)*Math.floor((t+5)/6)*16;case Wd:return Math.floor((e+9)/10)*Math.floor((t+7)/8)*16;case Xd:return Math.floor((e+9)/10)*Math.floor((t+9)/10)*16;case qd:return Math.floor((e+11)/12)*Math.floor((t+9)/10)*16;case jd:return Math.floor((e+11)/12)*Math.floor((t+11)/12)*16;case Yd:case Zd:case Jd:return Math.ceil(e/4)*Math.ceil(t/4)*16;case Kd:case $d:return Math.ceil(e/4)*Math.ceil(t/4)*8;case Qd:case e0:return Math.ceil(e/4)*Math.ceil(t/4)*16}throw Error(`Unable to determine texture byte length for ${i} format.`)}function Zf(e){switch(e){case Fr:case dd:return{byteLength:1,components:1};case Nc:case fd:case Rr:return{byteLength:2,components:1};case Ic:case Dc:return{byteLength:2,components:4};case Vr:case pd:case Co:return{byteLength:4,components:1};case md:case gd:return{byteLength:4,components:3}}throw Error(`THREE.TextureUtils: Unknown texture type ${e}.`)}typeof __THREE_DEVTOOLS__<"u"&&__THREE_DEVTOOLS__.dispatchEvent(new CustomEvent("register",{detail:{revision:"187"}})),typeof window<"u"&&(window.__THREE__?Je("WARNING: Multiple instances of Three.js being imported."):window.__THREE__="187");function Ru(){let e=null,t=!1,i=null,r=null;function a(o,s){r=e.requestAnimationFrame(a),i(o,s)}return{start:function(){t!==!0&&i!==null&&e!==null&&(r=e.requestAnimationFrame(a),t=!0)},stop:function(){e!==null&&e.cancelAnimationFrame(r),t=!1},setAnimationLoop:function(o){i=o},setContext:function(o){e=o}}}function Jf(e){let t=new WeakMap;function i(n,l){let h=n.array,c=n.usage,u=h.byteLength,d=e.createBuffer();e.bindBuffer(l,d),e.bufferData(l,h,c),n.onUploadCallback();let f;if(h instanceof Float32Array)f=e.FLOAT;else if(typeof Float16Array<"u"&&h instanceof Float16Array)f=e.HALF_FLOAT;else if(h instanceof Uint16Array)f=n.isFloat16BufferAttribute?e.HALF_FLOAT:e.UNSIGNED_SHORT;else if(h instanceof Int16Array)f=e.SHORT;else if(h instanceof Uint32Array)f=e.UNSIGNED_INT;else if(h instanceof Int32Array)f=e.INT;else if(h instanceof Int8Array)f=e.BYTE;else if(h instanceof Uint8Array)f=e.UNSIGNED_BYTE;else if(h instanceof Uint8ClampedArray)f=e.UNSIGNED_BYTE;else throw Error("THREE.WebGLAttributes: Unsupported buffer data format: "+h);return{buffer:d,type:f,bytesPerElement:h.BYTES_PER_ELEMENT,version:n.version,size:u}}function r(n,l,h){let c=l.array,u=l.updateRanges;if(e.bindBuffer(h,n),u.length===0)e.bufferSubData(h,0,c);else{u.sort((f,p)=>f.start-p.start);let d=0;for(let f=1;f<u.length;f++){let p=u[d],x=u[f];x.start<=p.start+p.count+1?p.count=Math.max(p.count,x.start+x.count-p.start):(++d,u[d]=x)}u.length=d+1;for(let f=0,p=u.length;f<p;f++){let x=u[f];e.bufferSubData(h,x.start*c.BYTES_PER_ELEMENT,c,x.start,x.count)}l.clearUpdateRanges()}l.onUploadCallback()}function a(n){return n.isInterleavedBufferAttribute&&(n=n.data),t.get(n)}function o(n){n.isInterleavedBufferAttribute&&(n=n.data);let l=t.get(n);l&&(e.deleteBuffer(l.buffer),t.delete(n))}function s(n,l){if(n.isInterleavedBufferAttribute&&(n=n.data),n.isGLBufferAttribute){let c=t.get(n);(!c||c.version<n.version)&&t.set(n,{buffer:n.buffer,type:n.type,bytesPerElement:n.elementSize,version:n.version});return}let h=t.get(n);if(h===void 0)t.set(n,i(n,l));else if(h.version<n.version){if(h.size!==n.array.byteLength)throw Error("THREE.WebGLAttributes: The size of the buffer attribute's array buffer does not match the original size. Resizing buffer attributes is not supported.");r(h.buffer,n,l),h.version=n.version}}return{get:a,remove:o,update:s}}var ot={alphahash_fragment:`#ifdef USE_ALPHAHASH
	if ( diffuseColor.a < getAlphaHashThreshold( vPosition ) ) discard;
#endif`,alphahash_pars_fragment:`#ifdef USE_ALPHAHASH
	const float ALPHA_HASH_SCALE = 0.05;
	float hash2D( vec2 value ) {
		return fract( 1.0e4 * sin( 17.0 * value.x + 0.1 * value.y ) * ( 0.1 + abs( sin( 13.0 * value.y + value.x ) ) ) );
	}
	float hash3D( vec3 value ) {
		return hash2D( vec2( hash2D( value.xy ), value.z ) );
	}
	float getAlphaHashThreshold( vec3 position ) {
		float maxDeriv = max(
			length( dFdx( position.xyz ) ),
			length( dFdy( position.xyz ) )
		);
		float pixScale = 1.0 / ( ALPHA_HASH_SCALE * maxDeriv );
		vec2 pixScales = vec2(
			exp2( floor( log2( pixScale ) ) ),
			exp2( ceil( log2( pixScale ) ) )
		);
		vec2 alpha = vec2(
			hash3D( floor( pixScales.x * position.xyz ) ),
			hash3D( floor( pixScales.y * position.xyz ) )
		);
		float lerpFactor = fract( log2( pixScale ) );
		float x = ( 1.0 - lerpFactor ) * alpha.x + lerpFactor * alpha.y;
		float a = min( lerpFactor, 1.0 - lerpFactor );
		vec3 cases = vec3(
			x * x / ( 2.0 * a * ( 1.0 - a ) ),
			( x - 0.5 * a ) / ( 1.0 - a ),
			1.0 - ( ( 1.0 - x ) * ( 1.0 - x ) / ( 2.0 * a * ( 1.0 - a ) ) )
		);
		float threshold = ( x < ( 1.0 - a ) )
			? ( ( x < a ) ? cases.x : cases.y )
			: cases.z;
		return clamp( threshold , 1.0e-6, 1.0 );
	}
#endif`,alphamap_fragment:`#ifdef USE_ALPHAMAP
	diffuseColor.a *= texture2D( alphaMap, vAlphaMapUv ).g;
#endif`,alphamap_pars_fragment:`#ifdef USE_ALPHAMAP
	uniform sampler2D alphaMap;
#endif`,alphatest_fragment:`#ifdef USE_ALPHATEST
	#ifdef ALPHA_TO_COVERAGE
	diffuseColor.a = smoothstep( alphaTest, alphaTest + fwidth( diffuseColor.a ), diffuseColor.a );
	if ( diffuseColor.a == 0.0 ) discard;
	#else
	if ( diffuseColor.a < alphaTest ) discard;
	#endif
#endif`,alphatest_pars_fragment:`#ifdef USE_ALPHATEST
	uniform float alphaTest;
#endif`,aomap_fragment:`#ifdef USE_AOMAP
	float ambientOcclusion = ( texture2D( aoMap, vAoMapUv ).r - 1.0 ) * aoMapIntensity + 1.0;
	reflectedLight.indirectDiffuse *= ambientOcclusion;
	#if defined( USE_CLEARCOAT ) 
		clearcoatSpecularIndirect *= ambientOcclusion;
	#endif
	#if defined( USE_SHEEN ) 
		sheenSpecularIndirect *= ambientOcclusion;
	#endif
	#if defined( USE_ENVMAP ) && defined( STANDARD )
		float dotNV = saturate( dot( geometryNormal, geometryViewDir ) );
		reflectedLight.indirectSpecular *= computeSpecularOcclusion( dotNV, ambientOcclusion, material.roughness );
	#endif
#endif`,aomap_pars_fragment:`#ifdef USE_AOMAP
	uniform sampler2D aoMap;
	uniform float aoMapIntensity;
#endif`,batching_pars_vertex:`#ifdef USE_BATCHING
	#if ! defined( GL_ANGLE_multi_draw )
	#define gl_DrawID _gl_DrawID
	uniform int _gl_DrawID;
	#endif
	uniform highp sampler2D batchingTexture;
	uniform highp usampler2D batchingIdTexture;
	mat4 getBatchingMatrix( const in float i ) {
		int size = textureSize( batchingTexture, 0 ).x;
		int j = int( i ) * 4;
		int x = j % size;
		int y = j / size;
		vec4 v1 = texelFetch( batchingTexture, ivec2( x, y ), 0 );
		vec4 v2 = texelFetch( batchingTexture, ivec2( x + 1, y ), 0 );
		vec4 v3 = texelFetch( batchingTexture, ivec2( x + 2, y ), 0 );
		vec4 v4 = texelFetch( batchingTexture, ivec2( x + 3, y ), 0 );
		return mat4( v1, v2, v3, v4 );
	}
	float getIndirectIndex( const in int i ) {
		int size = textureSize( batchingIdTexture, 0 ).x;
		int x = i % size;
		int y = i / size;
		return float( texelFetch( batchingIdTexture, ivec2( x, y ), 0 ).r );
	}
#endif
#ifdef USE_BATCHING_COLOR
	uniform sampler2D batchingColorTexture;
	vec4 getBatchingColor( const in float i ) {
		int size = textureSize( batchingColorTexture, 0 ).x;
		int j = int( i );
		int x = j % size;
		int y = j / size;
		return texelFetch( batchingColorTexture, ivec2( x, y ), 0 );
	}
#endif`,batching_vertex:`#ifdef USE_BATCHING
	mat4 batchingMatrix = getBatchingMatrix( getIndirectIndex( gl_DrawID ) );
#endif`,begin_vertex:`vec3 transformed = vec3( position );
#ifdef USE_ALPHAHASH
	vPosition = vec3( position );
#endif`,beginnormal_vertex:`vec3 objectNormal = vec3( normal );
#ifdef USE_TANGENT
	vec3 objectTangent = vec3( tangent.xyz );
#endif`,bsdfs:`float G_BlinnPhong_Implicit( ) {
	return 0.25;
}
float D_BlinnPhong( const in float shininess, const in float dotNH ) {
	return RECIPROCAL_PI * ( shininess * 0.5 + 1.0 ) * pow( dotNH, shininess );
}
vec3 BRDF_BlinnPhong( const in vec3 lightDir, const in vec3 viewDir, const in vec3 normal, const in vec3 specularColor, const in float shininess ) {
	vec3 halfDir = normalize( lightDir + viewDir );
	float dotNH = saturate( dot( normal, halfDir ) );
	float dotVH = saturate( dot( viewDir, halfDir ) );
	vec3 F = F_Schlick( specularColor, 1.0, dotVH );
	float G = G_BlinnPhong_Implicit( );
	float D = D_BlinnPhong( shininess, dotNH );
	return F * ( G * D );
} // validated`,iridescence_fragment:`#ifdef USE_IRIDESCENCE
	const mat3 XYZ_TO_REC709 = mat3(
		 3.2404542, -0.9692660,  0.0556434,
		-1.5371385,  1.8760108, -0.2040259,
		-0.4985314,  0.0415560,  1.0572252
	);
	vec3 Fresnel0ToIor( vec3 fresnel0 ) {
		vec3 sqrtF0 = sqrt( fresnel0 );
		return ( vec3( 1.0 ) + sqrtF0 ) / ( vec3( 1.0 ) - sqrtF0 );
	}
	vec3 IorToFresnel0( vec3 transmittedIor, float incidentIor ) {
		return pow2( ( transmittedIor - vec3( incidentIor ) ) / ( transmittedIor + vec3( incidentIor ) ) );
	}
	float IorToFresnel0( float transmittedIor, float incidentIor ) {
		return pow2( ( transmittedIor - incidentIor ) / ( transmittedIor + incidentIor ));
	}
	vec3 evalSensitivity( float OPD, vec3 shift ) {
		float phase = 2.0 * PI * OPD * 1.0e-9;
		vec3 val = vec3( 5.4856e-13, 4.4201e-13, 5.2481e-13 );
		vec3 pos = vec3( 1.6810e+06, 1.7953e+06, 2.2084e+06 );
		vec3 var = vec3( 4.3278e+09, 9.3046e+09, 6.6121e+09 );
		vec3 xyz = val * sqrt( 2.0 * PI * var ) * cos( pos * phase + shift ) * exp( - pow2( phase ) * var );
		xyz.x += 9.7470e-14 * sqrt( 2.0 * PI * 4.5282e+09 ) * cos( 2.2399e+06 * phase + shift[ 0 ] ) * exp( - 4.5282e+09 * pow2( phase ) );
		xyz /= 1.0685e-7;
		vec3 rgb = XYZ_TO_REC709 * xyz;
		return rgb;
	}
	vec3 evalIridescence( float outsideIOR, float eta2, float cosTheta1, float thinFilmThickness, vec3 baseF0 ) {
		vec3 I;
		float iridescenceIOR = mix( outsideIOR, eta2, smoothstep( 0.0, 0.03, thinFilmThickness ) );
		float sinTheta2Sq = pow2( outsideIOR / iridescenceIOR ) * ( 1.0 - pow2( cosTheta1 ) );
		float cosTheta2Sq = 1.0 - sinTheta2Sq;
		if ( cosTheta2Sq < 0.0 ) {
			return vec3( 1.0 );
		}
		float cosTheta2 = sqrt( cosTheta2Sq );
		float R0 = IorToFresnel0( iridescenceIOR, outsideIOR );
		float R12 = F_Schlick( R0, 1.0, cosTheta1 );
		float T121 = 1.0 - R12;
		float phi12 = 0.0;
		if ( iridescenceIOR < outsideIOR ) phi12 = PI;
		float phi21 = PI - phi12;
		vec3 baseIOR = Fresnel0ToIor( clamp( baseF0, 0.0, 0.9999 ) );		vec3 R1 = IorToFresnel0( baseIOR, iridescenceIOR );
		vec3 R23 = F_Schlick( R1, 1.0, cosTheta2 );
		vec3 phi23 = vec3( 0.0 );
		if ( baseIOR[ 0 ] < iridescenceIOR ) phi23[ 0 ] = PI;
		if ( baseIOR[ 1 ] < iridescenceIOR ) phi23[ 1 ] = PI;
		if ( baseIOR[ 2 ] < iridescenceIOR ) phi23[ 2 ] = PI;
		float OPD = 2.0 * iridescenceIOR * thinFilmThickness * cosTheta2;
		vec3 phi = vec3( phi21 ) + phi23;
		vec3 R123 = clamp( R12 * R23, 1e-5, 0.9999 );
		vec3 r123 = sqrt( R123 );
		vec3 Rs = pow2( T121 ) * R23 / ( vec3( 1.0 ) - R123 );
		vec3 C0 = R12 + Rs;
		I = C0;
		vec3 Cm = Rs - T121;
		for ( int m = 1; m <= 2; ++ m ) {
			Cm *= r123;
			vec3 Sm = 2.0 * evalSensitivity( float( m ) * OPD, float( m ) * phi );
			I += Cm * Sm;
		}
		return max( I, vec3( 0.0 ) );
	}
#endif`,bumpmap_pars_fragment:`#ifdef USE_BUMPMAP
	uniform sampler2D bumpMap;
	uniform float bumpScale;
	vec2 dHdxy_fwd() {
		vec2 dSTdx = dFdx( vBumpMapUv );
		vec2 dSTdy = dFdy( vBumpMapUv );
		float Hll = bumpScale * texture2D( bumpMap, vBumpMapUv ).x;
		float dBx = bumpScale * texture2D( bumpMap, vBumpMapUv + dSTdx ).x - Hll;
		float dBy = bumpScale * texture2D( bumpMap, vBumpMapUv + dSTdy ).x - Hll;
		return vec2( dBx, dBy );
	}
	vec3 perturbNormalArb( vec3 surf_pos, vec3 surf_norm, vec2 dHdxy, float faceDirection ) {
		vec3 vSigmaX = normalize( dFdx( surf_pos.xyz ) );
		vec3 vSigmaY = normalize( dFdy( surf_pos.xyz ) );
		vec3 vN = surf_norm;
		vec3 R1 = cross( vSigmaY, vN );
		vec3 R2 = cross( vN, vSigmaX );
		float fDet = dot( vSigmaX, R1 ) * faceDirection;
		vec3 vGrad = sign( fDet ) * ( dHdxy.x * R1 + dHdxy.y * R2 );
		return normalize( abs( fDet ) * surf_norm - vGrad );
	}
#endif`,clipping_planes_fragment:`#if NUM_CLIPPING_PLANES > 0
	vec4 plane;
	#ifdef ALPHA_TO_COVERAGE
		float distanceToPlane, distanceGradient;
		float clipOpacity = 1.0;
		#pragma unroll_loop_start
		for ( int i = 0; i < UNION_CLIPPING_PLANES; i ++ ) {
			plane = clippingPlanes[ i ];
			distanceToPlane = - dot( vClipPosition, plane.xyz ) + plane.w;
			distanceGradient = fwidth( distanceToPlane ) / 2.0;
			clipOpacity *= smoothstep( - distanceGradient, distanceGradient, distanceToPlane );
			if ( clipOpacity == 0.0 ) discard;
		}
		#pragma unroll_loop_end
		#if UNION_CLIPPING_PLANES < NUM_CLIPPING_PLANES
			float unionClipOpacity = 1.0;
			#pragma unroll_loop_start
			for ( int i = UNION_CLIPPING_PLANES; i < NUM_CLIPPING_PLANES; i ++ ) {
				plane = clippingPlanes[ i ];
				distanceToPlane = - dot( vClipPosition, plane.xyz ) + plane.w;
				distanceGradient = fwidth( distanceToPlane ) / 2.0;
				unionClipOpacity *= 1.0 - smoothstep( - distanceGradient, distanceGradient, distanceToPlane );
			}
			#pragma unroll_loop_end
			clipOpacity *= 1.0 - unionClipOpacity;
		#endif
		diffuseColor.a *= clipOpacity;
		if ( diffuseColor.a == 0.0 ) discard;
	#else
		#pragma unroll_loop_start
		for ( int i = 0; i < UNION_CLIPPING_PLANES; i ++ ) {
			plane = clippingPlanes[ i ];
			if ( dot( vClipPosition, plane.xyz ) > plane.w ) discard;
		}
		#pragma unroll_loop_end
		#if UNION_CLIPPING_PLANES < NUM_CLIPPING_PLANES
			bool clipped = true;
			#pragma unroll_loop_start
			for ( int i = UNION_CLIPPING_PLANES; i < NUM_CLIPPING_PLANES; i ++ ) {
				plane = clippingPlanes[ i ];
				clipped = ( dot( vClipPosition, plane.xyz ) > plane.w ) && clipped;
			}
			#pragma unroll_loop_end
			if ( clipped ) discard;
		#endif
	#endif
#endif`,clipping_planes_pars_fragment:`#if NUM_CLIPPING_PLANES > 0
	varying vec3 vClipPosition;
	uniform vec4 clippingPlanes[ NUM_CLIPPING_PLANES ];
#endif`,clipping_planes_pars_vertex:`#if NUM_CLIPPING_PLANES > 0
	varying vec3 vClipPosition;
#endif`,clipping_planes_vertex:`#if NUM_CLIPPING_PLANES > 0
	vClipPosition = - mvPosition.xyz;
#endif`,color_fragment:`#if defined( USE_COLOR ) || defined( USE_COLOR_ALPHA )
	diffuseColor *= vColor;
#endif`,color_pars_fragment:`#if defined( USE_COLOR ) || defined( USE_COLOR_ALPHA )
	varying vec4 vColor;
#endif`,color_pars_vertex:`#if defined( USE_COLOR ) || defined( USE_COLOR_ALPHA ) || defined( USE_INSTANCING_COLOR ) || defined( USE_BATCHING_COLOR )
	varying vec4 vColor;
#endif`,color_vertex:`#if defined( USE_COLOR ) || defined( USE_COLOR_ALPHA ) || defined( USE_INSTANCING_COLOR ) || defined( USE_BATCHING_COLOR )
	vColor = vec4( 1.0 );
#endif
#ifdef USE_COLOR_ALPHA
	vColor *= color;
#elif defined( USE_COLOR )
	vColor.rgb *= color;
#endif
#ifdef USE_INSTANCING_COLOR
	vColor.rgb *= instanceColor.rgb;
#endif
#ifdef USE_BATCHING_COLOR
	vColor *= getBatchingColor( getIndirectIndex( gl_DrawID ) );
#endif`,common:`#define PI 3.141592653589793
#define PI2 6.283185307179586
#define PI_HALF 1.5707963267948966
#define RECIPROCAL_PI 0.3183098861837907
#define RECIPROCAL_PI2 0.15915494309189535
#define EPSILON 1e-6
#ifndef saturate
#define saturate( a ) clamp( a, 0.0, 1.0 )
#endif
#define whiteComplement( a ) ( 1.0 - saturate( a ) )
float pow2( const in float x ) { return x*x; }
vec3 pow2( const in vec3 x ) { return x*x; }
float pow3( const in float x ) { return x*x*x; }
float pow4( const in float x ) { float x2 = x*x; return x2*x2; }
float max3( const in vec3 v ) { return max( max( v.x, v.y ), v.z ); }
float average( const in vec3 v ) { return dot( v, vec3( 0.3333333 ) ); }
highp float rand( const in vec2 uv ) {
	const highp float a = 12.9898, b = 78.233, c = 43758.5453;
	highp float dt = dot( uv.xy, vec2( a,b ) ), sn = mod( dt, PI );
	return fract( sin( sn ) * c );
}
#ifdef HIGH_PRECISION
	float precisionSafeLength( vec3 v ) { return length( v ); }
#else
	float precisionSafeLength( vec3 v ) {
		float maxComponent = max3( abs( v ) );
		return length( v / maxComponent ) * maxComponent;
	}
#endif
struct IncidentLight {
	vec3 color;
	vec3 direction;
	bool visible;
};
struct ReflectedLight {
	vec3 directDiffuse;
	vec3 directSpecular;
	vec3 indirectDiffuse;
	vec3 indirectSpecular;
};
#ifdef USE_ALPHAHASH
	varying vec3 vPosition;
#endif
vec3 transformDirection( in vec3 dir, in mat4 matrix ) {
	return normalize( ( matrix * vec4( dir, 0.0 ) ).xyz );
}
#define inverseTransformDirection transformDirectionByInverseViewMatrix
vec3 transformNormalByInverseViewMatrix( in vec3 normal, in mat4 viewMatrix ) {
	return normalize( ( vec4( normal, 0.0 ) * viewMatrix ).xyz );
}
vec3 transformDirectionByInverseViewMatrix( in vec3 dir, in mat4 viewMatrix ) {
	return normalize( ( vec4( dir, 0.0 ) * viewMatrix ).xyz );
}
bool isPerspectiveMatrix( mat4 m ) {
	return m[ 2 ][ 3 ] == - 1.0;
}
vec2 equirectUv( in vec3 dir ) {
	float u = atan( dir.z, dir.x ) * RECIPROCAL_PI2 + 0.5;
	float v = asin( clamp( dir.y, - 1.0, 1.0 ) ) * RECIPROCAL_PI + 0.5;
	return vec2( u, v );
}
vec3 BRDF_Lambert( const in vec3 diffuseColor ) {
	return RECIPROCAL_PI * diffuseColor;
}
vec3 F_Schlick( const in vec3 f0, const in float f90, const in float dotVH ) {
	float fresnel = exp2( ( - 5.55473 * dotVH - 6.98316 ) * dotVH );
	return f0 * ( 1.0 - fresnel ) + ( f90 * fresnel );
}
float F_Schlick( const in float f0, const in float f90, const in float dotVH ) {
	float fresnel = exp2( ( - 5.55473 * dotVH - 6.98316 ) * dotVH );
	return f0 * ( 1.0 - fresnel ) + ( f90 * fresnel );
} // validated`,defaultnormal_vertex:`vec3 transformedNormal = objectNormal;
#ifdef USE_TANGENT
	vec3 transformedTangent = objectTangent;
#endif
#ifdef USE_BATCHING
	mat3 bm = mat3( batchingMatrix );
	transformedNormal /= vec3( dot( bm[ 0 ], bm[ 0 ] ), dot( bm[ 1 ], bm[ 1 ] ), dot( bm[ 2 ], bm[ 2 ] ) );
	transformedNormal = bm * transformedNormal;
	#ifdef USE_TANGENT
		transformedTangent = bm * transformedTangent;
	#endif
#endif
#ifdef USE_INSTANCING
	mat3 im = mat3( instanceMatrix );
	transformedNormal /= vec3( dot( im[ 0 ], im[ 0 ] ), dot( im[ 1 ], im[ 1 ] ), dot( im[ 2 ], im[ 2 ] ) );
	transformedNormal = im * transformedNormal;
	#ifdef USE_TANGENT
		transformedTangent = im * transformedTangent;
	#endif
#endif
transformedNormal = normalMatrix * transformedNormal;
#ifdef FLIP_SIDED
	transformedNormal = - transformedNormal;
#endif
#ifdef USE_TANGENT
	transformedTangent = ( modelViewMatrix * vec4( transformedTangent, 0.0 ) ).xyz;
#endif`,displacementmap_pars_vertex:`#ifdef USE_DISPLACEMENTMAP
	uniform sampler2D displacementMap;
	uniform float displacementScale;
	uniform float displacementBias;
#endif`,displacementmap_vertex:`#ifdef USE_DISPLACEMENTMAP
	transformed += normalize( objectNormal ) * ( texture2D( displacementMap, vDisplacementMapUv ).x * displacementScale + displacementBias );
#endif`,emissivemap_fragment:`#ifdef USE_EMISSIVEMAP
	vec4 emissiveColor = texture2D( emissiveMap, vEmissiveMapUv );
	#ifdef DECODE_VIDEO_TEXTURE_EMISSIVE
		emissiveColor = sRGBTransferEOTF( emissiveColor );
	#endif
	totalEmissiveRadiance *= emissiveColor.rgb;
#endif`,emissivemap_pars_fragment:`#ifdef USE_EMISSIVEMAP
	uniform sampler2D emissiveMap;
#endif`,colorspace_fragment:"gl_FragColor = linearToOutputTexel( gl_FragColor );",colorspace_pars_fragment:`vec4 LinearTransferOETF( in vec4 value ) {
	return value;
}
vec4 sRGBTransferEOTF( in vec4 value ) {
	return vec4( mix( pow( value.rgb * 0.9478672986 + vec3( 0.0521327014 ), vec3( 2.4 ) ), value.rgb * 0.0773993808, vec3( lessThanEqual( value.rgb, vec3( 0.04045 ) ) ) ), value.a );
}
vec4 sRGBTransferOETF( in vec4 value ) {
	return vec4( mix( pow( value.rgb, vec3( 0.41666 ) ) * 1.055 - vec3( 0.055 ), value.rgb * 12.92, vec3( lessThanEqual( value.rgb, vec3( 0.0031308 ) ) ) ), value.a );
}`,envmap_fragment:`#ifdef USE_ENVMAP
	#ifdef ENV_WORLDPOS
		vec3 cameraToFrag;
		if ( isOrthographic ) {
			cameraToFrag = normalize( vec3( - viewMatrix[ 0 ][ 2 ], - viewMatrix[ 1 ][ 2 ], - viewMatrix[ 2 ][ 2 ] ) );
		} else {
			cameraToFrag = normalize( vWorldPosition - cameraPosition );
		}
		vec3 worldNormal = transformNormalByInverseViewMatrix( normal, viewMatrix );
		#ifdef ENVMAP_MODE_REFLECTION
			vec3 reflectVec = reflect( cameraToFrag, worldNormal );
		#else
			vec3 reflectVec = refract( cameraToFrag, worldNormal, refractionRatio );
		#endif
	#else
		vec3 reflectVec = vReflect;
	#endif
	#ifdef ENVMAP_TYPE_CUBE
		vec4 envColor = textureCube( envMap, envMapRotation * reflectVec );
		#ifdef ENVMAP_BLENDING_MULTIPLY
			outgoingLight = mix( outgoingLight, outgoingLight * envColor.xyz, specularStrength * reflectivity );
		#elif defined( ENVMAP_BLENDING_MIX )
			outgoingLight = mix( outgoingLight, envColor.xyz, specularStrength * reflectivity );
		#elif defined( ENVMAP_BLENDING_ADD )
			outgoingLight += envColor.xyz * specularStrength * reflectivity;
		#endif
	#endif
#endif`,envmap_common_pars_fragment:`#ifdef USE_ENVMAP
	uniform float envMapIntensity;
	uniform mat3 envMapRotation;
	uniform samplerCube envMap;
	#ifdef ENVMAP_TYPE_PMREM
		float roughnessToMip( const in float roughness ) {
			float r = clamp( roughness, 0.0, 1.0 );
			return ENVMAP_MAX_LOD * r * ( 2.0 - r );
		}
	#endif
#endif`,envmap_pars_fragment:`#ifdef USE_ENVMAP
	uniform float reflectivity;
	#if defined( USE_BUMPMAP ) || defined( USE_NORMALMAP ) || defined( PHONG ) || defined( LAMBERT )
		#define ENV_WORLDPOS
	#endif
	#ifdef ENV_WORLDPOS
		varying vec3 vWorldPosition;
		uniform float refractionRatio;
	#else
		varying vec3 vReflect;
	#endif
#endif`,envmap_pars_vertex:`#ifdef USE_ENVMAP
	#if defined( USE_BUMPMAP ) || defined( USE_NORMALMAP ) || defined( PHONG ) || defined( LAMBERT )
		#define ENV_WORLDPOS
	#endif
	#ifdef ENV_WORLDPOS
		
		varying vec3 vWorldPosition;
	#else
		varying vec3 vReflect;
		uniform float refractionRatio;
	#endif
#endif`,envmap_physical_pars_fragment:`#ifdef USE_ENVMAP
	vec3 getIBLIrradiance( const in vec3 normal ) {
		#ifdef ENVMAP_TYPE_PMREM
			vec3 worldNormal = transformNormalByInverseViewMatrix( normal, viewMatrix );
			vec4 envMapColor = textureLod( envMap, envMapRotation * worldNormal, ENVMAP_MAX_LOD );
			return PI * envMapColor.rgb * envMapIntensity;
		#else
			return vec3( 0.0 );
		#endif
	}
	vec3 getIBLRadiance( const in vec3 viewDir, const in vec3 normal, const in float roughness ) {
		#ifdef ENVMAP_TYPE_PMREM
			vec3 reflectVec = reflect( - viewDir, normal );
			reflectVec = normalize( mix( reflectVec, normal, pow4( roughness ) ) );
			reflectVec = transformDirectionByInverseViewMatrix( reflectVec, viewMatrix );
			vec4 envMapColor = textureLod( envMap, envMapRotation * reflectVec, roughnessToMip( roughness ) );
			return envMapColor.rgb * envMapIntensity;
		#else
			return vec3( 0.0 );
		#endif
	}
	#ifdef USE_RETROREFLECTION
		vec3 getIBLRetroRadiance( const in vec3 viewDir, const in vec3 normal, const in float roughness ) {
			#ifdef ENVMAP_TYPE_PMREM
				vec3 retroVec = normalize( mix( viewDir, normal, pow4( roughness ) ) );
				retroVec = transformDirectionByInverseViewMatrix( retroVec, viewMatrix );
				vec4 envMapColor = textureLod( envMap, envMapRotation * retroVec, roughnessToMip( roughness ) );
				return envMapColor.rgb * envMapIntensity;
			#else
				return vec3( 0.0 );
			#endif
		}
	#endif
	#ifdef USE_ANISOTROPY
		vec3 getIBLAnisotropyRadiance( const in vec3 viewDir, const in vec3 normal, const in float roughness, const in vec3 bitangent, const in float anisotropy ) {
			#ifdef ENVMAP_TYPE_PMREM
				vec3 bentNormal = cross( bitangent, viewDir );
				bentNormal = normalize( cross( bentNormal, bitangent ) );
				bentNormal = normalize( mix( bentNormal, normal, pow2( pow2( 1.0 - anisotropy * ( 1.0 - roughness ) ) ) ) );
				return getIBLRadiance( viewDir, bentNormal, roughness );
			#else
				return vec3( 0.0 );
			#endif
		}
		#ifdef USE_RETROREFLECTION
			vec3 getIBLAnisotropyRetroRadiance( const in vec3 viewDir, const in vec3 normal, const in float roughness, const in vec3 bitangent, const in float anisotropy ) {
				#ifdef ENVMAP_TYPE_PMREM
					vec3 bentNormal = cross( bitangent, viewDir );
					bentNormal = normalize( cross( bentNormal, bitangent ) );
					bentNormal = normalize( mix( bentNormal, normal, pow2( pow2( 1.0 - anisotropy * ( 1.0 - roughness ) ) ) ) );
					return getIBLRetroRadiance( viewDir, bentNormal, roughness );
				#else
					return vec3( 0.0 );
				#endif
			}
		#endif
	#endif
#endif`,envmap_vertex:`#ifdef USE_ENVMAP
	#ifdef ENV_WORLDPOS
		vWorldPosition = worldPosition.xyz;
	#else
		vec3 cameraToVertex;
		if ( isOrthographic ) {
			cameraToVertex = normalize( vec3( - viewMatrix[ 0 ][ 2 ], - viewMatrix[ 1 ][ 2 ], - viewMatrix[ 2 ][ 2 ] ) );
		} else {
			cameraToVertex = normalize( worldPosition.xyz - cameraPosition );
		}
		vec3 worldNormal = transformNormalByInverseViewMatrix( transformedNormal, viewMatrix );
		#ifdef ENVMAP_MODE_REFLECTION
			vReflect = reflect( cameraToVertex, worldNormal );
		#else
			vReflect = refract( cameraToVertex, worldNormal, refractionRatio );
		#endif
	#endif
#endif`,fog_vertex:`#ifdef USE_FOG
	vFogDepth = - mvPosition.z;
#endif`,fog_pars_vertex:`#ifdef USE_FOG
	varying float vFogDepth;
#endif`,fog_fragment:`#ifdef USE_FOG
	#ifdef FOG_EXP2
		float fogFactor = 1.0 - exp( - fogDensity * fogDensity * vFogDepth * vFogDepth );
	#else
		float fogFactor = smoothstep( fogNear, fogFar, vFogDepth );
	#endif
	gl_FragColor.rgb = mix( gl_FragColor.rgb, fogColor, fogFactor );
#endif`,fog_pars_fragment:`#ifdef USE_FOG
	uniform vec3 fogColor;
	varying float vFogDepth;
	#ifdef FOG_EXP2
		uniform float fogDensity;
	#else
		uniform float fogNear;
		uniform float fogFar;
	#endif
#endif`,gradientmap_pars_fragment:`#ifdef USE_GRADIENTMAP
	uniform sampler2D gradientMap;
#endif
vec3 getGradientIrradiance( vec3 normal, vec3 lightDirection ) {
	float dotNL = dot( normal, lightDirection );
	vec2 coord = vec2( dotNL * 0.5 + 0.5, 0.0 );
	#ifdef USE_GRADIENTMAP
		return vec3( texture2D( gradientMap, coord ).r );
	#else
		vec2 fw = fwidth( coord ) * 0.5;
		return mix( vec3( 0.7 ), vec3( 1.0 ), smoothstep( 0.7 - fw.x, 0.7 + fw.x, coord.x ) );
	#endif
}`,lightmap_pars_fragment:`#ifdef USE_LIGHTMAP
	uniform sampler2D lightMap;
	uniform float lightMapIntensity;
#endif`,lights_lambert_fragment:`LambertMaterial material;
material.diffuseColor = diffuseColor.rgb;
material.specularStrength = specularStrength;`,lights_lambert_pars_fragment:`varying vec3 vViewPosition;
struct LambertMaterial {
	vec3 diffuseColor;
	float specularStrength;
};
void RE_Direct_Lambert( const in IncidentLight directLight, const in vec3 geometryPosition, const in vec3 geometryNormal, const in vec3 geometryViewDir, const in vec3 geometryClearcoatNormal, const in LambertMaterial material, inout ReflectedLight reflectedLight ) {
	float dotNL = saturate( dot( geometryNormal, directLight.direction ) );
	vec3 irradiance = dotNL * directLight.color;
	reflectedLight.directDiffuse += irradiance * BRDF_Lambert( material.diffuseColor );
}
void RE_IndirectDiffuse_Lambert( const in vec3 irradiance, const in vec3 geometryPosition, const in vec3 geometryNormal, const in vec3 geometryViewDir, const in vec3 geometryClearcoatNormal, const in LambertMaterial material, inout ReflectedLight reflectedLight ) {
	reflectedLight.indirectDiffuse += irradiance * BRDF_Lambert( material.diffuseColor );
}
#define RE_Direct				RE_Direct_Lambert
#define RE_IndirectDiffuse		RE_IndirectDiffuse_Lambert`,lights_pars_begin:`uniform bool receiveShadow;
uniform vec3 ambientLightColor;
#if defined( USE_LIGHT_PROBES )
	uniform vec3 lightProbe[ 9 ];
#endif
vec3 shGetIrradianceAt( in vec3 normal, in vec3 shCoefficients[ 9 ] ) {
	float x = normal.x, y = normal.y, z = normal.z;
	vec3 result = shCoefficients[ 0 ] * 0.886227;
	result += shCoefficients[ 1 ] * 2.0 * 0.511664 * y;
	result += shCoefficients[ 2 ] * 2.0 * 0.511664 * z;
	result += shCoefficients[ 3 ] * 2.0 * 0.511664 * x;
	result += shCoefficients[ 4 ] * 2.0 * 0.429043 * x * y;
	result += shCoefficients[ 5 ] * 2.0 * 0.429043 * y * z;
	result += shCoefficients[ 6 ] * ( 0.743125 * z * z - 0.247708 );
	result += shCoefficients[ 7 ] * 2.0 * 0.429043 * x * z;
	result += shCoefficients[ 8 ] * 0.429043 * ( x * x - y * y );
	return result;
}
vec3 getLightProbeIrradiance( const in vec3 lightProbe[ 9 ], const in vec3 normal ) {
	vec3 worldNormal = transformNormalByInverseViewMatrix( normal, viewMatrix );
	vec3 irradiance = shGetIrradianceAt( worldNormal, lightProbe );
	return irradiance;
}
vec3 getAmbientLightIrradiance( const in vec3 ambientLightColor ) {
	vec3 irradiance = ambientLightColor;
	return irradiance;
}
float getDistanceAttenuation( const in float lightDistance, const in float cutoffDistance, const in float decayExponent ) {
	float distanceFalloff = 1.0 / max( pow( lightDistance, decayExponent ), 0.01 );
	if ( cutoffDistance > 0.0 ) {
		distanceFalloff *= pow2( saturate( 1.0 - pow4( lightDistance / cutoffDistance ) ) );
	}
	return distanceFalloff;
}
float getSpotAttenuation( const in float coneCosine, const in float penumbraCosine, const in float angleCosine ) {
	return smoothstep( coneCosine, penumbraCosine, angleCosine );
}
#if NUM_SUN_LIGHTS > 0
	struct SunLight {
		vec3 direction;
		vec3 color;
	};
	uniform SunLight sunLights[ NUM_SUN_LIGHTS ];
	void getSunLightInfo( const in SunLight sunLight, out IncidentLight light ) {
		light.color = sunLight.color;
		light.direction = sunLight.direction;
		light.visible = true;
	}
#endif
#if NUM_DIR_LIGHTS > 0
	struct DirectionalLight {
		vec3 direction;
		vec3 color;
	};
	uniform DirectionalLight directionalLights[ NUM_DIR_LIGHTS ];
	void getDirectionalLightInfo( const in DirectionalLight directionalLight, out IncidentLight light ) {
		light.color = directionalLight.color;
		light.direction = directionalLight.direction;
		light.visible = true;
	}
#endif
#if NUM_POINT_LIGHTS > 0
	struct PointLight {
		vec3 position;
		vec3 color;
		float distance;
		float decay;
	};
	uniform PointLight pointLights[ NUM_POINT_LIGHTS ];
	void getPointLightInfo( const in PointLight pointLight, const in vec3 geometryPosition, out IncidentLight light ) {
		vec3 lVector = pointLight.position - geometryPosition;
		light.direction = normalize( lVector );
		float lightDistance = length( lVector );
		light.color = pointLight.color;
		light.color *= getDistanceAttenuation( lightDistance, pointLight.distance, pointLight.decay );
		light.visible = ( light.color != vec3( 0.0 ) );
	}
#endif
#if NUM_SPOT_LIGHTS > 0
	struct SpotLight {
		vec3 position;
		vec3 direction;
		vec3 color;
		float distance;
		float decay;
		float coneCos;
		float penumbraCos;
	};
	uniform SpotLight spotLights[ NUM_SPOT_LIGHTS ];
	void getSpotLightInfo( const in SpotLight spotLight, const in vec3 geometryPosition, out IncidentLight light ) {
		vec3 lVector = spotLight.position - geometryPosition;
		light.direction = normalize( lVector );
		float angleCos = dot( light.direction, spotLight.direction );
		float spotAttenuation = getSpotAttenuation( spotLight.coneCos, spotLight.penumbraCos, angleCos );
		if ( spotAttenuation > 0.0 ) {
			float lightDistance = length( lVector );
			light.color = spotLight.color * spotAttenuation;
			light.color *= getDistanceAttenuation( lightDistance, spotLight.distance, spotLight.decay );
			light.visible = ( light.color != vec3( 0.0 ) );
		} else {
			light.color = vec3( 0.0 );
			light.visible = false;
		}
	}
#endif
#if NUM_RECT_AREA_LIGHTS > 0
	struct RectAreaLight {
		vec3 color;
		vec3 position;
		vec3 halfWidth;
		vec3 halfHeight;
	};
	uniform sampler2D ltc_1;	uniform sampler2D ltc_2;
	uniform RectAreaLight rectAreaLights[ NUM_RECT_AREA_LIGHTS ];
#endif
#if NUM_HEMI_LIGHTS > 0
	struct HemisphereLight {
		vec3 direction;
		vec3 skyColor;
		vec3 groundColor;
	};
	uniform HemisphereLight hemisphereLights[ NUM_HEMI_LIGHTS ];
	vec3 getHemisphereLightIrradiance( const in HemisphereLight hemiLight, const in vec3 normal ) {
		float dotNL = dot( normal, hemiLight.direction );
		float hemiDiffuseWeight = 0.5 * dotNL + 0.5;
		vec3 irradiance = mix( hemiLight.groundColor, hemiLight.skyColor, hemiDiffuseWeight );
		return irradiance;
	}
#endif
#include <lightprobes_pars_fragment>`,lights_toon_fragment:`ToonMaterial material;
material.diffuseColor = diffuseColor.rgb;`,lights_toon_pars_fragment:`varying vec3 vViewPosition;
struct ToonMaterial {
	vec3 diffuseColor;
};
void RE_Direct_Toon( const in IncidentLight directLight, const in vec3 geometryPosition, const in vec3 geometryNormal, const in vec3 geometryViewDir, const in vec3 geometryClearcoatNormal, const in ToonMaterial material, inout ReflectedLight reflectedLight ) {
	vec3 irradiance = getGradientIrradiance( geometryNormal, directLight.direction ) * directLight.color;
	reflectedLight.directDiffuse += irradiance * BRDF_Lambert( material.diffuseColor );
}
void RE_IndirectDiffuse_Toon( const in vec3 irradiance, const in vec3 geometryPosition, const in vec3 geometryNormal, const in vec3 geometryViewDir, const in vec3 geometryClearcoatNormal, const in ToonMaterial material, inout ReflectedLight reflectedLight ) {
	reflectedLight.indirectDiffuse += irradiance * BRDF_Lambert( material.diffuseColor );
}
#define RE_Direct				RE_Direct_Toon
#define RE_IndirectDiffuse		RE_IndirectDiffuse_Toon`,lights_phong_fragment:`BlinnPhongMaterial material;
material.diffuseColor = diffuseColor.rgb;
material.specularColor = specular;
material.specularShininess = shininess;
material.specularStrength = specularStrength;`,lights_phong_pars_fragment:`varying vec3 vViewPosition;
struct BlinnPhongMaterial {
	vec3 diffuseColor;
	vec3 specularColor;
	float specularShininess;
	float specularStrength;
};
void RE_Direct_BlinnPhong( const in IncidentLight directLight, const in vec3 geometryPosition, const in vec3 geometryNormal, const in vec3 geometryViewDir, const in vec3 geometryClearcoatNormal, const in BlinnPhongMaterial material, inout ReflectedLight reflectedLight ) {
	float dotNL = saturate( dot( geometryNormal, directLight.direction ) );
	vec3 irradiance = dotNL * directLight.color;
	reflectedLight.directDiffuse += irradiance * BRDF_Lambert( material.diffuseColor );
	reflectedLight.directSpecular += irradiance * BRDF_BlinnPhong( directLight.direction, geometryViewDir, geometryNormal, material.specularColor, material.specularShininess ) * material.specularStrength;
}
void RE_IndirectDiffuse_BlinnPhong( const in vec3 irradiance, const in vec3 geometryPosition, const in vec3 geometryNormal, const in vec3 geometryViewDir, const in vec3 geometryClearcoatNormal, const in BlinnPhongMaterial material, inout ReflectedLight reflectedLight ) {
	reflectedLight.indirectDiffuse += irradiance * BRDF_Lambert( material.diffuseColor );
}
#define RE_Direct				RE_Direct_BlinnPhong
#define RE_IndirectDiffuse		RE_IndirectDiffuse_BlinnPhong`,lights_physical_fragment:`PhysicalMaterial material;
material.diffuseColor = diffuseColor.rgb;
material.diffuseContribution = diffuseColor.rgb * ( 1.0 - metalnessFactor );
material.metalness = metalnessFactor;
vec3 dxy = max( abs( dFdx( nonPerturbedNormal ) ), abs( dFdy( nonPerturbedNormal ) ) );
float geometryRoughness = max( max( dxy.x, dxy.y ), dxy.z );
material.roughness = max( roughnessFactor, 0.045 );
material.roughness += geometryRoughness;
material.roughness = min( material.roughness, 1.0 );
#ifdef USE_DIFFUSE_ROUGHNESS
	float diffuseRoughnessFactor = diffuseRoughness;
	#ifdef USE_DIFFUSE_ROUGHNESSMAP
		diffuseRoughnessFactor *= texture2D( diffuseRoughnessMap, vDiffuseRoughnessMapUv ).r;
	#endif
	material.diffuseRoughness = saturate( diffuseRoughnessFactor );
#endif
#ifdef IOR
	material.ior = ior;
	#ifdef USE_SPECULAR
		float specularIntensityFactor = specularIntensity;
		vec3 specularColorFactor = specularColor;
		#ifdef USE_SPECULAR_COLORMAP
			specularColorFactor *= texture2D( specularColorMap, vSpecularColorMapUv ).rgb;
		#endif
		#ifdef USE_SPECULAR_INTENSITYMAP
			specularIntensityFactor *= texture2D( specularIntensityMap, vSpecularIntensityMapUv ).a;
		#endif
		material.specularF90 = mix( specularIntensityFactor, 1.0, metalnessFactor );
	#else
		float specularIntensityFactor = 1.0;
		vec3 specularColorFactor = vec3( 1.0 );
		material.specularF90 = 1.0;
	#endif
	material.specularColor = min( pow2( ( material.ior - 1.0 ) / ( material.ior + 1.0 ) ) * specularColorFactor, vec3( 1.0 ) ) * specularIntensityFactor;
	material.specularColorBlended = mix( material.specularColor, diffuseColor.rgb, metalnessFactor );
#else
	material.specularColor = vec3( 0.04 );
	material.specularColorBlended = mix( material.specularColor, diffuseColor.rgb, metalnessFactor );
	material.specularF90 = 1.0;
#endif
#ifdef USE_CLEARCOAT
	material.clearcoat = clearcoat;
	material.clearcoatRoughness = clearcoatRoughness;
	material.clearcoatF0 = vec3( 0.04 );
	material.clearcoatF90 = 1.0;
	#ifdef USE_CLEARCOATMAP
		material.clearcoat *= texture2D( clearcoatMap, vClearcoatMapUv ).x;
	#endif
	#ifdef USE_CLEARCOAT_ROUGHNESSMAP
		material.clearcoatRoughness *= texture2D( clearcoatRoughnessMap, vClearcoatRoughnessMapUv ).y;
	#endif
	material.clearcoat = saturate( material.clearcoat );	material.clearcoatRoughness = max( material.clearcoatRoughness, 0.045 );
	material.clearcoatRoughness += geometryRoughness;
	material.clearcoatRoughness = min( material.clearcoatRoughness, 1.0 );
#endif
#ifdef USE_DISPERSION
	material.dispersion = dispersion;
#endif
#ifdef USE_RETROREFLECTION
	material.retroreflectivity = retroreflectivity;
#endif
#ifdef USE_IRIDESCENCE
	material.iridescence = iridescence;
	material.iridescenceIOR = iridescenceIOR;
	#ifdef USE_IRIDESCENCEMAP
		material.iridescence *= texture2D( iridescenceMap, vIridescenceMapUv ).r;
	#endif
	#ifdef USE_IRIDESCENCE_THICKNESSMAP
		material.iridescenceThickness = (iridescenceThicknessMaximum - iridescenceThicknessMinimum) * texture2D( iridescenceThicknessMap, vIridescenceThicknessMapUv ).g + iridescenceThicknessMinimum;
	#else
		material.iridescenceThickness = iridescenceThicknessMaximum;
	#endif
#endif
#ifdef USE_SHEEN
	material.sheenColor = sheenColor;
	#ifdef USE_SHEEN_COLORMAP
		material.sheenColor *= texture2D( sheenColorMap, vSheenColorMapUv ).rgb;
	#endif
	material.sheenRoughness = clamp( sheenRoughness, 0.0001, 1.0 );
	#ifdef USE_SHEEN_ROUGHNESSMAP
		material.sheenRoughness *= texture2D( sheenRoughnessMap, vSheenRoughnessMapUv ).a;
	#endif
#endif
#ifdef USE_ANISOTROPY
	#ifdef USE_ANISOTROPYMAP
		mat2 anisotropyMat = mat2( anisotropyVector.x, anisotropyVector.y, - anisotropyVector.y, anisotropyVector.x );
		vec3 anisotropyPolar = texture2D( anisotropyMap, vAnisotropyMapUv ).rgb;
		vec2 anisotropyV = anisotropyMat * normalize( 2.0 * anisotropyPolar.rg - vec2( 1.0 ) ) * anisotropyPolar.b;
	#else
		vec2 anisotropyV = anisotropyVector;
	#endif
	material.anisotropy = length( anisotropyV );
	if( material.anisotropy == 0.0 ) {
		anisotropyV = vec2( 1.0, 0.0 );
	} else {
		anisotropyV /= material.anisotropy;
		material.anisotropy = saturate( material.anisotropy );
	}
	material.alphaT = mix( pow2( material.roughness ), 1.0, pow2( material.anisotropy ) );
	material.anisotropyT = tbn[ 0 ] * anisotropyV.x + tbn[ 1 ] * anisotropyV.y;
	material.anisotropyB = tbn[ 1 ] * anisotropyV.x - tbn[ 0 ] * anisotropyV.y;
#endif`,lights_physical_pars_fragment:`uniform sampler2D dfgLUT;
struct PhysicalMaterial {
	vec3 diffuseColor;
	vec3 diffuseContribution;
	vec3 specularColor;
	vec3 specularColorBlended;
	float roughness;
	float metalness;
	float specularF90;
	float dispersion;
	vec2 dfg;
	vec3 multiScatteringCompensation;
	#ifdef USE_DIFFUSE_ROUGHNESS
		float diffuseRoughness;
	#endif
	#ifdef USE_RETROREFLECTION
		float retroreflectivity;
	#endif
	#ifdef USE_CLEARCOAT
		float clearcoat;
		float clearcoatRoughness;
		vec3 clearcoatF0;
		float clearcoatF90;
	#endif
	#ifdef USE_IRIDESCENCE
		float iridescence;
		float iridescenceIOR;
		float iridescenceThickness;
		vec3 iridescenceFresnel;
		vec3 iridescenceF0Dielectric;
		vec3 iridescenceF0Metallic;
	#endif
	#ifdef USE_SHEEN
		vec3 sheenColor;
		float sheenRoughness;
	#endif
	#ifdef IOR
		float ior;
	#endif
	#ifdef USE_TRANSMISSION
		float transmission;
		float transmissionAlpha;
		float thickness;
		float attenuationDistance;
		vec3 attenuationColor;
	#endif
	#ifdef USE_ANISOTROPY
		float anisotropy;
		float alphaT;
		vec3 anisotropyT;
		vec3 anisotropyB;
	#endif
};
vec3 clearcoatSpecularDirect = vec3( 0.0 );
vec3 clearcoatSpecularIndirect = vec3( 0.0 );
vec3 sheenSpecularDirect = vec3( 0.0 );
vec3 sheenSpecularIndirect = vec3(0.0 );
vec3 Schlick_to_F0( const in vec3 f, const in float f90, const in float dotVH ) {
    float x = clamp( 1.0 - dotVH, 0.0, 1.0 );
    float x2 = x * x;
    float x5 = clamp( x * x2 * x2, 0.0, 0.9999 );
    return ( f - vec3( f90 ) * x5 ) / ( 1.0 - x5 );
}
#ifdef USE_DIFFUSE_ROUGHNESS
	const float EON_EPSILON = 1e-7;
	float FON_DirectionalAlbedo( const in float mu, const in float roughness, const in float A ) {
		float muComp = 1.0 - mu;
		float gOverPi = muComp * ( 0.0571085289 + muComp * ( 0.491881867 + muComp * ( - 0.332181442 + muComp * 0.0714429953 ) ) );
		return A * ( 1.0 + roughness * gOverPi );
	}
	vec3 BRDF_EON( const in vec3 lightDir, const in vec3 viewDir, const in vec3 normal, const in vec3 diffuseColor, const in float roughness ) {
		vec3 rho = saturate( diffuseColor );
		if ( roughness <= EON_EPSILON ) return BRDF_Lambert( rho );
		float dotNL = saturate( dot( normal, lightDir ) );
		float dotNV = saturate( dot( normal, viewDir ) );
		float s = dot( lightDir, viewDir ) - dotNL * dotNV;
		float sOverT = ( s > 0.0 ) ? s / max( max( dotNL, dotNV ), EON_EPSILON ) : s;
		float A = 1.0 / ( 1.0 + ( 0.5 - 2.0 / ( 3.0 * PI ) ) * roughness );
		vec3 singleScatter = rho * RECIPROCAL_PI * A * ( 1.0 + roughness * sOverT );
		float averageAlbedo = A * ( 1.0 + ( 2.0 / 3.0 - 28.0 / ( 15.0 * PI ) ) * roughness );
		float albedoV = FON_DirectionalAlbedo( dotNV, roughness, A );
		float albedoL = FON_DirectionalAlbedo( dotNL, roughness, A );
		vec3 rhoMultiScatter = rho * rho * averageAlbedo / max( vec3( EON_EPSILON ), vec3( 1.0 ) - rho * ( 1.0 - averageAlbedo ) );
		vec3 multiScatter = rhoMultiScatter * RECIPROCAL_PI
			* max( EON_EPSILON, 1.0 - albedoV )
			* max( EON_EPSILON, 1.0 - albedoL )
			/ max( EON_EPSILON, 1.0 - averageAlbedo );
		return singleScatter + multiScatter;
	}
	vec3 EON_DirectionalAlbedo( const in vec3 diffuseColor, const in float roughness, const in float dotNV ) {
		vec3 rho = saturate( diffuseColor );
		if ( roughness <= EON_EPSILON ) return rho;
		float A = 1.0 / ( 1.0 + ( 0.5 - 2.0 / ( 3.0 * PI ) ) * roughness );
		float directionalAlbedo = FON_DirectionalAlbedo( dotNV, roughness, A );
		float averageAlbedo = A * ( 1.0 + ( 2.0 / 3.0 - 28.0 / ( 15.0 * PI ) ) * roughness );
		vec3 rhoMultiScatter = rho * rho * averageAlbedo / max( vec3( EON_EPSILON ), vec3( 1.0 ) - rho * ( 1.0 - averageAlbedo ) );
		return rho * directionalAlbedo + rhoMultiScatter * ( 1.0 - directionalAlbedo );
	}
#endif
float V_GGX_SmithCorrelated( const in float alpha, const in float dotNL, const in float dotNV ) {
	float a2 = pow2( alpha );
	float gv = dotNL * sqrt( a2 + ( 1.0 - a2 ) * pow2( dotNV ) );
	float gl = dotNV * sqrt( a2 + ( 1.0 - a2 ) * pow2( dotNL ) );
	return 0.5 / max( gv + gl, EPSILON );
}
float D_GGX( const in float alpha, const in float dotNH ) {
	float a2 = pow2( alpha );
	float denom = pow2( dotNH ) * ( a2 - 1.0 ) + 1.0;
	return RECIPROCAL_PI * a2 / pow2( denom );
}
#ifdef USE_ANISOTROPY
	float V_GGX_SmithCorrelated_Anisotropic( const in float alphaT, const in float alphaB, const in float dotTV, const in float dotBV, const in float dotTL, const in float dotBL, const in float dotNV, const in float dotNL ) {
		float gv = dotNL * length( vec3( alphaT * dotTV, alphaB * dotBV, dotNV ) );
		float gl = dotNV * length( vec3( alphaT * dotTL, alphaB * dotBL, dotNL ) );
		return 0.5 / max( gv + gl, EPSILON );
	}
	float D_GGX_Anisotropic( const in float alphaT, const in float alphaB, const in float dotNH, const in float dotTH, const in float dotBH ) {
		float a2 = alphaT * alphaB;
		highp vec3 v = vec3( alphaB * dotTH, alphaT * dotBH, a2 * dotNH );
		highp float v2 = dot( v, v );
		float w2 = a2 / v2;
		return RECIPROCAL_PI * a2 * pow2 ( w2 );
	}
#endif
#ifdef USE_CLEARCOAT
	vec3 BRDF_GGX_Clearcoat( const in vec3 lightDir, const in vec3 viewDir, const in vec3 normal, const in PhysicalMaterial material) {
		vec3 f0 = material.clearcoatF0;
		float f90 = material.clearcoatF90;
		float roughness = max( material.clearcoatRoughness, 0.045 );
		float alpha = pow2( roughness );
		vec3 halfDir = normalize( lightDir + viewDir );
		float dotNL = saturate( dot( normal, lightDir ) );
		float dotNV = saturate( dot( normal, viewDir ) );
		float dotNH = saturate( dot( normal, halfDir ) );
		float dotVH = saturate( dot( viewDir, halfDir ) );
		vec3 F = F_Schlick( f0, f90, dotVH );
		float V = V_GGX_SmithCorrelated( alpha, dotNL, dotNV );
		float D = D_GGX( alpha, dotNH );
		return F * ( V * D );
	}
#endif
vec3 BRDF_GGX( const in vec3 lightDir, const in vec3 viewDir, const in vec3 normal, const in PhysicalMaterial material ) {
	vec3 f0 = material.specularColorBlended;
	float f90 = material.specularF90;
	float roughness = max( material.roughness, 0.045 );
	float alpha = pow2( roughness );
	vec3 halfDir = normalize( lightDir + viewDir );
	float dotNL = saturate( dot( normal, lightDir ) );
	float dotNV = saturate( dot( normal, viewDir ) );
	float dotNH = saturate( dot( normal, halfDir ) );
	float dotVH = saturate( dot( viewDir, halfDir ) );
	vec3 F = F_Schlick( f0, f90, dotVH );
	#ifdef USE_IRIDESCENCE
		F = mix( F, material.iridescenceFresnel, material.iridescence );
	#endif
	#ifdef USE_ANISOTROPY
		float dotTL = dot( material.anisotropyT, lightDir );
		float dotTV = dot( material.anisotropyT, viewDir );
		float dotTH = dot( material.anisotropyT, halfDir );
		float dotBL = dot( material.anisotropyB, lightDir );
		float dotBV = dot( material.anisotropyB, viewDir );
		float dotBH = dot( material.anisotropyB, halfDir );
		float alphaT = max( material.alphaT, alpha );
		float V = V_GGX_SmithCorrelated_Anisotropic( alphaT, alpha, dotTV, dotBV, dotTL, dotBL, dotNV, dotNL );
		float D = D_GGX_Anisotropic( alphaT, alpha, dotNH, dotTH, dotBH );
	#else
		float V = V_GGX_SmithCorrelated( alpha, dotNL, dotNV );
		float D = D_GGX( alpha, dotNH );
	#endif
	return F * ( V * D );
}
vec2 LTC_Uv( const in vec3 N, const in vec3 V, const in float roughness ) {
	const float LUT_SIZE = 64.0;
	const float LUT_SCALE = ( LUT_SIZE - 1.0 ) / LUT_SIZE;
	const float LUT_BIAS = 0.5 / LUT_SIZE;
	float dotNV = saturate( dot( N, V ) );
	vec2 uv = vec2( roughness, sqrt( 1.0 - dotNV ) );
	uv = uv * LUT_SCALE + LUT_BIAS;
	return uv;
}
float LTC_ClippedSphereFormFactor( const in vec3 f ) {
	float l = length( f );
	return max( ( l * l + f.z ) / ( l + 1.0 ), 0.0 );
}
vec3 LTC_EdgeVectorFormFactor( const in vec3 v1, const in vec3 v2 ) {
	float x = dot( v1, v2 );
	float y = abs( x );
	float a = 0.8543985 + ( 0.4965155 + 0.0145206 * y ) * y;
	float b = 3.4175940 + ( 4.1616724 + y ) * y;
	float v = a / b;
	float theta_sintheta = ( x > 0.0 ) ? v : 0.5 * inversesqrt( max( 1.0 - x * x, 1e-7 ) ) - v;
	return cross( v1, v2 ) * theta_sintheta;
}
vec3 LTC_Evaluate( const in vec3 N, const in vec3 V, const in vec3 P, const in mat3 mInv, const in vec3 rectCoords[ 4 ] ) {
	vec3 v1 = rectCoords[ 1 ] - rectCoords[ 0 ];
	vec3 v2 = rectCoords[ 3 ] - rectCoords[ 0 ];
	vec3 lightNormal = cross( v1, v2 );
	if( dot( lightNormal, P - rectCoords[ 0 ] ) < 0.0 ) return vec3( 0.0 );
	vec3 T1, T2;
	T1 = normalize( V - N * dot( V, N ) );
	T2 = - cross( N, T1 );
	mat3 mat = mInv * transpose( mat3( T1, T2, N ) );
	vec3 coords[ 4 ];
	coords[ 0 ] = mat * ( rectCoords[ 0 ] - P );
	coords[ 1 ] = mat * ( rectCoords[ 1 ] - P );
	coords[ 2 ] = mat * ( rectCoords[ 2 ] - P );
	coords[ 3 ] = mat * ( rectCoords[ 3 ] - P );
	coords[ 0 ] = normalize( coords[ 0 ] );
	coords[ 1 ] = normalize( coords[ 1 ] );
	coords[ 2 ] = normalize( coords[ 2 ] );
	coords[ 3 ] = normalize( coords[ 3 ] );
	vec3 vectorFormFactor = vec3( 0.0 );
	vectorFormFactor += LTC_EdgeVectorFormFactor( coords[ 0 ], coords[ 1 ] );
	vectorFormFactor += LTC_EdgeVectorFormFactor( coords[ 1 ], coords[ 2 ] );
	vectorFormFactor += LTC_EdgeVectorFormFactor( coords[ 2 ], coords[ 3 ] );
	vectorFormFactor += LTC_EdgeVectorFormFactor( coords[ 3 ], coords[ 0 ] );
	float result = LTC_ClippedSphereFormFactor( vectorFormFactor );
	return vec3( result );
}
#if defined( USE_SHEEN )
float D_Charlie( float roughness, float dotNH ) {
	float alpha = pow2( roughness );
	float invAlpha = 1.0 / alpha;
	float cos2h = dotNH * dotNH;
	float sin2h = max( 1.0 - cos2h, 0.0078125 );
	return ( 2.0 + invAlpha ) * pow( sin2h, invAlpha * 0.5 ) / ( 2.0 * PI );
}
float V_Neubelt( float dotNV, float dotNL ) {
	return saturate( 1.0 / ( 4.0 * ( dotNL + dotNV - dotNL * dotNV ) ) );
}
vec3 BRDF_Sheen( const in vec3 lightDir, const in vec3 viewDir, const in vec3 normal, vec3 sheenColor, const in float sheenRoughness ) {
	vec3 halfDir = normalize( lightDir + viewDir );
	float dotNL = saturate( dot( normal, lightDir ) );
	float dotNV = saturate( dot( normal, viewDir ) );
	float dotNH = saturate( dot( normal, halfDir ) );
	float D = D_Charlie( sheenRoughness, dotNH );
	float V = V_Neubelt( dotNV, dotNL );
	return sheenColor * ( D * V );
}
#endif
float IBLSheenBRDF( const in vec3 normal, const in vec3 viewDir, const in float roughness ) {
	float dotNV = saturate( dot( normal, viewDir ) );
	float r2 = roughness * roughness;
	float rInv = 1.0 / ( roughness + 0.1 );
	float a = -1.9362 + 1.0678 * roughness + 0.4573 * r2 - 0.8469 * rInv;
	float b = -0.6014 + 0.5538 * roughness - 0.4670 * r2 - 0.1255 * rInv;
	float DG = exp( a * dotNV + b );
	return saturate( DG );
}
vec3 EnvironmentBRDF( const in vec3 normal, const in vec3 viewDir, const in vec3 specularColor, const in float specularF90, const in float roughness ) {
	float dotNV = saturate( dot( normal, viewDir ) );
	vec2 fab = texture2D( dfgLUT, vec2( roughness, dotNV ) ).rg;
	return specularColor * fab.x + specularF90 * fab.y;
}
#ifdef USE_IRIDESCENCE
void computeMultiscatteringIridescence( const in vec2 fab, const in vec3 specularColor, const in float specularF90, const in float iridescence, const in vec3 iridescenceF0, inout vec3 singleScatter, inout vec3 multiScatter ) {
#else
void computeMultiscattering( const in vec2 fab, const in vec3 specularColor, const in float specularF90, inout vec3 singleScatter, inout vec3 multiScatter ) {
#endif
	#ifdef USE_IRIDESCENCE
		vec3 Fr = mix( specularColor, iridescenceF0, iridescence );
	#else
		vec3 Fr = specularColor;
	#endif
	vec3 FssEss = Fr * fab.x + specularF90 * fab.y;
	float Ess = fab.x + fab.y;
	float Ems = 1.0 - Ess;
	vec3 Favg = Fr + ( 1.0 - Fr ) * 0.047619;	vec3 Fms = FssEss * Favg / ( 1.0 - Ems * Favg );
	singleScatter += FssEss;
	multiScatter += Fms * Ems;
}
#if NUM_RECT_AREA_LIGHTS > 0
	void RE_Direct_RectArea_Physical( const in RectAreaLight rectAreaLight, const in vec3 geometryPosition, const in vec3 geometryNormal, const in vec3 geometryViewDir, const in vec3 geometryClearcoatNormal, const in PhysicalMaterial material, inout ReflectedLight reflectedLight ) {
		vec3 normal = geometryNormal;
		vec3 viewDir = geometryViewDir;
		vec3 position = geometryPosition;
		vec3 lightPos = rectAreaLight.position;
		vec3 halfWidth = rectAreaLight.halfWidth;
		vec3 halfHeight = rectAreaLight.halfHeight;
		vec3 lightColor = rectAreaLight.color;
		float roughness = material.roughness;
		vec3 rectCoords[ 4 ];
		rectCoords[ 0 ] = lightPos + halfWidth - halfHeight;		rectCoords[ 1 ] = lightPos - halfWidth - halfHeight;
		rectCoords[ 2 ] = lightPos - halfWidth + halfHeight;
		rectCoords[ 3 ] = lightPos + halfWidth + halfHeight;
		vec2 uv = LTC_Uv( normal, viewDir, roughness );
		vec4 t1 = texture2D( ltc_1, uv );
		vec4 t2 = texture2D( ltc_2, uv );
		mat3 mInv = mat3(
			vec3( t1.x, 0, t1.y ),
			vec3(    0, 1,    0 ),
			vec3( t1.z, 0, t1.w )
		);
		vec3 fresnel = ( material.specularColorBlended * t2.x + ( material.specularF90 - material.specularColorBlended ) * t2.y );
		reflectedLight.directSpecular += lightColor * fresnel * LTC_Evaluate( normal, viewDir, position, mInv, rectCoords );
		reflectedLight.directDiffuse += lightColor * material.diffuseContribution * LTC_Evaluate( normal, viewDir, position, mat3( 1.0 ), rectCoords );
		#ifdef USE_CLEARCOAT
			vec3 Ncc = geometryClearcoatNormal;
			vec2 uvClearcoat = LTC_Uv( Ncc, viewDir, material.clearcoatRoughness );
			vec4 t1Clearcoat = texture2D( ltc_1, uvClearcoat );
			vec4 t2Clearcoat = texture2D( ltc_2, uvClearcoat );
			mat3 mInvClearcoat = mat3(
				vec3( t1Clearcoat.x, 0, t1Clearcoat.y ),
				vec3(             0, 1,             0 ),
				vec3( t1Clearcoat.z, 0, t1Clearcoat.w )
			);
			vec3 fresnelClearcoat = material.clearcoatF0 * t2Clearcoat.x + ( material.clearcoatF90 - material.clearcoatF0 ) * t2Clearcoat.y;
			clearcoatSpecularDirect += lightColor * fresnelClearcoat * LTC_Evaluate( Ncc, viewDir, position, mInvClearcoat, rectCoords );
		#endif
	}
#endif
void RE_Direct_Physical( const in IncidentLight directLight, const in vec3 geometryPosition, const in vec3 geometryNormal, const in vec3 geometryViewDir, const in vec3 geometryClearcoatNormal, const in PhysicalMaterial material, inout ReflectedLight reflectedLight ) {
	float dotNL = saturate( dot( geometryNormal, directLight.direction ) );
	vec3 irradiance = dotNL * directLight.color;
	#ifdef USE_CLEARCOAT
		float dotNLcc = saturate( dot( geometryClearcoatNormal, directLight.direction ) );
		vec3 ccIrradiance = dotNLcc * directLight.color;
		clearcoatSpecularDirect += ccIrradiance * BRDF_GGX_Clearcoat( directLight.direction, geometryViewDir, geometryClearcoatNormal, material );
	#endif
	#ifdef USE_SHEEN
 
 		sheenSpecularDirect += irradiance * BRDF_Sheen( directLight.direction, geometryViewDir, geometryNormal, material.sheenColor, material.sheenRoughness );
 
 		float sheenAlbedoV = IBLSheenBRDF( geometryNormal, geometryViewDir, material.sheenRoughness );
 		float sheenAlbedoL = IBLSheenBRDF( geometryNormal, directLight.direction, material.sheenRoughness );
 
 		float sheenEnergyComp = 1.0 - max3( material.sheenColor ) * max( sheenAlbedoV, sheenAlbedoL );
 
 		irradiance *= sheenEnergyComp;
 
 	#endif
	vec3 specularBRDF = BRDF_GGX( directLight.direction, geometryViewDir, geometryNormal, material );
	#ifdef USE_RETROREFLECTION
		vec3 retroViewDir = reflect( - geometryViewDir, geometryNormal );
		vec3 retroSpecularBRDF = BRDF_GGX( directLight.direction, retroViewDir, geometryNormal, material );
		specularBRDF = mix( specularBRDF, retroSpecularBRDF, saturate( material.retroreflectivity ) );
	#endif
	reflectedLight.directSpecular += irradiance * specularBRDF * material.multiScatteringCompensation;
	vec3 halfDir = normalize( directLight.direction + geometryViewDir );
	float dotVH = saturate( dot( geometryViewDir, halfDir ) );
	vec3 F = F_Schlick( material.specularColor, material.specularF90, dotVH );
	#ifdef USE_RETROREFLECTION
		vec3 retroHalfDir = normalize( directLight.direction + retroViewDir );
		float dotRetroVH = saturate( dot( retroViewDir, retroHalfDir ) );
		vec3 retroF = F_Schlick( material.specularColor, material.specularF90, dotRetroVH );
		F = mix( F, retroF, saturate( material.retroreflectivity ) );
	#endif
	#ifdef USE_DIFFUSE_ROUGHNESS
		vec3 diffuseBRDF = BRDF_EON( directLight.direction, geometryViewDir, geometryNormal, material.diffuseColor, material.diffuseRoughness ) * ( 1.0 - material.metalness );
	#else
		vec3 diffuseBRDF = BRDF_Lambert( material.diffuseContribution );
	#endif
	reflectedLight.directDiffuse += irradiance * diffuseBRDF * ( 1.0 - F );
}
void RE_IndirectDiffuse_Physical( const in vec3 irradiance, const in vec3 geometryPosition, const in vec3 geometryNormal, const in vec3 geometryViewDir, const in vec3 geometryClearcoatNormal, const in PhysicalMaterial material, inout ReflectedLight reflectedLight ) {
	vec3 singleScattering = vec3( 0.0 );
	vec3 multiScattering = vec3( 0.0 );
	#ifdef USE_IRIDESCENCE
		computeMultiscatteringIridescence( material.dfg, material.specularColor, material.specularF90, material.iridescence, material.iridescenceF0Dielectric, singleScattering, multiScattering );
	#else
		computeMultiscattering( material.dfg, material.specularColor, material.specularF90, singleScattering, multiScattering );
	#endif
	#ifdef USE_DIFFUSE_ROUGHNESS
		float dotNV = saturate( dot( geometryNormal, geometryViewDir ) );
		vec3 diffuseAlbedo = EON_DirectionalAlbedo( material.diffuseColor, material.diffuseRoughness, dotNV ) * ( 1.0 - material.metalness );
		vec3 diffuse = irradiance * RECIPROCAL_PI * diffuseAlbedo * ( 1.0 - singleScattering - multiScattering );
	#else
		vec3 diffuse = irradiance * BRDF_Lambert( material.diffuseContribution ) * ( 1.0 - singleScattering - multiScattering );
	#endif
	#ifdef USE_SHEEN
		float sheenAlbedo = IBLSheenBRDF( geometryNormal, geometryViewDir, material.sheenRoughness );
		sheenSpecularIndirect += irradiance * material.sheenColor * sheenAlbedo * RECIPROCAL_PI;
		float sheenEnergyComp = 1.0 - max3( material.sheenColor ) * sheenAlbedo;
		diffuse *= sheenEnergyComp;
	#endif
	reflectedLight.indirectDiffuse += diffuse;
}
void RE_IndirectSpecular_Physical( const in vec3 radiance, const in vec3 irradiance, const in vec3 clearcoatRadiance, const in vec3 geometryPosition, const in vec3 geometryNormal, const in vec3 geometryViewDir, const in vec3 geometryClearcoatNormal, const in PhysicalMaterial material, inout ReflectedLight reflectedLight) {
	#ifdef USE_CLEARCOAT
		clearcoatSpecularIndirect += clearcoatRadiance * EnvironmentBRDF( geometryClearcoatNormal, geometryViewDir, material.clearcoatF0, material.clearcoatF90, material.clearcoatRoughness );
	#endif
	#ifdef USE_SHEEN
		sheenSpecularIndirect += irradiance * material.sheenColor * IBLSheenBRDF( geometryNormal, geometryViewDir, material.sheenRoughness ) * RECIPROCAL_PI;
 	#endif
	vec3 singleScatteringDielectric = vec3( 0.0 );
	vec3 multiScatteringDielectric = vec3( 0.0 );
	vec3 singleScatteringMetallic = vec3( 0.0 );
	vec3 multiScatteringMetallic = vec3( 0.0 );
	#ifdef USE_IRIDESCENCE
		computeMultiscatteringIridescence( material.dfg, material.specularColor, material.specularF90, material.iridescence, material.iridescenceF0Dielectric, singleScatteringDielectric, multiScatteringDielectric );
		computeMultiscatteringIridescence( material.dfg, material.diffuseColor, material.specularF90, material.iridescence, material.iridescenceF0Metallic, singleScatteringMetallic, multiScatteringMetallic );
	#else
		computeMultiscattering( material.dfg, material.specularColor, material.specularF90, singleScatteringDielectric, multiScatteringDielectric );
		computeMultiscattering( material.dfg, material.diffuseColor, material.specularF90, singleScatteringMetallic, multiScatteringMetallic );
	#endif
	vec3 singleScattering = mix( singleScatteringDielectric, singleScatteringMetallic, material.metalness );
	vec3 multiScattering = mix( multiScatteringDielectric, multiScatteringMetallic, material.metalness );
	vec3 totalScatteringDielectric = singleScatteringDielectric + multiScatteringDielectric;
	#ifdef USE_DIFFUSE_ROUGHNESS
		float dotNV = saturate( dot( geometryNormal, geometryViewDir ) );
		vec3 diffuse = EON_DirectionalAlbedo( material.diffuseColor, material.diffuseRoughness, dotNV ) * ( 1.0 - material.metalness ) * ( 1.0 - totalScatteringDielectric );
	#else
		vec3 diffuse = material.diffuseContribution * ( 1.0 - totalScatteringDielectric );
	#endif
	vec3 cosineWeightedIrradiance = irradiance * RECIPROCAL_PI;
	vec3 indirectSpecular = radiance * singleScattering;
	indirectSpecular += multiScattering * cosineWeightedIrradiance;
	vec3 indirectDiffuse = diffuse * cosineWeightedIrradiance;
	#ifdef USE_SHEEN
		float sheenAlbedo = IBLSheenBRDF( geometryNormal, geometryViewDir, material.sheenRoughness );
		float sheenEnergyComp = 1.0 - max3( material.sheenColor ) * sheenAlbedo;
		indirectSpecular *= sheenEnergyComp;
		indirectDiffuse *= sheenEnergyComp;
	#endif
	reflectedLight.indirectSpecular += indirectSpecular;
	reflectedLight.indirectDiffuse += indirectDiffuse;
}
#define RE_Direct				RE_Direct_Physical
#define RE_Direct_RectArea		RE_Direct_RectArea_Physical
#define RE_IndirectDiffuse		RE_IndirectDiffuse_Physical
#define RE_IndirectSpecular		RE_IndirectSpecular_Physical
float computeSpecularOcclusion( const in float dotNV, const in float ambientOcclusion, const in float roughness ) {
	return saturate( pow( dotNV + ambientOcclusion, exp2( - 16.0 * roughness - 1.0 ) ) - 1.0 + ambientOcclusion );
}`,lights_fragment_begin:`
vec3 geometryPosition = - vViewPosition;
vec3 geometryNormal = normal;
vec3 geometryViewDir = ( isOrthographic ) ? vec3( 0, 0, 1 ) : normalize( vViewPosition );
vec3 geometryClearcoatNormal = vec3( 0.0 );
#ifdef USE_CLEARCOAT
	geometryClearcoatNormal = clearcoatNormal;
#endif
#ifdef USE_IRIDESCENCE
	float dotNVi = saturate( dot( normal, geometryViewDir ) );
	if ( material.iridescenceThickness == 0.0 ) {
		material.iridescence = 0.0;
	} else {
		material.iridescence = saturate( material.iridescence );
	}
	if ( material.iridescence > 0.0 ) {
		vec3 iridescenceFresnelDielectric = evalIridescence( 1.0, material.iridescenceIOR, dotNVi, material.iridescenceThickness, material.specularColor );
		vec3 iridescenceFresnelMetallic = evalIridescence( 1.0, material.iridescenceIOR, dotNVi, material.iridescenceThickness, material.diffuseColor );
		material.iridescenceFresnel = mix( iridescenceFresnelDielectric, iridescenceFresnelMetallic, material.metalness );
		material.iridescenceF0Dielectric = Schlick_to_F0( iridescenceFresnelDielectric, 1.0, dotNVi );
		material.iridescenceF0Metallic = Schlick_to_F0( iridescenceFresnelMetallic, 1.0, dotNVi );
	}
#endif
#ifdef STANDARD
	float dotNVms = saturate( dot( geometryNormal, geometryViewDir ) );
	material.dfg = texture2D( dfgLUT, vec2( material.roughness, dotNVms ) ).rg;
	#if ( NUM_SUN_LIGHTS > 0 || NUM_DIR_LIGHTS > 0 || NUM_POINT_LIGHTS > 0 || NUM_SPOT_LIGHTS > 0 )
		float EssMs = material.dfg.x + material.dfg.y;
		material.multiScatteringCompensation = 1.0 + material.specularColorBlended * ( 1.0 / EssMs - 1.0 );
	#endif
#endif
IncidentLight directLight;
#if ( NUM_POINT_LIGHTS > 0 ) && defined( RE_Direct )
	PointLight pointLight;
	#if defined( USE_SHADOWMAP ) && NUM_POINT_LIGHT_SHADOWS > 0
	PointLightShadow pointLightShadow;
	#endif
	#pragma unroll_loop_start
	for ( int i = 0; i < NUM_POINT_LIGHTS; i ++ ) {
		pointLight = pointLights[ i ];
		getPointLightInfo( pointLight, geometryPosition, directLight );
		#if defined( USE_SHADOWMAP ) && ( UNROLLED_LOOP_INDEX < NUM_POINT_LIGHT_SHADOWS ) && ( defined( SHADOWMAP_TYPE_PCF ) || defined( SHADOWMAP_TYPE_BASIC ) )
		pointLightShadow = pointLightShadows[ i ];
		directLight.color *= ( directLight.visible && receiveShadow ) ? getPointShadow( pointShadowMap[ i ], pointLightShadow.shadowMapSize, pointLightShadow.shadowIntensity, pointLightShadow.shadowBias, pointLightShadow.shadowRadius, vPointShadowCoord[ i ], pointLightShadow.shadowCameraNear, pointLightShadow.shadowCameraFar ) : 1.0;
		#endif
		RE_Direct( directLight, geometryPosition, geometryNormal, geometryViewDir, geometryClearcoatNormal, material, reflectedLight );
	}
	#pragma unroll_loop_end
#endif
#if ( NUM_SPOT_LIGHTS > 0 ) && defined( RE_Direct )
	SpotLight spotLight;
	vec4 spotColor;
	vec3 spotLightCoord;
	bool inSpotLightMap;
	#if defined( USE_SHADOWMAP ) && NUM_SPOT_LIGHT_SHADOWS > 0
	SpotLightShadow spotLightShadow;
	#endif
	#pragma unroll_loop_start
	for ( int i = 0; i < NUM_SPOT_LIGHTS; i ++ ) {
		spotLight = spotLights[ i ];
		getSpotLightInfo( spotLight, geometryPosition, directLight );
		#if ( UNROLLED_LOOP_INDEX < NUM_SPOT_LIGHT_SHADOWS_WITH_MAPS )
		#define SPOT_LIGHT_MAP_INDEX UNROLLED_LOOP_INDEX
		#elif ( UNROLLED_LOOP_INDEX < NUM_SPOT_LIGHT_SHADOWS )
		#define SPOT_LIGHT_MAP_INDEX NUM_SPOT_LIGHT_MAPS
		#else
		#define SPOT_LIGHT_MAP_INDEX ( UNROLLED_LOOP_INDEX - NUM_SPOT_LIGHT_SHADOWS + NUM_SPOT_LIGHT_SHADOWS_WITH_MAPS )
		#endif
		#if ( SPOT_LIGHT_MAP_INDEX < NUM_SPOT_LIGHT_MAPS )
			spotLightCoord = vSpotLightCoord[ i ].xyz / vSpotLightCoord[ i ].w;
			inSpotLightMap = all( lessThan( abs( spotLightCoord * 2. - 1. ), vec3( 1.0 ) ) );
			spotColor = texture2D( spotLightMap[ SPOT_LIGHT_MAP_INDEX ], spotLightCoord.xy );
			directLight.color = inSpotLightMap ? directLight.color * spotColor.rgb : directLight.color;
		#endif
		#undef SPOT_LIGHT_MAP_INDEX
		#if defined( USE_SHADOWMAP ) && ( UNROLLED_LOOP_INDEX < NUM_SPOT_LIGHT_SHADOWS )
		spotLightShadow = spotLightShadows[ i ];
		directLight.color *= ( directLight.visible && receiveShadow ) ? getShadow( spotShadowMap[ i ], spotLightShadow.shadowMapSize, spotLightShadow.shadowIntensity, spotLightShadow.shadowBias, spotLightShadow.shadowRadius, vSpotLightCoord[ i ] ) : 1.0;
		#endif
		RE_Direct( directLight, geometryPosition, geometryNormal, geometryViewDir, geometryClearcoatNormal, material, reflectedLight );
	}
	#pragma unroll_loop_end
#endif
#if ( NUM_SUN_LIGHTS > 0 ) && defined( RE_Direct )
	SunLight sunLight;
	#if defined( USE_SHADOWMAP ) && NUM_SUN_LIGHT_SHADOWS > 0
	SunLightShadow sunLightShadow;
	#endif
	#pragma unroll_loop_start
	for ( int i = 0; i < NUM_SUN_LIGHTS; i ++ ) {
		sunLight = sunLights[ i ];
		getSunLightInfo( sunLight, directLight );
		#if defined( USE_SHADOWMAP ) && ( UNROLLED_LOOP_INDEX < NUM_SUN_LIGHT_SHADOWS )
		sunLightShadow = sunLightShadows[ i ];
		directLight.color *= ( directLight.visible && receiveShadow ) ? getSunShadow( sunShadowMap[ i ], sunLightShadow, UNROLLED_LOOP_INDEX ) : 1.0;
		#endif
		RE_Direct( directLight, geometryPosition, geometryNormal, geometryViewDir, geometryClearcoatNormal, material, reflectedLight );
	}
	#pragma unroll_loop_end
#endif
#if ( NUM_DIR_LIGHTS > 0 ) && defined( RE_Direct )
	DirectionalLight directionalLight;
	#if defined( USE_SHADOWMAP ) && NUM_DIR_LIGHT_SHADOWS > 0
	DirectionalLightShadow directionalLightShadow;
	#endif
	#pragma unroll_loop_start
	for ( int i = 0; i < NUM_DIR_LIGHTS; i ++ ) {
		directionalLight = directionalLights[ i ];
		getDirectionalLightInfo( directionalLight, directLight );
		#if defined( USE_SHADOWMAP ) && ( UNROLLED_LOOP_INDEX < NUM_DIR_LIGHT_SHADOWS )
		directionalLightShadow = directionalLightShadows[ i ];
		directLight.color *= ( directLight.visible && receiveShadow ) ? getShadow( directionalShadowMap[ i ], directionalLightShadow.shadowMapSize, directionalLightShadow.shadowIntensity, directionalLightShadow.shadowBias, directionalLightShadow.shadowRadius, vDirectionalShadowCoord[ i ] ) : 1.0;
		#endif
		RE_Direct( directLight, geometryPosition, geometryNormal, geometryViewDir, geometryClearcoatNormal, material, reflectedLight );
	}
	#pragma unroll_loop_end
#endif
#if ( NUM_RECT_AREA_LIGHTS > 0 ) && defined( RE_Direct_RectArea )
	RectAreaLight rectAreaLight;
	#pragma unroll_loop_start
	for ( int i = 0; i < NUM_RECT_AREA_LIGHTS; i ++ ) {
		rectAreaLight = rectAreaLights[ i ];
		RE_Direct_RectArea( rectAreaLight, geometryPosition, geometryNormal, geometryViewDir, geometryClearcoatNormal, material, reflectedLight );
	}
	#pragma unroll_loop_end
#endif
#if defined( RE_IndirectDiffuse )
	vec3 iblIrradiance = vec3( 0.0 );
	vec3 irradiance = getAmbientLightIrradiance( ambientLightColor );
	#if defined( USE_LIGHT_PROBES )
		irradiance += getLightProbeIrradiance( lightProbe, geometryNormal );
	#endif
	#if ( NUM_HEMI_LIGHTS > 0 )
		#pragma unroll_loop_start
		for ( int i = 0; i < NUM_HEMI_LIGHTS; i ++ ) {
			irradiance += getHemisphereLightIrradiance( hemisphereLights[ i ], geometryNormal );
		}
		#pragma unroll_loop_end
	#endif
	#if NUM_LIGHT_PROBE_GRIDS > 0
		vec3 probeWorldPos = ( ( vec4( geometryPosition, 1.0 ) - viewMatrix[ 3 ] ) * viewMatrix ).xyz;
		vec3 probeWorldNormal = transformNormalByInverseViewMatrix( geometryNormal, viewMatrix );
		int probeGrid = - 1;
		#pragma unroll_loop_start
		for ( int i = 0; i < NUM_LIGHT_PROBE_GRIDS; i ++ ) {
			if ( isInsideLightProbeGrid( probeWorldPos, probeWorldNormal, probesMin[ i ], probesMax[ i ], probesResolution[ i ] ) ) probeGrid = UNROLLED_LOOP_INDEX;
		}
		#pragma unroll_loop_end
		#pragma unroll_loop_start
		for ( int i = 0; i < NUM_LIGHT_PROBE_GRIDS; i ++ ) {
			if ( probeGrid == UNROLLED_LOOP_INDEX ) irradiance += getLightProbeGridIrradiance( probesSH[ i ], probesMin[ i ], probesMax[ i ], probesResolution[ i ], probeWorldPos, probeWorldNormal );
		}
		#pragma unroll_loop_end
	#endif
#endif
#if defined( RE_IndirectSpecular )
	vec3 radiance = vec3( 0.0 );
	vec3 clearcoatRadiance = vec3( 0.0 );
#endif`,lights_fragment_maps:`#if defined( RE_IndirectDiffuse )
	#ifdef USE_LIGHTMAP
		vec4 lightMapTexel = texture2D( lightMap, vLightMapUv );
		vec3 lightMapIrradiance = lightMapTexel.rgb * lightMapIntensity;
		irradiance += lightMapIrradiance;
	#endif
	#if defined( USE_ENVMAP ) && defined( ENVMAP_TYPE_PMREM )
		#if defined( STANDARD ) || defined( LAMBERT ) || defined( PHONG )
			iblIrradiance += getIBLIrradiance( geometryNormal );
		#endif
	#endif
#endif
#if defined( USE_ENVMAP ) && defined( RE_IndirectSpecular )
	#ifdef USE_ANISOTROPY
		vec3 iblRadiance = getIBLAnisotropyRadiance( geometryViewDir, geometryNormal, material.roughness, material.anisotropyB, material.anisotropy );
	#else
		vec3 iblRadiance = getIBLRadiance( geometryViewDir, geometryNormal, material.roughness );
	#endif
	#ifdef USE_RETROREFLECTION
		#ifdef USE_ANISOTROPY
			vec3 retroIBLRadiance = getIBLAnisotropyRetroRadiance( geometryViewDir, geometryNormal, material.roughness, material.anisotropyB, material.anisotropy );
		#else
			vec3 retroIBLRadiance = getIBLRetroRadiance( geometryViewDir, geometryNormal, material.roughness );
		#endif
		iblRadiance = mix( iblRadiance, retroIBLRadiance, saturate( material.retroreflectivity ) );
	#endif
	radiance += iblRadiance;
	#ifdef USE_CLEARCOAT
		clearcoatRadiance += getIBLRadiance( geometryViewDir, geometryClearcoatNormal, material.clearcoatRoughness );
	#endif
#endif`,lights_fragment_end:`#if defined( RE_IndirectDiffuse )
	#if defined( LAMBERT ) || defined( PHONG )
		irradiance += iblIrradiance;
	#endif
	RE_IndirectDiffuse( irradiance, geometryPosition, geometryNormal, geometryViewDir, geometryClearcoatNormal, material, reflectedLight );
#endif
#if defined( RE_IndirectSpecular )
	RE_IndirectSpecular( radiance, iblIrradiance, clearcoatRadiance, geometryPosition, geometryNormal, geometryViewDir, geometryClearcoatNormal, material, reflectedLight );
#endif`,lightprobes_pars_fragment:`#if NUM_LIGHT_PROBE_GRIDS > 0
uniform highp sampler3D probesSH[ NUM_LIGHT_PROBE_GRIDS ];
uniform vec3 probesMin[ NUM_LIGHT_PROBE_GRIDS ];
uniform vec3 probesMax[ NUM_LIGHT_PROBE_GRIDS ];
uniform vec3 probesResolution[ NUM_LIGHT_PROBE_GRIDS ];
bool isInsideLightProbeGrid( vec3 worldPos, vec3 worldNormal, vec3 probesMin, vec3 probesMax, vec3 probesResolution ) {
	vec3 spacing = ( probesMax - probesMin ) / probesResolution;
	vec3 samplePos = worldPos + worldNormal * spacing * 0.5;
	vec3 margin = spacing * 0.01;
	return all( greaterThanEqual( samplePos, probesMin - margin ) ) && all( lessThanEqual( samplePos, probesMax + margin ) );
}
vec3 getLightProbeGridIrradiance( highp sampler3D probesSH, vec3 probesMin, vec3 probesMax, vec3 probesResolution, vec3 worldPos, vec3 worldNormal ) {
	vec3 res = probesResolution;
	vec3 gridRange = probesMax - probesMin;
	vec3 probeSpacing = gridRange / res;
	vec3 samplePos = worldPos + worldNormal * probeSpacing * 0.5;
	vec3 uvw = clamp( ( samplePos - probesMin ) / gridRange, 0.5 / res, 1.0 - 0.5 / res );
	float nz          = res.z;
	float paddedSlices = nz + 2.0;
	float atlasDepth  = 7.0 * paddedSlices;
	float uvZBase     = uvw.z * nz + 1.0;
	vec4 s0 = texture( probesSH, vec3( uvw.xy, ( uvZBase                       ) / atlasDepth ) );
	vec4 s1 = texture( probesSH, vec3( uvw.xy, ( uvZBase +       paddedSlices   ) / atlasDepth ) );
	vec4 s2 = texture( probesSH, vec3( uvw.xy, ( uvZBase + 2.0 * paddedSlices   ) / atlasDepth ) );
	vec4 s3 = texture( probesSH, vec3( uvw.xy, ( uvZBase + 3.0 * paddedSlices   ) / atlasDepth ) );
	vec4 s4 = texture( probesSH, vec3( uvw.xy, ( uvZBase + 4.0 * paddedSlices   ) / atlasDepth ) );
	vec4 s5 = texture( probesSH, vec3( uvw.xy, ( uvZBase + 5.0 * paddedSlices   ) / atlasDepth ) );
	vec4 s6 = texture( probesSH, vec3( uvw.xy, ( uvZBase + 6.0 * paddedSlices   ) / atlasDepth ) );
	vec3 c0 = s0.xyz;
	vec3 c1 = vec3( s0.w, s1.xy );
	vec3 c2 = vec3( s1.zw, s2.x );
	vec3 c3 = s2.yzw;
	vec3 c4 = s3.xyz;
	vec3 c5 = vec3( s3.w, s4.xy );
	vec3 c6 = vec3( s4.zw, s5.x );
	vec3 c7 = s5.yzw;
	vec3 c8 = s6.xyz;
	float x = worldNormal.x, y = worldNormal.y, z = worldNormal.z;
	vec3 result = c0 * 0.886227;
	result += c1 * 2.0 * 0.511664 * y;
	result += c2 * 2.0 * 0.511664 * z;
	result += c3 * 2.0 * 0.511664 * x;
	result += c4 * 2.0 * 0.429043 * x * y;
	result += c5 * 2.0 * 0.429043 * y * z;
	result += c6 * ( 0.743125 * z * z - 0.247708 );
	result += c7 * 2.0 * 0.429043 * x * z;
	result += c8 * 0.429043 * ( x * x - y * y );
	return max( result, vec3( 0.0 ) );
}
#endif`,logdepthbuf_fragment:`#if defined( USE_LOGARITHMIC_DEPTH_BUFFER )
	gl_FragDepth = vIsPerspective == 0.0 ? gl_FragCoord.z : log2( vFragDepth ) * logDepthBufFC * 0.5;
#endif`,logdepthbuf_pars_fragment:`#if defined( USE_LOGARITHMIC_DEPTH_BUFFER )
	uniform float logDepthBufFC;
	varying float vFragDepth;
	varying float vIsPerspective;
#endif`,logdepthbuf_pars_vertex:`#ifdef USE_LOGARITHMIC_DEPTH_BUFFER
	varying float vFragDepth;
	varying float vIsPerspective;
#endif`,logdepthbuf_vertex:`#ifdef USE_LOGARITHMIC_DEPTH_BUFFER
	vFragDepth = 1.0 + gl_Position.w;
	vIsPerspective = float( isPerspectiveMatrix( projectionMatrix ) );
#endif`,map_fragment:`#ifdef USE_MAP
	vec4 sampledDiffuseColor = texture2D( map, vMapUv );
	#ifdef DECODE_VIDEO_TEXTURE
		sampledDiffuseColor = sRGBTransferEOTF( sampledDiffuseColor );
	#endif
	diffuseColor *= sampledDiffuseColor;
#endif`,map_pars_fragment:`#ifdef USE_MAP
	uniform sampler2D map;
#endif`,map_particle_fragment:`#if defined( USE_MAP ) || defined( USE_ALPHAMAP )
	#if defined( USE_POINTS_UV )
		vec2 uv = vUv;
	#else
		vec2 uv = ( uvTransform * vec3( gl_PointCoord.x, 1.0 - gl_PointCoord.y, 1 ) ).xy;
	#endif
#endif
#ifdef USE_MAP
	diffuseColor *= texture2D( map, uv );
#endif
#ifdef USE_ALPHAMAP
	diffuseColor.a *= texture2D( alphaMap, uv ).g;
#endif`,map_particle_pars_fragment:`#if defined( USE_POINTS_UV )
	varying vec2 vUv;
#else
	#if defined( USE_MAP ) || defined( USE_ALPHAMAP )
		uniform mat3 uvTransform;
	#endif
#endif
#ifdef USE_MAP
	uniform sampler2D map;
#endif
#ifdef USE_ALPHAMAP
	uniform sampler2D alphaMap;
#endif`,metalnessmap_fragment:`float metalnessFactor = metalness;
#ifdef USE_METALNESSMAP
	vec4 texelMetalness = texture2D( metalnessMap, vMetalnessMapUv );
	metalnessFactor *= texelMetalness.b;
#endif`,metalnessmap_pars_fragment:`#ifdef USE_METALNESSMAP
	uniform sampler2D metalnessMap;
#endif`,morphinstance_vertex:`#ifdef USE_INSTANCING_MORPH
	float morphTargetInfluences[ MORPHTARGETS_COUNT ];
	float morphTargetBaseInfluence = texelFetch( morphTexture, ivec2( 0, gl_InstanceID ), 0 ).r;
	for ( int i = 0; i < MORPHTARGETS_COUNT; i ++ ) {
		morphTargetInfluences[i] =  texelFetch( morphTexture, ivec2( i + 1, gl_InstanceID ), 0 ).r;
	}
#endif`,morphcolor_vertex:`#if defined( USE_MORPHCOLORS )
	#if defined( USE_COLOR_ALPHA )
		vColor *= morphTargetBaseInfluence;
	#elif defined( USE_COLOR )
		vColor.rgb *= morphTargetBaseInfluence;
	#endif
	for ( int i = 0; i < MORPHTARGETS_COUNT; i ++ ) {
		#if defined( USE_COLOR_ALPHA )
			if ( morphTargetInfluences[ i ] != 0.0 ) vColor += getMorph( gl_VertexID, i, 2 ) * morphTargetInfluences[ i ];
		#elif defined( USE_COLOR )
			if ( morphTargetInfluences[ i ] != 0.0 ) vColor.rgb += getMorph( gl_VertexID, i, 2 ).rgb * morphTargetInfluences[ i ];
		#endif
	}
#endif`,morphnormal_vertex:`#ifdef USE_MORPHNORMALS
	objectNormal *= morphTargetBaseInfluence;
	for ( int i = 0; i < MORPHTARGETS_COUNT; i ++ ) {
		if ( morphTargetInfluences[ i ] != 0.0 ) objectNormal += getMorph( gl_VertexID, i, 1 ).xyz * morphTargetInfluences[ i ];
	}
#endif`,morphtarget_pars_vertex:`#ifdef USE_MORPHTARGETS
	#ifndef USE_INSTANCING_MORPH
		uniform float morphTargetBaseInfluence;
		uniform float morphTargetInfluences[ MORPHTARGETS_COUNT ];
	#endif
	uniform sampler2DArray morphTargetsTexture;
	uniform ivec2 morphTargetsTextureSize;
	vec4 getMorph( const in int vertexIndex, const in int morphTargetIndex, const in int offset ) {
		int texelIndex = vertexIndex * MORPHTARGETS_TEXTURE_STRIDE + offset;
		int y = texelIndex / morphTargetsTextureSize.x;
		int x = texelIndex - y * morphTargetsTextureSize.x;
		ivec3 morphUV = ivec3( x, y, morphTargetIndex );
		return texelFetch( morphTargetsTexture, morphUV, 0 );
	}
#endif`,morphtarget_vertex:`#ifdef USE_MORPHTARGETS
	transformed *= morphTargetBaseInfluence;
	for ( int i = 0; i < MORPHTARGETS_COUNT; i ++ ) {
		if ( morphTargetInfluences[ i ] != 0.0 ) transformed += getMorph( gl_VertexID, i, 0 ).xyz * morphTargetInfluences[ i ];
	}
#endif`,normal_fragment_begin:`float faceDirection = gl_FrontFacing ? 1.0 : - 1.0;
#ifdef FLAT_SHADED
	vec3 fdx = dFdx( vViewPosition );
	vec3 fdy = dFdy( vViewPosition );
	vec3 normal = normalize( cross( fdx, fdy ) );
#else
	vec3 normal = normalize( vNormal );
	#ifdef DOUBLE_SIDED
		normal *= faceDirection;
	#endif
#endif
#if defined( USE_NORMALMAP_TANGENTSPACE ) || defined( USE_CLEARCOAT_NORMALMAP ) || defined( USE_ANISOTROPY )
	#ifdef USE_TANGENT
		mat3 tbn = mat3( normalize( vTangent ), normalize( vBitangent ), normal );
	#else
		mat3 tbn = getTangentFrame( - vViewPosition, normal,
		#if defined( USE_NORMALMAP )
			vNormalMapUv
		#elif defined( USE_CLEARCOAT_NORMALMAP )
			vClearcoatNormalMapUv
		#else
			vUv
		#endif
		);
	#endif
	#ifdef DOUBLE_SIDED
		tbn[0] *= faceDirection;
		tbn[1] *= faceDirection;
	#endif
#endif
#ifdef USE_CLEARCOAT_NORMALMAP
	#ifdef USE_TANGENT
		mat3 tbn2 = mat3( normalize( vTangent ), normalize( vBitangent ), normal );
	#else
		mat3 tbn2 = getTangentFrame( - vViewPosition, normal, vClearcoatNormalMapUv );
	#endif
	#ifdef DOUBLE_SIDED
		tbn2[0] *= faceDirection;
		tbn2[1] *= faceDirection;
	#endif
#endif
vec3 nonPerturbedNormal = normal;`,normal_fragment_maps:`#ifdef USE_NORMALMAP_OBJECTSPACE
	normal = texture2D( normalMap, vNormalMapUv ).xyz * 2.0 - 1.0;
	#ifdef FLIP_SIDED
		normal = - normal;
	#endif
	#ifdef DOUBLE_SIDED
		normal = normal * faceDirection;
	#endif
	normal = normalize( normalMatrix * normal );
#elif defined( USE_NORMALMAP_TANGENTSPACE )
	vec3 mapN = texture2D( normalMap, vNormalMapUv ).xyz * 2.0 - 1.0;
	#if defined( USE_PACKED_NORMALMAP )
		mapN = vec3( mapN.xy, sqrt( saturate( 1.0 - dot( mapN.xy, mapN.xy ) ) ) );
	#endif
	mapN.xy *= normalScale;
	normal = normalize( tbn * mapN );
#elif defined( USE_BUMPMAP )
	normal = perturbNormalArb( - vViewPosition, normal, dHdxy_fwd(), faceDirection );
#endif`,normal_pars_fragment:`#ifndef FLAT_SHADED
	varying vec3 vNormal;
	#ifdef USE_TANGENT
		varying vec3 vTangent;
		varying vec3 vBitangent;
	#endif
#endif`,normal_pars_vertex:`#ifndef FLAT_SHADED
	varying vec3 vNormal;
	#ifdef USE_TANGENT
		varying vec3 vTangent;
		varying vec3 vBitangent;
	#endif
#endif`,normal_vertex:`#ifndef FLAT_SHADED
	vNormal = normalize( transformedNormal );
	#ifdef USE_TANGENT
		vTangent = normalize( transformedTangent );
		vBitangent = normalize( cross( vNormal, vTangent ) * tangent.w );
		#ifdef FLIP_SIDED
			vBitangent = - vBitangent;
		#endif
	#endif
#endif`,normalmap_pars_fragment:`#ifdef USE_NORMALMAP
	uniform sampler2D normalMap;
	uniform vec2 normalScale;
#endif
#ifdef USE_NORMALMAP_OBJECTSPACE
	uniform mat3 normalMatrix;
#endif
#if ! defined ( USE_TANGENT ) && ( defined ( USE_NORMALMAP_TANGENTSPACE ) || defined ( USE_CLEARCOAT_NORMALMAP ) || defined( USE_ANISOTROPY ) )
	mat3 getTangentFrame( vec3 eye_pos, vec3 surf_norm, vec2 uv ) {
		vec3 q0 = dFdx( eye_pos.xyz );
		vec3 q1 = dFdy( eye_pos.xyz );
		vec2 st0 = dFdx( uv.st );
		vec2 st1 = dFdy( uv.st );
		vec3 N = surf_norm;
		vec3 q1perp = cross( q1, N );
		vec3 q0perp = cross( N, q0 );
		vec3 T = q1perp * st0.x + q0perp * st1.x;
		vec3 B = q1perp * st0.y + q0perp * st1.y;
		float det = max( dot( T, T ), dot( B, B ) );
		float scale = ( det == 0.0 ) ? 0.0 : inversesqrt( det );
		return mat3( T * scale, B * scale, N );
	}
#endif`,clearcoat_normal_fragment_begin:`#ifdef USE_CLEARCOAT
	vec3 clearcoatNormal = nonPerturbedNormal;
#endif`,clearcoat_normal_fragment_maps:`#ifdef USE_CLEARCOAT_NORMALMAP
	vec3 clearcoatMapN = texture2D( clearcoatNormalMap, vClearcoatNormalMapUv ).xyz * 2.0 - 1.0;
	clearcoatMapN.xy *= clearcoatNormalScale;
	clearcoatNormal = normalize( tbn2 * clearcoatMapN );
#endif`,clearcoat_pars_fragment:`#ifdef USE_CLEARCOATMAP
	uniform sampler2D clearcoatMap;
#endif
#ifdef USE_CLEARCOAT_NORMALMAP
	uniform sampler2D clearcoatNormalMap;
	uniform vec2 clearcoatNormalScale;
#endif
#ifdef USE_CLEARCOAT_ROUGHNESSMAP
	uniform sampler2D clearcoatRoughnessMap;
#endif`,iridescence_pars_fragment:`#ifdef USE_IRIDESCENCEMAP
	uniform sampler2D iridescenceMap;
#endif
#ifdef USE_IRIDESCENCE_THICKNESSMAP
	uniform sampler2D iridescenceThicknessMap;
#endif`,opaque_fragment:`#ifdef OPAQUE
diffuseColor.a = 1.0;
#endif
#ifdef USE_TRANSMISSION
diffuseColor.a *= material.transmissionAlpha;
#endif
gl_FragColor = vec4( outgoingLight, diffuseColor.a );`,packing:`vec3 packNormalToRGB( const in vec3 normal ) {
	return normalize( normal ) * 0.5 + 0.5;
}
vec3 unpackRGBToNormal( const in vec3 rgb ) {
	return 2.0 * rgb.xyz - 1.0;
}
const float PackUpscale = 256. / 255.;const float UnpackDownscale = 255. / 256.;const float ShiftRight8 = 1. / 256.;
const float Inv255 = 1. / 255.;
const vec4 PackFactors = vec4( 1.0, 256.0, 256.0 * 256.0, 256.0 * 256.0 * 256.0 );
const vec2 UnpackFactors2 = vec2( UnpackDownscale, 1.0 / PackFactors.g );
const vec3 UnpackFactors3 = vec3( UnpackDownscale / PackFactors.rg, 1.0 / PackFactors.b );
const vec4 UnpackFactors4 = vec4( UnpackDownscale / PackFactors.rgb, 1.0 / PackFactors.a );
vec4 packDepthToRGBA( const in float v ) {
	if( v <= 0.0 )
		return vec4( 0., 0., 0., 0. );
	if( v >= 1.0 )
		return vec4( 1., 1., 1., 1. );
	float vuf;
	float af = modf( v * PackFactors.a, vuf );
	float bf = modf( vuf * ShiftRight8, vuf );
	float gf = modf( vuf * ShiftRight8, vuf );
	return vec4( vuf * Inv255, gf * PackUpscale, bf * PackUpscale, af );
}
vec3 packDepthToRGB( const in float v ) {
	if( v <= 0.0 )
		return vec3( 0., 0., 0. );
	if( v >= 1.0 )
		return vec3( 1., 1., 1. );
	float vuf;
	float bf = modf( v * PackFactors.b, vuf );
	float gf = modf( vuf * ShiftRight8, vuf );
	return vec3( vuf * Inv255, gf * PackUpscale, bf );
}
vec2 packDepthToRG( const in float v ) {
	if( v <= 0.0 )
		return vec2( 0., 0. );
	if( v >= 1.0 )
		return vec2( 1., 1. );
	float vuf;
	float gf = modf( v * 256., vuf );
	return vec2( vuf * Inv255, gf );
}
float unpackRGBAToDepth( const in vec4 v ) {
	return dot( v, UnpackFactors4 );
}
float unpackRGBToDepth( const in vec3 v ) {
	return dot( v, UnpackFactors3 );
}
float unpackRGToDepth( const in vec2 v ) {
	return v.r * UnpackFactors2.r + v.g * UnpackFactors2.g;
}
vec4 pack2HalfToRGBA( const in vec2 v ) {
	vec4 r = vec4( v.x, fract( v.x * 255.0 ), v.y, fract( v.y * 255.0 ) );
	return vec4( r.x - r.y / 255.0, r.y, r.z - r.w / 255.0, r.w );
}
vec2 unpackRGBATo2Half( const in vec4 v ) {
	return vec2( v.x + ( v.y / 255.0 ), v.z + ( v.w / 255.0 ) );
}
float viewZToOrthographicDepth( const in float viewZ, const in float near, const in float far ) {
	return ( viewZ + near ) / ( near - far );
}
float orthographicDepthToViewZ( const in float depth, const in float near, const in float far ) {
	#ifdef USE_REVERSED_DEPTH_BUFFER
	
		return depth * ( far - near ) - far;
	#else
		return depth * ( near - far ) - near;
	#endif
}
float viewZToPerspectiveDepth( const in float viewZ, const in float near, const in float far ) {
	return ( ( near + viewZ ) * far ) / ( ( far - near ) * viewZ );
}
float perspectiveDepthToViewZ( const in float depth, const in float near, const in float far ) {
	
	#ifdef USE_REVERSED_DEPTH_BUFFER
		return ( near * far ) / ( ( near - far ) * depth - near );
	#else
		return ( near * far ) / ( ( far - near ) * depth - far );
	#endif
}`,premultiplied_alpha_fragment:`#ifdef PREMULTIPLIED_ALPHA
	gl_FragColor.rgb *= gl_FragColor.a;
#endif`,project_vertex:`vec4 mvPosition = vec4( transformed, 1.0 );
#ifdef USE_BATCHING
	mvPosition = batchingMatrix * mvPosition;
#endif
#ifdef USE_INSTANCING
	mvPosition = instanceMatrix * mvPosition;
#endif
mvPosition = modelViewMatrix * mvPosition;
gl_Position = projectionMatrix * mvPosition;`,dithering_fragment:`#ifdef DITHERING
	gl_FragColor.rgb = dithering( gl_FragColor.rgb );
#endif`,dithering_pars_fragment:`#ifdef DITHERING
	vec3 dithering( vec3 color ) {
		float grid_position = rand( gl_FragCoord.xy );
		vec3 dither_shift_RGB = vec3( 0.25 / 255.0, -0.25 / 255.0, 0.25 / 255.0 );
		dither_shift_RGB = mix( 2.0 * dither_shift_RGB, -2.0 * dither_shift_RGB, grid_position );
		return color + dither_shift_RGB;
	}
#endif`,roughnessmap_fragment:`float roughnessFactor = roughness;
#ifdef USE_ROUGHNESSMAP
	vec4 texelRoughness = texture2D( roughnessMap, vRoughnessMapUv );
	roughnessFactor *= texelRoughness.g;
#endif`,roughnessmap_pars_fragment:`#ifdef USE_ROUGHNESSMAP
	uniform sampler2D roughnessMap;
#endif`,shadowmap_pars_fragment:`#if NUM_SPOT_LIGHT_COORDS > 0
	varying vec4 vSpotLightCoord[ NUM_SPOT_LIGHT_COORDS ];
#endif
#if NUM_SPOT_LIGHT_MAPS > 0
	uniform sampler2D spotLightMap[ NUM_SPOT_LIGHT_MAPS ];
#endif
#ifdef USE_SHADOWMAP
	#if NUM_SUN_LIGHT_SHADOWS > 0
		#define SUN_LIGHT_CASCADES 2
		#if defined( SHADOWMAP_TYPE_PCF )
			uniform sampler2DShadow sunShadowMap[ NUM_SUN_LIGHT_SHADOWS ];
		#else
			uniform sampler2D sunShadowMap[ NUM_SUN_LIGHT_SHADOWS ];
		#endif
		uniform mat4 sunShadowMatrix[ NUM_SUN_LIGHT_SHADOWS * SUN_LIGHT_CASCADES ];
		uniform vec4 sunShadowCascade[ NUM_SUN_LIGHT_SHADOWS * SUN_LIGHT_CASCADES ];
		varying vec4 vSunShadowWorldPosition;
		varying vec3 vSunShadowWorldNormal;
		struct SunLightShadow {
			float shadowIntensity;
			float shadowBias;
			float shadowNormalBias;
			float shadowRadius;
			vec2 shadowMapSize;
		};
		uniform SunLightShadow sunLightShadows[ NUM_SUN_LIGHT_SHADOWS ];
	#endif
	#if NUM_DIR_LIGHT_SHADOWS > 0
		#if defined( SHADOWMAP_TYPE_PCF )
			uniform sampler2DShadow directionalShadowMap[ NUM_DIR_LIGHT_SHADOWS ];
		#else
			uniform sampler2D directionalShadowMap[ NUM_DIR_LIGHT_SHADOWS ];
		#endif
		varying vec4 vDirectionalShadowCoord[ NUM_DIR_LIGHT_SHADOWS ];
		struct DirectionalLightShadow {
			float shadowIntensity;
			float shadowBias;
			float shadowNormalBias;
			float shadowRadius;
			vec2 shadowMapSize;
		};
		uniform DirectionalLightShadow directionalLightShadows[ NUM_DIR_LIGHT_SHADOWS ];
	#endif
	#if NUM_SPOT_LIGHT_SHADOWS > 0
		#if defined( SHADOWMAP_TYPE_PCF )
			uniform sampler2DShadow spotShadowMap[ NUM_SPOT_LIGHT_SHADOWS ];
		#else
			uniform sampler2D spotShadowMap[ NUM_SPOT_LIGHT_SHADOWS ];
		#endif
		struct SpotLightShadow {
			float shadowIntensity;
			float shadowBias;
			float shadowNormalBias;
			float shadowRadius;
			vec2 shadowMapSize;
		};
		uniform SpotLightShadow spotLightShadows[ NUM_SPOT_LIGHT_SHADOWS ];
	#endif
	#if NUM_POINT_LIGHT_SHADOWS > 0
		#if defined( SHADOWMAP_TYPE_PCF )
			uniform samplerCubeShadow pointShadowMap[ NUM_POINT_LIGHT_SHADOWS ];
		#elif defined( SHADOWMAP_TYPE_BASIC )
			uniform samplerCube pointShadowMap[ NUM_POINT_LIGHT_SHADOWS ];
		#endif
		varying vec4 vPointShadowCoord[ NUM_POINT_LIGHT_SHADOWS ];
		struct PointLightShadow {
			float shadowIntensity;
			float shadowBias;
			float shadowNormalBias;
			float shadowRadius;
			vec2 shadowMapSize;
			float shadowCameraNear;
			float shadowCameraFar;
		};
		uniform PointLightShadow pointLightShadows[ NUM_POINT_LIGHT_SHADOWS ];
	#endif
	#if defined( SHADOWMAP_TYPE_PCF )
		float interleavedGradientNoise( vec2 position ) {
			return fract( 52.9829189 * fract( dot( position, vec2( 0.06711056, 0.00583715 ) ) ) );
		}
		vec2 vogelDiskSample( int sampleIndex, int samplesCount, float phi ) {
			const float goldenAngle = 2.399963229728653;
			float r = sqrt( ( float( sampleIndex ) + 0.5 ) / float( samplesCount ) );
			float theta = float( sampleIndex ) * goldenAngle + phi;
			return vec2( cos( theta ), sin( theta ) ) * r;
		}
	#endif
	#if defined( SHADOWMAP_TYPE_PCF )
		float getShadow( sampler2DShadow shadowMap, vec2 shadowMapSize, float shadowIntensity, float shadowBias, float shadowRadius, vec4 shadowCoord ) {
			float shadow = 1.0;
			shadowCoord.xyz /= shadowCoord.w;
			#ifdef USE_REVERSED_DEPTH_BUFFER
				shadowCoord.z -= shadowBias;
			#else
				shadowCoord.z += shadowBias;
			#endif
			bool inFrustum = shadowCoord.x >= 0.0 && shadowCoord.x <= 1.0 && shadowCoord.y >= 0.0 && shadowCoord.y <= 1.0;
			bool frustumTest = inFrustum && shadowCoord.z <= 1.0;
			if ( frustumTest ) {
				vec2 texelSize = vec2( 1.0 ) / shadowMapSize;
				vec2 radius = shadowRadius * texelSize;
				float phi = interleavedGradientNoise( gl_FragCoord.xy ) * PI2;
				shadow = (
					texture( shadowMap, vec3( shadowCoord.xy + vogelDiskSample( 0, 5, phi ) * radius, shadowCoord.z ) ) +
					texture( shadowMap, vec3( shadowCoord.xy + vogelDiskSample( 1, 5, phi ) * radius, shadowCoord.z ) ) +
					texture( shadowMap, vec3( shadowCoord.xy + vogelDiskSample( 2, 5, phi ) * radius, shadowCoord.z ) ) +
					texture( shadowMap, vec3( shadowCoord.xy + vogelDiskSample( 3, 5, phi ) * radius, shadowCoord.z ) ) +
					texture( shadowMap, vec3( shadowCoord.xy + vogelDiskSample( 4, 5, phi ) * radius, shadowCoord.z ) )
				) * 0.2;
			}
			return mix( 1.0, shadow, shadowIntensity );
		}
	#elif defined( SHADOWMAP_TYPE_VSM )
		float getShadow( sampler2D shadowMap, vec2 shadowMapSize, float shadowIntensity, float shadowBias, float shadowRadius, vec4 shadowCoord ) {
			float shadow = 1.0;
			shadowCoord.xyz /= shadowCoord.w;
			#ifdef USE_REVERSED_DEPTH_BUFFER
				shadowCoord.z -= shadowBias;
			#else
				shadowCoord.z += shadowBias;
			#endif
			bool inFrustum = shadowCoord.x >= 0.0 && shadowCoord.x <= 1.0 && shadowCoord.y >= 0.0 && shadowCoord.y <= 1.0;
			bool frustumTest = inFrustum && shadowCoord.z <= 1.0;
			if ( frustumTest ) {
				vec2 distribution = texture2D( shadowMap, shadowCoord.xy ).rg;
				float mean = distribution.x;
				float variance = distribution.y * distribution.y;
				#ifdef USE_REVERSED_DEPTH_BUFFER
					float hard_shadow = step( mean, shadowCoord.z );
				#else
					float hard_shadow = step( shadowCoord.z, mean );
				#endif
				
				if ( hard_shadow == 1.0 ) {
					shadow = 1.0;
				} else {
					variance = max( variance, 0.0000001 );
					float d = shadowCoord.z - mean;
					float p_max = variance / ( variance + d * d );
					p_max = clamp( ( p_max - 0.3 ) / 0.65, 0.0, 1.0 );
					shadow = max( hard_shadow, p_max );
				}
			}
			return mix( 1.0, shadow, shadowIntensity );
		}
	#else
		float getShadow( sampler2D shadowMap, vec2 shadowMapSize, float shadowIntensity, float shadowBias, float shadowRadius, vec4 shadowCoord ) {
			float shadow = 1.0;
			shadowCoord.xyz /= shadowCoord.w;
			#ifdef USE_REVERSED_DEPTH_BUFFER
				shadowCoord.z -= shadowBias;
			#else
				shadowCoord.z += shadowBias;
			#endif
			bool inFrustum = shadowCoord.x >= 0.0 && shadowCoord.x <= 1.0 && shadowCoord.y >= 0.0 && shadowCoord.y <= 1.0;
			bool frustumTest = inFrustum && shadowCoord.z <= 1.0;
			if ( frustumTest ) {
				float depth = texture2D( shadowMap, shadowCoord.xy ).r;
				#ifdef USE_REVERSED_DEPTH_BUFFER
					shadow = step( depth, shadowCoord.z );
				#else
					shadow = step( shadowCoord.z, depth );
				#endif
			}
			return mix( 1.0, shadow, shadowIntensity );
		}
	#endif
	#if NUM_SUN_LIGHT_SHADOWS > 0
		float getSunShadow(
			#if defined( SHADOWMAP_TYPE_PCF )
				sampler2DShadow shadowMap,
			#else
				sampler2D shadowMap,
			#endif
			SunLightShadow sunLightShadow,
			int shadowIndex
		) {
			vec4 shadowWorldPosition = vec4( vSunShadowWorldPosition.xyz + vSunShadowWorldNormal * sunLightShadow.shadowNormalBias, 1.0 );
			float viewDepth = vSunShadowWorldPosition.w;
			int cascadeOffset = shadowIndex * SUN_LIGHT_CASCADES;
			float shadow = 1.0;
			for ( int i = SUN_LIGHT_CASCADES - 1; i >= 0; i -- ) {
				vec4 cascade = sunShadowCascade[ cascadeOffset + i ];
				if ( viewDepth >= cascade.x && viewDepth < cascade.y ) {
					float cascadeShadow = getShadow(
						shadowMap,
						sunLightShadow.shadowMapSize,
						sunLightShadow.shadowIntensity,
						sunLightShadow.shadowBias,
						sunLightShadow.shadowRadius,
						sunShadowMatrix[ cascadeOffset + i ] * shadowWorldPosition
					);
					shadow = mix( cascadeShadow, shadow, smoothstep( cascade.z, cascade.y, viewDepth ) );
				}
			}
			return shadow;
		}
	#endif
	#if NUM_POINT_LIGHT_SHADOWS > 0
	#if defined( SHADOWMAP_TYPE_PCF )
	float getPointShadow( samplerCubeShadow shadowMap, vec2 shadowMapSize, float shadowIntensity, float shadowBias, float shadowRadius, vec4 shadowCoord, float shadowCameraNear, float shadowCameraFar ) {
		float shadow = 1.0;
		vec3 lightToPosition = shadowCoord.xyz;
		vec3 bd3D = normalize( lightToPosition );
		vec3 absVec = abs( lightToPosition );
		float viewSpaceZ = max( max( absVec.x, absVec.y ), absVec.z );
		if ( viewSpaceZ - shadowCameraFar <= 0.0 && viewSpaceZ - shadowCameraNear >= 0.0 ) {
			float lightDistance = length( lightToPosition );
			float depthRange = shadowCameraFar - shadowCameraNear;
			#ifdef USE_REVERSED_DEPTH_BUFFER
				float depthA = - shadowCameraNear / depthRange - shadowBias;
				float depthB = shadowCameraNear * shadowCameraFar / ( depthRange * lightDistance );
			#else
				float depthA = shadowCameraFar / depthRange + shadowBias;
				float depthB = - shadowCameraNear * shadowCameraFar / ( depthRange * lightDistance );
			#endif
			float texelSize = shadowRadius / shadowMapSize.x;
			vec3 absDir = abs( bd3D );
			vec3 tangent = absDir.x > absDir.z ? vec3( 0.0, 1.0, 0.0 ) : vec3( 1.0, 0.0, 0.0 );
			tangent = normalize( cross( bd3D, tangent ) );
			vec3 bitangent = cross( bd3D, tangent );
			float phi = interleavedGradientNoise( gl_FragCoord.xy ) * PI2;
			shadow = 0.0;
			for ( int i = 0; i < 5; i ++ ) {
				vec2 offset = vogelDiskSample( i, 5, phi );
				vec3 direction = bd3D + ( tangent * offset.x + bitangent * offset.y ) * texelSize;
				vec3 absDirection = abs( direction );
				float dp = depthA + depthB / max( max( absDirection.x, absDirection.y ), absDirection.z );
				shadow += texture( shadowMap, vec4( direction, dp ) );
			}
			shadow *= 0.2;
		}
		return mix( 1.0, shadow, shadowIntensity );
	}
	#elif defined( SHADOWMAP_TYPE_BASIC )
	float getPointShadow( samplerCube shadowMap, vec2 shadowMapSize, float shadowIntensity, float shadowBias, float shadowRadius, vec4 shadowCoord, float shadowCameraNear, float shadowCameraFar ) {
		float shadow = 1.0;
		vec3 lightToPosition = shadowCoord.xyz;
		vec3 absVec = abs( lightToPosition );
		float viewSpaceZ = max( max( absVec.x, absVec.y ), absVec.z );
		if ( viewSpaceZ - shadowCameraFar <= 0.0 && viewSpaceZ - shadowCameraNear >= 0.0 ) {
			float dp = ( shadowCameraFar * ( viewSpaceZ - shadowCameraNear ) ) / ( viewSpaceZ * ( shadowCameraFar - shadowCameraNear ) );
			dp += shadowBias;
			vec3 bd3D = normalize( lightToPosition );
			float depth = textureCube( shadowMap, bd3D ).r;
			#ifdef USE_REVERSED_DEPTH_BUFFER
				depth = 1.0 - depth;
			#endif
			shadow = step( dp, depth );
		}
		return mix( 1.0, shadow, shadowIntensity );
	}
	#endif
	#endif
#endif`,shadowmap_pars_vertex:`#if NUM_SPOT_LIGHT_COORDS > 0
	uniform mat4 spotLightMatrix[ NUM_SPOT_LIGHT_COORDS ];
	varying vec4 vSpotLightCoord[ NUM_SPOT_LIGHT_COORDS ];
#endif
#ifdef USE_SHADOWMAP
	#if NUM_SUN_LIGHT_SHADOWS > 0
		varying vec4 vSunShadowWorldPosition;
		varying vec3 vSunShadowWorldNormal;
	#endif
	#if NUM_DIR_LIGHT_SHADOWS > 0
		uniform mat4 directionalShadowMatrix[ NUM_DIR_LIGHT_SHADOWS ];
		varying vec4 vDirectionalShadowCoord[ NUM_DIR_LIGHT_SHADOWS ];
		struct DirectionalLightShadow {
			float shadowIntensity;
			float shadowBias;
			float shadowNormalBias;
			float shadowRadius;
			vec2 shadowMapSize;
		};
		uniform DirectionalLightShadow directionalLightShadows[ NUM_DIR_LIGHT_SHADOWS ];
	#endif
	#if NUM_SPOT_LIGHT_SHADOWS > 0
		struct SpotLightShadow {
			float shadowIntensity;
			float shadowBias;
			float shadowNormalBias;
			float shadowRadius;
			vec2 shadowMapSize;
		};
		uniform SpotLightShadow spotLightShadows[ NUM_SPOT_LIGHT_SHADOWS ];
	#endif
	#if NUM_POINT_LIGHT_SHADOWS > 0
		uniform mat4 pointShadowMatrix[ NUM_POINT_LIGHT_SHADOWS ];
		varying vec4 vPointShadowCoord[ NUM_POINT_LIGHT_SHADOWS ];
		struct PointLightShadow {
			float shadowIntensity;
			float shadowBias;
			float shadowNormalBias;
			float shadowRadius;
			vec2 shadowMapSize;
			float shadowCameraNear;
			float shadowCameraFar;
		};
		uniform PointLightShadow pointLightShadows[ NUM_POINT_LIGHT_SHADOWS ];
	#endif
#endif`,shadowmap_vertex:`#if ( defined( USE_SHADOWMAP ) && ( NUM_DIR_LIGHT_SHADOWS > 0 || NUM_SUN_LIGHT_SHADOWS > 0 || NUM_POINT_LIGHT_SHADOWS > 0 ) ) || ( NUM_SPOT_LIGHT_COORDS > 0 )
	#ifdef HAS_NORMAL
		vec3 shadowWorldNormal = transformNormalByInverseViewMatrix( transformedNormal, viewMatrix );
	#else
		vec3 shadowWorldNormal = vec3( 0.0 );
	#endif
	vec4 shadowWorldPosition;
#endif
#if defined( USE_SHADOWMAP )
	#if NUM_SUN_LIGHT_SHADOWS > 0
		vSunShadowWorldPosition = vec4( worldPosition.xyz, - mvPosition.z );
		vSunShadowWorldNormal = shadowWorldNormal;
	#endif
	#if NUM_DIR_LIGHT_SHADOWS > 0
		#pragma unroll_loop_start
		for ( int i = 0; i < NUM_DIR_LIGHT_SHADOWS; i ++ ) {
			shadowWorldPosition = worldPosition + vec4( shadowWorldNormal * directionalLightShadows[ i ].shadowNormalBias, 0 );
			vDirectionalShadowCoord[ i ] = directionalShadowMatrix[ i ] * shadowWorldPosition;
		}
		#pragma unroll_loop_end
	#endif
	#if NUM_POINT_LIGHT_SHADOWS > 0
		#pragma unroll_loop_start
		for ( int i = 0; i < NUM_POINT_LIGHT_SHADOWS; i ++ ) {
			shadowWorldPosition = worldPosition + vec4( shadowWorldNormal * pointLightShadows[ i ].shadowNormalBias, 0 );
			vPointShadowCoord[ i ] = pointShadowMatrix[ i ] * shadowWorldPosition;
		}
		#pragma unroll_loop_end
	#endif
#endif
#if NUM_SPOT_LIGHT_COORDS > 0
	#pragma unroll_loop_start
	for ( int i = 0; i < NUM_SPOT_LIGHT_COORDS; i ++ ) {
		shadowWorldPosition = worldPosition;
		#if ( defined( USE_SHADOWMAP ) && UNROLLED_LOOP_INDEX < NUM_SPOT_LIGHT_SHADOWS )
			shadowWorldPosition.xyz += shadowWorldNormal * spotLightShadows[ i ].shadowNormalBias;
		#endif
		vSpotLightCoord[ i ] = spotLightMatrix[ i ] * shadowWorldPosition;
	}
	#pragma unroll_loop_end
#endif`,shadowmask_pars_fragment:`float getShadowMask() {
	float shadow = 1.0;
	#ifdef USE_SHADOWMAP
	#if NUM_SUN_LIGHT_SHADOWS > 0
	SunLightShadow sunLight;
	#pragma unroll_loop_start
	for ( int i = 0; i < NUM_SUN_LIGHT_SHADOWS; i ++ ) {
		sunLight = sunLightShadows[ i ];
		shadow *= receiveShadow ? getSunShadow( sunShadowMap[ i ], sunLight, UNROLLED_LOOP_INDEX ) : 1.0;
	}
	#pragma unroll_loop_end
	#endif
	#if NUM_DIR_LIGHT_SHADOWS > 0
	DirectionalLightShadow directionalLight;
	#pragma unroll_loop_start
	for ( int i = 0; i < NUM_DIR_LIGHT_SHADOWS; i ++ ) {
		directionalLight = directionalLightShadows[ i ];
		shadow *= receiveShadow ? getShadow( directionalShadowMap[ i ], directionalLight.shadowMapSize, directionalLight.shadowIntensity, directionalLight.shadowBias, directionalLight.shadowRadius, vDirectionalShadowCoord[ i ] ) : 1.0;
	}
	#pragma unroll_loop_end
	#endif
	#if NUM_SPOT_LIGHT_SHADOWS > 0
	SpotLightShadow spotLight;
	#pragma unroll_loop_start
	for ( int i = 0; i < NUM_SPOT_LIGHT_SHADOWS; i ++ ) {
		spotLight = spotLightShadows[ i ];
		shadow *= receiveShadow ? getShadow( spotShadowMap[ i ], spotLight.shadowMapSize, spotLight.shadowIntensity, spotLight.shadowBias, spotLight.shadowRadius, vSpotLightCoord[ i ] ) : 1.0;
	}
	#pragma unroll_loop_end
	#endif
	#if NUM_POINT_LIGHT_SHADOWS > 0 && ( defined( SHADOWMAP_TYPE_PCF ) || defined( SHADOWMAP_TYPE_BASIC ) )
	PointLightShadow pointLight;
	#pragma unroll_loop_start
	for ( int i = 0; i < NUM_POINT_LIGHT_SHADOWS; i ++ ) {
		pointLight = pointLightShadows[ i ];
		shadow *= receiveShadow ? getPointShadow( pointShadowMap[ i ], pointLight.shadowMapSize, pointLight.shadowIntensity, pointLight.shadowBias, pointLight.shadowRadius, vPointShadowCoord[ i ], pointLight.shadowCameraNear, pointLight.shadowCameraFar ) : 1.0;
	}
	#pragma unroll_loop_end
	#endif
	#endif
	return shadow;
}`,skinbase_vertex:`#ifdef USE_SKINNING
	mat4 boneMatX = getBoneMatrix( skinIndex.x );
	mat4 boneMatY = getBoneMatrix( skinIndex.y );
	mat4 boneMatZ = getBoneMatrix( skinIndex.z );
	mat4 boneMatW = getBoneMatrix( skinIndex.w );
#endif`,skinning_pars_vertex:`#ifdef USE_SKINNING
	uniform mat4 bindMatrix;
	uniform mat4 bindMatrixInverse;
	uniform highp sampler2D boneTexture;
	mat4 getBoneMatrix( const in float i ) {
		int size = textureSize( boneTexture, 0 ).x;
		int j = int( i ) * 4;
		int x = j % size;
		int y = j / size;
		vec4 v1 = texelFetch( boneTexture, ivec2( x, y ), 0 );
		vec4 v2 = texelFetch( boneTexture, ivec2( x + 1, y ), 0 );
		vec4 v3 = texelFetch( boneTexture, ivec2( x + 2, y ), 0 );
		vec4 v4 = texelFetch( boneTexture, ivec2( x + 3, y ), 0 );
		return mat4( v1, v2, v3, v4 );
	}
#endif`,skinning_vertex:`#ifdef USE_SKINNING
	vec4 skinVertex = bindMatrix * vec4( transformed, 1.0 );
	vec4 skinned = vec4( 0.0 );
	skinned += boneMatX * skinVertex * skinWeight.x;
	skinned += boneMatY * skinVertex * skinWeight.y;
	skinned += boneMatZ * skinVertex * skinWeight.z;
	skinned += boneMatW * skinVertex * skinWeight.w;
	transformed = ( bindMatrixInverse * skinned ).xyz;
#endif`,skinnormal_vertex:`#ifdef USE_SKINNING
	mat4 skinMatrix = mat4( 0.0 );
	skinMatrix += skinWeight.x * boneMatX;
	skinMatrix += skinWeight.y * boneMatY;
	skinMatrix += skinWeight.z * boneMatZ;
	skinMatrix += skinWeight.w * boneMatW;
	skinMatrix = bindMatrixInverse * skinMatrix * bindMatrix;
	objectNormal = vec4( skinMatrix * vec4( objectNormal, 0.0 ) ).xyz;
	#ifdef USE_TANGENT
		objectTangent = vec4( skinMatrix * vec4( objectTangent, 0.0 ) ).xyz;
	#endif
#endif`,specularmap_fragment:`float specularStrength;
#ifdef USE_SPECULARMAP
	vec4 texelSpecular = texture2D( specularMap, vSpecularMapUv );
	specularStrength = texelSpecular.r;
#else
	specularStrength = 1.0;
#endif`,specularmap_pars_fragment:`#ifdef USE_SPECULARMAP
	uniform sampler2D specularMap;
#endif`,tonemapping_fragment:`#if defined( TONE_MAPPING )
	gl_FragColor.rgb = toneMapping( gl_FragColor.rgb );
#endif`,tonemapping_pars_fragment:`#ifndef saturate
#define saturate( a ) clamp( a, 0.0, 1.0 )
#endif
uniform float toneMappingExposure;
vec3 LinearToneMapping( vec3 color ) {
	return saturate( toneMappingExposure * color );
}
vec3 ReinhardToneMapping( vec3 color ) {
	color *= toneMappingExposure;
	return saturate( color / ( vec3( 1.0 ) + color ) );
}
vec3 CineonToneMapping( vec3 color ) {
	color *= toneMappingExposure;
	color = max( vec3( 0.0 ), color - 0.004 );
	return pow( ( color * ( 6.2 * color + 0.5 ) ) / ( color * ( 6.2 * color + 1.7 ) + 0.06 ), vec3( 2.2 ) );
}
vec3 RRTAndODTFit( vec3 v ) {
	vec3 a = v * ( v + 0.0245786 ) - 0.000090537;
	vec3 b = v * ( 0.983729 * v + 0.4329510 ) + 0.238081;
	return a / b;
}
vec3 ACESFilmicToneMapping( vec3 color ) {
	const mat3 ACESInputMat = mat3(
		vec3( 0.59719, 0.07600, 0.02840 ),		vec3( 0.35458, 0.90834, 0.13383 ),
		vec3( 0.04823, 0.01566, 0.83777 )
	);
	const mat3 ACESOutputMat = mat3(
		vec3(  1.60475, -0.10208, -0.00327 ),		vec3( -0.53108,  1.10813, -0.07276 ),
		vec3( -0.07367, -0.00605,  1.07602 )
	);
	color *= toneMappingExposure / 0.6;
	color = ACESInputMat * color;
	color = RRTAndODTFit( color );
	color = ACESOutputMat * color;
	return saturate( color );
}
const mat3 LINEAR_REC2020_TO_LINEAR_SRGB = mat3(
	vec3( 1.6605, - 0.1246, - 0.0182 ),
	vec3( - 0.5876, 1.1329, - 0.1006 ),
	vec3( - 0.0728, - 0.0083, 1.1187 )
);
const mat3 LINEAR_SRGB_TO_LINEAR_REC2020 = mat3(
	vec3( 0.6274, 0.0691, 0.0164 ),
	vec3( 0.3293, 0.9195, 0.0880 ),
	vec3( 0.0433, 0.0113, 0.8956 )
);
vec3 agxDefaultContrastApprox( vec3 x ) {
	vec3 x2 = x * x;
	vec3 x4 = x2 * x2;
	return + 15.5 * x4 * x2
		- 40.14 * x4 * x
		+ 31.96 * x4
		- 6.868 * x2 * x
		+ 0.4298 * x2
		+ 0.1191 * x
		- 0.00232;
}
vec3 AgXToneMapping( vec3 color ) {
	const mat3 AgXInsetMatrix = mat3(
		vec3( 0.856627153315983, 0.137318972929847, 0.11189821299995 ),
		vec3( 0.0951212405381588, 0.761241990602591, 0.0767994186031903 ),
		vec3( 0.0482516061458583, 0.101439036467562, 0.811302368396859 )
	);
	const mat3 AgXOutsetMatrix = mat3(
		vec3( 1.1271005818144368, - 0.1413297634984383, - 0.14132976349843826 ),
		vec3( - 0.11060664309660323, 1.157823702216272, - 0.11060664309660294 ),
		vec3( - 0.016493938717834573, - 0.016493938717834257, 1.2519364065950405 )
	);
	const float AgxMinEv = - 12.47393;	const float AgxMaxEv = 4.026069;
	color *= toneMappingExposure;
	color = LINEAR_SRGB_TO_LINEAR_REC2020 * color;
	color = AgXInsetMatrix * color;
	color = max( color, 1e-10 );	color = log2( color );
	color = ( color - AgxMinEv ) / ( AgxMaxEv - AgxMinEv );
	color = clamp( color, 0.0, 1.0 );
	color = agxDefaultContrastApprox( color );
	color = AgXOutsetMatrix * color;
	color = pow( max( vec3( 0.0 ), color ), vec3( 2.2 ) );
	color = LINEAR_REC2020_TO_LINEAR_SRGB * color;
	color = clamp( color, 0.0, 1.0 );
	return color;
}
vec3 NeutralToneMapping( vec3 color ) {
	const float StartCompression = 0.8 - 0.04;
	const float Desaturation = 0.15;
	color *= toneMappingExposure;
	float x = min( color.r, min( color.g, color.b ) );
	float offset = x < 0.08 ? x - 6.25 * x * x : 0.04;
	color -= offset;
	float peak = max( color.r, max( color.g, color.b ) );
	if ( peak < StartCompression ) return color;
	float d = 1. - StartCompression;
	float newPeak = 1. - d * d / ( peak + d - StartCompression );
	color *= newPeak / peak;
	float g = 1. - 1. / ( Desaturation * ( peak - newPeak ) + 1. );
	return mix( color, vec3( newPeak ), g );
}
vec3 CustomToneMapping( vec3 color ) { return color; }`,transmission_fragment:`#ifdef USE_TRANSMISSION
	material.transmission = transmission;
	material.transmissionAlpha = 1.0;
	material.thickness = thickness;
	material.attenuationDistance = attenuationDistance;
	material.attenuationColor = attenuationColor;
	#ifdef USE_TRANSMISSIONMAP
		material.transmission *= texture2D( transmissionMap, vTransmissionMapUv ).r;
	#endif
	#ifdef USE_THICKNESSMAP
		material.thickness *= texture2D( thicknessMap, vThicknessMapUv ).g;
	#endif
	vec3 pos = vWorldPosition;
	vec3 v = normalize( cameraPosition - pos );
	vec3 n = transformNormalByInverseViewMatrix( normal, viewMatrix );
	vec4 transmitted = getIBLVolumeRefraction(
		n, v, material.roughness, material.diffuseContribution, material.specularColorBlended, material.specularF90,
		pos, modelMatrix, viewMatrix, projectionMatrix, material.dispersion, material.ior, material.thickness,
		material.attenuationColor, material.attenuationDistance );
	material.transmissionAlpha = mix( material.transmissionAlpha, transmitted.a, material.transmission );
	totalDiffuse = mix( totalDiffuse, transmitted.rgb, material.transmission );
#endif`,transmission_pars_fragment:`#ifdef USE_TRANSMISSION
	uniform float transmission;
	uniform float thickness;
	uniform float attenuationDistance;
	uniform vec3 attenuationColor;
	#ifdef USE_TRANSMISSIONMAP
		uniform sampler2D transmissionMap;
	#endif
	#ifdef USE_THICKNESSMAP
		uniform sampler2D thicknessMap;
	#endif
	uniform vec2 transmissionSamplerSize;
	uniform sampler2D transmissionSamplerMap;
	uniform mat4 modelMatrix;
	uniform mat4 projectionMatrix;
	varying vec3 vWorldPosition;
	float w0( float a ) {
		return ( 1.0 / 6.0 ) * ( a * ( a * ( - a + 3.0 ) - 3.0 ) + 1.0 );
	}
	float w1( float a ) {
		return ( 1.0 / 6.0 ) * ( a *  a * ( 3.0 * a - 6.0 ) + 4.0 );
	}
	float w2( float a ){
		return ( 1.0 / 6.0 ) * ( a * ( a * ( - 3.0 * a + 3.0 ) + 3.0 ) + 1.0 );
	}
	float w3( float a ) {
		return ( 1.0 / 6.0 ) * ( a * a * a );
	}
	float g0( float a ) {
		return w0( a ) + w1( a );
	}
	float g1( float a ) {
		return w2( a ) + w3( a );
	}
	float h0( float a ) {
		return - 1.0 + w1( a ) / ( w0( a ) + w1( a ) );
	}
	float h1( float a ) {
		return 1.0 + w3( a ) / ( w2( a ) + w3( a ) );
	}
	vec4 bicubic( sampler2D tex, vec2 uv, vec4 texelSize, float lod ) {
		uv = uv * texelSize.zw + 0.5;
		vec2 iuv = floor( uv );
		vec2 fuv = fract( uv );
		float g0x = g0( fuv.x );
		float g1x = g1( fuv.x );
		float h0x = h0( fuv.x );
		float h1x = h1( fuv.x );
		float h0y = h0( fuv.y );
		float h1y = h1( fuv.y );
		vec2 p0 = ( vec2( iuv.x + h0x, iuv.y + h0y ) - 0.5 ) * texelSize.xy;
		vec2 p1 = ( vec2( iuv.x + h1x, iuv.y + h0y ) - 0.5 ) * texelSize.xy;
		vec2 p2 = ( vec2( iuv.x + h0x, iuv.y + h1y ) - 0.5 ) * texelSize.xy;
		vec2 p3 = ( vec2( iuv.x + h1x, iuv.y + h1y ) - 0.5 ) * texelSize.xy;
		return g0( fuv.y ) * ( g0x * textureLod( tex, p0, lod ) + g1x * textureLod( tex, p1, lod ) ) +
			g1( fuv.y ) * ( g0x * textureLod( tex, p2, lod ) + g1x * textureLod( tex, p3, lod ) );
	}
	vec4 textureBicubic( sampler2D sampler, vec2 uv, float lod ) {
		vec2 fLodSize = vec2( textureSize( sampler, int( lod ) ) );
		vec2 cLodSize = vec2( textureSize( sampler, int( lod + 1.0 ) ) );
		vec2 fLodSizeInv = 1.0 / fLodSize;
		vec2 cLodSizeInv = 1.0 / cLodSize;
		vec4 fSample = bicubic( sampler, uv, vec4( fLodSizeInv, fLodSize ), floor( lod ) );
		vec4 cSample = bicubic( sampler, uv, vec4( cLodSizeInv, cLodSize ), ceil( lod ) );
		return mix( fSample, cSample, fract( lod ) );
	}
	vec3 getVolumeTransmissionRay( const in vec3 n, const in vec3 v, const in float thickness, const in float ior, const in mat4 modelMatrix ) {
		vec3 refractionVector = refract( - v, normalize( n ), 1.0 / ior );
		vec3 modelScale;
		modelScale.x = length( vec3( modelMatrix[ 0 ].xyz ) );
		modelScale.y = length( vec3( modelMatrix[ 1 ].xyz ) );
		modelScale.z = length( vec3( modelMatrix[ 2 ].xyz ) );
		return normalize( refractionVector ) * thickness * modelScale;
	}
	float applyIorToRoughness( const in float roughness, const in float ior ) {
		return roughness * clamp( ior * 2.0 - 2.0, 0.0, 1.0 );
	}
	vec4 getTransmissionSample( const in vec2 fragCoord, const in float roughness, const in float ior ) {
		float lod = log2( transmissionSamplerSize.x ) * applyIorToRoughness( roughness, ior );
		return textureBicubic( transmissionSamplerMap, fragCoord.xy, lod );
	}
	vec3 volumeAttenuation( const in float transmissionDistance, const in vec3 attenuationColor, const in float attenuationDistance ) {
		if ( isinf( attenuationDistance ) ) {
			return vec3( 1.0 );
		} else {
			vec3 attenuationCoefficient = -log( attenuationColor ) / attenuationDistance;
			vec3 transmittance = exp( - attenuationCoefficient * transmissionDistance );			return transmittance;
		}
	}
	vec4 getIBLVolumeRefraction( const in vec3 n, const in vec3 v, const in float roughness, const in vec3 diffuseColor,
		const in vec3 specularColor, const in float specularF90, const in vec3 position, const in mat4 modelMatrix,
		const in mat4 viewMatrix, const in mat4 projMatrix, const in float dispersion, const in float ior, const in float thickness,
		const in vec3 attenuationColor, const in float attenuationDistance ) {
		vec4 transmittedLight;
		vec3 transmittance;
		#ifdef USE_DISPERSION
			float halfSpread = ( ior - 1.0 ) * 0.025 * dispersion;
			vec3 iors = vec3( ior - halfSpread, ior, ior + halfSpread );
			for ( int i = 0; i < 3; i ++ ) {
				vec3 transmissionRay = getVolumeTransmissionRay( n, v, thickness, iors[ i ], modelMatrix );
				vec3 refractedRayExit = position + transmissionRay;
				vec4 ndcPos = projMatrix * viewMatrix * vec4( refractedRayExit, 1.0 );
				vec2 refractionCoords = ndcPos.xy / ndcPos.w;
				refractionCoords += 1.0;
				refractionCoords /= 2.0;
				vec4 transmissionSample = getTransmissionSample( refractionCoords, roughness, iors[ i ] );
				transmittedLight[ i ] = transmissionSample[ i ];
				transmittedLight.a += transmissionSample.a;
				transmittance[ i ] = diffuseColor[ i ] * volumeAttenuation( length( transmissionRay ), attenuationColor, attenuationDistance )[ i ];
			}
			transmittedLight.a /= 3.0;
		#else
			vec3 transmissionRay = getVolumeTransmissionRay( n, v, thickness, ior, modelMatrix );
			vec3 refractedRayExit = position + transmissionRay;
			vec4 ndcPos = projMatrix * viewMatrix * vec4( refractedRayExit, 1.0 );
			vec2 refractionCoords = ndcPos.xy / ndcPos.w;
			refractionCoords += 1.0;
			refractionCoords /= 2.0;
			transmittedLight = getTransmissionSample( refractionCoords, roughness, ior );
			transmittance = diffuseColor * volumeAttenuation( length( transmissionRay ), attenuationColor, attenuationDistance );
		#endif
		vec3 attenuatedColor = transmittance * transmittedLight.rgb;
		vec3 F = EnvironmentBRDF( n, v, specularColor, specularF90, roughness );
		float transmittanceFactor = ( transmittance.r + transmittance.g + transmittance.b ) / 3.0;
		return vec4( ( 1.0 - F ) * attenuatedColor, 1.0 - ( 1.0 - transmittedLight.a ) * transmittanceFactor );
	}
#endif`,uv_pars_fragment:`#if defined( USE_UV ) || defined( USE_ANISOTROPY )
	varying vec2 vUv;
#endif
#ifdef USE_MAP
	varying vec2 vMapUv;
#endif
#ifdef USE_ALPHAMAP
	varying vec2 vAlphaMapUv;
#endif
#ifdef USE_LIGHTMAP
	varying vec2 vLightMapUv;
#endif
#ifdef USE_AOMAP
	varying vec2 vAoMapUv;
#endif
#ifdef USE_BUMPMAP
	varying vec2 vBumpMapUv;
#endif
#ifdef USE_NORMALMAP
	varying vec2 vNormalMapUv;
#endif
#ifdef USE_EMISSIVEMAP
	varying vec2 vEmissiveMapUv;
#endif
#ifdef USE_METALNESSMAP
	varying vec2 vMetalnessMapUv;
#endif
#ifdef USE_ROUGHNESSMAP
	varying vec2 vRoughnessMapUv;
#endif
#ifdef USE_DIFFUSE_ROUGHNESSMAP
	varying vec2 vDiffuseRoughnessMapUv;
#endif
#ifdef USE_ANISOTROPYMAP
	varying vec2 vAnisotropyMapUv;
#endif
#ifdef USE_CLEARCOATMAP
	varying vec2 vClearcoatMapUv;
#endif
#ifdef USE_CLEARCOAT_NORMALMAP
	varying vec2 vClearcoatNormalMapUv;
#endif
#ifdef USE_CLEARCOAT_ROUGHNESSMAP
	varying vec2 vClearcoatRoughnessMapUv;
#endif
#ifdef USE_IRIDESCENCEMAP
	varying vec2 vIridescenceMapUv;
#endif
#ifdef USE_IRIDESCENCE_THICKNESSMAP
	varying vec2 vIridescenceThicknessMapUv;
#endif
#ifdef USE_SHEEN_COLORMAP
	varying vec2 vSheenColorMapUv;
#endif
#ifdef USE_SHEEN_ROUGHNESSMAP
	varying vec2 vSheenRoughnessMapUv;
#endif
#ifdef USE_SPECULARMAP
	varying vec2 vSpecularMapUv;
#endif
#ifdef USE_SPECULAR_COLORMAP
	varying vec2 vSpecularColorMapUv;
#endif
#ifdef USE_SPECULAR_INTENSITYMAP
	varying vec2 vSpecularIntensityMapUv;
#endif
#ifdef USE_TRANSMISSIONMAP
	uniform mat3 transmissionMapTransform;
	varying vec2 vTransmissionMapUv;
#endif
#ifdef USE_THICKNESSMAP
	uniform mat3 thicknessMapTransform;
	varying vec2 vThicknessMapUv;
#endif`,uv_pars_vertex:`#if defined( USE_UV ) || defined( USE_ANISOTROPY )
	varying vec2 vUv;
#endif
#ifdef USE_MAP
	uniform mat3 mapTransform;
	varying vec2 vMapUv;
#endif
#ifdef USE_ALPHAMAP
	uniform mat3 alphaMapTransform;
	varying vec2 vAlphaMapUv;
#endif
#ifdef USE_LIGHTMAP
	uniform mat3 lightMapTransform;
	varying vec2 vLightMapUv;
#endif
#ifdef USE_AOMAP
	uniform mat3 aoMapTransform;
	varying vec2 vAoMapUv;
#endif
#ifdef USE_BUMPMAP
	uniform mat3 bumpMapTransform;
	varying vec2 vBumpMapUv;
#endif
#ifdef USE_NORMALMAP
	uniform mat3 normalMapTransform;
	varying vec2 vNormalMapUv;
#endif
#ifdef USE_DISPLACEMENTMAP
	uniform mat3 displacementMapTransform;
	varying vec2 vDisplacementMapUv;
#endif
#ifdef USE_EMISSIVEMAP
	uniform mat3 emissiveMapTransform;
	varying vec2 vEmissiveMapUv;
#endif
#ifdef USE_METALNESSMAP
	uniform mat3 metalnessMapTransform;
	varying vec2 vMetalnessMapUv;
#endif
#ifdef USE_ROUGHNESSMAP
	uniform mat3 roughnessMapTransform;
	varying vec2 vRoughnessMapUv;
#endif
#ifdef USE_DIFFUSE_ROUGHNESSMAP
	uniform mat3 diffuseRoughnessMapTransform;
	varying vec2 vDiffuseRoughnessMapUv;
#endif
#ifdef USE_ANISOTROPYMAP
	uniform mat3 anisotropyMapTransform;
	varying vec2 vAnisotropyMapUv;
#endif
#ifdef USE_CLEARCOATMAP
	uniform mat3 clearcoatMapTransform;
	varying vec2 vClearcoatMapUv;
#endif
#ifdef USE_CLEARCOAT_NORMALMAP
	uniform mat3 clearcoatNormalMapTransform;
	varying vec2 vClearcoatNormalMapUv;
#endif
#ifdef USE_CLEARCOAT_ROUGHNESSMAP
	uniform mat3 clearcoatRoughnessMapTransform;
	varying vec2 vClearcoatRoughnessMapUv;
#endif
#ifdef USE_SHEEN_COLORMAP
	uniform mat3 sheenColorMapTransform;
	varying vec2 vSheenColorMapUv;
#endif
#ifdef USE_SHEEN_ROUGHNESSMAP
	uniform mat3 sheenRoughnessMapTransform;
	varying vec2 vSheenRoughnessMapUv;
#endif
#ifdef USE_IRIDESCENCEMAP
	uniform mat3 iridescenceMapTransform;
	varying vec2 vIridescenceMapUv;
#endif
#ifdef USE_IRIDESCENCE_THICKNESSMAP
	uniform mat3 iridescenceThicknessMapTransform;
	varying vec2 vIridescenceThicknessMapUv;
#endif
#ifdef USE_SPECULARMAP
	uniform mat3 specularMapTransform;
	varying vec2 vSpecularMapUv;
#endif
#ifdef USE_SPECULAR_COLORMAP
	uniform mat3 specularColorMapTransform;
	varying vec2 vSpecularColorMapUv;
#endif
#ifdef USE_SPECULAR_INTENSITYMAP
	uniform mat3 specularIntensityMapTransform;
	varying vec2 vSpecularIntensityMapUv;
#endif
#ifdef USE_TRANSMISSIONMAP
	uniform mat3 transmissionMapTransform;
	varying vec2 vTransmissionMapUv;
#endif
#ifdef USE_THICKNESSMAP
	uniform mat3 thicknessMapTransform;
	varying vec2 vThicknessMapUv;
#endif`,uv_vertex:`#if defined( USE_UV ) || defined( USE_ANISOTROPY )
	vUv = vec3( uv, 1 ).xy;
#endif
#ifdef USE_MAP
	vMapUv = ( mapTransform * vec3( MAP_UV, 1 ) ).xy;
#endif
#ifdef USE_ALPHAMAP
	vAlphaMapUv = ( alphaMapTransform * vec3( ALPHAMAP_UV, 1 ) ).xy;
#endif
#ifdef USE_LIGHTMAP
	vLightMapUv = ( lightMapTransform * vec3( LIGHTMAP_UV, 1 ) ).xy;
#endif
#ifdef USE_AOMAP
	vAoMapUv = ( aoMapTransform * vec3( AOMAP_UV, 1 ) ).xy;
#endif
#ifdef USE_BUMPMAP
	vBumpMapUv = ( bumpMapTransform * vec3( BUMPMAP_UV, 1 ) ).xy;
#endif
#ifdef USE_NORMALMAP
	vNormalMapUv = ( normalMapTransform * vec3( NORMALMAP_UV, 1 ) ).xy;
#endif
#ifdef USE_DISPLACEMENTMAP
	vDisplacementMapUv = ( displacementMapTransform * vec3( DISPLACEMENTMAP_UV, 1 ) ).xy;
#endif
#ifdef USE_EMISSIVEMAP
	vEmissiveMapUv = ( emissiveMapTransform * vec3( EMISSIVEMAP_UV, 1 ) ).xy;
#endif
#ifdef USE_METALNESSMAP
	vMetalnessMapUv = ( metalnessMapTransform * vec3( METALNESSMAP_UV, 1 ) ).xy;
#endif
#ifdef USE_ROUGHNESSMAP
	vRoughnessMapUv = ( roughnessMapTransform * vec3( ROUGHNESSMAP_UV, 1 ) ).xy;
#endif
#ifdef USE_DIFFUSE_ROUGHNESSMAP
	vDiffuseRoughnessMapUv = ( diffuseRoughnessMapTransform * vec3( DIFFUSE_ROUGHNESSMAP_UV, 1 ) ).xy;
#endif
#ifdef USE_ANISOTROPYMAP
	vAnisotropyMapUv = ( anisotropyMapTransform * vec3( ANISOTROPYMAP_UV, 1 ) ).xy;
#endif
#ifdef USE_CLEARCOATMAP
	vClearcoatMapUv = ( clearcoatMapTransform * vec3( CLEARCOATMAP_UV, 1 ) ).xy;
#endif
#ifdef USE_CLEARCOAT_NORMALMAP
	vClearcoatNormalMapUv = ( clearcoatNormalMapTransform * vec3( CLEARCOAT_NORMALMAP_UV, 1 ) ).xy;
#endif
#ifdef USE_CLEARCOAT_ROUGHNESSMAP
	vClearcoatRoughnessMapUv = ( clearcoatRoughnessMapTransform * vec3( CLEARCOAT_ROUGHNESSMAP_UV, 1 ) ).xy;
#endif
#ifdef USE_IRIDESCENCEMAP
	vIridescenceMapUv = ( iridescenceMapTransform * vec3( IRIDESCENCEMAP_UV, 1 ) ).xy;
#endif
#ifdef USE_IRIDESCENCE_THICKNESSMAP
	vIridescenceThicknessMapUv = ( iridescenceThicknessMapTransform * vec3( IRIDESCENCE_THICKNESSMAP_UV, 1 ) ).xy;
#endif
#ifdef USE_SHEEN_COLORMAP
	vSheenColorMapUv = ( sheenColorMapTransform * vec3( SHEEN_COLORMAP_UV, 1 ) ).xy;
#endif
#ifdef USE_SHEEN_ROUGHNESSMAP
	vSheenRoughnessMapUv = ( sheenRoughnessMapTransform * vec3( SHEEN_ROUGHNESSMAP_UV, 1 ) ).xy;
#endif
#ifdef USE_SPECULARMAP
	vSpecularMapUv = ( specularMapTransform * vec3( SPECULARMAP_UV, 1 ) ).xy;
#endif
#ifdef USE_SPECULAR_COLORMAP
	vSpecularColorMapUv = ( specularColorMapTransform * vec3( SPECULAR_COLORMAP_UV, 1 ) ).xy;
#endif
#ifdef USE_SPECULAR_INTENSITYMAP
	vSpecularIntensityMapUv = ( specularIntensityMapTransform * vec3( SPECULAR_INTENSITYMAP_UV, 1 ) ).xy;
#endif
#ifdef USE_TRANSMISSIONMAP
	vTransmissionMapUv = ( transmissionMapTransform * vec3( TRANSMISSIONMAP_UV, 1 ) ).xy;
#endif
#ifdef USE_THICKNESSMAP
	vThicknessMapUv = ( thicknessMapTransform * vec3( THICKNESSMAP_UV, 1 ) ).xy;
#endif`,worldpos_vertex:`#if defined( USE_ENVMAP ) || defined( DISTANCE ) || defined ( USE_SHADOWMAP ) || defined ( USE_TRANSMISSION ) || NUM_SPOT_LIGHT_COORDS > 0
	vec4 worldPosition = vec4( transformed, 1.0 );
	#ifdef USE_BATCHING
		worldPosition = batchingMatrix * worldPosition;
	#endif
	#ifdef USE_INSTANCING
		worldPosition = instanceMatrix * worldPosition;
	#endif
	worldPosition = modelMatrix * worldPosition;
#endif`,background_vert:`varying vec2 vUv;
uniform mat3 uvTransform;
void main() {
	vUv = ( uvTransform * vec3( uv, 1 ) ).xy;
	gl_Position = vec4( position.xy, 1.0, 1.0 );
}`,background_frag:`uniform sampler2D t2D;
uniform float backgroundIntensity;
varying vec2 vUv;
void main() {
	vec4 texColor = texture2D( t2D, vUv );
	#ifdef DECODE_VIDEO_TEXTURE
		texColor = vec4( mix( pow( texColor.rgb * 0.9478672986 + vec3( 0.0521327014 ), vec3( 2.4 ) ), texColor.rgb * 0.0773993808, vec3( lessThanEqual( texColor.rgb, vec3( 0.04045 ) ) ) ), texColor.w );
	#endif
	texColor.rgb *= backgroundIntensity;
	gl_FragColor = texColor;
	#include <tonemapping_fragment>
	#include <colorspace_fragment>
}`,backgroundCube_vert:`varying vec3 vWorldDirection;
#include <common>
void main() {
	vWorldDirection = transformDirection( position, modelMatrix );
	#include <begin_vertex>
	#include <project_vertex>
	gl_Position.z = gl_Position.w;
}`,backgroundCube_frag:`uniform float backgroundBlurriness;
uniform float backgroundIntensity;
uniform mat3 backgroundRotation;
varying vec3 vWorldDirection;
#include <envmap_common_pars_fragment>
void main() {
	#ifdef ENVMAP_TYPE_PMREM
		vec4 texColor = textureLod( envMap, backgroundRotation * vWorldDirection, roughnessToMip( backgroundBlurriness ) );
	#else
		vec4 texColor = textureCube( envMap, backgroundRotation * vWorldDirection );
	#endif
	texColor.rgb *= backgroundIntensity;
	gl_FragColor = texColor;
	#include <tonemapping_fragment>
	#include <colorspace_fragment>
}`,cube_vert:`varying vec3 vWorldDirection;
#include <common>
void main() {
	vWorldDirection = transformDirection( position, modelMatrix );
	#include <begin_vertex>
	#include <project_vertex>
	gl_Position.z = gl_Position.w;
}`,cube_frag:`uniform samplerCube tCube;
uniform float tFlip;
uniform float opacity;
varying vec3 vWorldDirection;
void main() {
	vec4 texColor = textureCube( tCube, vec3( tFlip * vWorldDirection.x, vWorldDirection.yz ) );
	gl_FragColor = texColor;
	gl_FragColor.a *= opacity;
	#include <tonemapping_fragment>
	#include <colorspace_fragment>
}`,depth_vert:`#include <common>
#include <batching_pars_vertex>
#include <uv_pars_vertex>
#include <displacementmap_pars_vertex>
#include <morphtarget_pars_vertex>
#include <skinning_pars_vertex>
#include <logdepthbuf_pars_vertex>
#include <clipping_planes_pars_vertex>
varying vec2 vHighPrecisionZW;
void main() {
	#include <uv_vertex>
	#include <batching_vertex>
	#include <skinbase_vertex>
	#include <morphinstance_vertex>
	#ifdef USE_DISPLACEMENTMAP
		#include <beginnormal_vertex>
		#include <morphnormal_vertex>
		#include <skinnormal_vertex>
	#endif
	#include <begin_vertex>
	#include <morphtarget_vertex>
	#include <skinning_vertex>
	#include <displacementmap_vertex>
	#include <project_vertex>
	#include <logdepthbuf_vertex>
	#include <clipping_planes_vertex>
	vHighPrecisionZW = gl_Position.zw;
}`,depth_frag:`#if DEPTH_PACKING == 3200
	uniform float opacity;
#endif
#include <common>
#include <packing>
#include <uv_pars_fragment>
#include <map_pars_fragment>
#include <alphamap_pars_fragment>
#include <alphatest_pars_fragment>
#include <alphahash_pars_fragment>
#include <logdepthbuf_pars_fragment>
#include <clipping_planes_pars_fragment>
varying vec2 vHighPrecisionZW;
void main() {
	vec4 diffuseColor = vec4( 1.0 );
	#include <clipping_planes_fragment>
	#if DEPTH_PACKING == 3200
		diffuseColor.a = opacity;
	#endif
	#include <map_fragment>
	#include <alphamap_fragment>
	#include <alphatest_fragment>
	#include <alphahash_fragment>
	#include <logdepthbuf_fragment>
	#ifdef USE_REVERSED_DEPTH_BUFFER
		float fragCoordZ = vHighPrecisionZW[ 0 ] / vHighPrecisionZW[ 1 ];
	#else
		float fragCoordZ = 0.5 * vHighPrecisionZW[ 0 ] / vHighPrecisionZW[ 1 ] + 0.5;
	#endif
	#if DEPTH_PACKING == 3200
		gl_FragColor = vec4( vec3( 1.0 - fragCoordZ ), opacity );
	#elif DEPTH_PACKING == 3201
		gl_FragColor = packDepthToRGBA( fragCoordZ );
	#elif DEPTH_PACKING == 3202
		gl_FragColor = vec4( packDepthToRGB( fragCoordZ ), 1.0 );
	#elif DEPTH_PACKING == 3203
		gl_FragColor = vec4( packDepthToRG( fragCoordZ ), 0.0, 1.0 );
	#endif
}`,distance_vert:`#define DISTANCE
varying vec3 vWorldPosition;
#include <common>
#include <batching_pars_vertex>
#include <uv_pars_vertex>
#include <displacementmap_pars_vertex>
#include <morphtarget_pars_vertex>
#include <skinning_pars_vertex>
#include <clipping_planes_pars_vertex>
void main() {
	#include <uv_vertex>
	#include <batching_vertex>
	#include <skinbase_vertex>
	#include <morphinstance_vertex>
	#ifdef USE_DISPLACEMENTMAP
		#include <beginnormal_vertex>
		#include <morphnormal_vertex>
		#include <skinnormal_vertex>
	#endif
	#include <begin_vertex>
	#include <morphtarget_vertex>
	#include <skinning_vertex>
	#include <displacementmap_vertex>
	#include <project_vertex>
	#include <worldpos_vertex>
	#include <clipping_planes_vertex>
	vWorldPosition = worldPosition.xyz;
}`,distance_frag:`#define DISTANCE
uniform vec3 referencePosition;
uniform float nearDistance;
uniform float farDistance;
varying vec3 vWorldPosition;
#include <common>
#include <uv_pars_fragment>
#include <map_pars_fragment>
#include <alphamap_pars_fragment>
#include <alphatest_pars_fragment>
#include <alphahash_pars_fragment>
#include <clipping_planes_pars_fragment>
void main() {
	vec4 diffuseColor = vec4( 1.0 );
	#include <clipping_planes_fragment>
	#include <map_fragment>
	#include <alphamap_fragment>
	#include <alphatest_fragment>
	#include <alphahash_fragment>
	float dist = length( vWorldPosition - referencePosition );
	dist = ( dist - nearDistance ) / ( farDistance - nearDistance );
	dist = saturate( dist );
	gl_FragColor = vec4( dist, 0.0, 0.0, 1.0 );
}`,equirect_vert:`varying vec3 vWorldDirection;
#include <common>
void main() {
	vWorldDirection = transformDirection( position, modelMatrix );
	#include <begin_vertex>
	#include <project_vertex>
}`,equirect_frag:`uniform sampler2D tEquirect;
varying vec3 vWorldDirection;
#include <common>
void main() {
	vec3 direction = normalize( vWorldDirection );
	vec2 sampleUV = equirectUv( direction );
	gl_FragColor = texture2D( tEquirect, sampleUV );
	#include <tonemapping_fragment>
	#include <colorspace_fragment>
}`,linedashed_vert:`uniform float scale;
attribute float lineDistance;
varying float vLineDistance;
#include <common>
#include <uv_pars_vertex>
#include <color_pars_vertex>
#include <fog_pars_vertex>
#include <morphtarget_pars_vertex>
#include <logdepthbuf_pars_vertex>
#include <clipping_planes_pars_vertex>
void main() {
	vLineDistance = scale * lineDistance;
	#include <uv_vertex>
	#include <color_vertex>
	#include <morphinstance_vertex>
	#include <morphcolor_vertex>
	#include <begin_vertex>
	#include <morphtarget_vertex>
	#include <project_vertex>
	#include <logdepthbuf_vertex>
	#include <clipping_planes_vertex>
	#include <fog_vertex>
}`,linedashed_frag:`uniform vec3 diffuse;
uniform float opacity;
uniform float dashSize;
uniform float totalSize;
varying float vLineDistance;
#include <common>
#include <color_pars_fragment>
#include <uv_pars_fragment>
#include <map_pars_fragment>
#include <fog_pars_fragment>
#include <logdepthbuf_pars_fragment>
#include <clipping_planes_pars_fragment>
void main() {
	vec4 diffuseColor = vec4( diffuse, opacity );
	#include <clipping_planes_fragment>
	if ( mod( vLineDistance, totalSize ) > dashSize ) {
		discard;
	}
	vec3 outgoingLight = vec3( 0.0 );
	#include <logdepthbuf_fragment>
	#include <map_fragment>
	#include <color_fragment>
	outgoingLight = diffuseColor.rgb;
	#include <opaque_fragment>
	#include <tonemapping_fragment>
	#include <colorspace_fragment>
	#include <fog_fragment>
	#include <premultiplied_alpha_fragment>
}`,meshbasic_vert:`#include <common>
#include <batching_pars_vertex>
#include <uv_pars_vertex>
#include <envmap_pars_vertex>
#include <color_pars_vertex>
#include <fog_pars_vertex>
#include <morphtarget_pars_vertex>
#include <skinning_pars_vertex>
#include <logdepthbuf_pars_vertex>
#include <clipping_planes_pars_vertex>
void main() {
	#include <uv_vertex>
	#include <color_vertex>
	#include <morphinstance_vertex>
	#include <morphcolor_vertex>
	#include <batching_vertex>
	#if defined ( USE_ENVMAP ) || defined ( USE_SKINNING )
		#include <beginnormal_vertex>
		#include <morphnormal_vertex>
		#include <skinbase_vertex>
		#include <skinnormal_vertex>
		#include <defaultnormal_vertex>
	#endif
	#include <begin_vertex>
	#include <morphtarget_vertex>
	#include <skinning_vertex>
	#include <project_vertex>
	#include <logdepthbuf_vertex>
	#include <clipping_planes_vertex>
	#include <worldpos_vertex>
	#include <envmap_vertex>
	#include <fog_vertex>
}`,meshbasic_frag:`uniform vec3 diffuse;
uniform float opacity;
#ifndef FLAT_SHADED
	varying vec3 vNormal;
#endif
#include <common>
#include <dithering_pars_fragment>
#include <color_pars_fragment>
#include <uv_pars_fragment>
#include <map_pars_fragment>
#include <alphamap_pars_fragment>
#include <alphatest_pars_fragment>
#include <alphahash_pars_fragment>
#include <aomap_pars_fragment>
#include <lightmap_pars_fragment>
#include <envmap_common_pars_fragment>
#include <envmap_pars_fragment>
#include <fog_pars_fragment>
#include <specularmap_pars_fragment>
#include <logdepthbuf_pars_fragment>
#include <clipping_planes_pars_fragment>
void main() {
	vec4 diffuseColor = vec4( diffuse, opacity );
	#include <clipping_planes_fragment>
	#include <logdepthbuf_fragment>
	#include <map_fragment>
	#include <color_fragment>
	#include <alphamap_fragment>
	#include <alphatest_fragment>
	#include <alphahash_fragment>
	#include <specularmap_fragment>
	ReflectedLight reflectedLight = ReflectedLight( vec3( 0.0 ), vec3( 0.0 ), vec3( 0.0 ), vec3( 0.0 ) );
	#ifdef USE_LIGHTMAP
		vec4 lightMapTexel = texture2D( lightMap, vLightMapUv );
		reflectedLight.indirectDiffuse += lightMapTexel.rgb * lightMapIntensity * RECIPROCAL_PI;
	#else
		reflectedLight.indirectDiffuse += vec3( 1.0 );
	#endif
	#include <aomap_fragment>
	reflectedLight.indirectDiffuse *= diffuseColor.rgb;
	vec3 outgoingLight = reflectedLight.indirectDiffuse;
	#include <envmap_fragment>
	#include <opaque_fragment>
	#include <tonemapping_fragment>
	#include <colorspace_fragment>
	#include <fog_fragment>
	#include <premultiplied_alpha_fragment>
	#include <dithering_fragment>
}`,meshlambert_vert:`#define LAMBERT
varying vec3 vViewPosition;
#include <common>
#include <batching_pars_vertex>
#include <uv_pars_vertex>
#include <displacementmap_pars_vertex>
#include <envmap_pars_vertex>
#include <color_pars_vertex>
#include <fog_pars_vertex>
#include <normal_pars_vertex>
#include <morphtarget_pars_vertex>
#include <skinning_pars_vertex>
#include <shadowmap_pars_vertex>
#include <logdepthbuf_pars_vertex>
#include <clipping_planes_pars_vertex>
void main() {
	#include <uv_vertex>
	#include <color_vertex>
	#include <morphinstance_vertex>
	#include <morphcolor_vertex>
	#include <batching_vertex>
	#include <beginnormal_vertex>
	#include <morphnormal_vertex>
	#include <skinbase_vertex>
	#include <skinnormal_vertex>
	#include <defaultnormal_vertex>
	#include <normal_vertex>
	#include <begin_vertex>
	#include <morphtarget_vertex>
	#include <skinning_vertex>
	#include <displacementmap_vertex>
	#include <project_vertex>
	#include <logdepthbuf_vertex>
	#include <clipping_planes_vertex>
	vViewPosition = - mvPosition.xyz;
	#include <worldpos_vertex>
	#include <envmap_vertex>
	#include <shadowmap_vertex>
	#include <fog_vertex>
}`,meshlambert_frag:`#define LAMBERT
uniform vec3 diffuse;
uniform vec3 emissive;
uniform float opacity;
#include <common>
#include <dithering_pars_fragment>
#include <color_pars_fragment>
#include <uv_pars_fragment>
#include <map_pars_fragment>
#include <alphamap_pars_fragment>
#include <alphatest_pars_fragment>
#include <alphahash_pars_fragment>
#include <aomap_pars_fragment>
#include <lightmap_pars_fragment>
#include <emissivemap_pars_fragment>
#include <envmap_common_pars_fragment>
#include <envmap_pars_fragment>
#include <envmap_physical_pars_fragment>
#include <fog_pars_fragment>
#include <bsdfs>
#include <lights_pars_begin>
#include <normal_pars_fragment>
#include <lights_lambert_pars_fragment>
#include <shadowmap_pars_fragment>
#include <bumpmap_pars_fragment>
#include <normalmap_pars_fragment>
#include <specularmap_pars_fragment>
#include <logdepthbuf_pars_fragment>
#include <clipping_planes_pars_fragment>
void main() {
	vec4 diffuseColor = vec4( diffuse, opacity );
	#include <clipping_planes_fragment>
	ReflectedLight reflectedLight = ReflectedLight( vec3( 0.0 ), vec3( 0.0 ), vec3( 0.0 ), vec3( 0.0 ) );
	vec3 totalEmissiveRadiance = emissive;
	#include <logdepthbuf_fragment>
	#include <map_fragment>
	#include <color_fragment>
	#include <alphamap_fragment>
	#include <alphatest_fragment>
	#include <alphahash_fragment>
	#include <specularmap_fragment>
	#include <normal_fragment_begin>
	#include <normal_fragment_maps>
	#include <emissivemap_fragment>
	#include <lights_lambert_fragment>
	#include <lights_fragment_begin>
	#include <lights_fragment_maps>
	#include <lights_fragment_end>
	#include <aomap_fragment>
	vec3 outgoingLight = reflectedLight.directDiffuse + reflectedLight.indirectDiffuse + totalEmissiveRadiance;
	#include <envmap_fragment>
	#include <opaque_fragment>
	#include <tonemapping_fragment>
	#include <colorspace_fragment>
	#include <fog_fragment>
	#include <premultiplied_alpha_fragment>
	#include <dithering_fragment>
}`,meshmatcap_vert:`#define MATCAP
varying vec3 vViewPosition;
#include <common>
#include <batching_pars_vertex>
#include <uv_pars_vertex>
#include <color_pars_vertex>
#include <displacementmap_pars_vertex>
#include <fog_pars_vertex>
#include <normal_pars_vertex>
#include <morphtarget_pars_vertex>
#include <skinning_pars_vertex>
#include <logdepthbuf_pars_vertex>
#include <clipping_planes_pars_vertex>
void main() {
	#include <uv_vertex>
	#include <color_vertex>
	#include <morphinstance_vertex>
	#include <morphcolor_vertex>
	#include <batching_vertex>
	#include <beginnormal_vertex>
	#include <morphnormal_vertex>
	#include <skinbase_vertex>
	#include <skinnormal_vertex>
	#include <defaultnormal_vertex>
	#include <normal_vertex>
	#include <begin_vertex>
	#include <morphtarget_vertex>
	#include <skinning_vertex>
	#include <displacementmap_vertex>
	#include <project_vertex>
	#include <logdepthbuf_vertex>
	#include <clipping_planes_vertex>
	#include <fog_vertex>
	vViewPosition = - mvPosition.xyz;
}`,meshmatcap_frag:`#define MATCAP
uniform vec3 diffuse;
uniform float opacity;
uniform sampler2D matcap;
varying vec3 vViewPosition;
#include <common>
#include <dithering_pars_fragment>
#include <color_pars_fragment>
#include <uv_pars_fragment>
#include <map_pars_fragment>
#include <alphamap_pars_fragment>
#include <alphatest_pars_fragment>
#include <alphahash_pars_fragment>
#include <fog_pars_fragment>
#include <normal_pars_fragment>
#include <bumpmap_pars_fragment>
#include <normalmap_pars_fragment>
#include <logdepthbuf_pars_fragment>
#include <clipping_planes_pars_fragment>
void main() {
	vec4 diffuseColor = vec4( diffuse, opacity );
	#include <clipping_planes_fragment>
	#include <logdepthbuf_fragment>
	#include <map_fragment>
	#include <color_fragment>
	#include <alphamap_fragment>
	#include <alphatest_fragment>
	#include <alphahash_fragment>
	#include <normal_fragment_begin>
	#include <normal_fragment_maps>
	vec3 viewDir = normalize( vViewPosition );
	vec3 x = normalize( vec3( viewDir.z, 0.0, - viewDir.x ) );
	vec3 y = cross( viewDir, x );
	vec2 uv = vec2( dot( x, normal ), dot( y, normal ) ) * 0.495 + 0.5;
	#ifdef USE_MATCAP
		vec4 matcapColor = texture2D( matcap, uv );
	#else
		vec4 matcapColor = vec4( vec3( mix( 0.2, 0.8, uv.y ) ), 1.0 );
	#endif
	vec3 outgoingLight = diffuseColor.rgb * matcapColor.rgb;
	#include <opaque_fragment>
	#include <tonemapping_fragment>
	#include <colorspace_fragment>
	#include <fog_fragment>
	#include <premultiplied_alpha_fragment>
	#include <dithering_fragment>
}`,meshnormal_vert:`#define NORMAL
#if defined( FLAT_SHADED ) || defined( USE_BUMPMAP ) || defined( USE_NORMALMAP_TANGENTSPACE )
	varying vec3 vViewPosition;
#endif
#include <common>
#include <batching_pars_vertex>
#include <uv_pars_vertex>
#include <displacementmap_pars_vertex>
#include <normal_pars_vertex>
#include <morphtarget_pars_vertex>
#include <skinning_pars_vertex>
#include <logdepthbuf_pars_vertex>
#include <clipping_planes_pars_vertex>
void main() {
	#include <uv_vertex>
	#include <batching_vertex>
	#include <beginnormal_vertex>
	#include <morphinstance_vertex>
	#include <morphnormal_vertex>
	#include <skinbase_vertex>
	#include <skinnormal_vertex>
	#include <defaultnormal_vertex>
	#include <normal_vertex>
	#include <begin_vertex>
	#include <morphtarget_vertex>
	#include <skinning_vertex>
	#include <displacementmap_vertex>
	#include <project_vertex>
	#include <logdepthbuf_vertex>
	#include <clipping_planes_vertex>
#if defined( FLAT_SHADED ) || defined( USE_BUMPMAP ) || defined( USE_NORMALMAP_TANGENTSPACE )
	vViewPosition = - mvPosition.xyz;
#endif
}`,meshnormal_frag:`#define NORMAL
uniform float opacity;
#if defined( FLAT_SHADED ) || defined( USE_BUMPMAP ) || defined( USE_NORMALMAP_TANGENTSPACE )
	varying vec3 vViewPosition;
#endif
#include <uv_pars_fragment>
#include <normal_pars_fragment>
#include <bumpmap_pars_fragment>
#include <normalmap_pars_fragment>
#include <logdepthbuf_pars_fragment>
#include <clipping_planes_pars_fragment>
void main() {
	vec4 diffuseColor = vec4( 0.0, 0.0, 0.0, opacity );
	#include <clipping_planes_fragment>
	#include <logdepthbuf_fragment>
	#include <normal_fragment_begin>
	#include <normal_fragment_maps>
	gl_FragColor = vec4( normalize( normal ) * 0.5 + 0.5, diffuseColor.a );
	#ifdef OPAQUE
		gl_FragColor.a = 1.0;
	#endif
}`,meshphong_vert:`#define PHONG
varying vec3 vViewPosition;
#include <common>
#include <batching_pars_vertex>
#include <uv_pars_vertex>
#include <displacementmap_pars_vertex>
#include <envmap_pars_vertex>
#include <color_pars_vertex>
#include <fog_pars_vertex>
#include <normal_pars_vertex>
#include <morphtarget_pars_vertex>
#include <skinning_pars_vertex>
#include <shadowmap_pars_vertex>
#include <logdepthbuf_pars_vertex>
#include <clipping_planes_pars_vertex>
void main() {
	#include <uv_vertex>
	#include <color_vertex>
	#include <morphcolor_vertex>
	#include <batching_vertex>
	#include <beginnormal_vertex>
	#include <morphinstance_vertex>
	#include <morphnormal_vertex>
	#include <skinbase_vertex>
	#include <skinnormal_vertex>
	#include <defaultnormal_vertex>
	#include <normal_vertex>
	#include <begin_vertex>
	#include <morphtarget_vertex>
	#include <skinning_vertex>
	#include <displacementmap_vertex>
	#include <project_vertex>
	#include <logdepthbuf_vertex>
	#include <clipping_planes_vertex>
	vViewPosition = - mvPosition.xyz;
	#include <worldpos_vertex>
	#include <envmap_vertex>
	#include <shadowmap_vertex>
	#include <fog_vertex>
}`,meshphong_frag:`#define PHONG
uniform vec3 diffuse;
uniform vec3 emissive;
uniform vec3 specular;
uniform float shininess;
uniform float opacity;
#include <common>
#include <dithering_pars_fragment>
#include <color_pars_fragment>
#include <uv_pars_fragment>
#include <map_pars_fragment>
#include <alphamap_pars_fragment>
#include <alphatest_pars_fragment>
#include <alphahash_pars_fragment>
#include <aomap_pars_fragment>
#include <lightmap_pars_fragment>
#include <emissivemap_pars_fragment>
#include <envmap_common_pars_fragment>
#include <envmap_pars_fragment>
#include <envmap_physical_pars_fragment>
#include <fog_pars_fragment>
#include <bsdfs>
#include <lights_pars_begin>
#include <normal_pars_fragment>
#include <lights_phong_pars_fragment>
#include <shadowmap_pars_fragment>
#include <bumpmap_pars_fragment>
#include <normalmap_pars_fragment>
#include <specularmap_pars_fragment>
#include <logdepthbuf_pars_fragment>
#include <clipping_planes_pars_fragment>
void main() {
	vec4 diffuseColor = vec4( diffuse, opacity );
	#include <clipping_planes_fragment>
	ReflectedLight reflectedLight = ReflectedLight( vec3( 0.0 ), vec3( 0.0 ), vec3( 0.0 ), vec3( 0.0 ) );
	vec3 totalEmissiveRadiance = emissive;
	#include <logdepthbuf_fragment>
	#include <map_fragment>
	#include <color_fragment>
	#include <alphamap_fragment>
	#include <alphatest_fragment>
	#include <alphahash_fragment>
	#include <specularmap_fragment>
	#include <normal_fragment_begin>
	#include <normal_fragment_maps>
	#include <emissivemap_fragment>
	#include <lights_phong_fragment>
	#include <lights_fragment_begin>
	#include <lights_fragment_maps>
	#include <lights_fragment_end>
	#include <aomap_fragment>
	vec3 outgoingLight = reflectedLight.directDiffuse + reflectedLight.indirectDiffuse + reflectedLight.directSpecular + reflectedLight.indirectSpecular + totalEmissiveRadiance;
	#include <envmap_fragment>
	#include <opaque_fragment>
	#include <tonemapping_fragment>
	#include <colorspace_fragment>
	#include <fog_fragment>
	#include <premultiplied_alpha_fragment>
	#include <dithering_fragment>
}`,meshphysical_vert:`#define STANDARD
varying vec3 vViewPosition;
#ifdef USE_TRANSMISSION
	varying vec3 vWorldPosition;
#endif
#include <common>
#include <batching_pars_vertex>
#include <uv_pars_vertex>
#include <displacementmap_pars_vertex>
#include <color_pars_vertex>
#include <fog_pars_vertex>
#include <normal_pars_vertex>
#include <morphtarget_pars_vertex>
#include <skinning_pars_vertex>
#include <shadowmap_pars_vertex>
#include <logdepthbuf_pars_vertex>
#include <clipping_planes_pars_vertex>
void main() {
	#include <uv_vertex>
	#include <color_vertex>
	#include <morphinstance_vertex>
	#include <morphcolor_vertex>
	#include <batching_vertex>
	#include <beginnormal_vertex>
	#include <morphnormal_vertex>
	#include <skinbase_vertex>
	#include <skinnormal_vertex>
	#include <defaultnormal_vertex>
	#include <normal_vertex>
	#include <begin_vertex>
	#include <morphtarget_vertex>
	#include <skinning_vertex>
	#include <displacementmap_vertex>
	#include <project_vertex>
	#include <logdepthbuf_vertex>
	#include <clipping_planes_vertex>
	vViewPosition = - mvPosition.xyz;
	#include <worldpos_vertex>
	#include <shadowmap_vertex>
	#include <fog_vertex>
#ifdef USE_TRANSMISSION
	vWorldPosition = worldPosition.xyz;
#endif
}`,meshphysical_frag:`#define STANDARD
#ifdef PHYSICAL
	#define IOR
	#define USE_SPECULAR
#endif
uniform vec3 diffuse;
uniform vec3 emissive;
uniform float roughness;
uniform float metalness;
uniform float opacity;
#ifdef IOR
	uniform float ior;
#endif
#ifdef USE_SPECULAR
	uniform float specularIntensity;
	uniform vec3 specularColor;
	#ifdef USE_SPECULAR_COLORMAP
		uniform sampler2D specularColorMap;
	#endif
	#ifdef USE_SPECULAR_INTENSITYMAP
		uniform sampler2D specularIntensityMap;
	#endif
#endif
#ifdef USE_CLEARCOAT
	uniform float clearcoat;
	uniform float clearcoatRoughness;
#endif
#ifdef USE_DIFFUSE_ROUGHNESS
	uniform float diffuseRoughness;
	#ifdef USE_DIFFUSE_ROUGHNESSMAP
		uniform sampler2D diffuseRoughnessMap;
	#endif
#endif
#ifdef USE_DISPERSION
	uniform float dispersion;
#endif
#ifdef USE_RETROREFLECTION
	uniform float retroreflectivity;
#endif
#ifdef USE_IRIDESCENCE
	uniform float iridescence;
	uniform float iridescenceIOR;
	uniform float iridescenceThicknessMinimum;
	uniform float iridescenceThicknessMaximum;
#endif
#ifdef USE_SHEEN
	uniform vec3 sheenColor;
	uniform float sheenRoughness;
	#ifdef USE_SHEEN_COLORMAP
		uniform sampler2D sheenColorMap;
	#endif
	#ifdef USE_SHEEN_ROUGHNESSMAP
		uniform sampler2D sheenRoughnessMap;
	#endif
#endif
#ifdef USE_ANISOTROPY
	uniform vec2 anisotropyVector;
	#ifdef USE_ANISOTROPYMAP
		uniform sampler2D anisotropyMap;
	#endif
#endif
varying vec3 vViewPosition;
#include <common>
#include <dithering_pars_fragment>
#include <color_pars_fragment>
#include <uv_pars_fragment>
#include <map_pars_fragment>
#include <alphamap_pars_fragment>
#include <alphatest_pars_fragment>
#include <alphahash_pars_fragment>
#include <aomap_pars_fragment>
#include <lightmap_pars_fragment>
#include <emissivemap_pars_fragment>
#include <iridescence_fragment>
#include <envmap_common_pars_fragment>
#include <envmap_physical_pars_fragment>
#include <fog_pars_fragment>
#include <lights_pars_begin>
#include <normal_pars_fragment>
#include <lights_physical_pars_fragment>
#include <transmission_pars_fragment>
#include <shadowmap_pars_fragment>
#include <bumpmap_pars_fragment>
#include <normalmap_pars_fragment>
#include <clearcoat_pars_fragment>
#include <iridescence_pars_fragment>
#include <roughnessmap_pars_fragment>
#include <metalnessmap_pars_fragment>
#include <logdepthbuf_pars_fragment>
#include <clipping_planes_pars_fragment>
void main() {
	vec4 diffuseColor = vec4( diffuse, opacity );
	#include <clipping_planes_fragment>
	ReflectedLight reflectedLight = ReflectedLight( vec3( 0.0 ), vec3( 0.0 ), vec3( 0.0 ), vec3( 0.0 ) );
	vec3 totalEmissiveRadiance = emissive;
	#include <logdepthbuf_fragment>
	#include <map_fragment>
	#include <color_fragment>
	#include <alphamap_fragment>
	#include <alphatest_fragment>
	#include <alphahash_fragment>
	#include <roughnessmap_fragment>
	#include <metalnessmap_fragment>
	#include <normal_fragment_begin>
	#include <normal_fragment_maps>
	#include <clearcoat_normal_fragment_begin>
	#include <clearcoat_normal_fragment_maps>
	#include <emissivemap_fragment>
	#include <lights_physical_fragment>
	#include <lights_fragment_begin>
	#include <lights_fragment_maps>
	#include <lights_fragment_end>
	#include <aomap_fragment>
	vec3 totalDiffuse = reflectedLight.directDiffuse + reflectedLight.indirectDiffuse;
	vec3 totalSpecular = reflectedLight.directSpecular + reflectedLight.indirectSpecular;
	#include <transmission_fragment>
	vec3 outgoingLight = totalDiffuse + totalSpecular + totalEmissiveRadiance;
	#ifdef USE_SHEEN
 
		outgoingLight = outgoingLight + sheenSpecularDirect + sheenSpecularIndirect;
 
 	#endif
	#ifdef USE_CLEARCOAT
		float dotNVcc = saturate( dot( geometryClearcoatNormal, geometryViewDir ) );
		vec3 Fcc = F_Schlick( material.clearcoatF0, material.clearcoatF90, dotNVcc );
		outgoingLight = outgoingLight * ( 1.0 - material.clearcoat * Fcc ) + ( clearcoatSpecularDirect + clearcoatSpecularIndirect ) * material.clearcoat;
	#endif
	#include <opaque_fragment>
	#include <tonemapping_fragment>
	#include <colorspace_fragment>
	#include <fog_fragment>
	#include <premultiplied_alpha_fragment>
	#include <dithering_fragment>
}`,meshtoon_vert:`#define TOON
varying vec3 vViewPosition;
#include <common>
#include <batching_pars_vertex>
#include <uv_pars_vertex>
#include <displacementmap_pars_vertex>
#include <color_pars_vertex>
#include <fog_pars_vertex>
#include <normal_pars_vertex>
#include <morphtarget_pars_vertex>
#include <skinning_pars_vertex>
#include <shadowmap_pars_vertex>
#include <logdepthbuf_pars_vertex>
#include <clipping_planes_pars_vertex>
void main() {
	#include <uv_vertex>
	#include <color_vertex>
	#include <morphinstance_vertex>
	#include <morphcolor_vertex>
	#include <batching_vertex>
	#include <beginnormal_vertex>
	#include <morphnormal_vertex>
	#include <skinbase_vertex>
	#include <skinnormal_vertex>
	#include <defaultnormal_vertex>
	#include <normal_vertex>
	#include <begin_vertex>
	#include <morphtarget_vertex>
	#include <skinning_vertex>
	#include <displacementmap_vertex>
	#include <project_vertex>
	#include <logdepthbuf_vertex>
	#include <clipping_planes_vertex>
	vViewPosition = - mvPosition.xyz;
	#include <worldpos_vertex>
	#include <shadowmap_vertex>
	#include <fog_vertex>
}`,meshtoon_frag:`#define TOON
uniform vec3 diffuse;
uniform vec3 emissive;
uniform float opacity;
#include <common>
#include <dithering_pars_fragment>
#include <color_pars_fragment>
#include <uv_pars_fragment>
#include <map_pars_fragment>
#include <alphamap_pars_fragment>
#include <alphatest_pars_fragment>
#include <alphahash_pars_fragment>
#include <aomap_pars_fragment>
#include <lightmap_pars_fragment>
#include <emissivemap_pars_fragment>
#include <gradientmap_pars_fragment>
#include <fog_pars_fragment>
#include <bsdfs>
#include <lights_pars_begin>
#include <normal_pars_fragment>
#include <lights_toon_pars_fragment>
#include <shadowmap_pars_fragment>
#include <bumpmap_pars_fragment>
#include <normalmap_pars_fragment>
#include <logdepthbuf_pars_fragment>
#include <clipping_planes_pars_fragment>
void main() {
	vec4 diffuseColor = vec4( diffuse, opacity );
	#include <clipping_planes_fragment>
	ReflectedLight reflectedLight = ReflectedLight( vec3( 0.0 ), vec3( 0.0 ), vec3( 0.0 ), vec3( 0.0 ) );
	vec3 totalEmissiveRadiance = emissive;
	#include <logdepthbuf_fragment>
	#include <map_fragment>
	#include <color_fragment>
	#include <alphamap_fragment>
	#include <alphatest_fragment>
	#include <alphahash_fragment>
	#include <normal_fragment_begin>
	#include <normal_fragment_maps>
	#include <emissivemap_fragment>
	#include <lights_toon_fragment>
	#include <lights_fragment_begin>
	#include <lights_fragment_maps>
	#include <lights_fragment_end>
	#include <aomap_fragment>
	vec3 outgoingLight = reflectedLight.directDiffuse + reflectedLight.indirectDiffuse + totalEmissiveRadiance;
	#include <opaque_fragment>
	#include <tonemapping_fragment>
	#include <colorspace_fragment>
	#include <fog_fragment>
	#include <premultiplied_alpha_fragment>
	#include <dithering_fragment>
}`,points_vert:`uniform float size;
uniform float scale;
#include <common>
#include <color_pars_vertex>
#include <fog_pars_vertex>
#include <morphtarget_pars_vertex>
#include <logdepthbuf_pars_vertex>
#include <clipping_planes_pars_vertex>
#ifdef USE_POINTS_UV
	varying vec2 vUv;
	uniform mat3 uvTransform;
#endif
void main() {
	#ifdef USE_POINTS_UV
		vUv = ( uvTransform * vec3( uv, 1 ) ).xy;
	#endif
	#include <color_vertex>
	#include <morphinstance_vertex>
	#include <morphcolor_vertex>
	#include <begin_vertex>
	#include <morphtarget_vertex>
	#include <project_vertex>
	gl_PointSize = size;
	#ifdef USE_SIZEATTENUATION
		bool isPerspective = isPerspectiveMatrix( projectionMatrix );
		if ( isPerspective ) gl_PointSize *= ( scale / - mvPosition.z );
	#endif
	#include <logdepthbuf_vertex>
	#include <clipping_planes_vertex>
	#include <worldpos_vertex>
	#include <fog_vertex>
}`,points_frag:`uniform vec3 diffuse;
uniform float opacity;
#include <common>
#include <color_pars_fragment>
#include <map_particle_pars_fragment>
#include <alphatest_pars_fragment>
#include <alphahash_pars_fragment>
#include <fog_pars_fragment>
#include <logdepthbuf_pars_fragment>
#include <clipping_planes_pars_fragment>
void main() {
	vec4 diffuseColor = vec4( diffuse, opacity );
	#include <clipping_planes_fragment>
	vec3 outgoingLight = vec3( 0.0 );
	#include <logdepthbuf_fragment>
	#include <map_particle_fragment>
	#include <color_fragment>
	#include <alphatest_fragment>
	#include <alphahash_fragment>
	outgoingLight = diffuseColor.rgb;
	#include <opaque_fragment>
	#include <tonemapping_fragment>
	#include <colorspace_fragment>
	#include <fog_fragment>
	#include <premultiplied_alpha_fragment>
}`,shadow_vert:`#include <common>
#include <batching_pars_vertex>
#include <fog_pars_vertex>
#include <morphtarget_pars_vertex>
#include <skinning_pars_vertex>
#include <logdepthbuf_pars_vertex>
#include <shadowmap_pars_vertex>
void main() {
	#include <batching_vertex>
	#include <beginnormal_vertex>
	#include <morphinstance_vertex>
	#include <morphnormal_vertex>
	#include <skinbase_vertex>
	#include <skinnormal_vertex>
	#include <defaultnormal_vertex>
	#include <begin_vertex>
	#include <morphtarget_vertex>
	#include <skinning_vertex>
	#include <project_vertex>
	#include <logdepthbuf_vertex>
	#include <worldpos_vertex>
	#include <shadowmap_vertex>
	#include <fog_vertex>
}`,shadow_frag:`uniform vec3 color;
uniform float opacity;
#include <common>
#include <fog_pars_fragment>
#include <bsdfs>
#include <lights_pars_begin>
#include <logdepthbuf_pars_fragment>
#include <shadowmap_pars_fragment>
#include <shadowmask_pars_fragment>
void main() {
	#include <logdepthbuf_fragment>
	gl_FragColor = vec4( color, opacity * ( 1.0 - getShadowMask() ) );
	#include <tonemapping_fragment>
	#include <colorspace_fragment>
	#include <fog_fragment>
	#include <premultiplied_alpha_fragment>
}`,sprite_vert:`uniform float rotation;
uniform vec2 center;
#include <common>
#include <uv_pars_vertex>
#include <fog_pars_vertex>
#include <logdepthbuf_pars_vertex>
#include <clipping_planes_pars_vertex>
void main() {
	#include <uv_vertex>
	vec4 mvPosition = modelViewMatrix[ 3 ];
	vec2 scale = vec2( length( modelMatrix[ 0 ].xyz ), length( modelMatrix[ 1 ].xyz ) );
	#ifndef USE_SIZEATTENUATION
		bool isPerspective = isPerspectiveMatrix( projectionMatrix );
		if ( isPerspective ) scale *= - mvPosition.z;
	#endif
	vec2 alignedPosition = ( position.xy - ( center - vec2( 0.5 ) ) ) * scale;
	vec2 rotatedPosition;
	rotatedPosition.x = cos( rotation ) * alignedPosition.x - sin( rotation ) * alignedPosition.y;
	rotatedPosition.y = sin( rotation ) * alignedPosition.x + cos( rotation ) * alignedPosition.y;
	mvPosition.xy += rotatedPosition;
	gl_Position = projectionMatrix * mvPosition;
	#include <logdepthbuf_vertex>
	#include <clipping_planes_vertex>
	#include <fog_vertex>
}`,sprite_frag:`uniform vec3 diffuse;
uniform float opacity;
#include <common>
#include <uv_pars_fragment>
#include <map_pars_fragment>
#include <alphamap_pars_fragment>
#include <alphatest_pars_fragment>
#include <alphahash_pars_fragment>
#include <fog_pars_fragment>
#include <logdepthbuf_pars_fragment>
#include <clipping_planes_pars_fragment>
void main() {
	vec4 diffuseColor = vec4( diffuse, opacity );
	#include <clipping_planes_fragment>
	vec3 outgoingLight = vec3( 0.0 );
	#include <logdepthbuf_fragment>
	#include <map_fragment>
	#include <alphamap_fragment>
	#include <alphatest_fragment>
	#include <alphahash_fragment>
	outgoingLight = diffuseColor.rgb;
	#include <opaque_fragment>
	#include <tonemapping_fragment>
	#include <colorspace_fragment>
	#include <fog_fragment>
}`},Le={common:{diffuse:{value:new dt(16777215)},opacity:{value:1},map:{value:null},mapTransform:{value:new it},alphaMap:{value:null},alphaMapTransform:{value:new it},alphaTest:{value:0}},specularmap:{specularMap:{value:null},specularMapTransform:{value:new it}},envmap:{envMap:{value:null},envMapRotation:{value:new it},reflectivity:{value:1},ior:{value:1.5},refractionRatio:{value:.98},dfgLUT:{value:null}},aomap:{aoMap:{value:null},aoMapIntensity:{value:1},aoMapTransform:{value:new it}},lightmap:{lightMap:{value:null},lightMapIntensity:{value:1},lightMapTransform:{value:new it}},bumpmap:{bumpMap:{value:null},bumpMapTransform:{value:new it},bumpScale:{value:1}},normalmap:{normalMap:{value:null},normalMapTransform:{value:new it},normalScale:{value:new ge(1,1)}},displacementmap:{displacementMap:{value:null},displacementMapTransform:{value:new it},displacementScale:{value:1},displacementBias:{value:0}},emissivemap:{emissiveMap:{value:null},emissiveMapTransform:{value:new it}},metalnessmap:{metalnessMap:{value:null},metalnessMapTransform:{value:new it}},roughnessmap:{roughnessMap:{value:null},roughnessMapTransform:{value:new it}},gradientmap:{gradientMap:{value:null}},fog:{fogDensity:{value:25e-5},fogNear:{value:1},fogFar:{value:2e3},fogColor:{value:new dt(16777215)}},lights:{ambientLightColor:{value:[]},lightProbe:{value:[]},sunLights:{value:[],properties:{direction:{},color:{}}},sunLightShadows:{value:[],properties:{shadowIntensity:1,shadowBias:{},shadowNormalBias:{},shadowRadius:{},shadowMapSize:{}}},sunShadowMatrix:{value:[]},sunShadowCascade:{value:[]},directionalLights:{value:[],properties:{direction:{},color:{}}},directionalLightShadows:{value:[],properties:{shadowIntensity:1,shadowBias:{},shadowNormalBias:{},shadowRadius:{},shadowMapSize:{}}},directionalShadowMatrix:{value:[]},spotLights:{value:[],properties:{color:{},position:{},direction:{},distance:{},coneCos:{},penumbraCos:{},decay:{}}},spotLightShadows:{value:[],properties:{shadowIntensity:1,shadowBias:{},shadowNormalBias:{},shadowRadius:{},shadowMapSize:{}}},spotLightMap:{value:[]},spotLightMatrix:{value:[]},pointLights:{value:[],properties:{color:{},position:{},decay:{},distance:{}}},pointLightShadows:{value:[],properties:{shadowIntensity:1,shadowBias:{},shadowNormalBias:{},shadowRadius:{},shadowMapSize:{},shadowCameraNear:{},shadowCameraFar:{}}},pointShadowMatrix:{value:[]},hemisphereLights:{value:[],properties:{direction:{},skyColor:{},groundColor:{}}},rectAreaLights:{value:[],properties:{color:{},position:{},width:{},height:{}}},ltc_1:{value:null},ltc_2:{value:null},probesSH:{value:[]},probesMin:{value:[]},probesMax:{value:[]},probesResolution:{value:[]}},points:{diffuse:{value:new dt(16777215)},opacity:{value:1},size:{value:1},scale:{value:1},map:{value:null},alphaMap:{value:null},alphaMapTransform:{value:new it},alphaTest:{value:0},uvTransform:{value:new it}},sprite:{diffuse:{value:new dt(16777215)},opacity:{value:1},center:{value:new ge(.5,.5)},rotation:{value:0},map:{value:null},mapTransform:{value:new it},alphaMap:{value:null},alphaMapTransform:{value:new it},alphaTest:{value:0}}},or={basic:{uniforms:mi([Le.common,Le.specularmap,Le.envmap,Le.aomap,Le.lightmap,Le.fog]),vertexShader:ot.meshbasic_vert,fragmentShader:ot.meshbasic_frag},lambert:{uniforms:mi([Le.common,Le.specularmap,Le.envmap,Le.aomap,Le.lightmap,Le.emissivemap,Le.bumpmap,Le.normalmap,Le.displacementmap,Le.fog,Le.lights,{emissive:{value:new dt(0)},envMapIntensity:{value:1}}]),vertexShader:ot.meshlambert_vert,fragmentShader:ot.meshlambert_frag},phong:{uniforms:mi([Le.common,Le.specularmap,Le.envmap,Le.aomap,Le.lightmap,Le.emissivemap,Le.bumpmap,Le.normalmap,Le.displacementmap,Le.fog,Le.lights,{emissive:{value:new dt(0)},specular:{value:new dt(1118481)},shininess:{value:30},envMapIntensity:{value:1}}]),vertexShader:ot.meshphong_vert,fragmentShader:ot.meshphong_frag},standard:{uniforms:mi([Le.common,Le.envmap,Le.aomap,Le.lightmap,Le.emissivemap,Le.bumpmap,Le.normalmap,Le.displacementmap,Le.roughnessmap,Le.metalnessmap,Le.fog,Le.lights,{emissive:{value:new dt(0)},roughness:{value:1},metalness:{value:0},envMapIntensity:{value:1}}]),vertexShader:ot.meshphysical_vert,fragmentShader:ot.meshphysical_frag},toon:{uniforms:mi([Le.common,Le.aomap,Le.lightmap,Le.emissivemap,Le.bumpmap,Le.normalmap,Le.displacementmap,Le.gradientmap,Le.fog,Le.lights,{emissive:{value:new dt(0)}}]),vertexShader:ot.meshtoon_vert,fragmentShader:ot.meshtoon_frag},matcap:{uniforms:mi([Le.common,Le.bumpmap,Le.normalmap,Le.displacementmap,Le.fog,{matcap:{value:null}}]),vertexShader:ot.meshmatcap_vert,fragmentShader:ot.meshmatcap_frag},points:{uniforms:mi([Le.points,Le.fog]),vertexShader:ot.points_vert,fragmentShader:ot.points_frag},dashed:{uniforms:mi([Le.common,Le.fog,{scale:{value:1},dashSize:{value:1},totalSize:{value:2}}]),vertexShader:ot.linedashed_vert,fragmentShader:ot.linedashed_frag},depth:{uniforms:mi([Le.common,Le.displacementmap]),vertexShader:ot.depth_vert,fragmentShader:ot.depth_frag},normal:{uniforms:mi([Le.common,Le.bumpmap,Le.normalmap,Le.displacementmap,{opacity:{value:1}}]),vertexShader:ot.meshnormal_vert,fragmentShader:ot.meshnormal_frag},sprite:{uniforms:mi([Le.sprite,Le.fog]),vertexShader:ot.sprite_vert,fragmentShader:ot.sprite_frag},background:{uniforms:{uvTransform:{value:new it},t2D:{value:null},backgroundIntensity:{value:1}},vertexShader:ot.background_vert,fragmentShader:ot.background_frag},backgroundCube:{uniforms:{envMap:{value:null},backgroundBlurriness:{value:0},backgroundIntensity:{value:1},backgroundRotation:{value:new it}},vertexShader:ot.backgroundCube_vert,fragmentShader:ot.backgroundCube_frag},cube:{uniforms:{tCube:{value:null},tFlip:{value:-1},opacity:{value:1}},vertexShader:ot.cube_vert,fragmentShader:ot.cube_frag},equirect:{uniforms:{tEquirect:{value:null}},vertexShader:ot.equirect_vert,fragmentShader:ot.equirect_frag},distance:{uniforms:mi([Le.common,Le.displacementmap,{referencePosition:{value:new I},nearDistance:{value:1},farDistance:{value:1e3}}]),vertexShader:ot.distance_vert,fragmentShader:ot.distance_frag},shadow:{uniforms:mi([Le.lights,Le.fog,{color:{value:new dt(0)},opacity:{value:1}}]),vertexShader:ot.shadow_vert,fragmentShader:ot.shadow_frag}};or.physical={uniforms:mi([or.standard.uniforms,{clearcoat:{value:0},clearcoatMap:{value:null},clearcoatMapTransform:{value:new it},clearcoatNormalMap:{value:null},clearcoatNormalMapTransform:{value:new it},clearcoatNormalScale:{value:new ge(1,1)},clearcoatRoughness:{value:0},clearcoatRoughnessMap:{value:null},clearcoatRoughnessMapTransform:{value:new it},diffuseRoughness:{value:0},diffuseRoughnessMap:{value:null},diffuseRoughnessMapTransform:{value:new it},dispersion:{value:0},retroreflectivity:{value:0},iridescence:{value:0},iridescenceMap:{value:null},iridescenceMapTransform:{value:new it},iridescenceIOR:{value:1.3},iridescenceThicknessMinimum:{value:100},iridescenceThicknessMaximum:{value:400},iridescenceThicknessMap:{value:null},iridescenceThicknessMapTransform:{value:new it},sheen:{value:0},sheenColor:{value:new dt(0)},sheenColorMap:{value:null},sheenColorMapTransform:{value:new it},sheenRoughness:{value:1},sheenRoughnessMap:{value:null},sheenRoughnessMapTransform:{value:new it},transmission:{value:0},transmissionMap:{value:null},transmissionMapTransform:{value:new it},transmissionSamplerSize:{value:new ge},transmissionSamplerMap:{value:null},thickness:{value:0},thicknessMap:{value:null},thicknessMapTransform:{value:new it},attenuationDistance:{value:0},attenuationColor:{value:new dt(0)},specularColor:{value:new dt(1,1,1)},specularColorMap:{value:null},specularColorMapTransform:{value:new it},specularIntensity:{value:1},specularIntensityMap:{value:null},specularIntensityMapTransform:{value:new it},anisotropyVector:{value:new ge},anisotropyMap:{value:null},anisotropyMapTransform:{value:new it}}]),vertexShader:ot.meshphysical_vert,fragmentShader:ot.meshphysical_frag};var ns={r:0,b:0,g:0},Kf=new gt,Cu=new it;Cu.set(-1,0,0,0,1,0,0,0,1);function $f(e,t,i,r,a,o){let s=new dt(0),n=a===!0?0:1,l,h,c=null,u=0,d=null;function f(y){let M=y.isScene===!0?y.background:null;if(M&&M.isTexture){let b=y.backgroundBlurriness>0;M=t.get(M,b)}return M}function p(y){let M=!1,b=f(y);b===null?g(s,n):b&&b.isColor&&(g(b,1),M=!0);let E=e.xr.getEnvironmentBlendMode();E==="additive"?i.buffers.color.setClear(0,0,0,1,o):E==="alpha-blend"&&i.buffers.color.setClear(0,0,0,0,o),(e.autoClear||M)&&(i.buffers.depth.setTest(!0),i.buffers.depth.setMask(!0),i.buffers.color.setMask(!0),e.clear(e.autoClearColor,e.autoClearDepth,e.autoClearStencil))}function x(y,M){let b=f(M);b&&b.isCubeTexture?(h===void 0&&(h=new Nt(new wr(1,1,1),new $i({name:"BackgroundCubeMaterial",uniforms:Ba(or.backgroundCube.uniforms),vertexShader:or.backgroundCube.vertexShader,fragmentShader:or.backgroundCube.fragmentShader,side:1,depthTest:!1,depthWrite:!1,fog:!1,allowOverride:!1})),h.geometry.deleteAttribute("normal"),h.geometry.deleteAttribute("uv"),h.onBeforeRender=function(E,T,v){this.matrixWorld.copyPosition(v.matrixWorld)},Object.defineProperty(h.material,"envMap",{get:function(){return this.uniforms.envMap.value}}),r.update(h)),h.material.uniforms.envMap.value=b,h.material.uniforms.backgroundBlurriness.value=M.backgroundBlurriness,h.material.uniforms.backgroundIntensity.value=M.backgroundIntensity,h.material.uniforms.backgroundRotation.value.setFromMatrix4(Kf.makeRotationFromEuler(M.backgroundRotation)).transpose(),b.isRenderTargetTexture===!1&&h.material.uniforms.backgroundRotation.value.premultiply(Cu),h.material.toneMapped=ft.getTransfer(b.colorSpace)!==Ds,(c!==b||u!==b.version||d!==e.toneMapping)&&(h.material.needsUpdate=!0,c=b,u=b.version,d=e.toneMapping),h.layers.enableAll(),y.unshift(h,h.geometry,h.material,0,0,null)):b&&b.isTexture&&(l===void 0&&(l=new Nt(new Br(2,2),new $i({name:"BackgroundMaterial",uniforms:Ba(or.background.uniforms),vertexShader:or.background.vertexShader,fragmentShader:or.background.fragmentShader,side:0,depthTest:!1,depthWrite:!1,fog:!1,allowOverride:!1})),l.geometry.deleteAttribute("normal"),Object.defineProperty(l.material,"map",{get:function(){return this.uniforms.t2D.value}}),r.update(l)),l.material.uniforms.t2D.value=b,l.material.uniforms.backgroundIntensity.value=M.backgroundIntensity,l.material.toneMapped=ft.getTransfer(b.colorSpace)!==Ds,b.matrixAutoUpdate===!0&&b.updateMatrix(),l.material.uniforms.uvTransform.value.copy(b.matrix),(c!==b||u!==b.version||d!==e.toneMapping)&&(l.material.needsUpdate=!0,c=b,u=b.version,d=e.toneMapping),l.layers.enableAll(),y.unshift(l,l.geometry,l.material,0,0,null))}function g(y,M){y.getRGB(ns,wu(e)),i.buffers.color.setClear(ns.r,ns.g,ns.b,M,o)}function m(){h!==void 0&&(h.geometry.dispose(),h.material.dispose(),h=void 0),l!==void 0&&(l.geometry.dispose(),l.material.dispose(),l=void 0)}return{getClearColor:function(){return s},setClearColor:function(y,M=1){s.set(y),n=M,g(s,n)},getClearAlpha:function(){return n},setClearAlpha:function(y){n=y,g(s,n)},render:p,addToRenderList:x,dispose:m}}function Qf(e,t){let i=e.getParameter(e.MAX_VERTEX_ATTRIBS),r={},a=d(null),o=a,s=!1;function n(R,L,G,P,z){let H=!1,D=u(R,P,G,L);o!==D&&(o=D,h(o.object)),H=f(R,P,G,z),H&&p(R,P,G,z),z!==null&&t.update(z,e.ELEMENT_ARRAY_BUFFER),(H||s)&&(s=!1,b(R,L,G,P),z!==null&&e.bindBuffer(e.ELEMENT_ARRAY_BUFFER,t.get(z).buffer))}function l(){return e.createVertexArray()}function h(R){return e.bindVertexArray(R)}function c(R){return e.deleteVertexArray(R)}function u(R,L,G,P){let z=P.wireframe===!0,H=r[L.id];H===void 0&&(H={},r[L.id]=H);let D=R.isInstancedMesh===!0?R.id:0,ae=H[D];ae===void 0&&(ae={},H[D]=ae);let j=ae[G.id];j===void 0&&(j={},ae[G.id]=j);let Y=j[z];return Y===void 0&&(Y=d(l()),j[z]=Y),Y}function d(R){let L=[],G=[],P=[];for(let z=0;z<i;z++)L[z]=0,G[z]=0,P[z]=0;return{geometry:null,program:null,wireframe:!1,newAttributes:L,enabledAttributes:G,attributeDivisors:P,object:R,attributes:{},index:null}}function f(R,L,G,P){let z=o.attributes,H=L.attributes,D=0,ae=G.getAttributes();for(let j in ae)if(ae[j].location>=0){let Y=z[j],ie=H[j];if(ie===void 0&&(j==="instanceMatrix"&&R.instanceMatrix&&(ie=R.instanceMatrix),j==="instanceColor"&&R.instanceColor&&(ie=R.instanceColor)),Y===void 0||Y.attribute!==ie||ie&&Y.data!==ie.data)return!0;D++}return o.attributesNum!==D||o.index!==P}function p(R,L,G,P){let z={},H=L.attributes,D=0,ae=G.getAttributes();for(let j in ae)if(ae[j].location>=0){let Y=H[j];Y===void 0&&(j==="instanceMatrix"&&R.instanceMatrix&&(Y=R.instanceMatrix),j==="instanceColor"&&R.instanceColor&&(Y=R.instanceColor));let ie={};ie.attribute=Y,Y&&Y.data&&(ie.data=Y.data),z[j]=ie,D++}o.attributes=z,o.attributesNum=D,o.index=P}function x(){let R=o.newAttributes;for(let L=0,G=R.length;L<G;L++)R[L]=0}function g(R){m(R,0)}function m(R,L){let G=o.newAttributes,P=o.enabledAttributes,z=o.attributeDivisors;G[R]=1,P[R]===0&&(e.enableVertexAttribArray(R),P[R]=1),z[R]!==L&&(e.vertexAttribDivisor(R,L),z[R]=L)}function y(){let R=o.newAttributes,L=o.enabledAttributes;for(let G=0,P=L.length;G<P;G++)L[G]!==R[G]&&(e.disableVertexAttribArray(G),L[G]=0)}function M(R,L,G,P,z,H,D){D===!0?e.vertexAttribIPointer(R,L,G,z,H):e.vertexAttribPointer(R,L,G,P,z,H)}function b(R,L,G,P){x();let z=P.attributes,H=G.getAttributes(),D=L.defaultAttributeValues;for(let ae in H){let j=H[ae];if(j.location>=0){let Y=z[ae];if(Y===void 0&&(ae==="instanceMatrix"&&R.instanceMatrix&&(Y=R.instanceMatrix),ae==="instanceColor"&&R.instanceColor&&(Y=R.instanceColor)),Y!==void 0){let ie=Y.normalized,Se=Y.itemSize,K=t.get(Y);if(K===void 0)continue;let re=K.buffer,xe=K.type,Ne=K.bytesPerElement,Me=xe===e.INT||xe===e.UNSIGNED_INT||Y.gpuType===1013;if(Y.isInterleavedBufferAttribute){let Q=Y.data,ke=Q.isInterleavedBuffer===!0?Q.stride:Q.itemSize,$e=Y.offset;if(Q.isInstancedInterleavedBuffer||Q.isInstancedBufferAttribute){for(let De=0;De<j.locationSize;De++)m(j.location+De,Q.meshPerAttribute);R.isInstancedMesh!==!0&&P._maxInstanceCount===void 0&&(P._maxInstanceCount=Q.meshPerAttribute*Q.count)}else for(let De=0;De<j.locationSize;De++)g(j.location+De);e.bindBuffer(e.ARRAY_BUFFER,re);for(let De=0;De<j.locationSize;De++)M(j.location+De,Se/j.locationSize,xe,ie,ke*Ne,($e+Se/j.locationSize*De)*Ne,Me)}else{if(Y.isInstancedBufferAttribute){for(let Q=0;Q<j.locationSize;Q++)m(j.location+Q,Y.meshPerAttribute);R.isInstancedMesh!==!0&&P._maxInstanceCount===void 0&&(P._maxInstanceCount=Y.meshPerAttribute*Y.count)}else for(let Q=0;Q<j.locationSize;Q++)g(j.location+Q);e.bindBuffer(e.ARRAY_BUFFER,re);for(let Q=0;Q<j.locationSize;Q++)M(j.location+Q,Se/j.locationSize,xe,ie,Se*Ne,Se/j.locationSize*Q*Ne,Me)}}else if(D!==void 0){let ie=D[ae];if(ie!==void 0)switch(ie.length){case 2:e.vertexAttrib2fv(j.location,ie);break;case 3:e.vertexAttrib3fv(j.location,ie);break;case 4:e.vertexAttrib4fv(j.location,ie);break;default:e.vertexAttrib1fv(j.location,ie)}}}}y()}function E(){A();for(let R in r){let L=r[R];for(let G in L){let P=L[G];for(let z in P){let H=P[z];for(let D in H)c(H[D].object),delete H[D];delete P[z]}}delete r[R]}}function T(R){if(r[R.id]===void 0)return;let L=r[R.id];for(let G in L){let P=L[G];for(let z in P){let H=P[z];for(let D in H)c(H[D].object),delete H[D];delete P[z]}}delete r[R.id]}function v(R){for(let L in r){let G=r[L];for(let P in G){let z=G[P];if(z[R.id]===void 0)continue;let H=z[R.id];for(let D in H)c(H[D].object),delete H[D];delete z[R.id]}}}function _(R){for(let L in r){let G=r[L],P=R.isInstancedMesh===!0?R.id:0,z=G[P];if(z!==void 0){for(let H in z){let D=z[H];for(let ae in D)c(D[ae].object),delete D[ae];delete z[H]}delete G[P],Object.keys(G).length===0&&delete r[L]}}}function A(){C(),s=!0,o!==a&&(o=a,h(o.object))}function C(){a.geometry=null,a.program=null,a.wireframe=!1}return{setup:n,reset:A,resetDefaultState:C,dispose:E,releaseStatesOfGeometry:T,releaseStatesOfObject:_,releaseStatesOfProgram:v,initAttributes:x,enableAttribute:g,disableUnusedAttributes:y}}function ep(e,t,i){let r;function a(l){r=l}function o(l,h){e.drawArrays(r,l,h),i.update(h,r,1)}function s(l,h,c){c!==0&&(e.drawArraysInstanced(r,l,h,c),i.update(h,r,c))}function n(l,h,c){if(c===0)return;t.get("WEBGL_multi_draw").multiDrawArraysWEBGL(r,l,0,h,0,c);let u=0;for(let d=0;d<c;d++)u+=h[d];i.update(u,r,1)}this.setMode=a,this.render=o,this.renderInstances=s,this.renderMultiDraw=n}function tp(e,t,i,r){let a;function o(){if(a!==void 0)return a;if(t.has("EXT_texture_filter_anisotropic")===!0){let v=t.get("EXT_texture_filter_anisotropic");a=e.getParameter(v.MAX_TEXTURE_MAX_ANISOTROPY_EXT)}else a=0;return a}function s(v){return v===1023||r.convert(v)===e.getParameter(e.IMPLEMENTATION_COLOR_READ_FORMAT)}function n(v){let _=v===1016&&(t.has("EXT_color_buffer_half_float")||t.has("EXT_color_buffer_float"));return!(v!==1009&&v!==1015&&!_&&r.convert(v)!==e.getParameter(e.IMPLEMENTATION_COLOR_READ_TYPE))}function l(v){if(v==="highp"){if(e.getShaderPrecisionFormat(e.VERTEX_SHADER,e.HIGH_FLOAT).precision>0&&e.getShaderPrecisionFormat(e.FRAGMENT_SHADER,e.HIGH_FLOAT).precision>0)return"highp";v="mediump"}return v==="mediump"&&e.getShaderPrecisionFormat(e.VERTEX_SHADER,e.MEDIUM_FLOAT).precision>0&&e.getShaderPrecisionFormat(e.FRAGMENT_SHADER,e.MEDIUM_FLOAT).precision>0?"mediump":"lowp"}let h=i.precision===void 0?"highp":i.precision,c=l(h);c!==h&&(Je("WebGLRenderer:",h,"not supported, using",c,"instead."),h=c);let u=i.reversedDepthBuffer===!0&&t.has("EXT_clip_control"),d=i.logarithmicDepthBuffer===!0&&u===!1;i.reversedDepthBuffer===!0&&u===!1&&Je("WebGLRenderer: Unable to use reversed depth buffer due to missing EXT_clip_control extension. Fallback to default depth buffer."),i.logarithmicDepthBuffer===!0&&u===!0&&Je("WebGLRenderer: Logarithmic depth buffer is ignored when reversed depth buffer is enabled.");let f=e.getParameter(e.MAX_TEXTURE_IMAGE_UNITS),p=e.getParameter(e.MAX_VERTEX_TEXTURE_IMAGE_UNITS),x=e.getParameter(e.MAX_TEXTURE_SIZE),g=e.getParameter(e.MAX_CUBE_MAP_TEXTURE_SIZE),m=e.getParameter(e.MAX_VERTEX_ATTRIBS),y=e.getParameter(e.MAX_VERTEX_UNIFORM_VECTORS),M=e.getParameter(e.MAX_VARYING_VECTORS),b=e.getParameter(e.MAX_FRAGMENT_UNIFORM_VECTORS),E=e.getParameter(e.MAX_SAMPLES),T=e.getParameter(e.SAMPLES);return{isWebGL2:!0,getMaxAnisotropy:o,getMaxPrecision:l,textureFormatReadable:s,textureTypeReadable:n,precision:h,logarithmicDepthBuffer:d,reversedDepthBuffer:u,maxTextures:f,maxVertexTextures:p,maxTextureSize:x,maxCubemapSize:g,maxAttributes:m,maxVertexUniforms:y,maxVaryings:M,maxFragmentUniforms:b,maxSamples:E,samples:T}}function ip(e){let t=this,i=null,r=0,a=!1,o=!1,s=new Or,n=new it,l={value:null,needsUpdate:!1};this.uniform=l,this.numPlanes=0,this.numIntersection=0,this.init=function(u,d){let f=u.length!==0||d||r!==0||a;return a=d,r=u.length,f},this.beginShadows=function(){o=!0,c(null)},this.endShadows=function(){o=!1},this.setGlobalState=function(u,d){i=c(u,d,0)},this.setState=function(u,d,f){let p=u.clippingPlanes,x=u.clipIntersection,g=u.clipShadows,m=e.get(u);if(!a||p===null||p.length===0||o&&!g)o?c(null):h();else{let y=o?0:r,M=y*4,b=m.clippingState||null;l.value=b,b=c(p,d,M,f);for(let E=0;E!==M;++E)b[E]=i[E];m.clippingState=b,this.numIntersection=x?this.numPlanes:0,this.numPlanes+=y}};function h(){l.value!==i&&(l.value=i,l.needsUpdate=r>0),t.numPlanes=r,t.numIntersection=0}function c(u,d,f,p){let x=u===null?0:u.length,g=null;if(x!==0){if(g=l.value,p!==!0||g===null){let m=f+x*4,y=d.matrixWorldInverse;n.getNormalMatrix(y),(g===null||g.length<m)&&(g=new Float32Array(m));for(let M=0,b=f;M!==x;++M,b+=4)s.copy(u[M]).applyMatrix4(y,n),s.normal.toArray(g,b),g[b+3]=s.constant}l.value=g,l.needsUpdate=!0}return t.numPlanes=x,t.numIntersection=0,g}}var Sl=class extends gi{constructor(e=1,t={}){super(e,e,t),this.isWebGLCubeRenderTarget=!0;let i={width:e,height:e,depth:1},r=[i,i,i,i,i,i];this.texture=new Zc(r),this._setTextureOptions(t),this.texture.isRenderTargetTexture=!0}fromEquirectangularTexture(e,t){this.texture.type=t.type,this.texture.colorSpace=t.colorSpace,this.texture.generateMipmaps=t.generateMipmaps,this.texture.minFilter=t.minFilter,this.texture.magFilter=t.magFilter;let i={uniforms:{tEquirect:{value:null}},vertexShader:`

				varying vec3 vWorldDirection;

				vec3 transformDirection( in vec3 dir, in mat4 matrix ) {

					return normalize( ( matrix * vec4( dir, 0.0 ) ).xyz );

				}

				void main() {

					vWorldDirection = transformDirection( position, modelMatrix );

					#include <begin_vertex>
					#include <project_vertex>

				}
			`,fragmentShader:`

				uniform sampler2D tEquirect;

				varying vec3 vWorldDirection;

				#include <common>

				void main() {

					vec3 direction = normalize( vWorldDirection );

					vec2 sampleUV = equirectUv( direction );

					gl_FragColor = texture2D( tEquirect, sampleUV );

				}
			`},r=new wr(5,5,5),a=new $i({name:"CubemapFromEquirect",uniforms:Ba(i.uniforms),vertexShader:i.vertexShader,fragmentShader:i.fragmentShader,side:1,blending:0});a.uniforms.tEquirect.value=t;let o=new Nt(r,a),s=t.minFilter;return t.minFilter===1008&&(t.minFilter=jt),new tl(1,10,this).update(e,o),t.minFilter=s,o.geometry.dispose(),o.material.dispose(),this}clear(e,t=!0,i=!0,r=!0){let a=e.getRenderTarget();for(let o=0;o<6;o++)e.setRenderTarget(this,o),e.clear(t,i,r);e.setRenderTarget(a)}},Ah=256,rp=3,ap=20,Pu=256,Lu=16,op=3,sp=new I,np=new dt,Rh=class Nu{constructor(t){this._renderer=t,this._cubeSize=0,this._sourceTarget=null,this._cubeCamera=new tl(1,10,null),this._boxMesh=new Nt(new wr(5,5,5),null),this._cubemapMaterial=null,this._equirectMaterial=null,this._blurMaterial=null,this._ggxMaterial=null,this._integrationMaterial=null}fromScene(t,i=0,r=.1,a=100,o={}){let{size:s=256,position:n=sp}=o,l=this._renderer;this._setSize(s);let h=this._allocateTarget(),c=this._getSourceTarget(!0);i>0&&(c.texture.mipmapsAutoUpdate=!1);let u=l.autoClear,d=l.autoClearColor,f=l.autoClearDepth,p=l.autoClearStencil,x=t.background;l.autoClear=!0,l.autoClearColor=!0,l.autoClearDepth=!0,l.autoClearStencil=!0,x===null&&(t.background=l.getClearColor(np));let g=new tl(r,a,c);return g.position.copy(n),g.update(l,t),l.autoClear=u,l.autoClearColor=d,l.autoClearDepth=f,l.autoClearStencil=p,t.background=x,i>0&&(c.texture.mipmapsAutoUpdate=!0,this._blur(h,i)),this._applyPMREM(h),h}fromEquirectangular(t,i=null){return this._fromTexture(t,i)}fromCubemap(t,i=null){return this._fromTexture(t,i)}compileCubemapShader(){this._cubemapMaterial===null&&(this._cubemapMaterial=Lh(),this._compileMaterial(this._cubemapMaterial))}compileEquirectangularShader(){this._equirectMaterial===null&&(this._equirectMaterial=Ph(),this._compileMaterial(this._equirectMaterial))}dispose(){this._sourceTarget!==null&&this._sourceTarget.dispose(),this._cubemapMaterial!==null&&this._cubemapMaterial.dispose(),this._equirectMaterial!==null&&this._equirectMaterial.dispose(),this._blurMaterial!==null&&this._blurMaterial.dispose(),this._ggxMaterial!==null&&this._ggxMaterial.dispose(),this._integrationMaterial!==null&&this._integrationMaterial.dispose(),this._boxMesh.geometry.dispose()}static lodToRoughness(t,i){return i>0?1-Math.sqrt(1-t/i):0}_setSize(t){this._cubeSize=Math.max(Ah,h0(t))}_fromTexture(t,i){t.mapping===301||t.mapping===302?this._setSize(t.image.length===0?Ah:t.image[0].width||t.image[0].image.width):this._setSize(t.image.width/4);let r=i||this._allocateTarget();return this._textureToCubemap(t),this._applyPMREM(r),r}_allocateTarget(){let t=this._cubeSize,i=Ch(t,!1,!1),r=Math.log2(t)-rp;for(let a=0;a<=r;a++)i.texture.mipmaps.push({width:t>>a,height:t>>a});return i.texture.name="PMREM",i.texture.isPMREMTexture=!0,i}_getSourceTarget(t=!1){let i=this._cubeSize,r=this._sourceTarget;return(r===null||r.width!==i||t&&r.depthBuffer===!1)&&(r!==null&&r.dispose(),this._sourceTarget=Ch(i,!0,t)),this._sourceTarget}_compileMaterial(t){this._boxMesh.material=t,this._renderer.compile(this._boxMesh,this._cubeCamera.children[0])}_renderCube(t,i,r){let a=this._boxMesh,o=this._cubeCamera;a.material=r,o.renderTarget=t,o.activeMipmapLevel=i;let s=t.width>>i;t.viewport.set(0,0,s,s),o.update(this._renderer,a),t.viewport.set(0,0,t.width,t.height)}_textureToCubemap(t){let i;t.mapping===301||t.mapping===302?(this._cubemapMaterial===null&&(this._cubemapMaterial=Lh()),i=this._cubemapMaterial,i.uniforms.flipEnvMap.value=t.isRenderTargetTexture===!1?-1:1):(this._equirectMaterial===null&&(this._equirectMaterial=Ph()),i=this._equirectMaterial),i.uniforms.envMap.value=t,this._renderCube(this._getSourceTarget(),0,i)}_applyPMREM(t){this._ggxMaterial===null&&(this._ggxMaterial=lp(),this._integrationMaterial=hp());let i=this._cubeSize,r=t.texture.mipmaps.length-1,a=this._ggxMaterial.uniforms;a.envMap.value=this._sourceTarget.texture;let o=this._integrationMaterial.uniforms;o.envMap.value=this._sourceTarget.texture,o.sourceLod.value=Math.log2(i/Lu);for(let s=0;s<=r;s++){let n=Nu.lodToRoughness(s,r);if(s>r-op)o.roughness.value=n,this._renderCube(t,s,this._integrationMaterial);else{let l=n>0?Math.log2(i)+.5*Math.log2(6/(Pu*n**4))+.5:0;a.roughness.value=n,a.lodBias.value=l,this._renderCube(t,s,this._ggxMaterial)}}}_blur(t,i){this._blurMaterial===null&&(this._blurMaterial=cp());let r=this._blurMaterial,a=r.uniforms,o=this._sourceTarget;a.sigma.value=Math.min(i,Math.PI)/Math.SQRT2,a.envMap.value=o.texture,this._renderCube(t,0,r),a.envMap.value=t.texture,this._renderCube(o,0,r)}};function Ch(e,t,i){return new Sl(e,{minFilter:Xa,generateMipmaps:t,type:Rr,colorSpace:Ns,depthBuffer:i})}function Do(e,t,i){return new $i({name:e,uniforms:t,vertexShader:ot.cube_vert,fragmentShader:i,side:1,blending:0,depthTest:!1,depthWrite:!1})}function lp(){return Do("PMREMGGXConvolution",{envMap:{value:null},roughness:{value:0},lodBias:{value:0}},`

		#define GGX_SAMPLES ${Pu}u

		varying vec3 vWorldDirection;

		uniform samplerCube envMap;
		uniform float roughness;
		uniform float lodBias;

		#include <common>

		// Van der Corput radical inverse
		float radicalInverse_VdC( uint bits ) {

			bits = ( bits << 16u ) | ( bits >> 16u );
			bits = ( ( bits & 0x55555555u ) << 1u ) | ( ( bits & 0xAAAAAAAAu ) >> 1u );
			bits = ( ( bits & 0x33333333u ) << 2u ) | ( ( bits & 0xCCCCCCCCu ) >> 2u );
			bits = ( ( bits & 0x0F0F0F0Fu ) << 4u ) | ( ( bits & 0xF0F0F0F0u ) >> 4u );
			bits = ( ( bits & 0x00FF00FFu ) << 8u ) | ( ( bits & 0xFF00FF00u ) >> 8u );
			return float( bits ) * 2.3283064365386963e-10; // / 0x100000000

		}

		// Hammersley sequence
		vec2 hammersley( uint i, uint N ) {

			return vec2( float( i ) / float( N ), radicalInverse_VdC( i ) );

		}

		void main() {

			vec3 N = normalize( vWorldDirection );

			// For very low roughness, just sample the environment directly
			if ( roughness < 0.001 ) {

				gl_FragColor = vec4( textureLod( envMap, N, 0.0 ).rgb, 1.0 );
				return;

			}

			float alpha = roughness * roughness;
			float alpha2 = alpha * alpha;

			// Tangent space basis for VNDF sampling
			vec3 up = abs( N.z ) < 0.999 ? vec3( 0.0, 0.0, 1.0 ) : vec3( 1.0, 0.0, 0.0 );
			vec3 tangent = normalize( cross( up, N ) );
			vec3 bitangent = cross( N, tangent );

			vec3 prefilteredColor = vec3( 0.0 );
			float totalWeight = 0.0;

			for ( uint i = 0u; i < GGX_SAMPLES; i ++ ) {

				vec2 Xi = hammersley( i, GGX_SAMPLES );

				// With V = N, sample the reflected direction directly.
				float invQ = 1.0 / ( 1.0 - Xi.x + alpha2 * Xi.x );
				float NdotL = ( 1.0 - Xi.x - alpha2 * Xi.x ) * invQ;

				if ( NdotL > 0.0 ) {

					float phi = 2.0 * PI * Xi.y;
					float sinTheta = 2.0 * alpha * sqrt( Xi.x * ( 1.0 - Xi.x ) ) * invQ;
					vec3 L = N * NdotL + ( tangent * cos( phi ) + bitangent * sin( phi ) ) * sinTheta;

					// Match the source mip to the sample's solid angle. The terms
					// independent of the sample are precomputed in lodBias.
					float d = alpha2 * invQ;
					float lod = max( log2( d ) + lodBias, 0.0 );

					// Weight by NdotL for the split-sum approximation
					prefilteredColor += textureLod( envMap, L, lod ).rgb * NdotL;
					totalWeight += NdotL;

				}

			}

			gl_FragColor = vec4( prefilteredColor / totalWeight, 1.0 );

		}
	`)}function hp(){return Do("PMREMGGXIntegration",{envMap:{value:null},roughness:{value:0},sourceLod:{value:0},sourceSize:{value:Lu}},`

		varying vec3 vWorldDirection;

		uniform samplerCube envMap;
		uniform float roughness;
		uniform float sourceLod;
		uniform int sourceSize;

		void main() {

			vec3 N = normalize( vWorldDirection );

			float alpha = roughness * roughness;
			float alpha2 = alpha * alpha;

			float texelSize = 2.0 / float( sourceSize );

			vec3 prefilteredColor = vec3( 0.0 );
			float totalWeight = 0.0;

			// Pair opposite texels: only the one in N's hemisphere contributes.
			for ( int face = 0; face < 3; face ++ ) {

				for ( int y = 0; y < sourceSize; y ++ ) {

					for ( int x = 0; x < sourceSize; x ++ ) {

						vec2 uv = ( vec2( x, y ) + 0.5 ) * texelSize - 1.0;
						vec3 texelDirection = face == 0 ? vec3( 1.0, uv ) : ( face == 1 ? vec3( uv.x, 1.0, uv.y ) : vec3( uv, 1.0 ) );

						float invDistance = inversesqrt( 1.0 + dot( uv, uv ) );
						float NdotL = dot( N, texelDirection );
						texelDirection *= NdotL < 0.0 ? - 1.0 : 1.0;
						NdotL = abs( NdotL ) * invDistance;

						// With V = N, NdotH squared is ( 1 + NdotL ) / 2. Common factors
						// in the GGX distribution and texel solid angle cancel when normalized.
						float d = 1.0 + alpha2 + ( alpha2 - 1.0 ) * NdotL;
						float weight = NdotL * invDistance * invDistance * invDistance / ( d * d );

						prefilteredColor += textureLod( envMap, texelDirection, sourceLod ).rgb * weight;
						totalWeight += weight;

					}

				}

			}

			gl_FragColor = vec4( prefilteredColor / totalWeight, 1.0 );

		}
	`)}function cp(){return Do("PMREMSphericalGaussianBlur",{envMap:{value:null},sigma:{value:0}},`

		#define SAMPLES ${ap}
		#define GOLDEN_ANGLE 2.39996322973

		varying vec3 vWorldDirection;

		uniform samplerCube envMap;
		uniform float sigma;

		#include <common>

		void main() {

			vec3 outputDirection = normalize( vWorldDirection );

			vec3 up = abs( outputDirection.z ) < 0.999 ? vec3( 0.0, 0.0, 1.0 ) : vec3( 1.0, 0.0, 0.0 );
			vec3 tangent = normalize( cross( up, outputDirection ) );
			vec3 bitangent = cross( outputDirection, tangent );

			// Truncate the kernel at three standard deviations or at the antipode.
			float thetaMax = min( 3.0 * sigma, PI );
			float truncation = 1.0 - exp( - 0.5 * thetaMax * thetaMax / ( sigma * sigma ) );

			vec3 accumColor = vec3( 0.0 );
			float accumWeight = 0.0;

			for ( int i = 0; i < SAMPLES; i ++ ) {

				// Stratified inverse-CDF sampling of the Gaussian, placed on a golden-angle spiral.
				float stratum = ( float( i ) + 0.5 ) / float( SAMPLES );
				float theta = sigma * sqrt( - 2.0 * log( 1.0 - stratum * truncation ) );
				float phi = float( i ) * GOLDEN_ANGLE;

				vec3 offset = cos( phi ) * tangent + sin( phi ) * bitangent;
				vec3 sampleDirection = cos( theta ) * outputDirection + sin( theta ) * offset;

				// Correct the planar sample density to solid angle.
				float weight = sin( theta ) / theta;

				accumColor += weight * textureLod( envMap, sampleDirection, 0.0 ).rgb;
				accumWeight += weight;

			}

			gl_FragColor = vec4( accumColor / accumWeight, 1.0 );

		}
	`)}function Ph(){return Do("PMREMEquirectangularToCubemap",{envMap:{value:null}},`

		varying vec3 vWorldDirection;

		uniform sampler2D envMap;

		#include <common>

		void main() {

			// Average four subpixel samples to preserve small, bright features.
			vec3 direction = normalize( vWorldDirection );
			vec3 dx = dFdx( direction ) * 0.25;
			vec3 dy = dFdy( direction ) * 0.25;

			vec3 color = textureLod( envMap, equirectUv( normalize( direction - dx - dy ) ), 0.0 ).rgb;
			color += textureLod( envMap, equirectUv( normalize( direction + dx - dy ) ), 0.0 ).rgb;
			color += textureLod( envMap, equirectUv( normalize( direction - dx + dy ) ), 0.0 ).rgb;
			color += textureLod( envMap, equirectUv( normalize( direction + dx + dy ) ), 0.0 ).rgb;
			gl_FragColor = vec4( color * 0.25, 1.0 );

		}
	`)}function Lh(){return Do("PMREMCubemapToCubemap",{envMap:{value:null},flipEnvMap:{value:-1}},`

		varying vec3 vWorldDirection;

		uniform samplerCube envMap;
		uniform float flipEnvMap;

		void main() {

			gl_FragColor = vec4( textureCube( envMap, vec3( flipEnvMap * vWorldDirection.x, vWorldDirection.yz ) ).rgb, 1.0 );

		}
	`)}function up(e){let t=new WeakMap,i=new WeakMap,r=null;function a(d,f=!1){return d==null?null:f?s(d):o(d)}function o(d){if(d&&d.isTexture){let f=d.mapping;if(f===303||f===304){if(t.has(d)){let p=t.get(d).texture;return n(p,d.mapping)}{let p=d.image;if(p&&p.height>0){let x=new Sl(p.height);return x.fromEquirectangularTexture(e,d),t.set(d,x),d.addEventListener("dispose",h),n(x.texture,d.mapping)}return null}}}return d}function s(d){if(d&&d.isTexture&&d.isPMREMTexture!==!0){let f=d.mapping,p=f===303||f===304,x=f===301||f===302;if(p||x){let g=i.get(d),m=g===void 0?0:g.texture.pmremVersion;if(d.isRenderTargetTexture&&d.pmremVersion!==m)return r===null&&(r=new Rh(e)),g=p?r.fromEquirectangular(d,g):r.fromCubemap(d,g),g.texture.pmremVersion=d.pmremVersion,i.set(d,g),g.texture;if(g!==void 0)return g.texture;{let y=d.image;return p&&y&&y.height>0||x&&y&&l(y)?(r===null&&(r=new Rh(e)),g=p?r.fromEquirectangular(d):r.fromCubemap(d),g.texture.pmremVersion=d.pmremVersion,i.set(d,g),d.addEventListener("dispose",c),g.texture):null}}}return d}function n(d,f){return f===303?d.mapping=301:f===304&&(d.mapping=302),d}function l(d){let f=0;for(let p=0;p<6;p++)d[p]!==void 0&&f++;return f===6}function h(d){let f=d.target;f.removeEventListener("dispose",h);let p=t.get(f);p!==void 0&&(t.delete(f),p.dispose())}function c(d){let f=d.target;f.removeEventListener("dispose",c);let p=i.get(f);p!==void 0&&(i.delete(f),p.dispose())}function u(){t=new WeakMap,i=new WeakMap,r!==null&&(r.dispose(),r=null)}return{get:a,dispose:u}}function dp(e){let t={};function i(r){if(t[r]!==void 0)return t[r];let a=e.getExtension(r);return t[r]=a,a}return{has:function(r){return i(r)!==null},init:function(){i("EXT_color_buffer_float"),i("WEBGL_clip_cull_distance"),i("OES_texture_float_linear"),i("EXT_color_buffer_half_float"),i("WEBGL_multisampled_render_to_texture"),i("WEBGL_render_shared_exponent")},get:function(r){let a=i(r);return a===null&&ys("WebGLRenderer: "+r+" extension not supported."),a}}}function fp(e,t,i,r){let a={},o=new WeakMap,s=new FinalizationRegistry(p=>delete a[p]);function n(p){l(p.target)}function l(p){p.index!==null&&t.remove(p.index);for(let g in p.attributes)t.remove(p.attributes[g]);p.removeEventListener("dispose",n),delete a[p.id],s.unregister(p);let x=o.get(p);x&&(t.remove(x),o.delete(p)),r.releaseStatesOfGeometry(p),p.isInstancedBufferGeometry===!0&&delete p._maxInstanceCount,i.memory.geometries--}function h(p,x){return a[x.id]===void 0&&(x.addEventListener("dispose",n),a[x.id]=new WeakRef(x),s.register(x,x.id,x),i.memory.geometries++),x}function c(p){let x=p.attributes;for(let g in x)t.update(x[g],e.ARRAY_BUFFER)}function u(p){let x=p.index,g=p.attributes.position,m=0;if(g===void 0)return;let y=x===null?g.array.length/3-1:x.array.length,M=new(g.count>=65535?qc:Xc)(Math.ceil(y/3)*6,1),b=M.array;if(x!==null){let T=x.array;m=x.version;for(let v=0,_=0;v<y;v+=3){let A=T[v+0],C=T[v+1],R=T[v+2];b[_++]=A,b[_++]=C,b[_++]=C,b[_++]=R,b[_++]=R,b[_++]=A}}else{m=g.version;for(let T=0,v=0;T<y;T+=3){let _=T+0,A=T+1,C=T+2;b[v++]=_,b[v++]=A,b[v++]=A,b[v++]=C,b[v++]=C,b[v++]=_}}M.version=m;let E=o.get(p);E&&t.remove(E),o.set(p,M)}function d(p){let x=o.get(p);if(x){let g=p.index;g!==null&&x.version<g.version&&u(p)}else u(p);return o.get(p)}function f(){for(let p in a){let x=a[p].deref();x!==void 0&&l(x)}}return{get:h,update:c,getWireframeAttribute:d,dispose:f}}function pp(e,t,i){let r;function a(u){r=u}let o,s;function n(u){o=u.type,s=u.bytesPerElement}function l(u,d){e.drawElements(r,d,o,u*s),i.update(d,r,1)}function h(u,d,f){f!==0&&(e.drawElementsInstanced(r,d,o,u*s,f),i.update(d,r,f))}function c(u,d,f){if(f===0)return;t.get("WEBGL_multi_draw").multiDrawElementsWEBGL(r,d,0,o,u,0,f);let p=0;for(let x=0;x<f;x++)p+=d[x];i.update(p,r,1)}this.setMode=a,this.setIndex=n,this.render=l,this.renderInstances=h,this.renderMultiDraw=c}function mp(e){let t={geometries:0,textures:0},i={frame:0,calls:0,triangles:0,points:0,lines:0};function r(o,s,n){switch(i.calls++,s){case e.TRIANGLES:i.triangles+=o/3*n;break;case e.LINES:i.lines+=o/2*n;break;case e.LINE_STRIP:i.lines+=n*(o-1);break;case e.LINE_LOOP:i.lines+=n*o;break;case e.POINTS:i.points+=n*o;break;default:Qe("WebGLInfo: Unknown draw mode:",s)}}function a(){i.calls=0,i.triangles=0,i.points=0,i.lines=0}return{memory:t,render:i,programs:null,autoReset:!0,reset:a,update:r}}function gp(e,t,i){let r=new WeakMap,a=new Ft;function o(s,n,l){let h=s.morphTargetInfluences,c=n.morphAttributes.position||n.morphAttributes.normal||n.morphAttributes.color,u=c===void 0?0:c.length,d=r.get(n);if(d===void 0||d.count!==u){let C=function(){_.dispose(),r.delete(n),n.removeEventListener("dispose",C)};var f=C;d!==void 0&&d.texture.dispose();let p=n.morphAttributes.position!==void 0,x=n.morphAttributes.normal!==void 0,g=n.morphAttributes.color!==void 0,m=n.morphAttributes.position||[],y=n.morphAttributes.normal||[],M=n.morphAttributes.color||[],b=0;p===!0&&(b=1),x===!0&&(b=2),g===!0&&(b=3);let E=n.attributes.position.count*b,T=1;E>t.maxTextureSize&&(T=Math.ceil(E/t.maxTextureSize),E=t.maxTextureSize);let v=new Float32Array(E*T*4*u),_=new Bc(v,E,T,u);_.type=Co,_.needsUpdate=!0;let A=b*4;for(let R=0;R<u;R++){let L=m[R],G=y[R],P=M[R],z=E*T*4*R;for(let H=0;H<L.count;H++){let D=H*A;p===!0&&(a.fromBufferAttribute(L,H),v[z+D+0]=a.x,v[z+D+1]=a.y,v[z+D+2]=a.z,v[z+D+3]=0),x===!0&&(a.fromBufferAttribute(G,H),v[z+D+4]=a.x,v[z+D+5]=a.y,v[z+D+6]=a.z,v[z+D+7]=0),g===!0&&(a.fromBufferAttribute(P,H),v[z+D+8]=a.x,v[z+D+9]=a.y,v[z+D+10]=a.z,v[z+D+11]=P.itemSize===4?a.w:1)}}d={count:u,texture:_,size:new ge(E,T)},r.set(n,d),n.addEventListener("dispose",C)}if(s.isInstancedMesh===!0&&s.morphTexture!==null)l.getUniforms().setValue(e,"morphTexture",s.morphTexture,i);else{let p=0;for(let g=0;g<h.length;g++)p+=h[g];let x=n.morphTargetsRelative?1:1-p;l.getUniforms().setValue(e,"morphTargetBaseInfluence",x),l.getUniforms().setValue(e,"morphTargetInfluences",h)}l.getUniforms().setValue(e,"morphTargetsTexture",d.texture,i),l.getUniforms().setValue(e,"morphTargetsTextureSize",d.size)}return{update:o}}function vp(e,t,i,r,a){let o=new WeakMap;function s(h){let c=a.render.frame,u=h.geometry,d=t.get(h,u);if(o.get(d)!==c&&(t.update(d),o.set(d,c)),h.isInstancedMesh&&(h.hasEventListener("dispose",l)===!1&&h.addEventListener("dispose",l),o.get(h)!==c&&(i.update(h.instanceMatrix,e.ARRAY_BUFFER),h.instanceColor!==null&&i.update(h.instanceColor,e.ARRAY_BUFFER),o.set(h,c))),h.isSkinnedMesh){let f=h.skeleton;o.get(f)!==c&&(f.update(),o.set(f,c))}return d}function n(){o=new WeakMap}function l(h){let c=h.target;c.removeEventListener("dispose",l),r.releaseStatesOfObject(c),i.remove(c.instanceMatrix),c.instanceColor!==null&&i.remove(c.instanceColor)}return{update:s,dispose:n}}var xp={1:"LINEAR_TONE_MAPPING",2:"REINHARD_TONE_MAPPING",3:"CINEON_TONE_MAPPING",4:"ACES_FILMIC_TONE_MAPPING",6:"AGX_TONE_MAPPING",7:"NEUTRAL_TONE_MAPPING",5:"CUSTOM_TONE_MAPPING"};function _p(e,t,i,r,a,o){let s=new gi(t,i,{type:e,depthBuffer:a,stencilBuffer:o,samples:r?4:0,storeMultisampledDepthBuffer:!1,storeMultisampledStencilBuffer:!1,resolveDepthBuffer:!1,resolveStencilBuffer:!1}),n=null,l=null,h=null,c=new ti;c.setAttribute("position",new wt([-1,3,0,-1,-1,0,3,-1,0],3)),c.setAttribute("uv",new wt([0,2,0,0,2,0],2));let u=new Tu({uniforms:{tDiffuse:{value:null}},vertexShader:`
			precision highp float;

			uniform mat4 modelViewMatrix;
			uniform mat4 projectionMatrix;

			attribute vec3 position;
			attribute vec2 uv;

			varying vec2 vUv;

			void main() {
				vUv = uv;
				gl_Position = projectionMatrix * modelViewMatrix * vec4( position, 1.0 );
			}`,fragmentShader:`
			precision highp float;

			uniform sampler2D tDiffuse;

			varying vec2 vUv;

			#include <tonemapping_pars_fragment>
			#include <colorspace_pars_fragment>

			void main() {
				gl_FragColor = texture2D( tDiffuse, vUv );

				#ifdef LINEAR_TONE_MAPPING
					gl_FragColor.rgb = LinearToneMapping( gl_FragColor.rgb );
				#elif defined( REINHARD_TONE_MAPPING )
					gl_FragColor.rgb = ReinhardToneMapping( gl_FragColor.rgb );
				#elif defined( CINEON_TONE_MAPPING )
					gl_FragColor.rgb = CineonToneMapping( gl_FragColor.rgb );
				#elif defined( ACES_FILMIC_TONE_MAPPING )
					gl_FragColor.rgb = ACESFilmicToneMapping( gl_FragColor.rgb );
				#elif defined( AGX_TONE_MAPPING )
					gl_FragColor.rgb = AgXToneMapping( gl_FragColor.rgb );
				#elif defined( NEUTRAL_TONE_MAPPING )
					gl_FragColor.rgb = NeutralToneMapping( gl_FragColor.rgb );
				#elif defined( CUSTOM_TONE_MAPPING )
					gl_FragColor.rgb = CustomToneMapping( gl_FragColor.rgb );
				#endif

				#ifdef SRGB_TRANSFER
					gl_FragColor = sRGBTransferOETF( gl_FragColor );
				#endif
			}`,depthTest:!1,depthWrite:!1}),d=new Nt(c,u),f=new el(-1,1,1,-1,0,1),p=null,x=null,g=!1,m,y=null,M=[],b=!1;this.setSize=function(E,T){s.setSize(E,T),n!==null&&n.setSize(E,T),l!==null&&l.setSize(E,T);for(let v=0;v<M.length;v++){let _=M[v];_.setSize&&_.setSize(E,T)}},this.setEffects=function(E){M=E,b=M.length>0&&M[0].isRenderPass===!0;let T=s.width,v=s.height;M.length>0&&n===null&&(n=new gi(T,v,{type:Rr,depthBuffer:!1,stencilBuffer:!1}),l=new gi(T,v,{type:Rr,depthBuffer:!1,stencilBuffer:!1}));let _=a===!0&&M.some(A=>A.needsDepthTexture===!0);_!==(s.depthTexture!==null)&&(_===!0&&h===null&&(h=new ra(T,v),o===!0&&(h.format=Cs,h.type=vl)),s.dispose(),s.depthTexture=_?h:null,s.resolveDepthBuffer=_,s.storeMultisampledDepthBuffer=_);for(let A=0;A<M.length;A++){let C=M[A];C.setSize&&C.setSize(T,v),_&&C.needsDepthTexture===!0&&C.setDepthTexture&&C.setDepthTexture(h)}},this.begin=function(E,T){if(g||E.toneMapping===0&&M.length===0)return!1;if(y=T,T!==null){let v=T.width,_=T.height;(s.width!==v||s.height!==_)&&this.setSize(v,_)}return b===!1&&E.setRenderTarget(s),m=E.toneMapping,E.toneMapping=0,!0},this.hasRenderPass=function(){return b},this.end=function(E,T){E.toneMapping=m,g=!0;let v=s,_=n;for(let A=0;A<M.length;A++){let C=M[A];C.enabled!==!1&&(C.render(E,_,v,T),C.needsSwap!==!1&&(v=_,_=_===n?l:n))}if(p!==E.outputColorSpace||x!==E.toneMapping){p=E.outputColorSpace,x=E.toneMapping,u.defines={},ft.getTransfer(p)==="srgb"&&(u.defines.SRGB_TRANSFER="");let A=xp[x];A&&(u.defines[A]=""),u.needsUpdate=!0}u.uniforms.tDiffuse.value=v.texture,E.setRenderTarget(y),E.render(d,f),y=null,g=!1},this.isCompositing=function(){return g},this.dispose=function(){s.dispose(),n!==null&&n.dispose(),l!==null&&l.dispose(),c.dispose(),u.dispose()}}var Iu=new zi,Du=new ra(1,1),Uu=new Jc(1),Ou=new Bc,zu=new v0,Fu=new Zc,Nh=[],Ih=[],Dh=new Float32Array(16),Uh=new Float32Array(9),Oh=new Float32Array(4);function ja(e,t,i){let r=e[0];if(r<=0||r>0)return e;let a=t*i,o=Nh[a];if(o===void 0&&(o=new Float32Array(a),Nh[a]=o),t!==0){r.toArray(o,0);for(let s=1,n=0;s!==t;++s)n+=i,e[s].toArray(o,n)}return o}function Jt(e,t){if(e.length!==t.length)return!1;for(let i=0,r=e.length;i<r;i++)if(e[i]!==t[i])return!1;return!0}function Kt(e,t){for(let i=0,r=t.length;i<r;i++)e[i]=t[i]}function Js(e,t){let i=Ih[t];i===void 0&&(i=new Int32Array(t),Ih[t]=i);for(let r=0;r!==t;++r)i[r]=e.allocateTextureUnit();return i}function yp(e,t){let i=this.cache;i[0]!==t&&(e.uniform1f(this.addr,t),i[0]=t)}function bp(e,t){let i=this.cache;if(t.x!==void 0)(i[0]!==t.x||i[1]!==t.y)&&(e.uniform2f(this.addr,t.x,t.y),i[0]=t.x,i[1]=t.y);else{if(Jt(i,t))return;e.uniform2fv(this.addr,t),Kt(i,t)}}function Mp(e,t){let i=this.cache;if(t.x!==void 0)(i[0]!==t.x||i[1]!==t.y||i[2]!==t.z)&&(e.uniform3f(this.addr,t.x,t.y,t.z),i[0]=t.x,i[1]=t.y,i[2]=t.z);else if(t.r!==void 0)(i[0]!==t.r||i[1]!==t.g||i[2]!==t.b)&&(e.uniform3f(this.addr,t.r,t.g,t.b),i[0]=t.r,i[1]=t.g,i[2]=t.b);else{if(Jt(i,t))return;e.uniform3fv(this.addr,t),Kt(i,t)}}function Sp(e,t){let i=this.cache;if(t.x!==void 0)(i[0]!==t.x||i[1]!==t.y||i[2]!==t.z||i[3]!==t.w)&&(e.uniform4f(this.addr,t.x,t.y,t.z,t.w),i[0]=t.x,i[1]=t.y,i[2]=t.z,i[3]=t.w);else{if(Jt(i,t))return;e.uniform4fv(this.addr,t),Kt(i,t)}}function wp(e,t){let i=this.cache,r=t.elements;if(r===void 0){if(Jt(i,t))return;e.uniformMatrix2fv(this.addr,!1,t),Kt(i,t)}else{if(Jt(i,r))return;Oh.set(r),e.uniformMatrix2fv(this.addr,!1,Oh),Kt(i,r)}}function Tp(e,t){let i=this.cache,r=t.elements;if(r===void 0){if(Jt(i,t))return;e.uniformMatrix3fv(this.addr,!1,t),Kt(i,t)}else{if(Jt(i,r))return;Uh.set(r),e.uniformMatrix3fv(this.addr,!1,Uh),Kt(i,r)}}function Ep(e,t){let i=this.cache,r=t.elements;if(r===void 0){if(Jt(i,t))return;e.uniformMatrix4fv(this.addr,!1,t),Kt(i,t)}else{if(Jt(i,r))return;Dh.set(r),e.uniformMatrix4fv(this.addr,!1,Dh),Kt(i,r)}}function Ap(e,t){let i=this.cache;i[0]!==t&&(e.uniform1i(this.addr,t),i[0]=t)}function Rp(e,t){let i=this.cache;if(t.x!==void 0)(i[0]!==t.x||i[1]!==t.y)&&(e.uniform2i(this.addr,t.x,t.y),i[0]=t.x,i[1]=t.y);else{if(Jt(i,t))return;e.uniform2iv(this.addr,t),Kt(i,t)}}function Cp(e,t){let i=this.cache;if(t.x!==void 0)(i[0]!==t.x||i[1]!==t.y||i[2]!==t.z)&&(e.uniform3i(this.addr,t.x,t.y,t.z),i[0]=t.x,i[1]=t.y,i[2]=t.z);else{if(Jt(i,t))return;e.uniform3iv(this.addr,t),Kt(i,t)}}function Pp(e,t){let i=this.cache;if(t.x!==void 0)(i[0]!==t.x||i[1]!==t.y||i[2]!==t.z||i[3]!==t.w)&&(e.uniform4i(this.addr,t.x,t.y,t.z,t.w),i[0]=t.x,i[1]=t.y,i[2]=t.z,i[3]=t.w);else{if(Jt(i,t))return;e.uniform4iv(this.addr,t),Kt(i,t)}}function Lp(e,t){let i=this.cache;i[0]!==t&&(e.uniform1ui(this.addr,t),i[0]=t)}function Np(e,t){let i=this.cache;if(t.x!==void 0)(i[0]!==t.x||i[1]!==t.y)&&(e.uniform2ui(this.addr,t.x,t.y),i[0]=t.x,i[1]=t.y);else{if(Jt(i,t))return;e.uniform2uiv(this.addr,t),Kt(i,t)}}function Ip(e,t){let i=this.cache;if(t.x!==void 0)(i[0]!==t.x||i[1]!==t.y||i[2]!==t.z)&&(e.uniform3ui(this.addr,t.x,t.y,t.z),i[0]=t.x,i[1]=t.y,i[2]=t.z);else{if(Jt(i,t))return;e.uniform3uiv(this.addr,t),Kt(i,t)}}function Dp(e,t){let i=this.cache;if(t.x!==void 0)(i[0]!==t.x||i[1]!==t.y||i[2]!==t.z||i[3]!==t.w)&&(e.uniform4ui(this.addr,t.x,t.y,t.z,t.w),i[0]=t.x,i[1]=t.y,i[2]=t.z,i[3]=t.w);else{if(Jt(i,t))return;e.uniform4uiv(this.addr,t),Kt(i,t)}}function Ks(e){return e.version===0&&(e.compareFunction=519,e.needsUpdate=!0),e}function Up(e,t,i){let r=this.cache,a=i.allocateTextureUnit();r[0]!==a&&(e.uniform1i(this.addr,a),r[0]=a);let o;o=this.type===e.SAMPLER_2D_SHADOW?Ks(Du):Iu,i.setTexture2D(t||o,a)}function Op(e,t,i){let r=this.cache,a=i.allocateTextureUnit();r[0]!==a&&(e.uniform1i(this.addr,a),r[0]=a),i.setTexture3D(t||zu,a)}function zp(e,t,i){let r=this.cache,a=i.allocateTextureUnit();r[0]!==a&&(e.uniform1i(this.addr,a),r[0]=a);let o;o=this.type===e.SAMPLER_CUBE_SHADOW?Ks(Uu):Fu,i.setTextureCube(t||o,a)}function Fp(e,t,i){let r=this.cache,a=i.allocateTextureUnit();r[0]!==a&&(e.uniform1i(this.addr,a),r[0]=a),i.setTexture2DArray(t||Ou,a)}function Bp(e){switch(e){case 5126:return yp;case 35664:return bp;case 35665:return Mp;case 35666:return Sp;case 35674:return wp;case 35675:return Tp;case 35676:return Ep;case 5124:case 35670:return Ap;case 35667:case 35671:return Rp;case 35668:case 35672:return Cp;case 35669:case 35673:return Pp;case 5125:return Lp;case 36294:return Np;case 36295:return Ip;case 36296:return Dp;case 35678:case 36198:case 36298:case 36306:case 35682:return Up;case 35679:case 36299:case 36307:return Op;case 35680:case 36300:case 36308:case 36293:return zp;case 36289:case 36303:case 36311:case 36292:return Fp}}function kp(e,t){e.uniform1fv(this.addr,t)}function Vp(e,t){let i=ja(t,this.size,2);e.uniform2fv(this.addr,i)}function Gp(e,t){let i=ja(t,this.size,3);e.uniform3fv(this.addr,i)}function Hp(e,t){let i=ja(t,this.size,4);e.uniform4fv(this.addr,i)}function Wp(e,t){let i=ja(t,this.size,4);e.uniformMatrix2fv(this.addr,!1,i)}function Xp(e,t){let i=ja(t,this.size,9);e.uniformMatrix3fv(this.addr,!1,i)}function qp(e,t){let i=ja(t,this.size,16);e.uniformMatrix4fv(this.addr,!1,i)}function jp(e,t){e.uniform1iv(this.addr,t)}function Yp(e,t){e.uniform2iv(this.addr,t)}function Zp(e,t){e.uniform3iv(this.addr,t)}function Jp(e,t){e.uniform4iv(this.addr,t)}function Kp(e,t){e.uniform1uiv(this.addr,t)}function $p(e,t){e.uniform2uiv(this.addr,t)}function Qp(e,t){e.uniform3uiv(this.addr,t)}function em(e,t){e.uniform4uiv(this.addr,t)}function tm(e,t,i){let r=this.cache,a=t.length,o=Js(i,a);Jt(r,o)||(e.uniform1iv(this.addr,o),Kt(r,o));let s;s=this.type===e.SAMPLER_2D_SHADOW?Ks(Du):Iu;for(let n=0;n!==a;++n)i.setTexture2D(t[n]||s,o[n])}function im(e,t,i){let r=this.cache,a=t.length,o=Js(i,a);Jt(r,o)||(e.uniform1iv(this.addr,o),Kt(r,o));for(let s=0;s!==a;++s)i.setTexture3D(t[s]||zu,o[s])}function rm(e,t,i){let r=this.cache,a=t.length,o=Js(i,a);Jt(r,o)||(e.uniform1iv(this.addr,o),Kt(r,o));let s;s=this.type===e.SAMPLER_CUBE_SHADOW?Ks(Uu):Fu;for(let n=0;n!==a;++n)i.setTextureCube(t[n]||s,o[n])}function am(e,t,i){let r=this.cache,a=t.length,o=Js(i,a);Jt(r,o)||(e.uniform1iv(this.addr,o),Kt(r,o));for(let s=0;s!==a;++s)i.setTexture2DArray(t[s]||Ou,o[s])}function om(e){switch(e){case 5126:return kp;case 35664:return Vp;case 35665:return Gp;case 35666:return Hp;case 35674:return Wp;case 35675:return Xp;case 35676:return qp;case 5124:case 35670:return jp;case 35667:case 35671:return Yp;case 35668:case 35672:return Zp;case 35669:case 35673:return Jp;case 5125:return Kp;case 36294:return $p;case 36295:return Qp;case 36296:return em;case 35678:case 36198:case 36298:case 36306:case 35682:return tm;case 35679:case 36299:case 36307:return im;case 35680:case 36300:case 36308:case 36293:return rm;case 36289:case 36303:case 36311:case 36292:return am}}var sm=class{constructor(e,t,i){this.id=e,this.addr=i,this.cache=[],this.type=t.type,this.setValue=Bp(t.type)}},nm=class{constructor(e,t,i){this.id=e,this.addr=i,this.cache=[],this.type=t.type,this.size=t.size,this.setValue=om(t.type)}},lm=class{constructor(e){this.id=e,this.seq=[],this.map={}}setValue(e,t,i){let r=this.seq;for(let a=0,o=r.length;a!==o;++a){let s=r[a];s.setValue(e,t[s.id],i)}}},Ln=/(\w+)(\])?(\[|\.)?/g;function zh(e,t){e.seq.push(t),e.map[t.id]=t}function hm(e,t,i){let r=e.name,a=r.length;for(Ln.lastIndex=0;;){let o=Ln.exec(r),s=Ln.lastIndex,n=o[1],l=o[2]==="]",h=o[3];if(l&&(n|=0),h===void 0||h==="["&&s+2===a){zh(i,h===void 0?new sm(n,e,t):new nm(n,e,t));break}{let c=i.map[n];c===void 0&&(c=new lm(n),zh(i,c)),i=c}}}var ws=class{constructor(e,t){this.seq=[],this.map={};let i=e.getProgramParameter(t,e.ACTIVE_UNIFORMS);for(let o=0;o<i;++o){let s=e.getActiveUniform(t,o);hm(s,e.getUniformLocation(t,s.name),this)}let r=[],a=[];for(let o of this.seq)o.type===e.SAMPLER_2D_SHADOW||o.type===e.SAMPLER_CUBE_SHADOW||o.type===e.SAMPLER_2D_ARRAY_SHADOW?r.push(o):a.push(o);r.length>0&&(this.seq=r.concat(a))}setValue(e,t,i,r){let a=this.map[t];a!==void 0&&a.setValue(e,i,r)}setOptional(e,t,i){let r=t[i];r!==void 0&&this.setValue(e,i,r)}static upload(e,t,i,r){for(let a=0,o=t.length;a!==o;++a){let s=t[a],n=i[s.id];n.needsUpdate!==!1&&s.setValue(e,n.value,r)}}static seqWithValue(e,t){let i=[];for(let r=0,a=e.length;r!==a;++r){let o=e[r];o.id in t&&i.push(o)}return i}};function Fh(e,t,i){let r=e.createShader(t);return e.shaderSource(r,i),e.compileShader(r),r}var cm=37297,um=0;function dm(e,t){let i=e.split(`
`),r=[],a=Math.max(t-6,0),o=Math.min(t+6,i.length);for(let s=a;s<o;s++){let n=s+1;r.push(`${n===t?">":" "} ${n}: ${i[s]}`)}return r.join(`
`)}var Bh=new it;function fm(e){ft._getMatrix(Bh,ft.workingColorSpace,e);let t=`mat3( ${Bh.elements.map(i=>i.toFixed(4))} )`;switch(ft.getTransfer(e)){case Is:return[t,"LinearTransferOETF"];case Ds:return[t,"sRGBTransferOETF"];default:return Je("WebGLProgram: Unsupported color space: ",e),[t,"LinearTransferOETF"]}}function kh(e,t,i){let r=e.getShaderParameter(t,e.COMPILE_STATUS),a=(e.getShaderInfoLog(t)||"").trim();if(r&&a==="")return"";let o=/ERROR: 0:(\d+)/.exec(a);if(o){let s=parseInt(o[1]);return i.toUpperCase()+`

`+a+`

`+dm(e.getShaderSource(t),s)}return a}function pm(e,t){let i=fm(t);return[`vec4 ${e}( vec4 value ) {`,`	return ${i[1]}( vec4( ${i[0]} * value.rgb, value.a ) );`,"}"].join(`
`)}var mm={1:"Linear",2:"Reinhard",3:"Cineon",4:"ACESFilmic",6:"AgX",7:"Neutral",5:"Custom"};function gm(e,t){let i=mm[t];return i===void 0?(Je("WebGLProgram: Unsupported toneMapping:",t),"vec3 "+e+"( vec3 color ) { return LinearToneMapping( color ); }"):"vec3 "+e+"( vec3 color ) { return "+i+"ToneMapping( color ); }"}var ls=new I;function vm(){return ft.getLuminanceCoefficients(ls),["float luminance( const in vec3 rgb ) {",`	const vec3 weights = vec3( ${ls.x.toFixed(4)}, ${ls.y.toFixed(4)}, ${ls.z.toFixed(4)} );`,"	return dot( weights, rgb );","}"].join(`
`)}function xm(e){return[e.extensionClipCullDistance?"#extension GL_ANGLE_clip_cull_distance : require":"",e.extensionMultiDraw?"#extension GL_ANGLE_multi_draw : require":""].filter(co).join(`
`)}function _m(e){let t=[];for(let i in e){let r=e[i];r!==!1&&t.push("#define "+i+" "+r)}return t.join(`
`)}function ym(e,t){let i={},r=e.getProgramParameter(t,e.ACTIVE_ATTRIBUTES);for(let a=0;a<r;a++){let o=e.getActiveAttrib(t,a),s=o.name,n=1;o.type===e.FLOAT_MAT2&&(n=2),o.type===e.FLOAT_MAT3&&(n=3),o.type===e.FLOAT_MAT4&&(n=4),i[s]={type:o.type,location:e.getAttribLocation(t,s),locationSize:n}}return i}function co(e){return e!==""}function Vh(e,t){let i=t.numSpotLightShadows+t.numSpotLightMaps-t.numSpotLightShadowsWithMaps;return e.replace(/NUM_SUN_LIGHTS/g,t.numSunLights).replace(/NUM_DIR_LIGHTS/g,t.numDirLights).replace(/NUM_SPOT_LIGHTS/g,t.numSpotLights).replace(/NUM_SPOT_LIGHT_MAPS/g,t.numSpotLightMaps).replace(/NUM_SPOT_LIGHT_COORDS/g,i).replace(/NUM_RECT_AREA_LIGHTS/g,t.numRectAreaLights).replace(/NUM_POINT_LIGHTS/g,t.numPointLights).replace(/NUM_HEMI_LIGHTS/g,t.numHemiLights).replace(/NUM_SUN_LIGHT_SHADOWS/g,t.numSunLightShadows).replace(/NUM_DIR_LIGHT_SHADOWS/g,t.numDirLightShadows).replace(/NUM_SPOT_LIGHT_SHADOWS_WITH_MAPS/g,t.numSpotLightShadowsWithMaps).replace(/NUM_SPOT_LIGHT_SHADOWS/g,t.numSpotLightShadows).replace(/NUM_POINT_LIGHT_SHADOWS/g,t.numPointLightShadows).replace(/NUM_LIGHT_PROBE_GRIDS/g,t.numLightProbeGrids)}function Gh(e,t){return e.replace(/NUM_CLIPPING_PLANES/g,t.numClippingPlanes).replace(/UNION_CLIPPING_PLANES/g,t.numClippingPlanes-t.numClipIntersection)}var bm=/^[ \t]*#include +<([\w\d./]+)>/gm;function il(e){return e.replace(bm,Mm)}function Mm(e,t){let i=ot[t];if(i===void 0)throw Error("THREE.WebGLProgram: Can not resolve #include <"+t+">");return il(i)}var Sm=/#pragma unroll_loop_start\s+for\s*\(\s*int\s+i\s*=\s*(\d+)\s*;\s*i\s*<\s*(\d+)\s*;\s*i\s*\+\+\s*\)\s*{([\s\S]+?)}\s+#pragma unroll_loop_end/g;function Hh(e){return e.replace(Sm,wm)}function wm(e,t,i,r){let a="";for(let o=parseInt(t);o<parseInt(i);o++)a+=r.replace(/\[\s*i\s*\]/g,"[ "+o+" ]").replace(/UNROLLED_LOOP_INDEX/g,o);return a}function Wh(e){let t=`precision ${e.precision} float;
	precision ${e.precision} int;
	precision ${e.precision} sampler2D;
	precision ${e.precision} samplerCube;
	precision ${e.precision} sampler3D;
	precision ${e.precision} sampler2DArray;
	precision ${e.precision} sampler2DShadow;
	precision ${e.precision} samplerCubeShadow;
	precision ${e.precision} sampler2DArrayShadow;
	precision ${e.precision} isampler2D;
	precision ${e.precision} isampler3D;
	precision ${e.precision} isamplerCube;
	precision ${e.precision} isampler2DArray;
	precision ${e.precision} usampler2D;
	precision ${e.precision} usampler3D;
	precision ${e.precision} usamplerCube;
	precision ${e.precision} usampler2DArray;
	`;return e.precision==="highp"?t+=`
#define HIGH_PRECISION`:e.precision==="mediump"?t+=`
#define MEDIUM_PRECISION`:e.precision==="lowp"&&(t+=`
#define LOW_PRECISION`),t}var Tm={1:"SHADOWMAP_TYPE_PCF",3:"SHADOWMAP_TYPE_VSM"};function Em(e){return Tm[e.shadowMapType]||"SHADOWMAP_TYPE_BASIC"}var Am={302:"ENVMAP_MODE_REFRACTION"};function Rm(e){return e.envMap===!1?"ENVMAP_MODE_REFLECTION":Am[e.envMapMode]||"ENVMAP_MODE_REFLECTION"}var Cm={0:"ENVMAP_BLENDING_MULTIPLY",1:"ENVMAP_BLENDING_MIX",2:"ENVMAP_BLENDING_ADD"};function Pm(e){return e.envMap===!1?"ENVMAP_BLENDING_NONE":Cm[e.combine]||"ENVMAP_BLENDING_NONE"}function Lm(e,t,i,r){let a=e.getContext(),o=i.defines,s=i.vertexShader,n=i.fragmentShader,l=Em(i),h=i.envMapPMREM===!0?"ENVMAP_TYPE_PMREM":"ENVMAP_TYPE_CUBE",c=Rm(i),u=Pm(i),d=xm(i),f=_m(o),p=a.createProgram(),x,g,m=i.glslVersion?"#version "+i.glslVersion+`
`:"";i.isRawShaderMaterial?(x=["#define SHADER_TYPE "+i.shaderType,"#define SHADER_NAME "+i.shaderName,f].filter(co).join(`
`),x.length>0&&(x+=`
`),g=["#define SHADER_TYPE "+i.shaderType,"#define SHADER_NAME "+i.shaderName,f].filter(co).join(`
`),g.length>0&&(g+=`
`)):(x=[Wh(i),"#define SHADER_TYPE "+i.shaderType,"#define SHADER_NAME "+i.shaderName,f,i.extensionClipCullDistance?"#define USE_CLIP_DISTANCE":"",i.batching?"#define USE_BATCHING":"",i.batchingColor?"#define USE_BATCHING_COLOR":"",i.instancing?"#define USE_INSTANCING":"",i.instancingColor?"#define USE_INSTANCING_COLOR":"",i.instancingMorph?"#define USE_INSTANCING_MORPH":"",i.useFog&&i.fog?"#define USE_FOG":"",i.useFog&&i.fogExp2?"#define FOG_EXP2":"",i.map?"#define USE_MAP":"",i.envMap?"#define USE_ENVMAP":"",i.envMap?"#define "+c:"",i.lightMap?"#define USE_LIGHTMAP":"",i.aoMap?"#define USE_AOMAP":"",i.bumpMap?"#define USE_BUMPMAP":"",i.normalMap?"#define USE_NORMALMAP":"",i.normalMapObjectSpace?"#define USE_NORMALMAP_OBJECTSPACE":"",i.normalMapTangentSpace?"#define USE_NORMALMAP_TANGENTSPACE":"",i.displacementMap?"#define USE_DISPLACEMENTMAP":"",i.emissiveMap?"#define USE_EMISSIVEMAP":"",i.anisotropy?"#define USE_ANISOTROPY":"",i.anisotropyMap?"#define USE_ANISOTROPYMAP":"",i.clearcoatMap?"#define USE_CLEARCOATMAP":"",i.clearcoatRoughnessMap?"#define USE_CLEARCOAT_ROUGHNESSMAP":"",i.clearcoatNormalMap?"#define USE_CLEARCOAT_NORMALMAP":"",i.diffuseRoughnessMap?"#define USE_DIFFUSE_ROUGHNESSMAP":"",i.iridescenceMap?"#define USE_IRIDESCENCEMAP":"",i.iridescenceThicknessMap?"#define USE_IRIDESCENCE_THICKNESSMAP":"",i.specularMap?"#define USE_SPECULARMAP":"",i.specularColorMap?"#define USE_SPECULAR_COLORMAP":"",i.specularIntensityMap?"#define USE_SPECULAR_INTENSITYMAP":"",i.roughnessMap?"#define USE_ROUGHNESSMAP":"",i.metalnessMap?"#define USE_METALNESSMAP":"",i.alphaMap?"#define USE_ALPHAMAP":"",i.alphaHash?"#define USE_ALPHAHASH":"",i.transmission?"#define USE_TRANSMISSION":"",i.transmissionMap?"#define USE_TRANSMISSIONMAP":"",i.thicknessMap?"#define USE_THICKNESSMAP":"",i.sheenColorMap?"#define USE_SHEEN_COLORMAP":"",i.sheenRoughnessMap?"#define USE_SHEEN_ROUGHNESSMAP":"",i.mapUv?"#define MAP_UV "+i.mapUv:"",i.alphaMapUv?"#define ALPHAMAP_UV "+i.alphaMapUv:"",i.lightMapUv?"#define LIGHTMAP_UV "+i.lightMapUv:"",i.aoMapUv?"#define AOMAP_UV "+i.aoMapUv:"",i.emissiveMapUv?"#define EMISSIVEMAP_UV "+i.emissiveMapUv:"",i.bumpMapUv?"#define BUMPMAP_UV "+i.bumpMapUv:"",i.normalMapUv?"#define NORMALMAP_UV "+i.normalMapUv:"",i.displacementMapUv?"#define DISPLACEMENTMAP_UV "+i.displacementMapUv:"",i.metalnessMapUv?"#define METALNESSMAP_UV "+i.metalnessMapUv:"",i.roughnessMapUv?"#define ROUGHNESSMAP_UV "+i.roughnessMapUv:"",i.anisotropyMapUv?"#define ANISOTROPYMAP_UV "+i.anisotropyMapUv:"",i.clearcoatMapUv?"#define CLEARCOATMAP_UV "+i.clearcoatMapUv:"",i.clearcoatNormalMapUv?"#define CLEARCOAT_NORMALMAP_UV "+i.clearcoatNormalMapUv:"",i.clearcoatRoughnessMapUv?"#define CLEARCOAT_ROUGHNESSMAP_UV "+i.clearcoatRoughnessMapUv:"",i.diffuseRoughnessMapUv?"#define DIFFUSE_ROUGHNESSMAP_UV "+i.diffuseRoughnessMapUv:"",i.iridescenceMapUv?"#define IRIDESCENCEMAP_UV "+i.iridescenceMapUv:"",i.iridescenceThicknessMapUv?"#define IRIDESCENCE_THICKNESSMAP_UV "+i.iridescenceThicknessMapUv:"",i.sheenColorMapUv?"#define SHEEN_COLORMAP_UV "+i.sheenColorMapUv:"",i.sheenRoughnessMapUv?"#define SHEEN_ROUGHNESSMAP_UV "+i.sheenRoughnessMapUv:"",i.specularMapUv?"#define SPECULARMAP_UV "+i.specularMapUv:"",i.specularColorMapUv?"#define SPECULAR_COLORMAP_UV "+i.specularColorMapUv:"",i.specularIntensityMapUv?"#define SPECULAR_INTENSITYMAP_UV "+i.specularIntensityMapUv:"",i.transmissionMapUv?"#define TRANSMISSIONMAP_UV "+i.transmissionMapUv:"",i.thicknessMapUv?"#define THICKNESSMAP_UV "+i.thicknessMapUv:"",i.vertexTangents&&i.flatShading===!1?"#define USE_TANGENT":"",i.vertexNormals?"#define HAS_NORMAL":"",i.vertexColors?"#define USE_COLOR":"",i.vertexAlphas?"#define USE_COLOR_ALPHA":"",i.vertexUv1s?"#define USE_UV1":"",i.vertexUv2s?"#define USE_UV2":"",i.vertexUv3s?"#define USE_UV3":"",i.pointsUvs?"#define USE_POINTS_UV":"",i.flatShading?"#define FLAT_SHADED":"",i.skinning?"#define USE_SKINNING":"",i.morphTargets?"#define USE_MORPHTARGETS":"",i.morphNormals&&i.flatShading===!1?"#define USE_MORPHNORMALS":"",i.morphColors?"#define USE_MORPHCOLORS":"",i.morphTargetsCount>0?"#define MORPHTARGETS_TEXTURE_STRIDE "+i.morphTextureStride:"",i.morphTargetsCount>0?"#define MORPHTARGETS_COUNT "+i.morphTargetsCount:"",i.doubleSided?"#define DOUBLE_SIDED":"",i.flipSided?"#define FLIP_SIDED":"",i.shadowMapEnabled?"#define USE_SHADOWMAP":"",i.shadowMapEnabled?"#define "+l:"",i.sizeAttenuation?"#define USE_SIZEATTENUATION":"",i.numLightProbes>0?"#define USE_LIGHT_PROBES":"",i.logarithmicDepthBuffer?"#define USE_LOGARITHMIC_DEPTH_BUFFER":"",i.reversedDepthBuffer?"#define USE_REVERSED_DEPTH_BUFFER":"","uniform mat4 modelMatrix;","uniform mat4 modelViewMatrix;","uniform mat4 projectionMatrix;","uniform mat4 viewMatrix;","uniform mat3 normalMatrix;","uniform vec3 cameraPosition;","uniform bool isOrthographic;","#ifdef USE_INSTANCING","	attribute mat4 instanceMatrix;","#endif","#ifdef USE_INSTANCING_COLOR","	attribute vec3 instanceColor;","#endif","#ifdef USE_INSTANCING_MORPH","	uniform sampler2D morphTexture;","#endif","attribute vec3 position;","attribute vec3 normal;","attribute vec2 uv;","#ifdef USE_UV1","	attribute vec2 uv1;","#endif","#ifdef USE_UV2","	attribute vec2 uv2;","#endif","#ifdef USE_UV3","	attribute vec2 uv3;","#endif","#ifdef USE_TANGENT","	attribute vec4 tangent;","#endif","#if defined( USE_COLOR_ALPHA )","	attribute vec4 color;","#elif defined( USE_COLOR )","	attribute vec3 color;","#endif","#ifdef USE_SKINNING","	attribute vec4 skinIndex;","	attribute vec4 skinWeight;","#endif",`
`].filter(co).join(`
`),g=[Wh(i),"#define SHADER_TYPE "+i.shaderType,"#define SHADER_NAME "+i.shaderName,f,i.useFog&&i.fog?"#define USE_FOG":"",i.useFog&&i.fogExp2?"#define FOG_EXP2":"",i.alphaToCoverage?"#define ALPHA_TO_COVERAGE":"",i.map?"#define USE_MAP":"",i.matcap?"#define USE_MATCAP":"",i.envMap?"#define USE_ENVMAP":"",i.envMap?"#define "+h:"",i.envMap?"#define "+c:"",i.envMap?"#define "+u:"",i.envMapPMREM?"#define ENVMAP_MAX_LOD "+i.envMapMaxLod+".0":"",i.envMapPMREM?"#define ENVMAP_SIZE "+i.envMapSize+".0":"",i.lightMap?"#define USE_LIGHTMAP":"",i.aoMap?"#define USE_AOMAP":"",i.bumpMap?"#define USE_BUMPMAP":"",i.normalMap?"#define USE_NORMALMAP":"",i.normalMapObjectSpace?"#define USE_NORMALMAP_OBJECTSPACE":"",i.normalMapTangentSpace?"#define USE_NORMALMAP_TANGENTSPACE":"",i.packedNormalMap?"#define USE_PACKED_NORMALMAP":"",i.emissiveMap?"#define USE_EMISSIVEMAP":"",i.anisotropy?"#define USE_ANISOTROPY":"",i.anisotropyMap?"#define USE_ANISOTROPYMAP":"",i.clearcoat?"#define USE_CLEARCOAT":"",i.clearcoatMap?"#define USE_CLEARCOATMAP":"",i.clearcoatRoughnessMap?"#define USE_CLEARCOAT_ROUGHNESSMAP":"",i.clearcoatNormalMap?"#define USE_CLEARCOAT_NORMALMAP":"",i.diffuseRoughness?"#define USE_DIFFUSE_ROUGHNESS":"",i.diffuseRoughnessMap?"#define USE_DIFFUSE_ROUGHNESSMAP":"",i.dispersion?"#define USE_DISPERSION":"",i.retroreflection?"#define USE_RETROREFLECTION":"",i.iridescence?"#define USE_IRIDESCENCE":"",i.iridescenceMap?"#define USE_IRIDESCENCEMAP":"",i.iridescenceThicknessMap?"#define USE_IRIDESCENCE_THICKNESSMAP":"",i.specularMap?"#define USE_SPECULARMAP":"",i.specularColorMap?"#define USE_SPECULAR_COLORMAP":"",i.specularIntensityMap?"#define USE_SPECULAR_INTENSITYMAP":"",i.roughnessMap?"#define USE_ROUGHNESSMAP":"",i.metalnessMap?"#define USE_METALNESSMAP":"",i.alphaMap?"#define USE_ALPHAMAP":"",i.alphaTest?"#define USE_ALPHATEST":"",i.alphaHash?"#define USE_ALPHAHASH":"",i.sheen?"#define USE_SHEEN":"",i.sheenColorMap?"#define USE_SHEEN_COLORMAP":"",i.sheenRoughnessMap?"#define USE_SHEEN_ROUGHNESSMAP":"",i.transmission?"#define USE_TRANSMISSION":"",i.transmissionMap?"#define USE_TRANSMISSIONMAP":"",i.thicknessMap?"#define USE_THICKNESSMAP":"",i.vertexTangents&&i.flatShading===!1?"#define USE_TANGENT":"",i.vertexColors||i.instancingColor?"#define USE_COLOR":"",i.vertexAlphas||i.batchingColor?"#define USE_COLOR_ALPHA":"",i.vertexUv1s?"#define USE_UV1":"",i.vertexUv2s?"#define USE_UV2":"",i.vertexUv3s?"#define USE_UV3":"",i.pointsUvs?"#define USE_POINTS_UV":"",i.gradientMap?"#define USE_GRADIENTMAP":"",i.flatShading?"#define FLAT_SHADED":"",i.doubleSided?"#define DOUBLE_SIDED":"",i.flipSided?"#define FLIP_SIDED":"",i.shadowMapEnabled?"#define USE_SHADOWMAP":"",i.shadowMapEnabled?"#define "+l:"",i.premultipliedAlpha?"#define PREMULTIPLIED_ALPHA":"",i.numLightProbes>0?"#define USE_LIGHT_PROBES":"",i.decodeVideoTexture?"#define DECODE_VIDEO_TEXTURE":"",i.decodeVideoTextureEmissive?"#define DECODE_VIDEO_TEXTURE_EMISSIVE":"",i.logarithmicDepthBuffer?"#define USE_LOGARITHMIC_DEPTH_BUFFER":"",i.reversedDepthBuffer?"#define USE_REVERSED_DEPTH_BUFFER":"","uniform mat4 viewMatrix;","uniform vec3 cameraPosition;","uniform bool isOrthographic;",i.toneMapping===0?"":"#define TONE_MAPPING",i.toneMapping===0?"":ot.tonemapping_pars_fragment,i.toneMapping===0?"":gm("toneMapping",i.toneMapping),i.dithering?"#define DITHERING":"",i.opaque?"#define OPAQUE":"",ot.colorspace_pars_fragment,pm("linearToOutputTexel",i.outputColorSpace),vm(),i.useDepthPacking?"#define DEPTH_PACKING "+i.depthPacking:"",`
`].filter(co).join(`
`)),s=il(s),s=Vh(s,i),s=Gh(s,i),n=il(n),n=Vh(n,i),n=Gh(n,i),s=Hh(s),n=Hh(n),i.isRawShaderMaterial!==!0&&(m=`#version 300 es
`,x=[d,"#define attribute in","#define varying out","#define texture2D texture"].join(`
`)+`
`+x,g=["#define varying in",i.glslVersion==="300 es"?"":"layout(location = 0) out highp vec4 pc_fragColor;",i.glslVersion==="300 es"?"":"#define gl_FragColor pc_fragColor","#define gl_FragDepthEXT gl_FragDepth","#define texture2D texture","#define textureCube texture","#define texture2DProj textureProj","#define texture2DLodEXT textureLod","#define texture2DProjLodEXT textureProjLod","#define textureCubeLodEXT textureLod","#define texture2DGradEXT textureGrad","#define texture2DProjGradEXT textureProjGrad","#define textureCubeGradEXT textureGrad"].join(`
`)+`
`+g);let y=m+x+s,M=m+g+n,b=Fh(a,a.VERTEX_SHADER,y),E=Fh(a,a.FRAGMENT_SHADER,M);a.attachShader(p,b),a.attachShader(p,E),i.index0AttributeName===void 0?i.hasPositionAttribute===!0&&a.bindAttribLocation(p,0,"position"):a.bindAttribLocation(p,0,i.index0AttributeName),a.linkProgram(p);function T(C){if(e.debug.checkShaderErrors){let R=a.getProgramInfoLog(p)||"",L=a.getShaderInfoLog(b)||"",G=a.getShaderInfoLog(E)||"",P=R.trim(),z=L.trim(),H=G.trim(),D=!0,ae=!0;if(a.getProgramParameter(p,a.LINK_STATUS)===!1)if(D=!1,typeof e.debug.onShaderError=="function")e.debug.onShaderError(a,p,b,E);else{let j=kh(a,b,"vertex"),Y=kh(a,E,"fragment");Qe("WebGLProgram: Shader Error "+a.getError()+" - VALIDATE_STATUS "+a.getProgramParameter(p,a.VALIDATE_STATUS)+`

Material Name: `+C.name+`
Material Type: `+C.type+`

Program Info Log: `+P+`
`+j+`
`+Y)}else P===""?(z===""||H==="")&&(ae=!1):Je("WebGLProgram: Program Info Log:",P);ae&&(C.diagnostics={runnable:D,programLog:P,vertexShader:{log:z,prefix:x},fragmentShader:{log:H,prefix:g}})}a.deleteShader(b),a.deleteShader(E),v=new ws(a,p),_=ym(a,p)}let v;this.getUniforms=function(){return v===void 0&&T(this),v};let _;this.getAttributes=function(){return _===void 0&&T(this),_};let A=i.rendererExtensionParallelShaderCompile===!1;return this.isReady=function(){return A===!1&&(A=a.getProgramParameter(p,cm)),A},this.destroy=function(){r.releaseStatesOfProgram(this),a.deleteProgram(p),this.program=void 0},this.type=i.shaderType,this.name=i.shaderName,this.id=um++,this.cacheKey=t,this.usedTimes=1,this.program=p,this.vertexShader=b,this.fragmentShader=E,this}var Nm=0,Im=class{constructor(){this.shaderCache=new Map,this.materialCache=new Map}update(e,t,i){let r=this._getShaderCacheForMaterial(e);return r.has(t)===!1&&(r.add(t),t.usedTimes++),r.has(i)===!1&&(r.add(i),i.usedTimes++),this}remove(e){let t=this.materialCache.get(e);for(let i of t)i.usedTimes--,i.usedTimes===0&&this.shaderCache.delete(i.code);return this.materialCache.delete(e),this}getVertexShaderStage(e){return this._getShaderStage(e.vertexShader)}getFragmentShaderStage(e){return this._getShaderStage(e.fragmentShader)}dispose(){this.shaderCache.clear(),this.materialCache.clear()}_getShaderCacheForMaterial(e){let t=this.materialCache,i=t.get(e);return i===void 0&&(i=new Set,t.set(e,i)),i}_getShaderStage(e){let t=this.shaderCache,i=t.get(e);return i===void 0&&(i=new Dm(e),t.set(e,i)),i}},Dm=class{constructor(e){this.id=Nm++,this.code=e,this.usedTimes=0}};function Um(e){return e===1030||e===37490||e===36285}function Om(e,t,i,r,a,o){let s=new Gc,n=new Im,l=new Set,h=[],c=new Map,u=r.logarithmicDepthBuffer,d={MeshDepthMaterial:"depth",MeshDistanceMaterial:"distance",MeshNormalMaterial:"normal",MeshBasicMaterial:"basic",MeshLambertMaterial:"lambert",MeshPhongMaterial:"phong",MeshToonMaterial:"toon",MeshStandardMaterial:"physical",MeshPhysicalMaterial:"physical",MeshMatcapMaterial:"matcap",LineBasicMaterial:"basic",LineDashedMaterial:"dashed",PointsMaterial:"points",ShadowMaterial:"shadow",SpriteMaterial:"sprite"};function f(v){return l.add(v),v===0?"uv":`uv${v}`}function p(v,_,A,C,R,L){let G=C.fog,P=R.geometry,z=v.isMeshStandardMaterial||v.isMeshLambertMaterial||v.isMeshPhongMaterial?C.environment:null,H=v.isMeshStandardMaterial||v.isMeshLambertMaterial&&!v.envMap||v.isMeshPhongMaterial&&!v.envMap,D=t.get(v.envMap||z,H),ae=d[v.type],j=r.precision;v.precision!==null&&(j=r.getMaxPrecision(v.precision),j!==v.precision&&Je("WebGLProgram.getParameters:",v.precision,"not supported, using",j,"instead."));let Y=P.morphAttributes.position||P.morphAttributes.normal||P.morphAttributes.color,ie=Y===void 0?0:Y.length,Se=0;P.morphAttributes.position!==void 0&&(Se=1),P.morphAttributes.normal!==void 0&&(Se=2),P.morphAttributes.color!==void 0&&(Se=3);let K,re,xe,Ne;if(ae){let Oe=or[ae];K=Oe.vertexShader,re=Oe.fragmentShader}else{K=v.vertexShader,re=v.fragmentShader;let Oe=n.getVertexShaderStage(v),Ot=n.getFragmentShaderStage(v);n.update(v,Oe,Ot),xe=Oe.id,Ne=Ot.id}let Me=e.getRenderTarget(),Q=e.state.buffers.depth.getReversed(),ke=R.isInstancedMesh===!0,$e=R.isBatchedMesh===!0,De=!!v.map,nt=!!v.matcap,oe=!!D,he=oe&&D.isPMREMTexture===!0,de=!!v.aoMap,Ue=!!v.lightMap,Ie=!!v.bumpMap&&v.wireframe===!1,F=!!v.normalMap,te=!!v.displacementMap,$=!!v.emissiveMap,Te=!!v.metalnessMap,Be=!!v.roughnessMap,U=v.anisotropy>0,ut=v.clearcoat>0,B=v.diffuseRoughness>0,Ce=v.dispersion>0,ne=v.retroreflectivity>0,lt=v.iridescence>0,Ge=v.sheen>0,je=v.transmission>0,At=U&&!!v.anisotropyMap,N=ut&&!!v.clearcoatMap,S=ut&&!!v.clearcoatNormalMap,W=ut&&!!v.clearcoatRoughnessMap,ce=B&&!!v.diffuseRoughnessMap,ee=lt&&!!v.iridescenceMap,ye=lt&&!!v.iridescenceThicknessMap,_e=Ge&&!!v.sheenColorMap,V=Ge&&!!v.sheenRoughnessMap,le=!!v.specularMap,Pe=!!v.specularColorMap,O=!!v.specularIntensityMap,Z=je&&!!v.transmissionMap,ue=je&&!!v.thicknessMap,me=!!v.gradientMap,Ae=!!v.alphaMap,Ee=v.alphaTest>0,Ye=!!v.alphaHash,fe=!!v.extensions,tt=0;v.toneMapped&&(Me===null||Me.isXRRenderTarget===!0)&&(tt=e.toneMapping);let Ze=he?D.isCompressedCubeTexture?D.image[0].mipmaps:D.mipmaps:null,mt={shaderID:ae,shaderType:v.type,shaderName:v.name,vertexShader:K,fragmentShader:re,defines:v.defines,customVertexShaderID:xe,customFragmentShaderID:Ne,isRawShaderMaterial:v.isRawShaderMaterial===!0,glslVersion:v.glslVersion,precision:j,batching:$e,batchingColor:$e&&R._colorsTexture!==null,instancing:ke,instancingColor:ke&&R.instanceColor!==null,instancingMorph:ke&&R.morphTexture!==null,outputColorSpace:Me===null?e.outputColorSpace:Me.isXRRenderTarget===!0?Me.texture.colorSpace:ft.workingColorSpace,alphaToCoverage:!!v.alphaToCoverage,map:De,matcap:nt,envMap:oe,envMapMode:oe&&D.mapping,envMapPMREM:he,envMapMaxLod:he?Ze.length-1:null,envMapSize:he?Ze[0].width:null,aoMap:de,lightMap:Ue,bumpMap:Ie,normalMap:F,displacementMap:te,emissiveMap:$,normalMapObjectSpace:F&&v.normalMapType===1,normalMapTangentSpace:F&&v.normalMapType===0,packedNormalMap:F&&v.normalMapType===0&&Um(v.normalMap.format),metalnessMap:Te,roughnessMap:Be,anisotropy:U,anisotropyMap:At,clearcoat:ut,clearcoatMap:N,clearcoatNormalMap:S,clearcoatRoughnessMap:W,diffuseRoughness:B,diffuseRoughnessMap:ce,dispersion:Ce,retroreflection:ne,iridescence:lt,iridescenceMap:ee,iridescenceThicknessMap:ye,sheen:Ge,sheenColorMap:_e,sheenRoughnessMap:V,specularMap:le,specularColorMap:Pe,specularIntensityMap:O,transmission:je,transmissionMap:Z,thicknessMap:ue,gradientMap:me,opaque:v.transparent===!1&&v.blending===1&&v.alphaToCoverage===!1,alphaMap:Ae,alphaTest:Ee,alphaHash:Ye,combine:v.combine,mapUv:De&&f(v.map.channel),aoMapUv:de&&f(v.aoMap.channel),lightMapUv:Ue&&f(v.lightMap.channel),bumpMapUv:Ie&&f(v.bumpMap.channel),normalMapUv:F&&f(v.normalMap.channel),displacementMapUv:te&&f(v.displacementMap.channel),emissiveMapUv:$&&f(v.emissiveMap.channel),metalnessMapUv:Te&&f(v.metalnessMap.channel),roughnessMapUv:Be&&f(v.roughnessMap.channel),anisotropyMapUv:At&&f(v.anisotropyMap.channel),clearcoatMapUv:N&&f(v.clearcoatMap.channel),clearcoatNormalMapUv:S&&f(v.clearcoatNormalMap.channel),clearcoatRoughnessMapUv:W&&f(v.clearcoatRoughnessMap.channel),diffuseRoughnessMapUv:ce&&f(v.diffuseRoughnessMap.channel),iridescenceMapUv:ee&&f(v.iridescenceMap.channel),iridescenceThicknessMapUv:ye&&f(v.iridescenceThicknessMap.channel),sheenColorMapUv:_e&&f(v.sheenColorMap.channel),sheenRoughnessMapUv:V&&f(v.sheenRoughnessMap.channel),specularMapUv:le&&f(v.specularMap.channel),specularColorMapUv:Pe&&f(v.specularColorMap.channel),specularIntensityMapUv:O&&f(v.specularIntensityMap.channel),transmissionMapUv:Z&&f(v.transmissionMap.channel),thicknessMapUv:ue&&f(v.thicknessMap.channel),alphaMapUv:Ae&&f(v.alphaMap.channel),vertexTangents:!!P.attributes.tangent&&(F||U),vertexNormals:!!P.attributes.normal,vertexColors:v.vertexColors,vertexAlphas:v.vertexColors===!0&&!!P.attributes.color&&P.attributes.color.itemSize===4,pointsUvs:R.isPoints===!0&&!!P.attributes.uv&&(De||Ae),fog:!!G,useFog:v.fog===!0,fogExp2:!!G&&G.isFogExp2,flatShading:v.wireframe===!1&&(v.flatShading===!0||P.attributes.normal===void 0&&F===!1&&(v.isMeshLambertMaterial||v.isMeshPhongMaterial||v.isMeshStandardMaterial||v.isMeshPhysicalMaterial)),sizeAttenuation:v.sizeAttenuation===!0,logarithmicDepthBuffer:u,reversedDepthBuffer:Q,skinning:R.isSkinnedMesh===!0,hasPositionAttribute:P.attributes.position!==void 0,morphTargets:P.morphAttributes.position!==void 0,morphNormals:P.morphAttributes.normal!==void 0,morphColors:P.morphAttributes.color!==void 0,morphTargetsCount:ie,morphTextureStride:Se,numSunLights:_.sun.length,numDirLights:_.directional.length,numPointLights:_.point.length,numSpotLights:_.spot.length,numSpotLightMaps:_.spotLightMap.length,numRectAreaLights:_.rectArea.length,numHemiLights:_.hemi.length,numSunLightShadows:_.sunShadowMap.length,numDirLightShadows:_.directionalShadowMap.length,numPointLightShadows:_.pointShadowMap.length,numSpotLightShadows:_.spotShadowMap.length,numSpotLightShadowsWithMaps:_.numSpotLightShadowsWithMaps,numLightProbes:_.numLightProbes,numLightProbeGrids:L.filter(Oe=>Oe.texture!==null).length,numClippingPlanes:o.numPlanes,numClipIntersection:o.numIntersection,dithering:v.dithering,shadowMapEnabled:e.shadowMap.enabled&&A.length>0,shadowMapType:e.shadowMap.type,toneMapping:tt,decodeVideoTexture:De&&v.map.isVideoTexture===!0&&ft.getTransfer(v.map.colorSpace)==="srgb",decodeVideoTextureEmissive:$&&v.emissiveMap.isVideoTexture===!0&&ft.getTransfer(v.emissiveMap.colorSpace)==="srgb",premultipliedAlpha:v.premultipliedAlpha,doubleSided:v.side===2,flipSided:v.side===1,useDepthPacking:v.depthPacking>=0,depthPacking:v.depthPacking||0,index0AttributeName:v.index0AttributeName,extensionClipCullDistance:fe&&v.extensions.clipCullDistance===!0&&i.has("WEBGL_clip_cull_distance"),extensionMultiDraw:(fe&&v.extensions.multiDraw===!0||$e)&&i.has("WEBGL_multi_draw"),rendererExtensionParallelShaderCompile:i.has("KHR_parallel_shader_compile"),customProgramCacheKey:v.customProgramCacheKey()};return mt.vertexUv1s=l.has(1),mt.vertexUv2s=l.has(2),mt.vertexUv3s=l.has(3),l.clear(),mt}function x(v){let _=[];if(v.shaderID?_.push(v.shaderID):(_.push(v.customVertexShaderID),_.push(v.customFragmentShaderID)),v.defines!==void 0)for(let A in v.defines)_.push(A),_.push(v.defines[A]);return v.isRawShaderMaterial===!1&&(g(_,v),m(_,v),_.push(e.outputColorSpace)),_.push(v.customProgramCacheKey),_.join()}function g(v,_){v.push(_.precision),v.push(_.outputColorSpace),v.push(_.envMapMode),v.push(_.envMapPMREM),v.push(_.envMapMaxLod),v.push(_.envMapSize),v.push(_.mapUv),v.push(_.alphaMapUv),v.push(_.lightMapUv),v.push(_.aoMapUv),v.push(_.bumpMapUv),v.push(_.normalMapUv),v.push(_.displacementMapUv),v.push(_.emissiveMapUv),v.push(_.metalnessMapUv),v.push(_.roughnessMapUv),v.push(_.anisotropyMapUv),v.push(_.clearcoatMapUv),v.push(_.clearcoatNormalMapUv),v.push(_.clearcoatRoughnessMapUv),v.push(_.diffuseRoughnessMapUv),v.push(_.iridescenceMapUv),v.push(_.iridescenceThicknessMapUv),v.push(_.sheenColorMapUv),v.push(_.sheenRoughnessMapUv),v.push(_.specularMapUv),v.push(_.specularColorMapUv),v.push(_.specularIntensityMapUv),v.push(_.transmissionMapUv),v.push(_.thicknessMapUv),v.push(_.combine),v.push(_.fogExp2),v.push(_.sizeAttenuation),v.push(_.morphTargetsCount),v.push(_.numSunLights),v.push(_.numDirLights),v.push(_.numPointLights),v.push(_.numSpotLights),v.push(_.numSpotLightMaps),v.push(_.numHemiLights),v.push(_.numRectAreaLights),v.push(_.numSunLightShadows),v.push(_.numDirLightShadows),v.push(_.numPointLightShadows),v.push(_.numSpotLightShadows),v.push(_.numSpotLightShadowsWithMaps),v.push(_.numLightProbes),v.push(_.numLightProbeGrids),v.push(_.shadowMapType),v.push(_.toneMapping),v.push(_.numClippingPlanes),v.push(_.numClipIntersection),v.push(_.depthPacking)}function m(v,_){s.disableAll(),_.instancing&&s.enable(0),_.instancingColor&&s.enable(1),_.instancingMorph&&s.enable(2),_.matcap&&s.enable(3),_.envMap&&s.enable(4),_.normalMapObjectSpace&&s.enable(5),_.normalMapTangentSpace&&s.enable(6),_.clearcoat&&s.enable(7),_.iridescence&&s.enable(8),_.alphaTest&&s.enable(9),_.vertexColors&&s.enable(10),_.vertexAlphas&&s.enable(11),_.vertexUv1s&&s.enable(12),_.vertexUv2s&&s.enable(13),_.vertexUv3s&&s.enable(14),_.vertexTangents&&s.enable(15),_.anisotropy&&s.enable(16),_.alphaHash&&s.enable(17),_.batching&&s.enable(18),_.dispersion&&s.enable(19),_.retroreflection&&s.enable(24),_.diffuseRoughness&&s.enable(25),_.batchingColor&&s.enable(20),_.gradientMap&&s.enable(21),_.packedNormalMap&&s.enable(22),_.vertexNormals&&s.enable(23),v.push(s.mask),s.disableAll(),_.fog&&s.enable(0),_.useFog&&s.enable(1),_.flatShading&&s.enable(2),_.logarithmicDepthBuffer&&s.enable(3),_.reversedDepthBuffer&&s.enable(4),_.skinning&&s.enable(5),_.morphTargets&&s.enable(6),_.morphNormals&&s.enable(7),_.morphColors&&s.enable(8),_.premultipliedAlpha&&s.enable(9),_.shadowMapEnabled&&s.enable(10),_.doubleSided&&s.enable(11),_.flipSided&&s.enable(12),_.useDepthPacking&&s.enable(13),_.dithering&&s.enable(14),_.transmission&&s.enable(15),_.sheen&&s.enable(16),_.opaque&&s.enable(17),_.pointsUvs&&s.enable(18),_.decodeVideoTexture&&s.enable(19),_.decodeVideoTextureEmissive&&s.enable(20),_.alphaToCoverage&&s.enable(21),_.numLightProbeGrids>0&&s.enable(22),_.hasPositionAttribute&&s.enable(23),v.push(s.mask)}function y(v){let _=d[v.type],A;if(_){let C=or[_];A=Sf.clone(C.uniforms)}else A=v.uniforms;return A}function M(v,_){let A=c.get(_);return A===void 0?(A=new Lm(e,_,v,a),h.push(A),c.set(_,A)):++A.usedTimes,A}function b(v){if(--v.usedTimes===0){let _=h.indexOf(v);h[_]=h[h.length-1],h.pop(),c.delete(v.cacheKey),v.destroy()}}function E(v){n.remove(v)}function T(){n.dispose()}return{getParameters:p,getProgramCacheKey:x,getUniforms:y,acquireProgram:M,releaseProgram:b,releaseShaderCache:E,programs:h,dispose:T}}function zm(){let e=new WeakMap;function t(s){return e.has(s)}function i(s){let n=e.get(s);return n===void 0&&(n={},e.set(s,n)),n}function r(s){e.delete(s)}function a(s,n,l){e.get(s)[n]=l}function o(){e=new WeakMap}return{has:t,get:i,remove:r,update:a,dispose:o}}function Fm(e,t){return e.groupOrder===t.groupOrder?e.renderOrder===t.renderOrder?e.material.id===t.material.id?e.materialVariant===t.materialVariant?e.z===t.z?e.id-t.id:e.z-t.z:e.materialVariant-t.materialVariant:e.material.id-t.material.id:e.renderOrder-t.renderOrder:e.groupOrder-t.groupOrder}function Xh(e,t){return e.groupOrder===t.groupOrder?e.renderOrder===t.renderOrder?e.z===t.z?e.id-t.id:t.z-e.z:e.renderOrder-t.renderOrder:e.groupOrder-t.groupOrder}function qh(){let e=[],t=0,i=[],r=[],a=[];function o(){t=0,i.length=0,r.length=0,a.length=0}function s(d){let f=0;return d.isInstancedMesh&&(f+=2),d.isSkinnedMesh&&(f+=1),f}function n(d,f,p,x,g,m){let y=e[t];return y===void 0?(y={id:d.id,object:d,geometry:f,material:p,materialVariant:s(d),groupOrder:x,renderOrder:d.renderOrder,z:g,group:m},e[t]=y):(y.id=d.id,y.object=d,y.geometry=f,y.material=p,y.materialVariant=s(d),y.groupOrder=x,y.renderOrder=d.renderOrder,y.z=g,y.group=m),t++,y}function l(d,f,p,x,g,m,y){y.reversedDepth===!0&&(g=-g);let M=n(d,f,p,x,g,m);p.transmission>0?r.push(M):p.transparent===!0?a.push(M):i.push(M)}function h(d,f,p,x,g,m){let y=n(d,f,p,x,g,m);p.transmission>0?r.unshift(y):p.transparent===!0?a.unshift(y):i.unshift(y)}function c(d,f){i.length>1&&i.sort(d||Fm),r.length>1&&r.sort(f||Xh),a.length>1&&a.sort(f||Xh)}function u(){for(let d=t,f=e.length;d<f;d++){let p=e[d];if(p.id===null)break;p.id=null,p.object=null,p.geometry=null,p.material=null,p.group=null}}return{opaque:i,transmissive:r,transparent:a,init:o,push:l,unshift:h,finish:u,sort:c}}function Bm(){let e=new WeakMap;function t(r,a){let o=e.get(r),s;return o===void 0?(s=new qh,e.set(r,[s])):a>=o.length?(s=new qh,o.push(s)):s=o[a],s}function i(){e=new WeakMap}return{get:t,dispose:i}}function km(){let e={};return{get:function(t){if(e[t.id]!==void 0)return e[t.id];let i;switch(t.type){case"SunLight":case"DirectionalLight":i={direction:new I,color:new dt};break;case"SpotLight":i={position:new I,direction:new I,color:new dt,distance:0,coneCos:0,penumbraCos:0,decay:0};break;case"PointLight":i={position:new I,color:new dt,distance:0,decay:0};break;case"HemisphereLight":i={direction:new I,skyColor:new dt,groundColor:new dt};break;case"RectAreaLight":i={color:new dt,position:new I,halfWidth:new I,halfHeight:new I}}return e[t.id]=i,i}}}function Vm(){let e={};return{get:function(t){if(e[t.id]!==void 0)return e[t.id];let i;switch(t.type){case"SunLight":case"DirectionalLight":i={shadowIntensity:1,shadowBias:0,shadowNormalBias:0,shadowRadius:1,shadowMapSize:new ge};break;case"SpotLight":i={shadowIntensity:1,shadowBias:0,shadowNormalBias:0,shadowRadius:1,shadowMapSize:new ge};break;case"PointLight":i={shadowIntensity:1,shadowBias:0,shadowNormalBias:0,shadowRadius:1,shadowMapSize:new ge,shadowCameraNear:1,shadowCameraFar:1e3}}return e[t.id]=i,i}}}var Gm=0;function Hm(e,t){return(t.castShadow?2:0)-(e.castShadow?2:0)+ +!!t.map-!!e.map}function Wm(e){let t=new km,i=Vm(),r={version:0,hash:{sunLength:-1,directionalLength:-1,pointLength:-1,spotLength:-1,rectAreaLength:-1,hemiLength:-1,numSunShadows:-1,numDirectionalShadows:-1,numPointShadows:-1,numSpotShadows:-1,numSpotMaps:-1,numLightProbes:-1},ambient:[0,0,0],probe:[],sun:[],sunShadow:[],sunShadowMap:[],sunShadowMatrix:[],sunShadowCascade:[],directional:[],directionalShadow:[],directionalShadowMap:[],directionalShadowMatrix:[],spot:[],spotLightMap:[],spotShadow:[],spotShadowMap:[],spotLightMatrix:[],rectArea:[],rectAreaLTC1:null,rectAreaLTC2:null,point:[],pointShadow:[],pointShadowMap:[],pointShadowMatrix:[],hemi:[],numSpotLightShadowsWithMaps:0,numLightProbes:0};for(let h=0;h<9;h++)r.probe.push(new I);let a=new I,o=new gt,s=new gt;function n(h){let c=0,u=0,d=0;for(let L=0;L<9;L++)r.probe[L].set(0,0,0);let f=0,p=0,x=0,g=0,m=0,y=0,M=0,b=0,E=0,T=0,v=0,_=0,A=0,C=0;h.sort(Hm);for(let L=0,G=h.length;L<G;L++){let P=h[L],z=P.color,H=P.intensity,D=P.distance,ae=null;if(P.shadow&&P.shadow.map&&(ae=P.shadow.map.texture.format===1030?P.shadow.map.texture:P.shadow.map.depthTexture||P.shadow.map.texture),P.isAmbientLight)c+=z.r*H,u+=z.g*H,d+=z.b*H;else if(P.isLightProbe){for(let j=0;j<9;j++)r.probe[j].addScaledVector(P.sh.coefficients[j],H);C++}else if(P.isSunLight){let j=t.get(P);if(j.color.copy(P.color).multiplyScalar(P.intensity),P.castShadow){let Y=P.shadow,ie=i.get(P);ie.shadowIntensity=Y.intensity,ie.shadowBias=Y.bias,ie.shadowNormalBias=Y.normalBias,ie.shadowRadius=Y.radius,ie.shadowMapSize.copy(Y.mapSize).multiply(Y.getFrameExtents()),r.sunShadow[p]=ie,r.sunShadowMap[p]=ae;let Se=Y.getViewportCount();for(let K=0;K<Se;K++)r.sunShadowMatrix[x+K]=Y.getMatrix(K),r.sunShadowCascade[x+K]=Y._cascadeData[K];x+=Se,p++}r.sun[f]=j,f++}else if(P.isDirectionalLight){let j=t.get(P);if(j.color.copy(P.color).multiplyScalar(P.intensity),P.castShadow){let Y=P.shadow,ie=i.get(P);ie.shadowIntensity=Y.intensity,ie.shadowBias=Y.bias,ie.shadowNormalBias=Y.normalBias,ie.shadowRadius=Y.radius,ie.shadowMapSize=Y.mapSize,r.directionalShadow[g]=ie,r.directionalShadowMap[g]=ae,r.directionalShadowMatrix[g]=P.shadow.matrix,E++}r.directional[g]=j,g++}else if(P.isSpotLight){let j=t.get(P);j.position.setFromMatrixPosition(P.matrixWorld),j.color.copy(z).multiplyScalar(H),j.distance=D,j.coneCos=Math.cos(P.angle),j.penumbraCos=Math.cos(P.angle*(1-P.penumbra)),j.decay=P.decay,r.spot[y]=j;let Y=P.shadow;if(P.map&&(r.spotLightMap[_]=P.map,_++,Y.updateMatrices(P),P.castShadow&&A++),r.spotLightMatrix[y]=Y.matrix,P.castShadow){let ie=i.get(P);ie.shadowIntensity=Y.intensity,ie.shadowBias=Y.bias,ie.shadowNormalBias=Y.normalBias,ie.shadowRadius=Y.radius,ie.shadowMapSize=Y.mapSize,r.spotShadow[y]=ie,r.spotShadowMap[y]=ae,v++}y++}else if(P.isRectAreaLight){let j=t.get(P);j.color.copy(z).multiplyScalar(H),j.halfWidth.set(P.width*.5,0,0),j.halfHeight.set(0,P.height*.5,0),r.rectArea[M]=j,M++}else if(P.isPointLight){let j=t.get(P);if(j.color.copy(P.color).multiplyScalar(P.intensity),j.distance=P.distance,j.decay=P.decay,P.castShadow){let Y=P.shadow,ie=i.get(P);ie.shadowIntensity=Y.intensity,ie.shadowBias=Y.bias,ie.shadowNormalBias=Y.normalBias,ie.shadowRadius=Y.radius,ie.shadowMapSize=Y.mapSize,ie.shadowCameraNear=Y.camera.near,ie.shadowCameraFar=Y.camera.far,r.pointShadow[m]=ie,r.pointShadowMap[m]=ae,r.pointShadowMatrix[m]=P.shadow.matrix,T++}r.point[m]=j,m++}else if(P.isHemisphereLight){let j=t.get(P);j.skyColor.copy(P.color).multiplyScalar(H),j.groundColor.copy(P.groundColor).multiplyScalar(H),r.hemi[b]=j,b++}}M>0&&(e.has("OES_texture_float_linear")===!0?(r.rectAreaLTC1=Le.LTC_FLOAT_1,r.rectAreaLTC2=Le.LTC_FLOAT_2):(r.rectAreaLTC1=Le.LTC_HALF_1,r.rectAreaLTC2=Le.LTC_HALF_2)),r.ambient[0]=c,r.ambient[1]=u,r.ambient[2]=d;let R=r.hash;(R.sunLength!==f||R.directionalLength!==g||R.pointLength!==m||R.spotLength!==y||R.rectAreaLength!==M||R.hemiLength!==b||R.numSunShadows!==p||R.numDirectionalShadows!==E||R.numPointShadows!==T||R.numSpotShadows!==v||R.numSpotMaps!==_||R.numLightProbes!==C)&&(r.sun.length=f,r.directional.length=g,r.spot.length=y,r.rectArea.length=M,r.point.length=m,r.hemi.length=b,r.sunShadow.length=p,r.sunShadowMap.length=p,r.sunShadowMatrix.length=x,r.sunShadowCascade.length=x,r.directionalShadow.length=E,r.directionalShadowMap.length=E,r.directionalShadowMatrix.length=E,r.pointShadow.length=T,r.pointShadowMap.length=T,r.pointShadowMatrix.length=T,r.spotShadow.length=v,r.spotShadowMap.length=v,r.spotLightMatrix.length=v+_-A,r.spotLightMap.length=_,r.numSpotLightShadowsWithMaps=A,r.numLightProbes=C,R.sunLength=f,R.directionalLength=g,R.pointLength=m,R.spotLength=y,R.rectAreaLength=M,R.hemiLength=b,R.numSunShadows=p,R.numDirectionalShadows=E,R.numPointShadows=T,R.numSpotShadows=v,R.numSpotMaps=_,R.numLightProbes=C,r.version=Gm++)}function l(h,c){let u=0,d=0,f=0,p=0,x=0,g=0,m=c.matrixWorldInverse;for(let y=0,M=h.length;y<M;y++){let b=h[y];if(b.isSunLight){let E=r.sun[u];E.direction.setFromMatrixPosition(b.matrixWorld),E.direction.transformDirection(m),u++}else if(b.isDirectionalLight){let E=r.directional[d];E.direction.setFromMatrixPosition(b.matrixWorld),a.setFromMatrixPosition(b.target.matrixWorld),E.direction.sub(a),E.direction.transformDirection(m),d++}else if(b.isSpotLight){let E=r.spot[p];E.position.setFromMatrixPosition(b.matrixWorld),E.position.applyMatrix4(m),E.direction.setFromMatrixPosition(b.matrixWorld),a.setFromMatrixPosition(b.target.matrixWorld),E.direction.sub(a),E.direction.transformDirection(m),p++}else if(b.isRectAreaLight){let E=r.rectArea[x];E.position.setFromMatrixPosition(b.matrixWorld),E.position.applyMatrix4(m),s.identity(),o.copy(b.matrixWorld),o.premultiply(m),s.extractRotation(o),E.halfWidth.set(b.width*.5,0,0),E.halfHeight.set(0,b.height*.5,0),E.halfWidth.applyMatrix4(s),E.halfHeight.applyMatrix4(s),x++}else if(b.isPointLight){let E=r.point[f];E.position.setFromMatrixPosition(b.matrixWorld),E.position.applyMatrix4(m),f++}else if(b.isHemisphereLight){let E=r.hemi[g];E.direction.setFromMatrixPosition(b.matrixWorld),E.direction.transformDirection(m),g++}}}return{setup:n,setupView:l,state:r}}function jh(e){let t=new Wm(e),i=[],r=[],a=[];function o(u){c.camera=u,i.length=0,r.length=0,a.length=0}function s(u){u.isLightProbeGrid?a.push(u):i.push(u)}function n(u){r.push(u)}function l(){t.setup(i),a.sort((u,d)=>u.priority-d.priority||u.id-d.id)}function h(u){t.setupView(i,u)}let c={lightsArray:i,shadowsArray:r,lightProbeGridArray:a,camera:null,lights:t,transmissionRenderTarget:{},textureUnits:0};return{init:o,state:c,setupLights:l,setupLightsView:h,pushLight:s,pushShadow:n}}function Xm(e){let t=new WeakMap;function i(a,o=0){let s=t.get(a),n;return s===void 0?(n=new jh(e),t.set(a,[n])):o>=s.length?(n=new jh(e),s.push(n)):n=s[o],n}function r(){t=new WeakMap}return{get:i,dispose:r}}var qm=`void main() {
	gl_Position = vec4( position, 1.0 );
}`,jm=`uniform sampler2D shadow_pass;
uniform vec2 resolution;
uniform float radius;
void main() {
	const float samples = float( VSM_SAMPLES );
	float mean = 0.0;
	float squared_mean = 0.0;
	float uvStride = samples <= 1.0 ? 0.0 : 2.0 / ( samples - 1.0 );
	float uvStart = samples <= 1.0 ? 0.0 : - 1.0;
	for ( float i = 0.0; i < samples; i ++ ) {
		float uvOffset = uvStart + i * uvStride;
		#ifdef HORIZONTAL_PASS
			vec2 distribution = texture2D( shadow_pass, ( gl_FragCoord.xy + vec2( uvOffset, 0.0 ) * radius ) / resolution ).rg;
			mean += distribution.x;
			squared_mean += distribution.y * distribution.y + distribution.x * distribution.x;
		#else
			float depth = texture2D( shadow_pass, ( gl_FragCoord.xy + vec2( 0.0, uvOffset ) * radius ) / resolution ).r;
			mean += depth;
			squared_mean += depth * depth;
		#endif
	}
	mean = mean / samples;
	squared_mean = squared_mean / samples;
	float std_dev = sqrt( max( 0.0, squared_mean - mean * mean ) );
	gl_FragColor = vec4( mean, std_dev, 0.0, 1.0 );
}`,Ym=[new I(1,0,0),new I(-1,0,0),new I(0,1,0),new I(0,-1,0),new I(0,0,1),new I(0,0,-1)],Zm=[new I(0,-1,0),new I(0,-1,0),new I(0,0,1),new I(0,0,-1),new I(0,-1,0),new I(0,-1,0)],Yh=new gt,ao=new I,Nn=new I;function Jm(e,t,i){let r=new Os,a=new ge,o=new ge,s=new Ft,n=new Ef,l=new Af,h={},c=i.maxTextureSize,u={0:1,1:0,2:2},d=new $i({defines:{VSM_SAMPLES:8},uniforms:{shadow_pass:{value:null},resolution:{value:new ge},radius:{value:4}},vertexShader:qm,fragmentShader:jm}),f=d.clone();f.defines.HORIZONTAL_PASS=1;let p=new ti;p.setAttribute("position",new si(new Float32Array([-1,-1,.5,3,-1,.5,-1,3,.5]),3));let x=new Nt(p,d),g=this;this.enabled=!1,this.autoUpdate=!0,this.needsUpdate=!1,this.type=1;let m=this.type;this.render=function(T,v,_){if(g.enabled===!1||g.autoUpdate===!1&&g.needsUpdate===!1||T.length===0)return;this.type===2&&(Je("WebGLShadowMap: PCFSoftShadowMap has been removed. Using PCFShadowMap instead."),this.type=1);let A=e.getRenderTarget(),C=e.getActiveCubeFace(),R=e.getActiveMipmapLevel(),L=e.state;L.setBlending(0),L.buffers.depth.getReversed()===!0?L.buffers.color.setClear(0,0,0,0):L.buffers.color.setClear(1,1,1,1),L.buffers.depth.setTest(!0),L.setScissorTest(!1);let G=m!==this.type;G&&v.traverse(function(P){P.material&&(Array.isArray(P.material)?P.material.forEach(z=>z.needsUpdate=!0):P.material.needsUpdate=!0)});for(let P=0,z=T.length;P<z;P++){let H=T[P],D=H.shadow;if(D===void 0){Je("WebGLShadowMap:",H,"has no shadow.");continue}if(D.autoUpdate===!1&&D.needsUpdate===!1)continue;a.copy(D.mapSize);let ae=D.getFrameExtents();a.multiply(ae),o.copy(D.mapSize),(a.x>c||a.y>c)&&(a.x>c&&(o.x=Math.floor(c/ae.x),a.x=o.x*ae.x,D.mapSize.x=o.x),a.y>c&&(o.y=Math.floor(c/ae.y),a.y=o.y*ae.y,D.mapSize.y=o.y));let j=e.state.buffers.depth.getReversed();if(D.camera._reversedDepth=j,D.map===null||G===!0){if(D.map!==null&&(D.map.depthTexture!==null&&(D.map.depthTexture.dispose(),D.map.depthTexture=null),D.map.dispose()),this.type===3){if(H.isPointLight){Je("WebGLShadowMap: VSM shadow maps are not supported for PointLights. Use PCF or BasicShadowMap instead.");continue}D.map=new gi(a.x,a.y,{format:Ps,type:Rr,minFilter:jt,magFilter:jt,generateMipmaps:!1}),D.map.texture.name=H.name+".shadowMap",D.map.depthTexture=new ra(a.x,a.y,Co),D.map.depthTexture.name=H.name+".shadowMapDepth",D.map.depthTexture.format=bo,D.map.depthTexture.compareFunction=null,D.map.depthTexture.minFilter=oi,D.map.depthTexture.magFilter=oi}else H.isPointLight?(D.map=new Sl(a.x),D.map.depthTexture=new Jc(a.x,Vr)):(D.map=new gi(a.x,a.y),D.map.depthTexture=new ra(a.x,a.y,Vr)),D.map.depthTexture.name=H.name+".shadowMap",D.map.depthTexture.format=bo,this.type===1?(D.map.depthTexture.compareFunction=j?518:515,D.map.depthTexture.minFilter=jt,D.map.depthTexture.magFilter=jt):(D.map.depthTexture.compareFunction=null,D.map.depthTexture.minFilter=oi,D.map.depthTexture.magFilter=oi);D.camera.updateProjectionMatrix()}D.map.isWebGLCubeRenderTarget!==!0&&(D.map.width!==a.x||D.map.height!==a.y)&&D.map.setSize(a.x,a.y);let Y=D.map.isWebGLCubeRenderTarget?6:D.getViewportCount();H.isPointLight!==!0&&D.updateMatrices(H,_);let ie=D.camera.layers.mask&4294967294?D.camera.layers:_.layers;for(let Se=0;Se<Y;Se++){let K=D.getCamera(Se);if(H.isPointLight){let re=D.camera,xe=D.matrix,Ne=H.distance||re.far;Ne!==re.far&&(re.far=Ne,re.updateProjectionMatrix()),ao.setFromMatrixPosition(H.matrixWorld),re.position.copy(ao),Nn.copy(re.position),Nn.add(Ym[Se]),re.up.copy(Zm[Se]),re.lookAt(Nn),re.updateMatrixWorld(),xe.makeTranslation(-ao.x,-ao.y,-ao.z),Yh.multiplyMatrices(re.projectionMatrix,re.matrixWorldInverse),D._frustum.setFromProjectionMatrix(Yh,re.coordinateSystem,re.reversedDepth)}if(D.map.isWebGLCubeRenderTarget)e.setRenderTarget(D.map,Se),e.clear();else{Se===0&&(e.setRenderTarget(D.map),e.clear());let re=D.getViewport(Se);s.set(o.x*re.x,o.y*re.y,o.x*re.z,o.y*re.w),L.viewport(s)}r=D.getFrustum(Se),b(v,_,K,H,ie,this.type)}D.isPointLightShadow!==!0&&this.type===3&&y(D,_),D.needsUpdate=!1}m=this.type,g.needsUpdate=!1,e.setRenderTarget(A,C,R)};function y(T,v){let _=t.update(x);d.defines.VSM_SAMPLES!==T.blurSamples&&(d.defines.VSM_SAMPLES=T.blurSamples,f.defines.VSM_SAMPLES=T.blurSamples,d.needsUpdate=!0,f.needsUpdate=!0),T.mapPass===null?T.mapPass=new gi(a.x,a.y,{format:Ps,type:Rr}):(T.mapPass.width!==T.map.width||T.mapPass.height!==T.map.height)&&T.mapPass.setSize(T.map.width,T.map.height),d.uniforms.shadow_pass.value=T.map.depthTexture,d.uniforms.resolution.value.set(T.map.width,T.map.height),d.uniforms.radius.value=T.radius,e.setRenderTarget(T.mapPass),e.clear(),e.renderBufferDirect(v,null,_,d,x,null),f.uniforms.shadow_pass.value=T.mapPass.texture,f.uniforms.resolution.value.set(T.map.width,T.map.height),f.uniforms.radius.value=T.radius,e.setRenderTarget(T.map),e.clear(),e.renderBufferDirect(v,null,_,f,x,null)}function M(T,v,_,A){let C=null,R=_.isPointLight===!0?T.customDistanceMaterial:T.customDepthMaterial;if(R!==void 0)C=R;else if(C=_.isPointLight===!0?l:n,e.localClippingEnabled&&v.clipShadows===!0&&Array.isArray(v.clippingPlanes)&&v.clippingPlanes.length!==0||v.displacementMap&&v.displacementScale!==0||v.alphaMap&&v.alphaTest>0||v.map&&v.alphaTest>0||v.alphaToCoverage===!0){let L=C.uuid,G=v.uuid,P=h[L];P===void 0&&(P={},h[L]=P);let z=P[G];z===void 0&&(z=C.clone(),P[G]=z,v.addEventListener("dispose",E)),C=z}if(C.visible=v.visible,C.wireframe=v.wireframe,A===3?C.side=v.shadowSide===null?v.side:v.shadowSide:C.side=v.shadowSide===null?u[v.side]:v.shadowSide,C.alphaMap=v.alphaMap,C.alphaTest=v.alphaToCoverage===!0?.5:v.alphaTest,C.map=v.map,C.clipShadows=v.clipShadows,C.clippingPlanes=v.clippingPlanes,C.clipIntersection=v.clipIntersection,C.displacementMap=v.displacementMap,C.displacementScale=v.displacementScale,C.displacementBias=v.displacementBias,C.wireframeLinewidth=v.wireframeLinewidth,C.linewidth=v.linewidth,_.isPointLight===!0&&C.isMeshDistanceMaterial===!0){let L=e.properties.get(C);L.light=_}return C}function b(T,v,_,A,C,R){if(T.visible===!1)return;if(T.layers.test(C)&&(T.isMesh||T.isLine||T.isPoints)&&(T.castShadow||T.receiveShadow&&R===3)&&(!T.frustumCulled||T.intersectsFrustum(r))){T.modelViewMatrix.multiplyMatrices(_.matrixWorldInverse,T.matrixWorld);let G=t.update(T),P=T.material;if(Array.isArray(P)){let z=G.groups;for(let H=0,D=z.length;H<D;H++){let ae=z[H],j=P[ae.materialIndex];if(j&&j.visible){let Y=M(T,j,A,R);T.onBeforeShadow(e,T,v,_,G,Y,ae),e.renderBufferDirect(_,null,G,Y,T,ae),T.onAfterShadow(e,T,v,_,G,Y,ae)}}}else if(P.visible){let z=M(T,P,A,R);T.onBeforeShadow(e,T,v,_,G,z,null),e.renderBufferDirect(_,null,G,z,T,null),T.onAfterShadow(e,T,v,_,G,z,null)}}let L=T.children;for(let G=0,P=L.length;G<P;G++)b(L[G],v,_,A,C,R)}function E(T){T.target.removeEventListener("dispose",E);for(let v in h){let _=h[v],A=T.target.uuid;A in _&&(_[A].dispose(),delete _[A])}}}function Km(e,t){function i(){let O=!1,Z=new Ft,ue=null,me=new Ft(0,0,0,0);return{setMask:function(Ae){ue!==Ae&&!O&&(e.colorMask(Ae,Ae,Ae,Ae),ue=Ae)},setLocked:function(Ae){O=Ae},setClear:function(Ae,Ee,Ye,fe,tt){tt===!0&&(Ae*=fe,Ee*=fe,Ye*=fe),Z.set(Ae,Ee,Ye,fe),me.equals(Z)===!1&&(e.clearColor(Ae,Ee,Ye,fe),me.copy(Z))},reset:function(){O=!1,ue=null,me.set(-1,0,0,0)}}}function r(){let O=!1,Z=!1,ue=null,me=null,Ae=null;return{setReversed:function(Ee){if(Z!==Ee){let Ye=t.get("EXT_clip_control");Ee?Ye.clipControlEXT(Ye.LOWER_LEFT_EXT,Ye.ZERO_TO_ONE_EXT):Ye.clipControlEXT(Ye.LOWER_LEFT_EXT,Ye.NEGATIVE_ONE_TO_ONE_EXT),Z=Ee;let fe=Ae;Ae=null,this.setClear(fe)}},getReversed:function(){return Z},setTest:function(Ee){Ee?Q(e.DEPTH_TEST):ke(e.DEPTH_TEST)},setMask:function(Ee){ue!==Ee&&!O&&(e.depthMask(Ee),ue=Ee)},setFunc:function(Ee){if(Z&&(Ee=n0[Ee]),me!==Ee){switch(Ee){case 0:e.depthFunc(e.NEVER);break;case 1:e.depthFunc(e.ALWAYS);break;case 2:e.depthFunc(e.LESS);break;case 3:e.depthFunc(e.LEQUAL);break;case 4:e.depthFunc(e.EQUAL);break;case 5:e.depthFunc(e.GEQUAL);break;case 6:e.depthFunc(e.GREATER);break;case 7:e.depthFunc(e.NOTEQUAL);break;default:e.depthFunc(e.LEQUAL)}me=Ee}},setLocked:function(Ee){O=Ee},setClear:function(Ee){Ae!==Ee&&(Ae=Ee,Z&&(Ee=1-Ee),e.clearDepth(Ee))},reset:function(){O=!1,ue=null,me=null,Ae=null,Z=!1}}}function a(){let O=!1,Z=null,ue=null,me=null,Ae=null,Ee=null,Ye=null,fe=null,tt=null;return{setTest:function(Ze){O||(Ze?Q(e.STENCIL_TEST):ke(e.STENCIL_TEST))},setMask:function(Ze){Z!==Ze&&!O&&(e.stencilMask(Ze),Z=Ze)},setFunc:function(Ze,mt,Oe){(ue!==Ze||me!==mt||Ae!==Oe)&&(e.stencilFunc(Ze,mt,Oe),ue=Ze,me=mt,Ae=Oe)},setOp:function(Ze,mt,Oe){(Ee!==Ze||Ye!==mt||fe!==Oe)&&(e.stencilOp(Ze,mt,Oe),Ee=Ze,Ye=mt,fe=Oe)},setLocked:function(Ze){O=Ze},setClear:function(Ze){tt!==Ze&&(e.clearStencil(Ze),tt=Ze)},reset:function(){O=!1,Z=null,ue=null,me=null,Ae=null,Ee=null,Ye=null,fe=null,tt=null}}}let o=new i,s=new r,n=new a,l={},h=new WeakMap,c={},u={},d={},f=new WeakMap,p=[],x=null,g=!1,m=null,y=null,M=null,b=null,E=null,T=null,v=null,_=new dt(0,0,0),A=0,C=!1,R=null,L=null,G=null,P=null,z=null,H=e.getParameter(e.MAX_COMBINED_TEXTURE_IMAGE_UNITS),D=!1,ae=0,j=e.getParameter(e.VERSION);j.indexOf("WebGL")===-1?j.indexOf("OpenGL ES")!==-1&&(ae=parseFloat(/^OpenGL ES (\d)/.exec(j)[1]),D=ae>=2):(ae=parseFloat(/^WebGL (\d)/.exec(j)[1]),D=ae>=1);let Y=null,ie={},Se=e.getParameter(e.SCISSOR_BOX),K=e.getParameter(e.VIEWPORT),re=new Ft().fromArray(Se),xe=new Ft().fromArray(K);function Ne(O,Z,ue,me){let Ae=new Uint8Array(4),Ee=e.createTexture();e.bindTexture(O,Ee),e.texParameteri(O,e.TEXTURE_MIN_FILTER,e.NEAREST),e.texParameteri(O,e.TEXTURE_MAG_FILTER,e.NEAREST);for(let Ye=0;Ye<ue;Ye++)O===e.TEXTURE_3D||O===e.TEXTURE_2D_ARRAY?e.texImage3D(Z,0,e.RGBA,1,1,me,0,e.RGBA,e.UNSIGNED_BYTE,Ae):e.texImage2D(Z+Ye,0,e.RGBA,1,1,0,e.RGBA,e.UNSIGNED_BYTE,Ae);return Ee}let Me={};Me[e.TEXTURE_2D]=Ne(e.TEXTURE_2D,e.TEXTURE_2D,1),Me[e.TEXTURE_CUBE_MAP]=Ne(e.TEXTURE_CUBE_MAP,e.TEXTURE_CUBE_MAP_POSITIVE_X,6),Me[e.TEXTURE_2D_ARRAY]=Ne(e.TEXTURE_2D_ARRAY,e.TEXTURE_2D_ARRAY,1,1),Me[e.TEXTURE_3D]=Ne(e.TEXTURE_3D,e.TEXTURE_3D,1,1),o.setClear(0,0,0,1),s.setClear(1),n.setClear(0),Q(e.DEPTH_TEST),s.setFunc(3),Ie(!1),F(1),Q(e.CULL_FACE),de(0);function Q(O){c[O]!==!0&&(e.enable(O),c[O]=!0)}function ke(O){c[O]!==!1&&(e.disable(O),c[O]=!1)}function $e(O,Z){return d[O]!==Z&&(e.bindFramebuffer(O,Z),d[O]=Z,O===e.DRAW_FRAMEBUFFER&&(d[e.FRAMEBUFFER]=Z),O===e.FRAMEBUFFER&&(d[e.DRAW_FRAMEBUFFER]=Z),!0)}function De(O,Z){let ue=p,me=!1;if(O){ue=f.get(Z),ue===void 0&&(ue=[],f.set(Z,ue));let Ae=O.textures;if(ue.length!==Ae.length||ue[0]!==e.COLOR_ATTACHMENT0){for(let Ee=0,Ye=Ae.length;Ee<Ye;Ee++)ue[Ee]=e.COLOR_ATTACHMENT0+Ee;ue.length=Ae.length,me=!0}}else ue[0]!==e.BACK&&(ue[0]=e.BACK,me=!0);me&&e.drawBuffers(ue)}function nt(O){return x!==O&&(e.useProgram(O),x=O,!0)}let oe={100:e.FUNC_ADD,101:e.FUNC_SUBTRACT,102:e.FUNC_REVERSE_SUBTRACT};oe[103]=e.MIN,oe[104]=e.MAX;let he={200:e.ZERO,201:e.ONE,202:e.SRC_COLOR,204:e.SRC_ALPHA,210:e.SRC_ALPHA_SATURATE,208:e.DST_COLOR,206:e.DST_ALPHA,203:e.ONE_MINUS_SRC_COLOR,205:e.ONE_MINUS_SRC_ALPHA,209:e.ONE_MINUS_DST_COLOR,207:e.ONE_MINUS_DST_ALPHA,211:e.CONSTANT_COLOR,212:e.ONE_MINUS_CONSTANT_COLOR,213:e.CONSTANT_ALPHA,214:e.ONE_MINUS_CONSTANT_ALPHA};function de(O,Z,ue,me,Ae,Ee,Ye,fe,tt,Ze){if(O===0)g===!0&&(ke(e.BLEND),g=!1);else if(g===!1&&(Q(e.BLEND),g=!0),O!==5){if(O!==m||Ze!==C){if((y!==100||E!==100)&&(e.blendEquation(e.FUNC_ADD),y=100,E=100),Ze)switch(O){case 1:e.blendFuncSeparate(e.ONE,e.ONE_MINUS_SRC_ALPHA,e.ONE,e.ONE_MINUS_SRC_ALPHA);break;case 2:e.blendFunc(e.ONE,e.ONE);break;case 3:e.blendFuncSeparate(e.ZERO,e.ONE_MINUS_SRC_COLOR,e.ZERO,e.ONE);break;case 4:e.blendFuncSeparate(e.DST_COLOR,e.ONE_MINUS_SRC_ALPHA,e.ZERO,e.ONE);break;default:Qe("WebGLState: Invalid blending: ",O)}else switch(O){case 1:e.blendFuncSeparate(e.SRC_ALPHA,e.ONE_MINUS_SRC_ALPHA,e.ONE,e.ONE_MINUS_SRC_ALPHA);break;case 2:e.blendFuncSeparate(e.SRC_ALPHA,e.ONE,e.ONE,e.ONE);break;case 3:Qe("WebGLState: SubtractiveBlending requires material.premultipliedAlpha = true");break;case 4:Qe("WebGLState: MultiplyBlending requires material.premultipliedAlpha = true");break;default:Qe("WebGLState: Invalid blending: ",O)}M=null,b=null,T=null,v=null,_.set(0,0,0),A=0,m=O,C=Ze}}else Ae||=Z,Ee||=ue,Ye||=me,(Z!==y||Ae!==E)&&(e.blendEquationSeparate(oe[Z],oe[Ae]),y=Z,E=Ae),(ue!==M||me!==b||Ee!==T||Ye!==v)&&(e.blendFuncSeparate(he[ue],he[me],he[Ee],he[Ye]),M=ue,b=me,T=Ee,v=Ye),(fe.equals(_)===!1||tt!==A)&&(e.blendColor(fe.r,fe.g,fe.b,tt),_.copy(fe),A=tt),m=O,C=!1}function Ue(O,Z){O.side===2?ke(e.CULL_FACE):Q(e.CULL_FACE);let ue=O.side===1;Z&&(ue=!ue),Ie(ue),O.blending===1&&O.transparent===!1?de(0):de(O.blending,O.blendEquation,O.blendSrc,O.blendDst,O.blendEquationAlpha,O.blendSrcAlpha,O.blendDstAlpha,O.blendColor,O.blendAlpha,O.premultipliedAlpha),s.setFunc(O.depthFunc),s.setTest(O.depthTest),s.setMask(O.depthWrite),o.setMask(O.colorWrite);let me=O.stencilWrite;n.setTest(me),me&&(n.setMask(O.stencilWriteMask),n.setFunc(O.stencilFunc,O.stencilRef,O.stencilFuncMask),n.setOp(O.stencilFail,O.stencilZFail,O.stencilZPass)),$(O.polygonOffset,O.polygonOffsetFactor,O.polygonOffsetUnits),O.alphaToCoverage===!0?Q(e.SAMPLE_ALPHA_TO_COVERAGE):ke(e.SAMPLE_ALPHA_TO_COVERAGE)}function Ie(O){R!==O&&(O?e.frontFace(e.CW):e.frontFace(e.CCW),R=O)}function F(O){O===0?ke(e.CULL_FACE):(Q(e.CULL_FACE),O!==L&&(O===1?e.cullFace(e.BACK):O===2?e.cullFace(e.FRONT):e.cullFace(e.FRONT_AND_BACK))),L=O}function te(O){O!==G&&(D&&e.lineWidth(O),G=O)}function $(O,Z,ue){O?(Q(e.POLYGON_OFFSET_FILL),(P!==Z||z!==ue)&&(P=Z,z=ue,s.getReversed()&&(Z=-Z,ue=-ue),e.polygonOffset(Z,ue))):ke(e.POLYGON_OFFSET_FILL)}function Te(O){O?Q(e.SCISSOR_TEST):ke(e.SCISSOR_TEST)}function Be(O){O===void 0&&(O=e.TEXTURE0+H-1),Y!==O&&(e.activeTexture(O),Y=O)}function U(O,Z,ue){ue===void 0&&(ue=Y===null?e.TEXTURE0+H-1:Y);let me=ie[ue];me===void 0&&(me={type:void 0,texture:void 0},ie[ue]=me),(me.type!==O||me.texture!==Z)&&(Y!==ue&&(e.activeTexture(ue),Y=ue),e.bindTexture(O,Z||Me[O]),me.type=O,me.texture=Z)}function ut(){let O=ie[Y];O!==void 0&&O.type!==void 0&&(e.bindTexture(O.type,null),O.type=void 0,O.texture=void 0)}function B(){try{e.compressedTexImage2D(...arguments)}catch(O){Qe("WebGLState:",O)}}function Ce(){try{e.compressedTexImage3D(...arguments)}catch(O){Qe("WebGLState:",O)}}function ne(){try{e.texSubImage2D(...arguments)}catch(O){Qe("WebGLState:",O)}}function lt(){try{e.texSubImage3D(...arguments)}catch(O){Qe("WebGLState:",O)}}function Ge(){try{e.compressedTexSubImage2D(...arguments)}catch(O){Qe("WebGLState:",O)}}function je(){try{e.compressedTexSubImage3D(...arguments)}catch(O){Qe("WebGLState:",O)}}function At(){try{e.texStorage2D(...arguments)}catch(O){Qe("WebGLState:",O)}}function N(){try{e.texStorage3D(...arguments)}catch(O){Qe("WebGLState:",O)}}function S(){try{e.texImage2D(...arguments)}catch(O){Qe("WebGLState:",O)}}function W(){try{e.texImage3D(...arguments)}catch(O){Qe("WebGLState:",O)}}function ce(O){return u[O]===void 0?e.getParameter(O):u[O]}function ee(O,Z){u[O]!==Z&&(e.pixelStorei(O,Z),u[O]=Z)}function ye(O){re.equals(O)===!1&&(e.scissor(O.x,O.y,O.z,O.w),re.copy(O))}function _e(O){xe.equals(O)===!1&&(e.viewport(O.x,O.y,O.z,O.w),xe.copy(O))}function V(O,Z){let ue=h.get(Z);ue===void 0&&(ue=new WeakMap,h.set(Z,ue));let me=ue.get(O);me===void 0&&(me=e.getUniformBlockIndex(Z,O.name),e.uniformBlockBinding(Z,me,me),ue.set(O,me))}function le(O,Z,ue){let me=h.get(Z).get(O);l[me]!==ue&&(e.bindBufferBase(e.UNIFORM_BUFFER,me,ue),l[me]=ue)}function Pe(){e.disable(e.BLEND),e.disable(e.CULL_FACE),e.disable(e.DEPTH_TEST),e.disable(e.POLYGON_OFFSET_FILL),e.disable(e.SCISSOR_TEST),e.disable(e.STENCIL_TEST),e.disable(e.SAMPLE_ALPHA_TO_COVERAGE),e.blendEquation(e.FUNC_ADD),e.blendFunc(e.ONE,e.ZERO),e.blendFuncSeparate(e.ONE,e.ZERO,e.ONE,e.ZERO),e.blendColor(0,0,0,0),e.colorMask(!0,!0,!0,!0),e.clearColor(0,0,0,0),e.depthMask(!0),e.depthFunc(e.LESS),s.setReversed(!1),e.clearDepth(1),e.stencilMask(4294967295),e.stencilFunc(e.ALWAYS,0,4294967295),e.stencilOp(e.KEEP,e.KEEP,e.KEEP),e.clearStencil(0),e.cullFace(e.BACK),e.frontFace(e.CCW),e.polygonOffset(0,0),e.activeTexture(e.TEXTURE0),e.bindFramebuffer(e.FRAMEBUFFER,null),e.bindFramebuffer(e.DRAW_FRAMEBUFFER,null),e.bindFramebuffer(e.READ_FRAMEBUFFER,null),e.useProgram(null),e.lineWidth(1),e.scissor(0,0,e.canvas.width,e.canvas.height),e.viewport(0,0,e.canvas.width,e.canvas.height),e.pixelStorei(e.PACK_ALIGNMENT,4),e.pixelStorei(e.UNPACK_ALIGNMENT,4),e.pixelStorei(e.UNPACK_FLIP_Y_WEBGL,!1),e.pixelStorei(e.UNPACK_PREMULTIPLY_ALPHA_WEBGL,!1),e.pixelStorei(e.UNPACK_COLORSPACE_CONVERSION_WEBGL,e.BROWSER_DEFAULT_WEBGL),e.pixelStorei(e.PACK_ROW_LENGTH,0),e.pixelStorei(e.PACK_SKIP_PIXELS,0),e.pixelStorei(e.PACK_SKIP_ROWS,0),e.pixelStorei(e.UNPACK_ROW_LENGTH,0),e.pixelStorei(e.UNPACK_IMAGE_HEIGHT,0),e.pixelStorei(e.UNPACK_SKIP_PIXELS,0),e.pixelStorei(e.UNPACK_SKIP_ROWS,0),e.pixelStorei(e.UNPACK_SKIP_IMAGES,0),c={},u={},Y=null,ie={},d={},l={},f=new WeakMap,p=[],x=null,g=!1,m=null,y=null,M=null,b=null,E=null,T=null,v=null,_=new dt(0,0,0),A=0,C=!1,R=null,L=null,G=null,P=null,z=null,re.set(0,0,e.canvas.width,e.canvas.height),xe.set(0,0,e.canvas.width,e.canvas.height),o.reset(),s.reset(),n.reset()}return{buffers:{color:o,depth:s,stencil:n},enable:Q,disable:ke,bindFramebuffer:$e,drawBuffers:De,useProgram:nt,setBlending:de,setMaterial:Ue,setFlipSided:Ie,setCullFace:F,setLineWidth:te,setPolygonOffset:$,setScissorTest:Te,activeTexture:Be,bindTexture:U,unbindTexture:ut,compressedTexImage2D:B,compressedTexImage3D:Ce,texImage2D:S,texImage3D:W,pixelStorei:ee,getParameter:ce,updateUBOMapping:V,uniformBlockBinding:le,texStorage2D:At,texStorage3D:N,texSubImage2D:ne,texSubImage3D:lt,compressedTexSubImage2D:Ge,compressedTexSubImage3D:je,scissor:ye,viewport:_e,reset:Pe}}function $m(e,t,i,r,a,o,s){let n=t.has("WEBGL_multisampled_render_to_texture")?t.get("WEBGL_multisampled_render_to_texture"):null,l=typeof navigator>"u"?!1:/OculusBrowser/g.test(navigator.userAgent),h=new ge,c=new WeakMap,u=new Set,d,f=new WeakMap,p=new Set,x=new FinalizationRegistry(N=>p.delete(N)),g=!1;try{g=typeof OffscreenCanvas<"u"&&new OffscreenCanvas(1,1).getContext("2d")!==null}catch{}function m(N,S){return g?new OffscreenCanvas(N,S):Us("canvas")}function y(N,S,W){let ce=1,ee=At(N);if((ee.width>W||ee.height>W)&&(ce=W/Math.max(ee.width,ee.height)),ce<1){if(typeof HTMLImageElement<"u"&&N instanceof HTMLImageElement||typeof HTMLCanvasElement<"u"&&N instanceof HTMLCanvasElement||typeof ImageBitmap<"u"&&N instanceof ImageBitmap||typeof VideoFrame<"u"&&N instanceof VideoFrame){let ye=Math.floor(ce*ee.width),_e=Math.floor(ce*ee.height);d===void 0&&(d=m(ye,_e));let V=S?m(ye,_e):d;return V.width=ye,V.height=_e,V.getContext("2d").drawImage(N,0,0,ye,_e),Je("WebGLRenderer: Texture has been resized from ("+ee.width+"x"+ee.height+") to ("+ye+"x"+_e+")."),V}return"data"in N&&Je("WebGLRenderer: Image in DataTexture is too big ("+ee.width+"x"+ee.height+")."),N}return N}function M(N){return N.generateMipmaps}function b(N){e.generateMipmap(N)}function E(N){return N.isWebGLCubeRenderTarget?e.TEXTURE_CUBE_MAP:N.isWebGL3DRenderTarget?e.TEXTURE_3D:N.isWebGLArrayRenderTarget||N.isCompressedArrayTexture?e.TEXTURE_2D_ARRAY:e.TEXTURE_2D}function T(N,S,W,ce,ee,ye=!1){if(N!==null){if(e[N]!==void 0)return e[N];Je("WebGLRenderer: Attempt to use non-existing WebGL internal format '"+N+"'")}let _e;ce&&(_e=t.get("EXT_texture_norm16"),_e||Je("WebGLRenderer: Unable to use normalized textures without EXT_texture_norm16 extension"));let V=S;if(S===e.RED&&(W===e.FLOAT&&(V=e.R32F),W===e.HALF_FLOAT&&(V=e.R16F),W===e.UNSIGNED_BYTE&&(V=e.R8),W===e.UNSIGNED_SHORT&&_e&&(V=_e.R16_EXT),W===e.SHORT&&_e&&(V=_e.R16_SNORM_EXT)),S===e.RED_INTEGER&&(W===e.UNSIGNED_BYTE&&(V=e.R8UI),W===e.UNSIGNED_SHORT&&(V=e.R16UI),W===e.UNSIGNED_INT&&(V=e.R32UI),W===e.BYTE&&(V=e.R8I),W===e.SHORT&&(V=e.R16I),W===e.INT&&(V=e.R32I)),S===e.RG&&(W===e.FLOAT&&(V=e.RG32F),W===e.HALF_FLOAT&&(V=e.RG16F),W===e.UNSIGNED_BYTE&&(V=e.RG8),W===e.UNSIGNED_SHORT&&_e&&(V=_e.RG16_EXT),W===e.SHORT&&_e&&(V=_e.RG16_SNORM_EXT)),S===e.RG_INTEGER&&(W===e.UNSIGNED_BYTE&&(V=e.RG8UI),W===e.UNSIGNED_SHORT&&(V=e.RG16UI),W===e.UNSIGNED_INT&&(V=e.RG32UI),W===e.BYTE&&(V=e.RG8I),W===e.SHORT&&(V=e.RG16I),W===e.INT&&(V=e.RG32I)),S===e.RGB_INTEGER&&(W===e.UNSIGNED_BYTE&&(V=e.RGB8UI),W===e.UNSIGNED_SHORT&&(V=e.RGB16UI),W===e.UNSIGNED_INT&&(V=e.RGB32UI),W===e.BYTE&&(V=e.RGB8I),W===e.SHORT&&(V=e.RGB16I),W===e.INT&&(V=e.RGB32I)),S===e.RGBA_INTEGER&&(W===e.UNSIGNED_BYTE&&(V=e.RGBA8UI),W===e.UNSIGNED_SHORT&&(V=e.RGBA16UI),W===e.UNSIGNED_INT&&(V=e.RGBA32UI),W===e.BYTE&&(V=e.RGBA8I),W===e.SHORT&&(V=e.RGBA16I),W===e.INT&&(V=e.RGBA32I)),S===e.RGB&&(W===e.UNSIGNED_SHORT&&_e&&(V=_e.RGB16_EXT),W===e.SHORT&&_e&&(V=_e.RGB16_SNORM_EXT),W===e.UNSIGNED_INT_5_9_9_9_REV&&(V=e.RGB9_E5),W===e.UNSIGNED_INT_10F_11F_11F_REV&&(V=e.R11F_G11F_B10F)),S===e.RGBA){let le=ye?Is:ft.getTransfer(ee);W===e.FLOAT&&(V=e.RGBA32F),W===e.HALF_FLOAT&&(V=e.RGBA16F),W===e.UNSIGNED_BYTE&&(V=le==="srgb"?e.SRGB8_ALPHA8:e.RGBA8),W===e.UNSIGNED_SHORT&&_e&&(V=_e.RGBA16_EXT),W===e.SHORT&&_e&&(V=_e.RGBA16_SNORM_EXT),W===e.UNSIGNED_SHORT_4_4_4_4&&(V=e.RGBA4),W===e.UNSIGNED_SHORT_5_5_5_1&&(V=e.RGB5_A1)}return(V===e.R16F||V===e.R32F||V===e.RG16F||V===e.RG32F||V===e.RGBA16F||V===e.RGBA32F)&&t.get("EXT_color_buffer_float"),V}function v(N,S){let W;return N?S===null||S===1014||S===1020?W=e.DEPTH24_STENCIL8:S===1015?W=e.DEPTH32F_STENCIL8:S===1012&&(W=e.DEPTH24_STENCIL8,Je("DepthTexture: 16 bit depth attachment is not supported with stencil. Using 24-bit attachment.")):S===null||S===1014||S===1020?W=e.DEPTH_COMPONENT24:S===1015?W=e.DEPTH_COMPONENT32F:S===1012&&(W=e.DEPTH_COMPONENT16),W}function _(N,S){return M(N)===!0||N.isFramebufferTexture&&N.minFilter!==1003&&N.minFilter!==1006?Math.log2(Math.max(S.width,S.height))+1:N.mipmaps.length>0?N.mipmaps.length:N.isCompressedTexture&&Array.isArray(N.image)?S.mipmaps.length:1}function A(N){let S=new WeakRef(N);r.get(N).__ref=S,p.add(S),x.register(N,S,S)}function C(N){let S=r.get(N).__ref;S!==void 0&&(p.delete(S),x.unregister(S))}function R(N){G(N.target)}function L(N){P(N.target)}function G(N){N.removeEventListener("dispose",R),C(N),H(N),N.isVideoTexture&&c.delete(N),N.isHTMLTexture&&u.delete(N)}function P(N){N.removeEventListener("dispose",L),C(N),ae(N)}function z(){for(let N of p){let S=N.deref();S!==void 0&&(S.isRenderTarget===!0?P(S):G(S))}p.clear(),u.clear()}function H(N){let S=r.get(N);if(S.__webglInit===void 0)return;let W=N.source,ce=f.get(W);if(ce){let ee=ce[S.__cacheKey];ee.usedTimes--,ee.usedTimes===0&&D(N),Object.keys(ce).length===0&&f.delete(W)}r.remove(N)}function D(N){let S=r.get(N);e.deleteTexture(S.__webglTexture);let W=N.source,ce=f.get(W);delete ce[S.__cacheKey],s.memory.textures--}function ae(N){let S=r.get(N);S.__depthDisposeCallback&&S.__depthDisposeCallback();let W=N.depthTexture;if(W&&W.renderTarget===N&&G(W),N.isWebGLCubeRenderTarget)for(let ee=0;ee<6;ee++){if(Array.isArray(S.__webglFramebuffer[ee]))for(let ye=0;ye<S.__webglFramebuffer[ee].length;ye++)e.deleteFramebuffer(S.__webglFramebuffer[ee][ye]);else e.deleteFramebuffer(S.__webglFramebuffer[ee]);S.__webglDepthbuffer&&e.deleteRenderbuffer(S.__webglDepthbuffer[ee])}else{if(Array.isArray(S.__webglFramebuffer))for(let ee=0;ee<S.__webglFramebuffer.length;ee++)e.deleteFramebuffer(S.__webglFramebuffer[ee]);else e.deleteFramebuffer(S.__webglFramebuffer);if(S.__webglDepthbuffer&&e.deleteRenderbuffer(S.__webglDepthbuffer),S.__webglMultisampledFramebuffer&&e.deleteFramebuffer(S.__webglMultisampledFramebuffer),S.__webglColorRenderbuffer)for(let ee=0;ee<S.__webglColorRenderbuffer.length;ee++)S.__webglColorRenderbuffer[ee]&&e.deleteRenderbuffer(S.__webglColorRenderbuffer[ee]);S.__webglDepthRenderbuffer&&e.deleteRenderbuffer(S.__webglDepthRenderbuffer)}let ce=N.textures;for(let ee=0,ye=ce.length;ee<ye;ee++){let _e=r.get(ce[ee]);_e.__webglTexture&&(e.deleteTexture(_e.__webglTexture),s.memory.textures--),r.remove(ce[ee])}r.remove(N)}let j=0;function Y(){j=0}function ie(){return j}function Se(N){j=N}function K(){let N=j;return N>=a.maxTextures&&Je("WebGLTextures: Trying to use "+(N+1)+" texture units while this GPU supports only "+a.maxTextures),j+=1,N}function re(N){let S=[];return S.push(N.wrapS),S.push(N.wrapT),S.push(N.wrapR||0),S.push(N.magFilter),S.push(N.minFilter),S.push(N.anisotropy),S.push(N.internalFormat),S.push(N.format),S.push(N.type),S.push(N.generateMipmaps),S.push(N.mipmapsAutoUpdate),S.push(N.premultiplyAlpha),S.push(N.flipY),S.push(N.unpackAlignment),S.push(N.colorSpace),S.join()}function xe(N,S){let W=r.get(N);if(N.isVideoTexture&&Ge(N),N.isRenderTargetTexture===!1&&N.isExternalTexture!==!0&&N.version>0&&W.__version!==N.version){let ce=N.image;if(ce===null)Je("WebGLRenderer: Texture marked for update but no image data found.");else if(ce.complete===!1)Je("WebGLRenderer: Texture marked for update but image is incomplete");else{Ue(W,N,S);return}}else N.isExternalTexture&&(W.__webglTexture=N.sourceTexture?N.sourceTexture:null);i.bindTexture(e.TEXTURE_2D,W.__webglTexture,e.TEXTURE0+S)}function Ne(N,S){let W=r.get(N);N.isRenderTargetTexture===!1&&N.version>0&&W.__version!==N.version?Ue(W,N,S):(N.isExternalTexture&&(W.__webglTexture=N.sourceTexture?N.sourceTexture:null),i.bindTexture(e.TEXTURE_2D_ARRAY,W.__webglTexture,e.TEXTURE0+S))}function Me(N,S){let W=r.get(N);N.isRenderTargetTexture===!1&&N.version>0&&W.__version!==N.version?Ue(W,N,S):i.bindTexture(e.TEXTURE_3D,W.__webglTexture,e.TEXTURE0+S)}function Q(N,S){let W=r.get(N);N.version>0&&W.__version!==N.version?Ie(W,N,S):i.bindTexture(e.TEXTURE_CUBE_MAP,W.__webglTexture,e.TEXTURE0+S)}let ke={[Rs]:e.REPEAT,[Zi]:e.CLAMP_TO_EDGE,[Zn]:e.MIRRORED_REPEAT},$e={[oi]:e.NEAREST,[hd]:e.NEAREST_MIPMAP_NEAREST,[cd]:e.NEAREST_MIPMAP_LINEAR,[jt]:e.LINEAR,[ud]:e.LINEAR_MIPMAP_NEAREST,[Xa]:e.LINEAR_MIPMAP_LINEAR},De={512:e.NEVER,519:e.ALWAYS,513:e.LESS,515:e.LEQUAL,514:e.EQUAL,518:e.GEQUAL,516:e.GREATER,517:e.NOTEQUAL};function nt(N,S){if(S.type===1015&&t.has("OES_texture_float_linear")===!1&&(S.magFilter===1006||S.magFilter===1007||S.magFilter===1005||S.magFilter===1008||S.minFilter===1006||S.minFilter===1007||S.minFilter===1005||S.minFilter===1008)&&Je("WebGLRenderer: Unable to use linear filtering with floating point textures. OES_texture_float_linear not supported on this device."),e.texParameteri(N,e.TEXTURE_WRAP_S,ke[S.wrapS]),e.texParameteri(N,e.TEXTURE_WRAP_T,ke[S.wrapT]),(N===e.TEXTURE_3D||N===e.TEXTURE_2D_ARRAY)&&e.texParameteri(N,e.TEXTURE_WRAP_R,ke[S.wrapR]),e.texParameteri(N,e.TEXTURE_MAG_FILTER,$e[S.magFilter]),e.texParameteri(N,e.TEXTURE_MIN_FILTER,$e[S.minFilter]),S.compareFunction&&(e.texParameteri(N,e.TEXTURE_COMPARE_MODE,e.COMPARE_REF_TO_TEXTURE),e.texParameteri(N,e.TEXTURE_COMPARE_FUNC,De[S.compareFunction])),t.has("EXT_texture_filter_anisotropic")===!0){if(S.magFilter===1003||S.minFilter!==1005&&S.minFilter!==1008||S.type===1015&&t.has("OES_texture_float_linear")===!1)return;if(S.anisotropy>1||r.get(S).__currentAnisotropy){let W=t.get("EXT_texture_filter_anisotropic");e.texParameterf(N,W.TEXTURE_MAX_ANISOTROPY_EXT,Math.min(S.anisotropy,a.getMaxAnisotropy())),r.get(S).__currentAnisotropy=S.anisotropy}}}function oe(N,S){let W=!1;N.__webglInit===void 0&&(N.__webglInit=!0,S.addEventListener("dispose",R),A(S));let ce=S.source,ee=f.get(ce);ee===void 0&&(ee={},f.set(ce,ee));let ye=re(S);if(ye!==N.__cacheKey){ee[ye]===void 0&&(ee[ye]={texture:e.createTexture(),usedTimes:0},s.memory.textures++,W=!0),ee[ye].usedTimes++;let _e=ee[N.__cacheKey];_e!==void 0&&(ee[N.__cacheKey].usedTimes--,_e.usedTimes===0&&D(S)),N.__cacheKey=ye,N.__webglTexture=ee[ye].texture}return W}function he(N,S,W){return Math.floor(Math.floor(N/W)/S)}function de(N,S,W,ce){let ee=N.updateRanges;if(ee.length===0)i.texSubImage2D(e.TEXTURE_2D,0,0,0,S.width,S.height,W,ce,S.data);else{ee.sort((Pe,O)=>Pe.start-O.start);let ye=0;for(let Pe=1;Pe<ee.length;Pe++){let O=ee[ye],Z=ee[Pe],ue=O.start+O.count,me=he(Z.start,S.width,4),Ae=he(O.start,S.width,4);Z.start<=ue+1&&me===Ae&&he(Z.start+Z.count-1,S.width,4)===me?O.count=Math.max(O.count,Z.start+Z.count-O.start):(++ye,ee[ye]=Z)}ee.length=ye+1;let _e=i.getParameter(e.UNPACK_ROW_LENGTH),V=i.getParameter(e.UNPACK_SKIP_PIXELS),le=i.getParameter(e.UNPACK_SKIP_ROWS);i.pixelStorei(e.UNPACK_ROW_LENGTH,S.width);for(let Pe=0,O=ee.length;Pe<O;Pe++){let Z=ee[Pe],ue=Math.floor(Z.start/4),me=Math.ceil(Z.count/4),Ae=ue%S.width,Ee=Math.floor(ue/S.width),Ye=me;i.pixelStorei(e.UNPACK_SKIP_PIXELS,Ae),i.pixelStorei(e.UNPACK_SKIP_ROWS,Ee),i.texSubImage2D(e.TEXTURE_2D,0,Ae,Ee,Ye,1,W,ce,S.data)}N.clearUpdateRanges(),i.pixelStorei(e.UNPACK_ROW_LENGTH,_e),i.pixelStorei(e.UNPACK_SKIP_PIXELS,V),i.pixelStorei(e.UNPACK_SKIP_ROWS,le)}}function Ue(N,S,W){let ce=e.TEXTURE_2D;(S.isDataArrayTexture||S.isCompressedArrayTexture)&&(ce=e.TEXTURE_2D_ARRAY),S.isData3DTexture&&(ce=e.TEXTURE_3D);let ee=oe(N,S),ye=S.source;i.bindTexture(ce,N.__webglTexture,e.TEXTURE0+W);let _e=r.get(ye);if(ye.version!==_e.__version||ee===!0){if(i.activeTexture(e.TEXTURE0+W),!(typeof ImageBitmap<"u"&&S.image instanceof ImageBitmap)){let fe=ft.getPrimaries(ft.workingColorSpace),tt=S.colorSpace===""?null:ft.getPrimaries(S.colorSpace),Ze=S.colorSpace===""||fe===tt?e.NONE:e.BROWSER_DEFAULT_WEBGL;i.pixelStorei(e.UNPACK_FLIP_Y_WEBGL,S.flipY),i.pixelStorei(e.UNPACK_PREMULTIPLY_ALPHA_WEBGL,S.premultiplyAlpha),i.pixelStorei(e.UNPACK_COLORSPACE_CONVERSION_WEBGL,Ze)}i.pixelStorei(e.UNPACK_ALIGNMENT,S.unpackAlignment);let V=y(S.image,!1,a.maxTextureSize);V=je(S,V);let le=o.convert(S.format,S.colorSpace),Pe=o.convert(S.type),O=T(S.internalFormat,le,Pe,S.normalized,S.colorSpace,S.isVideoTexture);nt(ce,S);let Z,ue=S.mipmaps,me=S.isVideoTexture!==!0,Ae=_e.__version===void 0||ee===!0,Ee=ye.dataReady,Ye=_(S,V);if(S.isDepthTexture)O=v(S.format===Cs,S.type),Ae&&(me?i.texStorage2D(e.TEXTURE_2D,1,O,V.width,V.height):i.texImage2D(e.TEXTURE_2D,0,O,V.width,V.height,0,le,Pe,null));else if(S.isDataTexture)if(ue.length>0){me&&Ae&&i.texStorage2D(e.TEXTURE_2D,Ye,O,ue[0].width,ue[0].height);for(let fe=0,tt=ue.length;fe<tt;fe++)Z=ue[fe],me?Ee&&i.texSubImage2D(e.TEXTURE_2D,fe,0,0,Z.width,Z.height,le,Pe,Z.data):i.texImage2D(e.TEXTURE_2D,fe,O,Z.width,Z.height,0,le,Pe,Z.data);S.generateMipmaps=!1}else me?(Ae&&i.texStorage2D(e.TEXTURE_2D,Ye,O,V.width,V.height),Ee&&de(S,V,le,Pe)):i.texImage2D(e.TEXTURE_2D,0,O,V.width,V.height,0,le,Pe,V.data);else if(S.isCompressedTexture)if(S.isCompressedArrayTexture){me&&Ae&&i.texStorage3D(e.TEXTURE_2D_ARRAY,Ye,O,ue[0].width,ue[0].height,V.depth);for(let fe=0,tt=ue.length;fe<tt;fe++)if(Z=ue[fe],S.format!==1023)if(le!==null)if(me){if(Ee)if(S.layerUpdates.size>0){let Ze=Eh(Z.width,Z.height,S.format,S.type);for(let mt of S.layerUpdates){let Oe=Z.data.subarray(mt*Ze/Z.data.BYTES_PER_ELEMENT,(mt+1)*Ze/Z.data.BYTES_PER_ELEMENT);i.compressedTexSubImage3D(e.TEXTURE_2D_ARRAY,fe,0,0,mt,Z.width,Z.height,1,le,Oe)}}else i.compressedTexSubImage3D(e.TEXTURE_2D_ARRAY,fe,0,0,0,Z.width,Z.height,V.depth,le,Z.data)}else i.compressedTexImage3D(e.TEXTURE_2D_ARRAY,fe,O,Z.width,Z.height,V.depth,0,Z.data,0,0);else Je("WebGLRenderer: Attempt to load unsupported compressed texture format in .uploadTexture()");else me?Ee&&i.texSubImage3D(e.TEXTURE_2D_ARRAY,fe,0,0,0,Z.width,Z.height,V.depth,le,Pe,Z.data):i.texImage3D(e.TEXTURE_2D_ARRAY,fe,O,Z.width,Z.height,V.depth,0,le,Pe,Z.data);S.layerUpdates.size>0&&S.clearLayerUpdates()}else{me&&Ae&&i.texStorage2D(e.TEXTURE_2D,Ye,O,ue[0].width,ue[0].height);for(let fe=0,tt=ue.length;fe<tt;fe++)Z=ue[fe],S.format===1023?me?Ee&&i.texSubImage2D(e.TEXTURE_2D,fe,0,0,Z.width,Z.height,le,Pe,Z.data):i.texImage2D(e.TEXTURE_2D,fe,O,Z.width,Z.height,0,le,Pe,Z.data):le===null?Je("WebGLRenderer: Attempt to load unsupported compressed texture format in .uploadTexture()"):me?Ee&&i.compressedTexSubImage2D(e.TEXTURE_2D,fe,0,0,Z.width,Z.height,le,Z.data):i.compressedTexImage2D(e.TEXTURE_2D,fe,O,Z.width,Z.height,0,Z.data)}else if(S.isDataArrayTexture)if(me){if(Ae&&i.texStorage3D(e.TEXTURE_2D_ARRAY,Ye,O,V.width,V.height,V.depth),Ee)if(S.layerUpdates.size>0){let fe=Eh(V.width,V.height,S.format,S.type);for(let tt of S.layerUpdates){let Ze=V.data.subarray(tt*fe/V.data.BYTES_PER_ELEMENT,(tt+1)*fe/V.data.BYTES_PER_ELEMENT);i.texSubImage3D(e.TEXTURE_2D_ARRAY,0,0,0,tt,V.width,V.height,1,le,Pe,Ze)}S.clearLayerUpdates()}else i.texSubImage3D(e.TEXTURE_2D_ARRAY,0,0,0,0,V.width,V.height,V.depth,le,Pe,V.data)}else i.texImage3D(e.TEXTURE_2D_ARRAY,0,O,V.width,V.height,V.depth,0,le,Pe,V.data);else if(S.isData3DTexture)me?(Ae&&i.texStorage3D(e.TEXTURE_3D,Ye,O,V.width,V.height,V.depth),Ee&&i.texSubImage3D(e.TEXTURE_3D,0,0,0,0,V.width,V.height,V.depth,le,Pe,V.data)):i.texImage3D(e.TEXTURE_3D,0,O,V.width,V.height,V.depth,0,le,Pe,V.data);else if(S.isFramebufferTexture){if(Ae)if(me)i.texStorage2D(e.TEXTURE_2D,Ye,O,V.width,V.height);else{let fe=V.width,tt=V.height;for(let Ze=0;Ze<Ye;Ze++)i.texImage2D(e.TEXTURE_2D,Ze,O,fe,tt,0,le,Pe,null),fe>>=1,tt>>=1}}else if(S.isHTMLTexture){if("texElementImage2D"in e){let fe=e.canvas;if(fe.hasAttribute("layoutsubtree")||fe.setAttribute("layoutsubtree","true"),V.parentNode!==fe){fe.appendChild(V),u.add(S),fe.onpaint=tt=>{let Ze=tt.changedElements;for(let mt of u)Ze.includes(mt.image)&&(mt.needsUpdate=!0)},fe.requestPaint();return}if(e.texElementImage2D.length===3)e.texElementImage2D(e.TEXTURE_2D,e.RGBA8,V);else{let tt=e.RGBA,Ze=e.RGBA,mt=e.UNSIGNED_BYTE;e.texElementImage2D(e.TEXTURE_2D,0,tt,Ze,mt,V)}e.texParameteri(e.TEXTURE_2D,e.TEXTURE_MIN_FILTER,e.LINEAR),e.texParameteri(e.TEXTURE_2D,e.TEXTURE_WRAP_S,e.CLAMP_TO_EDGE),e.texParameteri(e.TEXTURE_2D,e.TEXTURE_WRAP_T,e.CLAMP_TO_EDGE)}}else if(ue.length>0){if(me&&Ae){let fe=At(ue[0]);i.texStorage2D(e.TEXTURE_2D,Ye,O,fe.width,fe.height)}for(let fe=0,tt=ue.length;fe<tt;fe++)Z=ue[fe],me?Ee&&i.texSubImage2D(e.TEXTURE_2D,fe,0,0,le,Pe,Z):i.texImage2D(e.TEXTURE_2D,fe,O,le,Pe,Z);S.generateMipmaps=!1}else if(me){if(Ae){let fe=At(V);i.texStorage2D(e.TEXTURE_2D,Ye,O,fe.width,fe.height)}Ee&&i.texSubImage2D(e.TEXTURE_2D,0,0,0,le,Pe,V)}else i.texImage2D(e.TEXTURE_2D,0,O,le,Pe,V);M(S)&&S.mipmapsAutoUpdate===!0&&b(ce),_e.__version=ye.version,S.onUpdate&&S.onUpdate(S)}N.__version=S.version}function Ie(N,S,W){if(S.image.length!==6)return;let ce=oe(N,S),ee=S.source;i.bindTexture(e.TEXTURE_CUBE_MAP,N.__webglTexture,e.TEXTURE0+W);let ye=r.get(ee);if(ee.version!==ye.__version||ce===!0){i.activeTexture(e.TEXTURE0+W);let _e=ft.getPrimaries(ft.workingColorSpace),V=S.colorSpace===""?null:ft.getPrimaries(S.colorSpace),le=S.colorSpace===""||_e===V?e.NONE:e.BROWSER_DEFAULT_WEBGL;i.pixelStorei(e.UNPACK_FLIP_Y_WEBGL,S.flipY),i.pixelStorei(e.UNPACK_PREMULTIPLY_ALPHA_WEBGL,S.premultiplyAlpha),i.pixelStorei(e.UNPACK_ALIGNMENT,S.unpackAlignment),i.pixelStorei(e.UNPACK_COLORSPACE_CONVERSION_WEBGL,le);let Pe=S.isCompressedTexture||S.image[0].isCompressedTexture,O=S.image[0]&&S.image[0].isDataTexture,Z=[];for(let Oe=0;Oe<6;Oe++)!Pe&&!O?Z[Oe]=y(S.image[Oe],!0,a.maxCubemapSize):Z[Oe]=O?S.image[Oe].image:S.image[Oe],Z[Oe]=je(S,Z[Oe]);let ue=Z[0],me=o.convert(S.format,S.colorSpace),Ae=o.convert(S.type),Ee=T(S.internalFormat,me,Ae,S.normalized,S.colorSpace),Ye=S.isVideoTexture!==!0,fe=ye.__version===void 0||ce===!0,tt=ee.dataReady,Ze=_(S,ue);nt(e.TEXTURE_CUBE_MAP,S);let mt;if(S.isDepthTexture){if(Ee=v(S.format===Cs,S.type),fe)if(Ye)i.texStorage2D(e.TEXTURE_CUBE_MAP,1,Ee,ue.width,ue.height);else for(let Oe=0;Oe<6;Oe++)i.texImage2D(e.TEXTURE_CUBE_MAP_POSITIVE_X+Oe,0,Ee,ue.width,ue.height,0,me,Ae,null)}else if(Pe){Ye&&fe&&i.texStorage2D(e.TEXTURE_CUBE_MAP,Ze,Ee,ue.width,ue.height);for(let Oe=0;Oe<6;Oe++){mt=Z[Oe].mipmaps;for(let Ot=0;Ot<mt.length;Ot++){let zt=mt[Ot];S.format===1023?Ye?tt&&i.texSubImage2D(e.TEXTURE_CUBE_MAP_POSITIVE_X+Oe,Ot,0,0,zt.width,zt.height,me,Ae,zt.data):i.texImage2D(e.TEXTURE_CUBE_MAP_POSITIVE_X+Oe,Ot,Ee,zt.width,zt.height,0,me,Ae,zt.data):me===null?Je("WebGLRenderer: Attempt to load unsupported compressed texture format in .setTextureCube()"):Ye?tt&&i.compressedTexSubImage2D(e.TEXTURE_CUBE_MAP_POSITIVE_X+Oe,Ot,0,0,zt.width,zt.height,me,zt.data):i.compressedTexImage2D(e.TEXTURE_CUBE_MAP_POSITIVE_X+Oe,Ot,Ee,zt.width,zt.height,0,zt.data)}}}else{if(mt=S.mipmaps,Ye&&fe){mt.length>0&&Ze++;let Oe=At(Z[0]);i.texStorage2D(e.TEXTURE_CUBE_MAP,Ze,Ee,Oe.width,Oe.height)}for(let Oe=0;Oe<6;Oe++)if(O){Ye?tt&&i.texSubImage2D(e.TEXTURE_CUBE_MAP_POSITIVE_X+Oe,0,0,0,Z[Oe].width,Z[Oe].height,me,Ae,Z[Oe].data):i.texImage2D(e.TEXTURE_CUBE_MAP_POSITIVE_X+Oe,0,Ee,Z[Oe].width,Z[Oe].height,0,me,Ae,Z[Oe].data);for(let Ot=0;Ot<mt.length;Ot++){let zt=mt[Ot].image[Oe].image;Ye?tt&&i.texSubImage2D(e.TEXTURE_CUBE_MAP_POSITIVE_X+Oe,Ot+1,0,0,zt.width,zt.height,me,Ae,zt.data):i.texImage2D(e.TEXTURE_CUBE_MAP_POSITIVE_X+Oe,Ot+1,Ee,zt.width,zt.height,0,me,Ae,zt.data)}}else{Ye?tt&&i.texSubImage2D(e.TEXTURE_CUBE_MAP_POSITIVE_X+Oe,0,0,0,me,Ae,Z[Oe]):i.texImage2D(e.TEXTURE_CUBE_MAP_POSITIVE_X+Oe,0,Ee,me,Ae,Z[Oe]);for(let Ot=0;Ot<mt.length;Ot++){let zt=mt[Ot];Ye?tt&&i.texSubImage2D(e.TEXTURE_CUBE_MAP_POSITIVE_X+Oe,Ot+1,0,0,me,Ae,zt.image[Oe]):i.texImage2D(e.TEXTURE_CUBE_MAP_POSITIVE_X+Oe,Ot+1,Ee,me,Ae,zt.image[Oe])}}}M(S)&&S.mipmapsAutoUpdate===!0&&b(e.TEXTURE_CUBE_MAP),ye.__version=ee.version,S.onUpdate&&S.onUpdate(S)}N.__version=S.version}function F(N,S,W,ce,ee,ye){let _e=o.convert(W.format,W.colorSpace),V=o.convert(W.type),le=T(W.internalFormat,_e,V,W.normalized,W.colorSpace),Pe=r.get(S),O=r.get(W);if(O.__renderTarget=S,!Pe.__hasExternalTextures){let Z=Math.max(1,S.width>>ye),ue=Math.max(1,S.height>>ye);ee===e.TEXTURE_3D||ee===e.TEXTURE_2D_ARRAY?i.texImage3D(ee,ye,le,Z,ue,S.depth,0,_e,V,null):i.texImage2D(ee,ye,le,Z,ue,0,_e,V,null)}i.bindFramebuffer(e.FRAMEBUFFER,N),lt(S)?n.framebufferTexture2DMultisampleEXT(e.FRAMEBUFFER,ce,ee,O.__webglTexture,0,ne(S)):(ee===e.TEXTURE_2D||ee>=e.TEXTURE_CUBE_MAP_POSITIVE_X&&ee<=e.TEXTURE_CUBE_MAP_NEGATIVE_Z)&&e.framebufferTexture2D(e.FRAMEBUFFER,ce,ee,O.__webglTexture,ye),i.bindFramebuffer(e.FRAMEBUFFER,null)}function te(N,S,W){if(e.bindRenderbuffer(e.RENDERBUFFER,N),S.depthBuffer){let ce=S.depthTexture,ee=ce&&ce.isDepthTexture?ce.type:null,ye=v(S.stencilBuffer,ee),_e=S.stencilBuffer?e.DEPTH_STENCIL_ATTACHMENT:e.DEPTH_ATTACHMENT;lt(S)?n.renderbufferStorageMultisampleEXT(e.RENDERBUFFER,ne(S),ye,S.width,S.height):W?e.renderbufferStorageMultisample(e.RENDERBUFFER,ne(S),ye,S.width,S.height):e.renderbufferStorage(e.RENDERBUFFER,ye,S.width,S.height),e.framebufferRenderbuffer(e.FRAMEBUFFER,_e,e.RENDERBUFFER,N)}else{let ce=S.textures;for(let ee=0;ee<ce.length;ee++){let ye=ce[ee],_e=o.convert(ye.format,ye.colorSpace),V=o.convert(ye.type),le=T(ye.internalFormat,_e,V,ye.normalized,ye.colorSpace);lt(S)?n.renderbufferStorageMultisampleEXT(e.RENDERBUFFER,ne(S),le,S.width,S.height):W?e.renderbufferStorageMultisample(e.RENDERBUFFER,ne(S),le,S.width,S.height):e.renderbufferStorage(e.RENDERBUFFER,le,S.width,S.height)}}e.bindRenderbuffer(e.RENDERBUFFER,null)}function $(N,S,W){let ce=S.isWebGLCubeRenderTarget===!0;if(i.bindFramebuffer(e.FRAMEBUFFER,N),!(S.depthTexture&&S.depthTexture.isDepthTexture))throw Error("THREE.WebGLTextures: renderTarget.depthTexture must be an instance of THREE.DepthTexture.");let ee=r.get(S.depthTexture);ee.__renderTarget=S;let ye=ce?S.depthTexture.image[0]:S.depthTexture.image;ee.__webglTexture&&ye.width===S.width&&ye.height===S.height||(ye.width=S.width,ye.height=S.height,S.depthTexture.needsUpdate=!0),ce?Q(S.depthTexture,0):xe(S.depthTexture,0);let _e=ee.__webglTexture,V=ne(S),le=ce?e.TEXTURE_CUBE_MAP_POSITIVE_X+W:e.TEXTURE_2D,Pe=S.depthTexture.format===1027?e.DEPTH_STENCIL_ATTACHMENT:e.DEPTH_ATTACHMENT;if(S.depthTexture.format===1026)lt(S)?n.framebufferTexture2DMultisampleEXT(e.FRAMEBUFFER,Pe,le,_e,0,V):e.framebufferTexture2D(e.FRAMEBUFFER,Pe,le,_e,0);else if(S.depthTexture.format===1027)lt(S)?n.framebufferTexture2DMultisampleEXT(e.FRAMEBUFFER,Pe,le,_e,0,V):e.framebufferTexture2D(e.FRAMEBUFFER,Pe,le,_e,0);else throw Error("THREE.WebGLTextures: Unknown depthTexture format.")}function Te(N){let S=r.get(N),W=N.isWebGLCubeRenderTarget===!0;if(S.__boundDepthTexture!==N.depthTexture){let ce=N.depthTexture;if(S.__depthDisposeCallback&&S.__depthDisposeCallback(),ce){let ee=()=>{delete S.__boundDepthTexture,delete S.__depthDisposeCallback,ce.removeEventListener("dispose",ee)};ce.addEventListener("dispose",ee),S.__depthDisposeCallback=ee}S.__boundDepthTexture=ce}if(N.depthTexture&&!S.__autoAllocateDepthBuffer)if(W)for(let ce=0;ce<6;ce++)$(S.__webglFramebuffer[ce],N,ce);else N.texture.mipmaps.length>0?$(S.__webglFramebuffer[0],N,0):$(S.__webglFramebuffer,N,0);else if(W){S.__webglDepthbuffer=[];for(let ce=0;ce<6;ce++)if(i.bindFramebuffer(e.FRAMEBUFFER,S.__webglFramebuffer[ce]),S.__webglDepthbuffer[ce]===void 0)S.__webglDepthbuffer[ce]=e.createRenderbuffer(),te(S.__webglDepthbuffer[ce],N,!1);else{let ee=N.stencilBuffer?e.DEPTH_STENCIL_ATTACHMENT:e.DEPTH_ATTACHMENT,ye=S.__webglDepthbuffer[ce];e.bindRenderbuffer(e.RENDERBUFFER,ye),e.framebufferRenderbuffer(e.FRAMEBUFFER,ee,e.RENDERBUFFER,ye)}}else if(N.texture.mipmaps.length>0?i.bindFramebuffer(e.FRAMEBUFFER,S.__webglFramebuffer[0]):i.bindFramebuffer(e.FRAMEBUFFER,S.__webglFramebuffer),S.__webglDepthbuffer===void 0)S.__webglDepthbuffer=e.createRenderbuffer(),te(S.__webglDepthbuffer,N,!1);else{let ce=N.stencilBuffer?e.DEPTH_STENCIL_ATTACHMENT:e.DEPTH_ATTACHMENT,ee=S.__webglDepthbuffer;e.bindRenderbuffer(e.RENDERBUFFER,ee),e.framebufferRenderbuffer(e.FRAMEBUFFER,ce,e.RENDERBUFFER,ee)}i.bindFramebuffer(e.FRAMEBUFFER,null)}function Be(N,S,W){let ce=r.get(N);S!==void 0&&F(ce.__webglFramebuffer,N,N.texture,e.COLOR_ATTACHMENT0,e.TEXTURE_2D,0),W!==void 0&&Te(N)}function U(N){let S=N.texture,W=r.get(N),ce=r.get(S);N.addEventListener("dispose",L),A(N);let ee=N.textures,ye=N.isWebGLCubeRenderTarget===!0,_e=ee.length>1;if(_e||(ce.__webglTexture===void 0&&(ce.__webglTexture=e.createTexture()),ce.__version=S.version,s.memory.textures++),ye){W.__webglFramebuffer=[];for(let V=0;V<6;V++)if(S.mipmaps.length>0){W.__webglFramebuffer[V]=[];for(let le=0;le<S.mipmaps.length;le++)W.__webglFramebuffer[V][le]=e.createFramebuffer()}else W.__webglFramebuffer[V]=e.createFramebuffer()}else{if(S.mipmaps.length>0){W.__webglFramebuffer=[];for(let V=0;V<S.mipmaps.length;V++)W.__webglFramebuffer[V]=e.createFramebuffer()}else W.__webglFramebuffer=e.createFramebuffer();if(_e)for(let V=0,le=ee.length;V<le;V++){let Pe=r.get(ee[V]);Pe.__webglTexture===void 0&&(Pe.__webglTexture=e.createTexture(),s.memory.textures++)}if(N.samples>0&&lt(N)===!1){W.__webglMultisampledFramebuffer=e.createFramebuffer(),W.__webglColorRenderbuffer=[],i.bindFramebuffer(e.FRAMEBUFFER,W.__webglMultisampledFramebuffer);for(let V=0;V<ee.length;V++){let le=ee[V];W.__webglColorRenderbuffer[V]=e.createRenderbuffer(),e.bindRenderbuffer(e.RENDERBUFFER,W.__webglColorRenderbuffer[V]);let Pe=o.convert(le.format,le.colorSpace),O=o.convert(le.type),Z=T(le.internalFormat,Pe,O,le.normalized,le.colorSpace,N.isXRRenderTarget===!0),ue=ne(N);e.renderbufferStorageMultisample(e.RENDERBUFFER,ue,Z,N.width,N.height),e.framebufferRenderbuffer(e.FRAMEBUFFER,e.COLOR_ATTACHMENT0+V,e.RENDERBUFFER,W.__webglColorRenderbuffer[V])}e.bindRenderbuffer(e.RENDERBUFFER,null),N.depthBuffer&&(W.__webglDepthRenderbuffer=e.createRenderbuffer(),te(W.__webglDepthRenderbuffer,N,!0)),i.bindFramebuffer(e.FRAMEBUFFER,null)}}if(ye){i.bindTexture(e.TEXTURE_CUBE_MAP,ce.__webglTexture),nt(e.TEXTURE_CUBE_MAP,S);for(let V=0;V<6;V++)if(S.mipmaps.length>0)for(let le=0;le<S.mipmaps.length;le++)F(W.__webglFramebuffer[V][le],N,S,e.COLOR_ATTACHMENT0,e.TEXTURE_CUBE_MAP_POSITIVE_X+V,le);else F(W.__webglFramebuffer[V],N,S,e.COLOR_ATTACHMENT0,e.TEXTURE_CUBE_MAP_POSITIVE_X+V,0);M(S)?b(e.TEXTURE_CUBE_MAP):S.mipmaps.length>0&&e.texParameteri(e.TEXTURE_CUBE_MAP,e.TEXTURE_MAX_LEVEL,S.mipmaps.length-1),i.unbindTexture()}else if(_e){for(let V=0,le=ee.length;V<le;V++){let Pe=ee[V],O=r.get(Pe),Z=e.TEXTURE_2D;(N.isWebGL3DRenderTarget||N.isWebGLArrayRenderTarget)&&(Z=N.isWebGL3DRenderTarget?e.TEXTURE_3D:e.TEXTURE_2D_ARRAY),i.bindTexture(Z,O.__webglTexture),nt(Z,Pe),F(W.__webglFramebuffer,N,Pe,e.COLOR_ATTACHMENT0+V,Z,0),M(Pe)&&b(Z)}i.unbindTexture()}else{let V=e.TEXTURE_2D;if((N.isWebGL3DRenderTarget||N.isWebGLArrayRenderTarget)&&(V=N.isWebGL3DRenderTarget?e.TEXTURE_3D:e.TEXTURE_2D_ARRAY),i.bindTexture(V,ce.__webglTexture),nt(V,S),S.mipmaps.length>0)for(let le=0;le<S.mipmaps.length;le++)F(W.__webglFramebuffer[le],N,S,e.COLOR_ATTACHMENT0,V,le);else F(W.__webglFramebuffer,N,S,e.COLOR_ATTACHMENT0,V,0);M(S)?b(V):S.mipmaps.length>0&&e.texParameteri(V,e.TEXTURE_MAX_LEVEL,S.mipmaps.length-1),i.unbindTexture()}N.depthBuffer&&Te(N)}function ut(N){let S=N.textures;for(let W=0,ce=S.length;W<ce;W++){let ee=S[W];if(M(ee)&&ee.mipmapsAutoUpdate===!0){let ye=E(N),_e=r.get(ee).__webglTexture;i.bindTexture(ye,_e),b(ye),i.unbindTexture()}}}let B=[];function Ce(N){if(N.samples>0){if(lt(N)===!1){let S=N.textures,W=N.width,ce=N.height,ee=e.COLOR_BUFFER_BIT,ye=N.stencilBuffer?e.DEPTH_STENCIL_ATTACHMENT:e.DEPTH_ATTACHMENT,_e=r.get(N),V=S.length>1;if(V)for(let le=0;le<S.length;le++)i.bindFramebuffer(e.FRAMEBUFFER,_e.__webglMultisampledFramebuffer),e.framebufferRenderbuffer(e.FRAMEBUFFER,e.COLOR_ATTACHMENT0+le,e.RENDERBUFFER,null),i.bindFramebuffer(e.FRAMEBUFFER,_e.__webglFramebuffer),e.framebufferTexture2D(e.DRAW_FRAMEBUFFER,e.COLOR_ATTACHMENT0+le,e.TEXTURE_2D,null,0);i.bindFramebuffer(e.READ_FRAMEBUFFER,_e.__webglMultisampledFramebuffer),N.texture.mipmaps.length>0?i.bindFramebuffer(e.DRAW_FRAMEBUFFER,_e.__webglFramebuffer[0]):i.bindFramebuffer(e.DRAW_FRAMEBUFFER,_e.__webglFramebuffer);for(let le=0;le<S.length;le++){if(N.resolveDepthBuffer&&(N.depthBuffer&&(ee|=e.DEPTH_BUFFER_BIT),N.stencilBuffer&&N.resolveStencilBuffer&&(ee|=e.STENCIL_BUFFER_BIT)),V){e.framebufferRenderbuffer(e.READ_FRAMEBUFFER,e.COLOR_ATTACHMENT0,e.RENDERBUFFER,_e.__webglColorRenderbuffer[le]);let Pe=r.get(S[le]).__webglTexture;e.framebufferTexture2D(e.DRAW_FRAMEBUFFER,e.COLOR_ATTACHMENT0,e.TEXTURE_2D,Pe,0)}e.blitFramebuffer(0,0,W,ce,0,0,W,ce,ee,e.NEAREST),l===!0&&(B.length=0,B.push(e.COLOR_ATTACHMENT0+le),N.depthBuffer&&N.storeMultisampledDepthBuffer===!1&&B.push(ye),e.invalidateFramebuffer(e.READ_FRAMEBUFFER,B))}if(i.bindFramebuffer(e.READ_FRAMEBUFFER,null),i.bindFramebuffer(e.DRAW_FRAMEBUFFER,null),V)for(let le=0;le<S.length;le++){i.bindFramebuffer(e.FRAMEBUFFER,_e.__webglMultisampledFramebuffer),e.framebufferRenderbuffer(e.FRAMEBUFFER,e.COLOR_ATTACHMENT0+le,e.RENDERBUFFER,_e.__webglColorRenderbuffer[le]);let Pe=r.get(S[le]).__webglTexture;i.bindFramebuffer(e.FRAMEBUFFER,_e.__webglFramebuffer),e.framebufferTexture2D(e.DRAW_FRAMEBUFFER,e.COLOR_ATTACHMENT0+le,e.TEXTURE_2D,Pe,0)}i.bindFramebuffer(e.DRAW_FRAMEBUFFER,_e.__webglMultisampledFramebuffer)}else if(N.depthBuffer&&N.storeMultisampledDepthBuffer===!1&&l){let S=N.stencilBuffer?e.DEPTH_STENCIL_ATTACHMENT:e.DEPTH_ATTACHMENT;e.invalidateFramebuffer(e.DRAW_FRAMEBUFFER,[S])}}}function ne(N){return Math.min(a.maxSamples,N.samples)}function lt(N){let S=r.get(N);return N.samples>0&&t.has("WEBGL_multisampled_render_to_texture")===!0&&S.__useRenderToTexture!==!1}function Ge(N){let S=s.render.frame;c.get(N)!==S&&(c.set(N,S),N.update())}function je(N,S){let W=N.colorSpace,ce=N.format,ee=N.type;return N.isCompressedTexture===!0||N.isVideoTexture===!0||W!=="srgb-linear"&&W!==""&&(ft.getTransfer(W)==="srgb"?(ce!==1023||ee!==1009)&&Je("WebGLTextures: sRGB encoded textures have to use RGBAFormat and UnsignedByteType."):Qe("WebGLTextures: Unsupported texture color space:",W)),S}function At(N){return typeof HTMLImageElement<"u"&&N instanceof HTMLImageElement?(h.width=N.naturalWidth||N.width,h.height=N.naturalHeight||N.height):typeof VideoFrame<"u"&&N instanceof VideoFrame?(h.width=N.displayWidth,h.height=N.displayHeight):(h.width=N.width,h.height=N.height),h}this.allocateTextureUnit=K,this.resetTextureUnits=Y,this.getTextureUnits=ie,this.setTextureUnits=Se,this.setTexture2D=xe,this.setTexture2DArray=Ne,this.setTexture3D=Me,this.setTextureCube=Q,this.rebindTextures=Be,this.setupRenderTarget=U,this.updateRenderTargetMipmap=ut,this.updateMultisampleRenderTarget=Ce,this.setupDepthRenderbuffer=Te,this.setupFrameBufferTexture=F,this.useMultisampledRTT=lt,this.dispose=z}function Qm(e,t){function i(r,a=""){let o,s=ft.getTransfer(a);if(r===1009)return e.UNSIGNED_BYTE;if(r===1017)return e.UNSIGNED_SHORT_4_4_4_4;if(r===1018)return e.UNSIGNED_SHORT_5_5_5_1;if(r===35902)return e.UNSIGNED_INT_5_9_9_9_REV;if(r===35899)return e.UNSIGNED_INT_10F_11F_11F_REV;if(r===1010)return e.BYTE;if(r===1011)return e.SHORT;if(r===1012)return e.UNSIGNED_SHORT;if(r===1013)return e.INT;if(r===1014)return e.UNSIGNED_INT;if(r===1015)return e.FLOAT;if(r===1016)return e.HALF_FLOAT;if(r===1021)return e.ALPHA;if(r===1022)return e.RGB;if(r===1023)return e.RGBA;if(r===1026)return e.DEPTH_COMPONENT;if(r===1027)return e.DEPTH_STENCIL;if(r===1028)return e.RED;if(r===1029)return e.RED_INTEGER;if(r===1030)return e.RG;if(r===1031)return e.RG_INTEGER;if(r===1033)return e.RGBA_INTEGER;if(r===33776||r===33777||r===33778||r===33779)if(s==="srgb")if(o=t.get("WEBGL_compressed_texture_s3tc_srgb"),o!==null){if(r===33776)return o.COMPRESSED_SRGB_S3TC_DXT1_EXT;if(r===33777)return o.COMPRESSED_SRGB_ALPHA_S3TC_DXT1_EXT;if(r===33778)return o.COMPRESSED_SRGB_ALPHA_S3TC_DXT3_EXT;if(r===33779)return o.COMPRESSED_SRGB_ALPHA_S3TC_DXT5_EXT}else return null;else if(o=t.get("WEBGL_compressed_texture_s3tc"),o!==null){if(r===33776)return o.COMPRESSED_RGB_S3TC_DXT1_EXT;if(r===33777)return o.COMPRESSED_RGBA_S3TC_DXT1_EXT;if(r===33778)return o.COMPRESSED_RGBA_S3TC_DXT3_EXT;if(r===33779)return o.COMPRESSED_RGBA_S3TC_DXT5_EXT}else return null;if(r===35840||r===35841||r===35842||r===35843)if(o=t.get("WEBGL_compressed_texture_pvrtc"),o!==null){if(r===35840)return o.COMPRESSED_RGB_PVRTC_4BPPV1_IMG;if(r===35841)return o.COMPRESSED_RGB_PVRTC_2BPPV1_IMG;if(r===35842)return o.COMPRESSED_RGBA_PVRTC_4BPPV1_IMG;if(r===35843)return o.COMPRESSED_RGBA_PVRTC_2BPPV1_IMG}else return null;if(r===36196||r===37492||r===37496||r===37488||r===37489||r===37490||r===37491)if(o=t.get("WEBGL_compressed_texture_etc"),o!==null){if(r===36196||r===37492)return s==="srgb"?o.COMPRESSED_SRGB8_ETC2:o.COMPRESSED_RGB8_ETC2;if(r===37496)return s==="srgb"?o.COMPRESSED_SRGB8_ALPHA8_ETC2_EAC:o.COMPRESSED_RGBA8_ETC2_EAC;if(r===37488)return o.COMPRESSED_R11_EAC;if(r===37489)return o.COMPRESSED_SIGNED_R11_EAC;if(r===37490)return o.COMPRESSED_RG11_EAC;if(r===37491)return o.COMPRESSED_SIGNED_RG11_EAC}else return null;if(r===37808||r===37809||r===37810||r===37811||r===37812||r===37813||r===37814||r===37815||r===37816||r===37817||r===37818||r===37819||r===37820||r===37821)if(o=t.get("WEBGL_compressed_texture_astc"),o!==null){if(r===37808)return s==="srgb"?o.COMPRESSED_SRGB8_ALPHA8_ASTC_4x4_KHR:o.COMPRESSED_RGBA_ASTC_4x4_KHR;if(r===37809)return s==="srgb"?o.COMPRESSED_SRGB8_ALPHA8_ASTC_5x4_KHR:o.COMPRESSED_RGBA_ASTC_5x4_KHR;if(r===37810)return s==="srgb"?o.COMPRESSED_SRGB8_ALPHA8_ASTC_5x5_KHR:o.COMPRESSED_RGBA_ASTC_5x5_KHR;if(r===37811)return s==="srgb"?o.COMPRESSED_SRGB8_ALPHA8_ASTC_6x5_KHR:o.COMPRESSED_RGBA_ASTC_6x5_KHR;if(r===37812)return s==="srgb"?o.COMPRESSED_SRGB8_ALPHA8_ASTC_6x6_KHR:o.COMPRESSED_RGBA_ASTC_6x6_KHR;if(r===37813)return s==="srgb"?o.COMPRESSED_SRGB8_ALPHA8_ASTC_8x5_KHR:o.COMPRESSED_RGBA_ASTC_8x5_KHR;if(r===37814)return s==="srgb"?o.COMPRESSED_SRGB8_ALPHA8_ASTC_8x6_KHR:o.COMPRESSED_RGBA_ASTC_8x6_KHR;if(r===37815)return s==="srgb"?o.COMPRESSED_SRGB8_ALPHA8_ASTC_8x8_KHR:o.COMPRESSED_RGBA_ASTC_8x8_KHR;if(r===37816)return s==="srgb"?o.COMPRESSED_SRGB8_ALPHA8_ASTC_10x5_KHR:o.COMPRESSED_RGBA_ASTC_10x5_KHR;if(r===37817)return s==="srgb"?o.COMPRESSED_SRGB8_ALPHA8_ASTC_10x6_KHR:o.COMPRESSED_RGBA_ASTC_10x6_KHR;if(r===37818)return s==="srgb"?o.COMPRESSED_SRGB8_ALPHA8_ASTC_10x8_KHR:o.COMPRESSED_RGBA_ASTC_10x8_KHR;if(r===37819)return s==="srgb"?o.COMPRESSED_SRGB8_ALPHA8_ASTC_10x10_KHR:o.COMPRESSED_RGBA_ASTC_10x10_KHR;if(r===37820)return s==="srgb"?o.COMPRESSED_SRGB8_ALPHA8_ASTC_12x10_KHR:o.COMPRESSED_RGBA_ASTC_12x10_KHR;if(r===37821)return s==="srgb"?o.COMPRESSED_SRGB8_ALPHA8_ASTC_12x12_KHR:o.COMPRESSED_RGBA_ASTC_12x12_KHR}else return null;if(r===36492||r===36494||r===36495)if(o=t.get("EXT_texture_compression_bptc"),o!==null){if(r===36492)return s==="srgb"?o.COMPRESSED_SRGB_ALPHA_BPTC_UNORM_EXT:o.COMPRESSED_RGBA_BPTC_UNORM_EXT;if(r===36494)return o.COMPRESSED_RGB_BPTC_SIGNED_FLOAT_EXT;if(r===36495)return o.COMPRESSED_RGB_BPTC_UNSIGNED_FLOAT_EXT}else return null;if(r===36283||r===36284||r===36285||r===36286)if(o=t.get("EXT_texture_compression_rgtc"),o!==null){if(r===36283)return o.COMPRESSED_RED_RGTC1_EXT;if(r===36284)return o.COMPRESSED_SIGNED_RED_RGTC1_EXT;if(r===36285)return o.COMPRESSED_RED_GREEN_RGTC2_EXT;if(r===36286)return o.COMPRESSED_SIGNED_RED_GREEN_RGTC2_EXT}else return null;return r===1020?e.UNSIGNED_INT_24_8:e[r]===void 0?null:e[r]}return{convert:i}}var e1=`
void main() {

	gl_Position = vec4( position, 1.0 );

}`,t1=`
uniform sampler2DArray depthColor;
uniform float depthWidth;
uniform float depthHeight;

void main() {

	vec2 coord = vec2( gl_FragCoord.x / depthWidth, gl_FragCoord.y / depthHeight );

	if ( coord.x >= 1.0 ) {

		gl_FragDepth = texture( depthColor, vec3( coord.x - 1.0, coord.y, 1 ) ).r;

	} else {

		gl_FragDepth = texture( depthColor, vec3( coord.x, coord.y, 0 ) ).r;

	}

}`,i1=class{constructor(){this.texture=null,this.mesh=null,this.depthNear=0,this.depthFar=0}init(e,t){if(this.texture===null){let i=new Kc(e.texture);(e.depthNear!==t.depthNear||e.depthFar!==t.depthFar)&&(this.depthNear=e.depthNear,this.depthFar=e.depthFar),this.texture=i}}getMesh(e){if(this.texture!==null&&this.mesh===null){let t=e.cameras[0].viewport,i=new $i({vertexShader:e1,fragmentShader:t1,uniforms:{depthColor:{value:this.texture},depthWidth:{value:t.z},depthHeight:{value:t.w}}});this.mesh=new Nt(new Br(20,20),i)}return this.mesh}reset(){this.texture=null,this.mesh=null}getDepthTexture(){return this.texture}},r1=class extends na{constructor(e,t){super();let i=this,r=null,a=1,o=null,s="local-floor",n=1,l=null,h=null,c=null,u=null,d=null,f=null,p=typeof XRWebGLBinding<"u",x=new i1,g={},m=t.getContextAttributes(),y=null,M=null,b=[],E=[],T=new ge,v=null,_=null,A=new ji;A.viewport=new Ft;let C=new ji;C.viewport=new Ft;let R=[A,C],L=new Ff,G=null,P=null;this.cameraAutoUpdate=!0,this.enabled=!1,this.isPresenting=!1,this.getController=function(K){let re=b[K];return re===void 0&&(re=new dn,b[K]=re),re.getTargetRaySpace()},this.getControllerGrip=function(K){let re=b[K];return re===void 0&&(re=new dn,b[K]=re),re.getGripSpace()},this.getHand=function(K){let re=b[K];return re===void 0&&(re=new dn,b[K]=re),re.getHandSpace()};function z(K){let re=E.indexOf(K.inputSource);if(re===-1)return;let xe=b[re];xe!==void 0&&(xe.update(K.inputSource,K.frame,l||o),xe.dispatchEvent({type:K.type,data:K.inputSource}))}function H(){r.removeEventListener("select",z),r.removeEventListener("selectstart",z),r.removeEventListener("selectend",z),r.removeEventListener("squeeze",z),r.removeEventListener("squeezestart",z),r.removeEventListener("squeezeend",z),r.removeEventListener("end",H),r.removeEventListener("inputsourceschange",D);for(let K=0;K<b.length;K++){let re=E[K];re!==null&&(E[K]=null,b[K].disconnect(re))}G=null,P=null,x.reset();for(let K in g)delete g[K];if(e.setRenderTarget(y),d=null,u=null,c=null,r=null,M=null,Se.stop(),i.isPresenting=!1,e.setPixelRatio(v),e.setSize(T.width,T.height,!1),_!==null){let K=_.camera;K.fov=_.fov,K.zoom=_.zoom,K.updateProjectionMatrix(),_=null}i.dispatchEvent({type:"sessionend"})}this.setFramebufferScaleFactor=function(K){a=K,i.isPresenting===!0&&Je("WebXRManager: Cannot change framebuffer scale while presenting.")},this.setReferenceSpaceType=function(K){s=K,i.isPresenting===!0&&Je("WebXRManager: Cannot change reference space type while presenting.")},this.getReferenceSpace=function(){return l||o},this.setReferenceSpace=function(K){l=K},this.getBaseLayer=function(){return u===null?d:u},this.getBinding=function(){return c===null&&p&&(c=new XRWebGLBinding(r,t)),c},this.getFrame=function(){return f},this.getSession=function(){return r},this.setSession=async function(K){if(r=K,r!==null){if(y=e.getRenderTarget(),r.addEventListener("select",z),r.addEventListener("selectstart",z),r.addEventListener("selectend",z),r.addEventListener("squeeze",z),r.addEventListener("squeezestart",z),r.addEventListener("squeezeend",z),r.addEventListener("end",H),r.addEventListener("inputsourceschange",D),m.xrCompatible!==!0&&await t.makeXRCompatible(),v=e.getPixelRatio(),e.getSize(T),p&&"createProjectionLayer"in XRWebGLBinding.prototype){let re=null,xe=null,Ne=null;m.depth&&(Ne=m.stencil?t.DEPTH24_STENCIL8:t.DEPTH_COMPONENT24,re=m.stencil?Cs:bo,xe=m.stencil?vl:Vr);let Me={colorFormat:t.RGBA8,depthFormat:Ne,scaleFactor:a};c=this.getBinding(),u=c.createProjectionLayer(Me),r.updateRenderState({layers:[u]}),e.setPixelRatio(1),e.setSize(u.textureWidth,u.textureHeight,!1),M=new gi(u.textureWidth,u.textureHeight,{format:yo,type:Fr,depthTexture:new ra(u.textureWidth,u.textureHeight,xe,void 0,void 0,void 0,void 0,void 0,void 0,re),stencilBuffer:m.stencil,colorSpace:e.outputColorSpace,samples:m.antialias?4:0,resolveDepthBuffer:u.ignoreDepthValues===!1,resolveStencilBuffer:u.ignoreDepthValues===!1,storeMultisampledDepthBuffer:u.ignoreDepthValues===!1,storeMultisampledStencilBuffer:u.ignoreDepthValues===!1})}else{let re={antialias:m.antialias,alpha:!0,depth:m.depth,stencil:m.stencil,framebufferScaleFactor:a};d=new XRWebGLLayer(r,t,re),r.updateRenderState({baseLayer:d}),e.setPixelRatio(1),e.setSize(d.framebufferWidth,d.framebufferHeight,!1),M=new gi(d.framebufferWidth,d.framebufferHeight,{format:yo,type:Fr,colorSpace:e.outputColorSpace,stencilBuffer:m.stencil,resolveDepthBuffer:d.ignoreDepthValues===!1,resolveStencilBuffer:d.ignoreDepthValues===!1,storeMultisampledDepthBuffer:d.ignoreDepthValues===!1,storeMultisampledStencilBuffer:d.ignoreDepthValues===!1})}M.isXRRenderTarget=!0,this.setFoveation(n),l=null,o=await r.requestReferenceSpace(s),Se.setContext(r),Se.start(),i.isPresenting=!0,i.dispatchEvent({type:"sessionstart"})}},this.getEnvironmentBlendMode=function(){if(r!==null)return r.environmentBlendMode},this.getDepthTexture=function(){return x.getDepthTexture()};function D(K){for(let re=0;re<K.removed.length;re++){let xe=K.removed[re],Ne=E.indexOf(xe);Ne>=0&&(E[Ne]=null,b[Ne].disconnect(xe))}for(let re=0;re<K.added.length;re++){let xe=K.added[re],Ne=E.indexOf(xe);if(Ne===-1){for(let Q=0;Q<b.length;Q++)if(Q>=E.length){E.push(xe),Ne=Q;break}else if(E[Q]===null){E[Q]=xe,Ne=Q;break}if(Ne===-1)break}let Me=b[Ne];Me&&Me.connect(xe)}}function ae(K,re){re===null?K.matrixWorld.copy(K.matrix):K.matrixWorld.multiplyMatrices(re.matrixWorld,K.matrix),K.matrixWorldInverse.copy(K.matrixWorld).invert()}this.updateCamera=function(K){if(r===null)return;let re=K.near,xe=K.far;x.texture!==null&&(x.depthNear>0&&(re=x.depthNear),x.depthFar>0&&(xe=x.depthFar)),L.near=C.near=A.near=re,L.far=C.far=A.far=xe,(G!==L.near||P!==L.far)&&(r.updateRenderState({depthNear:L.near,depthFar:L.far}),G=L.near,P=L.far),L.layers.mask=K.layers.mask|6,A.layers.mask=L.layers.mask&-5,C.layers.mask=L.layers.mask&-3;let Ne=K.parent,Me=L.cameras;ae(L,Ne);for(let Q=0;Q<Me.length;Q++)ae(Me[Q],Ne);L.projectionMatrix.copy(A.projectionMatrix),L.projectionMatrixInverse.copy(A.projectionMatrixInverse),_===null&&K.isPerspectiveCamera&&(_={camera:K,fov:K.fov,zoom:K.zoom}),j(K,L,Ne)};function j(K,re,xe){xe===null?K.matrix.copy(re.matrixWorld):(K.matrix.copy(xe.matrixWorld),K.matrix.invert(),K.matrix.multiply(re.matrixWorld)),K.matrix.decompose(K.position,K.quaternion,K.scale),K.updateMatrixWorld(!0),K.projectionMatrix.copy(re.projectionMatrix),K.projectionMatrixInverse.copy(re.projectionMatrixInverse),K.isPerspectiveCamera&&(K.fov=Kn*2*Math.atan(1/K.projectionMatrix.elements[5]),K.zoom=1)}this.getCamera=function(){return L},this.getFoveation=function(){if(u!==null||d!==null)return n},this.setFoveation=function(K){n=K,u!==null&&(u.fixedFoveation=K),d!==null&&d.fixedFoveation!==void 0&&(d.fixedFoveation=K)},this.hasDepthSensing=function(){return x.texture!==null},this.getDepthSensingMesh=function(){return x.getMesh(L)},this.getCameraTexture=function(K){return g[K]};let Y=null;function ie(K,re){if(h=re.getViewerPose(l||o),f=re,h!==null){let xe=h.views;d!==null&&(e.setRenderTargetFramebuffer(M,d.framebuffer),e.setRenderTarget(M));let Ne=!1;xe.length!==L.cameras.length&&(L.cameras.length=0,Ne=!0);for(let Q=0;Q<xe.length;Q++){let ke=xe[Q],$e=null;if(d!==null)$e=d.getViewport(ke);else{let nt=c.getViewSubImage(u,ke);$e=nt.viewport,Q===0&&(e.setRenderTargetTextures(M,nt.colorTexture,nt.depthStencilTexture),e.setRenderTarget(M))}let De=R[Q];De===void 0&&(De=new ji,De.layers.enable(Q),De.viewport=new Ft,R[Q]=De),De.matrix.fromArray(ke.transform.matrix),De.matrix.decompose(De.position,De.quaternion,De.scale),De.projectionMatrix.fromArray(ke.projectionMatrix),De.projectionMatrixInverse.copy(De.projectionMatrix).invert(),De.viewport.set($e.x,$e.y,$e.width,$e.height),Q===0&&(L.matrix.copy(De.matrix),L.matrix.decompose(L.position,L.quaternion,L.scale)),Ne===!0&&L.cameras.push(De)}let Me=r.enabledFeatures;if(Me&&Me.includes("depth-sensing")&&r.depthUsage=="gpu-optimized"&&p){c=i.getBinding();let Q=c.getDepthInformation(xe[0]);Q&&Q.isValid&&Q.texture&&x.init(Q,r.renderState)}if(Me&&Me.includes("camera-access")&&p){e.state.unbindTexture(),c=i.getBinding();for(let Q=0;Q<xe.length;Q++){let ke=xe[Q].camera;if(ke){let $e=g[ke];$e||($e=new Kc,g[ke]=$e);let De=c.getCameraImage(ke);$e.sourceTexture=De}}}}for(let xe=0;xe<b.length;xe++){let Ne=E[xe],Me=b[xe];Ne!==null&&Me!==void 0&&Me.update(Ne,re,l||o)}Y&&Y(K,re),re.detectedPlanes&&i.dispatchEvent({type:"planesdetected",data:re}),f=null}let Se=new Ru;Se.setAnimationLoop(ie),this.setAnimationLoop=function(K){Y=K},this.dispose=function(){}}},a1=new gt,Bu=new it;Bu.set(-1,0,0,0,1,0,0,0,1);function o1(e,t){function i(g,m){g.matrixAutoUpdate===!0&&g.updateMatrix(),m.value.copy(g.matrix)}function r(g,m){m.color.getRGB(g.fogColor.value,wu(e)),m.isFog?(g.fogNear.value=m.near,g.fogFar.value=m.far):m.isFogExp2&&(g.fogDensity.value=m.density)}function a(g,m,y,M,b){m.isNodeMaterial?m.uniformsNeedUpdate=!1:m.isMeshBasicMaterial?o(g,m):m.isMeshLambertMaterial?(o(g,m),m.envMap&&(g.envMapIntensity.value=m.envMapIntensity)):m.isMeshToonMaterial?(o(g,m),u(g,m)):m.isMeshPhongMaterial?(o(g,m),c(g,m),m.envMap&&(g.envMapIntensity.value=m.envMapIntensity)):m.isMeshStandardMaterial?(o(g,m),d(g,m),m.isMeshPhysicalMaterial&&f(g,m,b)):m.isMeshMatcapMaterial?(o(g,m),p(g,m)):m.isMeshDepthMaterial?o(g,m):m.isMeshDistanceMaterial?(o(g,m),x(g,m)):m.isMeshNormalMaterial?o(g,m):m.isLineBasicMaterial?(s(g,m),m.isLineDashedMaterial&&n(g,m)):m.isPointsMaterial?l(g,m,y,M):m.isSpriteMaterial?h(g,m):m.isShadowMaterial?(g.color.value.copy(m.color),g.opacity.value=m.opacity):m.isShaderMaterial&&(m.uniformsNeedUpdate=!1)}function o(g,m){g.opacity.value=m.opacity,m.color&&g.diffuse.value.copy(m.color),m.emissive&&g.emissive.value.copy(m.emissive).multiplyScalar(m.emissiveIntensity),m.map&&(g.map.value=m.map,i(m.map,g.mapTransform)),m.alphaMap&&(g.alphaMap.value=m.alphaMap,i(m.alphaMap,g.alphaMapTransform)),m.bumpMap&&(g.bumpMap.value=m.bumpMap,i(m.bumpMap,g.bumpMapTransform),g.bumpScale.value=m.bumpScale,m.side===1&&(g.bumpScale.value*=-1)),m.normalMap&&(g.normalMap.value=m.normalMap,i(m.normalMap,g.normalMapTransform),g.normalScale.value.copy(m.normalScale),m.side===1&&g.normalScale.value.negate()),m.displacementMap&&(g.displacementMap.value=m.displacementMap,i(m.displacementMap,g.displacementMapTransform),g.displacementScale.value=m.displacementScale,g.displacementBias.value=m.displacementBias),m.emissiveMap&&(g.emissiveMap.value=m.emissiveMap,i(m.emissiveMap,g.emissiveMapTransform)),m.specularMap&&(g.specularMap.value=m.specularMap,i(m.specularMap,g.specularMapTransform)),m.alphaTest>0&&(g.alphaTest.value=m.alphaTest);let y=t.get(m),M=y.envMap,b=y.envMapRotation;M&&(g.envMap.value=M,g.envMapRotation.value.setFromMatrix4(a1.makeRotationFromEuler(b)).transpose(),M.isCubeTexture&&M.isRenderTargetTexture===!1&&g.envMapRotation.value.premultiply(Bu),g.reflectivity.value=m.reflectivity,g.ior.value=m.ior,g.refractionRatio.value=m.refractionRatio),m.lightMap&&(g.lightMap.value=m.lightMap,g.lightMapIntensity.value=m.lightMapIntensity,i(m.lightMap,g.lightMapTransform)),m.aoMap&&(g.aoMap.value=m.aoMap,g.aoMapIntensity.value=m.aoMapIntensity,i(m.aoMap,g.aoMapTransform))}function s(g,m){g.diffuse.value.copy(m.color),g.opacity.value=m.opacity,m.map&&(g.map.value=m.map,i(m.map,g.mapTransform))}function n(g,m){g.dashSize.value=m.dashSize,g.totalSize.value=m.dashSize+m.gapSize,g.scale.value=m.scale}function l(g,m,y,M){g.diffuse.value.copy(m.color),g.opacity.value=m.opacity,g.size.value=m.size*y,g.scale.value=M*.5,m.map&&(g.map.value=m.map,i(m.map,g.uvTransform)),m.alphaMap&&(g.alphaMap.value=m.alphaMap,i(m.alphaMap,g.alphaMapTransform)),m.alphaTest>0&&(g.alphaTest.value=m.alphaTest)}function h(g,m){g.diffuse.value.copy(m.color),g.opacity.value=m.opacity,g.rotation.value=m.rotation,m.map&&(g.map.value=m.map,i(m.map,g.mapTransform)),m.alphaMap&&(g.alphaMap.value=m.alphaMap,i(m.alphaMap,g.alphaMapTransform)),m.alphaTest>0&&(g.alphaTest.value=m.alphaTest)}function c(g,m){g.specular.value.copy(m.specular),g.shininess.value=Math.max(m.shininess,1e-4)}function u(g,m){m.gradientMap&&(g.gradientMap.value=m.gradientMap)}function d(g,m){g.metalness.value=m.metalness,m.metalnessMap&&(g.metalnessMap.value=m.metalnessMap,i(m.metalnessMap,g.metalnessMapTransform)),g.roughness.value=m.roughness,m.roughnessMap&&(g.roughnessMap.value=m.roughnessMap,i(m.roughnessMap,g.roughnessMapTransform)),m.envMap&&(g.envMapIntensity.value=m.envMapIntensity)}function f(g,m,y){g.ior.value=m.ior,m.diffuseRoughness>0&&(g.diffuseRoughness.value=m.diffuseRoughness,m.diffuseRoughnessMap&&(g.diffuseRoughnessMap.value=m.diffuseRoughnessMap,i(m.diffuseRoughnessMap,g.diffuseRoughnessMapTransform))),m.sheen>0&&(g.sheenColor.value.copy(m.sheenColor).multiplyScalar(m.sheen),g.sheenRoughness.value=m.sheenRoughness,m.sheenColorMap&&(g.sheenColorMap.value=m.sheenColorMap,i(m.sheenColorMap,g.sheenColorMapTransform)),m.sheenRoughnessMap&&(g.sheenRoughnessMap.value=m.sheenRoughnessMap,i(m.sheenRoughnessMap,g.sheenRoughnessMapTransform))),m.clearcoat>0&&(g.clearcoat.value=m.clearcoat,g.clearcoatRoughness.value=m.clearcoatRoughness,m.clearcoatMap&&(g.clearcoatMap.value=m.clearcoatMap,i(m.clearcoatMap,g.clearcoatMapTransform)),m.clearcoatRoughnessMap&&(g.clearcoatRoughnessMap.value=m.clearcoatRoughnessMap,i(m.clearcoatRoughnessMap,g.clearcoatRoughnessMapTransform)),m.clearcoatNormalMap&&(g.clearcoatNormalMap.value=m.clearcoatNormalMap,i(m.clearcoatNormalMap,g.clearcoatNormalMapTransform),g.clearcoatNormalScale.value.copy(m.clearcoatNormalScale),m.side===1&&g.clearcoatNormalScale.value.negate())),m.dispersion>0&&(g.dispersion.value=m.dispersion),m.retroreflectivity>0&&(g.retroreflectivity.value=m.retroreflectivity),m.iridescence>0&&(g.iridescence.value=m.iridescence,g.iridescenceIOR.value=m.iridescenceIOR,g.iridescenceThicknessMinimum.value=m.iridescenceThicknessMinimum,g.iridescenceThicknessMaximum.value=m.iridescenceThicknessMaximum,m.iridescenceMap&&(g.iridescenceMap.value=m.iridescenceMap,i(m.iridescenceMap,g.iridescenceMapTransform)),m.iridescenceThicknessMap&&(g.iridescenceThicknessMap.value=m.iridescenceThicknessMap,i(m.iridescenceThicknessMap,g.iridescenceThicknessMapTransform))),m.transmission>0&&(g.transmission.value=m.transmission,g.transmissionSamplerMap.value=y.texture,g.transmissionSamplerSize.value.set(y.width,y.height),m.transmissionMap&&(g.transmissionMap.value=m.transmissionMap,i(m.transmissionMap,g.transmissionMapTransform)),g.thickness.value=m.thickness,m.thicknessMap&&(g.thicknessMap.value=m.thicknessMap,i(m.thicknessMap,g.thicknessMapTransform)),g.attenuationDistance.value=m.attenuationDistance,g.attenuationColor.value.copy(m.attenuationColor)),m.anisotropy>0&&(g.anisotropyVector.value.set(m.anisotropy*Math.cos(m.anisotropyRotation),m.anisotropy*Math.sin(m.anisotropyRotation)),m.anisotropyMap&&(g.anisotropyMap.value=m.anisotropyMap,i(m.anisotropyMap,g.anisotropyMapTransform))),g.specularIntensity.value=m.specularIntensity,g.specularColor.value.copy(m.specularColor),m.specularColorMap&&(g.specularColorMap.value=m.specularColorMap,i(m.specularColorMap,g.specularColorMapTransform)),m.specularIntensityMap&&(g.specularIntensityMap.value=m.specularIntensityMap,i(m.specularIntensityMap,g.specularIntensityMapTransform))}function p(g,m){m.matcap&&(g.matcap.value=m.matcap)}function x(g,m){let y=t.get(m).light;g.referencePosition.value.setFromMatrixPosition(y.matrixWorld),g.nearDistance.value=y.shadow.camera.near,g.farDistance.value=y.shadow.camera.far}return{refreshFogUniforms:r,refreshMaterialUniforms:a}}function s1(e,t,i,r){let a={},o={};function s(y,M){let b=M.program;r.uniformBlockBinding(y,b,a[y.id])}function n(y,M){let b=a[y.id];b===void 0&&(p(y),b=l(y),a[y.id]=b,y.addEventListener("dispose",g));let E=M.program;r.updateUBOMapping(y,E);let T=t.render.frame;o[y.id]!==T&&(h(y),o[y.id]=T)}function l(y){let M=e.createBuffer(),b=y.__size,E=y.usage;return e.bindBuffer(e.UNIFORM_BUFFER,M),e.bufferData(e.UNIFORM_BUFFER,b,E),e.bindBuffer(e.UNIFORM_BUFFER,null),M}function h(y){let M=a[y.id],b=y.uniforms,E=y.__cache;e.bindBuffer(e.UNIFORM_BUFFER,M);for(let T=0,v=b.length;T<v;T++){let _=b[T];if(Array.isArray(_))for(let A=0,C=_.length;A<C;A++)c(_[A],T,A,E);else c(_,T,0,E)}e.bindBuffer(e.UNIFORM_BUFFER,null)}function c(y,M,b,E){if(d(y,M,b,E)===!0){let T=y.__offset,v=y.value;if(Array.isArray(v)){let _=0;for(let A=0;A<v.length;A++){let C=v[A],R=x(C,!0);u(C,y.__data,_),_+=R.storage/Float32Array.BYTES_PER_ELEMENT}}else u(v,y.__data,0);e.bufferSubData(e.UNIFORM_BUFFER,T,y.__data)}}function u(y,M,b){typeof y=="number"||typeof y=="boolean"?M[b]=y:y.isMatrix3?(M[b]=y.elements[0],M[b+1]=y.elements[1],M[b+2]=y.elements[2],M[b+3]=0,M[b+4]=y.elements[3],M[b+5]=y.elements[4],M[b+6]=y.elements[5],M[b+7]=0,M[b+8]=y.elements[6],M[b+9]=y.elements[7],M[b+10]=y.elements[8],M[b+11]=0):ArrayBuffer.isView(y)?M.set(y.subarray(0,M.length-b),b):y.toArray(M,b)}function d(y,M,b,E){let T=y.value,v=M+"_"+b;if(Array.isArray(T)){E[v]===void 0&&(E[v]=[]);let _=E[v],A=!1;for(let C=0;C<T.length;C++)f(T[C],_,C)===!0&&(A=!0);return A}return f(T,E,v)}function f(y,M,b){if(M[b]===void 0)return M[b]=typeof y=="number"||typeof y=="boolean"?y:ArrayBuffer.isView(y)?y.slice():y.clone(),!0;{let E=M[b];if(typeof y=="number"||typeof y=="boolean"){if(E!==y)return M[b]=y,!0}else{if(ArrayBuffer.isView(y))return!0;if(E.equals(y)===!1)return E.copy(y),!0}}return!1}function p(y){let M=y.uniforms,b=0;for(let T=0,v=M.length;T<v;T++){let _=Array.isArray(M[T])?M[T]:[M[T]];for(let A=0,C=_.length;A<C;A++){let R=_[A],L=Array.isArray(R.value)?R.value:[R.value],G=Array.isArray(M[T])||Array.isArray(R.value),P=b;for(let z=0,H=L.length;z<H;z++){let D=L[z],ae=x(D,G),j=b%16,Y=j%ae.boundary,ie=j+Y;b+=Y,ie!==0&&16-ie<ae.storage&&(b+=16-ie),z===0&&(P=b),b+=ae.storage}R.__data=new(R.type==="int"?Int32Array:R.type==="uint"?Uint32Array:Float32Array)((b-P)/Float32Array.BYTES_PER_ELEMENT),R.__offset=P}}let E=b%16;return E>0&&(b+=16-E),y.__size=b,y.__cache={},this}function x(y,M){let b={boundary:0,storage:0};return typeof y=="number"||typeof y=="boolean"?(b.boundary=4,b.storage=4):y.isVector2?(b.boundary=8,b.storage=8):y.isVector3||y.isColor?(b.boundary=16,b.storage=12):y.isVector4?(b.boundary=16,b.storage=16):y.isMatrix3?(b.boundary=16,b.storage=48):y.isMatrix4?(b.boundary=16,b.storage=64):y.isTexture?Je("WebGLRenderer: Texture samplers can not be part of an uniforms group."):ArrayBuffer.isView(y)?(b.boundary=16,b.storage=y.byteLength):Je("WebGLRenderer: Unsupported uniform value type.",y),M===!0&&(b.boundary=16,b.storage=Math.ceil(b.storage/16)*16),b}function g(y){let M=y.target;M.removeEventListener("dispose",g),e.deleteBuffer(a[M.id]),delete a[M.id],delete o[M.id]}function m(){for(let y in a)e.deleteBuffer(a[y]);a={},o={}}return{bind:s,update:n,dispose:m}}var n1=new Uint16Array([12469,15057,12620,14925,13266,14620,13807,14376,14323,13990,14545,13625,14713,13328,14840,12882,14931,12528,14996,12233,15039,11829,15066,11525,15080,11295,15085,10976,15082,10705,15073,10495,13880,14564,13898,14542,13977,14430,14158,14124,14393,13732,14556,13410,14702,12996,14814,12596,14891,12291,14937,11834,14957,11489,14958,11194,14943,10803,14921,10506,14893,10278,14858,9960,14484,14039,14487,14025,14499,13941,14524,13740,14574,13468,14654,13106,14743,12678,14818,12344,14867,11893,14889,11509,14893,11180,14881,10751,14852,10428,14812,10128,14765,9754,14712,9466,14764,13480,14764,13475,14766,13440,14766,13347,14769,13070,14786,12713,14816,12387,14844,11957,14860,11549,14868,11215,14855,10751,14825,10403,14782,10044,14729,9651,14666,9352,14599,9029,14967,12835,14966,12831,14963,12804,14954,12723,14936,12564,14917,12347,14900,11958,14886,11569,14878,11247,14859,10765,14828,10401,14784,10011,14727,9600,14660,9289,14586,8893,14508,8533,15111,12234,15110,12234,15104,12216,15092,12156,15067,12010,15028,11776,14981,11500,14942,11205,14902,10752,14861,10393,14812,9991,14752,9570,14682,9252,14603,8808,14519,8445,14431,8145,15209,11449,15208,11451,15202,11451,15190,11438,15163,11384,15117,11274,15055,10979,14994,10648,14932,10343,14871,9936,14803,9532,14729,9218,14645,8742,14556,8381,14461,8020,14365,7603,15273,10603,15272,10607,15267,10619,15256,10631,15231,10614,15182,10535,15118,10389,15042,10167,14963,9787,14883,9447,14800,9115,14710,8665,14615,8318,14514,7911,14411,7507,14279,7198,15314,9675,15313,9683,15309,9712,15298,9759,15277,9797,15229,9773,15166,9668,15084,9487,14995,9274,14898,8910,14800,8539,14697,8234,14590,7790,14479,7409,14367,7067,14178,6621,15337,8619,15337,8631,15333,8677,15325,8769,15305,8871,15264,8940,15202,8909,15119,8775,15022,8565,14916,8328,14804,8009,14688,7614,14569,7287,14448,6888,14321,6483,14088,6171,15350,7402,15350,7419,15347,7480,15340,7613,15322,7804,15287,7973,15229,8057,15148,8012,15046,7846,14933,7611,14810,7357,14682,7069,14552,6656,14421,6316,14251,5948,14007,5528,15356,5942,15356,5977,15353,6119,15348,6294,15332,6551,15302,6824,15249,7044,15171,7122,15070,7050,14949,6861,14818,6611,14679,6349,14538,6067,14398,5651,14189,5311,13935,4958,15359,4123,15359,4153,15356,4296,15353,4646,15338,5160,15311,5508,15263,5829,15188,6042,15088,6094,14966,6001,14826,5796,14678,5543,14527,5287,14377,4985,14133,4586,13869,4257,15360,1563,15360,1642,15358,2076,15354,2636,15341,3350,15317,4019,15273,4429,15203,4732,15105,4911,14981,4932,14836,4818,14679,4621,14517,4386,14359,4156,14083,3795,13808,3437,15360,122,15360,137,15358,285,15355,636,15344,1274,15322,2177,15281,2765,15215,3223,15120,3451,14995,3569,14846,3567,14681,3466,14511,3305,14344,3121,14037,2800,13753,2467,15360,0,15360,1,15359,21,15355,89,15346,253,15325,479,15287,796,15225,1148,15133,1492,15008,1749,14856,1882,14685,1886,14506,1783,14324,1608,13996,1398,13702,1183]),tr=null;function l1(){return tr===null&&(tr=new No(n1,16,16,Ps,Rr),tr.name="DFG_LUT",tr.minFilter=jt,tr.magFilter=jt,tr.wrapS=Zi,tr.wrapT=Zi,tr.generateMipmaps=!1,tr.needsUpdate=!0),tr}var h1=class{constructor(e={}){let{canvas:t=o0(),context:i=null,depth:r=!0,stencil:a=!1,alpha:o=!1,antialias:s=!1,premultipliedAlpha:n=!0,preserveDrawingBuffer:l=!1,powerPreference:h="default",failIfMajorPerformanceCaveat:c=!1,reversedDepthBuffer:u=!1,outputBufferType:d=Fr}=e;this.isWebGLRenderer=!0;let f;if(i!==null){if(typeof WebGLRenderingContext<"u"&&i instanceof WebGLRenderingContext)throw Error("THREE.WebGLRenderer: WebGL 1 is not supported since r163.");f=i.getContextAttributes().alpha}else f=o;let p=d,x=new Set([zc,Oc,Uc]),g=new Set([Fr,Vr,Nc,vl,Ic,Dc]),m=new Uint32Array(4),y=new Int32Array(4),M={textures:[],min:[],max:[],resolution:[]},b=null,E=null,T=[],v=[],_=null;this.domElement=t,this.debug={checkShaderErrors:!0,diagnostics:{keywords:!1},onShaderError:null},this.autoClear=!0,this.autoClearColor=!0,this.autoClearDepth=!0,this.autoClearStencil=!0,this.sortObjects=!0,this.clippingPlanes=[],this.localClippingEnabled=!1,this.toneMapping=0,this.toneMappingExposure=1,this.transmissionResolutionScale=1;let A=this,C=!1,R=null,L=null,G=null,P=null;this._outputColorSpace=qi;let z=0,H=0,D=null,ae=-1,j=null,Y=new Ft,ie=new Ft,Se=null,K=new dt(0),re=0,xe=t.width,Ne=t.height,Me=1,Q=null,ke=null,$e=new Ft(0,0,xe,Ne),De=new Ft(0,0,xe,Ne),nt=!1,oe=new Os,he=new O0,de=!1,Ue=!1,Ie=new gt,F=new I,te=new Ft,$={background:null,fog:null,environment:null,overrideMaterial:null,isScene:!0},Te=!1;function Be(){return D===null?Me:1}let U=i;function ut(w,k){return t.getContext(w,k)}let B,Ce,ne,lt,Ge,je,At,N,S,W,ce,ee,ye,_e,V,le,Pe,O,Z,ue,me,Ae,Ee;try{let w={alpha:!0,depth:r,stencil:a,antialias:s,premultipliedAlpha:n,preserveDrawingBuffer:l,powerPreference:h,failIfMajorPerformanceCaveat:c};if("setAttribute"in t&&t.setAttribute("data-engine","three.js r187"),t.addEventListener("webglcontextlost",tt,!1),t.addEventListener("webglcontextrestored",Ze,!1),t.addEventListener("webglcontextcreationerror",mt,!1),U===null){let k="webgl2";if(U=ut(k,w),U===null)throw ut(k)?Error("THREE.WebGLRenderer: Error creating WebGL context with your selected attributes."):Error("THREE.WebGLRenderer: Error creating WebGL context.")}Ye()}catch(w){throw t.removeEventListener("webglcontextlost",tt,!1),t.removeEventListener("webglcontextrestored",Ze,!1),t.removeEventListener("webglcontextcreationerror",mt,!1),Qe("WebGLRenderer: "+w.message),w}function Ye(){B=new dp(U),B.init(),me=new Qm(U,B),Ce=new tp(U,B,e,me),ne=new Km(U,B),Ce.reversedDepthBuffer&&u&&ne.buffers.depth.setReversed(!0),L=U.createFramebuffer(),G=U.createFramebuffer(),P=U.createFramebuffer(),lt=new mp(U),Ge=new zm,je=new $m(U,B,ne,Ge,Ce,me,lt),At=new up(A),N=new Jf(U),Ae=new Qf(U,N),S=new fp(U,N,lt,Ae),W=new vp(U,S,N,Ae,lt),O=new gp(U,Ce,je),V=new ip(Ge),ce=new Om(A,At,B,Ce,Ae,V),ee=new o1(A,Ge),ye=new Bm,_e=new Xm(B),Pe=new $f(A,At,ne,W,f,n),le=new Jm(A,W,Ce),Ee=new s1(U,lt,Ce,ne),Z=new ep(U,B,lt),ue=new pp(U,B,lt),lt.programs=ce.programs,A.capabilities=Ce,A.extensions=B,A.properties=Ge,A.renderLists=ye,A.shadowMap=le,A.state=ne,A.info=lt}p!==1009&&(_=new _p(p,t.width,t.height,s,r,a));let fe=new r1(A,U);this.xr=fe,this.getContext=function(){return U},this.getContextAttributes=function(){return U.getContextAttributes()},this.forceContextLoss=function(){let w=B.get("WEBGL_lose_context");w&&w.loseContext()},this.forceContextRestore=function(){let w=B.get("WEBGL_lose_context");w&&w.restoreContext()},this.getPixelRatio=function(){return Me},this.setPixelRatio=function(w){w!==void 0&&(Me=w,this.setSize(xe,Ne,!1))},this.getSize=function(w){return w.set(xe,Ne)},this.setSize=function(w,k,J=!0){fe.isPresenting?Je("WebGLRenderer: Can't change size while VR device is presenting."):(xe=w,Ne=k,t.width=Math.floor(w*Me),t.height=Math.floor(k*Me),J===!0&&(t.style.width=w+"px",t.style.height=k+"px"),_!==null&&_.setSize(t.width,t.height),this.setViewport(0,0,w,k))},this.getDrawingBufferSize=function(w){return w.set(xe*Me,Ne*Me).floor()},this.setDrawingBufferSize=function(w,k,J){xe=w,Ne=k,Me=J,t.width=Math.floor(w*J),t.height=Math.floor(k*J),this.setViewport(0,0,w,k)},this.setEffects=function(w){if(p===1009)Qe("WebGLRenderer: setEffects() requires outputBufferType set to HalfFloatType or FloatType.");else{if(w){for(let k=0;k<w.length;k++)if(w[k].isOutputPass===!0){Je("WebGLRenderer: OutputPass is not needed in setEffects(). Tone mapping and color space conversion are applied automatically.");break}}_.setEffects(w||[])}},this.getCurrentViewport=function(w){return w.copy(Y)},this.getViewport=function(w){return w.copy($e)},this.setViewport=function(w,k,J,q){w.isVector4?$e.set(w.x,w.y,w.z,w.w):$e.set(w,k,J,q),ne.viewport(Y.copy($e).multiplyScalar(Be()).round())},this.getScissor=function(w){return w.copy(De)},this.setScissor=function(w,k,J,q){w.isVector4?De.set(w.x,w.y,w.z,w.w):De.set(w,k,J,q),ne.scissor(ie.copy(De).multiplyScalar(Be()).round())},this.getScissorTest=function(){return nt},this.setScissorTest=function(w){ne.setScissorTest(nt=w)},this.setOpaqueSort=function(w){Q=w},this.setTransparentSort=function(w){ke=w},this.getClearColor=function(w){return w.copy(Pe.getClearColor())},this.setClearColor=function(){Pe.setClearColor(...arguments)},this.getClearAlpha=function(){return Pe.getClearAlpha()},this.setClearAlpha=function(){Pe.setClearAlpha(...arguments)},this.clear=function(w=!0,k=!0,J=!0){let q=0;if(w){let X=!1;if(D!==null){let ve=D.texture.format;X=x.has(ve)}if(X){let ve=D.texture.type,Re=g.has(ve),we=Pe.getClearColor(),ze=Pe.getClearAlpha(),qe=we.r,ht=we.g,vt=we.b;Re?(m[0]=qe,m[1]=ht,m[2]=vt,m[3]=ze,U.clearBufferuiv(U.COLOR,0,m)):(y[0]=qe,y[1]=ht,y[2]=vt,y[3]=ze,U.clearBufferiv(U.COLOR,0,y))}else q|=U.COLOR_BUFFER_BIT}k&&(q|=U.DEPTH_BUFFER_BIT,this.state.buffers.depth.setMask(!0)),J&&(q|=U.STENCIL_BUFFER_BIT,this.state.buffers.stencil.setMask(4294967295)),q!==0&&U.clear(q)},this.clearColor=function(){this.clear(!0,!1,!1)},this.clearDepth=function(){this.clear(!1,!0,!1)},this.clearStencil=function(){this.clear(!1,!1,!0)},this.setNodesHandler=function(w){w.setRenderer(this),R=w},this.dispose=function(){t.removeEventListener("webglcontextlost",tt,!1),t.removeEventListener("webglcontextrestored",Ze,!1),t.removeEventListener("webglcontextcreationerror",mt,!1),Pe.dispose(),ye.dispose(),_e.dispose(),je.dispose(),Ge.dispose(),At.dispose(),W.dispose(),S.dispose(),Ae.dispose(),Ee.dispose(),ce.dispose(),fe.dispose(),fe.removeEventListener("sessionstart",Dl),fe.removeEventListener("sessionend",Ul),Hr.stop()};function tt(w){w.preventDefault(),Zl("WebGLRenderer: Context Lost."),C=!0}function Ze(){Zl("WebGLRenderer: Context Restored."),C=!1;let w=lt.autoReset,k=le.enabled,J=le.autoUpdate,q=le.needsUpdate,X=le.type;Ye(),lt.autoReset=w,le.enabled=k,le.autoUpdate=J,le.needsUpdate=q,le.type=X}function mt(w){Qe("WebGLRenderer: A WebGL context could not be created. Reason: ",w.statusMessage)}function Oe(w){let k=w.target;k.removeEventListener("dispose",Oe),Ot(k)}function Ot(w){zt(w),Ge.remove(w)}function zt(w){let k=Ge.get(w).programs;k!==void 0&&(k.forEach(function(J){ce.releaseProgram(J)}),w.isShaderMaterial&&ce.releaseShaderCache(w))}this.renderBufferDirect=function(w,k,J,q,X,ve){k===null&&(k=$);let Re=X.isMesh&&X.matrixWorld.determinantAffine()<0,we=sd(w,k,J,q,X);ne.setMaterial(q,Re);let ze=J.index,qe=1;if(q.wireframe===!0){if(ze=S.getWireframeAttribute(J),ze===void 0)return;qe=2}let ht=J.drawRange,vt=J.attributes.position,Ve=ht.start*qe,Tt=(ht.start+ht.count)*qe;ve!==null&&(Ve=Math.max(Ve,ve.start*qe),Tt=Math.min(Tt,(ve.start+ve.count)*qe)),ze===null?vt!=null&&(Ve=Math.max(Ve,0),Tt=Math.min(Tt,vt.count)):(Ve=Math.max(Ve,0),Tt=Math.min(Tt,ze.count));let qt=Tt-Ve;if(qt<0||qt===1/0)return;Ae.setup(X,q,we,J,ze);let Lt,Mt=Z;if(ze!==null&&(Lt=N.get(ze),Mt=ue,Mt.setIndex(Lt)),X.isMesh)q.wireframe===!0?(ne.setLineWidth(q.wireframeLinewidth*Be()),Mt.setMode(U.LINES)):Mt.setMode(U.TRIANGLES);else if(X.isLine){let ni=q.linewidth;ni===void 0&&(ni=1),ne.setLineWidth(ni*Be()),X.isLineSegments?Mt.setMode(U.LINES):X.isLineLoop?Mt.setMode(U.LINE_LOOP):Mt.setMode(U.LINE_STRIP)}else X.isPoints?Mt.setMode(U.POINTS):X.isSprite&&Mt.setMode(U.TRIANGLES);if(X.isBatchedMesh)if(B.get("WEBGL_multi_draw"))Mt.renderMultiDraw(X._multiDrawStarts,X._multiDrawCounts,X._multiDrawCount);else{let ni=X._multiDrawStarts,We=X._multiDrawCounts,Si=X._multiDrawCount,Qi=ze?N.get(ze).bytesPerElement:1,Wr=Ge.get(q).currentProgram.getUniforms();for(let li=0;li<Si;li++)Wr.setValue(U,"_gl_DrawID",li),Mt.render(ni[li]/Qi,We[li])}else if(X.isInstancedMesh)Mt.renderInstances(Ve,qt,X.count);else if(J.isInstancedBufferGeometry){let ni=J._maxInstanceCount===void 0?1/0:J._maxInstanceCount,We=Math.min(J.instanceCount,ni);Mt.renderInstances(Ve,qt,We)}else Mt.render(Ve,qt)};function Il(w,k,J,q){R!==null&&w.isNodeMaterial&&R.setObject(q,w),de===!0&&V.setState(w,J,!1),w.transparent===!0&&w.side===2&&w.forceSinglePass===!1?(w.side=1,w.needsUpdate=!0,Ya(w,k,q),w.side=0,w.needsUpdate=!0,Ya(w,k,q),w.side=2):Ya(w,k,q)}this.compile=function(w,k,J=null){J===null&&(J=w),R!==null&&R.renderStart(w,k,J),E=_e.get(J),E.init(k),v.push(E),J.traverseVisible(function(X){X.isLight&&X.layers.test(k.layers)&&(E.pushLight(X),X.castShadow&&E.pushShadow(X))}),w!==J&&w.traverseVisible(function(X){X.isLight&&X.layers.test(k.layers)&&(E.pushLight(X),X.castShadow&&E.pushShadow(X))}),E.setupLights(),R!==null&&R.updateLights(E.state.lightsArray),Ue=this.localClippingEnabled,de=V.init(this.clippingPlanes,Ue),de===!0&&V.setGlobalState(this.clippingPlanes,k),R!==null&&le.render(E.state.shadowsArray,J,k);let q=new Set;return w.traverse(function(X){if(!(X.isMesh||X.isPoints||X.isLine||X.isSprite))return;let ve=X.material;if(ve)if(Array.isArray(ve))for(let Re=0;Re<ve.length;Re++){let we=ve[Re];Il(we,J,k,X),q.add(we)}else Il(ve,J,k,X),q.add(ve)}),E=v.pop(),R!==null&&R.renderEnd(),q},this.compileAsync=function(w,k,J=null){let q=this.compile(w,k,J);return new Promise(X=>{function ve(){q.forEach(function(Re){let we=Ge.get(Re).currentProgram;(we===void 0||we.isReady())&&q.delete(Re)}),q.size===0?X(w):setTimeout(ve,10)}B.get("KHR_parallel_shader_compile")===null?setTimeout(ve,10):ve()})};let en=null;function ad(w){en&&en(w)}function Dl(){Hr.stop()}function Ul(){Hr.start()}let Hr=new Ru;Hr.setAnimationLoop(ad),typeof self<"u"&&Hr.setContext(self),this.setAnimationLoop=function(w){en=w,fe.setAnimationLoop(w),w===null?Hr.stop():Hr.start()},fe.addEventListener("sessionstart",Dl),fe.addEventListener("sessionend",Ul),this.render=function(w,k){if(k!==void 0&&k.isCamera!==!0){Qe("WebGLRenderer.render: camera is not an instance of THREE.Camera.");return}if(C===!0)return;R!==null&&R.renderStart(w,k);let J=fe.enabled===!0&&fe.isPresenting===!0,q=_!==null&&(D===null||J)&&_.begin(A,D);if(w.matrixWorldAutoUpdate===!0&&w.updateMatrixWorld(),k.parent===null&&k.matrixWorldAutoUpdate===!0&&k.updateMatrixWorld(),fe.enabled===!0&&fe.isPresenting===!0&&(_===null||_.isCompositing()===!1)&&(fe.cameraAutoUpdate===!0&&fe.updateCamera(k),k=fe.getCamera()),w.isScene===!0&&w.onBeforeRender(A,w,k,D),E=_e.get(w,v.length),E.init(k),E.state.textureUnits=je.getTextureUnits(),v.push(E),Ie.multiplyMatrices(k.projectionMatrix,k.matrixWorldInverse),k.isArrayCamera?he.setFromArrayCamera(k):oe.setFromProjectionMatrix(Ie,ia,k.reversedDepth),Ue=this.localClippingEnabled,de=V.init(this.clippingPlanes,Ue),b=ye.get(w,T.length),b.init(),T.push(b),this.info.render.frame++,fe.enabled===!0&&fe.isPresenting===!0){let ve=A.xr.getDepthSensingMesh();ve!==null&&tn(ve,k,-1/0,A.sortObjects)}tn(w,k,0,A.sortObjects),b.finish(),R!==null&&R.updateLights(E.state.lightsArray),A.sortObjects===!0&&b.sort(Q,ke),Te=fe.enabled===!1||fe.isPresenting===!1||fe.hasDepthSensing()===!1,Te&&Pe.addToRenderList(b,w),this.info.autoReset===!0&&this.info.reset(),de===!0&&V.beginShadows();let X=E.state.shadowsArray;if(le.render(X,w,k),de===!0&&V.endShadows(),(q&&_.hasRenderPass())===!1){let ve=b.opaque,Re=b.transmissive;if(E.setupLights(),k.isArrayCamera){let we=k.cameras;if(Re.length>0)for(let ze=0,qe=we.length;ze<qe;ze++){let ht=we[ze];zl(ve,Re,w,ht)}Te&&Pe.render(w);for(let ze=0,qe=we.length;ze<qe;ze++){let ht=we[ze];Ol(b,w,ht,ht.viewport)}}else Re.length>0&&zl(ve,Re,w,k),Te&&Pe.render(w),Ol(b,w,k)}D!==null&&H===0&&(je.updateMultisampleRenderTarget(D),je.updateRenderTargetMipmap(D)),q&&_.end(A),w.isScene===!0&&w.onAfterRender(A,w,k),Ae.resetDefaultState(),ae=-1,j=null,v.pop(),v.length>0?(E=v[v.length-1],je.setTextureUnits(E.state.textureUnits),de===!0&&V.setGlobalState(A.clippingPlanes,E.state.camera)):E=null,T.pop(),b=T.length>0?T[T.length-1]:null,R!==null&&R.renderEnd()};function tn(w,k,J,q){if(w.visible===!1)return;if(w.layers.test(k.layers)){if(w.isGroup)J=w.renderOrder;else if(w.isLOD)w.autoUpdate===!0&&w.update(k);else if(w.isLight)E.pushLight(w),w.castShadow&&E.pushShadow(w);else if(w.isSprite){let ve=k.isArrayCamera?he:oe;if(!w.frustumCulled||w.intersectsFrustum(ve)){q&&te.setFromMatrixPosition(w.matrixWorld).applyMatrix4(Ie);let Re=W.update(w),we=w.material;we.visible&&b.push(w,Re,we,J,te.z,null,k)}}else if(w.isMesh||w.isLine||w.isPoints){let ve=k.isArrayCamera?he:oe;if(!w.frustumCulled||w.intersectsFrustum(ve)){let Re=W.update(w),we=w.material;if(q&&(w.boundingSphere===void 0?(Re.boundingSphere===null&&Re.computeBoundingSphere(),te.copy(Re.boundingSphere.center)):(w.boundingSphere===null&&w.computeBoundingSphere(),te.copy(w.boundingSphere.center)),te.applyMatrix4(w.matrixWorld).applyMatrix4(Ie)),Array.isArray(we)){let ze=Re.groups;for(let qe=0,ht=ze.length;qe<ht;qe++){let vt=ze[qe],Ve=we[vt.materialIndex];Ve&&Ve.visible&&b.push(w,Re,Ve,J,te.z,vt,k)}}else we.visible&&b.push(w,Re,we,J,te.z,null,k)}}}let X=w.children;for(let ve=0,Re=X.length;ve<Re;ve++)tn(X[ve],k,J,q)}function Ol(w,k,J,q){let{opaque:X,transmissive:ve,transparent:Re}=w;E.setupLightsView(J),de===!0&&V.setGlobalState(A.clippingPlanes,J),q&&ne.viewport(Y.copy(q)),X.length>0&&Uo(X,k,J),ve.length>0&&Uo(ve,k,J),Re.length>0&&Uo(Re,k,J),ne.buffers.depth.setTest(!0),ne.buffers.depth.setMask(!0),ne.buffers.color.setMask(!0),ne.setPolygonOffset(!1)}function zl(w,k,J,q){if((J.isScene===!0?J.overrideMaterial:null)!==null)return;if(E.state.transmissionRenderTarget[q.id]===void 0){let vt=B.has("EXT_color_buffer_half_float")||B.has("EXT_color_buffer_float");E.state.transmissionRenderTarget[q.id]=new gi(1,1,{generateMipmaps:!0,type:vt?Rr:Fr,minFilter:Xa,samples:Math.max(4,Ce.samples),stencilBuffer:a,resolveDepthBuffer:!1,resolveStencilBuffer:!1,storeMultisampledDepthBuffer:!1,storeMultisampledStencilBuffer:!1,colorSpace:ft.workingColorSpace})}let X=E.state.transmissionRenderTarget[q.id],ve=q.viewport||Y;X.setSize(ve.z*A.transmissionResolutionScale,ve.w*A.transmissionResolutionScale);let Re=A.getRenderTarget(),we=A.getActiveCubeFace(),ze=A.getActiveMipmapLevel();A.setRenderTarget(X),A.getClearColor(K),re=A.getClearAlpha(),re<1&&A.setClearColor(16777215,.5),A.clear(),Te&&Pe.render(J);let qe=A.toneMapping;A.toneMapping=0;let ht=q.viewport;if(q.viewport!==void 0&&(q.viewport=void 0),E.setupLightsView(q),de===!0&&V.setGlobalState(A.clippingPlanes,q),Uo(w,J,q),je.updateMultisampleRenderTarget(X),je.updateRenderTargetMipmap(X),B.has("WEBGL_multisampled_render_to_texture")===!1){let vt=!1;for(let Ve=0,Tt=k.length;Ve<Tt;Ve++){let{object:qt,geometry:Lt,material:Mt,group:ni}=k[Ve];if(Mt.side===2&&qt.layers.test(q.layers)){let We=Mt.side;Mt.side=1,Mt.needsUpdate=!0,Fl(qt,J,q,Lt,Mt,ni),Mt.side=We,Mt.needsUpdate=!0,vt=!0}}vt===!0&&(je.updateMultisampleRenderTarget(X),je.updateRenderTargetMipmap(X))}A.setRenderTarget(Re,we,ze),A.setClearColor(K,re),ht!==void 0&&(q.viewport=ht),A.toneMapping=qe}function Uo(w,k,J){let q=k.isScene===!0?k.overrideMaterial:null;for(let X=0,ve=w.length;X<ve;X++){let Re=w[X],{object:we,geometry:ze,group:qe}=Re,ht=Re.material;ht.allowOverride===!0&&q!==null&&(ht=q),we.layers.test(J.layers)&&Fl(we,k,J,ze,ht,qe)}}function Fl(w,k,J,q,X,ve){R!==null&&X.isNodeMaterial&&R.setObject(w,X),w.onBeforeRender(A,k,J,q,X,ve),w.modelViewMatrix.multiplyMatrices(J.matrixWorldInverse,w.matrixWorld),w.normalMatrix.getNormalMatrix(w.modelViewMatrix),X.onBeforeRender(A,k,J,q,w,ve),X.transparent===!0&&X.side===2&&X.forceSinglePass===!1?(X.side=1,X.needsUpdate=!0,A.renderBufferDirect(J,k,q,X,w,ve),X.side=0,X.needsUpdate=!0,A.renderBufferDirect(J,k,q,X,w,ve),X.side=2):A.renderBufferDirect(J,k,q,X,w,ve),w.onAfterRender(A,k,J,q,X,ve)}function Bl(w){let k=0;for(let J of w)J.texture!==null&&k++;return k}function Ya(w,k,J){k.isScene!==!0&&(k=$);let q=Ge.get(w),X=E.state.lights,ve=E.state.shadowsArray,Re=X.state.version,we=ce.getParameters(w,X.state,ve,k,J,E.state.lightProbeGridArray),ze=ce.getProgramCacheKey(we),qe=q.programs;q.environment=w.isMeshStandardMaterial||w.isMeshLambertMaterial||w.isMeshPhongMaterial?k.environment:null,q.fog=k.fog;let ht=w.isMeshStandardMaterial||w.isMeshLambertMaterial&&!w.envMap||w.isMeshPhongMaterial&&!w.envMap;q.envMap=At.get(w.envMap||q.environment,ht),q.envMapRotation=q.environment!==null&&w.envMap===null?k.environmentRotation:w.envMapRotation,qe===void 0&&(w.addEventListener("dispose",Oe),qe=new Map,q.programs=qe,q.programVariants=new Map);let vt=qe.get(ze);if(vt!==void 0){if(q.currentProgram===vt&&q.lightsStateVersion===Re)return Vl(w,we),vt}else we.uniforms=ce.getUniforms(w),R!==null&&w.isNodeMaterial&&R.build(w,J,we),w.onBeforeCompile(we,A),vt=ce.acquireProgram(we,ze),qe.set(ze,vt),q.uniforms=we.uniforms;let Ve=q.uniforms;return(!w.isShaderMaterial&&!w.isRawShaderMaterial||w.clipping===!0)&&(Ve.clippingPlanes=V.uniform),Vl(w,we),q.needsLights=ld(w),q.lightsStateVersion=Re,q.needsLights&&(Ve.ambientLightColor.value=X.state.ambient,Ve.lightProbe.value=X.state.probe,Ve.sunLights.value=X.state.sun,Ve.sunLightShadows.value=X.state.sunShadow,Ve.directionalLights.value=X.state.directional,Ve.directionalLightShadows.value=X.state.directionalShadow,Ve.spotLights.value=X.state.spot,Ve.spotLightShadows.value=X.state.spotShadow,Ve.rectAreaLights.value=X.state.rectArea,Ve.ltc_1.value=X.state.rectAreaLTC1,Ve.ltc_2.value=X.state.rectAreaLTC2,Ve.pointLights.value=X.state.point,Ve.pointLightShadows.value=X.state.pointShadow,Ve.hemisphereLights.value=X.state.hemi,Ve.sunShadowMatrix.value=X.state.sunShadowMatrix,Ve.sunShadowCascade.value=X.state.sunShadowCascade,Ve.directionalShadowMatrix.value=X.state.directionalShadowMatrix,Ve.spotLightMatrix.value=X.state.spotLightMatrix,Ve.spotLightMap.value=X.state.spotLightMap,Ve.pointShadowMatrix.value=X.state.pointShadowMatrix),q.numLightProbeGrids=Bl(E.state.lightProbeGridArray),q.currentProgram=vt,q.uniformsList=null,vt}function od(w,k,J,q,X){let ve=Ge.get(w),Re=ve.currentProgram.id+","+q+","+X,we=ve.programVariants.get(Re);return we===void 0?(we=Ya(w,k,J),ve.programVariants.set(Re,we)):(ve.currentProgram=we,ve.uniformsList=null,ve.outputColorSpace=q,ve.toneMapping=X),we}function kl(w){if(w.uniformsList===null){let k=w.currentProgram.getUniforms();w.uniformsList=ws.seqWithValue(k.seq,w.uniforms)}return w.uniformsList}function Vl(w,k){let J=Ge.get(w);J.outputColorSpace=k.outputColorSpace,J.batching=k.batching,J.batchingColor=k.batchingColor,J.instancing=k.instancing,J.instancingColor=k.instancingColor,J.instancingMorph=k.instancingMorph,J.skinning=k.skinning,J.morphTargets=k.morphTargets,J.morphNormals=k.morphNormals,J.morphColors=k.morphColors,J.morphTargetsCount=k.morphTargetsCount,J.numClippingPlanes=k.numClippingPlanes,J.numIntersection=k.numClipIntersection,J.vertexAlphas=k.vertexAlphas,J.vertexTangents=k.vertexTangents,J.toneMapping=k.toneMapping}function sd(w,k,J,q,X){k.isScene!==!0&&(k=$),je.resetTextureUnits();let ve=k.fog,Re=q.isMeshStandardMaterial||q.isMeshLambertMaterial||q.isMeshPhongMaterial?k.environment:null,we=D===null?A.outputColorSpace:D.isXRRenderTarget===!0?D.texture.colorSpace:ft.workingColorSpace,ze=q.isMeshStandardMaterial||q.isMeshLambertMaterial&&!q.envMap||q.isMeshPhongMaterial&&!q.envMap,qe=At.get(q.envMap||Re,ze),ht=q.vertexColors===!0&&!!J.attributes.color&&J.attributes.color.itemSize===4,vt=!!J.attributes.tangent&&(!!q.normalMap||q.anisotropy>0),Ve=!!J.morphAttributes.position,Tt=!!J.morphAttributes.normal,qt=!!J.morphAttributes.color,Lt=0;q.toneMapped&&(D===null||D.isXRRenderTarget===!0)&&(Lt=A.toneMapping);let Mt=J.morphAttributes.position||J.morphAttributes.normal||J.morphAttributes.color,ni=Mt===void 0?0:Mt.length,We=Ge.get(q),Si=E.state.lights;if(de===!0&&(Ue===!0||w!==j)){let Dt=w===j&&q.id===ae;V.setState(q,w,Dt)}let Qi=!1,Wr=!1;q.version===We.__version?We.needsLights&&We.lightsStateVersion!==Si.state.version||X.isBatchedMesh&&We.batching===!1||!X.isBatchedMesh&&We.batching===!0||X.isBatchedMesh&&We.batchingColor===!0&&X._colorsTexture===null||X.isBatchedMesh&&We.batchingColor===!1&&X._colorsTexture!==null||X.isInstancedMesh&&We.instancing===!1||!X.isInstancedMesh&&We.instancing===!0||X.isSkinnedMesh&&We.skinning===!1||!X.isSkinnedMesh&&We.skinning===!0||X.isInstancedMesh&&We.instancingColor===!0&&X.instanceColor===null||X.isInstancedMesh&&We.instancingColor===!1&&X.instanceColor!==null||X.isInstancedMesh&&We.instancingMorph===!0&&X.morphTexture===null||X.isInstancedMesh&&We.instancingMorph===!1&&X.morphTexture!==null?Qi=!0:We.envMap===qe?q.fog===!0&&We.fog!==ve||We.numClippingPlanes!==void 0&&(We.numClippingPlanes!==V.numPlanes||We.numIntersection!==V.numIntersection)?Qi=!0:We.vertexAlphas===ht&&We.vertexTangents===vt&&We.morphTargets===Ve&&We.morphNormals===Tt&&We.morphColors===qt&&We.morphTargetsCount===ni&&We.numLightProbeGrids===Bl(E.state.lightProbeGridArray)?(We.outputColorSpace!==we||We.toneMapping!==Lt)&&(Wr=!0):Qi=!0:Qi=!0:(Qi=!0,We.__version=q.version);let li=We.currentProgram;(Qi===!0||Wr===!0)&&(li=Wr===!0?od(q,k,X,we,Lt):Ya(q,k,X),R&&q.isNodeMaterial&&R.onUpdateProgram(q,li,We));let Oo=!1,Pr=!1,Xr=!1,xt=li.getUniforms(),ii=We.uniforms;if(ne.useProgram(li.program)&&(Oo=!0,Pr=!0,Xr=!0),q.id!==ae&&(ae=q.id,Pr=!0),Oo||j!==w){ne.buffers.depth.getReversed()&&w.reversedDepth!==!0&&(w._reversedDepth=!0,w.updateProjectionMatrix()),xt.setValue(U,"projectionMatrix",w.projectionMatrix),xt.setValue(U,"viewMatrix",w.matrixWorldInverse);let Dt=xt.map.cameraPosition;Dt!==void 0&&Dt.setValue(U,F.setFromMatrixPosition(w.matrixWorld)),Ce.logarithmicDepthBuffer&&xt.setValue(U,"logDepthBufFC",2/(Math.log(w.far+1)/Math.LN2)),(q.isMeshPhongMaterial||q.isMeshToonMaterial||q.isMeshLambertMaterial||q.isMeshBasicMaterial||q.isMeshStandardMaterial||q.isShaderMaterial)&&xt.setValue(U,"isOrthographic",w.isOrthographicCamera===!0),j!==w&&(j=w,Pr=!0,Xr=!0)}if(We.needsLights&&(Si.state.sunShadowMap.length>0&&xt.setValue(U,"sunShadowMap",Si.state.sunShadowMap,je),Si.state.directionalShadowMap.length>0&&xt.setValue(U,"directionalShadowMap",Si.state.directionalShadowMap,je),Si.state.spotShadowMap.length>0&&xt.setValue(U,"spotShadowMap",Si.state.spotShadowMap,je),Si.state.pointShadowMap.length>0&&xt.setValue(U,"pointShadowMap",Si.state.pointShadowMap,je)),X.isSkinnedMesh){xt.setOptional(U,X,"bindMatrix"),xt.setOptional(U,X,"bindMatrixInverse");let Dt=X.skeleton;Dt&&(Dt.boneTexture===null&&Dt.computeBoneTexture(),xt.setValue(U,"boneTexture",Dt.boneTexture,je))}X.isBatchedMesh&&(xt.setOptional(U,X,"batchingTexture"),xt.setValue(U,"batchingTexture",X._matricesTexture,je),xt.setOptional(U,X,"batchingIdTexture"),xt.setValue(U,"batchingIdTexture",X._indirectTexture,je),xt.setOptional(U,X,"batchingColorTexture"),X._colorsTexture!==null&&xt.setValue(U,"batchingColorTexture",X._colorsTexture,je));let Fi=J.morphAttributes;if((Fi.position!==void 0||Fi.normal!==void 0||Fi.color!==void 0)&&O.update(X,J,li),(Pr||We.receiveShadow!==X.receiveShadow)&&(We.receiveShadow=X.receiveShadow,xt.setValue(U,"receiveShadow",X.receiveShadow)),(q.isMeshStandardMaterial||q.isMeshLambertMaterial||q.isMeshPhongMaterial)&&q.envMap===null&&k.environment!==null&&(ii.envMapIntensity.value=k.environmentIntensity),ii.dfgLUT!==void 0&&(ii.dfgLUT.value=l1()),Pr){if(xt.setValue(U,"toneMappingExposure",A.toneMappingExposure),We.needsLights&&nd(ii,Xr),ve&&q.fog===!0&&ee.refreshFogUniforms(ii,ve),ee.refreshMaterialUniforms(ii,q,Me,Ne,E.state.transmissionRenderTarget[w.id]),We.needsLights&&We.numLightProbeGrids>0){let Dt=M,fi=0;for(let gr of E.state.lightProbeGridArray)gr.texture!==null&&(Dt.textures[fi]=gr.texture,Dt.min[fi]=gr.boundingBox.min,Dt.max[fi]=gr.boundingBox.max,Dt.resolution[fi]=gr.resolution,fi++);for(let gr in Dt)Dt[gr].length=fi;ii.probesSH.value=Dt.textures,ii.probesMin.value=Dt.min,ii.probesMax.value=Dt.max,ii.probesResolution.value=Dt.resolution}ws.upload(U,kl(We),ii,je)}if(q.isShaderMaterial&&q.uniformsNeedUpdate===!0&&(ws.upload(U,kl(We),ii,je),q.uniformsNeedUpdate=!1),q.isSpriteMaterial&&xt.setValue(U,"center",X.center),xt.setValue(U,"modelViewMatrix",X.modelViewMatrix),xt.setValue(U,"normalMatrix",X.normalMatrix),xt.setValue(U,"modelMatrix",X.matrixWorld),q.uniformsGroups!==void 0){let Dt=q.uniformsGroups;for(let fi=0,gr=Dt.length;fi<gr;fi++){let Hl=Dt[fi];Ee.update(Hl,li),Ee.bind(Hl,li)}}return li}function nd(w,k){w.ambientLightColor.needsUpdate=k,w.lightProbe.needsUpdate=k,w.sunLights.needsUpdate=k,w.sunLightShadows.needsUpdate=k,w.directionalLights.needsUpdate=k,w.directionalLightShadows.needsUpdate=k,w.pointLights.needsUpdate=k,w.pointLightShadows.needsUpdate=k,w.spotLights.needsUpdate=k,w.spotLightShadows.needsUpdate=k,w.rectAreaLights.needsUpdate=k,w.hemisphereLights.needsUpdate=k}function ld(w){return w.isMeshLambertMaterial||w.isMeshToonMaterial||w.isMeshPhongMaterial||w.isMeshStandardMaterial||w.isShadowMaterial||w.isShaderMaterial&&w.lights===!0}this.getActiveCubeFace=function(){return z},this.getActiveMipmapLevel=function(){return H},this.getRenderTarget=function(){return D},this.setRenderTargetTextures=function(w,k,J){let q=Ge.get(w);q.__autoAllocateDepthBuffer=w.resolveDepthBuffer===!1,q.__autoAllocateDepthBuffer===!1&&(q.__useRenderToTexture=!1),Ge.get(w.texture).__webglTexture=k,Ge.get(w.depthTexture).__webglTexture=q.__autoAllocateDepthBuffer?void 0:J,q.__hasExternalTextures=!0},this.setRenderTargetFramebuffer=function(w,k){let J=Ge.get(w);J.__webglFramebuffer=k,J.__useDefaultFramebuffer=k===void 0},this.setRenderTarget=function(w,k=0,J=0){D=w,z=k,H=J;let q=null,X=!1,ve=!1;if(w){let Re=Ge.get(w);if(Re.__useDefaultFramebuffer!==void 0){ne.bindFramebuffer(U.FRAMEBUFFER,Re.__webglFramebuffer),Y.copy(w.viewport),ie.copy(w.scissor),Se=w.scissorTest,ne.viewport(Y),ne.scissor(ie),ne.setScissorTest(Se),ae=-1;return}if(Re.__webglFramebuffer===void 0)je.setupRenderTarget(w);else if(Re.__hasExternalTextures)je.rebindTextures(w,Ge.get(w.texture).__webglTexture,Ge.get(w.depthTexture).__webglTexture);else if(w.depthBuffer){let qe=w.depthTexture;if(Re.__boundDepthTexture!==qe){if(qe!==null&&Ge.has(qe)&&(w.width!==qe.image.width||w.height!==qe.image.height))throw Error("THREE.WebGLRenderer: Attached DepthTexture is initialized to the incorrect size.");je.setupDepthRenderbuffer(w)}}let we=w.texture;(we.isData3DTexture||we.isDataArrayTexture||we.isCompressedArrayTexture)&&(ve=!0);let ze=Ge.get(w).__webglFramebuffer;w.isWebGLCubeRenderTarget?(q=Array.isArray(ze[k])?ze[k][J]:ze[k],X=!0):q=w.samples>0&&je.useMultisampledRTT(w)===!1?Ge.get(w).__webglMultisampledFramebuffer:Array.isArray(ze)?ze[J]:ze,Y.copy(w.viewport),ie.copy(w.scissor),Se=w.scissorTest}else Y.copy($e).multiplyScalar(Me).floor(),ie.copy(De).multiplyScalar(Me).floor(),Se=nt;if(J!==0&&(q=L),ne.bindFramebuffer(U.FRAMEBUFFER,q)&&ne.drawBuffers(w,q),ne.viewport(Y),ne.scissor(ie),ne.setScissorTest(Se),X){let Re=Ge.get(w.texture);U.framebufferTexture2D(U.FRAMEBUFFER,U.COLOR_ATTACHMENT0,U.TEXTURE_CUBE_MAP_POSITIVE_X+k,Re.__webglTexture,J)}else if(ve){let Re=k;for(let we=0;we<w.textures.length;we++){let ze=Ge.get(w.textures[we]);U.framebufferTextureLayer(U.FRAMEBUFFER,U.COLOR_ATTACHMENT0+we,ze.__webglTexture,J,Re)}}else if(w!==null&&J!==0){let Re=Ge.get(w.texture);U.framebufferTexture2D(U.FRAMEBUFFER,U.COLOR_ATTACHMENT0,U.TEXTURE_2D,Re.__webglTexture,J)}ae=-1};function Gl(w){let k=Ge.get(w);return(k.__readFormat!==w.format||k.__readType!==w.type)&&(k.__readFormat=w.format,k.__readType=w.type,k.__formatReadable=Ce.textureFormatReadable(w.format),k.__typeReadable=Ce.textureTypeReadable(w.type)),k}this.readRenderTargetPixels=function(w,k,J,q,X,ve,Re,we=0){if(!(w&&w.isWebGLRenderTarget)){Qe("WebGLRenderer.readRenderTargetPixels: renderTarget is not THREE.WebGLRenderTarget.");return}let ze=Ge.get(w).__webglFramebuffer;if(w.isWebGLCubeRenderTarget&&Re!==void 0&&(ze=ze[Re],Array.isArray(ze)&&(ze=ze[0])),ze){ne.bindFramebuffer(U.FRAMEBUFFER,ze);try{let qe=w.textures[we],ht=qe.format,vt=qe.type;w.textures.length>1&&U.readBuffer(U.COLOR_ATTACHMENT0+we);let Ve=Gl(qe);if(Ve.__formatReadable===!1){Qe("WebGLRenderer.readRenderTargetPixels: renderTarget is not in RGBA or implementation defined format.");return}if(Ve.__typeReadable===!1){Qe("WebGLRenderer.readRenderTargetPixels: renderTarget is not in UnsignedByteType or implementation defined type.");return}k>=0&&k<=w.width-q&&J>=0&&J<=w.height-X&&U.readPixels(k,J,q,X,me.convert(ht),me.convert(vt),ve)}finally{let qe=D===null?null:Ge.get(D).__webglFramebuffer;ne.bindFramebuffer(U.FRAMEBUFFER,qe)}}},this.readRenderTargetPixelsAsync=async function(w,k,J,q,X,ve,Re,we=0){if(!(w&&w.isWebGLRenderTarget))throw Error("THREE.WebGLRenderer.readRenderTargetPixels: renderTarget is not THREE.WebGLRenderTarget.");let ze=Ge.get(w).__webglFramebuffer;if(w.isWebGLCubeRenderTarget&&Re!==void 0&&(ze=ze[Re],Array.isArray(ze)&&(ze=ze[0])),ze){if(k>=0&&k<=w.width-q&&J>=0&&J<=w.height-X){ne.bindFramebuffer(U.FRAMEBUFFER,ze);let qe=w.textures[we],ht=qe.format,vt=qe.type;w.textures.length>1&&U.readBuffer(U.COLOR_ATTACHMENT0+we);let Ve=Gl(qe);if(Ve.__formatReadable===!1)throw Error("THREE.WebGLRenderer.readRenderTargetPixelsAsync: renderTarget is not in RGBA or implementation defined format.");if(Ve.__typeReadable===!1)throw Error("THREE.WebGLRenderer.readRenderTargetPixelsAsync: renderTarget is not in UnsignedByteType or implementation defined type.");let Tt=U.createBuffer();U.bindBuffer(U.PIXEL_PACK_BUFFER,Tt),U.bufferData(U.PIXEL_PACK_BUFFER,ve.byteLength,U.STREAM_READ),U.readPixels(k,J,q,X,me.convert(ht),me.convert(vt),0),U.bindBuffer(U.PIXEL_PACK_BUFFER,null);let qt=D===null?null:Ge.get(D).__webglFramebuffer;ne.bindFramebuffer(U.FRAMEBUFFER,qt);let Lt=U.fenceSync(U.SYNC_GPU_COMMANDS_COMPLETE,0);return U.flush(),await s0(U,Lt,4),U.bindBuffer(U.PIXEL_PACK_BUFFER,Tt),U.getBufferSubData(U.PIXEL_PACK_BUFFER,0,ve),U.bindBuffer(U.PIXEL_PACK_BUFFER,null),U.deleteBuffer(Tt),U.deleteSync(Lt),ve}throw Error("THREE.WebGLRenderer.readRenderTargetPixelsAsync: requested read bounds are out of range.")}},this.copyFramebufferToTexture=function(w,k=null,J=0){let q=2**-J,X=Math.floor(w.image.width*q),ve=Math.floor(w.image.height*q),Re=k===null?0:k.x,we=k===null?0:k.y;je.setTexture2D(w,0),U.copyTexSubImage2D(U.TEXTURE_2D,J,0,0,Re,we,X,ve),ne.unbindTexture()},this.copyTextureToTexture=function(w,k,J=null,q=null,X=0,ve=0){let Re,we,ze,qe,ht,vt,Ve,Tt,qt,Lt=w.isCompressedTexture?w.mipmaps[ve]:w.image;if(J!==null)Re=J.max.x-J.min.x,we=J.max.y-J.min.y,ze=J.isBox3?J.max.z-J.min.z:1,qe=J.min.x,ht=J.min.y,vt=J.isBox3?J.min.z:0;else{let xt=2**-X;Re=Math.floor(Lt.width*xt),we=Math.floor(Lt.height*xt),ze=w.isDataArrayTexture?Lt.depth:w.isData3DTexture?Math.floor(Lt.depth*xt):1,qe=0,ht=0,vt=0}q===null?(Ve=0,Tt=0,qt=0):(Ve=q.x,Tt=q.y,qt=q.z);let Mt=me.convert(k.format),ni=me.convert(k.type),We;k.isData3DTexture?(je.setTexture3D(k,0),We=U.TEXTURE_3D):k.isDataArrayTexture||k.isCompressedArrayTexture?(je.setTexture2DArray(k,0),We=U.TEXTURE_2D_ARRAY):(je.setTexture2D(k,0),We=U.TEXTURE_2D),ne.activeTexture(U.TEXTURE0),ne.pixelStorei(U.UNPACK_FLIP_Y_WEBGL,k.flipY),ne.pixelStorei(U.UNPACK_PREMULTIPLY_ALPHA_WEBGL,k.premultiplyAlpha),ne.pixelStorei(U.UNPACK_ALIGNMENT,k.unpackAlignment);let Si=ne.getParameter(U.UNPACK_ROW_LENGTH),Qi=ne.getParameter(U.UNPACK_IMAGE_HEIGHT),Wr=ne.getParameter(U.UNPACK_SKIP_PIXELS),li=ne.getParameter(U.UNPACK_SKIP_ROWS),Oo=ne.getParameter(U.UNPACK_SKIP_IMAGES);ne.pixelStorei(U.UNPACK_ROW_LENGTH,Lt.width),ne.pixelStorei(U.UNPACK_IMAGE_HEIGHT,Lt.height),ne.pixelStorei(U.UNPACK_SKIP_PIXELS,qe),ne.pixelStorei(U.UNPACK_SKIP_ROWS,ht),ne.pixelStorei(U.UNPACK_SKIP_IMAGES,vt);let Pr=w.isDataArrayTexture||w.isData3DTexture,Xr=k.isDataArrayTexture||k.isData3DTexture;if(w.isDepthTexture){let xt=Ge.get(w),ii=Ge.get(k),Fi=Ge.get(xt.__renderTarget),Dt=Ge.get(ii.__renderTarget);ne.bindFramebuffer(U.READ_FRAMEBUFFER,Fi.__webglFramebuffer),ne.bindFramebuffer(U.DRAW_FRAMEBUFFER,Dt.__webglFramebuffer);for(let fi=0;fi<ze;fi++)Pr&&(U.framebufferTextureLayer(U.READ_FRAMEBUFFER,U.COLOR_ATTACHMENT0,Ge.get(w).__webglTexture,X,vt+fi),U.framebufferTextureLayer(U.DRAW_FRAMEBUFFER,U.COLOR_ATTACHMENT0,Ge.get(k).__webglTexture,ve,qt+fi)),U.blitFramebuffer(qe,ht,qe+Re,ht+we,Ve,Tt,Ve+Re,Tt+we,U.DEPTH_BUFFER_BIT,U.NEAREST);ne.bindFramebuffer(U.READ_FRAMEBUFFER,null),ne.bindFramebuffer(U.DRAW_FRAMEBUFFER,null)}else if(X!==0||w.isRenderTargetTexture||Ge.has(w)){let xt=Ge.get(w),ii=Ge.get(k);ne.bindFramebuffer(U.READ_FRAMEBUFFER,G),ne.bindFramebuffer(U.DRAW_FRAMEBUFFER,P);for(let Fi=0;Fi<ze;Fi++)Pr?U.framebufferTextureLayer(U.READ_FRAMEBUFFER,U.COLOR_ATTACHMENT0,xt.__webglTexture,X,vt+Fi):U.framebufferTexture2D(U.READ_FRAMEBUFFER,U.COLOR_ATTACHMENT0,U.TEXTURE_2D,xt.__webglTexture,X),Xr?U.framebufferTextureLayer(U.DRAW_FRAMEBUFFER,U.COLOR_ATTACHMENT0,ii.__webglTexture,ve,qt+Fi):U.framebufferTexture2D(U.DRAW_FRAMEBUFFER,U.COLOR_ATTACHMENT0,U.TEXTURE_2D,ii.__webglTexture,ve),X===0?Xr?U.copyTexSubImage3D(We,ve,Ve,Tt,qt+Fi,qe,ht,Re,we):U.copyTexSubImage2D(We,ve,Ve,Tt,qe,ht,Re,we):U.blitFramebuffer(qe,ht,qe+Re,ht+we,Ve,Tt,Ve+Re,Tt+we,U.COLOR_BUFFER_BIT,U.NEAREST);ne.bindFramebuffer(U.READ_FRAMEBUFFER,null),ne.bindFramebuffer(U.DRAW_FRAMEBUFFER,null)}else Xr?w.isDataTexture||w.isData3DTexture?U.texSubImage3D(We,ve,Ve,Tt,qt,Re,we,ze,Mt,ni,Lt.data):k.isCompressedArrayTexture?U.compressedTexSubImage3D(We,ve,Ve,Tt,qt,Re,we,ze,Mt,Lt.data):U.texSubImage3D(We,ve,Ve,Tt,qt,Re,we,ze,Mt,ni,Lt):w.isDataTexture?U.texSubImage2D(U.TEXTURE_2D,ve,Ve,Tt,Re,we,Mt,ni,Lt.data):w.isCompressedTexture?U.compressedTexSubImage2D(U.TEXTURE_2D,ve,Ve,Tt,Lt.width,Lt.height,Mt,Lt.data):U.texSubImage2D(U.TEXTURE_2D,ve,Ve,Tt,Re,we,Mt,ni,Lt);ne.pixelStorei(U.UNPACK_ROW_LENGTH,Si),ne.pixelStorei(U.UNPACK_IMAGE_HEIGHT,Qi),ne.pixelStorei(U.UNPACK_SKIP_PIXELS,Wr),ne.pixelStorei(U.UNPACK_SKIP_ROWS,li),ne.pixelStorei(U.UNPACK_SKIP_IMAGES,Oo),ve===0&&k.generateMipmaps===!0&&k.mipmapsAutoUpdate===!0&&U.generateMipmap(We),ne.unbindTexture()},this.initRenderTarget=function(w){Ge.get(w).__webglFramebuffer===void 0&&je.setupRenderTarget(w)},this.initTexture=function(w){w.isCubeTexture?je.setTextureCube(w,0):w.isData3DTexture?je.setTexture3D(w,0):w.isDataArrayTexture||w.isCompressedArrayTexture?je.setTexture2DArray(w,0):je.setTexture2D(w,0),ne.unbindTexture()},this.resetState=function(){z=0,H=0,D=null,ne.reset(),Ae.reset()},typeof __THREE_DEVTOOLS__<"u"&&__THREE_DEVTOOLS__.dispatchEvent(new CustomEvent("observe",{detail:this}))}get coordinateSystem(){return ia}get outputColorSpace(){return this._outputColorSpace}set outputColorSpace(e){this._outputColorSpace=e;let t=this.getContext();t.drawingBufferColorSpace=ft._getDrawingBufferColorSpace(e),t.unpackColorSpace=ft._getUnpackColorSpace()}};
```
@@END:engine.js@@

### core.js

@@BEGIN:core.js@@
```javascript
var Xe=Math.PI*2,Yt=(e,t,i)=>e<t?t:e>i?i:e,c1=e=>e<0?0:e>1?1:e,st=(e,t,i)=>e+(t-e)*i,bt=(e,t,i)=>{
let r=c1((i-e)/(t-e));
return r*r*(3-2*r)}
,Et=(e,t,i,r)=>st(e,t,1-Math.exp(-i*r)),u1=(e,t,i,r)=>e+(((t-e+Math.PI)%Xe+Xe)%Xe-Math.PI)*(1-Math.exp(-i*r));
function Mi(e){
let t=e>>>0,i=()=>{
t=t+1831565813|0;
let r=Math.imul(t^t>>>15,1|t);
return r=r+Math.imul(r^r>>>7,61|r)^r,((r^r>>>14)>>>0)/4294967296}
;
return i.range=(r,a)=>r+(a-r)*i(),i.int=(r,a)=>Math.floor(r+(a-r+1)*i()),i.pick=r=>r[Math.floor(i()*r.length)],i.chance=r=>i()<r,i.sign=()=>i()<.5?-1:1,i}
function hs(e,t){
let i=Math.imul(e|0,374761393)+Math.imul(t|0,668265263);
return i=Math.imul(i^i>>>13,1274126177),((i^i>>>16)>>>0)/4294967296}
function To(e,t){
let i=Math.floor(e),r=Math.floor(t),a=e-i,o=t-r,s=a*a*(3-2*a),n=o*o*(3-2*o),l=hs(i,r),h=hs(i+1,r),c=hs(i,r+1),u=hs(i+1,r+1);
return l+(h-l)*s+(c-l)*n+(l-h-c+u)*s*n}
function ka(e,t,i=4){
let r=0,a=.5,o=1,s=0;
for(let n=0;
n<i;
n++)r+=a*To(e*o+n*17.3,t*o-n*9.1),s+=a,a*=.5,o*=2.03;
return r/s}
function Rt(e){
if(Array.isArray(e))return e;
let t=parseInt(e.replace("#",""),16);
return[(t>>16&255)/255,(t>>8&255)/255,(t&255)/255]}
var Ni=(e,t,i)=>[st(e[0],t[0],i),st(e[1],t[1],i),st(e[2],t[2],i)];
function Va(e,t,i,r,a,o){
let s=a-i,n=o-r,l=s*s+n*n,h=l>0?((e-i)*s+(t-r)*n)/l:0;
h=h<0?0:h>1?1:h;
let c=i+s*h-e,u=r+n*h-t;
return Math.sqrt(c*c+u*u)}
function ku(e,t=1,i=!1){
let r=e;
for(let a=0;
a<t;
a++){
let o=[],s=r.length;
i||o.push(r[0]);
let n=i?s:s-1;
for(let l=0;
l<n;
l++){
let h=r[l],c=r[(l+1)%s];
o.push([h[0]*.75+c[0]*.25,h[1]*.75+c[1]*.25]),o.push([h[0]*.25+c[0]*.75,h[1]*.25+c[1]*.75])}
i||o.push(r[s-1]),r=o}
return r}
ft.enabled=!1;
var Ei=(e,t,i)=>new I(e,t,i),et={
uTime:{
value:0}
,uSunDir:{
value:Ei(-.7,.3,.45).normalize()}
,uSunPos:{
value:Ei(-.7,.3,.45).normalize()}
,uMoonPos:{
value:Ei(.4,.6,.5).normalize()}
,uSunTint:{
value:Ei(1.1,1,.85)}
,uShadeTint:{
value:Ei(.55,.52,.74)}
,uSkyAmb:{
value:Ei(.05,.07,.12)}
,uFogCol:{
value:Ei(.75,.8,.9)}
,uFogSun:{
value:Ei(1,.85,.6)}
,uFogParams:{
value:new Ft(22e-5,500,.96,0)}
,uZenith:{
value:Ei(.3,.5,.8)}
,uMidSky:{
value:Ei(.5,.7,.9)}
,uSunDisc:{
value:Ei(1,.95,.8)}
,uCloudLit:{
value:Ei(1,.97,.9)}
,uCloudShade:{
value:Ei(.75,.72,.85)}
,uNight:{
value:0}
,uStars:{
value:0}
,uShadowMap:{
value:null}
,uShadowMatrix:{
value:new gt}
,uShadowParams:{
value:new Ft(1/2048,.0012,1,.6)}
,uNoise:{
value:null}
,uHatch:{
value:.55}
,uWind:{
value:new ge(1,.3)}
}
;
function d1(e,t,i){
let r=new Float32Array(t*t);
for(let o=0;
o<r.length;
o++)r[o]=i();
let a=new Float32Array(e*e);
for(let o=0;
o<e;
o++)for(let s=0;
s<e;
s++){
let n=s/e*t,l=o/e*t,h=Math.floor(n),c=Math.floor(l),u=n-h,d=l-c,f=u*u*(3-2*u),p=d*d*(3-2*d),x=(h+1)%t,g=(c+1)%t,m=r[c*t+h],y=r[c*t+x],M=r[g*t+h],b=r[g*t+x];
a[o*e+s]=m+(y-m)*f+(M-m)*p+(m-y-M+b)*f*p}
return a}
function In(e,t,i,r){
let a=new Float32Array(e*e),o=.5,s=0,n=t;
for(let l=0;
l<i;
l++){
let h=d1(e,n,r);
for(let c=0;
c<a.length;
c++)a[c]+=h[c]*o;
s+=o,o*=.5,n*=2}
for(let l=0;
l<a.length;
l++)a[l]/=s;
return a}
function f1(){
let e=Mi(1709),t=In(256,4,5,e),i=In(256,6,4,e),r=In(256,16,3,e),a=new Uint8Array(262144),o=c=>{
let u=1,d=0;
for(let f of c)f<u&&(u=f),f>d&&(d=f);
return f=>(f-u)/(d-u)}
,s=o(t),n=o(i),l=o(r);
for(let c=0;
c<65536;
c++)a[c*4]=s(t[c])*255,a[c*4+1]=n(i[c])*255,a[c*4+2]=l(r[c])*255,a[c*4+3]=e()*255;
let h=new No(a,256,256,yo);
return h.wrapS=h.wrapT=Rs,h.magFilter=jt,h.minFilter=Xa,h.generateMipmaps=!0,h.needsUpdate=!0,h}
et.uNoise.value=f1();
var Eo=`
uniform float uTime;
uniform vec3 uSunDir, uSunPos, uMoonPos;
uniform vec3 uSunTint, uShadeTint, uSkyAmb;
uniform vec3 uFogCol, uFogSun;
uniform vec4 uFogParams;
uniform float uNight;
uniform sampler2D uNoise;

vec3 atmo(vec3 dir) {
  vec2 d = normalize(dir.xz + 1e-5);
  vec2 s = normalize(uSunPos.xz + 1e-5);
  float a = max(dot(d, s), 0.0);
  a = pow(a, uFogParams.w);
  return mix(uFogCol, uFogSun, a);
}
float fogAmt(float dist, float y) {
  float f = 1.0 - exp(-pow(dist * uFogParams.x, 1.3));
  f *= mix(1.0, 0.62, smoothstep(0.0, uFogParams.y, y));
  return min(f, uFogParams.z);
}
float hash12(vec2 p) {
  vec3 p3 = fract(vec3(p.xyx) * .1031);
  p3 += dot(p3, p3.yzx + 33.33);
  return fract((p3.x + p3.y) * p3.z);
}
`,p1=`
${Eo}
uniform mat4 uShadowMatrix;
uniform vec4 uShadowParams;
uniform vec2 uWind;
varying vec3 vWorldPos;
varying vec3 vNormalW;
varying vec3 vNormalV;
varying vec4 vColor;
varying float vViewZ;
varying vec3 vShadowCoord;
varying vec3 vObj;      // object-space position scaled to world units
varying vec3 vObjN;     // object-space normal
varying vec4 vSeed;     // per-instance random
/*VPARS*/
void main() {
  vec3 pos = position;
  vec3 nrm = normal;
  vec4 col = vec4(1.0);
  #ifdef USE_COLOR_ALPHA
    col = color;
  #elif defined(USE_COLOR)
    col.rgb = color;
  #endif
  mat4 m = modelMatrix;
  vec3 iscale = vec3(1.0);
  vSeed = vec4(0.0);
  #ifdef USE_INSTANCING
    m = modelMatrix * instanceMatrix;
    iscale = vec3(length(instanceMatrix[0].xyz), length(instanceMatrix[1].xyz), length(instanceMatrix[2].xyz));
    vec2 ip = instanceMatrix[3].xz;
    vSeed = vec4(hash12(ip), hash12(ip + 17.7), hash12(ip * 1.3 + 4.1), hash12(ip * 0.7 - 9.3));
  #endif
  vec3 icol = vec3(1.0);
  #ifdef USE_INSTANCING_COLOR
    icol = instanceColor;
  #endif
  /*VPRE*/
  col.rgb *= icol;
  vObj = pos * iscale;
  vObjN = nrm;
  vec4 wp = m * vec4(pos, 1.0);
  mat3 m3 = mat3(m);
  vec3 nw = normalize(m3 * (nrm / vec3(dot(m3[0], m3[0]), dot(m3[1], m3[1]), dot(m3[2], m3[2]))));
  /*VWORLD*/
  vWorldPos = wp.xyz;
  vNormalW = nw;
  vec4 vp = viewMatrix * wp;
  vNormalV = mat3(viewMatrix) * nw;
  vViewZ = -vp.z;
  vColor = col;
  vec4 sc = uShadowMatrix * vec4(wp.xyz + nw * uShadowParams.w, 1.0);
  vShadowCoord = sc.xyz;
  gl_Position = projectionMatrix * vp;
}
`,m1=`
layout(location = 0) out highp vec4 outColor;
layout(location = 1) out highp vec4 outGeo;
${Eo}
uniform highp sampler2DShadow uShadowMap;
uniform vec4 uShadowParams;
uniform float uHatch;
varying vec3 vWorldPos;
varying vec3 vNormalW;
varying vec3 vNormalV;
varying vec4 vColor;
varying float vViewZ;
varying vec3 vShadowCoord;
varying vec3 vObj;
varying vec3 vObjN;
varying vec4 vSeed;

float shadowAt() {
  vec3 sc = vShadowCoord;
  vec2 e = abs(sc.xy - 0.5) * 2.0;
  float edge = max(e.x, e.y);
  if (edge >= 1.0 || sc.z >= 1.0 || sc.z <= 0.0) return 1.0;
  float t = uShadowParams.x * 0.8;
  float z = sc.z - uShadowParams.y;
  float s = texture(uShadowMap, vec3(sc.xy + vec2(-t, -t), z))
          + texture(uShadowMap, vec3(sc.xy + vec2( t, -t), z))
          + texture(uShadowMap, vec3(sc.xy + vec2(-t,  t), z))
          + texture(uShadowMap, vec3(sc.xy + vec2( t,  t), z));
  s = smoothstep(0.3, 0.7, s * 0.25);
  return mix(mix(1.0, s, uShadowParams.z), 1.0, smoothstep(0.82, 1.0, edge));
}

// hand-inked hatching strokes, anchored to the surface
float hatch(vec3 p, vec3 n, float scale) {
  vec2 uv;
  if (abs(n.y) > 0.6) uv = p.xz;
  else uv = vec2(dot(p.xz, vec2(-n.z, n.x)), p.y);
  uv *= scale;
  vec2 q = vec2(uv.x + uv.y, uv.x - uv.y);
  float a = texture(uNoise, vec2(q.x * 0.23, q.y * 0.012)).b;
  float b = texture(uNoise, vec2(q.x * 0.011 + 0.37, q.y * 0.19)).g;
  return smoothstep(0.58, 0.66, a) * 0.8 + smoothstep(0.7, 0.76, b) * 0.0;
}
/*FPARS*/
void main() {
  vec3 N = normalize(vNormalW);
  vec3 NV = normalize(vNormalV);
  #ifdef DOUBLE_SIDED
    if (!gl_FrontFacing) { N = -N; NV = -NV; }
  #endif
  vec3 albedo = vColor.rgb;
  float flag = vColor.a;      // free per-vertex channel, meaning depends on the material
  float glow = 0.0;           // 0 lit .. 1 self-lit
  float ink = 1.0;            // 0 hides this surface from the outline pass
  float shadeMul = 1.0;
  float sh = 1.0;
  vec3 V = normalize(cameraPosition - vWorldPos);
  /*FALBEDO*/
  #ifndef NO_SHADOW
    sh = shadowAt();
  #endif
  float ndl = dot(N, uSunDir);
  float l1 = smoothstep(0.0, 0.05, ndl);
  float l2 = smoothstep(0.42, 0.48, ndl);
  float lit = min(l1, sh);
  float tone = lit * mix(0.58, 1.0, l2);
  vec3 shade = uShadeTint + uSkyAmb * (0.5 + 0.5 * N.y);
  // deep shade band for faces turned well away from the light
  shade *= mix(0.9, 1.0, smoothstep(-0.55, -0.45, ndl));
  vec3 light = mix(shade, uSunTint, tone);
  vec3 col = albedo * light * shadeMul;
  #ifndef NO_HATCH
    float hd = (1.0 - smoothstep(120.0, 520.0, vViewZ));
    if (hd > 0.0 && uHatch > 0.0) {
      float h = hatch(vWorldPos, N, 1.0);
      col *= 1.0 - h * hd * uHatch * (1.0 - tone) * 0.34;
    }
  #endif
  /*FLIT*/
  col = mix(col, albedo, glow);
  float dist = length(vWorldPos - cameraPosition);
  float fog = fogAmt(dist, vWorldPos.y);
  /*FFOG*/
  col = mix(col, atmo(-V), fog);
  outColor = vec4(col, 1.0);
  outGeo = vec4(NV * ink, vViewZ);
}
`,g1=`
uniform float uTime;
uniform vec2 uWind;
float hash12(vec2 p) {
  vec3 p3 = fract(vec3(p.xyx) * .1031);
  p3 += dot(p3, p3.yzx + 33.33);
  return fract((p3.x + p3.y) * p3.z);
}
/*VPARS*/
void main() {
  vec3 pos = position;
  vec3 nrm = normal;
  vec4 col = vec4(1.0);
  #ifdef USE_COLOR_ALPHA
    col = color;
  #endif
  mat4 m = modelMatrix;
  vec3 iscale = vec3(1.0);
  vec4 vSeed = vec4(0.0);
  vec3 icol = vec3(1.0);
  #ifdef USE_INSTANCING
    m = modelMatrix * instanceMatrix;
    iscale = vec3(length(instanceMatrix[0].xyz), length(instanceMatrix[1].xyz), length(instanceMatrix[2].xyz));
    vec2 ip = instanceMatrix[3].xz;
    vSeed = vec4(hash12(ip), hash12(ip + 17.7), hash12(ip * 1.3 + 4.1), hash12(ip * 0.7 - 9.3));
  #endif
  /*VPRE*/
  vec4 wp = m * vec4(pos, 1.0);
  vec3 nw = vec3(0.0, 1.0, 0.0);
  /*VWORLD*/
  gl_Position = projectionMatrix * viewMatrix * wp;
}
`,v1=`
layout(location = 0) out highp vec4 outColor;
void main() { outColor = vec4(1.0); }
`;
function rl(e,t){
return e.replace("/*VPARS*/",t.vPars||"").replace("/*VPRE*/",t.vPre||"").replace("/*VWORLD*/",t.vWorld||"").replace("/*FPARS*/",t.fPars||"").replace("/*FALBEDO*/",t.fAlbedo||"").replace("/*FLIT*/",t.fLit||"").replace("/*FFOG*/",t.fFog||"")}
var x1=["uTime","uSunDir","uSunPos","uMoonPos","uSunTint","uShadeTint","uSkyAmb","uFogCol","uFogSun","uFogParams","uNight","uNoise","uShadowMap","uShadowMatrix","uShadowParams","uHatch","uWind"];
function wl(e={
}
){
let t={
}
;
for(let i of x1)t[i]=et[i];
return Object.assign(t,e)}
var Zh=null;
function Vu(e={
}
){
let t=new $i({
glslVersion:Po,uniforms:{
uTime:et.uTime,uWind:et.uWind,...e.uniforms||{
}
}
,vertexShader:rl(g1,e),fragmentShader:v1,side:2,vertexColors:!!e.vertexColors,defines:e.defines||{
}
}
);
return t.colorWrite=!1,t.polygonOffset=!0,t.polygonOffsetFactor=1.5,t.polygonOffsetUnits=3,t}
function _1(){
return Zh||=Vu(),Zh}
function bi(e={
}
){
let t={
...e.defines||{
}
}
;
e.shadow===!1&&(t.NO_SHADOW=1),e.hatch===!1&&(t.NO_HATCH=1);
let i=new $i({
glslVersion:Po,uniforms:wl(e.uniforms||{
}
),vertexShader:rl(p1,e),fragmentShader:rl(m1,e),vertexColors:e.vertexColors!==!1,side:e.side??0,defines:t}
);
return e.customDepth&&(i.userData.depth=Vu({
vPars:e.vPars,vPre:e.vPre,vWorld:e.vWorld,uniforms:e.uniforms,vertexColors:e.vertexColors!==!1,defines:e.defines}
)),i}
function Ao(e={
}
){
return new $i({
glslVersion:Po,uniforms:wl(e.uniforms||{
}
),vertexShader:`
      ${Eo}
      varying vec4 vColor; varying vec2 vUv; varying vec3 vWorldPos;
      ${e.vPars||""}
      void main() {
        vec3 pos = position; vec4 col = vec4(1.0);
        #ifdef USE_COLOR_ALPHA
          col = color;
        #elif defined(USE_COLOR)
          col.rgb = color;
        #endif
        vUv = uv;
        mat4 m = modelMatrix;
        #ifdef USE_INSTANCING
          m = modelMatrix * instanceMatrix;
        #endif
        #ifdef USE_INSTANCING_COLOR
          col.rgb *= instanceColor;
        #endif
        ${e.vPre||""}
        vec4 wp = m * vec4(pos, 1.0);
        #ifdef BILLBOARD
          vec3 c = (m * vec4(0.0, 0.0, 0.0, 1.0)).xyz;
          float s = length(m[0].xyz);
          vec3 r = vec3(viewMatrix[0][0], viewMatrix[1][0], viewMatrix[2][0]);
          vec3 u = vec3(viewMatrix[0][1], viewMatrix[1][1], viewMatrix[2][1]);
          wp = vec4(c + (r * pos.x + u * pos.y) * s, 1.0);
        #endif
        ${e.vWorld||""}
        vWorldPos = wp.xyz;
        vColor = col;
        gl_Position = projectionMatrix * viewMatrix * wp;
      }`,fragmentShader:`
      layout(location = 0) out highp vec4 outColor;
      layout(location = 1) out highp vec4 outGeo;
      ${Eo}
      varying vec4 vColor; varying vec2 vUv; varying vec3 vWorldPos;
      ${e.fPars||""}
      void main() {
        vec3 col = vColor.rgb; float a = vColor.a;
        ${e.fMain||""}
        #ifndef NO_FOG
          float fog = fogAmt(length(vWorldPos - cameraPosition), vWorldPos.y);
          a *= 1.0 - fog;
        #endif
        #ifdef ADDITIVE
          outColor = vec4(col * a, 0.0);
        #else
          outColor = vec4(col * a, a);
        #endif
        outGeo = vec4(0.0);
      }`,vertexColors:!!e.vertexColors,side:e.side??2,defines:e.defines||{
}
,transparent:!0,depthWrite:!1,depthTest:e.depthTest!==!1,blending:5,blendEquation:100,blendSrc:201,blendDst:205,blendSrcAlpha:201,blendDstAlpha:205}
)}
var y1=`
in vec3 position;
out vec2 vUv;
void main() { vUv = position.xy * 0.5 + 0.5; gl_Position = vec4(position.xy, 0.0, 1.0); }
`,b1=`
precision highp float;
uniform sampler2D tColor;
uniform sampler2D tGeo;
uniform sampler2D uNoise;
uniform vec2 uRes;
uniform float uScale;
uniform vec3 uInk;
uniform float uInkAmount;
uniform vec4 uFogParams;
in vec2 vUv;
out vec4 o;

void main() {
  vec2 px = 1.0 / uRes;
  // a slow wobble so the line is never perfectly mechanical
  vec2 nuv = vUv * uRes / (uScale * 256.0);
  vec4 nz = texture(uNoise, nuv * 0.13);
  vec4 nz2 = texture(uNoise, nuv * 0.71 + 0.31);
  // Taps sit on exact pixel centres, a whole number of pixels either side of the middle one.
  // Lopsided taps leak the slope of 1/z into the "second derivative", which inks the sea near the horizon.
  vec2 uv = (floor((vUv + (nz.rg - 0.5) * 2.4 * uScale * px) * uRes) + 0.5) * px;

  vec4 g0 = texture(tGeo, uv);
  float d0 = g0.a;
  float i0 = 1.0 / d0;
  float k0 = step(0.5, dot(g0.xyz, g0.xyz));

  float w = mix(0.75, 1.3, nz2.g);           // line weight breathes a little
  vec2 ra = px * max(1.0, floor(uScale * 1.0 * w + 0.5));
  vec2 rb = px * max(1.0, floor(uScale * 2.3 * w * 0.7071 + 0.5));

  vec4 gl = texture(tGeo, uv + vec2(-ra.x, 0.0));
  vec4 gr = texture(tGeo, uv + vec2( ra.x, 0.0));
  vec4 gu = texture(tGeo, uv + vec2(0.0,  ra.y));
  vec4 gd = texture(tGeo, uv + vec2(0.0, -ra.y));

  vec4 b1 = texture(tGeo, uv + vec2(-rb.x, -rb.y));
  vec4 b2 = texture(tGeo, uv + vec2( rb.x,  rb.y));
  vec4 b3 = texture(tGeo, uv + vec2(-rb.x,  rb.y));
  vec4 b4 = texture(tGeo, uv + vec2( rb.x, -rb.y));

  float kA = k0 * step(0.5, dot(gl.xyz, gl.xyz)) * step(0.5, dot(gr.xyz, gr.xyz)) * step(0.5, dot(gu.xyz, gu.xyz)) * step(0.5, dot(gd.xyz, gd.xyz));
  float kB = k0 * step(0.5, dot(b1.xyz, b1.xyz)) * step(0.5, dot(b2.xyz, b2.xyz)) * step(0.5, dot(b3.xyz, b3.xyz)) * step(0.5, dot(b4.xyz, b4.xyz));

  // creases: where neighbouring normals disagree
  float nd = max(max(1.0 - dot(g0.xyz, gl.xyz), 1.0 - dot(g0.xyz, gr.xyz)), max(1.0 - dot(g0.xyz, gu.xyz), 1.0 - dot(g0.xyz, gd.xyz)));
  float eN = smoothstep(0.16, 0.42, nd) * kA;

  // silhouettes: second derivative of 1/z is zero on any flat surface, so only real breaks show
  float lapA = (abs(1.0 / gl.a + 1.0 / gr.a - 2.0 * i0) + abs(1.0 / gu.a + 1.0 / gd.a - 2.0 * i0)) / i0;
  float lapB = (abs(1.0 / b1.a + 1.0 / b2.a - 2.0 * i0) + abs(1.0 / b3.a + 1.0 / b4.a - 2.0 * i0)) / i0;
  float eA = smoothstep(0.012, 0.03, lapA) * kA;
  float eB = smoothstep(0.03, 0.07, lapB) * kB;

  float dMinA = min(d0, min(min(gl.a, gr.a), min(gu.a, gd.a)));
  float dMinB = min(dMinA, min(min(b1.a, b2.a), min(b3.a, b4.a)));

  float isSky = step(40000.0, dMinB);
  float fogA = 1.0 - exp(-pow(dMinA * uFogParams.x, 1.3));
  float fogB = 1.0 - exp(-pow(dMinB * uFogParams.x, 1.3));

  // line weight falls away with distance: thick silhouettes only near the eye
  float crease = eN * (1.0 - smoothstep(180.0, 1500.0, d0)) * mix(0.82, 0.4, smoothstep(60.0, 700.0, d0));
  float thin = eA * mix(1.0, 0.5, smoothstep(300.0, 2600.0, dMinA)) * (1.0 - fogA * 0.9);
  float thick = eB * (1.0 - smoothstep(140.0, 1100.0, dMinB)) * (1.0 - fogB);
  float edge = max(max(crease, thin), thick);
  edge = mix(edge, max(eA, eB) * 0.3, isSky);
  // dry-brush breakup
  edge *= mix(0.72, 1.0, smoothstep(0.2, 0.5, nz2.b));

  vec3 col = texture(tColor, vUv).rgb;
  vec3 inkCol = mix(uInk, col * uInk * 1.6, 0.35);
  col = mix(col, inkCol, clamp(edge * uInkAmount, 0.0, 1.0));
  o = vec4(col, 1.0);
}
`,M1=`
precision highp float;
uniform sampler2D tColor;
uniform vec2 uPx;
uniform float uThreshold;
in vec2 vUv;
out vec4 o;
void main() {
  vec3 c = texture(tColor, vUv + uPx * vec2(-1.0, -1.0)).rgb + texture(tColor, vUv + uPx * vec2(1.0, -1.0)).rgb
         + texture(tColor, vUv + uPx * vec2(-1.0, 1.0)).rgb + texture(tColor, vUv + uPx * vec2(1.0, 1.0)).rgb;
  c *= 0.25;
  float l = max(c.r, max(c.g, c.b));
  o = vec4(c * smoothstep(uThreshold, uThreshold + 0.18, l), 1.0);
}
`,S1=`
precision highp float;
uniform sampler2D tColor;
uniform vec2 uDir;
in vec2 vUv;
out vec4 o;
void main() {
  vec3 c = texture(tColor, vUv).rgb * 0.227027;
  c += (texture(tColor, vUv + uDir * 1.3846).rgb + texture(tColor, vUv - uDir * 1.3846).rgb) * 0.316216;
  c += (texture(tColor, vUv + uDir * 3.2308).rgb + texture(tColor, vUv - uDir * 3.2308).rgb) * 0.070270;
  o = vec4(c, 1.0);
}
`,w1=`
precision highp float;
uniform sampler2D tColor;
uniform sampler2D tBloom;
uniform sampler2D uNoise;
uniform vec2 uRes;
uniform float uTime;
uniform float uBloom;
uniform float uVignette;
uniform float uGrain;
uniform float uSat;
uniform float uFade;       // 0 normal .. 1 paper white
uniform vec3 uPaper;
in vec2 vUv;
out vec4 o;

vec3 fxaa(vec2 uv, vec2 px) {
  vec3 nw = texture(tColor, uv + vec2(-1.0, -1.0) * px).rgb;
  vec3 ne = texture(tColor, uv + vec2( 1.0, -1.0) * px).rgb;
  vec3 sw = texture(tColor, uv + vec2(-1.0,  1.0) * px).rgb;
  vec3 se = texture(tColor, uv + vec2( 1.0,  1.0) * px).rgb;
  vec3 m  = texture(tColor, uv).rgb;
  vec3 L = vec3(0.299, 0.587, 0.114);
  float lnw = dot(nw, L), lne = dot(ne, L), lsw = dot(sw, L), lse = dot(se, L), lm = dot(m, L);
  float lmin = min(lm, min(min(lnw, lne), min(lsw, lse)));
  float lmax = max(lm, max(max(lnw, lne), max(lsw, lse)));
  vec2 dir = vec2(-((lnw + lne) - (lsw + lse)), ((lnw + lsw) - (lne + lse)));
  float red = max((lnw + lne + lsw + lse) * 0.03125, 0.0078125);
  float rcp = 1.0 / (min(abs(dir.x), abs(dir.y)) + red);
  dir = clamp(dir * rcp, -8.0, 8.0) * px;
  vec3 a = 0.5 * (texture(tColor, uv + dir * (1.0 / 3.0 - 0.5)).rgb + texture(tColor, uv + dir * (2.0 / 3.0 - 0.5)).rgb);
  vec3 b = a * 0.5 + 0.25 * (texture(tColor, uv - dir * 0.5).rgb + texture(tColor, uv + dir * 0.5).rgb);
  float lb = dot(b, L);
  return (lb < lmin || lb > lmax) ? a : b;
}

void main() {
  vec2 px = 1.0 / uRes;
  vec3 col = fxaa(vUv, px);
  vec3 bloom = texture(tBloom, vUv).rgb;
  col += bloom * uBloom;
  // gentle grade
  float l = dot(col, vec3(0.299, 0.587, 0.114));
  col = mix(vec3(l), col, uSat);
  // paper tooth + grain
  vec2 guv = vUv * uRes / 256.0;
  float tooth = texture(uNoise, guv * 0.9).b - 0.5;
  float grain = texture(uNoise, guv * 1.7 + fract(uTime * vec2(7.13, 3.71))).a - 0.5;
  col += (tooth * 0.028 + grain * 0.02) * uGrain;
  // vignette, warm and soft
  vec2 q = vUv - 0.5;
  float v = smoothstep(0.35, 0.95, length(q * vec2(1.0, 0.9)) * 1.25);
  col *= 1.0 - v * uVignette;
  col = mix(col, uPaper, uFade);
  o = vec4(col, 1.0);
}
`;
function cs(e,t){
return new Tu({
glslVersion:Po,vertexShader:y1,fragmentShader:e,uniforms:t,depthTest:!1,depthWrite:!1}
)}
var T1=class{
constructor(e,t={
}
){
this.renderer=e,this.scale=1,this.w=2,this.h=2,this.gbuf=new gi(2,2,{
count:2,depthBuffer:!0,type:Fr,minFilter:jt,magFilter:jt,generateMipmaps:!1}
);
let i=this.gbuf.textures[1];
i.type=Rr,i.minFilter=oi,i.magFilter=oi;
let r={
depthBuffer:!1,minFilter:jt,magFilter:jt,generateMipmaps:!1}
;
this.inkRT=new gi(2,2,r),this.bloomA=new gi(2,2,r),this.bloomB=new gi(2,2,r),this.shadowRes=t.shadowRes||2048,this.shadowSize=520;
let a=new ra(this.shadowRes,this.shadowRes);
a.type=Vr,a.compareFunction=515,a.minFilter=jt,a.magFilter=jt,this.shadowRT=new gi(this.shadowRes,this.shadowRes,{
depthBuffer:!0,depthTexture:a}
),this.shadowCam=new el(-1,1,1,-1,10,5e3),this.shadowCam.layers.set(1),this.shadowOn=!0,et.uShadowMap.value=a,et.uShadowParams.value.x=1/this.shadowRes,this.casters=[],this.inkU={
tColor:{
value:this.gbuf.textures[0]}
,tGeo:{
value:this.gbuf.textures[1]}
,uNoise:et.uNoise,uRes:{
value:new ge(2,2)}
,uScale:{
value:1}
,uInk:{
value:new I(.09,.07,.12)}
,uInkAmount:{
value:1}
,uFogParams:et.uFogParams}
,this.brightU={
tColor:{
value:this.inkRT.texture}
,uPx:{
value:new ge}
,uThreshold:{
value:.82}
}
,this.blurU={
tColor:{
value:null}
,uDir:{
value:new ge}
}
,this.finalU={
tColor:{
value:this.inkRT.texture}
,tBloom:{
value:this.bloomA.texture}
,uNoise:et.uNoise,uRes:{
value:new ge(2,2)}
,uTime:et.uTime,uBloom:{
value:.35}
,uVignette:{
value:.3}
,uGrain:{
value:1}
,uSat:{
value:1.06}
,uFade:{
value:0}
,uPaper:{
value:new I(.96,.95,.93)}
}
,this.inkMat=cs(b1,this.inkU),this.brightMat=cs(M1,this.brightU),this.blurMat=cs(S1,this.blurU),this.finalMat=cs(w1,this.finalU);
let o=new ti;
o.setAttribute("position",new si(new Float32Array([-1,-1,0,3,-1,0,-1,3,0]),3)),this.quad=new Nt(o,this.inkMat),this.quad.frustumCulled=!1,this.fsScene=new Wc,this.fsScene.add(this.quad),this.fsCam=new el(-1,1,1,-1,0,1),this.bloomOn=!0,this._v=new I,this._r=new I,this._u=new I,this._bias=new gt().set(.5,0,0,.5,0,.5,0,.5,0,0,.5,.5,0,0,0,1)}
cast(e){
return e.layers.enable(1),this.casters.push(e),e}
setSize(e,t,i,r=this.scale){
this.scale=r;
let a=Math.max(2,Math.round(e*i*r)),o=Math.max(2,Math.round(t*i*r));
if(a===this.w&&o===this.h)return;
this.w=a,this.h=o,this.gbuf.setSize(a,o),this.inkRT.setSize(a,o);
let s=Math.max(2,a>>2),n=Math.max(2,o>>2);
this.bloomA.setSize(s,n),this.bloomB.setSize(s,n),this.inkU.uRes.value.set(a,o),this.finalU.uRes.value.set(a,o),this.inkU.uScale.value=Math.max(.85,Math.min(a,o*1.6)/1250),this.brightU.uPx.value.set(1/a,1/o)}
updateShadow(e,t){
let i=this.shadowCam,r=this.shadowSize,a=this._v.copy(t);
a.y=Math.max(a.y,.24),a.normalize(),i.left=-r,i.right=r,i.top=r,i.bottom=-r,i.near=10,i.far=4200,i.updateProjectionMatrix(),i.position.set(0,0,0),i.up.set(0,1,0),i.lookAt(-a.x,-a.y,-a.z),i.updateMatrixWorld(!0);
let o=i.matrixWorld.elements,s=this._r.set(o[0],o[1],o[2]),n=this._u.set(o[4],o[5],o[6]),l=2*r/this.shadowRes,h=e.dot(s),c=e.dot(n),u=Math.round(h/l)*l-h,d=Math.round(c/l)*l-c;
i.position.copy(e).addScaledVector(s,u).addScaledVector(n,d).addScaledVector(a,2200),i.updateMatrixWorld(!0),i.matrixWorldInverse.copy(i.matrixWorld).invert(),et.uShadowMatrix.value.copy(this._bias).multiply(i.projectionMatrix).multiply(i.matrixWorldInverse)}
render(e,t){
let i=this.renderer;
if(this.shadowOn){
let a=_1(),o=this.casters;
for(let s=0;
s<o.length;
s++){
let n=o[s];
n.userData._mat=n.material,n.material=n.material.userData.depth||a}
i.setRenderTarget(this.shadowRT),i.clear(),i.render(e,this.shadowCam);
for(let s=0;
s<o.length;
s++)o[s].material=o[s].userData._mat}
if(i.setRenderTarget(this.gbuf),i.render(e,t),this.quad.material=this.inkMat,i.setRenderTarget(this.inkRT),i.render(this.fsScene,this.fsCam),this.bloomOn){
this.quad.material=this.brightMat,i.setRenderTarget(this.bloomA),i.render(this.fsScene,this.fsCam),this.quad.material=this.blurMat;
for(let a=0;
a<2;
a++){
let o=a===0?1:2;
this.blurU.tColor.value=this.bloomA.texture,this.blurU.uDir.value.set(o/this.bloomA.width,0),i.setRenderTarget(this.bloomB),i.render(this.fsScene,this.fsCam),this.blurU.tColor.value=this.bloomB.texture,this.blurU.uDir.value.set(0,o/this.bloomA.height),i.setRenderTarget(this.bloomA),i.render(this.fsScene,this.fsCam)}
}
this.quad.material=this.finalMat;
let r=this.finalU.uBloom.value;
this.bloomOn||(this.finalU.uBloom.value=0),i.setRenderTarget(null),i.render(this.fsScene,this.fsCam),this.finalU.uBloom.value=r}
};
var La={
uDeep:{
value:new I(.29,.52,.65)}
,uShallow:{
value:new I(.5,.76,.7)}
,uFoamCol:{
value:new I(1,.98,.94)}
,uGlintCol:{
value:new I(1,.95,.8)}
,uGlint:{
value:1}
}
;
function W1(rect,signedDistance){
let e=2048,t=Math.round(e*(rect.z1-rect.z0)/(rect.x1-rect.x0)),i=new Uint8Array(e*t);
for(let a=0;
a<t;
a++){
let o=rect.z0+(a+.5)/t*(rect.z1-rect.z0);
for(let s=0;
s<e;
s++){
let n=signedDistance(rect.x0+(s+.5)/e*(rect.x1-rect.x0),o);
i[a*e+s]=Math.max(0,Math.min(255,(.5+n/400)*255))}
}
let r=new No(i,e,t,Ys);
return r.minFilter=r.magFilter=jt,r.wrapS=r.wrapT=Zi,r.needsUpdate=!0,r}
function X1(rect,signedDistance){
let e=new Br(6e4,6e4,1,1);
e.rotateX(-Math.PI/2);
let t=new Nt(e,bi({
vertexColors:!1,hatch:!1,uniforms:{
uShore:{
value:W1(rect,signedDistance)}
,uMapRect:{
value:new Ft(rect.x0,rect.z0,rect.x1-rect.x0,rect.z1-rect.z0)}
,uZenith:et.uZenith,uMidSky:et.uMidSky,...La}
,fPars:`
      uniform sampler2D uShore; uniform vec4 uMapRect;
      uniform vec3 uDeep, uShallow, uFoamCol, uGlintCol, uZenith, uMidSky;
      uniform float uGlint;
      vec3 wc; float foam; float glint;
      // little hand-drawn wave arcs that fade in and out, each on its own clock
      float waveMarks(vec2 p, vec2 cell, float t) {
        vec2 g = p / cell;
        g.x += hash12(vec2(floor(g.y), 3.7));
        vec2 id = floor(g);
        vec2 f = fract(g) - 0.5;
        float h = hash12(id), h2 = hash12(id + 19.19);
        f += (vec2(h, h2) - 0.5) * vec2(0.3, 0.4);
        float len = mix(0.2, 0.42, h2);
        float curve = f.y + f.x * f.x * 1.7 - 0.07;
        float thick = 0.05 * (1.0 - smoothstep(len * 0.4, len, abs(f.x)));
        float m = smoothstep(thick, thick * 0.35, abs(curve)) * step(abs(f.x), len);
        float life = sin(t * (0.35 + h * 0.5) + h * 6.283);
        return m * smoothstep(0.0, 0.5, life) * step(0.5, h2);
      }
      float sparkle(vec2 p, float size, float t, float density) {
        vec2 g = p / size;
        vec2 id = floor(g), f = fract(g) - 0.5;
        float h = hash12(id), h2 = hash12(id + 31.7);
        float tw = fract(h * 13.0 + t * (0.3 + h * 0.5));
        float on = step(1.0 - density, h2) * smoothstep(0.0, 0.12, tw) * (1.0 - smoothstep(0.3, 0.55, tw));
        f -= (vec2(hash12(id + 5.3), hash12(id + 9.1)) - 0.5) * 0.5;
        float d = length(f);
        return on * (1.0 - smoothstep(0.13, 0.2, d));
      }
    `,fAlbedo:`
      N = vec3(0.0, 1.0, 0.0);
      vec2 wuv = vWorldPos.xz;
      vec2 muv = (wuv - uMapRect.xy) / uMapRect.zw;
      vec2 me = abs(muv - 0.5) * 2.0;
      float sdv = mix(0.0, texture(uShore, muv).r, step(max(me.x, me.y), 1.0));
      float off = (0.5 - sdv) * 400.0;          // metres out from the shore
      float t = uTime;

      wc = mix(uShallow, uDeep, smoothstep(0.0, 70.0, off));
      // big slow patches of slightly different blue, cut like paper
      float big = texture(uNoise, wuv * 0.00055 + t * vec2(0.0022, 0.0009)).r;
      float mid = texture(uNoise, wuv * 0.0021 - t * vec2(0.004, -0.002)).g;
      wc *= mix(0.95, 1.04, smoothstep(0.46, 0.5, big * 0.65 + mid * 0.35));

      // the far water borrows the sky
      float fres = pow(1.0 - max(V.y, 0.0), 5.0);
      vec3 R = reflect(-V, N);
      vec3 skyRef = mix(atmo(R), uMidSky, smoothstep(0.0, 0.5, R.y));
      wc = mix(wc, skyRef, fres * 0.72);

      // wave marks drift with the wind
      vec2 wp2 = wuv + t * vec2(1.6, 0.5);
      float marks = waveMarks(wp2, vec2(34.0, 15.0), t) * (1.0 - smoothstep(250.0, 900.0, vViewZ))
                  + waveMarks(wp2 * 1.0 + 71.0, vec2(90.0, 38.0), t * 0.7) * (1.0 - smoothstep(700.0, 2600.0, vViewZ)) * smoothstep(120.0, 400.0, vViewZ);

      // glitter path: sparkles gather where the water reflects the sun
      // by night the moon lays down its own, quieter path
      float moonLit = smoothstep(0.55, 0.95, uNight);
      vec3 lamp = mix(uSunPos, uMoonPos, moonLit);
      vec2 rxz = normalize(R.xz + 1e-5), sxz = normalize(lamp.xz + 1e-5);
      float az = dot(rxz, sxz);
      float sunUp = max(smoothstep(-0.06, 0.05, uSunPos.y), moonLit * 0.75);
      float grazing = smoothstep(0.02, 0.4, 1.0 - V.y);
      float pathW = smoothstep(0.93, 0.9995, az) * sunUp * grazing;
      float dens = pathW * 0.75 + 0.02 * sunUp;
      float nearS = sparkle(wuv + t * vec2(0.8, 0.3), 4.0, t, dens) * (1.0 - smoothstep(250.0, 650.0, vViewZ));
      float midS = sparkle(wuv + t * vec2(0.8, 0.3), 13.0, t, dens) * smoothstep(150.0, 500.0, vViewZ) * (1.0 - smoothstep(1500.0, 3200.0, vViewZ));
      float farS = smoothstep(0.52, 0.56, texture(uNoise, wuv * 0.0035 + t * vec2(0.01, 0.004)).b) * pathW * smoothstep(1200.0, 3000.0, vViewZ) * 0.8;
      glint = max(max(nearS, midS), farS) * uGlint;

      // foam hugging the shore, plus slow rings travelling in
      float fn = texture(uNoise, wuv * 0.017 + t * 0.008).r;
      float fn2 = texture(uNoise, wuv * 0.06 - t * 0.012).g;
      foam = 1.0 - smoothstep(0.0, 1.2, off - 2.0 - fn * 7.0);
      float ring = sin(off * 0.36 - t * 0.9 + fn * 7.0);
      foam = max(foam, smoothstep(0.88, 0.93, ring) * (1.0 - smoothstep(6.0, 30.0, off)) * step(0.42, fn2));
      foam = max(foam, marks * 0.6);
      albedo = wc;
    `,fLit:`
      vec3 lw = mix(uShadeTint * 1.2 + uSkyAmb, vec3(1.0), sh);
      col = wc * lw;
      col = mix(col, uFoamCol * mix(lw, vec3(1.0), 0.5), foam);
      col = mix(col, uGlintCol, glint * sh);
    `}
));
return t.frustumCulled=!1,t.name="water",t}
function Yu(e,t=!1){
let i=e[0].index!==null,r=new Set(Object.keys(e[0].attributes)),a=new Set(Object.keys(e[0].morphAttributes)),o={
}
,s={
}
,n=e[0].morphTargetsRelative,l=new ti;
l.morphTargetsRelative=n;
let h=0;
for(let c=0;
c<e.length;
++c){
let u=e[c],d=0,f=0;
if(i!==(u.index!==null))return console.error("THREE.BufferGeometryUtils: .mergeGeometries() failed with geometry at index "+c+". All geometries must have compatible attributes; make sure index attribute exists among all geometries, or in none of them."),null;
for(let p in u.attributes){
if(!r.has(p))return console.error("THREE.BufferGeometryUtils: .mergeGeometries() failed with geometry at index "+c+'. All geometries must have compatible attributes; make sure "'+p+'" attribute exists among all geometries, or in none of them.'),null;
o[p]===void 0&&(o[p]=[]),o[p].push(u.attributes[p]),d++}
if(d!==r.size)return console.error("THREE.BufferGeometryUtils: .mergeGeometries() failed with geometry at index "+c+". Make sure all geometries have the same number of attributes."),null;
if(n!==u.morphTargetsRelative)return console.error("THREE.BufferGeometryUtils: .mergeGeometries() failed with geometry at index "+c+". .morphTargetsRelative must be consistent throughout all geometries."),null;
for(let p in u.morphAttributes){
if(!a.has(p))return console.error("THREE.BufferGeometryUtils: .mergeGeometries() failed with geometry at index "+c+".  .morphAttributes must be consistent throughout all geometries."),null;
if(u.morphAttributes[p].length!==e[0].morphAttributes[p].length)return console.error("THREE.BufferGeometryUtils: .mergeGeometries() failed with geometry at index "+c+'. All geometries must have the same number of morph targets for the "'+p+'" morph attribute.'),null;
s[p]===void 0&&(s[p]=[]),s[p].push(u.morphAttributes[p]),f++}
if(f!==a.size)return console.error("THREE.BufferGeometryUtils: .mergeGeometries() failed with geometry at index "+c+". Make sure all geometries have the same number of morph attributes."),null;
if(t){
let p;
if(i)p=u.index.count;
else if(u.attributes.position!==void 0)p=u.attributes.position.count;
else return console.error("THREE.BufferGeometryUtils: .mergeGeometries() failed with geometry at index "+c+". The geometry must have either an index or a position attribute"),null;
l.addGroup(h,p,c),h+=p}
}
if(i){
let c=0,u=[];
for(let d=0;
d<e.length;
++d){
let f=e[d].index;
for(let p=0;
p<f.count;
++p)u.push(f.getX(p)+c);
c+=e[d].attributes.position.count}
l.setIndex(u)}
for(let c in o){
let u=uc(o[c]);
if(!u)return console.error("THREE.BufferGeometryUtils: .mergeGeometries() failed while trying to merge the "+c+" attribute."),null;
l.setAttribute(c,u)}
for(let c in s){
let u=s[c][0].length;
if(u!==0){
l.morphAttributes=l.morphAttributes||{
}
,l.morphAttributes[c]=[];
for(let d=0;
d<u;
++d){
let f=[];
for(let x=0;
x<s[c].length;
++x)f.push(s[c][x][d]);
let p=uc(f);
if(!p)return console.error("THREE.BufferGeometryUtils: .mergeGeometries() failed while trying to merge the "+c+" morphAttribute."),null;
l.morphAttributes[c].push(p)}
}
}
return l}
function uc(e){
let t,i,r,a=-1,o=0;
for(let h=0;
h<e.length;
++h){
let c=e[h];
if(t===void 0&&(t=c.array.constructor),t!==c.array.constructor)return console.error("THREE.BufferGeometryUtils: .mergeAttributes() failed. BufferAttribute.array must be of consistent array types across matching attributes."),null;
if(i===void 0&&(i=c.itemSize),i!==c.itemSize)return console.error("THREE.BufferGeometryUtils: .mergeAttributes() failed. BufferAttribute.itemSize must be consistent across matching attributes."),null;
if(r===void 0&&(r=c.normalized),r!==c.normalized)return console.error("THREE.BufferGeometryUtils: .mergeAttributes() failed. BufferAttribute.normalized must be consistent across matching attributes."),null;
if(a===-1&&(a=c.gpuType),a!==c.gpuType)return console.error("THREE.BufferGeometryUtils: .mergeAttributes() failed. BufferAttribute.gpuType must be consistent across matching attributes."),null;
o+=c.count*i}
let s=new t(o),n=new si(s,i,r),l=0;
for(let h=0;
h<e.length;
++h){
let c=e[h];
if(c.isInterleavedBufferAttribute){
let u=l/i;
for(let d=0,f=c.count;
d<f;
d++)for(let p=0;
p<i;
p++){
let x=c.getComponent(d,p);
n.setComponent(d+u,p,x)}
}
else s.set(c.array,l);
l+=c.count*i}
return a!==void 0&&(n.gpuType=a),n}
function q1(){
let e=new Nt(new yu(9e3,32,16),new $i({
glslVersion:Po,uniforms:wl({
uZenith:et.uZenith,uMidSky:et.uMidSky,uSunDisc:et.uSunDisc,uStars:et.uStars,uCloudLit:et.uCloudLit,uCloudShade:et.uCloudShade,uCover:{
value:.71}
}
),vertexShader:`
      varying vec3 vDir;
      void main() {
        vDir = position;
        vec4 p = projectionMatrix * viewMatrix * vec4(position + cameraPosition, 1.0);
        gl_Position = p.xyww;
      }`,fragmentShader:`
      layout(location = 0) out highp vec4 outColor;
      layout(location = 1) out highp vec4 outGeo;
      ${Eo}
      uniform vec3 uZenith, uMidSky, uSunDisc, uCloudLit, uCloudShade;
      uniform float uStars, uCover;
      varying vec3 vDir;
      void main() {
        vec3 d = normalize(vDir);
        float h = d.y;
        vec3 hor = atmo(d);
        float t = pow(clamp(h, 0.0, 1.0), 0.5);
        vec3 col = mix(hor, uMidSky, smoothstep(0.0, 0.42, t));
        col = mix(col, uZenith, smoothstep(0.3, 1.0, t));
        col = mix(col, hor, (1.0 - smoothstep(-0.04, 0.0, h)));

        float depth = 60000.0;

        // stars
        if (uStars > 0.01 && h > 0.0) {
          vec2 suv = d.xz / (h + 0.4) * 60.0;
          vec2 cell = floor(suv), f = fract(suv) - 0.5;
          float rnd = hash12(cell);
          vec2 o = (vec2(hash12(cell + 3.1), hash12(cell + 7.7)) - 0.5) * 0.7;
          float sz = mix(0.035, 0.09, hash12(cell + 11.0));
          float star = step(0.9, rnd) * smoothstep(sz, sz * 0.4, length(f - o));
          star *= 0.65 + 0.35 * sin(uTime * 1.7 + rnd * 90.0);
          col += vec3(1.0, 0.96, 0.86) * star * uStars * smoothstep(0.02, 0.2, h);
        }

        // sun: a flat poster disc with two soft rings
        float sd = dot(d, uSunPos);
        float sunVis = smoothstep(-0.12, -0.02, uSunPos.y);
        float low = 1.0 - smoothstep(0.0, 0.45, uSunPos.y);       // bigger near the horizon
        float r = mix(0.022, 0.05, low);
        float ang = acos(clamp(sd, -1.0, 1.0));
        float disc = 1.0 - smoothstep(r - 0.0012, r, ang);
        float ring1 = (1.0 - smoothstep(r * 1.7 - 0.002, r * 1.7, ang)) * 0.22;
        float ring2 = (1.0 - smoothstep(r * 2.9 - 0.003, r * 2.9, ang)) * 0.12;
        float haze = pow(max(sd, 0.0), 14.0) * 0.16;
        col += uSunDisc * (ring1 + ring2 + haze) * sunVis * smoothstep(-0.06, 0.02, h);
        col = mix(col, uSunDisc, disc * sunVis * smoothstep(-0.03, 0.0, h));

        // moon: a crescent
        float md = acos(clamp(dot(d, uMoonPos), -1.0, 1.0));
        float moon = 1.0 - smoothstep(0.034, 0.036, md);
        vec3 mo = normalize(uMoonPos + vec3(0.016, 0.01, -0.012));
        float cut = 1.0 - smoothstep(0.031, 0.033, acos(clamp(dot(d, mo), -1.0, 1.0)));
        float cres = moon * (1.0 - cut);
        col = mix(col, vec3(1.0, 0.97, 0.88), cres * uStars);
        col += vec3(0.5, 0.6, 0.9) * pow(max(dot(d, uMoonPos), 0.0), 60.0) * 0.25 * uStars;

        // painted cloud bank, cut out like paper
        if (h > 0.01) {
          vec2 cuv = d.xz / (h + 0.2);
          vec2 w = uTime * vec2(0.0016, 0.0006);
          // long, low streaks: stretched noise reads as brushed cirrus
          vec2 sc1 = vec2(0.045, 0.13), sc2 = vec2(0.14, 0.37);
          float n = texture(uNoise, cuv * sc1 + w).r * 0.66 + texture(uNoise, cuv * sc2 + w * 1.6).g * 0.34;
          vec2 sdir = normalize(uSunPos.xz + 1e-5) * 0.05;
          float n2 = texture(uNoise, (cuv + sdir) * sc1 + w).r * 0.66 + texture(uNoise, (cuv + sdir) * sc2 + w * 1.6).g * 0.34;
          float fade = smoothstep(0.035, 0.1, h) * (1.0 - smoothstep(0.2, 0.42, h));
          float cov = smoothstep(uCover, uCover + 0.008, n) * fade;
          float litSide = smoothstep(uCover + 0.035, uCover + 0.02, n2);
          vec3 cc = mix(uCloudShade, uCloudLit, litSide);
          cc = mix(cc, hor, 0.3 + 0.4 * (1.0 - smoothstep(0.03, 0.3, h)));
          col = mix(col, cc, step(0.5, cov) * 0.62);
        }
        outColor = vec4(col, 1.0);
        outGeo = vec4(0.0, 0.0, 1.0, depth);
      }`,side:1,depthTest:!1,depthWrite:!1}
));
return e.frustumCulled=!1,e.renderOrder=-1e3,e.name="sky",e}
function j1(e){
let t=[],i=e.int(5,8),r=e.range(1.2,2.2);
for(let s=0;
s<i;
s++){
let n=i===1?0:s/(i-1)-.5,l=(.55+.45*Math.cos(n*Math.PI))*e.range(.45,.72),h=new gu(l,2);
h.scale(1.15,e.range(.72,.95),1),h.translate(n*r*1.6+e.range(-.1,.1),l*.55+e.range(-.05,.12),e.range(-.3,.3)),t.push(h)}
let a=Yu(t),o=a.attributes.position;
for(let s=0;
s<o.count;
s++)o.getY(s)<.06&&o.setY(s,.06+(o.getY(s)-.06)*.18);
return a.computeVertexNormals(),a.deleteAttribute("uv"),a}
function Y1(e=46){
let t=Mi(424),i=new Ji;
i.name="clouds";
let r=bi({
vertexColors:!1,shadow:!1,hatch:!1,uniforms:{
uCloudLit:et.uCloudLit,uCloudShade:et.uCloudShade}
,fPars:"uniform vec3 uCloudLit, uCloudShade;",fLit:`
      float cl = smoothstep(-0.05, 0.05, ndl);
      float under = (1.0 - smoothstep(-0.6, -0.2, N.y));
      col = mix(uCloudShade, uCloudLit, cl * (1.0 - under * 0.6));
      col = mix(col, uCloudShade * 0.93, under * 0.5);
    `,fFog:"fog = max(fog * 0.7, smoothstep(2200.0, 5200.0, dist)); ink *= 1.0 - smoothstep(0.35, 0.9, fog);"}
),a=Math.ceil(e/4),o=[];
for(let u=0;
u<4;
u++){
let d=new Ri(j1(t),r,a);
d.frustumCulled=!1;
let f=new gt,p=new ei,x=new I,g=new I;
for(let m=0;
m<a;
m++){
let y=t.range(70,170),M={
x:t.range(-9800,5200),z:t.range(-5400,3600),y:t.range(440,700)+(t.chance(.2)?180:0),s:y,rot:t.range(0,Xe),sp:t.range(.6,1.4),mesh:d,i:m}
;
o.push(M),p.setFromAxisAngle(new I(0,1,0),M.rot),f.compose(g.set(M.x,M.y,M.z),p,x.set(y,y*.8,y)),d.setMatrixAt(m,f)}
d.instanceMatrix.setUsage(Ci),i.add(d)}
let s=new gt,n=new ei,l=new I,h=new I,c=new I(0,1,0);
return{
group:i,update:u=>{
let d=et.uWind.value.x*3.2*u,f=et.uWind.value.y*3.2*u;
for(let p of o)p.x+=d*p.sp,p.z+=f*p.sp,p.x>5400&&(p.x-=15400),p.x<-1e4&&(p.x+=15400),p.z>3800&&(p.z-=9400),p.z<-5600&&(p.z+=9400),n.setFromAxisAngle(c,p.rot),s.compose(h.set(p.x,p.y,p.z),n,l.set(p.s,p.s*.8,p.s)),p.mesh.setMatrixAt(p.i,s);
for(let p of i.children)p.instanceMatrix.needsUpdate=!0}
,items:o}
}
var ir=(e,t,i)=>({
t:e,name:t,...i}
),rr=[ir(4.8,"night",{
zenith:"#0c143c",mid:"#18265c",fog:"#1d2c5e",fogSun:"#26386e",sun:[.5,.58,.92],shade:[.25,.29,.55],amb:[.02,.03,.07],disc:"#ffd9a0",cloudLit:"#3b4c86",cloudShade:"#1c2756",deep:"#101c4a",shallow:"#213266",foam:"#8fa2d6",glintC:"#c9d6ff",ink:"#05071a",fogD:17e-5,sunPow:3,night:1,stars:1,bloom:.7,glint:.35,cover:.6}
),ir(6,"first light",{
zenith:"#2c3a7e",mid:"#7a78b4",fog:"#8f86b6",fogSun:"#ffab8c",sun:[.74,.64,.8],shade:[.42,.42,.68],amb:[.03,.04,.08],disc:"#ffb48a",cloudLit:"#ffb9a0",cloudShade:"#6c6aa2",deep:"#3b4a86",shallow:"#8d8cb8",foam:"#f2e2e6",glintC:"#ffd9c0",ink:"#120f26",fogD:21e-5,sunPow:2.4,night:.45,stars:.3,bloom:.5,glint:.8,cover:.58}
),ir(7.2,"dawn",{
zenith:"#5b86cf",mid:"#b3c2e6",fog:"#d6cde0",fogSun:"#ffd0a6",sun:[1.07,.93,.8],shade:[.53,.54,.76],amb:[.04,.05,.09],disc:"#fff0c8",cloudLit:"#fff0e2",cloudShade:"#b9b0d6",deep:"#4f7fb0",shallow:"#9dcfd0",foam:"#fffaf0",glintC:"#fff4d6",ink:"#1a1428",fogD:21e-5,sunPow:2.6,night:0,stars:0,bloom:.36,glint:1,cover:.58}
),ir(10,"morning",{
zenith:"#3b8fe2",mid:"#8cc8f4",fog:"#d3e9f6",fogSun:"#f6f1dc",sun:[1.06,1.02,.95],shade:[.56,.6,.82],amb:[.04,.05,.08],disc:"#fffbe6",cloudLit:"#ffffff",cloudShade:"#c6d4ec",deep:"#2f80b8",shallow:"#66c6c6",foam:"#ffffff",glintC:"#ffffff",ink:"#1b1529",fogD:15e-5,sunPow:5,night:0,stars:0,bloom:.24,glint:.9,cover:.6}
),ir(13.2,"midday",{
zenith:"#2c84e6",mid:"#77c0f6",fog:"#d0eaf8",fogSun:"#eaf4f8",sun:[1.07,1.05,1],shade:[.57,.62,.84],amb:[.04,.05,.08],disc:"#ffffff",cloudLit:"#ffffff",cloudShade:"#c2d2ee",deep:"#2686c6",shallow:"#5fd2cc",foam:"#ffffff",glintC:"#ffffff",ink:"#1b1529",fogD:14e-5,sunPow:6,night:0,stars:0,bloom:.22,glint:.8,cover:.61}
),ir(16.4,"afternoon",{
zenith:"#3a88dc",mid:"#8fc4ea",fog:"#dde4e6",fogSun:"#fbe7b6",sun:[1.09,1.02,.9],shade:[.55,.57,.8],amb:[.04,.05,.08],disc:"#fff6d0",cloudLit:"#fffaf0",cloudShade:"#c8cde6",deep:"#3384ba",shallow:"#6fcdc4",foam:"#fffdf4",glintC:"#fff8dc",ink:"#1b1529",fogD:16e-5,sunPow:4,night:0,stars:0,bloom:.26,glint:1,cover:.6}
),ir(18.3,"golden hour",{
zenith:"#4a8ccc",mid:"#a9cbd8",fog:"#e6cdc0",fogSun:"#ffd27c",sun:[1.12,.99,.79],shade:[.52,.49,.73],amb:[.05,.05,.09],disc:"#fff2b4",cloudLit:"#fff0cc",cloudShade:"#d6a8b4",deep:"#447fa4",shallow:"#86c6b2",foam:"#fff6dc",glintC:"#fff0b0",ink:"#1d1426",fogD:21e-5,sunPow:2.6,night:0,stars:0,bloom:.42,glint:1,cover:.58}
),ir(19.5,"sunset",{
zenith:"#3c4f9e",mid:"#b47ca8",fog:"#a68ab4",fogSun:"#ff8c4c",sun:[1.1,.72,.52],shade:[.44,.39,.64],amb:[.05,.04,.08],disc:"#ff7a42",cloudLit:"#ffa880",cloudShade:"#86689e",deep:"#4a5896",shallow:"#a488ae",foam:"#ffd9c4",glintC:"#ffb47a",ink:"#190f22",fogD:23e-5,sunPow:2.2,night:.15,stars:.05,bloom:.55,glint:1,cover:.57}
),ir(20.4,"blue hour",{
zenith:"#1c2864",mid:"#5c4c8e",fog:"#4c4a7c",fogSun:"#c4627c",sun:[.5,.46,.74],shade:[.34,.34,.62],amb:[.03,.03,.07],disc:"#ff6a52",cloudLit:"#9a7fb4",cloudShade:"#46467e",deep:"#262f68",shallow:"#4c5088",foam:"#b9b2dc",glintC:"#ff9a8a",ink:"#0c0a20",fogD:2e-4,sunPow:2.4,night:.75,stars:.5,bloom:.65,glint:.5,cover:.58}
),ir(21.6,"night",{
zenith:"#0c143c",mid:"#18265c",fog:"#1d2c5e",fogSun:"#26386e",sun:[.5,.58,.92],shade:[.25,.29,.55],amb:[.02,.03,.07],disc:"#ffd9a0",cloudLit:"#3b4c86",cloudShade:"#1c2756",deep:"#101c4a",shallow:"#213266",foam:"#8fa2d6",glintC:"#c9d6ff",ink:"#05071a",fogD:17e-5,sunPow:3,night:1,stars:1,bloom:.7,glint:.35,cover:.6}
)],Z1=["zenith","mid","fog","fogSun","disc","cloudLit","cloudShade","deep","shallow","foam","glintC","ink"];
for(let e of rr)for(let t of Z1)e[t]=Rt(e[t]);
var dc=[{
id:"dawn",label:"Dawn",t:7}
,{
id:"day",label:"Midday",t:12.6}
,{
id:"golden",label:"Golden hour",t:18.15}
,{
id:"sunset",label:"Sunset",t:19.45}
,{
id:"night",label:"Night",t:22.5}
],pi=(e,t,i,r)=>r.set(st(e[0],t[0],i),st(e[1],t[1],i),st(e[2],t[2],i)),J1=class{
constructor(e){
this.pipe=e,this.t=18.15,this.target=null,this.flow=!0,this.hourSeconds=52,this.name="",this.night=0,this.sunUp=1,this.apply()}
goTo(e){
let t=((e-this.t)%24+24)%24;
this.target=this.t+t}
update(e){
if(this.target!==null){
let t=this.target-this.t,i=Math.max(.3,t*1)*e;
t<=i?(this.t=this.target%24,this.target=null):this.t+=i}
else if(this.flow){
let t=this.t%24,i=t>21.6||t<5?2.6:t>17&&t<20?.62:1;
this.t+=e/this.hourSeconds*i}
this.target===null&&this.t>=24&&(this.t-=24),this.apply()}
apply(){
let e=(this.t%24+24)%24,t=rr[rr.length-1],i=rr[0],r=t.t-24,a=i.t;
for(let d=0;
d<rr.length;
d++)rr[d].t<=e&&(t=rr[d],r=t.t,i=rr[(d+1)%rr.length],a=d+1<rr.length?i.t:i.t+24);
let o=bt(0,1,(e-r)/Math.max(1e-4,a-r));
this.name=o<.5?t.name:i.name,pi(t.zenith,i.zenith,o,et.uZenith.value),pi(t.mid,i.mid,o,et.uMidSky.value),pi(t.fog,i.fog,o,et.uFogCol.value),pi(t.fogSun,i.fogSun,o,et.uFogSun.value),pi(t.sun,i.sun,o,et.uSunTint.value),pi(t.shade,i.shade,o,et.uShadeTint.value),pi(t.amb,i.amb,o,et.uSkyAmb.value),pi(t.disc,i.disc,o,et.uSunDisc.value),pi(t.cloudLit,i.cloudLit,o,et.uCloudLit.value),pi(t.cloudShade,i.cloudShade,o,et.uCloudShade.value),pi(t.deep,i.deep,o,La.uDeep.value),pi(t.shallow,i.shallow,o,La.uShallow.value),pi(t.foam,i.foam,o,La.uFoamCol.value),pi(t.glintC,i.glintC,o,La.uGlintCol.value),La.uGlint.value=st(t.glint,i.glint,o),et.uFogParams.value.x=st(t.fogD,i.fogD,o),et.uFogParams.value.w=st(t.sunPow,i.sunPow,o),et.uNight.value=this.night=st(t.night,i.night,o),et.uStars.value=st(t.stars,i.stars,o),this.pipe&&(pi(t.ink,i.ink,o,this.pipe.inkU.uInk.value),this.pipe.finalU.uBloom.value=st(t.bloom,i.bloom,o),this.pipe.brightU.uThreshold.value=st(.82,.6,st(t.night,i.night,o))),this.cover=st(t.cover,i.cover,o);
let s=(e-6.2)/13.8*Math.PI,n=et.uSunPos.value.set(Math.cos(s),Math.sin(s)*.86,Math.sin(s)*.5+.2).normalize();
this.sunUp=bt(-.08,.06,n.y);
let l=((e+12)%24-5)/14*Math.PI,h=et.uMoonPos.value.set(Math.cos(l)*.7,.42+Math.sin(l)*.3,.55).normalize(),c=e>20.4||e<5.5,u=et.uSunDir.value;
c?u.copy(h):(u.copy(n),u.y=Math.max(u.y,.09),u.normalize()),et.uShadowParams.value.z=c?.55:st(.5,1,bt(.02,.2,n.y))}
};
var ds=new Cr,fc=new ei,pc=new I,mc=new I;
function fs(e,t=!0){
let i=new ti,r=new Float32Array(e.length*3);
for(let a=0;
a<e.length;
a++)r[a*3]=e[a][0],r[a*3+1]=e[a][1],r[a*3+2]=e[a][2];
return i.setAttribute("position",new si(r,3)),i.computeVertexNormals(),i}
var Mr=(e,t,i,r)=>[e,t,i,e,i,r],K1=class{
constructor(){
this.parts=[],this.c=[1,1,1],this.a=0,this.stack=[new gt]}
col(e,t=0){
return this.c=Rt(e),this.a=t,this}
push(e=0,t=0,i=0,r=0,a=1,o=0,s=0){
ds.set(o,r,s,"YXZ");
let n=new gt().compose(pc.set(e,t,i),fc.setFromEuler(ds),mc.set(a,a,a));
return this.stack.push(this.stack[this.stack.length-1].clone().multiply(n)),this}
pop(){
return this.stack.pop(),this}
add(e,t=0,i=0,r=0,a={
}
){
let o=e.index?e.toNonIndexed():e;
o.attributes.uv&&o.deleteAttribute("uv"),a.flat&&o.computeVertexNormals(),ds.set(a.rx||0,a.ry||0,a.rz||0,a.order||"YXZ");
let s=new gt().compose(pc.set(t,i,r),fc.setFromEuler(ds),mc.set(a.sx??1,a.sy??1,a.sz??1));
s.premultiply(this.stack[this.stack.length-1]),o.applyMatrix4(s);
let n=o.attributes.position.count,l=new Float32Array(n*4),h=a.col?Rt(a.col):this.c,c=a.a??this.a;
for(let u=0;
u<n;
u++)l[u*4]=h[0],l[u*4+1]=h[1],l[u*4+2]=h[2],l[u*4+3]=c;
return o.setAttribute("color",new si(l,4)),this.parts.push(o),this}
box(e,t,i,r=0,a=0,o=0,s){
return this.add(new wr(e,t,i).translate(0,t/2,0),r,a,o,s)}
cbox(e,t,i,r=0,a=0,o=0,s){
return this.add(new wr(e,t,i),r,a,o,s)}
cyl(e,t,i,r=0,a=0,o=0,s=10,n={
}
){
return this.add(new eu(e,t,i,s,1,!!n.open).translate(0,i/2,0),r,a,o,n)}
cone(e,t,i=0,r=0,a=0,o=10,s){
return this.add(new B0(e,t,o).translate(0,t/2,0),i,r,a,s)}
sph(e,t=0,i=0,r=0,a={
}
){
return this.add(new gu(e,a.detail??2),t,i,r,a)}
dome(e,t=0,i=0,r=0,a=14,o={
}
){
return this.add(new yu(e,a,Math.max(4,a>>1),0,Math.PI*2,0,Math.PI/2),t,i,r,o)}
torus(e,t,i=0,r=0,a=0,o={
}
){
return this.add(new yf(e,t,o.rs??6,o.ts??16,o.arc??Math.PI*2),i,r,a,o)}
gable(e,t,i,r=0,a=0,o=0,s){
let n=[-e/2,0,i/2],l=[e/2,0,i/2],h=[e/2,0,-i/2],c=[-e/2,0,-i/2],u=[-e/2,t,0],d=[e/2,t,0];
return this.add(fs([...Mr(n,l,d,u),...Mr(h,c,u,d),n,u,c,l,h,d]),r,a,o,s)}
hip(e,t,i,r=0,a=0,o=0,s={
}
){
let n=Math.max(0,e/2-(s.inset??i/2)),l=[-e/2,0,i/2],h=[e/2,0,i/2],c=[e/2,0,-i/2],u=[-e/2,0,-i/2],d=[-n,t,0],f=[n,t,0],p=n>.001?[...Mr(l,h,f,d),...Mr(c,u,d,f),l,d,u,h,c,f]:[l,h,d,h,c,d,c,u,d,u,l,d];
return this.add(fs(p),r,a,o,s)}
frustum(e,t,i,r,a,o=0,s=0,n=0,l){
let h=[-e/2,0,t/2],c=[e/2,0,t/2],u=[e/2,0,-t/2],d=[-e/2,0,-t/2],f=[-i/2,a,r/2],p=[i/2,a,r/2],x=[i/2,a,-r/2],g=[-i/2,a,-r/2];
return this.add(fs([...Mr(h,c,p,f),...Mr(c,u,x,p),...Mr(u,d,g,x),...Mr(d,h,f,g),...Mr(f,p,x,g)]),o,s,n,l)}
lathe(e,t=0,i=0,r=0,a=16,o){
return this.add(new _f(e.map(s=>new ge(s[0],s[1])),a,o?.phi0??0,o?.phi??Math.PI*2),t,i,r,o)}
ext(e,t,i=0,r=0,a=0,o={
}
){
let s=new Ss(e.map(l=>new ge(l[0],l[1])));
if(o.holes)for(let l of o.holes)s.holes.push(new Fs(l.map(h=>new ge(h[0],h[1]))));
let n=new Pn(s,{
depth:t,bevelEnabled:!!o.bevel,bevelSize:o.bevel||0,bevelThickness:o.bevel||0,bevelSegments:o.bevelSeg??1,curveSegments:o.curve??8}
);
return n.translate(0,0,-t/2),this.add(n,i,r,a,o)}
slab(e,t,i,r,a=0,o=0,s=0,n={
}
){
let l=new Ss,h=e/2,c=t/2;
r=Math.min(r,h,c),l.moveTo(-h+r,-c),l.lineTo(h-r,-c),l.absarc(h-r,-c+r,r,-Math.PI/2,0),l.lineTo(h,c-r),l.absarc(h-r,c-r,r,0,Math.PI/2),l.lineTo(-h+r,c),l.absarc(-h+r,c-r,r,Math.PI/2,Math.PI),l.lineTo(-h,-c+r),l.absarc(-h+r,-c+r,r,Math.PI,Math.PI*1.5);
let u=new Pn(l,{
depth:i,bevelEnabled:!!n.bevel,bevelSize:n.bevel||0,bevelThickness:n.bevel||0,bevelSegments:2,curveSegments:n.curve??5}
);
return u.rotateX(-Math.PI/2),this.add(u,a,o,s,n)}
plan(e,t,i=0,r=0,a=0,o={
}
){
let s=new Ss(e.map(l=>new ge(l[0],-l[1])));
if(o.holes)for(let l of o.holes)s.holes.push(new Fs(l.map(h=>new ge(h[0],-h[1]))));
let n=new Pn(s,{
depth:t,bevelEnabled:!1,curveSegments:o.curve??8}
);
return n.rotateX(-Math.PI/2),this.add(n,i,r,a,o)}
span(e,t,i,r,a={
}
){
let o=t[0]-e[0],s=t[1]-e[1],n=t[2]-e[2],l=Math.hypot(o,n),h=new wr(i,r,Math.hypot(l,s)).translate(0,-r/2,0);
return this.add(h,(e[0]+t[0])/2,(e[1]+t[1])/2,(e[2]+t[2])/2,{
...a,ry:Math.atan2(o,n),rx:-Math.atan2(s,l)}
)}
ring(e,t,i,r,a=0){
for(let o=0;
o<e;
o++){
let s=a+o/e*Math.PI*2;
this.push(Math.sin(s)*t,i,Math.cos(s)*t,s),r(this,o,s),this.pop()}
return this}
tri(e,t=0,i=0,r=0,a){
return this.add(fs(e),t,i,r,a)}
tube(e,t,i=6,r={
}
){
let a=new au(e.map(o=>new I(o[0],o[1],o[2])),!1,"catmullrom",r.tension??.5);
return this.add(new bf(a,r.steps??e.length*4,t,i,!1),0,0,0,r)}
beam(e,t,i,r={
}
){
let a=t[0]-e[0],o=t[1]-e[1],s=t[2]-e[2],n=Math.hypot(a,o,s),l=new wr(i,n,r.t2??i),h=new ei().setFromUnitVectors(new I(0,1,0),new I(a/n,o/n,s/n));
return l.applyQuaternion(h),this.add(l,(e[0]+t[0])/2,(e[1]+t[1])/2,(e[2]+t[2])/2,r)}
arcade(e,t,i,r,a=0,o=0,s=0,n={
}
){
let l=n.pier??.28,h=e/r,c=h*(1-l),u=n.spring??t*.5,d=[[-e/2,0],[e/2,0],[e/2,t],[-e/2,t]],f=[];
for(let p=0;
p<r;
p++){
let x=-e/2+h*(p+.5),g=c/2,m=[];
m.push([x-g,.001],[x-g,u]);
for(let y=1;
y<8;
y++){
let M=Math.PI-y/8*Math.PI;
m.push([x+Math.cos(M)*g,u+Math.sin(M)*g*(n.pointed?1.5:1)])}
m.push([x+g,u],[x+g,.001]),f.push(m.reverse())}
return this.ext(d,i,a,o,s,{
...n,holes:f}
)}
build(){
let e=Yu(this.parts,!1);
return e.computeBoundingSphere(),e.computeBoundingBox(),e}
mesh(e){
return new Nt(this.build(),e)}
};

var pe=()=>new K1;
// Public facade: game semantics do not enter this contract.
globalThis.COZY={Scene:Wc,Group:Ji,Mesh:Nt,InstancedMesh:Ri,Vector3:I,Vector4:Ft,Quaternion:ei,Euler:Cr,Matrix4:gt,Color:dt,Renderer:h1,Camera:ji,Box:wr,Sphere:gu,Plane:Br,Attribute:si,Geometry:ti,Texture:No,Shader:$i,FloatType:Rr,GLSL3:Po,seed:Mi,damp:Et,dampAngle:u1,clamp:Yt,smooth:bt,noise:ka,build:pe,material:bi,transparent:Ao,Pipeline:T1,uniforms:et,sky:q1,clouds:Y1,Mood:J1,waterColors:La,water:X1,merge:Yu,DoubleSide:2,DynamicDraw:Ci};
```
@@END:core.js@@

### fonts.css

@@BEGIN:fonts.css@@
```css
@font-face{font-family:Caprasimo;font-style:normal;font-display:swap;font-weight:400;src:url(data:font/woff2;base64,d09GMgABAAAAAAkcAA0AAAAADvAAAAjLAAEAAAAAAAAAAAAAAAAAAAAAAAAAAAAAG1YGYACBHBEICpIMjzwLLAABNgIkA1QEIAWEFgeBDgwHG44MUZRQVgTAj4OcjNec9PT0XGK13bJsAlML4lHc/CRrV/DA51nv/6QaM0sihWeysE7rk3CCcIPdhrELZoBO7bI53poUjHmhilyTIimMDHeb04koZgQvYN79GvEPlWZJtF2j8khWIqVsO5tgPsyGqCbVRshUtSTexEsmNkJoh4o1PYSALVPa7r17IYAYSqtJtm7buTcJYX1MMQmo8MfBqE3IM6oYO7wuqEguch6HdhKk3EagIJDmF6veEitYwLI2tYCLZX+yrBAyJcbuQKUosfJZPwMGiUq7bt3a0RxtK8vwaB/Y+ouwZhtNm1jWAo2EHCg87cFlJ5UFT5cniUIcmpZsLl0MinV9QV6SDxShKIpd00wT0LRABB5uzu3MJ+YpU0d0e3RcxBaxRgQQwURsj14RJMEy2CdwWIo08cQhEKZJYm+SDipungX5A2o+UN7XDCAaHIUuHUnW46WZnFFV2ykMR2zvFr06H4EsckIinkymYLHJ6ydhArihalwA2wGH0NhgBIU85x8/n1axfTcKOXbtFAqJYggjYQA+ceUIAO8kl+0Mhw1XTwICZ5u4fh2FbIv1Y0lbzgDw2NObxbjiAICEVmNi+9uPJ1BTkfFHt3TBBEnT/OL+/ehJmJEoCokMIRE0GoDDFNImdQB46BI6GEIHb9H2RWibdquRl2p4zrMIK5IgBpqdwsqL7DVEpvi2ArBczLioRwpiAJicoAxbgwkNcBnem91eP36OQjWfOqW9Nh6X440HV9qeTCgq2bwdhVxzlOGY+4yqr4wKGgH4afhlIPgAV8A8GGaqrYoORljoTZeK5Mbe9FUkdds5FPIaI6j2VLk95hE41dh48SJqZAIiErbuHNCRMgQ/uMkjWELEDxZRcUi1TnJW2Heujo6skCRaV2MT/ScbE8dISAdWnoUuDWnCE3ZJmVBvMXEV+VbXIpFxFPIcgymIQnQekWdj6igAE2MCyMPPA7CtbEX+hgNP+2s1SFLB5s1x5zeiXzjpzhkPjD5RkWb0n1JcdeQq01dw+Ppe8RSABYEmVrc+omRGk06rjsxXAAlpYeFWP8BoPOuR43GB4GcYd+IMWdcCMM84vX/CKx5gaQhDcZMIjIa8oZtmhGcXe+WyMjapEyQ5WQdMrYzmfk6xlwJTidfJCWuscpAK2mRaieibqxPnPjri3RcT9+Xwt3ym4wvQeibuqOZ6/CzY86z0v5QspnlP/BWBdokD/xd8Dq7wK9YsfIs7rS4EDWxLSP7pgx9FX9l3/wKUCHd9EbLOdTwpjwia5RrZgoog7y3B8nKkDvd/AlhT/Yx7uqJ1dz+yY11Lmn/2Nm/Vwj9+ugeMrnykobNjIVpu9JY25rUdd2cUezK0PuBIMi9v+pLZihieJ4Wnfb7vaezcMT5ObJfz2QtVULq7XZ4vqxcYeHhH4h+NqXezd/WVnYY3BB98Xh/keqcPTbfOprHOc3mUVOcaz38+ktShEfN3mK0NB1cvkXqlKskDHyWrMvU8tKor92BfV/HRrR03VYA7qU1rk4EmpV2uUcT8F89bXGqpbwue+FWrtctATSNcSLa3+VgSyb3mVnaou+UJPq4xsUosx+IAhoSoOCExu85nXMebbk6ZDKR93EcrY1NeXn/HgvmsJGZ3efWG+uWBvU8cw2mthJ2oDW9j0s5Q+NPj3WtaYlMU85UKm0BhAa2QQMddw1ZPfOEjbhP2e+wLGwOLtz3LaOFmT201Q3qA+h2VxzYTfRA4L47DcIrFLrMrr8wMKIjf8stZrHFhL/0zbkuqN/NKoRNsdhc6c7EWQhfUYTMa9DZSrURCqjU49JZOe6fHTJzmLrOUKAw2fU2MRFJDsuv1oLUEKp12OXs7HRZTB5SuYjB32DqzeQZYYro3uxDsyPS5R1OzeS30TwHl5F8ItrHudoVfLho3XUYPaqKdh66Vr4B2cpsay6DSOo/Lb9H7NRY+/R86RZcB+XWNJvw0oWRayv8OkjPub7bkL/7xuPtxD39+/Ug9h9Qqsxe7Sprz/HKrWOTgOUwsShGFp0x3QMpiYPKBAoGogLxY0Oh2pFsrC+xMgddqJZzJizwiJnWMMymX3CtU5aydhZWZ1sY8pznd89ESwYFTzvBbxCQY4N454Hhb7bQaxh4++4BHBylxVcl/nvvjHLYrluKk8z4AJh+4otE49Xt1GqdarXHq9mp1ju2H5iTS8vi5D+bIC7O0usy2Sii/r92ZfNASPkxMvQUQyN/osrSKwtY3eDm03GTKrVSGh/7v1iFNbikEtVYCjCmzDPV6n55Fuysjs6QJLraRMd/nz+xU/Hom4+5PZM59nGwDA/YVAMTQ5JKBkqrKwbLikv6yysqB0hO0S5cOllRXDZUAzF8mXDk2nTbX5tp76imDRHuDGnea/3x+mq07Haei9iyZlPZQ1T6yLCpgI8lb4aeYWNyTK0HsANY/hZWYV0B9DDTfUV/Z/JSCNcdWngTziXL8SiIAQct3qae3DiPof9hj7QA8+g6WBHi8d2kds1Hkt2PhTwGsSAAE2PpH+eCImrr/WY6FBMnTpfcmHgmAdYOmOaZJKjDQaqDU3CTBcob9e83crkLDHbjCaIg1KHeTVVvaWLqd7ExEtcslDXIdvShWqiPjLO/r3oIUd70lCVz1VqhvjbdKSaO8LeToT62XWV2acEnwMyb1P4LRNFQNXtLFKAJUOqbuVNVoGh6kwSh8mObdaEtnWqGvYhR6T4sWWUVPPAMdK0H1hfal8FGI3zIoRlNDVdVVV0M+CK3rKBtpR7d0qe1oZLc7pRcZRAUGnxgj90UvNTVVz2Utmt4j8PTqI+1+pRoEMdwlmdqCsWXhx4hzfk35thQ+AqjUsTgv6cKgiCPcGG0MPEQfUU1U4W0l1OIIM4orzq59rQqqkiThmilC8I7pQKG0MupqrC+zbPGIJ9wVUihCFRZhFTaec8QLh+4aO8bjMoqcjoZR+riaW28g7F4bVxvw/EcQcEesgDX6KKO4qksfgfuaB4U5d5+6FA==)format("woff2"),url(data:font/woff;base64,d09GRgABAAAAAAtAAAwAAAAADsAAAQABAAAAAAAAAAAAAAAAAAAAAAAAAABHUE9TAAABHAAAAEkAAABWK5okZk9TLzIAAAFoAAAAVQAAAGB30TaVY21hcAAAAcAAAAByAAAAnAyKMJxnYXNwAAACNAAAAAgAAAAIAAAAEGdseWYAAAI8AAAGwQAACPT5mo5jaGVhZAAACQAAAAA1AAAANiiHDghoaGVhAAAJOAAAAB8AAAAkEB8EV2htdHgAAAlYAAAAPAAAAFRF9gSHbG9jYQAACZQAAAAsAAAALBwFHoxtYXhwAAAJwAAAABwAAAAgACQBJm5hbWUAAAncAAAA9QAAAhYr40j8cG9zdAAACtQAAABrAAAAjl/1qz542mNgZGBg4GJQYTBiYHJx8wlh4MtJLMlj4GNgYQCC///B8ozZqUV5DBwgFggD5ZiAPEYw5gHyQGJCDIxAyAzETP8/AgCyDQlKAAAAeNpjYGG5zjiBgZWBgXUWqzEDA8MJCM2kxsjIJASkOThZWJi5mVhYGhgY8hmQgLu/vzuDA4OCggJrwr9SBgb2CMarCgwMjCA5FqBpQEqBgQkAJs4LCwAAAHjaTckxDgFBAADA2VsJCUTkolLsA3zC+Yj7hHiCWqvxFZV4hIjKK/aW5sK0gwoRtSigNnDEylw0kTQuoQ2HcKpu1b16xBBjHKVUCpLNd3d/q9/yKtfS0p27fd7mJq/J4zx8vgXA0s9Mb6E2NfkA168fTQAAAAEAAf//AA942q1VZWDiWBd9SQjptJBBmxZNSAmQ4KSBBmmRQtstVuou4+4+s76fu7u7u7u7u7u7S+n3ksq38nfRe897Cefcc98FoOAgAGgGXwUYIABIGBmjlzEyB5FvtJvII9pX8dV/v9CPG+C+HADaGfwIKINRALwWLct4uH5RSkpS0ixJ/WIYhVmCiXdbLSRqtXRTZhdKdcNsZ2cOrnM+H0NotVaLC8UYDJ0oSu0fSKWE2+HQ/UTb5fa7ixL6Rano6HPsw79gMHOCxwG3IFKJjbAETreDeIctOhiNIKUXnhwQE3I04rQNseirjKyZstnDgoIFQjZHj5ExbL4kMp5negNBOQHBsJ0ykg79IbIn3etgmTjyG+RvAGrOAoC/EapiQRAkAUAg9z26xj1NLhTKIFwowhoTcaiVM4scy0KRMMPfOP+U17b++0cpH5AC8JWX0I31L7zx5PNOzDxnfZPQ9p2dOvOSpdbJsfpoGhk/dNeIQxyIJ2iBR78d5N3xxIDIr77w0p1Poc3Ne9C/tNm7Dp+5578fbhx9zfjCNCQE5re+q/kF/kjQgkw9nI/jWI+WVQL4UtjFXZjKCeZhDAIQ6YZ1hxgMuilKdWD3QqsFQi5MsQh9FT8ZMPWZ+QYvh7r0lA4bFnwDqdHTk/HDsTJ99OMvWH7MLx8daPqZoofLBPiEu7NBlCMCby2+/L5YMzvtXXvrE1oLz/6ws4PAcNdQIncrp9NHRLFWFUNZPmLf565feuMdT/3U4Y7ODrMYqzBbPZaYHGKOPDGejA4wOnft3Msvnn31mZSikoXdZYM+DO31lmpBP6LI5B7cWQ9uOC1sKgJR2krzoZK0+aH+4fhQvP0SigoI0WEJHZBKQlKwdn7ka4TeF/eVJISSSpzH46Xar47lYyflhDggiu4Afy9a8IXtTk5WeiYet9vtlpCx/SbkvM5JUr32eETBw4LNHuE2P/Rk3u8SFe7k1o+w92toMAETtcQULLLaMmrh2V3LFCX3d+z+hrnQ2I5h3duGhVGk7ZFcDlfBvfjCjRufu+Y+aUM6w13uIC3EejsJU6pfLPUna+kAn4lMP+fy8H1fOB6o+3BvHxNheL8R13R4I5GRkWXsostHe3R6HWFZedqhR724bCBxXSxVZRxJpzgUKhRDMV+K3mcMN+45cv05Y1o9Quoj+WkXFaH8lfxkXlH49vancD3+C3AbALjaTQp5TrUoSaIwV3Rt0w+ju6Gin8ihu6FaABe2bRaWlzgu08e7b0tWK8bCuTO5bNZqzWXyZ8/mjdVC6UA0emC4WIMrZ/PKSjY7dOZswTRaT1bdQt/vBLc7dOEXJ5rvmi0fn2uFDLVS6WA0erA83DAmlxfFbNZizWakxaWkoVkuKyulUt0gNOfOjE29r37i5wDZetXmPzVf0BbBmHLmiURcqTmLkRjr2fUIEt9psqQZfmMKmkhK6tK2Q3Dz9ihrIO87NxN+/1CXfPY9j60+/9hSvTKTlQU2mqllkHC0XIkOCa/p6fnrp1M3Tw4fn04UI4LI0ZxUlrC/7u84JYqbI1e/+YoV/uhS9VBhphBi6bGZo4Mkgca0TtraE/RFhU9++qvWxEJlYTJaLkd5mh+tLQAUyBo79gn858ACQgCY4TBWy60WnlAr7hWTkDJ8bcPE3oIcMfWGqiOFtVCReRPpNNh9yAd0VY0Ws82NFY6ORvaZu95Augz2gGYmOMtJR8uzJW+zaJgkumiOph7XsZ/IrU2Whxosoccr2i5G6ZINzQT2fK0H9MJku/V3JhScQrsUCPS53lyqXAx+2i3SlM/0738Z/d10v1vzUi6YC3+B4VmfSf+Wt+hNPpZnAAIVytjH8RTQQ327p10dD7KcQV+UluX05oqc0bxzoFJJpSpl9QoSZfDfAot6xV5J9oSzQX2POJ4eO3Wp8nLSbnCFNE3bZKs3NJdeq06edha0HezOXeidu8R3j7MiSe3vP6XH+ylSYEOuvgYtX2udxH+bngv1TrQcNNtH2uSlanUNqHfRYTj+NOCGiUcZA+ph2T0gifieXehQ9vA053FaPFzM5p0eyx6e8fY5DR6/kmmM2SyX4EZ8uTtm1Kjiy945C1T/MeyX+C3QBSj1F/rFHaOZ+8WIfuHK1fmFy1eQ1yrB4tVrGmxxbXV5eXV1SfleXF9bhFwHNC2M0QJgv597D7FPS6Ar3om03PK+WG8OM5Rg/uIXLQJFh02k5gP+Usn/BtJr8QQMujvu0BkCjMW7/+FkubW1/W9IHEE5oAEAEMAAfo0cgbj6/0H4IY6ruBEhkVdDXJ3NWg/EoTT47vwF+B9s3dYpAAAAeNpjYGRgAGKnsA5LhXh+m68M3BwMIHD/i5kSiH6wvtf0v/XfH5yd7DVALhsDE0gUADgKC/YAAAB42mNgZGBgj/inzcDA+e+/9R8Jzk6gCCoQBQCIBAWHAHjaY5rPkMrEwMDANpfBjnUvgy2rJkMUmy2DMvMFBj7WwwzbWM7+X8nCwGAFxMkgmgETW4JoZH0AZuIMvAAAABQAFAB/ANABQwGhAhICewLeAyUDTwNoA4sDrgPeBAQEMQRXBGMEbwR6eNpjYGRgYBBlWM3AwVDJwALmIQATAyMAHjIBS3jahZADbkRRAEVPbcYNiqC226C2HY/Nj2IfXVLjrmDWMIuYm8mYebr8eEA3Ptpoae8B/kjncAujpHK4lUH+c7iNHX5zuJ0pjBzuYIyPHO6UvpXDg0IQwNJIYLLHsoaJG4OgFEt4CVM4ojMu1S//njNuOMaphKFdPlG5izzhVcJW2innTcxArrwYE6yyxIrGao3uREW3mN3nnHuNc6Gm76xg1fljrQQ/GOL+7J9PsMaK1rrQixSvzlrf94ChMyTfnW0dYusMSDMwxWeq7tGvrhLKuVjCrWRUqkenGwOvkk6CQov40A0pa8p3V7271J9lKQPEe1UAAAAAeNpjYGIAg/9CjIwM2IAoEDMzqDCoMpgx2DCEMTQxLGO4z/CQ4RYjEyMzIwsjKyMbIzvDE4bdDE85E5NLS1KT83OTONOLEssgrJLMnBQwiyclM7UotTizGCycnFiUn6eXnFgAFwZxADjmISYA)format("woff");unicode-range:U+100-2BA,U+2BD-2C5,U+2C7-2CC,U+2CE-2D7,U+2DD-2FF,U+304,U+308,U+329,U+1D00-1DBF,U+1E00-1E9F,U+1EF2-1EFF,U+2020,U+20A0-20AB,U+20AD-20C0,U+2113,U+2C60-2C7F,U+A720-A7FF}
```
@@END:fonts.css@@

### ui.css

@@BEGIN:ui.css@@
```css
:root{--ink:#262139;--paper:#fbfaf5}*{box-sizing:border-box}body{margin:0;background:#d3d6bc;color:var(--ink);font-family:system-ui,sans-serif}canvas{position:fixed;inset:0;width:100%;height:100%}.card{position:fixed;left:24px;top:24px;max-width:min(340px,88vw);background:#fbfaf5ee;border:3px solid var(--ink);border-radius:14px 20px 13px 18px;box-shadow:4px 5px var(--ink);padding:18px;transform:rotate(-1deg)}h1{font-size:24px;margin:0 0 10px;color:#31587d}p{font-size:12px;line-height:1.8}button{min-height:46px;background:#efc85f;border:2px solid var(--ink);border-radius:9px 12px;box-shadow:0 3px var(--ink);padding:8px 14px;font:inherit;cursor:pointer}button:active{transform:translateY(2px);box-shadow:0 1px var(--ink)}button:focus-visible{outline:3px solid #31587d;outline-offset:4px}@media(max-width:600px){.card{left:16px;top:16px;padding:13px;max-width:240px}h1{font-size:20px}}
```
@@END:ui.css@@

### adapter.js

@@BEGIN:adapter.js@@
```javascript
(()=>{'use strict';try{const C=COZY,U=C.uniforms,renderer=new C.Renderer({canvas:document.querySelector('canvas'),antialias:false,alpha:false});renderer.outputColorSpace='srgb-linear';renderer.toneMapping=0;const pipe=new C.Pipeline(renderer,{shadowRes:1024});pipe.shadowSize=90;pipe.finalU.uTime={value:0};const scene=new C.Scene(),camera=new C.Camera(48,1,.5,12000),mood=new C.Mood(pipe);mood.flow=false;mood.t=18.3;mood.apply();scene.add(C.sky());const mat=C.material({hatch:true}),soft=C.material({hatch:false});function add(b,x=0,y=0,z=0,m=mat){const o=new C.Mesh(b.build(),m);o.position.set(x,y,z);scene.add(o);pipe.cast(o);return o;}
add(C.build().col('#aab68b').box(85,2,65),0,-2,0);add(C.build().col('#e5d7b8').box(70,.08,6),0,.03,8);let b=C.build();b.col('#e5c6a0').box(13,9,11).col('#bb7654').hip(14,3,12,0,9,0);for(let x of [-3,3])b.col('#f3e7cd').box(2,2.8,.2,x,3,5.6).col('#638798').box(1.45,2.2,.22,x,3.3,5.75);b.col('#587480').box(1.8,3,.25,0,0,5.6);add(b,-12,0,-8);let t=C.build();t.col('#887057').cyl(.45,.6,5,0,0,0,7).col('#7b9970').sph(3.2,0,6,0,{detail:1,sy:.8}).sph(2.2,1.6,7,.4,{detail:1});add(t,12,0,-8,soft);let vase=C.build();vase.col('#4a6e9c').lathe([[.6,0],[1.1,.3],[1.3,1.5],[.7,2.2],[.65,2.5]],0,0,0,12).col('#e7c259').cyl(.07,.09,2,0,2,0,6).sph(.5,0,4.3,0,{detail:1});add(vase,4,0,5);const water=C.water({x0:-250,x1:250,z0:-180,z1:-34},(x,z)=>z+34);water.geometry.scale(500/60000,1,146/60000);water.position.set(0,-.2,-107);scene.add(water);add(C.build().col('#b0b79a').sph(28,0,0,0,{detail:1,sy:.2,sx:1.8}),-45,-2,-145,soft);camera.position.set(40,31,54);camera.lookAt(0,3,-4);U.uTime.value=0;function render(){pipe.updateShadow(new C.Vector3(0,0,0),U.uSunDir.value);pipe.render(scene,camera);}function resize(){const d=Math.min(devicePixelRatio,1.5);renderer.setPixelRatio(d);renderer.setSize(innerWidth,innerHeight,false);pipe.setSize(innerWidth,innerHeight,d,1);camera.aspect=innerWidth/innerHeight;camera.updateProjectionMatrix();render();}addEventListener('resize',resize);document.querySelector('#mood').onclick=()=>{mood.t=mood.t<19?21.6:18.3;mood.apply();render();};resize();window.__style={render,renderer};}catch(e){document.querySelector('#status').textContent=e.message;console.error(e);}})();
```
@@END:adapter.js@@

### shell.html.txt

@@BEGIN:shell.html.txt@@
```html
<!doctype html><html lang="zh-CN"><meta charset="utf-8"><meta name="viewport" content="width=device-width,initial-scale=1"><title>墨线暖光 · 静态风格展示</title><style>__CSS__</style><canvas aria-label="墨线暖光静态品质板"></canvas><div class="card"><h1>墨线暖光</h1><p id="status">固定构图 · 地面 / 墙面 / 树 / 花器 / 水 / 天空<br>这里只展示画面风格，不包含玩法模板。</p><button id="mood">切换暖暮 / 月夜</button></div><script>__JS__</script></html>
```
@@END:shell.html.txt@@

## 9. 代码完整性

```json
{
  "engine.js": {
    "bytes": 570924,
    "sha256": "d6387df32eaddc7a56e52d5118593c0ab5a7267a7fb0193b22b36e15d054d97c"
  },
  "core.js": {
    "bytes": 51848,
    "sha256": "93cb6190f51f7f1930695e85fc0c6b4584f9228e4eb9323491aff315cd5c3c7d"
  },
  "fonts.css": {
    "bytes": 7307,
    "sha256": "2edba0461c557c3ff7f19dbe27169fc9d04d05a29270fff3da33d6761b7227c1"
  },
  "ui.css": {
    "bytes": 890,
    "sha256": "11c5adb71d25e7634b152ebd48161bd0e87360b6125d04ff6a09613b48edbf70"
  },
  "adapter.js": {
    "bytes": 2238,
    "sha256": "50bc5c6f8dccea6e9241f6b259f03f7b4bdb0a9a0feb97fe6511a417b7854931"
  },
  "shell.html.txt": {
    "bytes": 513,
    "sha256": "d2783ba99d2edae2412f4f7e3541911c0803076a3c7040e17e1c9a0a685dbd70"
  }
}
```
