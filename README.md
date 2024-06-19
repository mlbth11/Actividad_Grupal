# Actividad_Grupal
Github: Crear un repositorio Github. Control de versiones
# Estudiantes: Masache Debora y Enriquez Martha
## Nota: En la carpeta "Proyecto1" Se encuentran los resultados de cada etapa del proyecto
## Nota2: Para visualizar los archivos .qzv y .qza se utiliza: https://old-view.qiime2.org

Este tutorial está diseñado para servir para dos propósitos. En primer lugar, ilustra los pasos de procesamiento inicial del análisis de lectura de extremo emparejado, hasta el punto en que los pasos de análisis son idénticos al análisis de lectura de extremo único. Esto incluye los pasos de importación, desmultiplexación y desnado, y da como resultado una tabla de características y las secuencias de características asociadas. En segundo lugar, este pretende ser un ejercicio autoguiado que podría ejecutarse después del tutorial de imágenes en movimiento para obtener más experiencia con QIIME 2. Para este ejercicio, proporcionamos algunas preguntas que se pueden utilizar para guiar su análisis, pero no proporcionamos comandos que le permitan abordar cada una. En su lugar, debes aplicar los comandos que aprendiste en el tutorial de imágenes en movimiento.

# 1. Obtencion de datos

## Descarga de metadatos

```bash
wget -O "sample-metadata.tsv" "https://data.qiime2.org/2023.5/tutorials/atacama-soils/sample-metadata.tsv"
```

## Descarga de secuencias

```bash
mkdir emp-paired-end-sequences # Creamos el directorio para las secuencias

wget -O "emp-paired-end-sequences/forward.fastq.gz" "https://data.qiime2.org/2023.5/tutorials/atacama-soils/10p/forward.fastq.gz"
wget -O "emp-paired-end-sequences/reverse.fastq.gz" "https://data.qiime2.org/2023.5/tutorials/atacama-soils/10p/reverse.fastq.gz"
wget -O "emp-paired-end-sequences/barcodes.fastq.gz" "https://data.qiime2.org/2023.5/tutorials/atacama-soils/10p/barcodes.fastq.gz"
```

# 2. Creacion del artefacto de qiime2

```bash
qiime tools import --type EMPPairedEndSequences --input-path emp-paired-end-sequences --output-path emp-paired-end-sequences.qza
```

# 3. Demultiplex de lecturas

```bash
qiime demux emp-paired --m-barcodes-file sample-metadata.tsv --m-barcodes-column barcode-sequence --p-rev-comp-mapping-barcodes --i-seqs emp-paired-end-sequences.qza --o-per-sample-sequences demux-full.qza --o-error-correction-details demux-details.qza
```

# 4. Creacion de la submuestra

```bash
qiime demux subsample-paired --i-sequences demux-full.qza --p-fraction 0.3 --o-subsampled-sequences demux-subsample.qza
```

## Creación de la visualización

```bash
qiime demux summarize --i-data demux-subsample.qza --o-visualization demux-subsample.qzv
```

# 5. Filtracion por numero de lecturas

```bash
qiime tools export --input-path demux-subsample.qzv --output-path ./demux-subsample/

qiime demux filter-samples --i-demux demux-subsample.qza --m-metadata-file ./demux-subsample/per-sample-fastq-counts.tsv --p-where 'CAST([forward sequence count] AS INT) > 100' --o-filtered-demux demux.qza
```
# 6. Control de calidad y "denoising"

```bash
qiime dada2 denoise-paired --i-demultiplexed-seqs demux.qza --p-trim-left-f 13 --p-trim-left-r 13 --p-trunc-len-f 150 --p-trunc-len-r 150 --o-table table.qza --o-representative-sequences rep-seqs.qza --o-denoising-stats denoising-stats.qza
```

## Generación de resúmenes correspondientes

