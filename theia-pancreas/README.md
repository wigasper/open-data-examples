# THEIA-Pancreas

THEIA-Pancreas is an openly licensed (CC BY 4.0) collection of human pancreas whole-slide images
(WSIs), expert polygon annotations, and a pre-tiled semantic segmentation benchmark. It is hosted
on AWS through the [AWS Open Data Sponsorship Program](https://aws.amazon.com/opendata/open-data-sponsorship-program/)
and listed on the [Registry of Open Data on AWS](https://registry.opendata.aws/theia-pancreas/).

> **Status:** onboarding draft. The S3 bucket is not populated yet; prefixes and counts below
> describe the planned layout of the first release.

| Component | Contents | Scale |
|---|---|---|
| Donor pancreas atlas | WSIs from organ donors, sampled across head, body, tail, and uncinate process. Donors may have various pathologies or be completely normal. | 1,687 WSIs, 284 donors |
| Annotated PDAC slides | WSIs from a neoadjuvant-treated UT Southwestern (UTSW) cohort and the treatment-naive CPTAC-PDA cohort, with expert polygon annotations | 223 WSIs (50 UTSW + 173 CPTAC-PDA), more than 140,000 polygons |
| DHCP benchmark | DeepSeg Human-Curated PDAC: 1024x1024 tiles at 0.5 microns per pixel (mpp) with pixel-level masks, split 70/20/10 at the patient level | 75,519 image/mask pairs |

Total size: approximately 3.5 TB.

## Accessing the data

The bucket is public and read-only. No AWS account is needed.

```sh
aws s3 ls --no-sign-request s3://theia-pancreas/
```

```python
import boto3
from botocore import UNSIGNED
from botocore.config import Config

s3 = boto3.client("s3", config=Config(signature_version=UNSIGNED))
s3.list_objects_v2(Bucket="theia-pancreas", Delimiter="/")
```

A guided tour is in [`get-to-know-a-dataset.ipynb`](get-to-know-a-dataset.ipynb).

## Bucket layout

```
s3://theia-pancreas/
  README.txt
  LICENSE.txt
  metadata/
    slide_manifest.csv                  # donor atlas slide index (see data dictionary)
    dhcp-croissant.json                 # MLCommons Croissant metadata for the DHCP benchmark
  data/
    wsi/
      donor-atlas/
        donor_id=<DONOR_ID>/<DONOR_ID>_<REGION><REPLICATE>.svs
      pdac/
        cohort=utsw/<slide_id>.{svs,ndpi,qptiff}
        cohort=cptac-pda/<slide_id>.jpg # cropped from the source TCIA slides before annotation
    annotations/
      json/
        cohort=utsw/<slide_id>.json
        cohort=cptac-pda/<slide_id>.xml
    benchmark/
      dhcp/
        v1.0.0/
          classes_to_int.tsv
          train_data/{images,masks}/
          test_data/{images,masks}/
          validation_data/{images,masks}/
```

Design notes:

- Files are stored as individual objects (no archives), so a single slide, annotation, or tile
  can be fetched on its own.
- Partition-style prefixes (`donor_id=`, `cohort=`, `resolution=`) make the layout self-describing
  and work directly with tools such as Amazon Athena and `pyarrow` datasets.
- The benchmark is versioned (`v1.0.0/`). New releases go into new prefixes, so published results
  stay reproducible.

## Data dictionary

### `metadata/slide_manifest.csv` (donor atlas)

| Column | Type | Description |
|---|---|---|
| `filename` | string | WSI object name, e.g. `AIJ1440_B.svs` |
| `donor_id` | string | De-identified organ donor identifier |
| `region` | string | Sampling site. `H` = head, `B` = body, `T` = tail, `U` = uncinate process, `A` = other, `SPLEEN` = spleen (not pancreas) |
| `replicate` | integer | Section replicate number for that donor and region (1-3) |

### Whole-slide images

| Format | Where | Notes |
|---|---|---|
| SVS (`.svs`) | donor atlas, UTSW | Tiled, pyramidal TIFF. Full resolution is ~0.25 mpp (40x). |
| NDPI (`.ndpi`) | UTSW | Tiled, pyramidal TIFF variant. Full resolution is ~0.22 mpp (40x). |
| QPTIFF (`.qptiff`) | UTSW (1 slide) | Pyramidal TIFF |
| JPEG (`.jpg`) | CPTAC-PDA | Regions cropped from the original CPTAC-PDA WSIs ([TCIA](https://doi.org/10.7937/k9/tcia.2018.sc20fo18)) before annotation |

All pyramidal formats can be read with [OpenSlide](https://openslide.org/) (`openslide-python`
plus `openslide-bin`).

### DHCP benchmark

| Item | Description |
|---|---|
| `classes_to_int.tsv` | Tab-separated `class_name`, `class_int` |
| `*/images/*.jpg` | 1024x1024 RGB H&E tile at 0.5 mpp |
| `*/masks/*.png` | 1024x1024 single-channel uint8 mask. The pixel value is the class integer. |
| File name | `{slide_id}_{left}_{upper}_{right}_{lower}`: the source slide and the tile's bounding box in source-image pixels (full-resolution slide pixels for UTSW, cropped-JPEG pixels for CPTAC-PDA). Each image and its mask share the same stem. |

| Class | Integer |
|---|---|
| background | 0 |
| tumor | 1 |
| artery | 2 |
| vessel | 3 |
| nerve | 4 |
| lymphoid_aggregate | 5 |
| normal_pancreas | 6 |
| normal_epithelium | 7 |
| adipose | 8 |

| Split | Slides (UTSW / CPTAC-PDA) | Tiles (UTSW / CPTAC-PDA) |
|---|---|---|
| train | 35 / 118 | 43,071 / 10,626 |
| test | 10 / 34 | 10,612 / 3,117 |
| validation | 5 / 21 | 6,325 / 1,768 |

Machine-readable metadata for the benchmark is provided in
[Croissant](https://mlcommons.org/croissant/) format (`metadata/dhcp-croissant.json`).

## Known limitations

- Some classes (adipose, normal pancreas, normal epithelium) are annotated as broad regions rather
  than individual structures.
- Structures whose label could not be determined confidently may be left unlabeled.
- The PDAC cohorts are mostly resectable, localized disease. The UTSW cohort received neoadjuvant
  chemotherapy and the CPTAC-PDA cohort is treatment-naive.
- These data are for research use only and are not suitable for clinical or diagnostic use.

## License and citation

All data are released under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). The CPTAC-PDA
source images come from The Cancer Imaging Archive ([doi:10.7937/k9/tcia.2018.sc20fo18](https://doi.org/10.7937/k9/tcia.2018.sc20fo18)).

## Contact

Ligorio Lab & Ghersi Lab - william.gasper@utsouthwestern.edu
