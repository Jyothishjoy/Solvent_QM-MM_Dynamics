# Solvent_QM-MM_Dynamics
Procedure to run QM/MM Solvent Dynamics using Milo and Gaussian-16/ORCA-6.1

## Step-I: Packing the TS structure with Solvents and Equilibration

**Local Initialization**

Use the following `job` and `theory` sections to locally initialize Milo for packing the solvents.

    $job
      api                              aarontools
      job_type                         equilibration
      program                          gaussian
      integration_algorithm            velocity_verlet
      max_steps                        2
      current_step                     0
      step_size                        0.75
      explicit_solvent                 ACN 200
      packmol_tolerance                2.7
      boundary_side_length             16.4
      boundary_shape                   sphere
      explicit_solvent                 none
      frozen                           1
      temperature                      298.15
      translations                     none
      rotations                        none
      vibrations                       none
      vibrational_sampling             classical
      thermostat                       stochastic
      temperature_coupling_time        500.0
      temperature_coupling_frequency   1
      reference_temperature            283.15
      processors                       12
      memory                           96
     random_seed 60636271145
    $end
    
    $theory
      route                            external=('gau_xtb') 
    $end

After this run, use `python -m milo.setu_restart` to generate a restart input file. 

**Restart-I-298.15K**

In this step, run Milo equilibration for 15 ps (20 k steps with 0.75 fs step size) at 298.15k using `UMA` as a Gaussian external tool. 

Currently, `UMA` is the only reliable tool to match the speed of a force field simulation.

      $job
        api                              aarontools
        job_type                         equilibration
        program                          gaussian
        integration_algorithm            velocity_verlet
        max_steps                        20000
        current_step                     2
        step_size                        0.7500000000000001
        boundary_side_length             16.4
        boundary_shape                   sphere
        explicit_solvent                 none
        frozen                           1
        temperature                      298.15
        translations                     none
        rotations                        none
        vibrations                       none
        vibrational_sampling             classical
        thermostat                       stochastic
        temperature_coupling_time        500.0
        temperature_coupling_frequency   1
        reference_temperature            298.15
        processors                       1
        memory                           48
       random_seed 74693821381
      $end
      
      $theory
        route                                   external=('gauuma') 
      $end

**Restart-II-0.0 K**

In this step, cool down the system after the initial 15ps heating to 0 K. This step is necessary because Gaussian performs geometry optimization at 0 K.

If we skip this step, the solvent configuration we provide to Gaussian in the next step will be very different from what Gaussian can optimize. 

Now, run Milo equilibration for 15 ps (20 k steps with 0.75 fs step size) at 0.0 k using `UMA` as a Gaussian external tool. 

      $job
        api                              aarontools
        job_type                         equilibration
        program                          gaussian
        integration_algorithm            velocity_verlet
        max_steps                        50000
        current_step                     15000
        step_size                        0.75
        boundary_side_length             16.4
        boundary_shape                   sphere
        explicit_solvent                 none
        frozen                           1
        temperature                      0.0
        translations                     none
        rotations                        none
        vibrations                       none
        vibrational_sampling             classical
        thermostat                       stochastic
        temperature_coupling_time        500.0
        temperature_coupling_frequency   1
        reference_temperature            0.0
        processors                       1
        memory                           48
       random_seed 98298759143
      $end
      
      $theory
        route                                   external=('gauuma') 
      $end


## Step-II: Energy minimization in Gaussian and TS optimization

**Gaussian energy minimization of the Solvents *Only***

In this step, take the final structure from the above run (ideally, perform K-means clustering to obtain the best representative cluster) to initiate energy minimization of the solvents using `UMA`.

