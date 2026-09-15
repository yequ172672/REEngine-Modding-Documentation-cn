# RE Engine 模型与纹理替换 Mod 流程（OWOTS 补充）

本文面向已经在 Blender 中导入目标 `.mesh`、完成新模型绑定的情况。上游通用教程位于相邻目录 `../REEngine-Modding-Documentation-wiki/`；OWOTS 的版本信息则以本地 `RE-Mesh-Editor-main` 分支为准。

## 1. 导出前检查模型和 MDF

本地 CN 分支新增了按集合增量导出入口：在“打开 Mod 工作区”下方点击“导出 Mod 工作区”或“导出 Mod PAK”，可自动检测修改、转换图片并恢复游戏路径。新版导入会保存基线；旧工程需要先绑定原始文件恢复追踪。详细使用和限制见 `../RE-Mesh-Editor-main/docs/workspace-export.md`。下文的手动转换和导出流程仍可使用。

先在 Outliner 选中模型父集合或 Mesh 集合，导出按钮仅识别这些集合中的模型，并自动关联 MDF 与纹理。无需分别勾选 Mesh/MDF；对话框列出识别结果与修改摘要，未选中的模型不参与。

1. 每个网格对象只保留一个材质，并放入导入时生成的红色 `.mesh` Collection。
2. 对象命名保持 `Group_0_Sub_0__MaterialName` 一类的导入格式。
3. `.mesh` 与 `.mdf2` 的材质数量、名称和大小写必须完全一致，否则游戏中会出现橙白棋盘格或模型不可见。
4. 不要把 Blender 节点材质当成游戏材质；游戏实际读取的是 `.mdf2` 中的参数和 Texture Bindings。
5. 用当前 OWOTS 分支导出，不要手改版本后缀。该分支当前映射为 mesh `.260209350`、MDF `.51`、tex `.251111100`。

来源：Wiki `Custom-Model-Importing-Guide.md` 的 “Rules for RE Engine Meshes / MDF Editing / Tex Conversion”；`Troubleshooting.md` 的 checkerboard 条目；本地 `RE-Mesh-Editor-main/__init__.py`、`modules/tex/enums/game_version_enum.py`。

## 2. 修改纹理

工作区自动导出以原 TEX 的压缩格式与 sRGB 标记为准，忽略 Blender 图像色彩空间，不修改作者的图片设置。通道须由作者提前重组；“置空纹理绑定”只修改选中 MDF 材质的绑定路径，不自动重建视口材质。

先看原 MDF 材质的 Texture Bindings，确定真正需要替换的纹理类型。通常是 ALBD/ALBM（颜色）、NRMR（法线与粗糙度）以及 ATOS/ATOC 一类的遮罩贴图。

- ALBD：RGB 放颜色，Alpha 放反相 metallic/dielectric 信息；ALBM 的 Alpha 放 metallic。
- NRMR：R/G 放经过 RE Engine 布局处理的法线数据，B 通常不用，Alpha 放 roughness。不要只把普通 RGB 法线图原样另存为 NRMR。
- ATOS/ATOC/ACOT/ATOD：常见用法是 R 放透明遮罩、B 放 AO，其余未用通道填白；透明材质还要在 MDF 中启用 `BaseAlphaTestEnable`，并确保所用 master material 支持透明。
- DDS 保存格式：颜色贴图使用 `BC7 sRGB`，其余贴图使用 `BC7 Linear`。

如果只是替换原纹理，可沿用原 Texture Binding 路径和文件名。如果是给新模型建立自定义纹理路径，则在 MDF 的 Texture Bindings 中改成相对 `natives/STM/` 的路径：路径必须带 `.tex`，但不能带后面的数字版本。例如：

```text
MyMod/Character/body/body_albd.tex
```

来源：Wiki `Textures.md` 的 “About RE Engine textures / Texture Types”；`Custom-Texture-Setup-Guide.md`；`Custom-Model-Importing-Guide.md` 的 “Converting Textures to RE Engine Layouts / MDF Editing”；`Troubleshooting.md` 的 texture path 条目。

## 3. 在 Blender 中转换并放置 TEX

