---
title: Search highlight implementation
author: Zheng Yuan
date: 09/26/2019
tags: ['css', 'js', 'html', 'search']
---

Search highlight implementation
============

Everytime I use the search functionality in a web site, I think the highlight/mark is the most important part for a better UX since user not only want to get the matched result but also the part that make the element matched. But how?

Is it easy to implement?
-----

Not so hard. After I dived into the google and tried several times, I found the most difficult part for highlight is that you need to make the text matched wrapped with a 'decorator' which can be a simple `strong` tag or a custom tag that you want to style with, and you must need to make sure the wrapper would break current UI display and should be undoable. 

As for the logic to determine the highlight part, just like all the basic search case, you can use a regular expression to cover your back, which including case sensitive/insensitive and escape the unnecessary characters. 

Also, since the searching target can be all kinds of html markup, I think the best way to do the regular expression testing is do it on the innerHTML of a target so that we won't missing any text in the web site.

After wrapping all the necessary part we need, we have the folloing core code to implement the search highlight:

~~~javascript
var domNodeCheckerInOldWay = function(domNode) {

    // after building the regex in the way you liked.....

    regex.lastIndex = 0;
    var str = domNode.dataset.origHtml;
    if (!str) {
        str = domNode.innerHTML;
        domNode.dataset.origHtml = str;
    }

    // use matched flag to determine whether or not to show the domNode
    if(regex.exec(str)) {
        matched = true;
        str = str.replace(regex, function(target) {
            return '<mark>' + target + '</mark>';
        });
    } else {
        matched = false;
    }
    domNode.innerHTML = str;
};
~~~

And the key points in this snippet are:
* We use the dataset to store the original innerHTML of a domNode which can be used for the highlight undo when needed.
* use the `String.prototype.replace` with the regex we have to wrap the matched part with a custom tag that can be styled
* Replace the current innerHTML with the modified innerHTML that has the highlight tag.

And you might have a question, can the modified innerHTML work since the replace won't make sure the syntax is still correct. 

The answer is **yes** but you need to add some extra content to our current regex just like this 
~~~javascript
var regex = new RegExp(
        THE_QUERY_YOU_WANT + 
            '(?![^<>]*>)',
        'ig
    );
~~~

By using `(?![^<>]*>)` which makes a negative lookahead and excludes all the html markup syntax that is composed of `<>`. After doing this, now we can mark the matched part in the searched result with any effect you like.

Is this the best way to achieve the goal?
-----------

Unfortunately, **no**.

Although we implement the feature but we never consider the cost. Since every time we need to replace the innerHTML of a target DOM with the modified one, our browser need to re-parse it and then finish the layout.

Beside of the performance issue, the search and replace on the innerHTML are dangerous and unstable. if there is a DOM attribute or any content that match the regex but not located inside the text area, you might make the whole page broken. 

So what's the better solution?

There are several 3rd-party libraries that support the search highlight, eg: **typeahead.js** by twitter, **mark.js** by julmot. 

All these libraries I mentioned above don't use the innerHTML directly to implement the highlight, and they choose to go over all the textNode and make the changes on them to finish the job:

~~~javascript
// This is my script follow their core source code implementation
 var highlightTextNode = function(textNode, match) {
    var wrapperNode = document.createElement('mark');
    var patternNode = textNode.splitText(match.index);
    var remainingTextNode = patternNode.splitText(match[0].length);
    // we use this attribute to mark the wrapper 
    // which can be used for unhighlight later
    wrapperNode.setAttribute('wrapper', true);
    wrapperNode.appendChild(patternNode.cloneNode(true));
    textNode.parentNode.replaceChild(wrapperNode, patternNode);
    return remainingTextNode;
},
var domNodeChecker = function(domNode) {
    // we only focus on the textNode 
    if(domNode.nodeType === 3) {
        regex.lastIndex = 0;
        var match = regex.exec(domNode.data);
        if(match) {
            if(query) {
                var remainingTextNode = self.highlightTextNode(domNode, match);
                if(remainingTextNode && remainingTextNode.data) {
                    domNodeChecker(remainingTextNode);
                }
            }
            matched = true;
        }
    } else {
        Array.from(domNode.childNodes).map(function(childNode) {
            domNodeChecker(childNode);
        });
    }
};
~~~

The main change here is that we never touch the innerHTML directly and only modify the textNode we iterate over. As a result, we will never worry about our modification can broke the page. 

Besides, there is a huge performance boost since we don't need to parse the HTML and only need to focus on the layout cost. And based on my benchmark test (search through 3000 table row with range HTML markup) the time taken for each search & highlight task reduced from 200ms to 100ms (thanks to Chrome render engine, the textNode re-layout becomes bacthed)

And this is all the story about my working on the search highlight implementation. Feel free to email me if you have any questions or sharing your opinion.
