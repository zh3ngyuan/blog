---
title: Bootstrap modal zIndex issue
author: Zheng Yuan
date: 09/25/2019
tags: ['html', 'vanilla js', 'bootstrap', 'css', 'web']
---

Bootstrap modal zIndex issue
============

The first time I need to use multiple bootstrap modal in one page, I just call the bootstrap modal show multiple times like this:

~~~javascript
$('.modal#modalA').modal('show');
$('.modal#modalB').modal('show');
$('.modal#modalC').modal('show');
~~~

Then I found these modals actually stay in the same z-index because in **_modal.css** we have:
~~~css
.modal {
    position: fixed;
    top: 0;
    left: 0;
    z-index: 1050; /* <--- here */
    display: none;
    width: 100%;
    height: 100%;
    overflow: hidden;
    outline: 0;
}
~~~

which means all the modal has the same **z-index**.

What's the result?
-------

Well, in my case. The biggest problem is that user cannot make any operation on the previous opened modals but you can still see them as the latest modal and there is no any indicator to help you distinguish them. 

Another issue is that if you use **backdrop** (by default, bootstrap modal enables this option), when you click on another modal but it is actually positioned under the latest modal, the latest modal will dismiss since you click on the backdrop of the latest modal.

Solution?
------------

The easiest way to workaround this issue is that you can modify the **z-index** of each modal to make sure they are displayed in Z-axis order like this:
~~~javascript
$('.modal#modalA')
    .css('z-index', 1040) // 1040 is the default index value defined in _modal.css
    .modal('show');
$('.modal#modalB')
    .css('z-index', 1050) // usually, we choose 10 as the increment value to make sure the modal-gap is large enough
    .modal('show');
$('.modal#modalC')
    .css('z-index', 1060)
    .modal('show');
~~~

Is there a better way to handle this? Since it's unresonable to modify the z-index of the current modal with a proper value. So a better way to fix this problem is to add a listener on modal show event then use the number of visible modal to calculate the proper z-index for updating:
~~~javascript
 $(document).on('show.bs.modal', '.modal', function () {
    var zIndex = 1040 + (10 * $('.modal:visible').length);
    $(this).css('z-index', zIndex);
});
~~~

So next time when there is a new modal show up, you don't need to manually change this modal z-index to make it display properly.

After adding this modification, now you can make sure the modal is display in Z-axis order. But...

What about the backdrop? Since the modal backdrop is not a child of this modal DOM which means its z-index won't be changed if you simply change the modal z-index. 

Maybe we can add the modification for backdrop z-index in this listener too. Nice try, but if you simple add this following line like this:
~~~javascript
 $(document).on('show.bs.modal', '.modal', function () {
   
     //....//

    var opacity = 0.5 + $('.modal:visible').length * 0.1; 

    $('.modal-backdrop').not('.modal-stack').css({
        'z-index': zIndex - 1,
        'opacity': Math.min(1, opacity)
    }).addClass('modal-stack');
});
~~~
Two extra changes here:
* we reduce the opacity of the backdrop as the number of backdrop increases to make sure the web page content beneath the modal won't be overlapped. 
* we add **modal-stack** class as a flag for the modified backdrop

But after we make this update, the backdrop still not updated correctly. Why is that?

Nothing will happen since the backdrop won't be mounted right after the modal show gets called based on the source code **bootstrap.js**:
~~~javascript
_proto.show = function show(relatedTarget) {

    //....//

    var showEvent = EventHandler.trigger(this._element, Event$6.SHOW, { // <------ show.bs.modal triggered
        relatedTarget: relatedTarget
    });

    //....//
     
    this._showBackdrop(function () { // <----- Backdrop show and append to document
        return _this._showElement(relatedTarget);
    });
};
~~~
we can find that **show.bs.modal** event triggered before the backdrop show and append to our current document. So we should put that logic into a async task queue to make sure all the call stack finished before we find the modal backdrop: 
~~~javascript
 $(document).on('show.bs.modal', '.modal', function () {
   
     //....//

    setTimeout(function () {
        $('.modal-backdrop').not('.modal-stack').css({
            'z-index': zIndex - 1,
            'opacity': Math.min(1, opacity)
        }).addClass('modal-stack');
    }, 0);
});
~~~
after this update, now the backdrop and the modal can display with corrected z-index. 

