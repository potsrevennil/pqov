  
# OV signature

This project contains the reference and optimized implementations of the oil and vinegar (OV) signature system.

## Licence

Public domain unless stated differently at the top of each file.

## Parameters

| Parameter    | signature size | pk size  | sk size | pkc size | compressed-sk size |  
|:-----------: |:--------------:|--------- |---------|------------|--------------------|
|GF(16),160,64 | 96             |412,160   |348,704  |66,576      | 48                 |
|GF(256),112,44| 128            |278,432   |237,896  |43,576      | 48                 |
|GF(256),184,72| 200            |1,225,440 |1,044,320|189,232     | 48                 |
|GF(256),244,96| 260            |2,869,440 |2,436,704|446,992     | 48                 |

## Contents

- **src** : Source code.
- **utils**  : utilities for AES, SHAKE, and PRNGs. The default setting calls openssl library.
- **unit_tests**  : unit testers.

## Generate NIST and SUPERCOP projects

1. NIST submission:  
  * creat basic nist project.
```
python3 ./create_nist_project.py
```
  * generate KAT files. (I personally recommend skipping this step for version controls of the source code because the size of KATs are enomerous.)
```
cd pqov_nist_submission/Optimized_Implementation/avx2/
source ./generate_KAT.sh
mv KAT ../../
cd ../../../
```
  * generate file descriptions required from NIST.
```
cd pqov_nist_submission/
ls -alR | python ../generate_filedescriptions_for_nist.py >> README
cd ../
```

2. SUPERCOP:  
```
python3 ./create_supercop_project.py
```


## Instructions for testing/benchmarks

Type **make**    
```
make
```
for generating 3 executables:  
1. sign_api-test : testing for API functions (crypto_keygen(), crypto_sign(), and crypto_verify()).  
2. sign_api-benchmark: reporting performance numbers for signature API functions.  
2. rec-sign-benchmark: reporting more detailed performance numbers for signature API functions. Number format: ''average /numbers of testing (1st quartile, median, 3rd quartile)''  


### **Options for Parameters:**

For compiling different parameters, we use the macros ( **_OV256_112_44** / **_OV256_184_72** / **_OV256_244_96** / **_OV16_160_64** ) to control the C source code.  
The default setting is **_OV256_112_44** defined in **src/params.h**.  

The other option is to use our makefile:  
1. **_OV16_160_64** :
```
make PARAM=1
```
2. **_OV256_112_44** :
```
make
```
or
```
make PARAM=3
```
3. **_OV256_184_72** :
```
make PARAM=4
```
4. **_OV256_244_96** :
```
make PARAM=5
```


### **Options for Variants:**

For compiling different variants, we use the macros ( **_OV_CLASSIC** / **_OV_PKC** / **_OV_PKC_SKC** ) to control the C source code.  
The default setting is **_OV_CLASSIC** defined in **src/params.h**.  

The other option is to use our makefile:  
1. **_OV_CLASSIC** :
```
make
```
or
```
make VARIANT=1
```
2. **_OV_PKC** :
```
make VARIANT=2
```
3. **_OV_PKC_SKC** :
```
make VARIANT=3
```
4. **_OV256_244_96** and **_OV_PKC** :
```
make VARIANT=2 PARAM=5
```

### **Optimizations for Architectures:**

