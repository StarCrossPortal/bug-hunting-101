# bug-hunting-101

## What it about
As a freshman in bug hunting, I write these to exercise bug hunting skill.




## What will I do
The learning path is divided into three steps:
- In level 1, I will try to "rediscover" some cve without patch, this means I will search some cve report, just like [this](https://talosintelligence.com/vulnerability_reports/TALOS-2020-1127). And try to find the vulnerability with the help of "Details" which explain what is this cve about. It just like do ctf challenges, I guess it can be easy for you ;) 
- In level 2, I will replay level 1 without "Details". But in order to narrow down, I need know which part of the project exist this bug, for example, if the bug exist in ANGEL (one module of Chromium), I need some info about which file I need analysis. Most of the time, the path of the patch file can help me achieve this goal.
- Level 3, I will do as level 2 and construct Poc and exp(optional).




## LEVEL 1
|                Exercise No.                |        CVEs        |    Target    |
| :----------------------------------------: | :----------------: | :----------: |
| [LEVEl_1/exercise_1](./LEVEL_1/exercise_1) | **CVE-2020-6542**  | Chrome WebGL |
| [LEVEl_1/exercise_2](./LEVEL_1/exercise_2) | **CVE-2020-6542**  | Chrome ANGLE |
| [LEVEl_1/exercise_3](./LEVEL_1/exercise_3) | **CVE-2020-16005** |    ANGLE     |
| [LEVEl_1/exercise_4](./LEVEL_1/exercise_4) | **CVE-2021-21204** | Chrome Blink |
| [LEVEl_1/exercise_5](./LEVEL_1/exercise_5) | **CVE-2021-21203** |    Blink     |
| [LEVEl_1/exercise_6](./LEVEL_1/exercise_6) | **CVE-2021-21188** |    Blink     |
| [LEVEl_1/exercise_7](./LEVEL_1/exercise_7) | **CVE-2021-30565** |    V8 GC     |



## LEVEL 2

|               Exercise No.               |        CVEs        |    Target     |
| :--------------------------------------: | :----------------: | :-----------: |
| [LEVEL_2/exercise_1](LEVEL_2/exercise_1) | **CVE-2021-21128** |     Blink     |
| [LEVEL_2/exercise_2](LEVEL_2/exercise_2) | **CVE-2021-21122** |     Blink     |
| [LEVEL_2/exercise_3](LEVEL_2/exercise_3) | **CVE-2021-21112** |     Blink     |
| [LEVEL_2/exercise_4](LEVEL_2/exercise_4) | **CVE-2021-30565** |  Chrome Tab   |
| [LEVEL_2/exercise_5](LEVEL_2/exercise_5) | **CVE-2021-21159** |      Tab      |
| [LEVEL_2/exercise_6](LEVEL_2/exercise_6) | **CVE-2021-21190** | Chrome pdfium |
| [LEVEL_2/exercise_7](LEVEL_2/exercise_7) | **CVE-2020-6422**  |     Blink     |



## LEVEL 3

|                Exercise No.                |        CVEs        |        Target        |
| :----------------------------------------: | :----------------: | :------------------: |
| [LEVEl_3/exercise_1](./LEVEL_3/exercise_1) | **CVE-2021-21226** | navigation_predictor |
| [LEVEl_3/exercise_2](./LEVEL_3/exercise_2) | **CVE-2021-21224** |          V8          |
| [LEVEl_3/exercise_3](./LEVEL_3/exercise_3) | **CVE-2021-21223** |         mojo         |
| [LEVEl_3/exercise_4](./LEVEL_3/exercise_4) | **CVE-2021-21207** |       IndexDB        |
| [LEVEl_3/exercise_5](./LEVEL_3/exercise_5) | **CVE-2021-21202** |      extensions      |
| [LEVEl_3/exercise_6](./LEVEL_3/exercise_6) | **CVE-2021-21198** |         IPC          |
|                                            |                    |                      |





## How to contribute

Writing your exercise follow [this](./Template.md) format.



