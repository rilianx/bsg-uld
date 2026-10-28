# BSG for air cargo palletization (BSG_ULD)

Code, instances, solutions and scripts of the paper *A beam search for the air cargo
palletization problem with weight distribution constraints* (I. Araya).

`BSG_ULD` loads a set of heterogeneous boxes into unit load devices (ULDs) with bevelled
corners, weight capacities, a floor edge and an optional substructure, respecting the allowed
orientations, fragile boxes, support and center of gravity constraints. Each ULD is loaded by
BSG, a beam search guided by a greedy algorithm that places blocks of boxes in maximal free
spaces and evaluates them with the VCS function (Araya & Riff 2014; Araya, Guerrero & Nuñez
2017).

Installation
-------

     git clone https://github.com/rilianx/bsg-uld.git
     cd bsg-uld
     mkdir build && cd build
     cmake ..
     make

Contents
-------
* `src/`: the search framework (`State`, `SearchStrategy`, `Greedy`, `BSG`, `DoubleEffort`),
  the problem (`clpState`, `Block`, `Container`, `ULD`, `VCS_Function`) and the driver
  `main_uld.cpp`.
* `instances/`: instances of Paquay et al. (2018): the 300 final instances
  (`Final_instances`), the 150 training instances used for tuning (`training`) and the ULD
  types (`uld.csv`).
* `hessler/`: results of Hessler et al. (2025) per instance (`hessler2024_table*.json`, Tables
  8, 10 and 12 of their appendix) and per ULD (`hessler2024_ulds.json`, Tables 9, 11 and 13).
* `scripts/`: independent validator (`validate_uld.py`), lower bound (`lower_bound_uld.py`),
  comparison with Hessler et al. (`compare_hessler.py`, requires scipy) and the scripts of the
  computational experiments (`run_all.sh`, `run_uld.sh`, `summarize.py`, `collect.py`,
  `size_tables.py`).
* `tuning/`: irace scenario.
* `results/final/`: results of the experiments of the paper (all the runs in `runs.tar.gz`, summary in `summary.txt`, machine in `environment.txt`).

`BSG_ULD` solves the air cargo palletization problem with the instances of
Paquay et al. (2018) (`instances/`): all the boxes must be loaded
into unit load devices (ULDs) of six types (Boeing 777), minimizing the number of
ULDs. The ULDs are loaded one by one with BSG (all the free spaces anchored at the corner
closest to the origin, so that the load stays grouped and can be translated to center its
center of gravity); the next ULD type is chosen among the candidates of the rule of Hessler et al. (2025) by
loading each one and maximizing loaded volume - theta * ULD volume (`--theta`, default 0.2316;
negative: the rule of Hessler et al. alone; the candidates are loaded with the time of the
improvement phases and the chosen one again with the full time). Then the boxes
left unloaded (limited ULDs) are inserted into the loaded ULDs; if some are still left out, the
construction is restarted (at most `--restarts` times, default 3) loading those boxes first in
the first ULD in which they fit, and the solution with the least unloaded volume is kept. Then
the total volume of the
ULDs (the objective of Paquay et al. and Hessler et al.) is reduced: a ULD is removed, its
boxes are inserted into the other ULDs and the remaining ones are loaded into 1 or 2 new ULDs
of smaller total volume.

Constraints:
* ULD shapes: up to four corners cut in the x-z plane (a block is moved away from
  a cut when it does not fit at the anchor corner of the space);
* weight capacity of the ULD;
* allowed orientations of the boxes and fragile boxes (nothing on top of them);
* support (`--support`): blocks are generated with all their boxes fully supported.
  With `corners` (default, the criterion of Paquay et al. 2018 and Hessler et al. 2025),
  the four bottom corners of each box must rest on non-fragile boxes or on the floor and
  no box can touch the top of a fragile box. With `full`, the blocks are placed on the
  floor or on blocks whose top face is complete and without fragile boxes;
* center of gravity: at most the allowed deviation from the center in x and y (and the
  maximum height). The load of each ULD is translated to center its center of gravity, the
  evaluation of the solutions is divided by 1 + `--cog_penalty` * (relative excess of the
  deviation). (Ablation: `--cog_greedy mu` also divides the VCS value of each block by
  1 + mu * the deviation of the center of gravity after placing it; not used by default.)

Insertion of boxes into the loaded ULDs (`insert` in `main_uld.cpp`): ruin and recreate,
once with each proportion 0.3, 0.6 and 0.9: that proportion of the loaded volume of each ULD is
removed (top boxes first) and the ULDs are completed with the removed boxes and the pool, keeping
the remaining boxes fixed; accepted if the volume of the pool decreases (`--no_ruin` disables
it). Ablation: `--scratch` and `--priority` add a reload of each ULD from scratch and a reload
with priority to the boxes of the pool.