#### **Reference Version:**
The reference uses (1) source code in the directories: **src/** , **src/ref/**, and  
(2) directories for utilities of AES, SHAKE, and randombytes() : **utils/** .  
The default implementation for AES and SHAKE is from openssl library, controlled by the macro **_UTILS_OPENSSL\_** defined in **src/config.h**.  

Or, use our makefile:  
1. Reference version (**_OV256_112_44** and **_OV_CLASSIC**):
```
make
```
2. Reference version, **_OV256_244_96** , and **_OV_PKC** :
```
make VARIANT=2 PARAM=5
```


To turn on the option of 4-round AES, one need to turn on the macro **_4ROUND_AES\_** defined in **src/params.h**.  


#### **AVX2 Version:**
The AVX2 option uses (1) source code in the directories: **src/** , **src/amd64** , **src/ssse3** , **src/avx2**, and  
(2) directories for utilities of AES, SHAKE, and randombytes() : **utils/**, **utils/x86aesni** .  
(3) One stil need to turn on the macros **_BLAS_AVX2\_**, **_MUL_WITH_MULTAB\_**, **_UTILS_AESNI\_** defined in **src/config.h** to enable AVX2 optimization.  

Or, use our makefile:  
1. AVX2 version (**_OV256_112_44** and **_OV_CLASSIC**):
```
make PROJ=avx2
```
2. AVX2 version, **_OV256_184_72**, and **_OV_PKC**:
```
make PROJ=avx2 PARAM=4 VARIANT=2
```

#### **NEON Version:**
The NEON option uses (1) source code in the **src/** , **src/amd64** , **src/neon**, and  
(2) directories for utilities of AES, SHAKE, and randombytes() : **utils/**, ( **utils/neon_aesinst** (Armv8 AES instruction) or **utils/neon_aes**(NEON bitslice AES implemetation) ).  
(3) One stil need to turn on the macros **_BLAS_NEON\_** , **_UTILS_NEONAES\_** defined in **src/config.h** to enable NEON optimization.  
(4) Depending on the CPUs and parameters, one can choose to define the macro **_MUL_WITH_MULTAB\_** for GF multiplication with MUL tables. We suggest to turn on it for the **_OV16_160_64** parameter.  

Or, use our makefile:  
1. NEON version (**_OV256_112_44** and **_OV_CLASSIC**):
```
make PROJ=neon
```
2. Another example: NEON version, **_OV16_160_64**, and **_OV_PKC_SKC**:
```
make PROJ=neon PARAM=1 VARIANT=3
```

Notes for **Apple Mac M1**:  
1. We use
```
uname -s
```
to detect if running on Mac OS and 
```
uname -m
```
to detect if is an Arm-based Mac.
If `uname -s` returns **Darwin** and `uname -m` returns **arm64, we are running
on an Arm-based Mac (e.g., Apple M1).
The Makefile will then define the **_APPLE_SILICON\_** macro for enabling some optimization settings in the source code .  
2. The program needs **sudo** to benchmark on Mac OS correctly.


### **Options for Algorithm of Solving Linear Equation while Signing:**
1. Default setting: Gaussian Elimination and backward substitution.
2. Choose the algorithm of calculating inversion matrix with block matrix compution:  
  (a) Define the **_LDU_DECOMPOSE\_** macro in **src/parms.h**.  
  (b) Remove the **_BACK_SUBSTITUTION\_** macro in **src/ov.c**.  


## Benchmarks

1. AVX2 implementations on Intel(R) Xeon(R) CPU E3-1275 v5 @ 3.60GHz (Skylake), turbo boost off

| Parameter    | Key-generation cycles | signing cycles | sign-opening cycles |
|:-----------: |        --------------:|      ---------:|           ---------:|
| OV(16,160,64)-classic |  4404165.040000 /100 (4328746,4332050,4337824) |  116828.822000 /1000 (109180,109314,109476) |  58255.892000 /1000 (57504,58274,59024) |
| OV(16,160,64)-cpk |  4443962.080000 /100 (4372956,4376338,4381242) |  117521.884000 /1000 (110712,110824,110980) |  281266.102000 /1000 (280190,280946,281714) |
| OV(16,160,64)-cpk-csk |  4517415.360000 /100 (4448068,4450838,4455366) |  2480241.982000 /1000 (2471938,2473254,2475538) |  276703.810000 /1000 (275764,276520,277334) |
| OV(256,112,44)-classic |  2971691.280000 /100 (2901932,2903434,2905938) |  105757.650000 /1000 (105176,105324,105472) |  90378.776000 /1000 (89572,90336,91088) |
| OV(256,112,44)-cpk |  2924426.540000 /100 (2857738,2858724,2861124) |  107061.638000 /1000 (106332,106508,106688) |  224355.520000 /1000 (223762,224006,224286) |
| OV(256,112,44)-cpk-csk |  2913461.180000 /100 (2846270,2848774,2850886) |  1877833.322000 /1000 (1874834,1876442,1878078) |  224512.708000 /1000 (224076,224374,224684) |
| OV(256,184,72)-classic |  17690951.620000 /100 (17599644,17603360,17607160) |  300090.748000 /1000 (299110,299316,299532) |  241705.192000 /1000 (241098,241588,242066) |
| OV(256,184,72)-cpk |  17610902.060000 /100 (17532234,17534058,17537520) |  300786.974000 /1000 (299790,300010,300258) |  918983.580000 /1000 (916888,917402,918112) |
| OV(256,184,72)-cpk-csk |  17248300.300000 /100 (17154892,17157802,17161764) |  9950429.662000 /1000 (9898656,9965110,9995552) |  921044.166000 /1000 (920010,920616,921214) |
| OV(256,244,96)-classic |  48641435.640000 /100 (48457126,48480444,48500076) |  595198.860000 /1000 (589688,591812,594184) |  471052.054000 /1000 (468914,470886,472852) |
| OV(256,244,96)-cpk |  46753567.760000 /100 (46652250,46656796,46660876) |  597267.148000 /1000 (591802,594268,597160) |  2042818.886000 /1000 (2036396,2038408,2041114) |
| OV(256,244,96)-cpk-csk |  45627154.160000 /100 (45488036,45492216,45499014) |  22993692.878000 /1000 (22986750,22992816,22998322) |  2033715.538000 /1000 (2030920,2032992,2035192) |

2. With _4RAES_ macro, AVX2 implementations on Intel(R) Xeon(R) CPU E3-1275 v5 @ 3.60GHz (Skylake), turbo boost off

| Parameter    | Key-generation cycles | signing cycles | sign-opening cycles |
|:-----------: |        --------------:|      ---------:|           ---------:|
| OV(16,160,64)-classic |  4430025.800000 /100 (4356148,4358736,4364702) |  116570.692000 /1000 (109868,109996,110148) |  58059.062000 /1000 (57294,58050,58798) |
| OV(16,160,64)-cpk |  4405616.080000 /100 (4334642,4337958,4343666) |  117115.824000 /1000 (110470,110602,110760) |  168166.006000 /1000 (167288,167886,168470) |
| OV(16,160,64)-cpk-csk |  4318530.000000 /100 (4247340,4252570,4257254) |  2373560.138000 /1000 (2365812,2366766,2368256) |  168930.000000 /1000 (168132,168830,169436) |
| OV(256,112,44)-classic |  2868175.480000 /100 (2796552,2798734,2804066) |  106530.948000 /1000 (105906,106098,106268) |  89335.626000 /1000 (88582,89296,90126) |
| OV(256,112,44)-cpk |  2881418.040000 /100 (2814360,2815902,2819468) |  106800.630000 /1000 (106160,106336,106526) |  151526.502000 /1000 (150976,151236,151506) |
| OV(256,112,44)-cpk-csk |  2926573.520000 /100 (2858332,2861082,2867376) |  1819385.352000 /1000 (1817018,1818690,1819480) |  151029.002000 /1000 (150656,150902,151204) |
| OV(256,184,72)-classic |  17362839.880000 /100 (17272786,17274800,17278294) |  301435.272000 /1000 (299614,299818,300032) |  241093.452000 /1000 (240294,240958,241602) |
| OV(256,184,72)-cpk |  17518167.760000 /100 (17437532,17441792,17447672) |  301554.380000 /1000 (300466,300716,300982) |  591127.750000 /1000 (589324,589846,590356) |
| OV(256,184,72)-cpk-csk |  17001182.380000 /100 (16905392,16909288,16914922) |  9613919.664000 /1000 (9598624,9603518,9605894) |  591259.966000 /1000 (590390,590902,591422) |
| OV(256,244,96)-classic |  48088795.340000 /100 (47909346,47918484,47931396) |  589577.194000 /1000 (586850,587744,589090) |  461104.490000 /1000 (459508,460642,461702) |
| OV(256,244,96)-cpk |  45604738.980000 /100 (45505002,45508552,45512622) |  627582.498000 /1000 (620602,624774,629524) |  1286277.362000 /1000 (1279818,1282392,1285530) |
| OV(256,244,96)-cpk-csk |  44926343.860000 /100 (44787368,44792434,44801692) |  21833937.212000 /1000 (21817004,21823506,21830864) |  1271237.518000 /1000 (1266868,1268998,1270808) |


3. AVX2 implementations on Intel(R) Xeon(R) CPU E3-1230L v3 @ 1.80GHz (Haswell)), turbo boost off

| Parameter    | Key-generation cycles | signing cycles | sign-opening cycles |
|:-----------: |        --------------:|      ---------:|           ---------:|
| OV(16,160,64)-classic |  5016036.240000 /100 (4937420,4945376,4954124) |  131392.752000 /1000 (123156,123376,123620) |  60904.608000 /1000 (60076,60832,61504) |
| OV(16,160,64)-cpk |  5067757.040000 /100 (4992400,5002756,5013412) |  130966.108000 /1000 (123668,123908,124156) |  399596.960000 /1000 (397648,398596,399556) |
| OV(16,160,64)-cpk-csk |  5515641.200000 /100 (5443368,5448272,5457180) |  3052952.056000 /1000 (3038012,3042756,3052016) |  400434.368000 /1000 (398800,399724,400624) |
| OV(256,112,44)-classic |  3403510.500000 /100 (3304036,3311188,3317840) |  122641.960000 /1000 (121632,121772,121948) |  82859.460000 /1000 (82420,82668,83012) |
| OV(256,112,44)-cpk |  3459328.000000 /100 (3391588,3393872,3400424) |  116985.104000 /1000 (113816,116624,118816) |  313062.148000 /1000 (311876,312284,312712) |
| OV(256,112,44)-cpk-csk |  3360401.360000 /100 (3280040,3287336,3294824) |  2253385.692000 /1000 (2244896,2251440,2257484) |  312037.884000 /1000 (311328,311720,312160) |
| OV(256,184,72)-classic |  22138835.280000 /100 (22044264,22046680,22049832) |  350406.368000 /1000 (346180,346528,346836) |  275809.088000 /1000 (274848,275216,275652) |
| OV(256,184,72)-cpk |  22470932.960000 /100 (22385912,22389144,22394876) |  348133.452000 /1000 (345920,346424,346868) |  1282573.512000 /1000 (1278228,1280160,1281668) |
| OV(256,184,72)-cpk-csk |  21868033.520000 /100 (21777396,21779704,21782444) |  11381519.284000 /1000 (11368148,11381092,11388020) |  1283091.132000 /1000 (1279344,1281232,1282876) |
| OV(256,244,96)-classic |  58309948.240000 /100 (58156052,58162124,58172156) |  695597.460000 /1000 (689308,690752,693412) |  515532.460000 /1000 (513292,514100,515140) |
| OV(256,244,96)-cpk |  57414118.200000 /100 (57310516,57315504,57325072) |  734795.140000 /1000 (724380,726632,735368) |  2846586.460000 /1000 (2837992,2842448,2847740) |
| OV(256,244,96)-cpk-csk |  57445240.400000 /100 (57301900,57306980,57313992) |  26031018.856000 /1000 (26013280,26021784,26031760) |  2843408.596000 /1000 (2838024,2842416,2846564) |

4. Wit _4RAES_ macro, AVX2 implementations on Intel(R) Xeon(R) CPU E3-1230L v3 @ 1.80GHz (Haswell)), turbo boost off

| Parameter    | Key-generation cycles | signing cycles | sign-opening cycles |
|:-----------: |        --------------:|      ---------:|           ---------:|
| OV(16,160,64)-classic |  4975791.120000 /100 (4898016,4904192,4913388) |  130077.300000 /1000 (123308,123560,123808) |  60423.632000 /1000 (59592,60364,61092) |
| OV(16,160,64)-cpk |  4867485.520000 /100 (4795252,4799564,4809548) |  123498.396000 /1000 (117820,117948,118128) |  206137.548000 /1000 (204728,205504,206312) |
| OV(16,160,64)-cpk-csk |  4878610.640000 /100 (4803228,4810612,4819392) |  2764608.984000 /1000 (2751324,2755060,2765580) |  208606.620000 /1000 (207436,208188,208912) |
| OV(256,112,44)-classic |  3286078.300000 /100 (3197572,3204260,3214664) |  122923.608000 /1000 (121896,122028,122152) |  96773.664000 /1000 (96208,96644,97064) |
| OV(256,112,44)-cpk |  3196467.980000 /100 (3122680,3130128,3138036) |  115342.300000 /1000 (113108,114012,114976) |  196975.472000 /1000 (195972,196404,196840) |
| OV(256,112,44)-cpk-csk |  3219533.400000 /100 (3146820,3154404,3159528) |  2115833.688000 /1000 (2109292,2113924,2119136) |  182389.296000 /1000 (181824,182100,182376) |
| OV(256,184,72)-classic |  21494207.180000 /100 (21367644,21370252,21373736) |  348552.644000 /1000 (346504,346820,347232) |  275686.680000 /1000 (274668,275024,275476) |
| OV(256,184,72)-cpk |  21505762.640000 /100 (21416272,21419104,21423720) |  349823.884000 /1000 (348344,348756,349204) |  716735.288000 /1000 (713804,714252,714844) |
| OV(256,184,72)-cpk-csk |  21294242.160000 /100 (21200464,21203604,21206856) |  11222585.880000 /1000 (11211988,11222092,11229120) |  717092.608000 /1000 (714928,715432,716136) |
| OV(256,244,96)-classic |  57179699.360000 /100 (57023616,57049260,57070560) |  695777.216000 /1000 (687932,690340,695824) |  514094.544000 /1000 (511824,512628,513604) |
| OV(256,244,96)-cpk |  56078502.040000 /100 (55973132,55983388,55991336) |  740858.864000 /1000 (718072,723628,758304) |  1522603.924000 /1000 (1515360,1516652,1518708) |
| OV(256,244,96)-cpk-csk |  56273885.840000 /100 (56131464,56136556,56145308) |  24832148.256000 /1000 (24816540,24824672,24835632) |  1556615.036000 /1000 (1550748,1552732,1556188) |


5. neon implementation on Apple M1

| Parameter    | Key-generation cycles | signing cycles | sign-opening cycles |
|:-----------: |        --------------:|      ---------:|           ---------:|
| OV(16,160,64)-classic |  3393202.048000 /1000 (3389693,3391967,3395605) |  79649.492100 /10000 (74491,74633,74814) |  45974.790500 /10000 (44777,45908,47078) |
| OV(16,160,64)-cpk |  3366171.190000 /1000 (3359391,3360648,3363265) |  79637.568200 /10000 (74612,74757,74938) |  138389.038700 /10000 (136221,138541,140629) |
| OV(16,160,64)-cpk-csk |  3399677.154000 /1000 (3390185,3393812,3397537) |  2094305.040600 /10000 (2088941,2089131,2089364) |  138383.778500 /10000 (136248,138496,140612) |
| OV(256,112,44)-classic |  1797428.261000 /1000 (1791766,1793119,1794443) |  55554.494500 /10000 (55244,55320,55413) |  49796.923700 /10000 (49550,49719,49976) |
| OV(256,112,44)-cpk |  1779785.786000 /1000 (1774902,1775826,1776591) |  55520.226000 /10000 (55203,55289,55391) |  113035.876400 /10000 (112739,112934,113235) |
| OV(256,112,44)-cpk-csk |  1781368.708000 /1000 (1773490,1774748,1775610) |  1056925.781800 /10000 (1056437,1056617,1056802) |  113889.868100 /10000 (113626,113805,114074) |
| OV(256,184,72)-classic |  9842855.186000 /1000 (9833331,9836359,9842447) |  149716.423600 /10000 (148561,149280,149439) |  189911.501600 /10000 (189580,189837,190151) |
| OV(256,184,72)-cpk |  9808371.327000 /1000 (9801978,9803637,9805500) |  148068.639000 /10000 (147459,147564,147708) |  462334.340400 /10000 (461835,462306,462730) |
| OV(256,184,72)-cpk-csk |  9761487.225000 /1000 (9749338,9751198,9753375) |  6353695.307200 /10000 (6353126,6353401,6353665) |  462173.811200 /10000 (461515,461896,462369) |
| OV(256,244,96)-classic |  28294004.860000 /1000 (28275650,28286979,28297149) |  298569.667200 /10000 (293517,293826,294532) |  376079.556800 /10000 (375692,376000,376372) |
| OV(256,244,96)-cpk |  26889989.542000 /1000 (26710677,26743866,26754034) |  310212.719500 /10000 (308849,309159,309459) |  1012621.870400 /10000 (1012091,1012482,1012937) |
| OV(256,244,96)-cpk-csk |  26750917.303000 /1000 (26661383,26663940,26674888) |  15836867.328600 /10000 (15829550,15830169,15831069) |  1015436.129700 /10000 (1010911,1011331,1012047) |


6. neon implementation on Apple M1, defining _4RAES_ macro

| Parameter    | Key-generation cycles | signing cycles | sign-opening cycles |
|:-----------: |        --------------:|      ---------:|           ---------:|
| OV(16,160,64)-classic |  3354871.311000 /1000 (3348860,3349853,3351112) |  79175.906800 /10000 (74372,74515,74696) |  45800.548600 /10000 (44629,45711,46863) |
| OV(16,160,64)-cpk |  3327670.425000 /1000 (3321012,3324331,3326806) |  79323.975100 /10000 (74366,74503,74679) |  97395.223100 /10000 (95793,97391,98984) |
| OV(16,160,64)-cpk-csk |  3352449.915000 /1000 (3342295,3349000,3351955) |  2050209.412400 /10000 (2044850,2045042,2045331) |  97323.289600 /10000 (95756,97325,98867) |
| OV(256,112,44)-classic |  1766518.643000 /1000 (1761364,1762817,1763931) |  55363.380500 /10000 (55097,55175,55267) |  49893.352300 /10000 (49644,49808,50070) |
| OV(256,112,44)-cpk |  1749851.720000 /1000 (1745637,1746623,1747825) |  55542.583600 /10000 (55244,55329,55431) |  83097.113800 /10000 (82796,83021,83311) |
| OV(256,112,44)-cpk-csk |  1751771.015000 /1000 (1746535,1748646,1750924) |  1027002.902200 /10000 (1026583,1026701,1026841) |  84001.746000 /10000 (83738,83920,84191) |
| OV(256,184,72)-classic |  9694194.041000 /1000 (9688797,9690791,9692931) |  149529.498900 /10000 (148460,149120,149299) |  189887.573600 /10000 (189566,189817,190140) |
| OV(256,184,72)-cpk |  9645633.052000 /1000 (9639195,9640984,9643113) |  147986.401300 /10000 (147396,147524,147666) |  330573.232900 /10000 (330084,330463,330877) |
| OV(256,184,72)-cpk-csk |  9650337.274000 /1000 (9644063,9645510,9648015) |  6222689.873100 /10000 (6221007,6221280,6221588) |  331661.876800 /10000 (330625,331074,331700) |
| OV(256,244,96)-classic |  27859451.168000 /1000 (27846303,27852840,27859809) |  299725.738900 /10000 (292736,293117,308054) |  376210.326900 /10000 (375814,376132,376506) |
| OV(256,244,96)-cpk |  26580995.262000 /1000 (26298820,26305292,26340264) |  309852.521500 /10000 (308127,308410,308684) |  707297.318200 /10000 (706823,707217,707643) |
| OV(256,244,96)-cpk-csk |  26643439.819000 /1000 (26289085,26298657,26354765) |  15526142.527500 /10000 (15522021,15522513,15523170) |  706816.234400 /10000 (704644,704986,705439) |


7. neon implementation on ARM Cortex-A72(v8) @ 1.8GHz (Raspberry pi 4, aarch64, no aes extension)

| Parameter    | Key-generation cycles | signing cycles | sign-opening cycles |
|:-----------: |        --------------:|      ---------:|           ---------:|
| OV(16,160,64)-classic |  29290685.950000 /500 (29239158,29269925,29310858) |  494872.201800 /10000 (458114,460655,470144) |  143442.171400 /10000 (139658,141528,143558) |
| OV(16,160,64)-cpk |  28936858.182000 /500 (28894743,28906183,28925222) |  1580336802.216000 /10000 (542228,549105,559817) |  5080532.816700 /10000 (4993594,5087095,5174934) |
| OV(16,160,64)-cpk-csk |  29483404.598000 /500 (29451741,29467684,29491076) |  1395940904.282800 /10000 (16401080,16413501,16435793) |  1844672822620759.000000 /10000 (4974918,5070253,5156379) |
| OV(256,112,44)-classic |  11188099.140000 /500 (11163154,11172204,11185089) |  247106.639500 /10000 (244375,245095,245897) |  143717.544900 /10000 (142618,142868,143161) |
| OV(256,112,44)-cpk |  11224762.366000 /500 (11186542,11193794,11198712) |  256224.640400 /10000 (251210,253324,255524) |  3678706.104100 /10000 (3668527,3681898,3684604) |
| OV(256,112,44)-cpk-csk |  11295081.128000 /500 (11219457,11229231,11318013) |  7621048.374000 /10000 (7616217,7617137,7618574) |  3674931.824400 /10000 (3666397,3677844,3679458) |
| OV(256,184,72)-classic |  66931801.010000 /500 (66731124,66871027,67029434) |  1557611.746800 /10000 (1520321,1542143,1587587) |  588840.363000 /10000 (571977,574080,589334) |
| OV(256,184,72)-cpk |  66570566.088000 /500 (66275754,66554826,66793786) |  1659039.631200 /10000 (1623087,1632966,1692327) |  1374057048.781200 /10000 (17122760,17161246,17200844) |
| OV(256,184,72)-cpk-csk |  64199301.720000 /500 (64056764,64147364,64275590) |  42820542.201000 /10000 (42746359,42794977,42855995) |  1336706755.118000 /10000 (17338010,17369004,17405721) |
| OV(256,244,96)-classic |  313822979.386000 /500 (313313865,313814250,314327646) |  3347921.603300 /10000 (3272056,3337536,3403568) |  1342670.939700 /10000 (1310429,1319092,1390676) |
| OV(256,244,96)-cpk |  306220128.888000 /500 (305215682,305700907,306292823) |  3324891.704800 /10000 (3245750,3316413,3385823) |  39345343.824300 /10000 (39295821,39337795,39385602) |
| OV(256,244,96)-cpk-csk |  312769807.848000 /500 (312264832,312729427,313148491) |  107318888.779000 /10000 (107227206,107305680,107381535) |  39389996.820300 /10000 (39326493,39390657,39452436) |


7. neon implementation on ARM Cortex-A72(v8) @ 1.8GHz, defining _4RAES_ macro (Raspberry pi 4, aarch64, no aes extension)

| Parameter    | Key-generation cycles | signing cycles | sign-opening cycles |
|:-----------: |        --------------:|      ---------:|           ---------:|
| OV(16,160,64)-classic |  26104042.366000 /500 (26055865,26079083,26109987) |  479892.969200 /10000 (445703,448188,454231) |  135681.459000 /10000 (131743,133721,136119) |
| OV(16,160,64)-cpk |  25712346.908000 /500 (25690772,25698880,25710821) |  550544.434100 /10000 (508517,517451,530845) |  2308786.814100 /10000 (2264483,2309671,2354513) |
| OV(16,160,64)-cpk-csk |  28350136.382000 /500 (28308338,28324760,28346176) |  13374690.195600 /10000 (13328127,13333557,13351565) |  2263162.810800 /10000 (2223998,2266233,2304816) |
| OV(256,112,44)-classic |  9491444.574000 /500 (9419551,9436542,9450911) |  257070.261600 /10000 (252719,254211,255892) |  153938.584600 /10000 (151079,152130,153527) |
| OV(256,112,44)-cpk |  9196501.852000 /500 (9181131,9191247,9197749) |  253417.475800 /10000 (248689,249910,252309) |  1678078.192600 /10000 (1671641,1672544,1686208) |
| OV(256,112,44)-cpk-csk |  9478628.272000 /500 (9466694,9473513,9481548) |  5631076.342600 /10000 (5625684,5627393,5637580) |  1678325.878400 /10000 (1673384,1674207,1683582) |
| OV(256,184,72)-classic |  2371867147.266000 /500 (57079160,57311639,57579767) |  1595025.065900 /10000 (1558627,1569429,1623583) |  587322.646100 /10000 (570475,572965,586951) |
| OV(256,184,72)-cpk |  56918872.966000 /500 (56663215,56890636,57122899) |  1758320.144800 /10000 (1720138,1729733,1797286) |  8319356.344400 /10000 (8282039,8318527,8349960) |
| OV(256,184,72)-cpk-csk |  56859414.722000 /500 (56676866,56815652,56969672) |  1844673000101042.500000 /10000 (34488140,34533235,34600256) |  8636187.722000 /10000 (8602234,8634365,8663370) |
| OV(256,244,96)-classic |  295923082.904000 /500 (295058951,295693403,296663039) |  3352007.747000 /10000 (3270463,3339648,3408099) |  1352412.647500 /10000 (1320262,1328232,1398580) |
| OV(256,244,96)-cpk |  36893485758928136.000000 /500 (282248561,282742682,283172951) |  3387885.507700 /10000 (3312617,3376153,3440469) |  18848531.373100 /10000 (18789243,18831724,18894871) |
| OV(256,244,96)-cpk-csk |  291461557.636000 /500 (290948203,291438637,291896182) |  1419459809.967000 /10000 (86648551,86727909,86825980) |  18629406.251200 /10000 (18521783,18602008,18724379) |
