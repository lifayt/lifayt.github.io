---
layout: post
title: 	"Suspiciously Selective CSS"
date: 	2017-03-13 3:03:00 -0400
categories: blog
---
### Mystery

I was working on a little front end widget that pulled data from a Sharepoint list and displayed it as a series of warnings. I had previously deployed this widget, but was redeveloping it to be generically applicable for several different pages. To my surprise, when I deployed it for the first time, the CSS seemed to selectively apply, with some classes correctly styled, but not others. 

My first thought was that the CSS file wasn't loaded properly, so I poked around in the chrome dev tools. To my surprise, the first class I chose (the outer div of my widget) seemed to have it's CSS loaded properly. 

![outer_div](/assets/images/2017-03-13/outer_div.png)

Note 'fac_staff.css' - that was my custom css file. Unfortunately, none of the CSS higher up in the file seemed to be applied to the page. Several JQuery buttons were supposed to be styled, but judging from the inspector, the CSS file wasn't be applied at all! It wasn't even as though they were applied but overwritten by a conflicting rule - the rules were totally missing from the element in the first place. 

Intended:

![intended_button](/assets/images/2017-03-13/intended_button.png)

Actual:

![actual_button](/assets/images/2017-03-13/actual_button.png)

This was unusual behaviour mostly because CSS problems tend to boil down to the following:

* Is it showing up?
	* Yes 
* If no, has it loaded?
* If it loaded, is another rule overwriting it?

Usually, the CSS isn't loading in the first place, or a pesky `!important` is overwriting the new rules. 

### Investigation

After staring at the screen for 20 minutes (and then getting vietnamese food to clear my head), I did what I should have done in the first place, and pasted my CSS into [CSS Lint](http://csslint.net/). Sure enough, it threw several parsing errors. Unfortunately, the parsing errors were just as inscrutable as the suspiciously selective CSS. For example:

![wonky_lint](/assets/images/2017-03-13/wonky_lint.png)

Nothing seems to be wrong with the CSS statement on the surface, yet the linter complains that a RBRACE is expected at the end of the second line. On a hunch, I checked the css file the browser had actually loaded. Jackpot! The css chrome tried to apply to my code was totally garbled.

![garbled_css](/assets/images/2017-03-13/garbled_css.png)

Now I was getting close. Manually rewriting the css seemed to do the trick. The following two rules look the same, but the top one doesn't work while the bottom one does. The bottom one was handwritten on the fly, and the top one was copied from an old file (more on this later).

![functional_nonfunctional](/assets/images/2017-03-13/functional_nonfuctional.png)

Fixing all the other linted errors by rewriting the statements resolved my issue - the css rendered properly on the page, and all my other problems went away. Nonetheless, I was curious why my original version of the file was corrupted when the browser tried to parse it. I used this handy [encoding converter](https://r12a.github.io/apps/conversion/) to compare the functional & nonfunctional pieces of code in UTF-16. 

Non-functional:
```
002E 0069 006E 0070 0075 0074 005F 005F 0074 0069 006D 0065 00A0 007B 000A 0020 0020 0066 006F 006E 0074 002D 0073 0069 007A 0065 003A 00A0 0038 002E 0035 0070 0074 00A0 0021 0069 006D 0070 006F 0072 0074 0061 006E 0074 003B 000A 0020 0020 0062 0061 0063 006B 0067 0072 006F 0075 006E 0064 002D 0063 006F 006C 006F 0072 003A 00A0 0023 0064 0063 0064 0061 0064 0061 00A0 0021 0069 006D 0070 006F 0072 0074 0061 006E 0074 003B 000A 007D
```

Functional: 
```
002E 0069 006E 0070 0075 0074 005F 005F 0074 0069 006D 0065 0020 007B 000A 0020 0020 0066 006F 006E 0074 002D 0073 0069 007A 0065 003A 0020 0038 002E 0035 0070 0074 0020 0021 0069 006D 0070 006F 0072 0074 0061 006E 0074 003B 000A 0020 0020 0062 0061 0063 006B 0067 0072 006F 0075 006E 0064 002D 0063 006F 006C 006F 0072 003A 0020 0023 0064 0063 0064 0061 0064 0061 0020 0021 0069 006D 0070 006F 0072 0074 0061 006E 0074 003B 000A 007D
```

Notice the difference? The nonfunctional pieces of code seem to encode several of the spaces as `00A0` instead of `0020`. `00A0` is the Unicode character for a non-breaking space! Everywhere the 00A0 was present, the resulting CSS came out corrupted. Evidently, non-breaking spaces cause all sorts of trouble when the browser tries to parse the CSS file. 

If you look at the CSS [spec](http://www.w3.org/TR/CSS21/syndata.html), the distinction is pretty clear. 

> Only the characters "space" (U+0020), "tab" (U+0009), "line feed" (U+000A), "carriage return" (U+000D), and "form feed" (U+000C) can occur in white space. Other space-like characters, such as "em-space" (U+2003) and "ideographic space" (U+3000), are never part of white space.

`U+00A0` is not explicitly mentioned here as acceptable whitespace, which led me to believe it does not function as whitespace. Ironically, a section further down clears this up by specifically mentioning `U+00A0`, and it shines a light on why the non-breaking space seems to corrupt attempts at properly formatting whitespace in CSS blocks. 

> In CSS, identifiers (including element names, classes, and IDs in selectors) can contain only the characters [a-zA-Z0-9] and ISO 10646 characters U+00A0 and higher, plus the hyphen (-) and the underscore (_); they cannot start with a digit, two hyphens, or a hyphen followed by a digit. Identifiers can also contain escaped characters and any ISO 10646 character as a numeric code (see next item). For instance, the identifier "B&W?" may be written as "B\&W\?" or "B\26 W\3F".

So `U+00A0` is being treated as part of the selector when it comes after the selector, and not as the whitespace required to properly format the snippets of CSS above. After some digging I found a similarly informative [StackOverflow](http://stackoverflow.com/questions/28339327/why-non-breaking-space-is-not-allowed-in-css-declarations) answer on the same subject.

### Solution 

So where did all of these non-breaking spaces come from in the first place? Turns out my own bad habits are to blame. Remember how the non-functional piece of CSS was copied in from an old file? 

The day I was working on this particular file I wanted to continue working on the train, so I copied the file contents down from my windows VM onto my Macbook. Unfortunately, that seems to replace all instances of `U+0020` with `U+00A0`!

It's a relatively small annoyance in the grand scheme of things since most applications seem to fail gracefully or convert efficiently when confronted with `U+00A0`. Specifically formatted CSS is (understandably) one of the exceptions. Ah well. Likely, this is a problem with the RDP program I'm using (Jump Desktop), but I don't have quick access to another to compare. 