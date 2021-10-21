# Exercise x


## CVE-xxxx-xxxx
I sugget you don't search any report about it to prevents get too much info like patch. Our goal is to find some bugs, not construct Poc. But in truth, Poc can proof that we are right.


### Details

#### In level 1
In level 1, you can write some describ of how the bug form. In short just ctrl+c/ctrl+v from patch, but must some useful words.

**Good example**:
> Prior to the patch, the validity of the execution context was only
> checked on entry to the method; however, the execution context can
> be invalidated during the course of parsing keyframes or options.
> The parsing of options is upstream of Animatable::animate and caught by
> the existing check, but invalidation during keyframe parsing could fall
> through triggering a crash. 

**Bad example**
> Use after free in Blink in Google Chrome prior to 89.0.4389.72 allowed a remote attacker to potentially exploit heap corruption via a crafted HTML page.

#### In level 2
In level 2, this part should be banned


### Set environment

the way to get/download the source code, or how to compile it and others.


### Related code

some related source code or file path


### Do it
Do this exercise by yourself, when you have some idea, you can compare your answer with me. If you find my answer have something wrong, please tell me.


---------

<details>
  <summary>My answer</summary>

   [ write your answer here ]

</details>

--------
