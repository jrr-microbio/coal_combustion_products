Qiime2 methods

mamba activate qiime2-amplicon-2024.5

#Following the standard moving pictures tutorial: https://docs.qiime2.org/2024.5/tutorials/moving-pictures/

qiime tools import \
  --type 'SampleData[PairedEndSequencesWithQuality]' \
  --input-path ./qiime2_manifest_final_87samples.csv \
  --output-path paired-end-demux.qza \
  --input-format PairedEndFastqManifestPhred33

qiime demux summarize \
  --i-data paired-end-demux.qza \
  --o-visualization demux.qzv

#These are quite high quality just straight from Azenta. I'm not going to trim these. Now going to denoise with dada2 using the consensus method for chimera detection.
#  --p-trunc-len-f 0 to prevent trimming because solid qual. 
#  --p-trunc-len-r 0 to prevent trimming because solid qual. 
#  --p-chimera-method consensus because better for most types of data per: https://forum.qiime2.org/t/dada2-chimera-filtering-and-beyond/8685/2

qiime dada2 denoise-paired \
  --i-demultiplexed-seqs paired-end-demux.qza \
  --p-trunc-len-f 0 \
  --p-trunc-len-r 0 \
  --p-n-threads 10 \
  --p-trunc-q 2 \
  --p-chimera-method consensus \
  --o-representative-sequences rep-seqs-dada2.qza \
  --o-table table-dada2.qza \
  --o-denoising-stats stats-dada2.qza

qiime metadata tabulate \
  --m-input-file stats-dada2.qza \
  --o-visualization stats-dada2.qzv

#All good here, looks like ~10% is removed by denoising. This seems reasonable. Straight out of dada2, these are now technically ASVs / exact sequence variants (https://forum.qiime2.org/t/to-cluster-or-not-to-cluster/10022/6)

#Now going to classify these. Downloaded taxonomic classifiers (Silva 138 and GTDB r220) from: https://resources.qiime2.org/

#SILVA 138
qiime feature-classifier classify-sklearn \
  --i-classifier ./taxonomic_classifiers/silva-138-99-nb-classifier.qza \
  --i-reads rep-seqs-dada2.qza \
  --o-classification SILVA_taxonomy.qza

qiime taxa barplot \
  --i-table table-dada2.qza \
  --i-taxonomy SILVA_taxonomy.qza \
  --m-metadata-file qiime2_sample_metadata_final_87samples.txt \
  --o-visualization SILVA_taxa-bar-plots.qzv

qiime metadata tabulate \
  --m-input-file SILVA_taxonomy.qza \
  --o-visualization SILVA_taxonomy.qzv

#Apparently SILVA is already taking in GTDB taxonomic info - but i'm just running this to be safe becuase i dont know how often silva gets updated with gtdb releases. I'll use both.

#GTDB r220
  qiime feature-classifier classify-sklearn \
  --i-classifier ./taxonomic_classifiers/gtdb_classifier_r220.qza \
  --i-reads rep-seqs-dada2.qza \
  --o-classification GTDB_taxonomy.qza

  qiime taxa barplot \
  --i-table table-dada2.qza \
  --i-taxonomy GTDB_taxonomy.qza \
  --m-metadata-file qiime2_sample_metadata_final_87samples.txt \
  --o-visualization GTDB_taxa-bar-plots.qzv

  qiime metadata tabulate \
  --m-input-file GTDB_taxonomy.qza \
  --o-visualization GTDB_taxonomy.qzv

#These are now my ASVs/ESVs classified. Exporting these out:

qiime tools export \
  --input-path SILVA_taxonomy.qza \
  --output-path SILVA_exported-otu-table

qiime tools export \
  --input-path GTDB_taxonomy.qza \
  --output-path GTDB_exported-otu-table

#Now going to export the OTU table and merge it to the taxonomy calls externally in R. Need to export to biom then convert to table:

qiime tools export \
  --input-path table-dada2.qza \
  --output-path exported-feature-table

  biom convert --to-tsv -i ./exported-feature-table/feature-table.biom -o final_ASV_table.tsv