1. 在 RE Mesh/MDF 工具面板中把游戏选为 `Onimusha: Way of the Sword`。
2. `Image Directory` 指向保存 DDS 的目录。
3. `Mod Directory` 指向模组工作目录内的 `natives\STM`。
4. 依次点击“将目录转换为 Tex”（Convert Directory to Tex）和“复制转换后的 Tex 文件”（Copy Converted Tex Files）。
5. 检查转换数量与 Blender 系统控制台；插件会依据 MDF 的绑定路径复制 `.tex.251111100`。
6. “Apply Active MDF”只用于 Blender 中预览与检查路径警告，不会替代导出 MDF。

来源：Wiki `Custom-Model-Importing-Guide.md` 的 “Tex Conversion”；本地 `RE-Mesh-Editor-main/modules/tex/` 与 OWOTS 映射。

## 4. 导出并保持原游戏路径

用 `File > Export > RE Mesh/MDF` 导出目标 Collection，输出到模组根目录下与原文件完全相同的路径。典型结构如下（`...` 必须替换为原始游戏相对路径）：

```text
MyMod/
└─ natives/
   └─ STM/
      └─ .../
         ├─ target.mesh.260209350
         ├─ target.mdf2.51
         └─ custom/
            ├─ body_albd.tex.251111100
            ├─ body_nrmr.tex.251111100
            └─ body_atos.tex.251111100
```

若只改纹理，不必同时放 mesh；若材质名称、数量或绑定路径有改动，则必须一并放置新 MDF。

来源：Wiki `Custom-Model-Importing-Guide.md` 的导出步骤和 `Packaging-Mods.md` 的 file replacement 目录规范。

## 5. OWOTS 必须制作 patch PAK

通用 Wiki 明确说明：MH Wilds 及更新游戏不能 loose-load `.tex`，纹理必须放入 patch PAK。OWOTS 属于该新格式范围；本地 OWOTS 插件也按此流程实现。因此不要只把 `.tex` 作为 loose file 复制进游戏目录，否则可能黑屏。

1. 安装并启用 RE Asset Library；否则 RE Mesh 面板不会出现“创建 PAK 补丁”（Create Pak Patch）。
2. 点击该按钮，`Mod Directory` 选择上例中**包含 `natives` 文件夹的 `MyMod` 目录**，不是 `natives` 本身。
3. 选择输出 `.pak` 路径并生成。开发测试时可将该 PAK 放到游戏根目录的 `pak_mods`；插件的“复制 Mod 文件到游戏”也能自动复制工作区生成的 PAK。
4. patch PAK 需要最新版 REFramework 才能加载。也可以用支持该游戏的新版 Fluffy Mod Manager 安装生成的 patch PAK。

分发时可保留一个标准 Fluffy 模组目录，根部放 `modinfo.ini`、可选 `thumbnail.png` 和生成的 patch PAK，再压成 ZIP；先用你实际使用的 Fluffy 版本本地安装/卸载一次，确认它把 PAK部署到了 `pak_mods`。Wiki 的旧式 `modinfo.ini + natives` ZIP 结构仍适合纯 loose-file 替换，但不满足 OWOTS 纹理必须进 patch PAK 的要求。

来源：Wiki `Custom-Model-Importing-Guide.md` 的 2026-02-27 patch-pak 更新；本地 `RE-Asset-Library-cn/modules/pak/re_pak_operators.py`；本地 `RE-Mesh-Editor-main/README.md` 的 V0.45 说明；Wiki `Packaging-Mods.md`。

## 6. 最短排错清单

- 棋盘格/不可见：mesh 与 MDF 的材质数量、名称或大小写不一致。
- 无限加载：MDF 的纹理路径错误；确认含 `.tex`、不含 `.251111100`，且不含开头的 `natives/STM`。
- 黑屏：OWOTS 的 `.tex` 被当 loose file 放置，或 patch PAK/REFramework 未正确加载。
- 材质异常：使用了旧补丁版本的 MDF，或通道打包、sRGB/Linear 选错。
- 模型 T Pose/变形：权重或 Armature 关系错误；OWOTS 当前每顶点最多六个权重，导出前检查超限权重。
- 游戏仍显示原版：替换路径没有从 `natives` 开始精确复刻，或 Fluffy 中模组未真正启用。

## 本地资料版本

- 主仓库：`f4da2d63d92fa154e6f1d7bf1cf2cc82ecad3484`（2026-08-24）
- Wiki：`ed5ad82f394ad730673b595635a935de5d2a314b`（2026-08-24）
