Genweb 6-buildout
====================

This is the ultimate-uber-maxi-definitive-awesomefull buildout for Genweb-based
projects.

Usage
-----

You should choose what project do you want to work with or the basis of your new
one. For example, if you want to use a GenwebUPC buildout you should specify the
name of the project .cfg associated:

.. code-block:: bash

 $ create venv python 3.9.11

 $ cp customizeme.cfg.in customizeme.cfg

 $ cp bootstrap.sh.in bootstrap.sh

 $ run the ./bootstrap.sh venv_directory


Check out for the available projects in the projects.cfg file.

.. note:: Before that, remember to copy the customizeme.cfg.in file into customizeme.cfg configuring the required parameters.

Available projects/builds
-------------------------
* Genweb UPC
   - genwebupc.cfg
