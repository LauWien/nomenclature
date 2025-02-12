.. _usage:

.. currentmodule:: nomenclature

Usage
=====

.. contents::
   :depth: 3

The :class:`DataStructureDefinition` and :class:`RegionProcessor` classes are
initialized from yaml files that must follow specific formats. This page describes the
required specifications of the files.

DataStructureDefinition
-----------------------

A :class:`DataStructureDefinition` contains **CodeLists**
(:class:`nomenclature.codelist.CodeList`). A **CodeList** is a list of “allowed terms”
(or codes), where each term can have several attributes (e.g., description, unit, parent
region). By default :class:`DataStructureDefinition` reads *regions* and *variables* for
validation, however codelists can be used to validate any dimension in the IAMC format
(see :ref:`generic`).

Variable
~~~~~~~~

The *variable* codelist of the :class:`DataStructureDefinition` will be read from all
yaml files located in a folder of that name (including any sub-folders). They must be
formatted as a list of dictionaries mapping the variable (key) to its attributes.

.. code:: yaml

   - Some Variable:
       unit: A unit
       description: A short description
       <other attribute>: Some text, value, boolean or list (optional)

A *unit* attribute
^^^^^^^^^^^^^^^^^^

Every variable **must have** a **unit**, which should be compatible with
the `iam-units <https://github.com/iamconsortium/units>`_ package.

The unit attribute can be empty, i.e., the variable is *dimensionless*,

    .. code:: yaml

       - Some Variable:
           unit:

Attributes for region processing (optional)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

* By default, when applying a :class:`RegionProcessor` instance, all variables
  are processed using the method :meth:`pyam.IamDataFrame.aggregate_region`,
  which performs a simple summation of all subregions.

* Region aggregation for a particular variable can be skipped by using the attribute
  *skip-region-aggregation*.

    .. code:: yaml

       - Some Variable:
           skip-region-aggregation: true

* Any attributes which are arguments of
  :meth:`aggregate_region() <pyam.IamDataFrame.aggregate_region>` will
  be passed to that method, e.g., *method*, *weight*.

* It is possible to rename the variable returned by the region processing using
  a *region-aggregation* attribute, which must have a mapping of the target variable to
  arguments of :meth:`aggregate_region() <pyam.IamDataFrame.aggregate_region>`.

  This option can be used to compute several variables as part of the region-processing.
  In the example below, the variable *Price|Carbon* is computed as a weighted average
  using the CO2 emissions as weights, and in addition, the maximum carbon price within
  each aggregate-region is added as a new variable *Price|Carbon (Max)*.

    .. code:: yaml

        - Price|Carbon:
            unit: USD/t CO2
            region-aggregation:
              - Price|Carbon:
                  weight: Emissions|CO2
              - Price|Carbon (Max):
                  method: max

Other attributes
^^^^^^^^^^^^^^^^

Other attributes such as "description" are optional.

Region
~~~~~~

To avoid repeating a “hierarchy” attribute many times (e.g., country,
continent), the yaml files must have a nested dictionary structure:

.. code:: yaml

   - <Hierarchy Level>:
     - region 1:
         Attribute: Attribute value
     - region 2:
         Attribute: Attribute value 

**Notes**

* Every region **must be** defined as part of a hierarchy.
* When importing a *region* codelist, the hierarchy will be added as attribute, such
  that it can be retrieved as:

.. code:: python
  
   DataStructureDefinition.region["Region Name"]["Hierarchy"] = "<Hierarchy Level>"

