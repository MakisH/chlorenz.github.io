---
title: Configurations Actions
tags: [configurations, actions]
keywords: pages, authoring, exclusion, frontmatter
last_updated: July 16, 2016
summary:
sidebar: docs_sidebar
permalink: configurations_actions.html
folder: docs
---

Often, the solvers we want to couple may offer similar but not exactly the same kind of data on their interfaces. For instance, a fluid solver may only provide access to its surface forces, whereas a structure solver may only work with stresses. In such cases, the surface force values need to be divided by the corresponding surface area within the structure solver.

In order to flexibly modify the coupling values exchanged between the participants, preCICE supports _coupling actions_. Coupling actions are essentially a set of preCICE functionalities that have access to a coupling mesh and the corresponding data values. preCICE offers two approaches for defining coupling actions: pre-implemented coupling actions and through a Python callback interface.

# Basics and Pre-implemented Actions 

```xml
<participant name="MySolver1"> 
    <use-mesh name="MyMesh1" provide="yes"/> 
    <write-data name="Forces" mesh="MyMesh1"/> 
    ...
    <action:divide-by-area mesh="MyMesh1" timing="write-mapping-post">
        <target-data name="Forces"/>
    </action:divide-by-area>
    ...
</participant>
```

This example divides the force values by the respective element area, transforming forces into stresses. Please note that **mesh connectivity information needs to be provided** (edges, triangles, etc. through `setMeshEdge` or similar API functions).

`timing` defines _when_ the action is executed. Options are:
- `write-mapping-prior` and `write-mapping-post`: directly before or after each time the write mappings are applied.
- `read-mapping-prior` and `read-mapping-post`: directly before or after each time the read mappings are applied.
- `on-time-window-complete-post`: after the coupling in a complete time window has converged, after `read` data is mapped.

<details><summary>Older (preCICE version < 2.1.0) timings that are now deprecated and will revert to one of the above options: (...)</summary>

* `regular-prior`: In every `advance` call (also for subcycling) and in `initializeData`, after `write` data is mapped, but _before_ data might be sent. (*v2.1 or later: reverts to `write-mapping-prior`*)
* `regular-post`: In every `advance` call (also for subcycling), in `initializeData` and in `initialize`, before `read` data is mapped, but _after_ data might be received and after acceleration. (*v2.1 or later: reverts to `read-mapping-prior`*)
* `on-exchange-prior`: Only in those `advance` calls which lead to data exchange (and in `initializeData`), after `write` data is mapped, but _before_ data might be sent. (*v2.1 or later: reverts to `write-mapping-post`*)
* `on-exchange-post`: Only in those `advance` calls which lead to data exchange (and in `initializeData` and `ìnitialize`), before `read` data is mapped, but _after_ data might be received. (*v2.1 or later: reverts to `read-mapping-prior`*)
</details>

Pre-implemented actions are:
* `multiply-by-area` / `divide-by-area`: Modify coupling data by mesh area
* `scale-by-computed-dt-ratio` / `scale-by-computed-dt-part-ratio` / `scale-by-dt`: Modify coupling data by timestep size
* `compute-curvature`: Compute curvature values at vertices
* `summation`: Sum up the data from source participants and write to target participant

For more details, please refer to the [XML reference](XML-Reference).

***

# Python Callback Interface

Other than the pre-implemented coupling actions, preCICE also provides a callback interface for Python scripts to execute coupling actions. To use this feature, you need to [build preCICE with python support](Dependencies#python-optional). 

We show an example for the [1D elastic tube](1D-Example): 

```xml
<participant name="STRUCTURE">
    <use-mesh name="Structure_Nodes" provide="yes"/>
    <write-data name="CrossSectionLength" mesh="Structure_Nodes"/>
    <read-data  name="Pressure"      mesh="Structure_Nodes"/>
    <action:python mesh="Structure_Nodes" timing="read-mapping-prior">
        <path name="<PATH_TO_PYTHON_ACTION_SCRIPT>"/>
        <module name="<PYTHON_SCRIPT_NAME_WITHOUT_.PY>"/>
        <source-data name="Pressure"/>
        <target-data name="Pressure"/>
    </action:python>
</participant>
```

The callback interface consists of the following three (optional) functions:
```python
performAction(time, sourceData, targetData) 
vertexCallback(id, coords, normal) 
postAction()
```

`performAction` gives access to the coupling value arrays. You can store these values in global variables to grant access to the other two functions.

`vertexCallback` gives access to the geometric data of each vertex. This function is called successively for every vertex of the specified coupling mesh and you can use the corresponding geometric data. 

`postAction` is called at the final step. You can perform any finalizing code after deriving information from the vertices, if wished.

Without the Python action, the 1D elastic tube gives the following results:

![](https://raw.githubusercontent.com/wiki/precice/precice/images/elastictube_diameter.png)

Now, we want to ramp up the pressure values written by the fluid solver over time. A feature often needed to get a stable coupled simulation.

```python
mySourceData = 0
myTargetData = 0

def performAction(time, dt, sourceData, targetData):
    # This function is called first at configured timing. It can be omitted, if not
    # needed. Its parameters are time, timestep size, the source data, followed by the target data.
    # Source and target data can be omitted (selectively or both) by not mentioning
    # them in the preCICE XML configuration.

    global mySourceData
    global myTargetData

    mySourceData = sourceData # store (reference to) sourceData for later use
    myTargetData = targetData # store (reference to) targetData for later use

    timeThreshold = 0.2 # Ramp up the pressure values until this point in time

    if time < timeThreshold:
        for i in range(myTargetData.size):
	    # Ramp up pressure value
            myTargetData[i] = (time / timeThreshold) * mySourceData[i]

    else:
        for i in range(myTargetData.size):
            # Assign the computed physical pressure values
            myTargetData[i] = mySourceData[i]


def vertexCallback(id, coords, normal):
    # This function is called for every vertex in the configured mesh. It is called
    # after performAction, and can also be omitted.

    # Usage example:
    global mySourceData # Make global data set in performAction visible
    global myTargetData
    # Example usage, add data to vertex coords:
    # myTargetData[id] += coords[0] + mySourceData[id] 

def postAction():
    # This function is called at last, if not omitted.

    global mySourceData # Make global data set in performAction visible
    global myTargetData
    # Do something ...
```
With the Python action, you should now get the following results. Note the lower maximum diameter and the change at `t=0.2` (`t=20` in the graph). 
![](https://raw.githubusercontent.com/wiki/precice/precice/images/diameter_pythonAction.png)