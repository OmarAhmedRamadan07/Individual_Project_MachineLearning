# Deployment-Compatible Knowledge Transfer for Industrial Machine Monitoring

Individual Project, CET351 – Machine Learning

**Student:** Omar Ahmed Ramadan Ramadan
**Supervisor:** Eng. Ashraf A. Mohamed
**Status:** Phase I – data audit complete, feature extraction and baselines in progress

---

## 1. Project summary

Anomaly detection in CNC milling is more accurate when extra sensors (sound, vibration) are available, but many real machines only expose the numbers reported by their controller. This project asks:

> **Which information from additional sensors can actually be transferred to a controller-only model across changing machining conditions?**

A stronger "teacher" model (controller + sound) does not automatically give useful knowledge to the "student" model that will be deployed (controller only). The planned contribution is a **selective transfer objective** that separates teacher information that can be recovered from controller signals and is diagnostically useful from information that depends on sensors the deployed model will not have.

The deployed model must use **controller signals only**. Extra sensors are used during training only.

## 2. Important note: the data is NOT included in this repository

The dataset is **not uploaded to this repository because of its size**: it is about **14 GB to download and about 36 GB after extraction**, which is far above GitHub's file-size limits.

Everything needed to get it is below.

| Resource | Link |
| --- | --- |
| Dataset (Kaggle) | https://doi.org/10.34740/KAGGLE/DS/8392825 |
| Kaggle dataset id | `manufuturetoday/multi-sensor-for-metal-milling-anomaly` |
| Data paper (Scientific Data, 2026) | https://doi.org/10.1038/s41597-026-07255-7 |

The `Data/` folder is listed in `.gitignore` on purpose.

### How to download the data

A free Kaggle account is required.

```python
import kagglehub

path = kagglehub.dataset_download("manufuturetoday/multi-sensor-for-metal-milling-anomaly")
print(path)
```

Then arrange the extracted files so that a folder named `Data` sits next to the notebook:

```
Data/
├── imi_vm20i/          # laboratory machine, 3-axis (37 runs)
├── imi_vmx30ui/        # laboratory machine, 5-axis (19 runs)
├── tmf_vf10/           # industrial machine (13 runs)
├── dataset_summary.xlsx
├── imi_vm20i.xml       # MTConnect device models
├── imi_vmx30ui.xml
└── tmf_vf10.xml
```

Alternatively, set the environment variable `MSM_DATA_DIR` to the path of the data folder.

## 3. About the dataset

Three CNC milling machines were recorded while cutting metal. For every run there are sound recordings, vibration recordings, the values reported by the machine controller (MTConnect), and expert labels for each cutting period.

| Machine | Setting | Runs |
| --- | --- | --- |
| `imi_vm20i` | Laboratory, 3-axis Hurco VM20i | 37 |
| `imi_vmx30ui` | Laboratory, 5-axis Hurco VMX30Ui | 19 |
| `tmf_vf10` | Industrial, Haas VF-10 (dry cutting) | 13 |

**Total: 3 machines, 69 runs.** Labelled cutting time is 17,078 s (4 h 44 m 38 s), which matches the data paper.

### Labels

| Label | Meaning | Use in this project |
| --- | --- | --- |
| 0 | Normal cutting | Negative class |
| 1 | Abnormal cutting requiring intervention | **Primary positive class** |
| 2 | Deliberately defective tool, still cutting normally | Excluded from the primary binary task; evaluated separately later |

| Machine | Label 0 | Label 1 | Label 2 |
| --- | --- | --- | --- |
| `imi_vm20i` | 1,173 segments / 3,662 s | 325 segments / 1,394 s | 87 segments / 238 s |
| `imi_vmx30ui` | 704 segments / 1,527 s | 133 segments / 656 s | 90 segments / 185 s |
| `tmf_vf10` | 56 segments / 9,180 s | 7 segments / 237 s | none |
| **Total** | 1,933 / 14,369 s | 465 / 2,287 s | 177 / 423 s |

The industrial machine has only 7 abnormal segments, which are not necessarily 7 independent events. It is therefore reserved for a supplementary laboratory-to-industry evaluation.

### Sensors per machine

| Signal | `imi_vm20i` | `imi_vmx30ui` | `tmf_vf10` |
| --- | --- | --- | --- |
| Sound 0 and Sound 2 (`s0`, `s2`) | yes | yes | yes |
| Sound 1 (`s1`) | yes | no | no |
| Accelerometers | yes | no | no |
| Current transformer | no | no | yes |
| Motor vibration columns in `mtc.csv` | yes | yes | no |
| Power meter columns in `mtc.csv` | yes | yes | no |