Reduction of the volume (`reduce_volume`): the ULDs are visited from the largest to the
smallest lost volume (ULD volume - loaded volume); the boxes of the ULD are inserted into the
other ULDs and, if some remain, they are loaded into 1 or 2 new ULDs (`--reduce_new`) whose
total volume is smaller than the volume of the removed ULD, trying the combinations from the
smallest total volume. The first change found is accepted (every change reduces the total
volume, and a ULD is eliminated when all its boxes are inserted) and the procedure is repeated
until no ULD can be changed. `--eliminate_only` only eliminates ULDs (no new ULDs);
`--no_eliminate` disables the reduction.

Ablation: `--corner_anchor` anchors each space at its closest corner of the ULD (as in BSG for
container loading) and loads the ULD again with the spaces anchored at the origin when the
center of gravity is out of range (`--no_regroup` disables this reload); `--no_translate`
disables the translation of the load; `--no_substructure` the substructure.

Parameters: tuned with irace (`tuning/`, result in `tuning/best.txt`) on 150 preliminary
instances of Paquay et al. (`instances/training`, different from the final ones), each one in
the three settings, 0.1 s per ULD and at most 500 blocks (fixed), 6000 runs, minimizing
(volume of the ULDs + 100 * volume
left unloaded + 150 m3 * ULDs with the center of gravity out of range) / volume of the boxes:
cog_penalty=1.4035, theta=0.2316 and the VCS parameters alpha=0.5139, beta=4.7497,
gamma=0.7727, p=0.088 (max_bl=500).

The solutions are verified box by box. With `-o file` the solution is written (one line
per box: ULD, ULD type, item (line of the instance file, from 0), position and dimensions)
and it can be checked with an independent validator:

     python3 scripts/validate_uld.py <instance.csv> instances/uld.csv <solution> [--support corners|full|none]

Results on the 300 final instances of Paquay et al. (2018) (30 per size, 10 to 100 boxes),
5 runs per configuration (mean ± standard deviation), all the solutions validated with
`validate_uld.py` (`scripts/run_all.sh`, statistics with `scripts/paper_stats.py`; runs in
`results/final/runs.tar.gz`, summary in `results/final/summary.txt`). Comparison instance by
instance with the RGS of Hessler et al. (2025) (`hessler/`), with their objective: total volume
of the ULDs (with one ULD of each type, first the loaded volume). The RGS takes 0.19-0.42 s per
instance on an Apple M2 Pro, about 0.5-1.1 s on our machine (PassMark single thread, factor
2.53), comparable to `-t 0.1`.

* paquay: instances of Paquay et al., CoG deviation 5% of the half dimensions as in Hessler et
  al. (RGS: 18648 m3, 708 ULDs, 11.2% of the ULDs with the CoG out of range);
* adapted: edge of 10 mm with vertical offset 10 mm (substructure allowed), support by area
  (min overlap 0.9, padding 10 mm), CoG deviation 10% of the half dimensions (RGS: 16843 m3,
  653 ULDs);
* one: adapted with one ULD of each type (RGS: 16166 m3, 729 ULDs, 24 boxes / 25.9 m3 not
  loaded).

| instances | -t | volume (m3) | vs RGS | better / worse | ULDs | not loaded (m3) | CoG xy | CoG z | avg time |
|---|---|---|---|---|---|---|---|---|---|
| paquay  | 0.1 s | 18106 ± 17 | -2.9% | 158 / 74.4 | 777.6 | 0 | 7.8 | 4.2 | 0.56 s |
| paquay  | 1 s   | 17317 ± 4  | -7.1% | 182.8 / 51.8 | 732.6 | 0 | 0 | 1.4 | 3.5 s |
| adapted | 0.1 s | 16203 ± 25 | -3.8% | 151.4 / 62.8 | 725.8 | 0 | 3 | 3 | 0.77 s |
| adapted | 1 s   | 15461 ± 6  | -8.2% | 184 / 44.6 | 689.0 | 0 | 0 | 0 | 5.2 s |
| one     | 0.1 s | 15773 ± 16 | -2.4% | 132.6 / 51.2 | 776.6 | 15.4 (20.0) | 17 | 11 | 0.89 s |
| one     | 1 s   | 15207 ± 6  | -5.9% | 171 / 36 | 745.0 | 12 (17.8) | 10.6 | 8.8 | 6.1 s |

Machine: two Intel Xeon E5-2620 v4 (2.1 GHz, 8 cores each), 14 runs in parallel.

