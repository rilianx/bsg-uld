# BSG-ULD: beam search for air cargo palletization

Code, instances, solutions and scripts of the paper

> I. Araya. *A beam search for the air cargo palletization problem with weight distribution
> constraints.* Submitted to Computers & Operations Research.

`BSG_ULD` loads a set of heterogeneous boxes into unit load devices (ULDs) of minimum total
volume. The ULDs have bevelled corners, a weight capacity, a floor edge for the cargo net locks
and an optional substructure; the boxes have allowed orientations and may be fragile; every box
must be supported; and the center of gravity (CoG) of each ULD must lie in a given area around
its geometric center. This is the problem of Paquay et al. (2016, 2018) with the extensions of
Heßler et al. (2025).

Each ULD is loaded by BSG, a beam search guided by a greedy algorithm that places blocks of
boxes in maximal free spaces and evaluates them with the VCS function (Araya & Riff, 2014;
Araya et al., 2017). The ULDs are loaded one at a time, and the total volume is then reduced by
removing a ULD, inserting its boxes into the others and loading the rest into at most two
smaller ULDs.

## Build

Requirements: a C++11 compiler and CMake. The scripts need Python 3 (the comparison with
Heßler et al. also needs scipy).

    git clone https://github.com/rilianx/bsg-uld.git
    cd bsg-uld
    mkdir build && cd build
    cmake .. -DCMAKE_BUILD_TYPE=Release
    make

## Usage

    ./build/BSG_ULD <instance.csv> [options]

The ULD types are read from `uld.csv` in the parent directory of the instance (or `--uld`).
The three settings of the paper:

    # Paquay: original instances, CoG deviation 5% of the half dimensions
    ./build/BSG_ULD instances/Final_instances/n50sample0.csv -t 0.1 --cog_rel 0.05 -v

    # Adapted: floor edge and substructure, support by area with padding, CoG deviation 10%
    ./build/BSG_ULD instances/Final_instances/n50sample0.csv -t 0.1 \
        --edge 10 --edge_offset 10 --min_overlap 0.9 --padding 10 --cog_rel 0.1 -v

    # One: adapted with one ULD of each type (some boxes may remain unloaded)
    ./build/BSG_ULD instances/Final_instances/n50sample0.csv -t 0.1 \
        --edge 10 --edge_offset 10 --min_overlap 0.9 --padding 10 --cog_rel 0.1 --uld_count 1 -v

`-t` is the time limit of each run of BSG in the construction (seconds, without the block
generation); the improvement phases use `0.1 t`. A ULD is loaded several times, so the total
time per instance is several times `t`: with `-t 0.1` it is about 0.5–1 s, comparable to the
algorithm of Heßler et al. on their machine.

Main options (`./build/BSG_ULD --help` lists all of them):

| option | meaning | default |
|---|---|---|
| `-t` | time limit of each run of BSG (s) | 1 |
| `--cog_rel` | maximum CoG deviation in x and y, relative to the half dimensions | ULD file |
| `--edge`, `--edge_offset` | width and height of the floor edge (mm); with an edge, a substructure is also tried | 0 |
| `--min_overlap`, `--padding` | support by area: minimum supported proportion and maximum height of the padding material | 0 (four corners only) |
| `--uld_count` | number of ULDs of each type | unlimited |
| `--cog_penalty` | penalty λ of the CoG excess in the value of a loading | 1.4035 |
| `--theta` | weight θ of the ULD volume in the choice of the ULD type (negative: rule of Heßler et al.) | 0.2316 |
| `--alpha`, `--beta`, `--gamma`, `-p` | parameters of the VCS function | 0.5139, 4.7497, 0.7727, 0.088 |
| `--max_bl` | maximum number of blocks | 500 |
| `--must_w` | limited ULDs: extra weight of the boxes that must be loaded in the ULD | 10 |
| `--restarts` | limited ULDs: maximum number of restarts of the construction | 3 |
| `-o file` | write the solution | |
| `-v` | print the loaded ULDs | |

The options `--no_translate`, `--corner_anchor`, `--no_substructure`, `--no_ruin`,
`--reduce_new 1`, `--eliminate_only`, `--no_eliminate`, `--cog_greedy`, `--scratch` and
`--priority` switch components off or on for the ablation study.

## Solutions and validation

With `-o file` the solution is written with one line per box: ULD, ULD type, substructure,
item (line of the instance file, from 0), position and dimensions. It can be checked with an
independent validator, which shares no code with the solver:

    python3 scripts/validate_uld.py <instance.csv> instances/uld.csv <solution> [options]