```bash
qiime feature-table summarize --i-table table.qza --o-visualization table.qzv --m-sample-metadata-file sample-metadata.tsv

qiime feature-table tabulate-seqs --i-data rep-seqs.qza --o-visualization rep-seqs.qzv

qiime metadata tabulate --m-input-file denoising-stats.qza --o-visualization denoising-stats.qzv
```

# 7. ⁠Generacion del arbol para el analisis de diversidad filogenetica

```bash
qiime phylogeny align-to-tree-mafft-fasttree --i-sequences rep-seqs.qza --o-alignment aligned-rep-seqs.qza --o-masked-alignment masked-aligned-rep-seqs.qza --o-tree unrooted-tree.qza --o-rooted-tree rooted-tree.qza
```

# 8. Calculo de diversidad alfa y beta

```bash
qiime diversity core-metrics-phylogenetic --i-phylogeny rooted-tree.qza --i-table table.qza --p-sampling-depth 1103 --m-metadata-file sample-metadata.tsv --output-dir core-metrics-results
```

## Agrupación de datos para Alfa-diversidad

```bash
mkdir -p core-metrics-results/Alfa-diversity # Creo una carpeta para almacenar la información

qiime diversity alpha-group-significance --i-alpha-diversity core-metrics-results/faith_pd_vector.qza --m-metadata-file sample-metadata.tsv --o-visualization core-metrics-results/Alfa-diversity/faith-pd-group-significance.qzv

qiime diversity alpha-group-significance --i-alpha-diversity core-metrics-results/shannon_vector.qza --m-metadata-file sample-metadata.tsv --o-visualization core-metrics-results/Alfa-diversity/shannon-group-significance.qzv

qiime diversity alpha-group-significance --i-alpha-diversity core-metrics-results/observed_features_vector.qza --m-metadata-file sample-metadata.tsv --o-visualization core-metrics-results/Alfa-diversity/observed_features-group-significance.qzv

qiime diversity alpha-group-significance --i-alpha-diversity core-metrics-results/evenness_vector.qza --m-metadata-file sample-metadata.tsv --o-visualization core-metrics-results/Alfa-diversity/evenness-group-significance.qzv
```

## Análisis PERMANOVA de diversidad beta

```bash
mkdir -p core-metrics-results/Beta-diversity # Creo una carpeta para almacenar la información

# Introduzco todas las columnas "Categóricas" del fichero de metadato en un fichero llamado column-names.txt para ejecutar todas las columnas posibles en una sóla ejecución con cada uno de los parámetros:

for i in $(cat core-metrics-results/column-names.txt); do qiime diversity beta-group-significance --i-distance-matrix core-metrics-results/unweighted_unifrac_distance_matrix.qza --m-metadata-file sample-metadata.tsv --m-metadata-column $i --o-visualization core-metrics-results/Beta-diversity/unweighted-unifrac-"$i"-significance.qzv --p-pairwise; done

for i in $(cat core-metrics-results/column-names.txt); do qiime diversity beta-group-significance --i-distance-matrix core-metrics-results/weighted_unifrac_distance_matrix.qza --m-metadata-file sample-metadata.tsv --m-metadata-column $i --o-visualization core-metrics-results/Beta-diversity/weighted-unifrac-"$i"-significance.qzv --p-pairwise; done

for i in $(cat core-metrics-results/column-names.txt); do qiime diversity beta-group-significance --i-distance-matrix core-metrics-results/bray_curtis_distance_matrix.qza --m-metadata-file sample-metadata.tsv --m-metadata-column $i --o-visualization core-metrics-results/Beta-diversity/bray_curtis-"$i"-significance.qzv --p-pairwise; done

for i in $(cat core-metrics-results/column-names.txt); do qiime diversity beta-group-significance --i-distance-matrix core-metrics-results/jaccard_distance_matrix.qza --m-metadata-file sample-metadata.tsv --m-metadata-column $i --o-visualization core-metrics-results/Beta-diversity/jaccard-"$i"-significance.qzv --p-pairwise; done
```

