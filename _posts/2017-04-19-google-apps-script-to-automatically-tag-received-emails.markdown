---
layout: post
title: Google Apps Script to automatically tag emails with date-based labels
date: '2017-04-19 20:06:42'
---

I use to automatically tag my e-mails with time based labels, on Gmail.
Can I simply use filters?
Maybe yes, but I didn't get to make them work when specifying only dates as criterias.
Maybe I'm idiot, but Gmail simply doesn't allow me to specify such filters.
So I tricked him with a small Google Apps Script, that tags emails with a current year and a current month nested labels.
Please note that it just works for 2017 and month names are in Italian.
I scheduled it to run once a minute.

```
function labelAllMessages() {
  
  var label2017 = GmailApp.getUserLabelByName("2017");
  
  var months = ["gennaio", "febbraio", "marzo", "aprile", "maggio", "giugno", "luglio", "agosto", "settembre", "ottobre", "novembre", "dicembre"];
  
  var getNumberOfDaysByMonth = function(i) {
    return (i==1) ? 28 : (  !(i>=7 ^ !(i&1)) ? 30:31  );
  };
  
  
  var getDateRangeForMonth = function(ind) {
    var month = months[ind];
    var range = {
      start: new Date(2017, ind, 1).valueOf(),
      end: new Date(2017, ind, getNumberOfDaysByMonth(ind)).valueOf(),
      label: GmailApp.getUserLabelByName("2017/" + month)
    };
    return range;
  };
  
  
  var threads = GmailApp.search('-label:2017 -label:"before 2017"')
  if (threads.length <= 0) {
    Logger.log("No untagged message")
    return
  }
  for (var i=0; i<threads.length; i++) {
    var message_date = threads[i].getLastMessageDate().valueOf();
    for (var month in months) {
      var dateRange = getDateRangeForMonth(month)
      if (message_date >= dateRange.start && message_date < dateRange.end) {
        threads[i].addLabel(label2017);
        threads[i].addLabel(dateRange.label);
        Logger.log("Applied label");
        return;
      }
    }
  }
}
```
Are there more efficient ways to do the job? Of course.
Could I have written a better code? Of course.
But it just works after five minutes of scripting. Feel free to use it if you want.