Volume vs RGS according to the time limit (one run; 5 runs for 0.1 and 1 s; mean time per
instance in parentheses):

| -t | paquay | adapted | one (not loaded) |
|---|---|---|---|
| 0.02 s | +2.4% (0.24 s) | +1.2% (0.29 s) | +1.0%, 40.9 m3 (0.34 s) |
| 0.05 s | -1.3% (0.35 s) | -1.6% (0.46 s) | -1.4%, 18.3 m3 (0.55 s) |
| 0.1 s | -2.9% (0.56 s) | -3.8% (0.77 s) | -2.4%, 20.0 m3 (0.89 s) |
| 0.2 s | -5.9% (0.94 s) | -5.9% (1.32 s) | -3.5%, 10.7 m3 (1.50 s) |
| 0.5 s | -6.1% (1.92 s) | -6.7% (2.77 s) | -4.4%, 15.5 m3 (3.26 s) |
| 1 s | -7.1% (3.51 s) | -8.2% (5.15 s) | -5.9%, 17.8 m3 (6.13 s) |
| 2 s | -8.1% (6.64 s) | -8.7% (9.62 s) | -6.9%, 19.6 m3 (11.5 s) |

Ablation (0.1 s per ULD, one run; volume of the ULDs vs RGS and ULDs with the CoG out of range
in x or y; with one ULD of each type also the volume not loaded):

| variant | paquay | adapted | one |
|---|---|---|---|
| complete algorithm (mean of 5 runs) | -2.9% / 7.8 | -3.8% / 3 | -2.4% / 17, 20.0 m3 |
| no CoG penalty (`--cog_penalty 0`) | -6.0% / 460 | -3.0% / 281 | -1.0% / 326, 13.5 m3 |
| no translation (`--no_translate`) | -1.4% / 183 | -2.4% / 117 | -1.9% / 216, 36.7 m3 |
| CoG in the block evaluation (`--cog_greedy 0.5`) | -3.6% / 10 | -3.8% / 2 | -3.5% / 9, 21.6 m3 |
| spaces at the nearest corner (`--corner_anchor`) | -4.2% / 4 | -3.4% / 2 | -2.6% / 4, 26.1 m3 |
| no substructure (`--no_substructure`) | - | -2.2% / 7 | -1.6% / 23, 17.5 m3 |
| rule of Hessler et al. (`--theta -1`) | -2.5% / 7 | -3.6% / 3 | -2.7% / 18, 20.3 m3 |
| no restarts (`--restarts 0`) | - | - | -2.4% / 15, 39.6 m3 |
| insert also with reload from scratch (`--scratch`) | -2.3% / 10 | -2.9% / 3 | -2.4% / 15, 17.2 m3 |
| insert also with priority reload (`--priority`) | -2.0% / 12 | -2.0% / 2 | -2.6% / 14, 17.4 m3 |
| no insert (`--no_ruin`) | +0.2% / 10 | +0.2% / 4 | -1.2% / 21, 37.1 m3 |
| reduction with one new ULD (`--reduce_new 1`) | -1.6% / 8 | -2.5% / 2 | -2.2% / 16, 20.0 m3 |
| only elimination of ULDs (`--eliminate_only`) | +0.3% / 10 | +0.4% / 4 | -1.9% / 18, 19.3 m3 |
| no reduction (`--no_eliminate`) | +0.8% / 10 | +0.6% / 2 | -1.0% / 20, 20.0 m3 |
| basic BSG (all of the above off) | +13.9% / 90 | +18.4% / 61 | +10.1% / 72, 43.0 m3 |

```
./BSG_ULD {OPTIONS} [instance]

      -t[double]                        Timelimit of the search in each ULD
                                        (seconds, without the block generation). Default: 1
      --improve_time=[double]           Time of BSG in the insertion and reduction phases,
                                        relative to -t. Default: 0.1
      --cog_penalty=[double]            Penalty of the deviation of the center of gravity. Default: 1.4035
      --support=[string]                corners, full or none. Default: corners
      --max_bl=[int]                    Maximum number of blocks. Default: 500
      --theta=[double]                  Choice of the ULD type (negative: rule of Hessler et al.). Default: 0.2316
      --no_eliminate                    Do not reduce the volume of the ULDs
      --scratch, --priority, --no_ruin, --corner_anchor, --no_translate,
      --no_substructure                 Ablation (see above)
      --uld=[string]                    ULD file (default: uld.csv in the parent directory of the instance)
      -o[string], --output=[string]     Write the solution to this file
      -v, --verbose                     Print the loaded ULDs
```

Example:

     ./build/BSG_ULD instances/Final_instances/n50sample0.csv -t 1 -v
