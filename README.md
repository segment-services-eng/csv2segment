**Welcome!**  *With this script you can transform CSV files into Segment API calls and send those calls locally from your machine.  This script has been pressure-tested at volume and has sent over 100M events to some of Segment's largest customers.  It's based on code originally published in https://github.com/brennan/segment-upload-scripts.  Big thanks to https://github.com/brennan/ for the inspiration!*

**IMPORTANT DISCLAIMER**
*IN PROVIDING THIS SCRIPT, SEGMENT MAKES NO REPRESENTATIONS ABOUT WARRANTY FOR A PARTICULAR PURPOSE OR USE. SEGMENT CUSTOMERS & EMPLOYEES USING THIS SCRIPT DO SO AT THIER OWN RISK.  Please consult Segment's documentation, support, and success before sending large volumes of data into your workspace, and please test all data in a separate dev source before sending into production sources.*


# Getting Started
You can start sending CSVs to Segment by following 6 simple steps--also outlined in the comments of csv2segment:

1. Set the delimiter
2. Set write keys
3. Enable the API method(s)
4. Prepare column variables
5. Customize your API calls
6. Run `csv2segment <filename.csv>`

Let's explore each step in more detail!

## 1. Set the delimiter

Specify the character which separates your CSV columns. Consider pipe-delimiting (`|`) if you have JSON or commas in your column values

```delimiter: '|',```


---

## 2. Set your write keys
It's important for you to test your CSV-to-Segment calls before pushing to a production write key.  For that reason, this script supports two write keys. The default behavior of the script sends the data to the DEV/TESTING source.

```Javascript
//Send calls to a DEV/TESTING Segment Source (replace DEVELOPMENT WRITE KEY with your own!)
const segment = {
  devKey : "DEVELOPMENT WRITE KEY", 
  productionKey : "PRODUCTION WRITE KEY", 
```

Replace `DEVELOPMENT WRITE KEY` with the write key for your Segment testing source and replace `PRODUCTION WRITE KEY` with your Segment production source.


---

## 3. Enable the API method(s)

Select which Segment method(s) you'll be using with your CSV by setting the values to `true`. Each of these methods will fire with EACH ROW of your CSV.  Fun fact: you can select more than one method!  In the example below, we're sending Track and Identify calls:

```Javascript
  // Select which Segment API method(s) you'll be using by setting the value to 'true'
  const fireIdentify = true;
  const fireGroup = false;
  const fireTrack = true;
  const firePage = false;
  const fireScreen = false;
```

And here we're sending only Page calls:
```Javascript
  // Select which Segment API method(s) you'll be using by setting the value to 'true'
  const fireIdentify = false;
  const fireGroup = false;
  const fireTrack = false;
  const firePage = true;
  const fireScreen = false;
```


---

## 4. Prepare your column variables

Before we start writing calls to the Segment API, it's super helpful to define (and format) variables for each of your CSV columns.  

To get started, let's define each of the columns in the ***CSV File*** files as an element of the `fileCols` object:

#### Sample CSV
|User ID| First Name | Last Name | Email Address | Zip Code | Page Visited | IP Address | Campaign | Anonymous ID| Date Added|
|------|------|------|------|------|------|------|------|------|------|
|1|Test|User|demo@segment.com|90210|Home Page|127.0.0.1|New Product|null|10/3/2021|
 
#### FileCols Object
```Javascript
const fileCols = {
    'User ID' : { trait : 'userID', datatype: 'Number', isUser: true},
    'First Name' : { trait : 'first_name' },
    'Last Name' : { trait : 'last_name', allowNull: true },
    'Email Address' : { trait : 'email', datatype: 'String' },
    'Zip Code' : { trait : 'zip_code', datatype: 'Number' },
    'Page Visited' : { trait : 'page_name', datatype: 'String', isPage: true},
    'IP Address' : { trait : 'id_address', context: true },
    'Campaign' : { trait : 'campaign', context: true },
    'Anonymous ID' : { trait : 'anonymousId', isAnonymous: true },
    'Date Added' : { trait : 'size', datatype: 'Date', isTimestamp: true }
}
```
Replace each of the column names above with the headers for your own CSV, and keep the following in mind when defining your variables:
- `trait: "trait or property name"` *Optional* defaults to snake case of column header
- `datatype : "String"`  *Optional* - defaults to `String` 
    - `Number` Cast numbers in the proper format with `Number(data[n])`
    - `Boolean` Cast booleans in the proper format with `Boolean(data[n])`
    - `Date` Cast (ISO-8601) datestrings in the proper format with `new Date(data[n])`
    - `JSON` Cast JSON in the proper format with `JSON.parse(data[n])`
- `allowNull : false`   *Optional* - defaults to **false**, if **true** the column can be set to 'null'
    - The script treats all CSV values as strings by default--even `null` values!  This element will allow replace the String with **null**
- `isUser : false`  Sets column as the userId  
- `isAnonymous : false`  Sets column as the annonymousId
- `isEvent : false`  Sets column as the event name  **REQUIRED** for Track calls
- `isGroup : false`  Sets column as the group name  **REQUIRED** for Group calls
- `isPage : false`   Sets column as the page name  **REQUIRED** for Page calls
- `isScreen : false`  Sets column as the screen name  **REQUIRED** for Screen calls
- `isTimestamp : false`  Sets column as the timestamp  **REQUIRED** for loading historical data
- `context : false`  Defines the value as a context variable


---

## 5. Customize your calls (*OPTIONAL*)

Now all the basics are in place and the standard behavior will function without any additional changes.  HOWERVER, you can customize any of the Segment calls you selected in Step 3, and map those calls against the columns you defined in step 4.

