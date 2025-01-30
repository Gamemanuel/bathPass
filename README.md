# Bath Pass
A way to keep track of your students whereabouts and wanderings without leaving the computer.

## Initial setup 
### Setup of Google sheet

### 1. Create a new Google Sheet

- First, go to [Google Sheets](https://docs.google.com/spreadsheets) and `Start a new spreadsheet` with the `Blank` template.
- Rename it `Email Subscribers`. Or whatever, it doesn't matter.
- Put the following headers into the first row:

|   |     A     |   B   | C | ... |
|---|:---------:|:-----:|:-:|:---:|
| 1 | timestamp | email |   |     |

### 2. Create a Google Apps Script

- Click on `Tools > Script Editor…` which should open a new tab.
- Rename it `Submit Form to Google Sheets`. _Make sure to wait for it to actually save and update the title before editing the script._
- Now, delete the `function myFunction() {}` block within the `Code.gs` tab.
- Paste the following script in it's place and `File > Save`:

```js
// The default sheet name is 'Sheet1'. To target a different sheet, update this variable.
var sheetName = 'Sheet1'

/*
Gets a property store that all users can access, but only within this script.
https://developers.google.com/apps-script/reference/properties/properties-service#getScriptProperties()
*/
var scriptProp = PropertiesService.getScriptProperties()

/*
This is the initial setup function. It gets the active SpreadsheetApp ID and adds it to our PropertiesService.
https://developers.google.com/apps-script/reference/spreadsheet/spreadsheet-app#getactivespreadsheet
*/
function setup () {
  var doc = SpreadsheetApp.getActiveSpreadsheet()
  scriptProp.setProperty('key', doc.getId())
}

function doPost (e) {
  /*
  Gets a lock that prevents any user from concurrently running a section of code. A code section
  guarded by a script lock cannot be executed simultaneously regardless of the user's identity.
  https://developers.google.com/apps-script/reference/lock/lock-service#getScriptLock()
  */
  var lock = LockService.getScriptLock()

  /*
  Attempts to acquire the lock, timing out with an exception after the provided number of milliseconds.
  This method is the same as tryLock(timeoutInMillis) except it throws an exception when the lock
  cannot be acquired instead of returning false.
  https://developers.google.com/apps-script/reference/lock/lock#waitLock(Integer)
  */
  lock.waitLock(10000)

  try {
    /*
    Opens the spreadsheet with the given ID. A spreadsheet ID can be extracted from its URL. For example,
    the spreadsheet ID in the URL https://docs.google.com/spreadsheets/d/abc1234567/edit#gid=0 is "abc1234567".
    https://developers.google.com/apps-script/reference/spreadsheet/spreadsheet-app#openbyidid
    */
    var doc = SpreadsheetApp.openById(scriptProp.getProperty('key'))

    /*
    Returns a sheet with the given name. If multiple sheets have the same name,
    the leftmost one is returned. Returns null if there is no sheet with the given name.
    https://developers.google.com/apps-script/reference/spreadsheet/spreadsheet#getSheetByName(String)
    */
    var sheet = doc.getSheetByName(sheetName)

    /*
    Returns the range with the top left cell at the given coordinates, and with the given number of rows.
    https://developers.google.com/apps-script/reference/spreadsheet/sheet#getRange(Integer,Integer)

    Then returns the position of the last column that has content.
    https://developers.google.com/apps-script/reference/spreadsheet/sheet#getlastcolumn

    Then returns the rectangular grid of values for this range (a two-dimensional array of values, indexed by row, then by column.)
    https://developers.google.com/apps-script/reference/spreadsheet/range#getValues()
    */
    var headers = sheet.getRange(1, 1, 1, sheet.getLastColumn()).getValues()[0]
    // Gets the last row and then adds one
    var nextRow = sheet.getLastRow() + 1

    /*
    Maps the headers array to a new array. If a header's value is 'timestamp' then it
    returns a new Date() object, otherwise, it returns the value of the matching URL parameter
    https://developers.google.com/apps-script/guides/web
    */
    var newRow = headers.map(function(header) {
      return header === 'timestamp' ? new Date() : e.parameter[header]
    })

    /*
    Gets a range from the next row to the end row based on how many items are in newRow
    then sets the new values of the whole array at once.
    https://developers.google.com/apps-script/reference/spreadsheet/range#setValues(Object)
    */
    sheet.getRange(nextRow, 1, 1, newRow.length).setValues([newRow])

    /*
    Return success results as JSON
    https://developers.google.com/apps-script/reference/content/content-service
    */
    return ContentService
      .createTextOutput(JSON.stringify({ 'result': 'success', 'row': nextRow }))
      .setMimeType(ContentService.MimeType.JSON)
  }

  /*
  Return error results as JSON
  https://developers.google.com/apps-script/reference/content/content-service
  */
  catch (e) {
    return ContentService
      .createTextOutput(JSON.stringify({ 'result': 'error', 'error': e }))
      .setMimeType(ContentService.MimeType.JSON)
  }

  finally {
    /*
    Releases the lock, allowing other processes waiting on the lock to continue.
    https://developers.google.com/apps-script/reference/lock/lock#releaseLock()
    */
    lock.releaseLock()
  }
}

```

### 3. Run the setup function

- Next, go to `Run > Run Function > initialSetup` to run this function.
- In the `Authorization Required` dialog, click on `Review Permissions`.
- Sign in or pick the Google account associated with this projects.
- You should see a dialog that says `Hi {Your Name}`, `Submit Form to Google Sheets wants to`...
- Click `Allow`

### 4. Add a new project trigger 
- Click on `Edit > Current project’s triggers`. 
- In the dialog click `No triggers set up. Click here to add one now.` 
- In the dropdowns select `doPost`
- Set the events fields to `From spreadsheet` and `On form submit`
- Then click `Save`

### 5. Publish the project as a web app

- Click on `Publish > Deploy as web app…`.
- Set `Project Version` to `New` and put `initial version` in the input field below.
- Leave `Execute the app as:` set to `Me(your@address.com)`.
- For `Who has access to the app:` select `Anyone, even anonymous`.
- Click `Deploy`.
- In the popup, copy the `Current web app URL` from the dialog.
- And click `OK`.

> **IMPORTANT!** If you have a custom domain with Gmail, you _might_ need to click `OK`, refresh the page, and then go to `Publish > Deploy as web app…` again to get the proper web app URL. It should look something like `https://script.google.com/a/yourdomain.com/macros/s/XXXX…`.

