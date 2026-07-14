# StackVln

## **UNDER CONSTRUCTION**

## **Bring Scanpy-style stacked violin plots to Seurat.**
- 📊 **Stacked Violin Plots**: Create compact, Scanpy-style stacked violin plots colored by median expression.
- 🌳 **Automatic Clustering**: Dendrogram-based cluster ordering `BuildClusterTree()`
- 🎨 **Customizable**: Adjustable colors, sizes, and cluster orders
- 💾 **Easy Export**: Built-in .png saving.

🌍 [**Multilingual Overview**](https://github.com/AyumuOkumura/StackVln/blob/master/README.md#-multilingual-overview-1)
---
## AI Usage

This package was made by AI coding tools (Gemini and Github copilot)

## Installation

```r
# install.packages("devtools")
devtools::install_github("AyumuOkumura/StackVln")
```

## Quick Start

```r
library(Seurat)
library(StackVln)

# Load your Seurat object
seurat_obj <- readRDS("seurat_obj.rds")

# Create a stacked violin plot (uses DefaultAssay automatically)
# Save to file
ndim = 20
StackVln(
  seurat_object = seurat_obj,
  features = c("IL7R", "CCR7", "S100A4"),
  group.by = "seurat_clusters",
  dendrogram_method = "dims",
  ndim = ndim, # default 30 dimensions (ndim not specified)
  plot_width = 12,
  plot_heights = c(0.3, 3),
  # Define the height ratio between the "top graph (dendrogram)" and "bottom graph (violin plot)"
  save_dir = "./figures",
         )

genes <- c("CD3D", "CD8A", "CD4", "MS4A1", "CD14")

# With custom tag for organizing marker sets
StackVln(
  seurat_object = seurat_obj,
  features = genes,
  group.by = "seurat_clusters",
  dendrogram_method = "dims",
  ndim = 30,
  plot_width = 5,
  plot_heights = c(0.3, 3), 
  color_high = "#BD2130",
  tag = "immune_markers",  # Adds descriptive tag to filename
  save_dir = file.path("png/ViolinPlot")
)
# Output: seurat_obj_immune_markers_stack_vln.png
```
## Output

### Default Filename Format
When using `save_dir`, the output filename follows this pattern:
- **Without tag**: `{seurat_object}_stack_vln.png`
- **With tag**: `{seurat_object}_{tag}_stack_vln.png`

### Examples

```r
# Default filename
StackVln(seurat_obj, features = c("CD3D", "CD8A"), save_dir = "./plots")
# Output: seurat_obj_stack_vln.png

# With tag - for organizing different marker sets
StackVln(seurat_obj, features = c("CD3D", "CD8A", "CD4"), 
         save_dir = "./plots", tag = "tcells")
# Output: seurat_obj_tcells_stack_vln.png

StackVln(seurat_obj, features = c("MS4A1", "CD79A"), 
         save_dir = "./plots", tag = "bcells")
# Output: seurat_obj_bcells_stack_vln.png
```

**Note**: The `tag` parameter is only used when `save_dir` is specified. If you use `save_path` to provide a full file path, the `tag` parameter will be ignored.

<img src="example/pbmc_stack_vln.png" width="600" alt="PBMC Stacked Violin Plot">

## How StackVln Works: Step-by-Step Computation Details

Understanding the computational workflow behind `StackVln()` helps you make informed decisions about parameters and interpret your results. Here's what happens under the hood:

### Step 1: Data Preparation and Validation

1. **Assay Selection**: If no assay is specified, the function uses `DefaultAssay()` to select the assay
2. **Feature Validation**: The function checks that requested features exist in the selected assay
3. **Group Validation**: Verifies that the `group.by` column exists in the Seurat object's metadata

### Step 2: Cluster Tree Construction

The dendrogram is built using Seurat's `BuildClusterTree()` function. The computation method depends on your `dendrogram_method` parameter:

#### Method A: "dims" (Recommended)
```
1. Extract dimensionality reduction data (e.g., PCA)
   → Uses the specified reduction (default: "pca")
   
2. Calculate cluster centroids in reduced space
   → For each cluster, computes mean coordinates across specified dimensions (default: 1:30)
   
3. Compute pairwise distances between cluster centroids
   → Uses Euclidean distance in the reduced dimensional space
   
4. Perform hierarchical clustering
   → Method: Ward's linkage
   → Creates dendrogram based on cluster similarities
```

**Advantages**: Computationally efficient, robust to noise, captures global transcriptomic relationships

#### Method B: "features"
```
1. Extract expression data for specified features only
   → Uses only the genes you're plotting
   
2. Calculate average expression per cluster
   → For each cluster, computes mean expression of each feature
   
3. Compute correlation/distance between clusters
   → Based solely on the selected marker genes
   
4. Build hierarchical tree
   → Ward's linkage clustering on feature-specific relationships
```

**Advantages**: Shows relationships specific to your markers, useful for focused analysis

#### Method C: "all_variable"
```
1. Identify all variable features in the assay
   → Uses features marked as variable in Seurat object
   
2. Extract expression matrix for all variable features
   → Typically 2000-3000 genes
   
3. Calculate cluster-wise average expression
   → Mean expression across all variable features
   
4. Perform hierarchical clustering
   → Creates comprehensive gene-based dendrogram
```

**Advantages**: Comprehensive gene-based approach, no dimensionality reduction needed

### Step 3: Cluster Ordering

1. **Extract dendrogram structure** from the built cluster tree
2. **Determine optimal leaf order** using hierarchical clustering results
3. **Override with manual order** if `cluster_order` is specified by user

### Step 4: Expression Data Extraction and Scaling

```
For each feature:
1. Extract raw expression values from the specified assay
2. Group cells by cluster (using group.by parameter)
3. Scale expression to 0-1 range within each feature
   → Formula: (value - min) / (max - min)
   → This enables color mapping across different expression scales
```

### Step 5: Violin Plot Generation

```
For each feature × cluster combination:
1. Extract expression distribution for all cells in that cluster
2. Calculate kernel density estimation
   → Creates smooth distribution curve
3. Mirror the density plot to create violin shape
4. Color violin by mean expression (scaled 0-1)
   → Low expression → color_low (default: white)
   → High expression → color_high (default: #BD2130)
```

### Step 6: Layout Assembly

1. **Create dendrogram plot** with height ratio from `plot_heights[1]`
2. **Create stacked violin plots** with height ratio from `plot_heights[2]`
3. **Align plots** vertically using `patchwork` package
4. **Calculate total plot height**: 
   - If not specified: `max(5, length(features) × 0.4 + 2)` inches
   - Ensures adequate space for all features

### Step 7: Export

1. **Create output directory** if it doesn't exist
2. **Save plot** as PNG with specified dimensions
   - Default filename: `{seurat_object_name}_stack_vln.png`
   - Resolution: High-quality for publication

### Key Computational Notes

- **Memory efficiency**: The function processes one feature at a time rather than loading all data at once
- **Scaling strategy**: Per-feature scaling (not global) ensures each gene is visually comparable
- **Clustering algorithm**: Uses Ward's criterion (`ward.D2`) which minimizes within-cluster variance
- **Distance metric**: Euclidean distance in PCA space (for "dims" method) or expression space (other methods)

## Advanced Usage
### Custom Colors

```r
StackVln(
  seurat_object = seurat_obj,
  features = c("IL7R", "CCR7", "S100A4"),
  group.by = "seurat_clusters",
  dendrogram_method = "dims",
  plot_width = 12,
  plot_heights = c(0.3, 3),
  color_low = "lightblue",
　color_high = "darkred",
  save_dir = "./figures",
         )
```

### Adjusting Label Margins for Long Cluster Names

When cluster names are long and rotated at 45°, they may be cut off at the **bottom** or **left side** of the plot. Use margin parameters to ensure full visibility.

#### Understanding the Problem

```r
# Visualization of the cutoff issue:
#
#        [Plot Area]
#   |                    |
#   |  [Violin Plots]    |
#   |                    |
#   +--------------------+
#    ↓ Bottom cutoff      ← Left cutoff
#   Very-Long-Cluster-N..  ry-Long-Cluster-Name-1
```

#### Bottom Margin Adjustment

Use `x_label_bottom_margin` when cluster names extend **below** the plot area:

```r
# Default (1 inch) - good for short names
StackVln(seurat_obj, 
         features = c("CD3D", "CD8A"),
         x_label_bottom_margin = 1)  # Default

# Increase for longer names
StackVln(seurat_obj, 
         features = c("CD3D", "CD8A"),
         x_label_bottom_margin = 2)  # Moderate

# Maximum space for very long names
StackVln(seurat_obj, 
         features = c("CD3D", "CD8A"),
         x_label_bottom_margin = 3)  # Large
```

#### Left Margin Adjustment

Use `x_label_left_margin` when the **leftmost cluster name** is cut off on the left side:

```r
# Default (1 inch)
StackVln(seurat_obj, 
         features = c("CD3D", "CD8A"),
         x_label_left_margin = 1)  # Default

# Increase for long leftmost cluster name
StackVln(seurat_obj, 
         features = c("CD3D", "CD8A"),
         x_label_left_margin = 2)  # Moderate

# Maximum space
StackVln(seurat_obj, 
         features = c("CD3D", "CD8A"),
         x_label_left_margin = 3)  # Large
```

#### Adjusting Both Margins

For datasets with consistently long cluster names:

```r
# Adjust both bottom and left margins
StackVln(seurat_obj, 
         features = c("CD3D", "CD8A", "CD4"),
         group.by = "detailed_cell_type",
         x_label_bottom_margin = 2.5,  # Bottom space
         x_label_left_margin = 2.5)     # Left space
```

#### Practical Guide

| Cluster Name Length | Bottom Margin | Left Margin | Example Names |
|---------------------|---------------|-------------|---------------|
| Short (< 10 chars) | 1.0 (default) | 1.0 (default) | "0", "1", "T cell" |
| Medium (10-20 chars) | 1.5 | 1.5 | "Naive CD4 T", "Memory B" |
| Long (20-30 chars) | 2.0 | 2.0 | "CD8+ Effector Memory T cells" |
| Very Long (> 30 chars) | 2.5-3.0 | 2.5-3.0 | "CD4+ Central Memory T cells (CCR7+)" |

**Troubleshooting Tip**: 
1. Save your plot with default settings
2. Check if names are cut off at the **bottom** → increase `x_label_bottom_margin`
3. Check if names are cut off on the **left** → increase `x_label_left_margin`
4. Adjust incrementally by 0.5 inches until labels are fully visible

### Dendrogram Calculation Methods

The `StackVln()` function uses Seurat's `BuildClusterTree()` to calculate hierarchical relationships between clusters. Three methods are available:

#### Method 1: Using Dimensionality Reduction (Recommended)

This method uses PCA or other dimensionality reduction results to calculate cluster relationships. It's computationally efficient and robust to noise.

```r
# Use first 50 PCA dimensions
StackVln(seurat_obj,
         features = c("CD3D", "CD8A", "CD4"),
         dendrogram_method = "dims",
         ndim = 50,
         reduction_for_tree = "pca")

# Use default 30 dimensions (ndim not specified)
StackVln(seurat_obj,
         features = c("CD3D", "CD8A", "CD4"),
         dendrogram_method = "dims")
```

**When to use:**
- Standard workflow for most analyses
- When you have performed PCA or other dimensionality reduction
- For robust clustering relationships across the full transcriptome

**Note:** If `ndim` is not specified, the function defaults to using dimensions 1:30 and displays a message.

#### Method 2: Using Plot Features Only

This method calculates the dendrogram based only on the genes you're plotting.
dendrogram_method = "features"

```r
StackVln(seurat_obj,
         features = c("CD3D", "CD8A", "CD4", "MS4A1", "CD14"),
         dendrogram_method = "features")
```

**When to use:**
- When you want cluster relationships specific to your marker genes
- For focused analysis on a particular gene set
- When your features are carefully selected markers

#### Method 3: Using All Variable Features

This method uses all variable features identified in your Seurat object.

```r
StackVln(seurat_obj,
         features = c("CD3D", "CD8A"),
         dendrogram_method = "all_variable")
```

**When to use:**
- When you want comprehensive gene-based clustering
- As an alternative to dimensionality reduction method
- When you don't have dimensionality reduction results

### Plot Size Customization

Control the dimensions of your plots for optimal visualization and publication quality.

#### Automatic Height Calculation

By default, plot height is automatically calculated based on the number of features:
- Formula: `calc_height = max(5, length(features) * 0.4 + 2)`
- Minimum: 5 inches
- Scales: 0.4 inches per feature + 2 inches base

Examples:
- 3 features → 5 inches (minimum)
- 10 features → 6 inches
- 20 features → 10 inches

#### Adjusting Plot Width

```r
# Wider plot for many clusters
StackVln(seurat_obj,
         features = c("CD3D", "CD8A"),
         plot_width = 15)
```

#### Controlling Dendrogram vs Violin Ratio

The `plot_heights` parameter controls the relative space allocated to dendrogram and violin plots:

```r
# Default: minimal dendrogram, focus on violins
StackVln(seurat_obj, features = genes, plot_heights = c(1, 9))

# Emphasize dendrogram
StackVln(seurat_obj, features = genes, plot_heights = c(2, 8))

# Larger dendrogram for detailed branch structure
StackVln(seurat_obj, features = genes, plot_heights = c(3, 7))

# Minimal dendrogram
StackVln(seurat_obj, features = genes, plot_heights = c(0.5, 9.5))
```

## Troubleshooting

### Cluster Names are Cut Off

**Problem**: Long cluster names are truncated at the bottom or left side of the saved plot.

**Solution**: Use the appropriate margin parameter:
```r
# If names are cut off at the bottom
StackVln(seurat_obj, 
         features = genes,
         x_label_bottom_margin = 2)  # Increase from default 1

# If the leftmost name is cut off on the left
StackVln(seurat_obj, 
         features = genes,
         x_label_left_margin = 2)  # Increase from default 1

# If both issues occur
StackVln(seurat_obj, 
         features = genes,
         x_label_bottom_margin = 2,
         x_label_left_margin = 2)
```

Try values between 1 and 3 depending on your label length.

### Plot Height Issues

**Problem**: The plot is too tall or too short.

**Solution**: The height is automatically calculated based on the number of features. For manual control, you can save the plot object and use `ggsave()` with custom dimensions:
```r
p <- StackVln(seurat_obj, features = genes)
ggsave("custom_plot.png", p, width = 10, height = 8, dpi = 300)
```

## Function Arguments

- `seurat_object`: Seurat object containing the assay data
- `features`: Character vector of gene names to plot
- `group.by`: Column name in meta.data for grouping (default: "seurat_clusters")
- `cluster_order`: Optional character vector to specify cluster order manually
- `assay`: Assay to use (default: NULL, automatically uses DefaultAssay())
- `color_low`: Color for low expression (default: "white")
- `color_high`: Color for high expression (default: "#BD2130")
- `plot_heights`: Numeric vector (length=2) for relative heights of dendrogram and violin (default: c(1, 9))
- `plot_width`: Width of the saved plot in inches (default: 10)
- `x_label_bottom_margin`: Bottom margin space for rotated x-axis labels in inches (default: 1). Increase if cluster names are cut off at the bottom. Range: 1-3 inches.
- `x_label_left_margin`: Left margin space for rotated x-axis labels in inches (default: 1). Increase if the leftmost cluster name is cut off on the left side. Range: 1-3 inches.
- `save_path`: Full path to save the file. Overrides save_dir
- `save_dir`: Directory to save the file using a default filename
- `tag`: Optional tag to add to the filename (default: NULL). When specified with save_dir, generates `{seurat_object}_{tag}_stack_vln.png` instead of `{seurat_object}_stack_vln.png`. Ignored when save_path is used.
- `ndim`: Number of dimensions for dendrogram (when method = "dims", default: NULL = 30)
- `dendrogram_method`: Calculation method: "features", "dims" (recommended), or "all_variable" (default: "features")
- `reduction_for_tree`: Reduction to use for "dims" method (default: "pca")

## Citation

If you use StackVln in your research, please cite:

```
Okumura, A. (2026). StackVln: Enhanced Visualization for Seurat.
R package version 0.0.0.9000.
https://github.com/AyumuOkumura/StackVln
```

## License

MIT License

## Issues & Contributions

Please report issues at: https://github.com/AyumuOkumura/StackVln/issues
Contributions are welcome via pull requests!

## Links

- seurat <https://satijalab.org/seurat/>
- scannpy stacked_violin <https://scanpy.readthedocs.io/en/stable/generated/scanpy.pl.stacked_violin.html>

## 🌍 Multilingual Overview
> ⚠️ **Note**: The following translations were generated by AI (Google Gemini). Please refer to the [English version](https://github.com/AyumuOkumura/StackVln/blob/master/README.md#bring-scanpy-style-stacked-violin-plots-to-seurat) for the most accurate technical specifications.

<details>
<summary>Click to expand translations (日本語 / 中文 / Deutsch / ...)
</summary>

#### 🇯🇵 日本語 
- 📊 **スタックバイオリンプロット**: Scanpy風の発現中央値で色付けされた積層型バイオリンプロットを簡単に描画。
- 🌳 **自動クラスタリング**: BuildClusterTree() のデンドログラムに基づき、クラスタ順序を自動で生物学的に整列。
- 🎨 **高いカスタマイズ性**: 色、サイズ、表示順序を柔軟に調整可能。
- 💾 **簡単保存: 高解像度** .png 出力機能を内蔵。

#### 🇨🇳 中文 (Simplified) 
- 📊 **堆叠小提琴图**: 绘制 Scanpy 风格的堆叠小提琴图 (Stacked Violin Plot)，并根据表达中位数填充颜色。
- 🌳 **自动聚类排序**: 利用 BuildClusterTree() 树状图自动调整聚类顺序。
- 🎨 **高度可定制**: 灵活调整颜色、尺寸和基因分组。
- 💾 **便捷导出**: 内置高分辨率 .png 保存功能。

#### 🇩🇪 Deutsch 
- 📊 **Gestapelte Violin-Plots**: Erstellen Sie kompakte, Scanpy-artige gestapelte Violin-Plots, gefärbt nach **Median-Expression**.
- 🌳 **Automatische Cluster-Sortierung**: Automatische Anordnung der Cluster basierend auf `BuildClusterTree()` Dendrogrammen.
- 🎨 **Anpassbar**: Farben, Größen und Cluster-Reihenfolgen sind flexibel einstellbar.
- 💾 **Einfacher Export**: Integrierte Funktion zum Speichern hochauflösender `.png`-Dateien.

#### 🇪🇸 Español 
- 📊 **Gráficos de Violín Apilados**: Cree gráficos compactos estilo Scanpy, coloreados según la **expresión mediana**.
- 🌳 **Agrupamiento Automático**: Reordenación automática de clústeres basada en dendrogramas de `BuildClusterTree()`.
- 🎨 **Personalizable**: Colores, tamaños y orden de clústeres totalmente ajustables.
- 💾 **Exportación Fácil**: Guardado integrado de imágenes `.png` de alta resolución.

#### 🇵🇹 Português 
- 📊 **Violin Plots Empilhados**: Crie plots compactos estilo Scanpy, coloridos pela **expressão mediana**.
- 🌳 **Clusterização Automática**: Reordenação automática de clusters baseada em dendrogramas do `BuildClusterTree()`.
- 🎨 **Personalizável**: Cores, tamanhos e ordens de clusters totalmente ajustáveis.
- 💾 **Exportação Fácil**: Salvamento integrado em `.png` de alta resolução.

#### 🇮🇩 Bahasa Indonesia 
- 📊 **Stacked Violin Plots**: Buat plot biola bertumpuk gaya Scanpy yang ringkas, diwarnai berdasarkan **ekspresi median**.
- 🌳 **Klasterisasi Otomatis**: Pengurutan ulang klaster secara otomatis berdasarkan dendrogram `BuildClusterTree()`.
- 🎨 **Dapat Disesuaikan**: Warna, ukuran, dan urutan klaster yang fleksibel.
- 💾 **Ekspor Mudah**: Fitur penyimpanan `.png` resolusi tinggi bawaan.

#### 🇫🇷 Français 
- 📊 **Tracés en Violon Empilés**: Créez des "Stacked Violin Plots" compacts de style Scanpy, colorés par **expression médiane**.
- 🌳 **Clustering Automatique**: Réorganisation automatique des clusters basée sur les dendrogrammes de `BuildClusterTree()`.
- 🎨 **Personnalisable**: Couleurs, tailles et ordres des clusters entièrement ajustables.
- 💾 **Export Facile**: Fonction intégrée pour sauvegarder en `.png` haute résolution.

#### 🇰🇷 한국어 
- 📊 **스택 바이올린 플롯**: **중앙값 발현(median expression)**으로 채색된 컴팩트한 Scanpy 스타일의 스택 바이올린 플롯을 생성합니다.
- 🌳 **자동 클러스터링**: `BuildClusterTree()` 덴드로그램을 기반으로 클러스터 순서를 자동으로 재정렬합니다.
- 🎨 **사용자 정의 가능**: 색상, 크기 및 클러스터 순서를 완벽하게 조정할 수 있습니다.
- 💾 **간편한 내보내기**: 고해상도 `.png` 저장 기능이 내장되어 있습니다.

#### 🇷🇺 Русский 
- 📊 **Стекированные Скрипичные Диаграммы**: Создавайте компактные диаграммы в стиле Scanpy, окрашенные по **медианной экспрессии**.
- 🌳 **Автоматическая Кластеризация**: Автоматическое переупорядочивание кластеров на основе дендрограмм `BuildClusterTree()`.
- 🎨 **Настраиваемость**: Полностью настраиваемые цвета, размеры и порядок кластеров.
- 💾 **Легкий Экспорт**: Встроенная функция сохранения в высоком разрешении `.png`.

#### 🇮🇱 עברית (Hebrew)
- 📊 **תרשימי כינור מוערמים**: יצירת תרשימי כינור (Violin Plots) מוערמים וקומפקטיים בסגנון Scanpy, הצבועים לפי **חציון הביטוי**.
- 🌳 **אשכול אוטומטי**: סידור מחדש אוטומטי של אשכולות (Clusters) על בסיס דנדרוגרמות `BuildClusterTree()`.
- 🎨 **ניתן להתאמה אישית**: צבעים, גדלים וסדרי אשכולות ניתנים להתאמה מלאה.
- 💾 **ייצוא קל**: שמירה מובנית של קבצי `.png` ברזולוציה גבוהה.

#### 🇮🇷 فارسی (Persian)
- 📊 **نمودارهای ویولن انباشته**: ایجاد نمودارهای ویولن (Violin Plots) فشرده به سبک Scanpy، رنگ‌آمیزی شده بر اساس **میانه بیان**.
- 🌳 **خوشه‌بندی خودکار**: مرتب‌سازی مجدد خودکار خوشه‌ها بر اساس دندروگرام‌های `BuildClusterTree()`.
- 🎨 **قابل تنظیم**: رنگ‌ها، اندازه‌ها و ترتیب خوشه‌ها کاملاً قابل شخصی‌سازی هستند.
- 💾 **خروجی آسان**: قابلیت ذخیره داخلی فایل‌های `.png` با وضوح بالا.

#### 🇹🇭 ไทย (Thai)
- 📊 **แผนภูมิไวโอลินแบบซ้อน**: สร้างกราฟไวโอลินแบบซ้อนที่กะทัดรัดในสไตล์ Scanpy โดยระบายสีตาม **ค่ามัธยฐานของการแสดงออก**
- 🌳 **การจัดกลุ่มอัตโนมัติ**: จัดเรียงลำดับคลัสเตอร์ใหม่โดยอัตโนมัติตามเดนโดรแกรมของ `BuildClusterTree()`
- 🎨 **ปรับแต่งได้เต็มที่**: สามารถปรับแต่งสี ขนาด และลำดับคลัสเตอร์ได้อย่างอิสระ
- 💾 **ส่งออกไฟล์ง่าย**: มีฟังก์ชันบันทึกไฟล์ `.png` ความละเอียดสูงในตัว

#### 🇸🇪 Svenska (Swedish)
- 📊 **Staplade Violinplots**: Skapa kompakta staplade violinplots i Scanpy-stil, färglagda efter **medianuttryck** (median expression).
- 🌳 **Automatisk Klustring**: Automatisk omordning av kluster baserat på `BuildClusterTree()`-dendrogram.
- 🎨 **Anpassningsbar**: Färger, storlekar och klusterordning är helt justerbara.
- 💾 **Enkel Export**: Inbyggd funktion för att spara högupplösta `.png`-filer.

#### 🇩🇰 Dansk (Danish)
- 📊 **Stakede Violinplots**: Opret kompakte, stakede violinplots i Scanpy-stil, farvet efter **medianudtryk**.
- 🌳 **Automatisk Klyngedannelse**: Automatisk sortering af klynger baseret på `BuildClusterTree()`-dendrogrammer.
- 🎨 **Kan Tilpasses**: Farver, størrelser og klyngeorden kan justeres fuldt ud.
- 💾 **Nem Eksport**: Indbygget funktion til lagring af højopløselige `.png`-filer.

#### 🇮🇹 Italiano (Italian)
- 📊 **Violin Plot Impilati**: Crea grafici a violino impilati compatti in stile Scanpy, colorati in base all'**espressione mediana**.
- 🌳 **Clustering Automatico**: Riordinamento automatico dei cluster basato sui dendrogrammi di `BuildClusterTree()`.
- 🎨 **Personalizzabile**: Colori, dimensioni e ordine dei cluster completamente regolabili.
- 💾 **Esportazione Facile**: Funzione integrata per il salvataggio di immagini `.png` ad alta risoluzione.

#### 🇸🇦 العربية (Arabic)
- 📊 **مخططات الكمان المكدسة**: إنشاء مخططات كمان (Violin Plots) مكدسة ومدمجة بأسلوب Scanpy، ملونة حسب **متوسط التعبير** (median expression).
- 🌳 **تجميع تلقائي**: إعادة ترتيب المجموعات (Clusters) تلقائيًا بناءً على المخططات الشجرية لـ `BuildClusterTree()`.
- 🎨 **قابل للتخصيص**: الألوان والأحجام وترتيب المجموعات قابلة للتعديل بالكامل.
- 💾 **تصدير سهل**: ميزة مدمجة لحفظ ملفات `.png` عالية الدقة.
#### 🇮🇳 हिन्दी (Hindi)
- 📊 **स्टैक्ड वायलिन प्लॉट्स**: **मीडियन एक्सप्रेशन** (median expression) द्वारा रंगे हुए कॉम्पैक्ट, Scanpy-शैली के स्टैक्ड वायलिन प्लॉट्स बनाएं।
- 🌳 **स्वचालित क्लस्टरिंग**: `BuildClusterTree()` डेंड्रोग्राम के आधार पर क्लस्टरों को स्वचालित रूप से पुन: व्यवस्थित करें।
- 🎨 **अनुकूलन योग्य**: रंग, आकार और क्लस्टर क्रम पूरी तरह से समायोज्य हैं।
- 💾 **आसान निर्यात**: उच्च-रिज़ॉल्यूशन `.png` सेविंग की अंतर्निहित सुविधा।

#### 🇹🇷 Türkçe (Turkish)
- 📊 **Yığılmış Keman Grafikleri**: **Medyan ekspresyonuna** göre renklendirilmiş, kompakt, Scanpy tarzı yığılmış keman grafikleri (Stacked Violin Plots) oluşturun.
- 🌳 **Otomatik Kümeleme**: `BuildClusterTree()` dendrogramlarına dayalı olarak kümeleri otomatik yeniden sıralayın.
- 🎨 **Özelleştirilebilir**: Renkler, boyutlar ve küme sıraları tamamen ayarlanabilir.
- 💾 **Kolay Dışa Aktarım**: Dahili yüksek çözünürlüklü `.png` kaydetme özelliği.

#### 🇵🇱 Polski (Polish)
- 📊 **Ułożone Wykresy Skrzypcowe**: Twórz kompaktowe wykresy skrzypcowe (Stacked Violin Plots) w stylu Scanpy, kolorowane według **mediany ekspresji**.
- 🌳 **Automatyczne Klastrowanie**: Automatyczna zmiana kolejności klastrów w oparciu o dendrogramy `BuildClusterTree()`.
- 🎨 **Konfigurowalne**: Kolory, rozmiary i kolejność klastrów są w pełni regulowane.
- 💾 **Łatwy Eksport**: Wbudowana funkcja zapisywania plików `.png` w wysokiej rozdzielczości.

#### 🇻🇳 Tiếng Việt (Vietnamese)
- 📊 **Biểu đồ Violin Xếp chồng**: Tạo các biểu đồ violin xếp chồng (Stacked Violin Plots) gọn gàng theo phong cách Scanpy, được tô màu theo **biểu hiện trung vị** (median expression).
- 🌳 **Phân cụm Tự động**: Tự động sắp xếp lại các cụm dựa trên biểu đồ dendrogram `BuildClusterTree()`.
- 🎨 **Có thể Tùy chỉnh**: Màu sắc, kích thước và thứ tự cụm hoàn toàn có thể điều chỉnh.
- 💾 **Xuất Dễ dàng**: Tích hợp tính năng lưu file `.png` độ phân giải cao.
</details>

---
#### 🏷️ Keywords / Tags
##### `Single-cell Visualization` `Stacked ViolinPlot`
---
