# bug-hunting-101

## What is this?

This repository is to help new-comers (like ourselves) of binary bug hunting area to improve their skills.

Currently, the gap between CTF and real world bug hunting can be quite huge.
And this repository is our attempt to solve that problem by porting the real world bug hunting to small exercises.

CVEs are selected out and setup in a certain scene, your goal is to repeat the process of finding such vulnerabilities out.

## Intro

We have prepared 3 levels.
Each level provides excersises with different difficulties:

- Level 1: Details of the CVEs are provided to help you from "re-discovering" the original vulnerability. Reports like [this](https://talosintelligence.com/vulnerability_reports/TALOS-2020-1127) are provided.
So this should be the easiest level.
- Level 2: the details will be emitted. But to narrow down, information about which part of the project contains such vulnerability will be provided. For example, if the bug is about ANGEL (module of the Chromium project),
the information about the file will be provided.
Most of the time, the path to the patch file should help that.
- Level 3: quite like level 2, but need PoC and exploit (optional)


## LEVEL 1
|                Exercise No.                |        CVEs        |    Target    |
| :----------------------------------------: | :----------------: | :----------: |
| [LEVEl_1/exercise_1](./LEVEL_1/exercise_1) | **CVE-2020-6542**  | Chrome WebGL |
| [LEVEl_1/exercise_2](./LEVEL_1/exercise_2) | **CVE-2020-6463**  | Chrome ANGLE |
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
| [LEVEl_3/exercise_7](./LEVEL_3/exercise_7) | **CVE-2021-21155** |         Tab          |


## Original Author

- [ddme](https://github.com/ret2ddme)
- [lime](https://github.com/Limesss)


## How to contribute

Writing your exercise follow [this](./Template.md) format.



