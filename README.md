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
* `results/server/`: results of the experiments of the paper (all the runs in `runs.tar.gz`).

`BSG_ULD` solves the air cargo palletization problem with the instances of
Paquay et al. (2018) (`instances/`): all the boxes must be loaded
into unit load devices (ULDs) of six types (Boeing 777), minimizing the number of
ULDs. The ULDs are loaded one by one with BSG; the next ULD is selected as in
Hessler et al. (2025) and, at the end, the boxes of the least utilized ULD are reloaded into
a smaller ULD when possible.

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
* center of gravity: at most the allowed deviation from the center in x and y. The
  load of each ULD is translated to center its center of gravity, the evaluation of the
  solutions is divided by 1 + p * (relative excess of the deviation), and if the deviation
  is exceeded, the ULD is loaded again anchoring all the spaces at the origin
  (the load is grouped and it can be centered); this is also tried when the ULD is less than
  50% utilized and boxes remain. In the evaluation of the blocks, the VCS value is divided by
  1 + `--cog_greedy` (default 0.5987) * the deviation of the center of gravity after placing the
  block (and translating the load).

With a limited number of ULDs (`--uld_count`), the boxes that were not loaded are inserted
into the loaded ULDs before any ULD is eliminated, by ruin and recreate: a proportion (0.3,
0.6, 0.9) of the volume of each ULD is removed (boxes with nothing on top, from the top) and
the ULDs are loaded again keeping the remaining boxes fixed and adding the unloaded ones.

After the sequential loading, the program tries to eliminate ULDs (`--no_eliminate`
disables it): the boxes of a ULD (from the least utilized) are freed and the other ULDs
are loaded again with their boxes and the freed ones; the boxes that are not loaded pass
to the next ULD. The ULD is eliminated if all the freed boxes are loaded.

Parameters: tuned with irace (`tuning/`) on 150 preliminary instances of Paquay
et al. (`instances/training`, different from the final ones), 0.1 s per ULD, minimizing
the number of ULDs + 5 * the ULDs with the center of gravity out of range:
cog_penalty=12.6619, cog_greedy=0.5987, max_bl=500 and the VCS parameters alpha=1.2335,
beta=0.2389, gamma=0.2243, p=0.0574.

The solutions are verified box by box. With `-o file` the solution is written (one line
per box: ULD, ULD type, item (line of the instance file, from 0), position and dimensions)
and it can be checked with an independent validator:

     python3 scripts/validate_uld.py <instance.csv> instances/uld.csv <solution> [--support corners|full|none]

Results on the 300 final instances of Paquay et al. (2018) (30 per size, 10 to 100 boxes),
5 runs per configuration (mean ± standard deviation), all the solutions validated with
`validate_uld.py`, scripts in `scripts/` (`run_uld.sh`, `summarize.py`).
Comparison instance by instance with Hessler et al. (2025) (`hessler/hessler2024_table*.json`):
better means more boxes loaded or, with the same boxes, fewer ULDs.

* paquay: instances of Paquay et al. (Hessler et al.: 708 ULDs, 11.2% of the ULDs with the
  CoG out of range; Paquay et al. 2018: 1010 ULDs);
* adapted: edge of 10 mm with vertical offset 10 mm (substructure allowed), support by area
  (min overlap 0.9, padding 10 mm), CoG deviation 10% of the half dimensions (Hessler et al.:
  653 ULDs);
* one: adapted with one ULD of each type (Hessler et al.: 729 ULDs, 24 boxes not loaded).

CoG xy: ULDs with the CoG out of the allowed deviation in x or y; CoG z: CoG above the maximum
height (Paquay et al. 2016, not considered by Hessler et al.).

| instances | -t (per ULD) | ULDs | boxes not loaded | better / worse | CoG xy | CoG z | avg time |
|---|---|---|---|---|---|---|---|
| paquay  | 0.1 s | 617.0 ± 0.0 | 0 | 91.0 / 0 | 0 | 0 | 0.62 s |
| paquay  | 1 s   | 597.8 ± 0.4 | 0 | 111.2 / 1 | 1 | 0 | 5.4 s |
| adapted | 0.1 s | 598.0 ± 0.0 | 0 | 56.0 / 1 | 2 | 0 | 1.2 s |
| adapted | 1 s   | 576.0 ± 0.0 | 0 | 77.0 / 0 | 1 | 0 | 10.3 s |
| one     | 0.1 s | 706.0 ± 0.0 | 22.0 ± 0.0 | 42.0 / 22.0 | 3 | 0 | 1.6 s |
| one     | 1 s   | 675.8 ± 0.4 | 21.0 ± 0.0 | 64.4 / 20.2 | 2 | 1 | 13.0 s |

Machine: two Intel Xeon E5-2620 v4 (2.1 GHz, 8 cores each), 14 runs in parallel
(`scripts/run_all.sh`; results in `results/server`).

Lower bound (`lower_bound_uld.py`: total volume / largest ULD, total weight / largest capacity,
cliques of incompatible boxes): the number of ULDs is proven optimal in 140 (0.1 s) and 157
(1 s) of the 300 paquay instances (Hessler et al.: 80).

Ablation (0.1 s per ULD, one run, one component turned off; ULDs and ULDs with the CoG out of
range in x or y):

| without | paquay | adapted | one (ULDs, boxes not loaded) |
|---|---|---|---|
| (complete algorithm) | 617 / 0 | 598 / 2 | 706, 22 |
| CoG in the evaluation of the blocks (`--cog_greedy 0`) | 625 / 2 | 607 / 6 | 713, 29 |
| CoG penalty in the value (`--cog_penalty 0`) | 615 / 193 | 590 / 296 | 679, 17 (353 CoG) |
| translation of the load (`--no_translate`) | 624 / 84 | 602 / 84 | 723, 28 (158 CoG) |
| grouped reload (`--no_regroup`) | 622 / 25 | 604 / 31 | 726, 39 |
| elimination of ULDs (`--no_eliminate`) | 620 / 0 | 599 / 2 | 719, 22 |
| substructure (`--no_substructure`) | - | 618 / 1 | 731, 28 |
| all the previous except the CoG penalty and the translation | 630 / 26 | 634 / 61 | 804, 39 |

```
./BSG_ULD {OPTIONS} [instance]

      -t[double]                        Timelimit of the search in each ULD
                                        (seconds, without the block generation). Default: 1
      --cog_penalty=[double]            Penalty p of the deviation of the center of gravity. Default: 12.66
      --support=[string]                corners, full or none. Default: corners
      --max_bl=[int]                    Maximum number of blocks. Default: 500
      --no_eliminate                    Do not try to eliminate ULDs
      --uld=[string]                    ULD file (default: uld.csv in the parent directory of the instance)
      -o[string], --output=[string]     Write the solution to this file
      -v, --verbose                     Print the loaded ULDs
```

Example:

     ./build/BSG_ULD instances/Final_instances/n50sample0.csv -t 1 -v
