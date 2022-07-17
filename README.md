# Audio Data Importers
> A collection of data importers for various audio sources. A loose manual data pipeline.


## Install

```bash
pip install dataimporters
```

### Downloading Audio Sources

The audio sources have to be provided manually (for now).  
The scripts expect a data directory containing the audio folders:  
```
root
 |- data/
      |- original/ (where you have to place the soundbanks)
      |- intermediate/ (generated)
      |- dataset/ (generated)
```

## Contributing

We use **nbdev**, which compiles all notebooks into a package. The source is at the `nbs` folder.

## How to Use

To create a new dataset package, we simply:  
1. Define and process all sources,
1. import the `Dataset`,  
1. give it the sources we'd like to include and the path to our data,  
1. call `Dataset.compile`

This will process all sources and build a final `dataset.zip` file.  

The library is flexible, but here's the simplest and most common action we perform:

```python
#hide_ouput
from DataImporters.core import load_version

DATA_DIR = "data/"
VERSION = load_version()
VERSION
```




    17



```python
from DataImporters.sources.core import process
from DataImporters.sources.space_divers_mini import SpaceDiversMini
from DataImporters.sources.footsteps_one_ppsfx import FootstepsOnePpsfx
from DataImporters.sources.footsteps_two_ppsfx import FootstepsTwoPpsfx
from DataImporters.sources.edward import Edward
from DataImporters.sources.barefoot_metal_sonniss import BarefootMetalSonniss
from DataImporters.sources.custom_fsd import CustomFsd

all_sources = [
    SpaceDiversMini(),
    FootstepsOnePpsfx(),
    FootstepsTwoPpsfx(),
    Edward(),
    BarefootMetalSonniss(),
    CustomFsd()
]

for source in all_sources:
    process(source, DATA_DIR, VERSION)
```


Below are two examples, one creates a large dataset with the automatic processors, the other is a more balanced dataset, manually annotated.  
Choose one to run and then jump to `Verify Output`

### Larger Dataset

```python
from DataImporters.dataset import Dataset, DatasetPaths

DATASET_NAME = "large"

# Same as `all_sources` excluding SpaceDiversMini
sources = [
    FootstepsOnePpsfx(),
    FootstepsTwoPpsfx(),
    Edward(),
    BarefootMetalSonniss(),
    CustomFsd()
]

paths = DatasetPaths(DATA_DIR, DATASET_NAME)
metadata = Dataset(sources, paths).compile()
metadata.shape[0]
```

    Warning: 206 duplicate rows found. Some rows were dropped (all files copied).





    1646



### Smaller and Annotated

```python
# We also want to apply simple rename rules
from DataImporters.sources.core import normalise_label
import pandas as pd

def rename_categories(metadata: pd.DataFrame) -> pd.DataFrame:
    replacements = [
        (normalise_label("ship_horn"), normalise_label("horn")),
        (normalise_label("robot_movement"), normalise_label("robot")),
        (normalise_label("zombie_noises"), normalise_label("zombie")),
        (normalise_label("sword_hit"), normalise_label("sword")),
    ]

    metadata = metadata.copy()
    for old, new in replacements:
        metadata.loc[metadata["category"] == old, "category"] = new

    return metadata
```

```python
from DataImporters.dataset import Dataset, DatasetPaths

DATASET_NAME = "small_balanced"
ANNOTATION_PATH = os.path.join(DATA_DIR, "annotations", DATASET_NAME + ".csv")

sources = [
    CustomFsd()
]

paths = DatasetPaths(DATA_DIR, DATASET_NAME, ANNOTATION_PATH)
metadata = Dataset(sources, paths).compile(rename_categories)
metadata.shape[0]
```

    Warning: 207 duplicate rows found. Some rows were dropped (all files copied).





    284



### Verify Output

`Dataset.compile` will return the newly created metadata _(which has already been saved to `DATA_PATH`)_.  

We can use it to confirm we did indeed copy all files. Since the metadata aggregates all the source metadata, if a file is missing, it will still be in the metadata.  
On the other hand, this will also let us know when a file has been deleted from the source, but still exists in the dataset folder.  

```python
import os
assert len(os.listdir(paths.audio_output_path)) == len(metadata)
```

Everything is looking good, we should bump the `version`.

```python
#hide_output

from DataImporters.core import bump_version
bump_version()
```

If the assertion fails, this could be due to:  
* Genuine failure to copy  
* Some files in the target folder need deleting  
  * Please delete them, no code yet
* Hash conflict (same content from different sources)  
  * In this case, we must debug the sources and make sure there are no duplicates

## Dataset Structure

```
dataset/
  |- README.md
  |- metadata.csv
  |- audio/
       |- Long list of audio files, filenames are the xxhash64 of the content.
```

### Metadata

`metadata.csv` contains a list of all the files in the dataset and their labels.  

filename | category | label | extra | source | version 
--- | --- | --- | --- | --- | --- 
File name, assumes all files inside audio folder | Single major category name | Escaped (“”) comma separated list of labels, in snake_case | Extra text/details available for this row (unstructured) | Name of original sound library, snake_case | Version of the last change. Limited to last change only  

_`version` is a simple incremental integer. If you need to check if a file changed/added simply check if the row `version` is higher than the last `version` you ran. Deletes are not supported yet._

Here's an example from the sample code ran earlier:




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>filename</th>
      <th>category</th>
      <th>label</th>
      <th>extra</th>
      <th>source</th>
      <th>version</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>a2abde886f52e83e.wav</td>
      <td>Zombie_noises</td>
      <td>Horror</td>
      <td>NaN</td>
      <td>custom_fsd</td>
      <td>16</td>
    </tr>
    <tr>
      <th>1</th>
      <td>af864f34b0f0c323.wav</td>
      <td>Zombie_noises</td>
      <td>Scary,Growl,Monster,Zombie,Female,Scream</td>
      <td>NaN</td>
      <td>custom_fsd</td>
      <td>16</td>
    </tr>
  </tbody>
</table>
</div>



## Pipeline

```mermaid
flowchart TD
    sa[(Source A)] --> pa([Normalise data and create CSV]);
    pa --> ia[(Intermediate A)];
    sb[(Source B)] --> pb([Normalise data and create CSV]);
    pb --> ib[(Intermediate B)];
    ia & ib & a(WIP: Manual annotations by hash) --> c([Compile])
    c-- Some rows can be rejected at this stage --> d[(Dataset)];
```

Each loader outputs:  
* a CSV, which is then compiled into a single metadata.csv  
* the files into an intermediate folder  

The process above is done so that:  
* Each source is independent  
* We can easily compile a final dataset with different sources  
* Easier to make the split consistent across runs  
