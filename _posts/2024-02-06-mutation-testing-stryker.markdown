---
layout: post
title:  "Mutation testing with StrykerJS"
date:   2024-02-06 08:00:00 +0200
categories: Testing
---

<style>

  @media(min-width:1000px){
    .wrapper{
      max-width:80%;
    }
  }

  details{
    margin-bottom:36px;
  }

  iframe{
    margin-bottom:24px;
  }

</style>


Looking for a test case for your test cases?  Mutation testing might be an option worth considering. In this blog we will explain what is mutation testing and try it out with StrykerJS on the [Voca library](https://vocajs.pages.dev/) for string manipulations. 


## Mutation Testing
Mutation testing is a way to check how well your test suite is covering your code. Simply put, it works by introducing an error in the source code, for instance by changing the plus operator (+) to the minus operator (-), or hanging true to false. Every such code version with a change is called a mutant. For every mutant the whole test suite will be executed. And after execution at least one test must fail, since an error was introduced. If this happens, all is good: the mutant was detected and killed. What is bad is when mutants survive, when there are no failing tests in the suite after introducting an error. That means there are test cases missing, or even worse, errors in the source code.

An example of the type of changes the mutation testing makes is changing *arithmatic operators* (plus to minus, division to multiplication) and *comparison expressions* (smaller then (<) to bigger then (>), true to false). It will mutate your *regex*, remove *block statements*, and even swap *native functions* (like min with max, toUpper to toLower) and empty your arrays. For a full overview, see the [documentation](https://stryker-mutator.io/docs/mutation-testing-elements/supported-mutators/) from Stryker where you can see a nice categorisation and the changes in each category. Such code changes are something you might have done at some point manually, just to see if a test is working as expected. Stryker (or any other mutation-testing framework) will do this efficiently for you. And efficiency is what you need since it is going to make lots and lots of mutations and run the test suite against every single one seperately.

So the idea is pretty straightforward, but will implementing be easy, will it find bugs, and if so, what type of bugs? Read on for run of mutation testing on the voca library, where we will try to answer these questions. Reading it will give you some idea of the types of missing tests and potential bugs you can expect to find. Word or warning, it does not make for the easiest of reading, because talking about the mutants can get technical quite quickly and is usefull if you also look of the code being tested. 


## Voca mutation tested 
### What is Voca ?
The [Voca library](https://vocajs.pages.dev/) is a popular javascript library that can be used for many string manipulations like toUpperCase toKebabCase, trimming, substringging, searching, translating, replacing, isDigit and isBlank checks etc. Most functions are small and relatively straighforward, though there are some more complex functions filled with cases, conditions and states.

### Why mutation testing for Voca? 
What is also great about voca is that it has lots and lots of [test cases](https://github.com/panzerdp/voca/tree/master/test) (It claims to have 100% test coverage). So many that it is hard to see if there are any missing. How would you improve confidence in a library that already has a large number of tests and excellent code coverage ? Having recently discovered property based tests, this might seem like a good approach, but there are just so many functions. So instead lets see if mutation testing can further improve the confidence in voca and in its test suite. 

### Setting up stryker
StrykerJS is pretty simple to setup and get started with. You can yarn/npm install it, and then it has a cli-tool for setup that will guide you through some config/settings for dealing with esm/modules ts and matching up with your test runner of choice (jest in the case of voca). See strykerJS setup page for [details](https://stryker-mutator.io/docs/stryker-js/getting-started/).  That is it really: no need to write any code. Next, You can just run stryker with `npx stryker run` (or add it to the package json commands). If you just want to run stryker on one file of source code use `npx stryker run -m 'pathToFile'`.  

Running strykerJS with the standard settings on the voca source code, will take about a half hour (4 year old xps i7 16GB ssd). Overall StrykerJS made 1500 mutants (from more then 4000 lines of source code), and for each one of those ran a test suite of 800 tests.  Almost 500 of those 1500 survived in some way or form. 

That sounds like a lot of mutants to analyse, and luckily after running they are displayed nicely in a html report that is generated for you to get some overview and start your analysis. In the report, the  "mutants" tab shows the source code with all the mutants and whether they survived. Analysing the survived mutants is the place to start since this is where you willfind new test cases / code improvements. (From the "Tests" tab you can see how many mutants where killed by each tests, though we will not go over this tab at all in this blog)

Lets analyse and see what we find, if it is just coverage problems, or if there are actual bugs also.  

### A first intro to some real issues
Lets look at the output for the tr(anslate) function, for which strykerJS finds some issues. This function 'translates characters or replaces substrings in subject'. The first parameter is the string to translate. The second paramenter is either an object with key values (with the keys being the substrings to translate and the values the translations), or a string with the characters to translate. In the latter case, there also needs to be a third parameter (string) with the values to translate to. 

In the section below you find the actual output for this one file and its helper functions. On the opening page it shows the mutation score for this file. Click on the file to see the source code and click on a line with a red dot to see a mutant that survived: 

<iframe src="{{ site.baseurl }}/media/stryker/tr.html" width="100%" height="600px">
</iframe>

#### Problem with parameters
One of the issues is on line 40. When the functions checks which of the variants is used (2 strings or one object), we see that stryker finds that changing `&&` to `||` does not cause a failure. If either one of the arguments is not a string, then the code assumes a translation object is passed. That leaves out the case in which there is only one string passed. 

Now, before adding test cases in this case one could wonder if the function is clear enough about what should happen in these cases. The function will continue as if there is an translation object (which is actually a string or undefined). This does not trigger an error because the function transforms it to an object, but the outcome of this is probably unintended (see the testcase below).  Nevertheless, as an example, adding the following 2 test would silence the mutants (and the first test also makes clear that this is probably unintended behavior):  

``` javascript
  it('parses string as translation object if only one translation string is provided', ()=>{
    expect(v.tr('0or1', 'wd')).toBe("word") // -> wd is parsed as an object {0:'w',1:'d'}
    expect(v.tr('0or1', undefined,'wd')).toBe("0or1")
  })
```

But overall, whether this is  buggy behavior, or just intended and in need of a test case, it is a nice find by Stryker.


#### Missing testcase for out of bounds
Stryker finds another issue on line 56: when iterating over the translations the validation for checking if the 'from' and 'to' translations array are not out of bounds. It shows that the validation for the values does not matter (it can be set to true and it still does not trigger a test failure). This is a situation that only happens when the 'to' string is longer then the 'from' string. Now there are test cases testing from and to strings of unqueal length, yet they are not killing the mutant:

``` javascript
  it('should ignore the extra characters from keys and values', function() {
    expect(v.tr('test strtr', 'test', 'TESTz')).toBe('TEST STrTr'); 
    expect(v.tr('test strtr', 'testz', 'TEST')).toBe('TEST STrTr');
  });
```

This is because the out of bound character (the z) is actually not in the string. If the last character to translate would be in the string, the alteration stryker makes would trigger a faulure, since then undefined (for an out of bound array index) would be added to the actual output. So in this case the code looks good, only the test case needs to be improved. Here stryker has found a situation that is not covered with a specific enough tests. With the following test case added we are saved from the mutant: 

``` javascript
 expect(v.tr('zest strtr', 'testz', 'TEST')).toBe('zEST STrTr');
```



#### Sorting function failures?
The next issues stryker points to are with the sortStringByLength function, a helper for the tr function. The mutant shows it makes no difference what happens when sorting keys of the same length. I assume this does not matter for this code, because there cannot be two unique matching strings of the same length (keys in objects must be unique, so if they are the same length, there is only one that can match, and which you translate first will not make a difference). Unlike the other finds, this one is nice in that it points out something peculiar to the function. But the code itself is not wrong and there are no missing testcases.  Rewriting the function, which to me also simplifies the code, in the following way will solve the issue:

```javascript
function sortStringByLength(str1, str2) {
  return str2.length - str1.length
}
``` 

#### Results of first look
In this first file we saw some decent finds by mutation testing. One shows the function behavior needs some explication (or extra test cases), and another mutant showed a test case that could be improved. The last example from this file also hints to some issues with mutation testing. Sometimes you write code in a way that is perfectly valid, but nevertheless leaves mutants lingering. Rewriting was easy in this case, but it will not always be like that. (There is also an issue we skipped over, but which we will explore further in the next section).


### Stubborn and Equivalance mutants 

#### Equivalent mutants
One example of a survived mutant you will struggle with is the [equivalant mutant](https://stryker-mutator.io/docs/mutation-testing-elements/equivalent-mutants/). Loosely defined, this is when a mutation to some expression might make no difference because functionally the code has the same results even though the path through the code was different. An example:

![equivalence code example]({{ site.baseurl }}/media/stryker/equi.png)

The clipNumber function clips a number to be no larger then some max value, and no smaller then some min value. The equivalence in this implementation is that the equal sign does not matter: if the number is equal to the min, it does not matter whether you return the minvalue or the actual value (same for max). So the mutant that changes <= to <  (or vice versa) will never get detected. There is no testcase that can ever fail here. 

The way to work around the issue is to either disable stryker for this line `// Stryker disable next-line all`, or to rewrite it. One way that works is: `return Math.min(Math.max(value, downLimit), upLimit)``, but I am not sure it makes the function clearer. 


#### Stubborn mutants
After running stryker on the whole code base from voca, you will notice that there seems to be a lot of early return type statements in functions for which a mutant survived.  For instance, in this slugify function (that removed fancy diacretics like ) there are three:

![early return mutants]({{ site.baseurl }}/media/stryker/early_return.gif)

Now, the early return just returns an empty string  immediately if the inputstring is also an empty string. And there is a test case for this example:  `expect(v.slugify('')).toBe('');` But what happens is that the code after the early return, returns the same result for the empty string as input. So early returns returns the same thing, just immediately. These early returns are probably added for clarity, and  maybe sometimes for speed. Anyway, this type of surviving mutants you will find a lot after running mutation testing on the voca code.

One could categorise this as some form of equivalence mutant, but maybe a more generally category is the stubborn mutant: a mutant that is hard to get rid off. These mutants are stubborn in that there are ways to tests these early returns or rewrite the code: One way would just be to make sure that some later function is not called (by spying, though for some functions this seems hard to implement). But implementing or testing it in a way that strykerJS will terminate mutants is seems more trouble then it is worth. 

To take something positive away from it, these early return do pose the question why only some of the functions have this early return and some similar functions do not. Also, one can ask if early returns do not confuse maintainers of the code into thinking something else would happen without the early return (if it is not for speed). So there is also a case to make that mutation testing spotted a area in the code that could be made more consistent. 


All in all, there are more of these time consuming possibly stubborn mutants. This would not be a problem if they are spotted easily, but to me it was not always obvious and it feels uneasy to ignore them immediately. On the upside, I think any testing form that makes you look harder or from a different perspective at the code has at least some value. 


### Some more usefull finds 
But lets not and on a low, but instead look at some other impressive finds by stryker. 

#### Missing test cases

Lets look at the replaceAll function for some examples of more missing test cases. This function searches first for positions in a string where it finds a match using a while loop. To minimize the number of times it loops, the position from which the string is searched for the match is increased by the length of the string we are searching for. But stryker finds that mutants that end up assigning 1 to this variable also works (several mutants on line 39). 

<iframe src="{{ site.baseurl }}/media/stryker/replace_all.html" width="100%" height="600px">
</iframe>


Now it does matter from which next position to search for the next match. When you only increase the search position with 1, you could end up replacing more times then warranted. By adding the following test case the mutant is silenced: 

```javascript
expect(v.replaceAll("ooo ooo ooo", "oo", "h")).toBe("ho ho ho")
```

In the replaceAll stryker finds anothe missing test cases on line 32: there is no testcase covering the situation where something other then a function or a string is passed as the replace parameter. One can again wonder what should happen with arrays and objects.

```javascript
  it("should stringify the replace parameter if it is not a string or function",()=>{
    expect(v.replaceAll("abcd", "b", 1)).toBe("a1cd")
    expect(v.replaceAll("abcd", "c", [1,2,3])).toBe("ab1,2,3d")
  })
```

The EndsWith function provides an example of a missing test case found by stryker (the library probably predates when this function was added to all browsers?). This function checks to see if a string ends with some other string and optionally has an parameter to indicate you want to shorten the string (before checking if it ends with the endstring). 

Stryker finds the following surviving mutant on line 39: 


<iframe src="{{ site.baseurl }}/media/stryker/ends_with.html" width="100%" height="600px">
</iframe>

This extra validation (index -1) is needed when the position -1 and the index also -1. One could be tempted to remove the first part (at least i was), but this can actually happen if the end is longer then the string that is tested (a nice case for an early return). So the following test case can be added to kill the mutant:

```javascript
 it('deals with substring longer then string', ()=>{
    expect(v.endsWith('asdf',' asdf')).toBe(false) //note the extra space before the 2nd asfd
    expect(v.endsWith('asdf', "asdf", 3)).toBe(false)
  })
```

These examples to me are pretty convincing to me. One can wonder if an experienced tester would not have found these. Personally I would be happy if I found even 1 without strykers help. The downside again, is that is takes some time to analyse them, and these examples all point to missing testcases, but not bugs. 


#### Overcomplicated / untested functions
When the stryker gets impressive is with the strip_tag function. The strip_tag function takes in html and returns the text without the html tags (optionally there is an array argument to to allow_tags that will not get removed). This function uses a the hasSubstringAtIndex and the parse_tag_name helper function (both not exposed to the external user). Stryker finds issues with all of them even though all lines are covered by tests.


<iframe src="{{ site.baseurl }}/media/stryker/tags.html" width="100%" height="600px">
</iframe>


In the strip_tag function strykerJS finds a load of surviving mutants, most surprisingly even some indicating that a couple of the "select cases" can be removed without triggering a failure in the test cases.  This function is pretty complex, so it is hard to figure out if these are not covered by test cases, or maybe equivalant because the default case would do the same. (Since there is line coverage there must be at least some test cases that reach the case).  Either way, a case that can be removed definitely points to extra testcases being needed, or a function that can be simplified.  

(It is a bit unsatisfactory but since this is just supposed to be a tryout of strykerjs, and not a thorough examination of the strptag, but to me it seems the code can be removed: the following test fails with the e cases, but passes without it. 

``` javascript
    expect(
      v.stripTags('<!doctype html><html><b>Hello world!</b></html>', ['html'])
    ).toBe('<html>Hello world!</html>'); // actual: "< html><html>Hello world!</html>"

```
)


#### Many more issues
This is a selection of the mutants that survived. Overall there are more to be analysed, but I hope this section gives a good overview of the good and the somewhat bad with mutation testing. 


####  Bugs discovered Indirectly
Did find that the test setup was wrong because of not specifying subdirs. testMatch: ['<rootDir>/test/**/*.js'], was testMatch: ['<rootDir>/test/*/*.js'], causing it to not run som nested helper test files. 


### Some issues with strykerJS
#### Static Mutants
> A static mutant is a mutant that is executed once on startup (during the loading of the file) instead of when the tests are running

So say the [stryker docs](https://stryker-mutator.io/docs/mutation-testing-elements/static-mutants/)). Circumventing this would mean a longer runtime, because the files would need to be reloaded more often. 
You should be able to disable static mutants and then they should not show up, but that currently is not working (see bug). So be on the lookout for static mutants icon before loosing time debugging those. 


#### Timeout not always to be trusted
Sometimes running stryker leads to mutants being timed out (which is scored as a kill), but after running it a second time the mutants did not time out (and where either killed or survived). There are options to specify for how long to wait before marking a mutant timed out and to set the number of workers (as not to overload your computer). But even running only the mutation test on my laptop (and nothing else) would sometimes generate the timeouts and other times not. So the defaults were not great for me and it might be good to double check timeouts. 


### Topics for further exploration
Mutation testing is a well researched topic with lots of questions and ideas to explore further, for instance the following: 

#### Which mutants have most value ?
There are lots of mutants types that find the same issues and what type of mutants are the best at finding issues other mutants do not. 
With stryker you can choose to disable certain mutants. So looking into this certainly would make sense since there are just so many surviving mutants. 

#### Higher order mutants
Stryker limits itself to only add one code change at a time. So every mutant is a first order mutants. There is also research with higher order mutants, ones that introduce two or more errors (https://www.sciencedirect.com/science/article/abs/pii/S0950584909000688) 

#### As a measure
Stryker reports the mutation score for each file, directory and for the project.  And there is literature about which coverage metric is more usefull:  Line coverage is trumped by path coverage and maybe path coverage is trumped by mutation score. But for a good read about how (not) to use metrics and why mutation score might be more usefull, read [this](https://journal.optivem.com/p/code-coverage-vs-mutation-testing). 

#### Theoretical foundations
There is a theory about why (first order: with one simple mutation) mutation testing  works; the competent programmer and the coupling effect hypotheses.  It is explained nicely in [this](https://www.cs.cornell.edu/~dgeisler/mutation/testing/2021/11/07/mutation-testing2.html) blog. 



## Thoughts, Questions, Tips
Easy setup (at least for this example) and easy enough to just run for a whole library, or a file you added or changed. Also the dashboard is helpfull and clear (though not sure about the test tab).  

Difficult to triage and long time to analyse:
  - hordes and hordes of mutants,
  - Need to really graps the problem the code solves to understand some mutants. 
  - The stubborn /equivalent mutants are not easily to spot, making it uneasy to skip over some. You will be tempted to ignore surviving mutants thinking they are not relevant and only after a second reading understand what the problem is. 

It will find a lot of smaller issues that will take a lot of time to remove by adding a test (or refactoring the code) or cannot even be practically tested. 

Though some of the issues are relatively small, they do regularly show unnecessary / inconsistent / maybe easily misunderstood code that could be cleaned, by rewriting or simply removing lines, or adding coding conventions. 

Overall, despite having to go through sometimes boring and nitty gritty analysis, bottom line is that it found a good number of missing testcases and code improvements in a well tested library with almost 100% code coverage. 

I feel it would probably less overwhelming to run mutation testing on small part of the code base, right after working on it: Then you probably spot relevant mutants more easily. 

A goal of mutation testing can also be to calculate a mutation score (the (killed + timed out mutants)  / total mutants). I would not trust that to much: The value of a mutant is too different (too many stubborn or similar mutants) to be weighed in any meaningfull way. But it would show progress over time. 

There are bugs that mutation testing will not find: Not all code constructs have a mutator rule. For instance a lot of bitwise operators / constant numbers do not have  rules (though this is dependent on the language with stryker, and between frameworks), not all javascript apis (String etc) have rules, and functions from external libraries certainly won't have them either. Also most mutation testing frameworks only do first order mutants (so only one at a time), which will be limiting in finding bugs. And others
 
Remember, before starting your analysis, not every failure will mean that there is something wrong with the source code. There is just no test case covering the detection.  And beware of static mutants. 

