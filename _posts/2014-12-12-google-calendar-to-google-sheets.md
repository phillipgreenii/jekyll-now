---
layout: post
title: Google Calendar to Google Sheets Bidirectional Sync
date: '2014-12-12T07:47:00.001-05:00'
author: Phillip Green
tags:
- sheets
- javascript
- sync
- google calendar
- calendar
- google sheets
- google
modified_time: '2014-12-12T07:47:58.649-05:00'
blogger_id: tag:blogger.com,1999:blog-3096944600800047027.post-8062514110157538820
blogger_orig_url: http://coder-in-training.blogspot.com/2014/12/google-calendar-to-google-sheets.html
calendarUtilsLibProjectKey: MT9e6TqW6VlvXr43nv8kZqWRtpc-QmtJv
---

Personally, I love both spreadsheets and google calendar and some times it makes sense to keep them in sync.
The use case that started me on this was a calendar to keep track of important work dates.
Having things show up on my google calendar is great for keeping track of everything, but I have not found any simple way of doing bulk upload
My solution is to dump into a spreadsheet all of the events (holidays, major releases, ...).
There is one place to look.
I then added a menu to either push events from the spreadsheet to my calendar or vice versa.
The process is manual, but I haven't had any problems.
Generally, I view and update events through the calendar.
When necessary, I drop into the spreadsheet,
pull from the calendar, make my changes, then push back to the calendar.
Below is the code I use to make it happen.

##CalendarUtilsLib
###Creation
In order to re-use my sync code across multiple spreadsheets, I created a stand alone project that was reused as a library.
Go to https://drive.google.com/ and click
`New->More->Google Apps Script`

```javascript
//TODO: support day light savings time

function createEventsLookup(events){
  var lookup = {};
  events.forEach(function(event){
    lookup[event.getId()] = event;
  });

  return lookup;
}

function createEventInfosLookup(events){
  var lookup = {};
  events.forEach(function(event){
    lookup[event.getId()] = event;
  });

  return lookup;
}

function loadAllEventsFromCalendar(calendar) {
  return calendar.getEvents(new Date(2000,0,1), new Date(3000,0,1));
}
/*
* This throws in a pause (sleep) after so many steps.  
* This exists because google will fail if you attempt too many updates at once.
*/
function throttleSpeed(limit,count) {
  if(count % limit === (limit - 1)) {
    Utilities.sleep(1000);
  }
}

function publish2calendar(calendar, eventInfos) {
  var existingEvents = loadAllEventsFromCalendar(calendar)
  var existingEventsById = createEventsLookup(existingEvents);

  var updatedEvents = {},
  updateCounts = 0,
  createdCounts = 0,
  errorsCount = 0;
  eventInfos.forEach(function(info,index){
    throttleSpeed(15,index);
    if(info && info.isValid()){
      var existingEvent = existingEventsById[info.getId()];
      if(existingEvent) {
        updatedEvents[existingEvent.getId()] = true;
        if(info.updateEvent(existingEvent)) {
          Logger.log('Updated Event: %s (%s %s-%s)', existingEvent.getId(),existingEvent.getTitle(),existingEvent.getStartTime(),existingEvent.getEndTime());
          updateCounts = updateCounts + 1;
        }
        } else {
          Logger.log('Creating Event: %s %s-%s',info.getTitle(), info.getStartTime(), info.getEndTime());
          var event = info.createEventOnCalendar(calendar);
          info.setId(event.getId());
          // Make sure the cell is updated right away in case the script is interrupted
          SpreadsheetApp.flush();
          createdCounts = createdCounts + 1;
        }
        } else {
          Logger.log('Invalid row[%s]: %s', index, info && info.toString());
        }
        })

        var deletedCounts = 0;
        existingEvents.forEach(function(event,index){
          if(!updatedEvents[event.getId()]) {
            throttleSpeed(15,index);
            Logger.log('Deleting Event: %s (%s %s-%s)', event.getId(),event.getTitle(),event.getStartTime(),event.getEndTime());
            event.deleteEvent();
            deletedCounts = deletedCounts + 1;
          }
          });

          var msgLines = [],
          timeout = 5;
          if(updateCounts > 0) {
            msgLines.push("Updates:\t" + updateCounts);
            timeout = -1
          }
          if(createdCounts > 0) {
            msgLines.push("New Events:\t "+ createdCounts);
            timeout = -1
          }
          if(deletedCounts > 0) {
            msgLines.push("Deleted Events:\t "+ deletedCounts);
            timeout = -1
          }
          if(errorsCount > 0) {
            msgLines.push("Errors:\t" + errorsCount);
            timeout = -1
          }

          if(msgLines.length === 0) {
            msgLines.push("No changes");
          }
          msgLines.forEach(function(line) {Logger.log(line)});
          SpreadsheetApp.getActiveSpreadsheet().toast(msgLines.join("\n"),"Publish Complete",timeout);
        }

        function refreshFromCalendar(calendar, eventInfos, addEvent) {
          var existingEvents = loadAllEventsFromCalendar(calendar);
          var infosById = createEventInfosLookup(eventInfos);

          var updateCount = 0,
          newEntriesCount = 0,
          errorsCount = 0;
          existingEvents.forEach(function(event){
            var info = infosById[event.getId()];
            if(info) {
              try{
                if(info.updateSelf(event)) {
                  Logger.log('Updated Info: %s (%s %s-%s)', info.getId(),info.getTitle(),info.getStartTime(),info.getEndTime());
                  updateCount = updateCount + 1;
                }
              }
              catch(e) {
                Logger.log(e.message);
                errorsCount = errorsCount + 1;
              }
              } else {
                info = addEvent(event);
                Logger.log('Created Info: %s (%s %s-%s)', info.getId(),info.getTitle(),info.getStartTime(),info.getEndTime());
                newEntriesCount = newEntriesCount + 1;
              }
              SpreadsheetApp.flush();
              });

              var msgLines = [],
              timeout = 5;

              if(updateCount > 0) {
                msgLines.push("Updates:\t" + updateCount);
                timeout = -1;
              }  
              if(newEntriesCount > 0) {
                msgLines.push("New Entries:\t" + newEntriesCount);
                timeout = -1;
              }  
              if(errorsCount > 0) {
                msgLines.push("Errors:\t" + errorsCount);
                timeout = -1;
              }

              if(msgLines.length === 0) {
                msgLines.push("No changes");
              }
              msgLines.forEach(function(line) {Logger.log(line)});
              SpreadsheetApp.getActiveSpreadsheet().toast(msgLines.join("\n"),"Refresh Complete",timeout);
            }
```

