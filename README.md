# DataImporters

> A collection of data importers or clean-up for various sources.

Each loader outputs:  
* a CSV, which is then compiled into a single metadata.csv  
* the files into an intermediate folder  

The process above is done so that:  
* Each notebook is independent  
* We can easily compile a final dataset with different sources  
* Easier to make the split consistent across runs  

## Installation 

```
> pip install -r requirements.txt
```

The datasets have to be provided manually (for now).  

## Dataset structure

See [Dataset README](data/dataset/README.md).

## Pipeline

```mermaid
flowchart TD
    sa[(Source A)] --> pa([Normalise data and create CSV]);
    pa --> ia[(Intermediate A)];
    sb[(Source B)] --> pb([Normalise data and create CSV]);
    pb --> ib[(Intermediate B)];
    ia & ib --> c([Compile])
    c-- Some rows can be rejected at this stage --> d[(Dataset)];
```
