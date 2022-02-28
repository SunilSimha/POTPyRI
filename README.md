# POTPyRI: Pipeline for Optical/infrared Telescopes in Python for Reducing Images
Data reduction pipeline for imaging from large aperture telescopes

## Usage
If you use this code, please reference Paterson et al. 2022 (in prep.)

## Installation
To install run **git clone https://github.com/CIERA-Transients/Imaging_pipelines.git**. It is recommended that you create an environment for the pipeline to run in. There are two files available which you can use:
1) pipeline_environment.txt contains a explicit list of commands to install the needed packages. To use this file run **conda create --name myenv --file pipeline_environment.txt**, where myenv is the name of the environment.
2) pipeline_environment.yml contains the environment file. To use this file, first edit the *prefix* paramater in the file, and then run **conda env create -f pipeline_environment.yml**, where the name of the environment is set by the first line in the file.
Both these files were created from an environment on a Linux-64bit OS.

## Supported Instruments
- GMOS (Gemini)
- MOSFIRE (Keck)
- MMIRS (MMT)
- BINOSPEC (MMT)
- UKIRT (WFCAM)
- LRIS (Keck)
- DEIMOS (Keck)

## Running
To run the pipeline run:

**python main_pipeline.py instrument data_path**,

where **instrument** is the name of the instrument you wish to reduce data from (see **Supported Instruments**), and **data_path** is the full path of the data you wish you reduce (including the last "/" e.g. /data/). The **instrument** and **data_path** parameters are needed to run the pipeline and the pipeline will exit if missing these. Additional parameters that can be set when running the pipeline are discussed in **Pipeline parameters**.

The pipeline runs mostly automatic, but will prompt the user to check files at various points to allow edits, to discontinue reduction on a particular target if the user chooses, manually redo astrometry, and add user provide zero points during manual aperture photometry. The pipeline will display prompts or instructions on the terminal in green or red. For more details on each of these sections, including how the pipeline performs each task, how you can use the scripts outside the pipeline, and particular instructions, see below. Quality checks and things to pay attention to before, during and after running the pipeline are also described. Special cases as well as common errors are discussed at the end.

All processed data will be written out in the **data_path** under the **red** folder. A description of the outputs can be found in **Outputs**.

If you re-run the pipeline after it has already been run, it will read in the files using the file list that was created previously. To ``reset" the pipeline to the original state, please see details under **Special cases**.

## Flow diagram
![plot](./images/Pipeline_flow_diagram.pdf)

## Pipeline parameters
### main_pipeline.py
On top of the 2 needed parameters needed by the main pipeline, there are additional parameters that the user can set to``True" which affects how the pipeline runs, and what it does with the data.
### -- use_dome_flats
Option to use dome flats instead of science files for NIR images. Not recommended.
### -- skip_red
Skip reduction when possible i.e. creating master flats, individual images.
### -- target
Option to reduce a particular target if there are multiple targets in a folder. The pipeline will check if the string entered by the user is in the target name generated by the pipeline (this is case sensitive). This target name is read in from the header and is based on the target_keyword from the setting file, and will also include the filter and setup e.g. target_filter_amp_binning. Therefore, if you wish you reduce a particular target in a particular filter and setup, you will need to include all this e.g. target_filter will reduce all targets in that filter with different setups (some target names in the header have the filter already included resulting in double _filter in the target name generated by the pipeline). All other target where no match is found will be ignored. If your target is not found, check the file list for the target name read in by the pipeline.
### -- proc
Option to use not use the "_proc" images from BINOSPEC. It is recommended to use the "_proc" images (set as the default) due to the lack of WCS in the raw images.
### --cal_path
Calibration path if different from the default path in the setting file.
### -- phot
Option to manually perform aperture photometry on the final stack. You will be prompted in the terminal and plots will shall you the results and allow you to make changes.
### -- reset
Option to redo the file sorting and remake the file list.

## Outputs
All outputs from the pipeline are written out to the **red** folder. Calibration files such as the master bias, flat and dark are saved using mbias, mflat and mdark, along with additional information such as the filter, amp and binning in the file names. Likewise, fringe maps are called as such with the filter and setup.

Processed science files keep their original file name with the addition of *_* and a descriptor. Files with *_bkg* contain the background subtracted from the image. Files with *_sky* contain the sky (for NIR images) subtracted from the image. Files with Files with *_red* are the final processed images used in the stack. For instruments such as BINOSPEC with multiple images will have an addition *_left/_right* etc.