with the same problem options as the solver (`--cog_rel`, `--edge`, `--edge_offset`,
`--min_overlap`, `--padding`) and `--allow_unloaded` when the number of ULDs is limited.

It checks that every box is inside its ULD and its cuts, outside the edge, not overlapping other
boxes, supported, not on a fragile box and within the weight capacity, and it counts the ULDs
whose CoG is out of range. All the solutions of the paper pass it.

## Reproducing the experiments

    scripts/run_all.sh results/new        # all the experiments of the paper
    SETTINGS="one" scripts/run_all.sh results/new                       # one setting
    SETTINGS="adapted one" ONLY_CURVEFULL=1 scripts/run_all.sh results/new  # only some curves

`run_all.sh` compiles the code, runs each configuration on the 300 final instances (in
parallel, one thread per run), validates every solution and writes a summary to
`<output>/summary.txt`, with the commit and the machine in `<output>/environment.txt`. It can
be interrupted and launched again: finished configurations are not run again unless the binary
changes. `scripts/paper_stats.py` computes the statistics of the paper (comparison instance by
instance with Heßler et al., Wilcoxon test), `scripts/collect.py` the data of the time figure and
`scripts/size_tables.py` the tables by instance size.

The parameters were tuned with irace on the 150 training instances (`tuning/`, see
`tuning/scenario.txt`).

## Results

Results on the 300 final instances of Paquay et al. (2018) (30 per size, 10 to 100 boxes), five
runs per configuration (mean ± standard deviation), compared instance by instance with the
randomized greedy search (RGS) of Heßler et al. (2025) in their objective: total volume of the
ULDs (with one ULD of each type, first the loaded volume). The RGS takes 0.19–0.42 s per instance
on an Apple M2 Pro, about 0.5–1.1 s on our machine (PassMark single-thread ratio 2.53).
Experiments run on two Intel Xeon E5-2620 v4 (2.1 GHz), 14 runs in parallel. The runs are in
`results/final/runs.tar.gz` and, for the setting *one*, in `results/one_must/runs.tar.gz`.

* paquay: RGS 18648 m3, 708 ULDs, 11.2% of the ULDs with the CoG out of range;
* adapted: RGS 16843 m3, 653 ULDs, 7.2%;
* one: RGS 16166 m3, 729 ULDs, 10.1%, 24 boxes (25.9 m3) not loaded.

| instances | -t | volume (m3) | vs RGS | better / worse | ULDs | not loaded (m3) | CoG xy | CoG z | avg time |
|---|---|---|---|---|---|---|---|---|---|
| paquay  | 0.1 s | 18106 ± 17 | -2.9% | 158 / 74.4 | 777.6 | 0 | 7.8 | 4.2 | 0.56 s |
| paquay  | 1 s   | 17317 ± 4  | -7.1% | 182.8 / 51.8 | 732.6 | 0 | 0 | 1.4 | 3.5 s |
| adapted | 0.1 s | 16203 ± 25 | -3.8% | 151.4 / 62.8 | 725.8 | 0 | 3 | 3 | 0.77 s |
| adapted | 1 s   | 15461 ± 6  | -8.2% | 184 / 44.6 | 689.0 | 0 | 0 | 0 | 5.2 s |
| one     | 0.1 s | 15622 ± 7 | -3.4% | 138.2 / 44 | 758.8 | 27 (22.8) | 5 | 7.8 | 1.02 s |
| one     | 1 s   | 15008 ± 4  | -7.2% | 184.2 / 29 | 728.2 | 12 (12.6) | 4 | 7 | 6.8 s |

Volume vs RGS according to the time limit (one run; 5 runs for 0.1 and 1 s; mean time per
instance in parentheses):

| -t | paquay | adapted | one (not loaded) |
|---|---|---|---|
| 0.02 s | +2.4% (0.24 s) | +1.2% (0.29 s) | +0.1%, 43.6 m3 (0.43 s) |
| 0.05 s | -1.3% (0.35 s) | -1.6% (0.46 s) | -2.0%, 20.4 m3 (0.63 s) |
| 0.1 s | -2.9% (0.56 s) | -3.8% (0.77 s) | -3.4%, 22.8 m3 (1.02 s) |
| 0.2 s | -5.9% (0.94 s) | -5.9% (1.32 s) | -4.4%, 16.6 m3 (1.71 s) |
| 0.5 s | -6.1% (1.92 s) | -6.7% (2.77 s) | -6.1%, 11.4 m3 (3.68 s) |
| 1 s | -7.1% (3.51 s) | -8.2% (5.15 s) | -7.2%, 12.6 m3 (6.84 s) |
| 2 s | -8.1% (6.64 s) | -8.7% (9.62 s) | -7.9%, 11.5 m3 (12.4 s) |