## Gráfico PCoA

```bash
#Introduzco todas las columnas "Numéricas" del fichero de metadato en un fichero llamado column-names-numeric.txt para ejecutar todas las columnas posibles, también mencionar que se pueden hacer combinaciones de varias columnas a la vez "columna1 + columna2". Hay que tener cuidado con las columnas en las que hay valores faltantes.

for i in $(cat core-metrics-results/column-names-numeric.txt); do qiime emperor plot --i-pcoa core-metrics-results/unweighted_unifrac_pcoa_results.qza --m-metadata-file sample-metadata.tsv --p-custom-axes $i --o-visualization core-metrics-results/Beta-diversity/unweighted-unifrac-emperor-"$i".qzv; done

bash core-metrics-results/Beta-diversity/unweighted-emperor.bash # Script con la ejecución de individuales y combinadas

for i in $(cat core-metrics-results/column-names-numeric.txt); do qiime emperor plot --i-pcoa core-metrics-results/weighted_unifrac_pcoa_results.qza --m-metadata-file sample-metadata.tsv --p-custom-axes $i --o-visualization core-metrics-results/Beta-diversity/weighted-unifrac-emperor-"$i".qzv; done

bash core-metrics-results/Beta-diversity/weighted-emperor.bash # Script con la ejecución de individuales y combinadas

for i in $(cat core-metrics-results/column-names-numeric.txt); do qiime emperor plot --i-pcoa core-metrics-results/bray_curtis_pcoa_results.qza --m-metadata-file sample-metadata.tsv --p-custom-axes $i --o-visualization core-metrics-results/Beta-diversity/bray_curtis-emperor-"$i".qzv; done

bash core-metrics-results/Beta-diversity/bray_curtis-emperor.bash # Script con la ejecución de individuales y combinadas

for i in $(cat core-metrics-results/column-names-numeric.txt); do qiime emperor plot --i-pcoa core-metrics-results/jaccard_pcoa_results.qza --m-metadata-file sample-metadata.tsv --p-custom-axes $i --o-visualization core-metrics-results/Beta-diversity/jaccard-emperor-"$i".qzv; done

bash core-metrics-results/Beta-diversity/jaccard-emperor.bash # Script con la ejecución de individuales y combinadas
```

# 9. Analisis taxonomico

## Clasificación taxonómica

```bash
qiime feature-classifier classify-sklearn --i-classifier gg-13-8-99-515-806-nb-classifier.qza --i-reads rep-seqs.qza --o-classification taxonomy.qza
```

## Tabla de taxonomía

```bash
qiime metadata tabulate --m-input-file taxonomy.qza --o-visualization taxonomy.qzv

qiime taxa barplot --i-table table.qza --i-taxonomy taxonomy.qza --m-metadata-file metadata.tsv --o-visualization taxa-bar-plots.qzv
```

# 10. Analisis diferencial de abundancia microbiana

## Preparación de la tabla

```bash
qiime composition add-pseudocount --i-table table.qza --o-composition-table comp-table.qza
```

## AMCO

```bash
for i in $(cat core-metrics-results/column-names.txt); do qiime composition ancom --i-table comp-table.qza --m-metadata-file sample-metadata.tsv --m-metadata-column $i --o-visualization AMCO/ancom-"$i".qzv; done
```

## Agregar taxonomía

```bash
qiime taxa collapse --i-table table.qza --i-taxonomy taxonomy.qza --p-level 6 --o-collapsed-table table-l6.qza

qiime composition add-pseudocount --i-table table-l6.qza --o-composition-table comp-table-l6.qza

for i in $(cat core-metrics-results/column-names.txt); do qiime composition ancom --i-table comp-table-l6.qza --m-metadata-file sample-metadata.tsv --m-metadata-column $i --o-visualization AMCO/l6-ancom-"$i".qzv; done
```