Here, minimize only the solvents using `external=gauuma`, keeping the `solute frozen` and allowing the solvents to freely optimize their positions.

        %mem=24GB
        %nprocs=1
        # opt=(nomicro) external='gauuma'
        
        TS optimization
        
        0 2 0 2 0 2
         Fe               -1    1.19406000   -0.90534000    0.41536000 H
         O                -1   1.75154000   -1.01737000    2.26442000 H
         N                -1   -0.40681000   -0.12387000    1.28688000 H
         C                -1   -1.47378000   -0.90441000    1.55209000 H
         O                -1    2.40619000   -1.86372000   -0.15395000 H
         N                -1    1.84171000    1.05444000    0.52589000 H
         C                -1   -2.63945000   -0.34993000    2.09349000 H
         H                -1   -3.52867000   -0.98016000    2.24018000 H
         N                -1    0.42735000   -0.35977000   -1.42992000 H
         C                -1   -2.64505000    1.02662000    2.37915000 H
         H                -1   -3.55859000    1.49770000    2.77219000 H
         N                -1   -0.15643000   -2.53371000    0.26272000 H
         C                -1   -1.48896000    1.80243000    2.18685000 H
         H                -1   -1.45828000    2.87988000    2.40744000 H
         ...............................................................
         ...............................................................
         O               -1   -1.20389000    3.59377000   -0.47831000 H
         C               -1   -0.98569000    6.24323000   -0.70858000 H
         F               -1   -0.58797000    6.14445000   -1.99290000 H
         F               -1   -0.41391000    7.33807000   -0.16897000 H
         F               -1   -2.32335000    6.40893000   -0.68988000 H
         C                0    0.00636100   -7.53091200   -1.41428800 M
         N                0    0.69830700   -6.96523200   -0.69023500 M
         C                0   -0.85168900   -8.27199400   -2.32221800 M
         H                0   -1.00208000   -7.69137900   -3.23123100 M
         H                0   -0.37093900   -9.21114500   -2.59131700 M
         H                0   -1.80535000   -8.49024600   -1.83939800 M
         ...............................................................
         ...............................................................


**Gaussian energy minimization of the *full system* for the ModRed**

In the next step, use `ONIOM((ubp86/def2svp em=gd3bj:external='gauuma')` to run ModRed of the TS structure using the following input file.

        %chk=PyNMe3+cis-DMC-2Trifl-200ACN-Doublet-BP86D3BJUMA-ModRed.chk
        %mem=96GB
        %nprocs=24
        # opt=(modredundant,maxcycles=500,loose,nomicro) freq
        oniom(ubp86/def2svp em=gd3bj:external='gauuma') scf=(maxcycles=500,xqc)
        
        TS optimization
        
        0 2 0 2 0 2
         Fe               0    1.19406000   -0.90534000    0.41536000 H
         O                0    1.75154000   -1.01737000    2.26442000 H
         N                0   -0.40681000   -0.12387000    1.28688000 H
         C                0   -1.47378000   -0.90441000    1.55209000 H
         O                0    2.40619000   -1.86372000   -0.15395000 H
         N                0    1.84171000    1.05444000    0.52589000 H
         C                0   -2.63945000   -0.34993000    2.09349000 H
         H                0   -3.52867000   -0.98016000    2.24018000 H
         N                0    0.42735000   -0.35977000   -1.42992000 H
         C                0   -2.64505000    1.02662000    2.37915000 H
         H                0   -3.55859000    1.49770000    2.77219000 H
         N                0   -0.15643000   -2.53371000    0.26272000 H
         C                0   -1.48896000    1.80243000    2.18685000 H
         H                0   -1.45828000    2.87988000    2.40744000 H
         C                0    2.34274000   -1.99768000    2.90623000 H
         C                0   -0.35647000    1.17601000    1.65238000 H
         C                0    1.02323000    1.75119000    1.57615000 H
         H                0    1.50518000    1.54472000    2.55322000 H
         H                0    1.04545000    2.84548000    1.39708000 H
         C                0    3.27478000    1.14100000    0.90510000 H


**Gaussian OPTTS calculation using the ModRed**

