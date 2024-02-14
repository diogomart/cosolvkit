[![made-with-python](https://img.shields.io/badge/Made%20with-Python-1f425f.svg)](https://www.python.org/) [![GitHub license](https://img.shields.io/github/license/Naereen/StrapDown.js.svg)](https://github.com/Naereen/StrapDown.js/blob/master/LICENSE)

# CosolvKit
The python package for creating cosolvent system

## Prerequisites

You need, at a minimum (requirements):
* python 3
* numpy 
* scipy
* RDKit
* ambertools
* parmed
* MDAnalysis
* griddataformats
* MDTraj
* OpenMM
* OpenMM-ForceFields
* OpenFF-toolkit
* PDBFixer
* espaloma
* pymol

## Installation
I highly recommand you to install the Anaconda distribution (https://www.continuum.io/downloads) if you want a clean python environnment with nearly all the prerequisites already installed. To install everything properly, you just have to do this:
```bash
$ conda create -n cosolvkit -c conda-forge python=3 numpy scipy mkl \
rdkit ambertools parmed mdanalysis griddataformats openmm openmmforcefields \
openff-toolkit pdbfixer mdtraj espaloma pymol-open-source
$ conda activate cosolvkit
```

Finally, we can install the `CosolvKit` package
```bash
$ git clone https://github.com/forlilab/cosolvkit
$ cd cosolvkit
$ pip install -e .
```

## Quick tutorial

The script `create_cosolvent_system.py` provide all the necessary tools to build a cosolvent system and optionally run an MD simulation with standard setup.
The main entry point of the script is the file `config.json` where all the necessary flags and command line options are specified.

| Argument          | Description                                           | Default value   |
|:------------------------|:-------------------------------------------------|:----------------|
|cosolvents            | Path to the json file containing the cosolvents to add to the system. | no default |
|forcefields           | Path to the json file containing the forcefields to use. | no default |
|md_format             | Format to use for the MD simulations and topology files. Supported formats: [OPENMM, AMBER, GROMACS, CHARMM] | no default |
|receptor              | Boolean describing if the receptor is present or not. | no default |
|protein_path          | If receptor is `true` this should be the path to the protein structure. | no default |
|clean_protein         | Flag indicating if cleaning the protein with `PDBFixer` | TRUE |
|keep_heterogens       | Flag indicating if keeping the heterogen atoms while cleaning the protein. Waters will be always kept. | FALSE |
|variants              | List of residues for which a variant is requested (different protonation state), `None` for the rest of the residues. | empty list |
|add_repulsive         | Flag indicating if adding repulsive forces between certain residues or not. | FALSE |
|repulsive_resiudes    | List of residues for which applying the repulsive forces. | empty list |
|solvent_smiles        | Smiles string of the solvent to use. | H2O |
|solvent_copies        | If specified, the box won't be filled up with solvent, but will have the exact number of solvent molecules specified. | no default |
|membrane              | Flag indicating if the system has membranes or not. | FALSE |
|lipid_type            | If membrane is TRUE specify the lipid to use. Supported lipids: ["POPC", "POPE", "DLPC", "DLPE", "DMPC", "DOPC", "DPPC"] | "POPC" |
|lipid_patch_path      | If the lipid required is not in the available, it is possible to pass a pre-equilibrated patch of the lipid of interest. | no default |
|cosolvent_placement   | Integer deciding on which side of the membrane to place the cosolvents. Available options: [0 -> no preference, 1 -> outside, -1 -> inside] | 0 |
|waters_to_keep        | List of indices of waters of interest in a membrane system. | no default |
|radius                | If no receptor, the radius is necessary to set the size of the simulation box. | no default |
|output                | Path to where save the results. | no default |
|run_md                | Flag indicating if running the md simulation after creating the system or not. | FALSE |

1. **Preparation**
```bash
$ python create_cosolvent_system.py -c config.json
```

3. **Run MD simulations**
If you don't want to setup your own simulation, we provide a standard simulation protocol using `OpenMM`

```python
from cosolvkit.simulation import run_simulation

print("Running MD simulation")
start = time.time()
# Depending on the simulation format you would pass either a topology and positions file or a pdb and system file
run_simulation(
                simulation_format = simulation_format,
                topology = None,
                positions = None,
                pdb = 'system.pdb',
                system = 'system.xml',
                warming_steps = 100000,
                simulation_steps = 6250000, # 25ns
                results_path = results_path, # This should be the name of system being simulated
                seed=None
    )
print(f"Simulation finished after {(time.time() - start)/60:.2f} min.")
```

4. **Analysis**
```python
from cosolvkit.analysis import Report
"""
Report class:
    log_file: is the statistics.csv or whatever log_file produced during the simulation.
        At least Volume, Temperature and Pot_e should be reported on this log file.
    traj_file: trajectory file
    top_file: topology file
    cosolvents_file: json file describing the cosolvents

generate_report_():
    out_path: where to save the results. 3 folders will be created:
        - report
            - autocorrelation
            - rdf
    analysis_selection_string: selection string of cosolvents you want to analyse. This
        follows MDAnalysis selection strings style. If no selection string, one density file
        for each cosolvent will be created.

generate_pymol_report()
    selection_string: important residues to select and show in the PyMol session.
"""
report = Report(log_file, traj_file, top_file, cosolvents_file)
report.generate_report(out_path=out_path, analysis_selection_string="")
report.generate_pymol_reports(report.topology, 
                              report.trajectory, 
                              density_file=report.density_file, 
                              selection_string='', 
                              out_path=out_path)
```

## Add centroid-repulsive potential

To overcome aggregation of small hydrophobic molecules at high concentration (1 M), a repulsive interaction energy between fragments can be added, insuring a faster sampling. This repulsive potential is applied only to the selected fragments, without perturbing the interactions between fragments and the protein. The repulsive potential is implemented by adding a virtual site (massless particle) at the geometric center of each fragment, and the energy is described using a Lennard-Jones potential (epsilon = -0.01 kcal/mol and sigma = 12 Angstrom).

Luckily for us, OpenMM is flexible enough to make the addition of this repulsive potential between fragments effortless (for you). The addition of centroids in fragments and the repulsive potential to the `System` holds in one line using the `add_repulsive_centroid_force` function. Thus making the integration very easy in existing OpenMM protocols. In this example, we are adding repulsive forces between `BEN` and `PRP` molecules.

```python
from cosolvkit.cosolvent_system import CosolventSystem

cosolv = CosolventSystem(cosolvents, forcefields, simulation_format, receptor_path, radius=radius)
# build the system in water
cosolv.build(neutralize=True)
cosolv.add_repulsive_forces(resiude_names=["BEN", "PRP"])
```

## List of cosolvent molecules
Non-exhaustive list of suggested cosolvents (molecule_name, SMILES string and resname):
* Benzene 1ccccc1 BEN
* Methanol CO MEH
* Propane CCC PRP
* Imidazole C1=CN=CN1 IMI
* Acetamide CC(=O)NC ACM
* Methylammonium C[NH3+] MAM
* Acetate CC(=O)[O-] ACT
* Formamide C(=O)N FOM
* Acetaldehyde CC=O ACD

## Citations
* Ustach, Vincent D., et al. "Optimization and Evaluation of Site-Identification by Ligand Competitive Saturation (SILCS) as a Tool for Target-Based Ligand Optimization." Journal of chemical information and modeling 59.6 (2019): 3018-3035.
* Schott-Verdugo, S., & Gohlke, H. (2019). PACKMOL-memgen: a simple-to-use, generalized workflow for membrane-protein–lipid-bilayer system building. Journal of chemical information and modeling, 59(6), 2522-2528.