Ablation (0.1 s per ULD, one run; volume of the ULDs vs RGS and ULDs with the CoG out of range
in x or y; with one ULD of each type also the volume not loaded):

| variant | paquay | adapted | one |
|---|---|---|---|
| complete algorithm (mean of 5 runs) | -2.9% / 7.8 | -3.8% / 3 | -3.4% / 5, 22.8 m3 |
| no CoG penalty (`--cog_penalty 0`) | -6.0% / 460 | -3.0% / 281 | -2.0% / 308, 15.4 m3 |
| no translation (`--no_translate`) | -1.4% / 183 | -2.4% / 117 | -2.7% / 192, 16.1 m3 |
| CoG in the block evaluation (`--cog_greedy 0.5`) | -3.6% / 10 | -3.8% / 2 | -4.5% / 6, 20.6 m3 |
| spaces at the nearest corner (`--corner_anchor`) | -4.2% / 4 | -3.4% / 2 | -3.5% / 6, 21.5 m3 |
| no substructure (`--no_substructure`) | - | -2.2% / 7 | -1.7% / 14, 33.0 m3 |
| rule of Hessler et al. (`--theta -1`) | -2.5% / 7 | -3.6% / 3 | -3.6% / 5, 25.0 m3 |
| no restarts (`--restarts 0`) | - | - | -3.3% / 5, 38.3 m3 |
| no priority of the boxes that must be loaded (`--must_w 0`) | - | - | -8.0% / 5, 259.5 m3 |
| insert also with reload from scratch (`--scratch`) | -2.3% / 10 | -2.9% / 3 | -3.3% / 5, 20.8 m3 |
| insert also with priority reload (`--priority`) | -2.0% / 12 | -2.0% / 2 | -3.4% / 5, 20.2 m3 |
| no insert (`--no_ruin`) | +0.2% / 10 | +0.2% / 4 | -2.0% / 7, 24.8 m3 |
| reduction with one new ULD (`--reduce_new 1`) | -1.6% / 8 | -2.5% / 2 | -3.0% / 5, 22.8 m3 |
| only elimination of ULDs (`--eliminate_only`) | +0.3% / 10 | +0.4% / 4 | -2.7% / 6, 22.8 m3 |
| no reduction (`--no_eliminate`) | +0.8% / 10 | +0.6% / 2 | -1.8% / 6, 22.8 m3 |
| basic BSG (all of the above off) | +13.9% / 90 | +18.4% / 61 | +9.4% / 59, 54.6 m3 |

## Contents

* `src/`: the search framework (`State`, `SearchStrategy`, `Greedy`, `BSG`, `DoubleEffort`), the
  problem (`clpState`, `Block`, `Container`, `ULD`, `VCS_Function`) and the driver
  `main_uld.cpp`.
* `instances/`: the instances of Paquay et al. (2018): 300 final instances
  (`Final_instances`), 150 preliminary instances used for tuning (`training`) and the six ULD
  types of a Boeing 777 (`uld.csv`).
* `hessler/`: results of Heßler et al. (2025) per instance and per ULD (Tables 8–13 of their
  online appendix).
* `scripts/`: validator, experiment and analysis scripts.
* `tuning/`: irace scenario.
* `results/`: runs, summaries and machine of the experiments of the paper.

## References

* I. Araya, M.-C. Riff (2014). A beam search approach to the container loading problem.
  *Computers & Operations Research* 43, 100–107.
* I. Araya, K. Guerrero, E. Nuñez (2017). VCS: A new heuristic function for selecting boxes in
  the single container loading problem. *Computers & Operations Research* 82, 27–35.
* C. Paquay, M. Schyns, S. Limbourg (2016). A mixed integer programming formulation for the
  three-dimensional bin packing problem deriving from an air cargo application. *International
  Transactions in Operational Research* 23(1–2), 187–213.
* C. Paquay, S. Limbourg, M. Schyns (2018). A tailored two-phase constructive heuristic for the
  three-dimensional Multiple Bin Size Bin Packing Problem with transportation constraints.
  *European Journal of Operational Research* 267(1), 52–64.
* K. Heßler, T. Hintsch, L. Wienkamp (2025). A fast optimization approach for a complex
  real-life 3D Multiple Bin Size Bin Packing Problem. *European Journal of Operational Research*
  327(3), 820–837.

If you use this code, please cite the paper above. The instances are those of Paquay et al.
(2018); please also cite their work.

## License

GPL-3.0 (see `LICENSE`).