Remember the following when sending events to the Segment API:
- You *MUST* specify an `anonymousId` OR a `userId` in each call
- `Group` calls also require a `groupId`
- `Track` calls require an `event` value
- Timestamps *MUST* be in ISO-8601 format (e.g. `2017-03-12T15:15:41.029Z`) and cast as Dates to work properly
- The `integrations` object controls where your data goes downstream
- The `context` object can set useful context on your event (e.g. ip address)

---

### `Identify()` calls

Here's what a basic identify call looks like in the script:
```Javascript
function sendIdentifies(col){
    let package = {};
    package.userId = col.userId;
    if ("anonymousId" in col) package.anonymousId = col.anonymousId;
    package.traits = col.properties;
    package.integrations = {};
    if ("timestamp" in col) package.timestamp = col.timestamp;

    // Send the package via identify call
    analytics.identify(package);
    console.log('// identify call ' + col.userId)
  }
```

By default all traits defined in the **fileCols** object are sent as *properties* to with the `Identify` call, however you can customize the call by changing updating the **package** object.

---

### `Group()` calls

| ![alt text][warning] WARNING: You MUST set a column as `isGroup` for the call to go through!  |
| --- |

[warning]: https://res.cloudinary.com/segment-cx/image/upload/c_scale,w_18/v1563315183/warning.png "Warning Icon"

Here's the default `Group` call in our script:
```Javascript
function sendGroups(col){
    if ("groupId" in col) {
      let package = {};
      package.groupId = col.groupId;
      package.userId = col.userId;
      if ("anonymousId" in col) package.anonymousId = col.anonymousId;
      package.traits = col.properties;
      package.integrations = {};
      if ("timestamp" in col) package.timestamp = col.timestamp;
      analytics.group(package);
      console.log('// group call  ' + package.groupId + ' user ' + package.userId);
    } else {
      console.log('INVALID GROUP CALL - missing groupId');
    }
  }
```

By default all traits defined in the **fileCols** object are sent as *traits* to with the `Group` call, however you can customize the call by changing updating the **package** object.

---

### `Track()` Calls

| ![alt text][warning] WARNING: You MUST set a column as `isEvent` for the call to go through!  |
| --- |

Here's the default `Track` call in our script:
```Javascript
function sendTracks(col){
  
    if ("event" in col) {
      let package = {};
      package.event = col.event;
      package.userId = col.userId;
      if ("anonymousId" in col) package.anonymousId = col.anonymousId;
      package.properties = col.properties;
      package.integrations = {};
      if ("timestamp" in col) package.timestamp = col.timestamp;
    
      // Send the package via track call
      analytics.track(package);
      console.log('// track call ' + package.userId);
    } else {
      console.log('INVALID EVENT CALL - missing Event Name');
    }
 }
```

By default all traits defined in the **fileCols** object are sent as *properties* to with the `Track` call, however you can customize the call by changing updating the **package** object.


---

### `Page()` or `Screen()` Calls


| ![alt text][warning] WARNING: You MUST set a column as `isPage` or `isScreen` for the call to go through!  |
| --- |

Here's the default `Page` call in our script:
```Javascript
 function sendPages(col){
    if ("pageName" in col) {
      let package = {};
      package.name = col.pageName;
      package.userId = col.userId;
      if ("anonymousId" in col) package.anonymousId = col.anonymousId;
      package.properties = col.properties;
      package.integrations = {};
      if ("timestamp" in col) package.timestamp = col.timestamp;
      if ("context" in col) package.context =  col.context;
      
      // Send the package via page call
      analytics.page(package);
      console.log('// page call ' + package.name + ' by ' + package.userId);
    } else {
      console.log('INVALID PAGE CALL - missing Page Name');
    }
  }
```

By default traits defined in the **fileCols** object with `context: false` are sent as *properties* with the `Page()` or `Screen()` call and objects with `context: true` are sent as *context*  

You can customize the call by changing updating the **package** object.

---

## 6. Run the script
Once you've configured everything in your script, running it is super easy!

1. Open the script directory in your terminal
2. run `npm install` for the first time *only*
3. Make sure Node is version 7 or higher
3. run `./csv2segment <filename.csv>` and the script should run!

---

# Troubleshooting

## My calls aren't appearing in the debugger!
- Is there DEFINITELY an anonymousId OR userId defined for identify calls?
- Is there DEFINITELY an `isGroup` plus the above for Group calls?
- Is there DEFINITELY an `isEvent` value for Track calls?
- Are Node and NPM running on the latest version?
- Have you tried pipe-delimiting your CSV?

## My timestamps are off in the debugger
Segment shows your timestamps in the debugger relative to the timezone set in your Segment settings.  So if you send UTC timestamps into Segment but your workspace is set to EST, all timestamps will show up in the debugger 5 hours earlier than expected.  This only  happens in the Segment debugger, and the timestamps as reported in the actual event payload will be the one leveraged in your data warehouse and other downstream destinations.  

## 5-10% of my events didn't show up in Segment
Are you connected via WiFi?  Try Ethernet!

---

# Advanced Methods
Interested in doing some fancier things in your event payloads? Read on!

## Backdated Events with Custom Timestamps
Timestamps are optional, and help you backdate your events--which is super handy for backfilling historical data! If you plan on doing so...
- *Make sure your timestamps are in ISO-8601 format!!* (e.g. `2017-03-12T15:15:41.029Z`)
- Cast a timestamp appearing in column `n` using `new Date(data[n])`
- Transform timestamp values using string functions like var.concat()