* Other attributes specified in the yaml file can include (for countries)
  ISO2/3-codes (https://en.wikipedia.org/wiki/List_of_ISO_3166_country_codes), or the
  list of countries included in a macro-region (i.e., a continent or large region).

.. _generic:

Generic
~~~~~~~

In order to validate IAMC dimensions other than 'region' or 'variable' (e.g. 'scenario')
generic codelists can be used. 

.. code:: yaml

   - scenario 1:
      - description: Something about scenario 1
   - scenario 2
   - ...


**Notes**

* The requirements for generic codelists are more relaxed than for 'region' and
  'variable':

  * It must be a list (i.e. entries start with a dash '-') 
  * Entries can either be a key value pair (like 'scenario 1') or a simple string (like
    'scenario 2').

* The files belonging to this dimension need to be placed in a folder of the same name 
  as the IAMC dimension to be validated. In our example 'scenario'.
* When instantiating a :class:`DataStructureDefinition` with dimensions other than     
  'region' and 'varaible' a list of **all** dimensions must be provided:

.. code:: python

   dsd = DataStructureDefinition('definitions', ['region', 'variable', 'scenario'])


Tag
~~~

To avoid repetition (and subsequent errors), any number of yaml files can be used as
“tags” using a nested list of dictionaries. There must be only one top-level entry in
any yaml file to be used as tag. The files defining the tags must have a name starting
with ``tag_``.

.. code:: yaml

   - Tag:
     - Some Key:
         description: a short description of the key

When importing a *tag* codelist, any occurrence of ``{Tag}`` in a variable
name will be replaced by every element in the Tag dictionary. The
``{Tag}`` will also be replaced in any of the variable attributes.


RegionProcessor
---------------

The :class:`RegionProcessor` class holds a list of model mappings.

Model mapping
~~~~~~~~~~~~~

Model mappings, defined on a per-model basis serve three different purposes:

1. Defining a list of model native regions that are to be selected (and
   usually uploaded) from an IAM result. This also serves as an implicit
   exclusion list for model native regions, since only explicitly
   mentioned regions are selected.
2. Allowing for renaming of model native regions.
3. Defining how model native regions should be aggregated to common
   regions.

This example illustrates how such a model mapping looks like:

.. code:: yaml

  model: model_a
  native_regions:
    - region_a: alternative_name_a
    - region_b
   common_regions:
    - common_region_1:
      - region_a
      - region_b
    - common_region_2:
      - ...
  exclude_regions:
    - region_c
    - ... 

**Notes**

*  The names of the four top level keywords are fixed:

  * *model*
  * *native_regions*
  * *common_regions*
  * *exclude_regions*

*  Required properties are:
  
  * *model* and 
  * at least one of *native_regions* and *common_regions*

* Optional properties are:
  * *exclude_regions*

*  *model* (str or list of str): the model name(s) for which the mapping applies.
*  *native_regions* (list): a list of model native regions serves as
   a selection as to which regions to keep.

   *  In the above example *region_a* is to be renamed to
      *alternative_name_a*. This is done by defining a key-value pair
      of *model_native_name: new_name*.
   *  *region_b* is selected but the name is not changed.
   *  Assuming *model_a* also defines a third region *region_c*,
      since it is not mentioned it will be **dropped** from the data.

*  *common_regions* (list): list of common regions which will be
   computed as aggregates. They are defined as list entries which
   themselves have a list of constituent regions. These constituent
   regions must be model native regions.

* *exclude_regions* optional (list of str): If input data for region processing provided
  to :func:`process()` or :meth:`RegionProcessor.apply` contains regions which are not
  mentioned in *native_regions*, in *common_regions* (as the name of a common region or
  a constituent region)  an error will be raised. This is a safeguard against silently
  dropping regions which are not in named in *native_regions* or *common_regions*. 
  
  If regions are to be excluded, they can be explicitly named in the *exclude_regions*
  section which causes their presence to no longer raise an error.

.. note::
   The names of the constituent regions **must** refer to the **original** model
   native region names. In the above example *region_a* and *region_b* and **not**
   *alternative_name_a*.

Partial region aggregation
~~~~~~~~~~~~~~~~~~~~~~~~~~

During the region aggregation process provided and aggregated data are combined in a
process dubbed 'partial region aggregation'.

As an example, consider the following model mapping:

.. code:: yaml

   model: model_a  
   common_regions:
     - common_region_1:
       - region_a
       - region_b

If the data provided for region aggregation contains results for *common_region_1* they
are compared and combined according to the following logic:

1. If a variable is **not** reported for *common_region_1*, it is calculated through
   region aggregation of regions *region_a* and *region_b*.
2. If a variable is **only** reported for *common_region_1* level it is used directly.
3. If a variable is is reported for *common_region_1* **as well as** *region_a* and
   *region_b*. The **provided results** take **precedence** over the aggregated ones.
   Additionally, the aggregation is computed and compared to the provided results. If
   there are discrepancies, a warning is written to the logs.
   
   .. note::

      Please note that in case of differences no error is raised. Therefore it is
      necessary to check the logs to find out if there were any differences. This is
      intentional since some differences might be expected.

More points to note:

* The attribute `skip-region-aggregation` which can be set to `true` for variables as
  part of a DataStructureDefinition has the following effect. If `true`, aggregation for
  this variable is skipped. However, if the variable is part of the provided data, it
  **is** used.
* The `region-aggregation` attribute works with partial region aggregation. If a
  variable is found in the provided data, it is used over aggregated results. Any
  discrepancies between the provided and aggregated data are written to the log.
