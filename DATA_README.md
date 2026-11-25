# MSM Milling Dataset: Data Guide

A plain-language guide to the Multi-Sensor and MTConnect (MSM) metal-milling anomaly dataset used in this project.

- Paper: Kim et al., *Multi-sensor and MTConnect dataset of metal cutting anomaly in milling from laboratory and industry settings*, Scientific Data 13, 945 (2026). https://doi.org/10.1038/s41597-026-07255-7
- Dataset: https://doi.org/10.34740/KAGGLE/DS/8392825 (Kaggle: `manufuturetoday/multi-sensor-for-metal-milling-anomaly`)
- The data is large (a 14 GB download) and is **not stored in this repository**. See "Getting the data" below.

## What the data is

Three CNC milling machines were recorded while cutting metal. For every cutting run there are sound recordings, vibration recordings, the numbers the machine controller itself reports, and a label file where experts marked each cutting period as normal or abnormal.

The project goal is to detect abnormal cutting using **only the controller numbers** (the model that would run in a real factory), while using the extra sensors (sound, vibration) during training only.

## The three machines

| Machine | Setting | Runs |
| --- | --- | --- |
| `imi_vm20i` | Laboratory, 3-axis Hurco VM20i | 37 |
| `imi_vmx30ui` | Laboratory, 5-axis Hurco VMX30Ui | 19 |
| `tmf_vf10` | Real factory, Haas VF-10 (dry cutting) | 13 |

Each machine has a folder, and inside it one folder per run (`dataset1`, `dataset2`, ...). One folder is one run: one session of cutting from start to finish. 69 runs in total.

Recorded time is 8 h 12 m 23 s, of which 4 h 44 m 38 s is real cutting. Our own count of labelled cutting time is 17,078 s (4 h 44 m 38 s), which matches the paper.

## Files inside a run folder

| File | What it is | Machines that have it |
| --- | --- | --- |
| `*_s0.wav`, `*_s2.wav` | Sound from two microphones, 48 kHz, one channel | all three |
| `*_s1.wav` | Sound from a third microphone | `imi_vm20i` only |
| `*_acc.csv` | Vibration from two accelerometers (`acc0`, `acc1`), about 51,200 readings per second. The biggest files (about 309 MB for one run), and the reason the dataset is large | `imi_vm20i` only |
| `*_mtc.csv` | The numbers reported by the machine controller (MTConnect): spindle speed, spindle load, tool number, state, positions. Small files (about 0.3 MB) | all three |
| `*_ct.csv` | Electric current of one line, one reading about every 10 seconds | `tmf_vf10` only |
| `label.csv` | The expert labels: start time, end time, label, and cutting conditions | all three |
| `*.nc`, `*.mp4` | The cutting program and a video of the cut. Not used in this project | the two lab machines |

Outside the run folders:

- `dataset_summary.xlsx`: one row per run with machine, run number, workpiece material, tool and a remark (some runs stopped early, for example because of a clamp failure, excessive vibration or tool breakage).
- `imi_vm20i.xml`, `imi_vmx30ui.xml`, `tmf_vf10.xml`: the MTConnect device models, which describe what every controller variable means.

Times in the csv files and in `label.csv` are in seconds from the start of the run.

## Labels

| Label | Meaning |
| --- | --- |
| 0 | Normal cutting |
| 1 | Abnormal cutting, which should be stopped |
| 2 | Defective tool that is still cutting normally |

Tools marked with an asterisk in `dataset_summary.xlsx` (for example `A*`) are intentionally defective tools. A defective tool does not always get label 2: when the defect makes the cutting abnormal, the label is 1.

| Machine | Normal (0) | Abnormal (1) | Tool defect (2) |
| --- | --- | --- | --- |
| `imi_vm20i` | 1,173 segments, 3,662 s | 325 segments, 1,394 s | 87 segments, 238 s |
| `imi_vmx30ui` | 704 segments, 1,527 s | 133 segments, 656 s | 90 segments, 185 s |
| `tmf_vf10` | 56 segments, 9,180 s | 7 segments, 237 s | none |
| Total | 1,933 segments, 14,369 s | 465 segments, 2,287 s | 177 segments, 423 s |

The factory machine has only 7 abnormal segments, so a test on it alone would be noisy.

## Controller numbers versus extra sensors

`mtc.csv` mixes two kinds of columns:

- **Controller variables** (what the deployed model may use): spindle speed and load, tool number, execution state, mode, positions, feed rate and overrides, axis loads.
- **Extra sensors that arrive through MTConnect** (not available in a real deployment, so excluded from the controller-only model): `energy` and `watt` (power meter) and the motor vibration, displacement and temperature columns (names starting with `sm`, `xm`, `ym`, `zm`).

The three controllers name their variables differently. Only `time` and `avail` have the same name on all three, so shared variables are matched by meaning. The mapping is in `outputs/controller_variables_by_meaning.csv`. The factory machine has no axis loads, feed rate or overrides.

Extra sensors per machine:

| Signal | `imi_vm20i` | `imi_vmx30ui` | `tmf_vf10` |
| --- | --- | --- | --- |
| Sound 0 and sound 2 | yes | yes | yes |
| Sound 1 | yes | no | no |
| Accelerometers | yes | no | no |
| Current meter | no | no | yes |
| Motor vibration columns in `mtc.csv` | yes | yes | no |
| Power meter columns in `mtc.csv` | yes | yes | no |

## Getting the data

Download it from Kaggle (a free account is needed):

```python
import kagglehub

path = kagglehub.dataset_download("manufuturetoday/multi-sensor-for-metal-milling-anomaly")
print(path)
```

Put the extracted files so that a folder named `Data` sits next to the notebook, with `imi_vm20i`, `imi_vmx30ui`, `tmf_vf10` and `dataset_summary.xlsx` inside it. Or set the environment variable `MSM_DATA_DIR` to the data folder.

## Reproducing the tables

Open `MSM_Data_Loading_and_Sensor_Availability.ipynb` and run all cells. It writes these files to `outputs/`:

- `file_availability_by_machine.csv`
- `label_counts.csv`
- `mtc_columns_by_machine.csv`
- `controller_variables_by_meaning.csv`
- `extra_sensor_availability.csv`

## Licence

The paper's data-availability section states CC BY 4.0, while the article's rights section lists CC BY-NC-ND. Check the terms before publishing anything that uses the data, cite the paper, and do not redistribute the data files.
