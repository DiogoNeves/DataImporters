# DataImporters
A collection of data importers or clean-up for various sources

Each loader outputs:  
* a CSV, which is then compiled into a single metadata.csv  
* the files into an intermediate folder  

The process above is done so that:  
* Each notebook is independent  
* We can easily compile a final dataset with different sources  
* Easier to make the split consistent across runs  