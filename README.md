# wikIR
A python tool for building a large scale Wikipedia-based Information Retrieval dataset

# Requirements
  * Python 3.6+
  * If working with high ressource language (e.g. english) you need **20GB of RAM** to process the json file returned by  [Wikiextractor](https://github.com/attardi/wikiextractor)

# Installation

```bash
git clone --recurse-submodules https://github.com/getalp/wikIR.git
```

# Usage

  * Download and extract a XML wikipedia dump file from [here](https://dumps.wikimedia.org/backup-index.html) 
  * Use [Wikiextractor](https://github.com/attardi/wikiextractor) to get the text of the wikipedia pages in a signle json file, for example : 
```bash
python wikiextractor/WikiExtractor.py input --output - --bytes 100G --links --quiet --json > output.json
```
Where input is the XML wikipedia dump file and output is the output in json format

  * Call our script
```
python build_wikIR.py [-in,--input] [-o,--output_dir] [-q,--nb_queries] [-t,--train_part] [-v,--validation_part] [-test,--test_part] [-xml,--xml_output] [-both,--both_output] [-r,--random_seed] 

arguments : 

    [-in,--input]                The json file produced by wikiextractor
    
    [-o,--output_dir]            Directory where the collection will be stored

optional argument:

    [-q,--nb_queries]            Number of queries desired
                                 Default value -1 : all wikipedia articles will be used to produce a query

    [-t,--train_part]            Proportion of queries to use in the training process 
                                 Default value 0.8 
                                 
    [-v,--validation_part]       Proportion of queries to use in the validation process 
                                 Default value 0.1
    
    [-test,--test_part]          Proportion of queries to use in the test process 
                                 Default value 0.1
    
    [-xml,--xml_output]          If used, the documents and queries will be saved in xml format
                                 compatible with Terrier IR Platform
                                 If not used , the documents and queries will be saved in json format
                                 
    [-both,--both_output]        If used, the documents and queries will be saved in xml format
                                 and in json format
                                 
    [-r,--random_seed]           Random seed
                                 Default value 27355
    
output : our tool creates the 7 following files in the output directory

    documents.json             Documents
    
    train.queries.json         Train queries
    validation.queries.json    Validation queries
    test.queries.json          Test queries
    
    train.qrel                 Train relevance judgments
    validation.qrel            Validation relevance judgments
    test.qrel                  Test relevance judgments
    
```

# Quick Example (take a few minutes)

Clone our repository

```bash
git clone https://github.com/getalp/wikIR.git
```

Download the swahili wikipedia dump from 01/03/2019 
```bash
wget https://dumps.wikimedia.org/swwiki/20190301/swwiki-20190301-pages-articles-multistream.xml.bz2
```

Extract the file
```bash
bzip2 -dk swwiki-20190301-pages-articles-multistream.xml.bz2
```

Use Wikiextractor (ignore the WARNING: Template errors in article)
```bash
python wikiextractor/WikiExtractor.py swwiki-20190301-pages-articles-multistream.xml --output - --bytes 100G --links --quiet --json > wiki.json
```

Use wikIR builder
```bash
python build_wikIR.py -in wiki.json -o wikIR -t 0.8 -v 0.1 -test 0.1
```

# Example on English Wikipedia (take several hours)

Clone our repository

```bash
git clone https://github.com/getalp/wikIR.git
```
Download the english wikipedia dump from 01/03/2019 (:warning: 16.9 GB file)
```bash
wget https://dumps.wikimedia.org/enwiki/20190301/enwiki-20190301-pages-articles-multistream.xml.bz2
```

Extract the file (:warning: produces a 71.3 GB file)
```bash
bzip2 -dk enwiki-20190301-pages-articles-multistream.xml.bz2
```

Use Wikiextractor (:warning: produces a 17.2 GB file ; ignore the WARNING: Template errors in article)
```bash
python wikiextractor/WikiExtractor.py enwiki-20190301-pages-articles-multistream.xml --output - --bytes 100G --links --quiet --json > wiki.json
```

Use wikIR builder (:warning: produces a 5.9 GB directory)
```bash
python build_wikIR.py -in wiki.json -o wikIR
```

### :warning: **Do not forget to delete the dowloaded and intermediary files** :warning:

```bash
rm enwiki-20190301-pages-articles-multistream.xml.bz2
rm enwiki-20190301-pages-articles-multistream.xml
rm wiki.json
```

# Reproducibility

To reproduce the same dataset(s) we used in our experiment just call


```bash
./reproduce.sh COLLECTION_PATH
```

COLLECTION_PATH is the directory where the dataset will be stored

# Details
  * Our script takes **≈30 minutes** to build the collection on an Intel(R) Xeon(R) CPU E5-2623 v4 @ 2.60GHz
  * **≈ 5.8M queries** are extracted from the english wikipedia dump of 01/03/2019
  * Right now, our tokenizer was mainly designed for english and does not work on non-latin alphabets
  * We delete all non alphanumeric characters
  * All tokens are lowercased 
  * The data construction process is similar to [1] and [2] :
    * Only the first 200 words of each article is used to build the documents
    * The first sentence of each article is used to build the queries
    * We assign a **relevance of 2** if the query and document were extracted from the **same article**
    * We assign a **relevance of 1** if there is a **link from the article of the query to the article of the document**
    * We assign a **relevance of 0** to other documents


# Statistics of the wikIR english collection

| #Documents  | #Queries | #rels | Avg rel2 | Avg rel1 |
| :-: | :-: | :-: | :-: | :-: |
| 5.8M  | 5.8M | 52.2M | 1 | 8 |

There is as much queries as documents.

Each query is associated with only one document of relevance = 2 

On average each query has 8 documents of relevance = 1


# Using [Terrier IR Platform](http://terrier.org/) on wikIR

### :warning: **Do not forget to use the -xml or -both option when calling build_wikIR.py ** :warning:

## Indexation

Indexing wikIR with [Terrier](http://terrier.org/) (**30 minutes** on an Intel(R) Xeon(R) CPU E5-2623 v4 @ 2.60GHz)
```bash
cd TERRIER_PATH
bin/trec_setup.sh WIKIR_PATH/documents.xml
bin/terrier batchindexing -Dtermpipelines=Stopwords,PorterStemmer
```
## Retireval on validation with BM25 (**2 minutes** to evaluate 580 queries)

```bash
bin/terrier batchretrieve -Dtrec.model=BM25 -Dtrec.topics=WIKIR_PATH/validation.queries.xml
```
## Evaluate BM25 Retireval on validation

```bash
bin/terrier batchevaluate -Dtrec.qrels=WIKIR_PATH/validation.qrel
mv var/results/*.res WIKIR_PATH/terrier.validation.res 
```

## Terrier BM25 results on English Wikipeadia dump of 01/03/2019 

||MAP| NDCG | P@5 | NDCG@10 | Recall |
| :-: | :-: | :-: | :-: | :-: | :-: |
|validation (580 queries) | 31.45 | 57.04 | 25.02 | 56.17  | 48.62  |
| test (580 queries) | 31.81  | 57.60 | 25.59 | 56.82 | 48.96 |

:warning: These are not the results on the entire wikipedia dump, we used default parameter values of our script:warning:

Results were computed with [pytrec_eval](https://github.com/cvangysel/pytrec_eval) [3]

Hyperparameter values were optimized on the NDCG of the validation set (b = 0.6 ; k1 = 1.2 ; k3 = 8)

*****

[1] Shota Sasaki, Shuo Sun, Shigehiko Schamoni, Kevin Duh, and Kentaro Inui. 2018. Cross-lingual learning-to-rank with shared representations

[2] Shigehiko Schamoni, Felix Hieber, Artem Sokolov, and Stefan Riezler. 2014. Learning translational and knowledge-based similarities from relevance rankings for cross-language retrieval.

[3] Christophe Van Gysel and Maarten de Rijke. 2018. Pytrec_eval: An ExtremelyFast Python Interface to trec_eval.
