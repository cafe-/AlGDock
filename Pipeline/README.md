# THE PIPELINE

Running AlGDock requires files that describe the receptor, ligand, and complex. The required files are summarized in this spreadsheet: https://docs.google.com/spreadsheets/d/1KtO6mAUvGdnT1MyYZyLgerwiEUFGAhs4j2DzOL0k3aY/edit?usp=sharing

This directory contains scripts useful for setting up AlGDock calculations. The workflow can be summarized as follows:

![workflow](https://github.com/CCBatIIT/AlGDock/blob/master/Pipeline/images/workflow.jpeg)

Depending on the desired BPMFs, there are multiple possible entry points to the workflow. The instructions below assume minimal available information about the system.

The system preparation pipeline may be divided into two overall stages: calculations with UCSF DOCK 6 and then with AlGDock.

To go through this pipeline, it is helpful to define the environment variable $ALGDOCKHOME

From the $TARGET directory, if you run `python $PIPELINE/count.py` it will tell you how many receptors, complexes, and completed calculations there are.

# UCSF DOCK 6

## Prepare the receptor structures

### 1. Get the reference sequence, reference structure, and aligned chains

For a specified target, we need to find the reference sequence of amino acids. The following databases are recommended:

* http://www.pdb.org/ - the Protein Data Bank (PDB), which contains publicly available protein structures
* http://www.uniprot.org/ - contains publicly available protein sequences

If you blast the target name in the PDB, you can find a link to the UniProt sequence. The UniProt entry will list other crystal structures based on the same sequence, and the amino acid numbers present in the structures. This can be a starting point for how long the sequence should be.

When you have a starting sequence, create `$TARGET/receptor/1-search/seq.ali`. Its format should be similar to the following:

>\>P1;P31749
sequence:P31749:::::::0.00: 0.00
RVTMNEFEYLKLLGKGTFGKVILVKEKATGRYYAMKILKKEVIVAKDEVAHTLTENRVLQ
NSRHPFLTALKYSFQTHDRLCFVMEYANGGELFFHLSRERVFSEDRARFYGAEIVSALDY
LHSEKNVVYRDLKLENLMLDKDGHIKITDFGLCKEGIKDGATMKTFCGTPEYLAPEVLED
NDYGRAVDWWGLGVVMYEMMCGRLPFYNQDHEKLFELILMEEIRFPRTLGPEAKSLLSGL
LKKDPKQRLGGGSEDAKEIMQHRFFAGIVWQHVYEKKLSPPFKPQVTSETDTRYFDEEFT
AQMITITPPDQDDSMECVDSERRPHFPQFSYSASGTA*

For the next step, you will need `pdball.bin`. This file can be created from `pdball.pir`, a database that can be downloaded from the modeller web site. To do this conversion, change into the directory with `pdball.pir` and run,

    mod9.18 $PIPELINE/pdball_pir2bin.modeller.py

Once you have `pdball.bin`, you should change to the `$TARGET/receptor/1-search/` directory and run,

    mod9.18 $PIPELINE/profile.modeller.py $PDBALL_BIN

where $PDBALL_BIN refers to the path of `pdball.bin`. This will give you,

 - $TARGET/receptor/1-search/profile.prf
 - $TARGET/receptor/1-search/profile.ali

From `$TARGET/receptor/1-search/`, you should run, 

`python $PIPELINE/analyze_profile.py`

which will give you histograms of the sequence identity,

 - $TARGET/receptor/1-search/figures/hist_seq_id.png
 - $TARGET/receptor/1-search/figures/hist_seq_id_selected.png

and display the sequences that are present in the selected chains. The sequences will be sorted in descending order of the number of sequence identity and the number of equivalent positions. 

Selected chains have
* a sequence identity greater than `min_seq_identity` (90 by default) and
* at least `min_equivalent_positions` (1 by default) equivalent positions.

The defaults can be modified in `$TARGET/receptor/search_options.py`
using, e.g.,

>min_seq_identity = 100
min_equivalent_positions = 50

The reference sequence need not be the exact length of the chains; it can be shorter but should not be much longer (because additional residues will need to be modeled). However, it should not be too short because a truncated protein may not contain the binding site, or the missing residues may influence binding properties. 

If there are enough structures, then it is preferable for `min_seq_identity=100`. `min_seq_identity` can be reduced if necessary.

A reference structure should be reasonably complete and have high homology with the reference sequence. If there are no chains that meet this criterion, then we will not be able continue the docking and free energy calculation. If there are multiple chains, then the choice of reference structure is somewhat arbitrary.

We should repeat the above procedure until we are happy with the reference sequence, minimum sequence identity, and the reference structure.

Then we can edit (or create) `$TARGET/receptor/search_options.py` to describe the rationale behind our choices. For example,

>\# 3CQW is the reference structure from DUD-E
\>sp|P31749|144-448 is the reference sequence
\# According to UniProt, a number of crystal structures
\# contain P31749 amino acids 144-480.
\# However, many of them appear to be missing a loop after around 448.
\# The remaining residues are distal to the binding site.
>
>\# This will be loaded from seq.ali
>sequence = ''
>
>\# From DUD-E
ref_pdb_id = '3cqw'
ref_chain_id = 'A'
ref_res_id_range = (144,448)
>
>\# Do not exclude any data
\exclude = []
\# There are 17 chains with 100% sequence identity
min_seq_identity = 100

From `$TARGET/receptor/1-search/`, run

    python $PIPELINE/align3d.py 2>&1 | tee align3d.log

which will populate the directories

 - $TARGET/receptor/1-search/pdb_original/
 - $TARGET/receptor/1-search/chains_original/
 - $TARGET/receptor/1-search/chains_align/

and create 

 - $TARGET/receptor/1-search/reference.pdb
 - $TARGET/receptor/1-search/mappings.pkl
 - $TARGET/receptor/1-search/selected.pca.npz
 - $TARGET/receptor/1-search/align3d.log

reference.pdb should:
* be the same length as the reference sequence
* have chain names
* have standard PDB residue names

### 2. Measure the binding site

Create `$TARGET/receptor/xtal_ligand_selection.py`.
This file specifies ligands to be included or excluded from the binding site analysis, e.g.

>\# Ligands to include, specifying (pdb_id, chain_id, rename, res_id)
>
>selection = []
>
>\# Ligands to exclude, specifying (pdb_id, chain_id, rename, res_id)
>
>exclude = []
>
>\# Ligands to exclude, only specifying resname
>
>exclude_resname = ['HOH']
>
>\# The minimum number of atoms in a ligand
>
>minimum_natoms = 8
>
>\# Ligands will be clustered using hierarchical clustering with clustering_method = (method, number_of_clusters) or not clustered at all. If clustering is None, then a plot will be generated showing a number of possible clusterings. If is specified, then colors of the clusters will be shown.
>
> clustering_method = None
>
>\# If site_R is None, the site radius will come from rounding up the distance from the site center to the closest ligand center of mass
>
>site_R = None

From $TARGET/receptor/2-binding_site/, run

    python $PIPELINE/measure_binding_site.py

which will create 

 - $TARGET/receptor/2-binding_site/figures/clusters.png 
 - $TARGET/receptor/2-binding_site/measured_binding_site.py
 
and populate

 - $TARGET/receptor/2-binding_site/ligands_aligned/
 - $TARGET/receptor/2-binding_site/ligands_trans/
 - $TARGET/receptor/2-binding_site/receptor_trans/
 - $TARGET/receptor/2-binding_site/complexes_trans/

The ligand_trans directory contains `com.py` and `com_box.pdb`. If opened in UCSF chimera, they will show all the ligand centers of mass and a box that surrounds them, respectively. 

You should check that `com_box.pdb` has a reasonable size. If it is too small, you will not be able to dock any ligands in the binding site. You may want to go back and modify `$TARGET/receptor/search_options.py` 
so that more crystal structures are included. Alternatively, you can manually modify `$TARGET/receptor/2-binding_site/measured_binding_site.py` to encompass a larger region.

Iterate editing of `xtal_ligand_selection.py` and running `measure_binding_site.py` until the desired binding site is specified.

### 3. Build homology models

The above steps are relatively fast and are suitable for your laptop. 
It may be worth building homology models on the group cluster.

From `$TARGET/receptor/3-models/`, run 

    python $PIPELINE/run_homology_model.py --reference

The --reference option will only build the homology model for the reference chain.
It can be omitted to build models for all the chains.

### 4. Prepare models for UCSF DOCK

From `$TARGET/receptor/3-models/`, run 

    python $PIPELINE/prep_receptor_for_dock.py pdb_noH/$REFERENCE.pdb

You can see how the ligands fit into the spheres from `$TARGET/receptor` by running

    open 2-binding_site/ligand_trans/*.pdb dock_in/*.sph 

## Prepare the ligand models

For every chemical library, create `$LIGAND/$LIBRARY.ism`, text files that have SMILES strings at the beginning of every line. For example, the file names can be,

 - DUD-E.active.ism
 - DUD-E.decoy.ism
 - BindingDB.active.ism
 - BindingDB.inactive.ism
 - BindingDB.DUD-E.decoys.ism
 - xtal_ligands.ism

To create 3D models and parameterize the ligands, start in `$LIGAND/` on the group cluster or on the open science grid submit node and run 

    python $PIPELINE/run_prep_ligand_for_dock.py $LIBRARY.ism $USE_OPENEYE

If $USE_OPENEYE is 'N', the script will use balloon for conformer generation and chimera and antechamber for charges. Chimera may assign too many protons to the ligands, so be careful.

If $USE_OPENEYE is 'Y', the script will use the OpenEye toolkit. The Minh group has a free academic license, so only use this option for academic projects without commercial implications. Ask David if you are unsure.

## Molecular Docking with Anchor and Grow

To dock all ligands to all prepared receptor models, start in `$TARGET/dock6/` on the group cluster or on the open science grid submit node and run

    python $PIPELINE/run_anchor_and_grow.py

To convert the dock6 output to netcdf format, start in `$TARGET/` on the group cluster (has not been tested on OSG) and run

    python $PIPELINE/run_dock6_to_nc.py

# AlGDock Binding PMFs

After completing the dock6 pipeline, the system may be prepared for AlGDock.

## Prepare receptor grids

From `$TARGET/receptor/AlGDock_in/`, execute

python $PIPELINE/run_alchemicalGrids.py

which will prepare the Lennard Jones and Poisson-Boltzmann grids.

## Select and prepare ligand and complex files

It is probably unnecessary to calculate a binding PMF for every single molecule from docking. The next step is to select the most appropriate ligands.

From `$TARGET/`, execute

    python $PIPELINE/analyze_scores.py 

which will generate analysis/_dock6_scores.txt, and files corresponding to each library.

To see the options for systems to prepare, from `$TARGET/`, execute

    python $PIPELINE/run_prep_ligand_and_complex_for_AlGDock.py --help

To prepare the 50 top-scoring active molecules and 150 top-scoring decoys, execute:

    python $PIPELINE/run_prep_ligand_and_complex_for_AlGDock.py --complex_list analysis/DUDE.active_dock6_scores.txt --prepare_first 50
    python $PIPELINE/run_prep_ligand_and_complex_for_AlGDock.py --complex_list analysis/DUDE.decoy_dock6_scores.txt --prepare_first 150

To prepare 250 ligands with the string '.active.' in the library,

    python $PIPELINE/run_prep_ligand_and_complex_for_AlGDock.py --library_requirement .active. --prepare_number 250

## Run AlGDock

To run AlGDock, from `$TARGET/AlGDock/`, execute

    python $PIPELINE/run_AlGDock.py

on the group cluster or on the open science grid submit node.

## If relevant, analyze binary classification

After the BPMF calculations are complete, running

    python $PIPELINE/analyze_scores.py

again will generate ROC plots based on BPMF estimates.

% TODO: Procedure for dimers and other multimers, e.g. HIV protease
% TODO: Move $LIGAND to a separate directory.
