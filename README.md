<div align="center">    
  
  # Faster Learned Sparse Retrieval with Guided Traversal
</div>

This is the code and data repository for the paper **Faster Learned Sparse Retrieval with Guided Traversal** by Antonio Mallia, Joel Mackenzie, Torsten Suel, and Nicola Tonellotto. 

## Reference
```
@inproceedings{MMST2022,
  author    = {Antonio Mallia and Joel Mackenzie and Torsten Suel and Nicola Tonellotto},
  title     = {Faster Learned Sparse Retrieval with Guided Traversal},
  booktitle = {Proceedings of the 45th International ACM SIGIR Conference on Research and Development in Information Retrieval},
  year      = {2022},
}
```

## Installing the Codebase
The code for the dual score system is located in a [branch of the PISA system.](https://github.com/pisa-engine/pisa/tree/dual-score)

```
git clone --single-branch --branch=dual-score https://github.com/pisa-engine/pisa/
```

Then, follow the [build instructions](https://pisa.readthedocs.io/en/latest/getting_started.html#building-the-code) for the standard PISA system:
```
cd pisa
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j 8
```

## Getting the data
We provide a pre-built [CIFF](https://github.com/osirrc/ciff) file for ease of experimentation. The CIFF file already has the index pre-processing applied; the index is
expanded with [DocTTTTTQuery](https://github.com/castorini/docTTTTTquery), and has the frequency information as packed unsigned 32-bit integers, 16-bits allocated to the quantized BM25
score, and 16-bits allocated to the DeepImpact score. The index has been reordered with BP approach using the [faster-graph-bisection](https://github.com/mpetri/faster-graph-bisection) tool.

The data is available on [AARNET CloudStor](https://cloudstor.aarnet.edu.au/plus/s/6kNt9qXoXVK7SDy).

```
wget "https://cloudstor.aarnet.edu.au/plus/s/6kNt9qXoXVK7SDy/download" -O data.zip
```

Download the zip file and extract it -- it will extract into the `guided-traversal/` directory.

MD5 checksums for the data are as follows:
```
b5b094595d1e6098450a0125ba131242  dev.pisa
22809519a1a256b213777d417f39ed24  dl19.pisa
049f68bfa07d2126a34c8b8b2720242d  dl20.pisa
67d6f255a11f07c80eed392abc7b937b  dual.ciff
2f4be390198da108f6845c822e5ada14  qrels.dl19-passage.txt
0355ccee7509ac0463e8278186cdd8d1  qrels.dl20-passage.txt
38a80559a561707ac2ec0f150ecd1e8a  qrels.msmarco-passage.dev-subset.txt
```


## Building the index

1. Decode the CIFF file using [PISA's CIFF tool](https://github.com/pisa-engine/ciff/)
```
git clone https://github.com/pisa-engine/ciff/ pisa-ciff
cd pisa-ciff
cargo build --release
cd ..
./pisa-ciff/target/release/ciff2pisa --ciff-file guided-traversal/dual.ciff --output dual
```

2. Compress the inverted index
```
./pisa/build/bin/compress_inverted_index -c dual -e block_simdbp -o m_dual.block_simdbp.idx
``` 

3. Build the WAND upper-bound data
```
./pisa/create_wand_data --collection dual --scorer dual -b 40 -o m_dual.fixed-40.bmw
```

4. Build the term lexicon and document map
```
./pisa/build/bin/lexicon build dual.terms m_dual.termlex
./pisa/build/bin/lexicon build dual.documents m_dual.doclex
```

## Running Queries

Assuming the queries were downloaded to `guided-traversal/` then you can generate runs like:
```
./pisa/build/bin/evaluate_queries -e block_simdbp \
                                  -i m_dual.block_simdbp.idx \
                                  -w m_dual.fixed-40.bmw \
                                   --scorer dual \
                                   -k 1000  \
                                   -a maxscore  \
                                   --documents m_dual.doclex \
                                   --terms m_dual.termlex \
                                   -q guided-traversal/dl19.pisa \
                                   --interpolation-factor 0.5 > gti_dl19.run
```

You can then evaluate, assuming you have the qrels and `trec_eval` for example:
```
trec_eval -m ndcg_cut.10 guided-traversal/qrels.dl19-passage.txt gti_dl19.trec 
ndcg_cut_10           	all	0.7072

```