Stacks are called by the target name which includes the name of the target and the filter and setup of the observation. The *.cat* extension is the SExtractor file of the stack used to get star positions to solve the WCS of the image. Stack which have an updated WCS (added either automatically or manually through the pipeline) have *_wcs* added to the stack name. This *_wcs* stack is then used for the processing of the reduction, unless the pipeline failed to calculate a WCS solution. Alongside the *_wcs* fits image, the pipeline also produces two png plots to show the errors on the astrometry. The first, with extension *.png*, shows the wcs stack with the stars used to calculate the WCS solution (red) and the corresponding catalog stars based on the WCS of the image (green - GAIA DR3 is the default used by the automatic calculation); and the second, with the *.hist.png* extension, gives a scatter and histogram plots of the delta RA and Dec errors on these stars. A region file of these stars is given by the *.wcs*.reg extension. A table of the original GAIA stars are saved under the *.gaia* extension. If you choose to manually solve for the WCS, the pipeline will also produce *.xy/radec* files containing the x and y; and RA and Dec positions of the stars to be used in the WCS solution.

From the automatic PSF photometry, multiple output images and files are written out. The *.epsf.png* and *.psf.fits* files save the PSF determined for the stack. The stars used to determine the PSF is written out to the *psf.stars* files while the full catalog of stars for which PSF photometry is performed on is contained within the *.pcmp* file. The stars used to determine the zero point are saved in the *.zpt* file, with a region file of these stars is saved as *.zpt.reg*. The file *.png* of the stack shows the calculated PSF photometry vs. the catalog photometry of the stars used to calculate the zero point, and the corresponding zero pointed calculated.

## File list
The pipeline will sort through all the fits with the correct format for a given instrument (given by the **raw_format** function in the setting file) and create a file list with the file type, target name, exposure time, observation time, and instrument setup such as number of amps and binning. Once this file has been made, the user can review and edit it if there are any errors or additional files that need to be removed (due to suggestions from the observing log or known bad files etc.). If this file is present when the pipeline is run, the pipeline will read in the files and their information using this instead of re-sorting the files. To remake the file list, the pipeline needs to be run with the **--reset** option.

## Aligning
Images are aligned using unique star quads in the **align_quads.py** script. This script can be used as a stand alone script to align images. To do so, import align_quads in python (you will need to be in the Imaging_pipeline directory or have the path in your PYTHONPATH) and call the align_stars function:

**align_quads.align_stars(image_list,instr)**,

where **image_list** is the list of images to align (e.g. ['/fullpath/image1.fits','/fullpath/image2.fits']) and **instr** is the name of the instrument used in the main pipeline. This function will return the names of the aligned images (written out with *_aligned*) and the aligned arrays should you wish to work directly with the arrays e.g. combine the aligned images using ccdproc's combine.

## Astrometry
The pipeline uses Gaia stars from DR3 to automatically calculate the WCS on the stacked image. First, the pipeline uses unique star quads to calculate the initial shift on the WCS already in the header. The pipeline then does a star match between all the stars and the Gaia catalog to calculate the higher order polynomial distortions. This is done with the **solve_wcs.py** script, which can be used on images outside the pipeline to calculate the WCS. To do so, import solve_wcs in python (you will need to be in the Imaging_pipeline directory or have the path in your PYTHONPATH) and call the solve_wcs function:

**solve_wcs.solve_wcs(image,instr)**

where **image** is the name of the image to want to calculate astrometry for, and **instr** is the name of the instrument used in the main pipeline. The function will return the astrometric error on the solution found, and will write out the new image with the updated astrometry to a fits file with *_wcs*.

In the case the automatic WCS solution is not good enough, the pipeline has the option to manually select the stars that will be used to calculate the WCS solution. After the automatic WCS solution has run, the pipeline will ask:
*Please review the WCS plots and errors. Do you wish to manually redo wcs (yes or no)?*
If you select "yes", the pipeline will ask which catalog you wish to use, with various options printed out for you to choose from (check which catalog you wish to use before starting). This will be the catalog from which you will select stars for the WCS solution. An window will then pop up showing the stacked image, with the sources detected highlighted by red circles, and the catalog stars (whose positions are based on the WCS solution in the header) plotted with green circles. This is an interactive plot which allows the user to pick these circles to build a list of star x,y positions and their corresponding sky (RA,Dec) positions. The user must first select a detected star on the image (red circle), followed by the corresponding catalog match given be the green star. Each time you select a red or green circle, a message will appear on the terminal confirming the selection. For every red circle that is selected, a green circle must also be selected. As many stars as possible should be selected in order to produce as accurate WCS solution as possible. The pan and zoom buttons can be used to zoom in and move around the image, but must be de-selected before you can select stars. When the plot first appears, the terminal will display the instruments: 

  **Displaying interactive plot to select star for WCS solution.**
  
  **First select star (red) and corresponding catalog match (green).**
  
  **A message will confirm the selection of the star in the terminal.**
  
  **Note: star selection is turned off in zoom/pan mode (you need to de-select the mode in order to select stars).**
  
  **Close figure when finished.**

