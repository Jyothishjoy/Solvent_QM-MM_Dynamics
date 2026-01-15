# Procedure to locate the counter ion position

### GOAT/UMA for Initial Counter Ion Location

Given an approximate starting configuration, GOAT algorith implimented in ORCA 6.1 can give us potential counter ion locations. I used UMA for quick screening.

    ! ExtOpt GOAT
    
    %maxcore 8000
    
    %pal
    nprocs 24   
    end
    
    %GOAT 
    NWORKERS 24 
    MAXEN 10
    END
    
    %method
    ProgExt "/home/fslcollab286/orca_6_1_0/orca-external-tools/umaclient.sh"
    Ext_Params "-b 127.0.0.1:8888 --solvent=None"
    end
    
    %geom Constraints
            { C 0:74 C }
            end
    end
    
    * XYZFILE 0 2 input.xyz

### K-Menas Clustering of the GOAT-ensemble

After the Goat calculation, along with the global minimum structure, we get a file named `GOAT_temp.finalensemble.xyz`.

This file contains all the potential conformations GOAT found. We can use K-Means clustering implemented in AMBERTools to generate 10 potential structures (https://github.com/Jyothishjoy/Fe-Silylation/tree/main/K_Means_Clustering)

I used WSL2 for this step. Convert `xyz` file to `pdb` using the following command.

    babel -i xyz GOAT_temp.finalensemble.xyz -O GOAT_temp.finalensemble.pdb

Then use `mol_2_cluster.in` file with cpptraj to generate 10 representative clusters.

    parm GOAT_temp.finalensemble.pdb
    trajin GOAT_temp.finalensemble.pdb
    cluster c1 \
      kmeans clusters 10 randompoint maxit 500 \
      rms @1-999 \
      sieve 10 random \
      out clustertime.dat \
      summary summary.dat \
      info info.dat \
      cpopvtime cpopvtime.agr normframe \
      repout rep repfmt pdb \
      singlerepout singlerep.nc singlerepfmt netcdf \
      avgout avg avgfmt pdb
    run 


### DFT optimization of 10 Clusters

Use DFT to optimize the 10 representative clusters to identify the structure that best represents the counterion position.