Is that all?
------------------------
Maybe you think that's all we can do for the bootstrap modal. That's true. But there are still one corner case we don't cover although it sounds like a little bit wired: 

Imaging you want to display a modal right after another modal dismissed, what will you do?

For the most straightforward method to handle this case we might use:
~~~javascript
// hide modalA
$('.modal#modalA').modal('hide');

// and show modalB immediately
$('.modal#modalB').modal('show');
~~~
Looks like working, but it will introduce another problem caused by the listener we created above.

When you show **modalB** the listener will find all the previous visible modal and then calculate the 'correct' zIndex value for **modalB**. Since bootstrap modal hide won't make the modal invisible immediately, which means the zIndex we just calculated including the modal that is going to be hidden. As a result, after **modalB** hidden and another modal (let's say **modalC**) going to show, it will go over the same zIndex calculation logic, but got the same zIndex result with **modalB**.

So what's the solution for this corner case?

A basic idea to handle this situation is that we need to add listeners to cover not only **'show.bs.modal'**, but also **'hide.bs.modal'** and **'hidden.bs.modal'**:
~~~javascript
var closingModalSet = new Set();
var suspendingModalSet = new Set();

var modalUpdateHelper = function(modal) {
    var zIndex = 1040 + (10 * $('.modal:visible:not([suspending])').length);
    modal.css('z-index', zIndex);
};

var modalBackDropUpdateHelper = function(backdrop) {
    var zIndex = 1040 + (10 * $('.modal:visible:not([suspending])').length);
    var opacity = 0.5 + $('.modal:visible:not([suspending])').length * 0.1;
    backdrop.css({
        'z-index': zIndex - 1,
        'opacity': Math.min(1, opacity)
    }).addClass('modal-stack');
};

$(document).on('show.bs.modal', '.modal', function () {
    var modal = $(this);
    if(closingModalSet.size) {
        modal.attr('suspending', true);
        suspendingModalSet.add(modal.attr('id'));
    }
    modalUpdateHelper($(this));
    setTimeout(function () {
        var backDrop = $('.modal-backdrop').not('.modal-stack');
        if(modal.attr('suspending')) {
            modal.data('suspendingBackdrop', backDrop);
        }
        modalBackDropUpdateHelper(backDrop);
    }, 0);
});

$(document).on('hide.bs.modal', '.modal', function () {
    closingModalSet.add($(this).attr('id'));
});

$(document).on('hidden.bs.modal', '.modal', function () {
    closingModalSet.delete($(this).attr('id'));
    var isModalStillOpen = !!$('.modal:visible').length;
    setTimeout(function () {
        if (isModalStillOpen) {
            $('body').addClass('modal-open');
        }
    }, 0);

    Array.from(suspendingModalSet).map(function (modal_id) {
        var modal = $("#" + modal_id);
        var backDrop = modal.data('suspendingBackdrop');
        modalUpdateHelper(modal);
        modalBackDropUpdateHelper(backDrop);
        setTimeout(function () {
            modal.removeAttr('suspending');
            modal.removeData('suspendingBackdrop');
            suspendingModalSet.delete(modal_id);
        }, 0);
    });
});
~~~

This is just a quick scratch and there are a lot of places can be optimized, but the core idea is that:
> we call the going to be closed state for a modal **closing**
>
> Each time a **closing** modal appears, we record it by using a set.
>
> When a new modal show, if there are **closing** modal existing, we will mark it as **suspending** since its zIndex calculation is temporarily incorrect.
> 
> After this modal hidden, we will recalculate the zIndex of all the **suspending** modals. 

Practice
-----------
You might notice in our last code block, there are some lines looking strange:

~~~javascript
var isModalStillOpen = !!$('.modal:visible').length;
setTimeout(function () {
    if (isModalStillOpen) {
        $('body').addClass('modal-open');
    }
}, 0);
~~~

Why we need to change the **body** class here? I leave the question here for you to figure it out. 

Feel free to email me if you want to know the answer.
