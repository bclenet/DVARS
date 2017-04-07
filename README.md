# Introduction
Series of codes (mainly in MATLAB and Shell scripts), available here, to support DVARS inference and DSE variance decomposition proposed in

 __Afyouni S. & Nichols T.E, *insight and inference for DVARS*, 2017__
http://www.biorxiv.org/content/early/2017/04/06/125021

The toolbox can be used to:

* generate DVARS and five standardised variants of the measure.
* generate p-values for spikes to facilitate the decision whether a spike is corrupted and should be scrubbed.
* explain variance of 4D fMRI images via three variance components: fast (D-var), slow (S-var) and edge (E-var).
* explain what share of the whole variance each component occupies.

# Configuration

## Dependencies
* MATLAB2016b (or later) statistical toolbox [required].
* `'/Nifti_Util'` for neuroimaging analysis [optional].
* `FSL 5.0.9` (or later) to produce DSE variance images [optional].

## DVARS Inference
Using `DVARSCalc.m` allows you to estimate the DVARS-related statistics (e.g. D-var, p-values for spikes and standardised variants of the fast variance).

As a quick example, for a pure white noise with *I=1000* observations (voxels) and *T=100* data-points, we have:
```
Y = randn(1000,100);
[DVARS,Stat] = DVARSCalc(Y);
```

```
-Input is a Matrix.
-Extra-cranial areas removed: 1000x100
-No normalisation/scaling has been set!
-Data centred. Untouched Grand Mean: 0.0019315, Post-norm Grand Mean: 0.0019315
-Settings:
--Test Method:          Z
--ExpVal method:        median
--VarEst method:        hIQRd
--Power Transformation: 1

Settings: TestMethod=Z  I=1000  T=100
----Expected Values----------------------------------
    sig2bar    sig2median    median    sigbar2     xbar
    _______    __________    ______    _______    ______

    2.0342     1.984         1.9898    2.004      1.9998

----Variances----------------------------------------
     S2       IQRd         hIQRd
    ____    _________    _________

    0.01    0.0052876    0.0034822
```

As you can see in the printed log of the function, the default method of testing
 is Z with `median` of squared DVARS as expected value and `hIQR` robust
 variance. However, these settings are easily amendable. See example below:

```
Y = randn(1000,100);
[DVARS,Stat]=DVARSCalc(Y,'TestMethod','X2','VarType','IQR','MeanType',1,'RDVARS','TransPower',1/3);
```
Where the test is done via Chi-squared null model, expected value is estimated via
robust variance of differenced time series (see Eqn. 17), variance is estimated
via delta method variance of transformed (*d=1/3*) DVARS-squared (see Appendix E).
As argument `RDVARS` triggered, the Relative DVARS (RDVARS) is also calculated.
Contrary to the previous example, we use `IQR` for all robust estimates.
In the following, you can see the log generated by the function:
```
-Input is a Matrix.
-Extra-cranial areas removed: 1000x100
-No normalisation/scaling has been set!
-Data centred. Untouched Grand Mean: -0.0007454, Post-norm Grand Mean: -0.0007454
-Robust estimate of autocorrelation...
--voxel: 1000
-Settings:
--Test Method:          X2
--ExpVal method:        sig2bar
--VarEst method:        IQRd
--Power Transformation: 0.33

Settings: TestMethod=X2  I=1000  T=100
----Expected Values----------------------------------
    sig2bar    sig2median    median    sigbar2     xbar
    _______    __________    ______    _______    ______

    2.069      2.0066        2.0173    2.036      2.0182

----Variances----------------------------------------
     S2       IQRd        hIQRd
    ____    _________    ________

    0.01    0.0074099    0.009374
```
It is also interesting to see how a spike is detected using this code.

```
I=4e4; T=1200;
Y=randn(I,T);
Idx_OL = 463; %Idx_OL=randi(T); %< or if you want to pick a scan in random.
Y(:,Idx_OL)=Y(:,Idx_OL)+1;
[DVARS,Stat]=DVARSCalc(Y,'TestMethod','X2','VarType','IQR','MeanType',1,'TransPower',1/3,'RDVARS');
```

```
find(Stat.pvals<0.05./(T-1)) %using Bonferroni correction for Multiple Comparison Error
ans =
   462   463
Stat.RDVARS(460:465) %Relative DVARS (Nichols, 2013)
ans =
  0.9965    0.9983    1.7247    1.7310    1.0032    1.0031
Stat.SDVARS_X2(460:465) %Z(D-var)
ans =
     -1.2125   -0.6855  285.8207  289.0012    0.7476    0.7119

```
It is also worth mentioning that, input can be a path to a nifti file for neuroimaging analysis:

```
Input_Path='~/path/to/your/image.nii.gz';
[DVARS,Stat]=DVARSCalc(Input_Path,'RDVARS');
```

## DSE Variance Decomposition
Using `DSEvars.m` allows the DSE (fast, slow and edge) variance decomposition.

```
I=4e4; T=1200;
Y=randn(I,T);
[V,Stat]=DSEvars(Y);
```

```
-Input is a Matrix.
-Extra-cranial areas removed: 40000x1200
-No normalisation/scaling has been set!
-Data centred. Untouched Grand Mean: -6.3211e-05, Post-norm Grand Mean: -6.3211e-05, Post demean: -5.7165e-20
-Variance images will NOT be saved:
-- Either destination directory was not set OR the input is not a nifti.
----------------------
Sum-of-Mean-Squared (SMS) Table
                  Avar    Dvar    Svar    Evar
                  ____    ____    ____    ____

    Whole         1198    599     598     1   
    Global           0      0       0     0   
    non-Global    1198    599     598     1   

------------ANOVA table:
                  MS           RMS        Percentage_of_whole    Relative_to_iid
              __________    __________    ___________________    _______________

    Avar         0.99897       0.99949           100                   1        
    Dvar         0.49957        0.7068        50.009               1.001        
    Svar         0.49857       0.70609        49.908             0.99899        
    Evar      0.00083476      0.028892      0.083562              1.0027        
    g_Avar    2.4021e-05     0.0049011     0.0024046             0.96184        
    g_Dvar    1.1663e-05     0.0034151     0.0011675             0.93478        
    g_Svar     1.234e-05     0.0035129     0.0012353             0.98907        
    g_Evar    1.7706e-08    0.00013306    1.7724e-06             0.85076        

----------------------
```

## Visualisation
We suggest to visualise the DSE variance components and significant DVARS data-points
as proposed in paper (see Fig. 3, Fig. 4 and Fig. 5). This can easily done as below:

```
Input_Path='~/path/to/your/image.nii.gz';
[V,Stat] = DSEvars('Input_Path')

f_hndl=figure('position',[50,500,600,500]); hold on;
fMRIDiag_plot(V,'handle',f_hndl)
```