When a source is selected, the terminal will display the message:

**You selected x, y position: x, y**

And when a catalog star is selected, the terminal will display the message:

**Catalog star: RA = ra, Dec = dec**

Once you have finished selecting the stars, close the figure to continue the process. The pipeline will then save these positions to file (x and y to *.xy* and RA and Dec to *.radec*) and ask the user to review and edit the files in the case there was an error (e.g. a x,y position of a star was selected twice but only one entry from the catalog was selected; or if you wish to add more stars to the list; or manually enter all the star to be used for the WCS solution etc.). The pipeline will then use these stars to calculate the WCS solutions, report the astrometric error, make the error plots and create the *_wcs* file. The pipeline will once again ask you if you are satisfied with the astrometric error and the procedure can be repeated if needed (to select more stars etc.).

Just like **solve_wcs** (the automatic WCS solution), **man_wcs** (the manual WCS solution) can be run outside the pipeline on an image in order to solve the WCS on an image. This can be done by running:

**solve_wcs.man_wcs(instr, stack, cat, cat_stars_ra, cat_stars_dec)**,

where **instr** is the name of the instrument used in the main pipeline, stack is the file name of the image you wish to solve the WCS on, cat is the name of the catalog as given by the pipeline options, and **cat_stars_ra** and **cat_stars_dec** are the RA and Dec of stars from the chosen catalog queried through python (to see how this can be done, see the main_pipeline code ~lines 400).

## Photometry
### Automatic PSF photometry
The pipeline will perform automatic PSF photometry of sources in the stacked image. Firstly, the pipeline will detect sources within the image. Then, based on a number of cuts, such as roundness, FWHM, and brightness; the pipeline will create a list of bright stars which it will use to calculate the PSF. Once the pipeline has determined the PSF, it will then calculate PSF photometry for all the originally extracted sources and write them to a fits table saved as *.pcmp*. After the PSF photometry has been calculated, the pipeline will then calculate a zero point based on the PSF photometry and the catalog magnitudes of stars (the catalog used is determined by the **catalog_zp** function in the setting file and the sky coverage of the catalog). Zero points will be reported in AB mag.

## Manual aperture photometry
If the **--phot True** argument is used to run the pipeline, the pipeline will allow you to perform manual aperture photometry. The terminal will prompt you for details such as the zero point, position of target, and aperture and annulus to use. Cut-outs around the target of interest and radial plots will also be shown to help the user identify and choose parameters. First, the pipeline will ask which stack you would like to perform photometry on. The pipeline will print a list of options and then ask:

**Index (starting from 0) of the stack you want to perform aperture photometry on?**

For instruments with only one stack, the answer will always be 0. For instruments with multiple stacks, like BINOSPEC, you can choose which stack to choose in the list using python indexing.\\

Next, the pipeline will do perform PSF photometry and calculate a zero point if there is no photometry catalog file present in the red folder. Once the pipeline has done this, it will ask if the user wishes to use the zero point found by the PSF photometry, or enter a manual zero point (this zero point should be in AB mag):

**Enter user zeropoint instead of loading from psf photometry ("yes" or "no")?**

If you choose to load the zero point from the PSF photometry (answering "no" the prompt), the pipeline with use this value if available. If the zero point is not available (due to an error in the calculation such as no matching catalog for calibration), the pipeline will report there is no zero point and you will need to add in a zero point (if no zero point has been calculated, enter 0. This way it will be clear looking back at the log that no zero point was calculated). If you choose to enter your own zero point from the beginning (answering "yes" to the prompt), the pipeline will ask you to enter the zero point and the error in AB mag.\\

The pipeline will then ask you if you wish to enter the position of the target in RA and Dec ("wcs") or x and y coordinates ("xy"):

**Would you like to enter the RA and Dec ("wcs") or x and y ("xy") position of the target?**

After entering one of these options, it will ask you to enter the RA and Dec of the target in degrees (if you answered "wcs"), or the x and y position in pixels (if you answered "xy"). Once you have chosen a position, the pipeline will ask about the size of the cutout around the target for visualization:

**Would you like to choose the cutout size? Default is 50"x50".**

The default is 50x50". If you want to change the size, answer "yes" and then enter the radius of the box you want to use. Otherwise enter "no" or just hit enter to accept the 50"x50" cutout. Now the pipeline will show you the cutout centered on the position you chose, as well as the radial plot for that position. If you are happy with the position, close the figures to move on. If you would like to select a new position, double click on the cutout to select that position. You can use the tool bar on the figure to zoom in and pan should you want. Once you have selected a new position, the position will be reported on the terminal. Closing the figures will allow the pipeline to reload them with the updated position. Repeat this process until you are happy with the position. After this, the pipeline will ask if you want to use a centroid instead, in the case when you are unable to determine the center of your source manually:

**Would you like to use a centroid? Type 'yes' or 'no':**

If you answer "yes", the pipeline will use a 2D Gaussian to determine the center of the source. A new cutout and radial plot with show you to the position determined using the Gaussian. After closing the figures, the pipeline will ask if you are happen if the final position:

**Are you ok with this position? Type 'yes' or 'no':**

Answering "yes" will allow the pipeline to move on to the aperture and annulus selection, while answering "no" with repeat the whole process until you answer "yes".\\

Next, the pipeline will print out the automatic radii selected using the FWHM calculated from the PSF photometry (aperture = 2.5 x FHMW and the annulus = 2.5 x FHWM, 4.5 x FWHM). The pipeline if you would like to use these radii or enter your own:

**Would you like to use these radii? Type 'yes' or 'no':**

Answering "no" will allow you to enter your own radii before the cutout and radial plot (in pixels) will display the selected radii. If you are happy with these, close the figures and answer "yes" when the pipeline asks:

**Are you ok with the previously selected radii? Type 'yes' or 'no':**

Otherwise, take note the the radii you wish to use before closing the figures, then answer "no" to the pipeline when it asks if you wish to use these radii. If you answer "no", the pipeline will then ask you to enter new radii (in pixels) for the aperture and inner and outer annulus. The cutout and radial plot will now show the new radii entered. Repeat the process until you are happy. Once you are and have told the pipeline, the pipeline will then calculate aperture photometry based on these radii. If a magnitude with greater than 3 sigma can be calculated, the pipeline will report the target magnitude as such. If the error on the magnitude is greater than 1/3, the pipeline will be calculate a limiting magnitude by adding in fake sources around the position of the target.\\

Once a magnitude or limit has been calculated, the pipeline will then calculate an extinction at the position of the target. The extinction is calculated using the Schlafly & Finkbeiner (2011) maps, and is reported in mag for the respective filter of the observation.

