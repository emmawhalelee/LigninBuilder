# LigninBuilder User's Guide.
This is the README/User Guide for LigninBuilder, a program to build 3D lignin structures from topological libraries.

## Installation

The easiest way to install the VMD plugin aspect of LigninBuilder is to download this repository, and make VMD aware of its existence. Downloading the repository can be done via `git clone` or zip file downloads available at [github](https://github.com/jvermaas/LigninBuilder). The downloaded repository is assumed to be in your home directory, which for the purposes of this example, we will call `/home/user/LigninBuilder`. Once downloaded, we append the plugin directory to the `auto_path` tcl variable within VMD, so that the plugin can be found. In the TkConsole, this is done via the following commands:
```tcl
lappend auto_path /home/user/LigninBuilder/LigninBuilderPlugin
package require ligninbuilder
```
If `package require ligninbuilder` returns a version number, the plugin is loaded successfully, and can be used to generate lignin structures from lignin libraries.
To install LigninBuilder so that it is recognized every time VMD starts, the `lappend auto_path` line should be added to your .vmdrc file, which contains configuration and settings information for VMD. For more information about .vmdrc files, please see the [VMD userguide](https://www.ks.uiuc.edu/Research/vmd/current/ug/ug.html) (scroll down towards the end of the page).

## Usage

Once installed, the typical workflow is to build the lignin structures starting from an existing library, and subsequently minimizing the structures. An example using the included library for spruce is done as follows in VMD's TkConsole:
```tcl
package require ligninbuilder
::ligninbuilder::buildfromlibrary "/home/user/LigninBuilder/libraries/Miscanthus Library.txt" Miscanthus
::ligninbuilder::minimizestructures Miscanthus namd2 "+p8"
```
This will create a directory "Miscanthus" in your current working directory, and fill it with 100 lignin structure .psf and .pdb files (numbered L0-L99).
The last line then minimizes these structures using NAMD with 8 processors, overwriting the .pdb file generated in the first step.

There are several things that can go wrong when minimizing structures.
1. Missing NAMD executable. In the example above, `namd2` is in the execution path. The [QwikMD tutorial](https://www.ks.uiuc.edu/Training/Tutorials/qwikmd/qwikmd-tutorial.pdf) demonstrates how to install NAMD such that it is put into the path.
2. Missing force field term for your topology. The parameterized force field did not take into account new terms that might be required when adjacent linkages change atom types. For the included libraries, the required terms are all found in `/home/user/LigninBuilder/extraterms-par_lignin.prm`, which was generated by finding the missing terms within the full list of psf files created. If you create a new arrangement we have not encountered, please see the `determinemissingterms.py` section to replace the `extraterms-par_lignin.prm` file.

## Creating Lignin Models from a SMILES String

If creating a full library proves difficult, we have developed a prototype implementation that uses SMILES representation of lignin as input, found in `smilesdemo`. The implementation uses some ancilliary python scripts to facilitate substructure recognition and to determine what parameters might be missing due to unforseen linkages. It requires the following python libraries to be available, most of which are installable via pip:

- [Numpy](https://pypi.org/project/numpy/)
- [ParmEd](https://pypi.org/project/ParmEd/)
- [RDKit](https://www.rdkit.org/docs/Install.html)
- [weighted_levenshtein](https://pypi.org/project/weighted-levenshtein/)

Some of the commands below are specific to Unix-style operating systems. If you'd like to follow along explicitly on a windows operating system, try the Windows Subsystem for Linux, but otherwise some of these commands will need to be translated.

We start in the `smilesdemo` directory by inspecting the `demo.smiles` file. Each line contains a [SMILES string](https://en.wikipedia.org/wiki/Simplified_molecular-input_line-entry_system), which can be visualized in many different programs. There are three different lignin polymers specified. The first step is to create the instructions to create the lignin topology. This is done using the `writepsfgen.py` script:
```bash
python writepsfgen.py
```
This will create a `psfgen.tcl` file, which VMD can use to build the `.psf` files that represent the topology for each lignin polymer. This procedure uses RDKit to find substructures that match lignin fragments previously identified. Thus, there is no error checking whatsoever, and it is the user's responsibility to check the output for sanity at the end. In VMD's tkconsole, this script can be run via:
```tcl
source psfgen.tcl
```
This will create three `.psf` files, one for each lignin. Move these three files to a new folder `output`:
```bash
mkdir output
mv L*psf output
```
Some of these polymers contain linkages and combinations that were not explicitly parameterized. We determine these parameters by analogy from the existing CGenFF and lignin force fields.
```bash
python findmissingterms.py
```
The script uses a weighted [Levenshtein metric](https://en.wikipedia.org/wiki/Levenshtein_distance) to penalize how "different" missing parameters are from existing parameters. The algorithm chooses values for these missing parameters by analogy from the closest set of existing parameters, and creates supplementary parameters in the file `extraparameters.prm`. Next, we assign coordinates to the psf files, again in the VMD TkConsole.
```tcl
package require ligninbuilder
::ligninbuilder::makelignincoordinates output .
```
The first argument is the input directory, and the second is the output directory relative to the input directory. The '.' signifies that the input and output directories should be the same. The `extraparamters.prm` file can be used as input to `minimizestructures`:
```tcl
package require ligninbuilder
::ligninbuilder::minimizestructures output namd2 "+p8" "parameters extraparameters.prm \n parameters toppar/par_all36_cgenff.prm \n"
```
The last argument is added into the NAMD configuration file right before minimization, and in this case serves to provide NAMD with the extra parameters it requires for the spirodienone. The minimized the structures will be in the `output` directory, and ready for further simulation.