Save the above script as "CalendarUtilsLib".
Then click `File->Manage Versions...`.
Create the initial version (the comment doesn't matter).
Lastly, go to `File->Project Properties`.
On the page lists the "Project key".
It will look something like this: `{{ page.calendarUtilsLibProjectKey }}`.
The project key is what is necessary to import this script as a library.

###Explanation
CalendarUtilsLib exposes two functions: `refreshFromCalendar(calendar, eventInfos, addEvent)` and `publish2calendar(calendar, eventInfos)`.
The first is the pull function that takes events from the calendar and will add them to your spread sheet, the second is the push.
The first parameter in each function is the calendar object from the Google Calendar API, the second parameter is a list of EventInfos which is sourced from the spreadsheet.
The next paragraph goes into more detail.
Lastly, `refreshFromCalendar()` has a third parameter: `addEvent`.
This is actually a function that accepts a calendar event and will create a new `EventInfo` and add it to your spread sheet.
This function is called when an event exists on the calendar, but not in the spreadsheet.
An example implementation will be showed below.
CalendarUtilsLib works through an abstraction called `EventInfo`.
Essentially, `EventInfo`s allow you to customize how the calendar events map to your spread sheet.
`EventInfo`s must have three methods:
`updateSelf(calendarEvent)`, `updateEvent(calendarEvent)`, and `isValid()`.
The first updates the spreadsheet with the calendar event, the second updates the calendar event.
Each return a boolean of whether or not anything changes occurred.
The last method is used to keep garbage data from messing things up.
If the info says it isn't valid, then it won't be pushed to the calendar.
Below is sample implementations, so I won't go into detail here.

##WorkSchedule Script
Below is the spreadsheet code for the spreadsheet that I want to sync with a calendar.
I start by creating my script files and then import CalendarUtilsLib.

###Creation
Open the spreadsheet you want to sync with a calendar.
Click `Tools->Script Editor`.
Add the following code segments.
I split them into two scripts, but that is my personal preference.

####main.gs
```javascript
function retrieveDatesCalendar() {
  return CalendarApp.getCalendarsByName('Work')[0];
}

function loadAllInfos() {
  return loadWorkScheduleEventInfos();
}

function publish2calendar() {
  CalendarUtilsLib.publish2calendar(retrieveDatesCalendar(),loadAllInfos());
}

function refreshFromCalendar() {
  CalendarUtilsLib.refreshFromCalendar(retrieveDatesCalendar(),loadAllInfos(), addEventToWorkSheet);
}
/*This function adds the Menu options*/
function onOpen() {
  var ss = SpreadsheetApp.getActiveSpreadsheet();
  var menuEntries = [
{name: "Refresh (calendar -> sheet)", functionName: "refreshFromCalendar"} ,
{name: "Publish (sheet -> calendar)", functionName: "publish2calendar"}
];
ss.addMenu("Calendar", menuEntries);
}
```

####WorkScheduleEventInfo.gs
```javascript
function loadWorkScheduleEventInfos() {
  var sheet = SpreadsheetApp.getActive().getSheetByName('Work')
  var startRow = 2;  // First row of data to process
  var numRows = sheet.getLastRow();  // Number of rows to process
  var dataRange = sheet.getRange(startRow, 1, numRows, 8);
  return dataRange.getValues().map(function(row,index){
    return new MiscEventInfo(sheet,startRow+index,row);
  });
}

function addEventToWorkSheet(event) {
  var sheet = SpreadsheetApp.getActive().getSheetByName('Work');
  var rowNumber = sheet.getLastRow()+1;
  var info = new WorkScheduleEventInfo(sheet,rowNumber,[]);
  info.updateSelf(event);
  //This is a little tweak I added so I can recognize when new events are added to the sheet.
  var newRowAsRange = sheet.getRange(rowNumber + ":" + rowNumber);
  newRowAsRange.setBackground("red");
  return info;
}

function WorkScheduleEventInfo(sheet, rowNumber, rowData) {
  var COLUMN_TITLE_INDEX = 0;
  var COLUMN_DESCRIPTION_INDEX = 1;
  var COLUMN_LOCATION_INDEX = 2;
  var COLUMN_START_TIMESTAMP_INDEX = 3;
  var COLUMN_END_TIMESTAMP_INDEX = 4;
  var COLUMN_EVENT_ID_INDEX = 5;

  this.isValid = function() {
    return rowData[COLUMN_TITLE_INDEX] && rowData[COLUMN_START_TIMESTAMP_INDEX] && rowData[COLUMN_END_TIMESTAMP_INDEX];
  }

  function updateField(index,value) {
    sheet.getRange(rowNumber, index+1).setValue(value);
    rowData[index] = value;
  }

  this.getTitle = function() {
    return rowData[COLUMN_TITLE_INDEX];
  };

  this.setTitle = function(title) {
    updateField(COLUMN_TITLE_INDEX,title);
  }

  this.getDescription = function() {
    return rowData[COLUMN_DESCRIPTION_INDEX];
  };

  this.setDescription = function(description) {
    updateField(COLUMN_DESCRIPTION_INDEX,description);
  }

  this.getLocation = function() {
    return rowData[COLUMN_LOCATION_INDEX];
  };

  this.setLocation = function(location) {
    updateField(COLUMN_LOCATION_INDEX,location);
  }

  this.getStartTime = function() {
    return rowData[COLUMN_START_TIMESTAMP_INDEX];
  }

  this.setStartTime = function(startTime) {
    updateField(COLUMN_START_TIMESTAMP_INDEX,startTime);
  }

  this.getEndTime = function() {
    return rowData[COLUMN_END_TIMESTAMP_INDEX];
  }  

  this.setEndTime = function(endTime) {
    updateField(COLUMN_END_TIMESTAMP_INDEX,endTime);
  }

  this.getId = function() {
    return rowData[COLUMN_EVENT_ID_INDEX];  
  }

  this.setId = function(id) {
    updateField(COLUMN_EVENT_ID_INDEX,id);
  }

  this.createEventOnCalendar = function(calendar) {
    return calendar.createEvent(this.getTitle(), this.getStartTime(), this.getEndTime(), {description:this.getDescription(),location:this.getLocation()});
  }

  this.updateSelf = function(event) {
    var updateOccurred = false;
    if(this.getId() != event.getId()) {
      if(this.getId()) {
        throw Error('Cannot update self with event with different id');
        } else {
          this.setId(event.getId());
          updateOccurred = true;
        }
      }
      if(this.getTitle() != event.getTitle()) {
        this.setTitle(event.getTitle());
        updateOccurred = true;
      }
      if(!this.getStartTime() || this.getStartTime().getTime() != event.getStartTime().getTime()) {
        this.setStartTime(event.getStartTime());
        updateOccurred = true;
      }
      if(!this.getEndTime() || this.getEndTime().getTime() != event.getEndTime().getTime()) {
        this.setEndTime(event.getEndTime());
        updateOccurred = true;
      }
      if(this.getLocation() != event.getLocation()) {
        this.setLocation(event.getLocation());
        updateOccurred = true;
      }
      if(this.getDescription() != event.getDescription()) {
        this.setDescription(event.getDescription());
        updateOccurred = true;
      }

      return updateOccurred;
    }

    this.updateEvent = function(event) {
      if(this.getId() !== event.getId()) {
        throw Error('Cannot update event with different id');
      }
      changeOccurred = false;
      if(event.getTitle() != this.getTitle()) {
        event.setTitle(this.getTitle());
        changeOccurred = true;
      }
      if(event.getStartTime().getTime() != this.getStartTime().getTime() || event.getEndTime().getTime() != this.getEndTime().getTime()) {
        event.setTime(this.getStartTime(),this.getEndTime());
        changeOccurred = true;
      }
      if(event.getLocation() != this.getLocation()) {
        event.setLocation(this.getLocation());
        changeOccurred = true;
      }
      if(event.getDescription() != this.getDescription()) {
        event.setDescription(this.getDescription());
        changeOccurred = true;
      }
      return changeOccurred;
    }

    this.toString = function() {
      return this.rowData;
    }
  }
```

After adding the code, you import CalendarUtilsLib by clicking `Resources->Libraries`.
Paste the Project Key from earlier (`{{ page.calendarUtilsLibProjectKey }}`).
Be sure to select a version.
The development mode allows you to edit the library and test it at the same time, but it is slower and google suggests not doing it.
If you aren't making changes to CalendarUtilsLib, then disable development mode.
At this point, you are ready to tinker and customize for your code.
Read below for more details.

###Explanation

The code above assumes a worksheet in your spreadsheet called "Work" with the following columns:

<table class="with-full-borders"><thead>
<tr>
  <th>Title</th>
  <th>Description</th>
  <th>Location</th>
  <th>Start Timestamp</th>
  <th>End Timestamp</th>
  <th>Event Id (don't touch)</th>
</tr>
</thead><tbody>
<tr>
  <td>Christmas Vacation</td>
  <td></td>
  <td>Home</td>
  <td>12/24/2012 0:00:00</td>
  <td>12/28/2012 0:00:00</td>
  <td></td>
</tr>
</tbody></table>


The event id column is very important.
This is what matches `EventInfo`s against Calendar events.
Because of that, it must exist and you should never manually change this column.

####main.gs
This script is where we setup the menu.
It needs to be able to retrieve the calendar (`retrieveDatesCalendar()`) and the infos (`loadAllInfos()`).
If your calendar is named differently, then update `retrieveDatesCalendar()`.
 I have two different spreadsheets that use CalenderUtilsLib and this portion of the script looks nearly the same in each.
 Pretty much the only difference is calendar name.
 The real customization occurs within the EventInfo.

####WorkScheduleEventInfo.gs
This is where you will be spending most of your tweaking.
I like to break my code into smaller chunks, which is why you see all of the setters and getters.
For me it makes the implementation of `updateEvent()`
and `updateSelf()` easier.
While the methods look big, each section is simply doing a comparison between event info and the calendar event to see if an update should occur.
 If so, it makes the change and marks that a change has occurred.
 The last bit of customization is in `addEventToWorkSheet()`.
 It works by creating a new `EventInfo` on the next blank line and doing an update.
 I have found this pattern works very well.
 For the sake of my own sanity, I color any new rows red.

##Conclusion
While there is a bit of setup work, I have used this pattern a couple times and it has really worked well for me.
 For my own spreadsheets, I have created varies `EventInfo` implementations where I some times dynamically generate fields.
 It has allowed a lot of flexibility.
