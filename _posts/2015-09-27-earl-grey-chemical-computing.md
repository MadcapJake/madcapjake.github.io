---
title:  "Chemical Programming in Earl-Grey"
date:   2015-09-27 11:50:00
description: An exploration of Carin Meier's excellent talk on an unconventional paradigm via Earl Grey.
---

While driving home from Madison, I decided to listen to [The Changelog](https://changelog.com/) and my favorite topic is always languages so I opted for their [episode](https://changelog.com/171/) with [Carin Meier](http://gigasquidsoftware.com/) on Clojure and her new book, [Living Clojure](http://www.amazon.com/exec/obidos/ASIN/1491909048/5by5-20).  In the podcast, Carin discusses her love of reading PLT and especially unconventional paradigms.  Though my ability to read PLT is limited (I have little-to-no background in Calculus), I do really love finding innovative ways to program.

It just so happens that Carin's mentioned upcoming talk on the chemical computing paradigm happened just this past week and was uploaded the day I got back home!  So I decided to delve in and learn more about this interesting reaction-oriented paradigm.

<iframe width="520" height="293" src="https://www.youtube.com/embed/cHoYNStQOEc" frameborder="0" allowfullscreen></iframe>

Of course, whenever there is a well laid-out description of some programming/algorithm, I can't help but want to implement it and my current *lingua scelta* is [Earl-Grey](https://breuleux.github.io/earl-grey/).  So here are Carin Meier's exercises in chemically computing primes written in Earl-Grey:

{% highlight earl-grey %}
require: underscore as U

predicate! is-prime = n ->
   let possible-factors = range(2, n - 1)
   let remainders = possible-factors each f -> n mod f
   not true in [remainders each r -> r == 0]

print is-prime? 5 ;;= true
print is-prime? 6 ;;= false

primes = n -> [2..n] each val when is-prime? val -> val

print primes(100)[0..10] ;;= [2, 3, 5, 7, 11, 13, 17, 19, 23]

prime-reaction = ({a, b} or a and b is undefined) ->
   if undefined? b: return
   elif a > b and (a mod b) == 0: {(a / b), b}
   else: {a, b}

print prime-reaction({6, 2}) ;;= [ 3, 2 ]
print prime-reaction with {5, 2} ;;= [ 5, 2 ]

pairs = arr ->
   var {i, result} = {0, {}}
   for val of arr:
      if i mod 2 == 0: result ++= {{val}}
      else: result[i - result.length] ++= val
      i += 1
   result

molecules = range(2, 50)

mix-and-react = mols ->
   mixed   = pairs(U.shuffle(consume(mols)))
   reacted = mixed each m -> prime-reaction(m)
   U.without(U.flatten(reacted), null)

r-cycle = n -> f(n, molecules) where
   f = (i, mols) ->
      if i == 0: mols
      else: f(i--, mix-and-react(mols))

print U.sort-by(set(r-cycle(1000)).toJSON()) where
   set = a -> new Set(a)
;;= [2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37, 41, 43, 47]
{% endhighlight %}

Check out the [article](http://gigasquidsoftware.com/chemical-computing/index.html) for more info and the demos shown in the video.

I was surprised how easily I was able to translate some of the Clojure code into Earl Grey.  There were a few details of the JSverse that needed washing over but I would say this looks readable even for someone not familiar with Earl Grey.  Let me know if there is anything that you don't follow (or any JS detail that I could simplify).  Thanks and hope you enjoyed the talk and my translation!