## Quality checks
The pipeline will automatically create a log file in the **red** folder named after the instrument, followed by the date and time at the start of the run (UTC). A lot of information is written to the log file about the various steps step in the pipeline and the reduction process. Since this log file can also contain the reduction of multiple targets, it is suggested to make a personal log for each target with key information that can be used for the data analysis. Before, during and after the pipeline the has run, there are some key aspect you should check in terms of quality control that should be noted in the target log. These steps are list and explained below:
### Before
- Check observing log: Check the observing log for notes from the observer about any poor images or images that should be removed. Unless there is a keyword in the header indicating that the image is bad, the pipeline will not be able to identify them and they need to be move the bad folder manually.
### During
- Check the file list: The first thing the pipeline will do is to create a file list containing information about the files. The pipeline will ask you to review this file list and make edit if needed. Things that should be checked are the exposure times, the file type and the target name. If there are any images you want to remove, change the type to BAD. Edits to the target name can be made directly under the target column. \\
- Check stack: After the pipeline has reduced and stacked the files for a target, the pipeline will ask you to check the quality of the stack before asking if you wish to continue with the reduction. Check the fits image named after the target name. If the image is of poor quality with very few stars, the pipeline will struggle to solve the WCS and the reduction should be aborted for that target. It there is reason for the poor quality, i.e. bad weather etc., indicate such in the target log and update the watchlist. If there is no obvious reason why the stack would be bad, and you have checked the observing log, the individual images, the calibration files, and known issues, open an Issue on GitHub for follow up. \\
- Check WCS solution: Once the stack is approved by the user, the pipeline will calculate an automatic WCS solution. The pipeline will then ask you to review the error plots created and the reported rms on the astrometry, before asking if you wish to perform an manual WCS solution. The rms on the astromtery should be $<$ 0.5" and the stars aligned well near your target as well as across the image. Repeat the process if you chose to manually determine the WCS solution. After you are happy with the astrometry, the pipeline will ask if you wish to continue with the PSF photometry. If the WCS is off by a large number of pixels, the zero point will not be calculated correctly and the reduction should be aborted. You can open an Issue on GitHub for follow up.
### After
- Check shifts and flips calculated: When the pipeline aligns images, it will calculate the relative shifts based on the stars. These shifts are related to the dither pattern and size selected during observing. In general, these shifts shouldn't be too large (can be on the order on 10s of pixels for smaller dithers to ~300 pixels for large dither patterns) and should not flips the image unless you are combining images from multiple nights (in this case, very large shifts and flips can be expected). If this isn't the case, the individual images should be check to see if the large shifts are indeed correct, otherwise you can open an Issue on GitHub. \\
- Check removed images during quality control: The pipeline will perform an automatic quality control of the images to be stacked. This includes removing outlying in terms of seeing, the number of stars detected (in the case of clouds) and the alignment. The pipeline will report the number of images removed during this process. The removed images should be checked to see if they indeed needed to be removed. If large number of images are removed in the quality control, you can open an Issue on GitHub for follow up. \\
- Check PSF: The PSF calculated by the pipeline will determine if the PSF photometry, as well as the zero point can be trusted. The PSF is saved in the .epsf.png file, and should look like a 2D Gaussian. The pipeline will also report the x and y sigma of a 2D Gaussian fitted to the PSF. If the PSF looks good and the x and y sigma values reeported are similar, the PSF photometry can be trusted. If the PSF doesn't look Gaussian-like and the reported x and y sigma values are very different or negative, then there was an issue determining the PSF. If there are a lot of good stars in the final stack that could have been used to determine the PSF, you can open an Issue on GitHub for follow up.\\
- Check zero point calculation: If the PSF photometry is reliable, the zero point calculation should be good if a large number of stars is used to calculate it. If less than ~10 stars are used to calculate the zero point, this should be noted on the target log and may not be reliable. If there are more stars in the field that the pipeline should have used, or the pipeline reported that no stars were found but there is coverage in the corresponding catalog, you can open an Issue on GitHub for follow up.\\

## Special cases
This section describes some special cases in which the pipeline can be run to reproduce particular files.
### Redo sort files:
To remake the file list, run the pipeline using the **--reset** option. There are two options: **all** and **raw**. **--reset all** will move all files (good, bad, spec etc.) back to the default path before remaking the file list and continuing to run the pipeline. This is useful if changes have been implemented with the **Sort_file.py** script and the file list needs to remade but there are no issues with the data itself. **--reset raw** will move only the files in the raw folder to the default path before remaking the file list and continuing to run the pipeline. This is useful if you manually moved bad images to the bad folder and don't want to include them in the reduction process.

### Redo WCS on stack:
To redo the astrometry on a stack, you will either need to delete the *_wcs* image and rerun the pipeline (deleting the *_wcs* image allows you to use **--skip_red True** to skip calibration files), otherwise just rerunning the pipeline with redo everything, or manually run **solve_wcs** or **man_wcs** on the image.

### Redo PSF photometry:
If you wish to redo the PSF photometry for a target (i.e. the final stack is good but some updates were made to the PSF calculation), delete the *.pcmp* file for the target and rerun the pipeline with **--skip_red True** and **--phot True**. Since the pipeline will look for the PSF catalog before performing the manual aperture photometry, it will create this file if it is missing. If you don't wish to perform manual aperture photometry after the PSF photometry is complete, just kill the pipeline with Cntrl-z once it is done.

### Standards with the same target name
Move files to separate folder, along with calibrations and run with **--skip_red True**, or edit file list to have the correct name.

### Data over multiple nights
Move all raw to new folder. Run pipeline on new folder.

## Issues and error
Should you encounter an error in the pipeline, have a special data setup that doesn't run through the pipeline, or wish to add an additional instrument, please open an Issue on GitHub.
### Common errors and fixes
### Error with files
The pipeline can't find right the dark calibration file: check the file list for the exposure times of files.
### Wrong target name
Edit the file list.
### DEIMOS target in the chip gap
Check the header if PONAME==DEIMOS, this means the pointing origin was set to center of CCD in gap. Error with observing, no solution for data reduction.
### Kspec filter
On MMT, the Kspec filter is used for spectroscopy. If it is used for imaging, the images will be saturated. Error with observing, no solution for data reduction.
### Bad WCS in crowded field
Sometimes for crowded fields, the automatic WCS solutions fails to find the correct unique star matches. Manual WCS needs to do done.