Sound (`s0`, `s2`) is the only extra sensor available on all three machines, so it is the teacher's extra input in the pilot.

## 4. Experimental design (agreed with the supervisor)

- **Primary task:** binary detection, label 1 versus label 0.
- **Student (deployed model):** controller-only supervised model.
- **Teacher (training only):** controller + sound (`s0`, `s2`).
- **Windows:** the same causal (trailing) time windows for both models, with no interpolation or normalisation that uses future parts of a run.
- **Controller features:** start with spindle speed and spindle load. Position coordinates, timers and operating modes stay excluded until their mappings are verified.
- **Excluded from predictors:** externally measured power, vibration, displacement and temperature (even when transmitted through MTConnect), program names, block numbers, tool identifiers, remarks, and stop/intervention indicators.
- **Primary evaluation:** held-out machining conditions on the two laboratory machines, with complete runs kept in one partition (no windows from one run across partitions).
- **Supplementary evaluation:** `tmf_vf10` for laboratory-to-industry transfer.
- **Metrics:** PR-AUC, recall at a validation-fixed false-positive operating point, calibration, results by condition, computational cost, and conditions where transfer worsens performance.

### Methods to compare

1. Controller-only supervised model
2. Full-sensor reference model (controller + sound)
3. Standard knowledge distillation
4. Confidence-weighted knowledge distillation
5. An established privileged-information transfer method
6. Proposed selective transfer, with ablations separating recoverability estimation from transfer selection

## 5. Repository structure

```
.
├── README.md                  # this file
├── DATA_README.md             # detailed guide to the dataset files and labels
├── code/
│   └── MSM_Data_Loading_and_Sensor_Availability.ipynb
├── outputs/
│   ├── file_availability_by_machine.csv
│   ├── label_counts.csv
│   ├── mtc_columns_by_machine.csv
│   ├── controller_variables_by_meaning.csv
│   └── extra_sensor_availability.csv
└── .gitignore                 # excludes Data/ (too large to upload)
```

## 6. Dependencies

Python 3.10 or later.

```bash
pip install numpy pandas scipy matplotlib openpyxl jupyter kagglehub
```

| Package | Used for |
| --- | --- |
| `numpy`, `pandas` | data handling |
| `scipy` | reading `.wav` files (`scipy.io.wavfile`) |
| `matplotlib` | plots |
| `openpyxl` | reading `dataset_summary.xlsx` |
| `kagglehub` | downloading the dataset |

## 7. How to run

1. Download the data as described in section 2.
2. Open `code/MSM_Data_Loading_and_Sensor_Availability.ipynb`.
3. Make sure the `Data/` folder is next to the notebook, or set `MSM_DATA_DIR`.
4. Run all cells.

The notebook discovers the 69 runs, counts the label segments and durations, separates controller variables from extra sensors, builds the availability tables, and writes the five CSV files above to `outputs/`.

## 8. Progress

### Completed
- Registered the project and agreed the direction with the supervisor.
- Executed data-loading notebook: 3 machines, 69 runs, labelled cutting time matching the data paper.
- Sensor-availability tables and controller-variable mapping by meaning.
- Data guide (`DATA_README.md`).

### In progress
- Auditing all 69 runs: schema consistency, missingness, channel validity, timestamp ordering and update intervals (the current column inventory reads only the first run per machine).
- Verifying the remaining controller-variable mappings marked "to confirm" against the MTConnect device models.
- Focused literature comparison with cross-modal distillation and privileged-information methods.

### Next
- Run–condition–label inventory and frozen split manifest.
- Verified predictor whitelist.
- Audio–controller alignment example.
- Feature extraction and reproducible baselines: controller-only student versus controller + sound teacher.
- Selective-transfer objective, only after the literature comparison is complete.

## 9. Known limitations and challenges

- The raw data is too large to host here; results can only be reproduced after downloading it from Kaggle.
- Matching controller column meanings across machines does not guarantee equivalent measurement scales, so cross-machine feature sets need unit checks.
- The industrial machine has few abnormal segments, so results on it alone would be noisy.
- Controller-based diagnostics must avoid variables that identify experimental conditions or operator responses (program names, block numbers, tool identifiers, stop indicators).

## 10. Licence and citation

The paper's data-availability section states CC BY 4.0, while the article's rights section lists CC BY-NC-ND. Check the terms before publishing anything that uses the data, and do not redistribute the data files.

If you use the dataset, cite the data paper:

> Kim et al., *Multi-sensor and MTConnect dataset of metal cutting anomaly in milling from laboratory and industry settings*, Scientific Data 13, 945 (2026). https://doi.org/10.1038/s41597-026-07255-7

## 11. Author

Omar Ahmed Ramadan Ramadan, omar240102684@sut.edu.eg
