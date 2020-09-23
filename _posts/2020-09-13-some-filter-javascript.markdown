---
title:  "some() , filer() , includes() in javascript"
date:   2020-09-13 22:04:23
categories: [javascript]
tags: [javascript]
---

I have begun learning javascript. This skill will prove useful in merging with my security experience. Like almost everyone, you learn about variables, working with objects and arrays. This will focus on the latter. The task was multifaceted, but the first task was to create a function that counts the correct answers in the submissions array. This was straight forward enough.

{% highlight js linenos %}

function countCorrectQuestions(submissions) {
  let correctSubmissions = 0;
  for ( let i = 0; i < submissions.length; i++) {
    if (submissions[i].isCorrect === true) {
      correctSubmissions++
    }
  }
  return correctSubmissions
}

{% endhighlight %}

A simple *for* loop starting at zero, to loop through each entry in the array and check whether *isCorrect* element equals true using strict equality, if so increment correctSubmissions by 1.

Moving on to the next task at hand, I had to create a function that takes an array and a boolean value as arguments. This function would check the submissions array element *isEassyQuestion* against the argument passed to the function. I had to return an array of all the submissions of that appropiate type. Initially, since we had covered mainly for loops when dealing with arrays, I tried that.

{% highlight js linenos %}

let trueEssay = [];
let falseEssay = [];


  for (let i = 0; i < submissions.length; i++) {
    if ( essayquestion === true) {
      if ( submissions[i].isEssayQuestion === true) {
        trueEssay.push(submissions[i]);
      } else {
        falseEssay.push(submissions[i]);
      }
{% endhighlight %}

This code works, half the time. I played with the code some more but knew there had to be a better way of doing this. After a bit of research, I found Javascript's array [filter() method](https://www.geeksforgeeks.org/javascript-array-filter-method/?ref=rp). Per the link:

> The arr.filter() method is used to create a new array from a given array consisting of only those elements from the given array which satisfy a condition set by the argument method.

This fits my issue perfectly as I had to return a new array with only the appropiate type (if *isEssayQuestion* is true or false). After some reading and noticing that using the arrow *=>* notation was much easier (hey, am brand new to this) I got this:

{% highlight javascript linenos %}

function filterQuestionsByType(submissions, essayquestion) {
 return submissions.filter(({isEssayQuestion}) => isEssayQuestion === essayquestion)

 {% endhighlight %}

 The array filter() method will loop through each entry in the array and check *isEssayQuestion* strictly equals the argument essayquestion passed to the function. If true or false it will return a new array containing the matching elements.

 The final piece to the task was to check if the *[i].responses* element contained a string that would be passed as an argument to the function I was to create. The instructions included a hint to look into using [string includes() method](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/includes).Apparently I did not thoroughly read the instructions as I did not end up using str.includes but used an [array includes()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/includes)

 > The includes() method determines whether an array includes a certain value among its entries, returning true or false as appropriate.


{% highlight js linenos %}

function checkForPlagiarism(submissions, string) {
  for ( let i = 0; i < submissions.length; i++) {
    let checkStr = submissions.some(i => i.response.includes(string));
    return checkStr;
  }
  return false;
}

{% endhighlight %}

To break it down, I use some() and arrow notation to specifically check the *responses* entry in the array index (i) includes *string* which is a parameter passed to the function as seen on Line 1. Outside of the loop I return false if the condition was not met. I really enjoyed this module in the course as it made me look deeper into javascript since the methods I had laerned thus far was causing my code to only work sometimes. I need to spend more time and identify why it did that. I think these examples were used to push us into researching better solutions and making us better programmers.
