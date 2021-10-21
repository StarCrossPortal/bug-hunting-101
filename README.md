# bug-hunting-101

## What it about
As a freshman in bug hunting, I found few resource help me to learn form zero to one. But we can enhance our power by ourselves, so I write these to record how I exercise bug hunting step by step.


## What will I do
The learning path is divided into three steps:
- In level 1, I will try to "rediscover" some cve without patch, this means I will search some cve report, just like [this](https://talosintelligence.com/vulnerability_reports/TALOS-2020-1127), whithout analysis blog before. And I try to find the vulnerability with the help of "Details" which explain what is this cve about. So we can know where does the bug occur roughly, It just like do ctf challenges, I guess it can be easy for you ;) , tips: we can do it by code audit or fuzzing!
- In level 2, I will replay level 1 without "Details". But in order to narrow down, I need know which part of the project exist this bug, for example, if the bug exist in ANGEL (one module of Chromium), I hope get some info about which file of whole process I need analysis. Obviously I can not analysis entire ANGEL, needless to say Chromium :(
- Level 3(maybe), I will find some projects which have similar features, and find some new bugs which have similar characteristics, codeql may also be used.



The above are just my preliminary plan, I can't promise that I'll always follow that, I'll update if something change.


## How to contribute

Writing your exercise follow [this](./Template.md) format.




