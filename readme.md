### What is it?
It's the Google Voice API for [node.js](http://nodejs.org/). Except there is no official "Google Voice API", so node-google-voice is also the only (javascript) Google Voice API. 
It currently allows you to:

* place calls
* send SMSs
* access and manipulate GV messages
* access GV settings

### What's new in 0.1 ?

* new common, consistent methods for calling/texting and getting/setting messages
* voicemail download
* new methods for getting Google Voice settings
* extra get and set methods, such as 'unread', 'saveNote'/'deleteNote', 'saveTranscript'/'restoreTranscript'
* faster message requests, due to internal parallel downloading of data
* no need to obtain the RNR_SE id manually

(Note that the call scheduler has been removed in 0.1.)

### Dependencies

* [node.js](http://nodejs.org) - node-google-voice has been tested with Node 0.6.7. YMMV with other versions. 
* [googleclientlogin](https://github.com/Ajnasz/GoogleClientLogin) - used for authentication to the Google Voice service
* [xml2js](https://github.com/Leonidas-from-XIV/node-xml2js) - used for extracting JSON & HTML data from Google Voice XML responses
* [jsdom](https://github.com/tmpvar/jsdom) - for extracting SMS threads from GV data
* [request](https://github.com/mikeal/request) - for all http requests

[npm](https://github.com/isaacs/npm) should take care of dependencies, but in case it fails to do so, try installing these modules (and *their* dependencies) independently.

### Installation
Install node-google-voice via npm:

	npm install google-voice
	
(In more recent versions of Node, npm is included. See [the npm page](https://github.com/isaacs/npm) for information on installing npm with older versions of Node.)

### Examples
###### Create a GV.Client instance

	var GV = require('google-voice');

	var client = new GV.Client({
		email: 'email@gmail.com',
		password: 'password'
	});
	
###### Get the numbers of unread messages in each GV label

	client.getCounts(function(error, counts){
		if(err){
			console.log('Error: ',err);
			return;
		}
		
		console.log(counts);
	});

###### Place a call to 18005551212 using the mobile phone number 1234567890 associated with the GV account
	
	client.connect('call',{outgoingNumber:'18005551212', forwardingNumber:'1234567890', phoneType:2}, function(error, response, body){
		if(error){
			console.log('Error: ', error);
		}else{
			console.log('Call placed with status: ',body);
		}
	});


###### Send a text to 18005551212 and 1234567890

	client.connect('sms',{outgoingNumber:['18005551212','1234567890'], text:'Guys, come over for dinner tomorrow!'}, function(error, response, body){
		var data = JSON.parse(body);
		if(error || !data.ok){
			console.log('Error: ', error, ', response: ', body);
		}else{
			console.log('Text sent succesfully.');
		}
	});

###### Cancel the last call

	client.connect('cancel');

###### Display messages in the inbox, indicating if they have been read or not

	client.get('inbox',null,function(error, response){
		if(error){
			console.log('Error: ',error);
		}else{
			console.log('There are',response.total,'messages in the inbox. The last', response.messages.length, 'are:');
			response.messages.forEach(function(msg, index){
				console.log(msg.isRead ? ' ' : '*', (index+1)+'.', msg.displayStartDateTime, msg.displayNumber);
			});
		}
	});

###### Retrieve the most recent SMS and display its message thread
	
	client.get('sms',{limit:1},function(error, response){
		if(error){
			console.log('Error: ',error);
		}else{
			var latest = response.messages[0];
			
			if(!latest){ 
				console.log('No texts found.');
				return; 
			}
			
			console.log('From ', latest.displayNumber, ' at ', latest.displayStartDateTime);
			latest.thread.forEach(function(sms){
				console.log(sms.time, sms.from, sms.text);
			})
		}
	});

###### Star all messages to/from/mentioning 'mom' and download any that are voicemails
	
	client.get('search',{query: 'mom', limit:Infinity}, function(error,response){
		if(error){
			console.log('Error: ', error);
		}else{
			response.messages.forEach(function(msg){
				client.set('star',{id: msg.id});
				if(!!~msg.labels.indexOf('voicemail')){
					var fileName = msg.id + '.mp3';
					client.download({id: msg.id, file: fileName}, function(error, httpResponse, body){
						console.log(error ? 'Error downloading message ' + msg.id : 'Downloaded '+fileName);
					});
				}
			})
		}
	});
	
The above approach sends a 'star' request to Google Voice for *each* message. This may result in too much network traffic. A much more efficient way would be to store the message ids in an Array and issue the 'star' request once:
	
	client.get('search',{query: 'mom', limit:Infinity}, function(error, response){
		if(error){
			console.log('Error: ', error);
		}else{
			var idArray = [];
			response.messages.forEach(function(msg){
				var idArray.push(msg.id);
				
				if(!!~msg.labels.indexOf('voicemail')){
					var fileName = msg.id + '.mp3';
					client.download({id: msg.id, file: fileName}, function(error, httpResponse, body){
						console.log(error ? 'Error downloading message ' + msg.id : 'Downloaded '+fileName);
					});
				}
			});
			
			client.set('star',{id:idArray});
		}
	});

This results in only TWO requests being sent to Google Voice.

###### Update the transcript of the most recent voicemail and donate it to Google so that the good folks there can improve their transcribing feature

	client.get('voicemail', null, function(error,response){
		if(error){
			console.log('Error: ',error);
			return;
		}
		if(!response.messages.length){ return; }
		
		var latest = response.messages[0];
		client.set('saveTranscript',{id: latest.id, transcript: 'Here is the correct transcript, Google!'}, function(error, httpResponse, body){
			var data = JSON.parse(body);
			if(error || !data.ok){
				console.log('Error: ', error, ', response: ', body);
			}else{
				console.log('Transcript updated.');
				client.set('donate',{id: latest.id});
			}
		})
		
	});

###### Get all the phones associated with the Google Voice account
	
	client.getSettings(function(error, settings){
		if(error){ console.log('Error: ',error); return; }
		console.log(settings.phones);
	});


### API
The GV object has the following properties:

* Client
	* connect(method, options, callback)
	* get(type, options, callback)
	* set(type, options, callback)
	* download(id/options, callback)
	* getCounts(callback)
	* getSettings(callback)
	* config
		* email
		* password
		* rnr_se
	* settings
	* unreadCounts
* STATUSES


### Instantiate a Google Voice client: new GV.Client(options)

Google Voice client instances are created in the following manner

	var GV = require('google-voice');
	var client = new GV.Client(options);
	
where `options` is an Object with the following properties:

* `email` (String, required) - your Google Voice login email
* `password` (String, required) - your Google Voice password
* `rnr_se` (String, optional) - the unique identifier associated with your GV account
	* The `rnr_se` is some kind of ID created by Google for your Google Voice account. At this time, it can be obtained by logging into the Google Voice web front-end and running the following javascript snippet in the browser window:
	
	```javascript
	javascript:alert(_gcData._rnr_se);
	```
	
	* You only have to get your rnr_se once, because it doesn't appear to change. However, if something does stop working in node-google-voice and you are manually supplying the `rnr_se`, first check that your `rnr_se` hasn't changed.
	* If you supply `rnr_se`, you will save node-google-voice one extra HTTP request on first authentication. If you don't supply it, node-google-voice will make an attempt to get it automatically. If it fails, a 'GET_RNRSE_ERROR' error will occur.



### Calling and Texting: GV.Client.connect(method, options, callback)
This is the common method for texting, calling, and canceling calls. The parameters are:

* `method` (String, required): one of 'call', 'sms', or 'cancel'
* `options`  (Object, required/optional): different properties are required, depending on the `method` used. See below.
* `callback`  (Function(error, response, body), optional):
	* `error` (Number) - See Status Codes below.
	* `response` (Http.ClientResponse): an instance of Node's [http.ClientResponse](http://nodejs.org/docs/v0.4.7/api/http.html#http.ClientResponse). This is the given response for that particular request. It is provided for cases where you would like to get more information about what went wrong (or right) and act on it.
	* `body` (String): the response from Google Voice for the request. See 'Google Response' below.


#### Options parameters depending on `method`

###### 'call'

* `outgoingNumber` (String, required) - the number you wish to be connected TO
* `forwardingNumber` (String, required) - the phone on your GV account that you wish to connect WITH
* `phoneType` (Number, required) - the phoneType of the `forwardingNumber`. Options are:
	* 1 - Home
	* 2 - Mobile
	* 3 - Work
	* 7 - Gizmo (may not work, as Google has acquired Gizmo)
	* 9 - Google Talk

	(Note that information about the phones and phoneTypes on your GV account can be obtained from the `client.settings.phones`, after fetching the settings. See `GV.Client.getSettings` below.)

###### 'sms'

* `outgoingNumber` (String or Array of Strings, required)
* `text` (String, optional)

###### 'cancel'

This method cancels the current outgoing call before it is connected. No options are required. `options` can be `null`, or the following form can be used: `GV.Client.connect('cancel',callback)`




### Retrieving Google Voice Messages: GV.Client.get(type, options, callback)
This is the common method for fetching Google Voice messages, whether they are texts, voicemails, missed calls, etc..

* `type` (String, required): Corresponds to the standard Google Voice labels. Can be one of the following:
	
	```javascript
		'unread'
		'inbox'
		'all'
		'spam'
		'trash'
		'starred'
		'sms'
		'voicemail'
		'placed'
		'missed'
		'received'
		'recorded'
		'search' // this method REQUIRES a `query` property in `options`
	```
	
* `options` (Object, required/null): REQUIRED when `type` is search. Optional otherwise, but must be set to `null`. Can have the following properties:
	* `start` (Number, optional, default is `1`): the index of the first message. An 'OUT_OF_BOUND_LIMIT' error will occur if `start` is greater than the number of messages for the request.
	* `limit` (Number, optional, default is variable): maximum number of messages to return. The default is whatever Google's `resultsPerPage` value is (at the time of this writing it's 10). Set this to `Infinity` to retrieve ALL messages pertaining to the request. 
	* `query` (String, required for type=='search'): The search function is entirely implemented by Google, so the search results are the same as would be returned by searching from the Google Voice web interface.
* `callback` (Function(error, response), required)
	* `error` (Number): see Status Codes below.
	* `response` (Object): has the following properties:
		* `total` (Number): the total number of messages matching this request
		* `messages` (Array): an array of Google Voice message objects, sorted by `startTime`. The properties of each message object are whatever Google Voice is supplying at the time of the request. A few properties are added by node-google-voice (indicated below). At the time of this writing, an example message looked like this:	
			
			```javascript	
				{ id: 'someStringIdentifier',
				  phoneNumber: '+18005551212',
				  displayNumber: '(800) 555-1212',
				  startTime: '1305138033000',
				  displayStartDateTime: '5/11/11 2:20 PM',
				  displayStartTime: '2:20 PM',
				  relativeStartTime: '3 weeks ago',
				  note: 'custom note',
				  isRead: true,
				  isSpam: false,
				  isTrash: false,
				  star: false,
				  labels: [ 'missed', 'all' ],
				  type: 0,
				  children: '' ,
				  messageText: 'Hi. This is the message transcript',
				}
			```
			
			Note that the `length` of the Array may be equal to OR less than `options.limit`. It will be LESS, if there are less messages matching the request.
			
			Properties added by node-google-voice to some messages:
			* `url` (String, only for voicemails and recordings): the url of the message audio
			* `thread` (Array, only for texts): the conversation thread, containing Objects with the following properties
				* `from` (String)
				* `time` (String)
				* `text` (String)



### Updating unread counts: GV.Client.getCounts(callback)

Every time `.get()` is used, the client's `unreadCounts` property is updated with the latest unread count for each label in Google Voice.
There is also a `.getCounts(callback)` method to do this manually. It takes one argument, `callback`:

* callback (Function(error, counts)) where
	* `error` (Number): see Status Codes below
	* `counts` (Object): the `unreadCounts` object given by Google Voice. At the time of this writing, it had the following properties:
	



### Downloading voicemails and recorded calls: GV.Client.download(id,callback), GV.Client.download(options,callback)
These two methods allow you to download the audio recording of voicemails and recorded calls. Both versions download and present the binary data to `callback`. The second version also can save the recording to the file system. The arguments are:

* id (String): the unique message id of the voicemail or recording. It is up to you to make sure that the id you supply is for a voicemail or recording; otherwise you will get an 'HTTP_ERROR'
* options (Object) with the following properties:
	* id (String, required) - Unique voicemail message id
	* file (String, optional) - The file name to use when saving the audio (this will overwrite any identically-named files!!!). If this is omitted, the voicemail will NOT be saved to disk, but only presented to `callback`.
* callback (Function(error, httpResponse, body), optional)
	* error (Number) - see Status Codes below
	* httpResponse (Http.ClientResponse)
	* body (Buffer) - the binary audio data

### Modifying Google Voice messages: GV.Client.set(type, options, callback)
This is the common setter method that manipulates GV messages on the server. Arguments are:

* `type` (String, required)

	```javascript
		'read' 				// mark message(s) as read
		'unread' 			// mark message(s) as unread
		'archive'			// archive message(s)
		'unarchive'			// unarchive message(s) (move back to inbox)
		'star'				// star message(s)
		'unstar',			// remove star label from message(s)
		'spam', 			// mark message(s) as spam
		'unspam' 			// unmark message(s) as spam
		'deleteForever' 	// permanently deletes the message(s)
		'toggleTrash' 		// calling this on a message will move it to the inbox if it is in the trash OR will move it to the trash if it is somewhere else
		'block' 			// block the number associated with the message(s)
		'unblock' 			// unblock the number associated with the message(s)
		'saveNote'			// save custom note for a message
		'deleteNote'		// delete a message's note
		'saveTranscript'	// modify the transcript of a voicemail
		'restoreTranscript'	// restore the transcript of a voicemail to Google Voice's original transcript
		'donate' 			// 'donate' the transcript of a voicemail to Google for testing with a human operator
		'undonate'			// undo a 'donate'
		'forward'			// forward the transcript of a voicemail (and optionally, a link to the mp3) to a list of email addresses, with a custom subject and body. Recorded calls can be sent this way too, but they are not transcribed, so make sure to include link:true so that the audio is sent.
	```

* `options` (Object, required): properties include:
	* `id` (String or Array of Strings, required): the unique message id(s) of the messages to manipulate. Note: if type is one of 'saveNote', 'deleteNote', 'saveTranscript', 'restoreTranscript', or 'forward', an Array of ids MAY NOT BE USED. Only one message can be manipulated at a time with these methods. A CANNOT_SET_MULTIPLE_MSGS error will occur. 
	* `note` (String, required for type=='saveNote')
	* `transcript` (String, required only for type=='saveTranscript')
	* `email` (String or Array of Strings, required only for type=='forward'): email address(es) to which the voicemail or recording will be sent
	* `subject` (String, optional, only for type=='forward'): the subject of the email message
	* `body` (String, optional, only for type=='forward'): the body of the email message
	* `link` (Boolean, optional, default is false, only for type=='forward'): whether the email should include a link to the mp3 of the voicemail
* `callback` (Function(error, response, body), optional)
	* `error` (Number): see Status Codes below
	* `response` (Http.ClientResponse): an instance of Node's [http.ClientResponse](http://nodejs.org/docs/v0.4.7/api/http.html#http.ClientResponse). 
	* `body` (String): the response from Google Voice for the request. See 'Google Responses' below.
### Getting Google Voice settings: GV.Client.getSettings(callback)
* `callback` (Function(error, settings))
	* `error` (Number): see Status Codes below
	* `callback` (Function(error, settings)): `settings` is the settings object from GV. At the time of this writing, it had the following form:
	
	```javascript
	{ phones: 
	   { '2': 
	      { id: 2,
	        name: 'Gizmo',
	        phoneNumber: '+##########',
	        type: 7,
	        verified: true,
	        policyBitmask: 0,
	        dEPRECATEDDisabled: false,
	        telephonyVerified: true,
	        smsEnabled: false,
	        incomingAccessNumber: '',
	        voicemailForwardingVerified: false,
	        behaviorOnRedirect: 0,
	        carrier: '',
	        customOverrideState: 0,
	        inVerification: false,
	        androidVoipJid: '',
	        recentlyProvisionedOrDeprovisioned: false,
	        formattedNumber: '(###) ###-####',
	        wd: [Object],
	        we: [Object],
	        scheduleSet: false,
	        weekdayAllDay: false,
	        weekdayTimes: [],
	        weekendAllDay: false,
	        weekendTimes: [],
	        redirectToVoicemail: false,
	        active: true,
	        enabledForOthers: true },
	     '3': 
	      { id: 3,
	        name: 'GMail Talk',
	        phoneNumber: 'username@gmail.com',
	        type: 9,
	        verified: true,
	        policyBitmask: 0,
	        dEPRECATEDDisabled: false,
	        telephonyVerified: false,
	        smsEnabled: false,
	        incomingAccessNumber: '',
	        voicemailForwardingVerified: false,
	        behaviorOnRedirect: 0,
	        carrier: '',
	        customOverrideState: 0,
	        inVerification: false,
	        androidVoipJid: '',
	        recentlyProvisionedOrDeprovisioned: false,
	        formattedNumber: 'username@gmail.com',
	        wd: [Object],
	        we: [Object],
	        scheduleSet: false,
	        weekdayAllDay: false,
	        weekdayTimes: [],
	        weekendAllDay: false,
	        weekendTimes: [],
	        redirectToVoicemail: false,
	        active: true,
	        enabledForOthers: true }},
	  phoneList: [ 2, 3],
	  settings: 
	   { primaryDid: '+##########',
	     language: 'en',
	     screenBehavior: 1,
	     useDidAsCallerId: false,
	     credits: 600,
	     timezone: 'America/New_York',
	     doNotDisturb: false,
	     directRtp: 0,
	     filterGlobalSpam: 1,
	     enablePinAccess: 1,
	     didInfos: [],
	     smsNotifications: [],
	     emailNotificationActive: true,
	     emailNotificationAddress: 'username@gmail.com',
	     smsToEmailActive: true,
	     smsToEmailSubject: false,
	     missedToEmail: true,
	     showTranscripts: true,
	     directConnect: false,
	     useDidAsSource: true,
	     emailToSmsActive: false,
	     i18nSmsActive: false,
	     missedToInbox: false,
	     greetings: [ [Object] ],
	     greetingsMap: {},
	     activeForwardingIds: [ 2, 3],
	     disabledIdMap: {},
	     defaultGreetingId: 0,
	     webCallButtons: [ [Object] ],
	     groups: 
	      { '13': [Object],
	        '14': [Object],
	        '15': [Object],
	        '9999': [Object],
	        '10000': [Object],
	        '1431499560409739077': [Object] },
	     groupList: [ '10000', '9999', '15', '13', '14', '1431499560409739077' ],
	     lowBalanceNotificationEnabled: false,
	     emailAddresses: [ 'username@gmail.com', 'an_email_@push.boxcar.io' ],
	     baseUrl: 'https://www.google.com/voice' } }
	```
	

### Status Codes: GV.STATUSES
In usual Node fashion, every callback's first argument is `error`. By convention, `error` is `0` or `null` for a successful request. Other codes indicate failure. The list of all codes is found in `GV.STATUSES`. An `error` of `0` for the `client.connect` and `client.set` methods just means that there were no errors up to the point of submitting the request to Google. The response from Google may have an error code in it, and can be parsed by the end user. See 'Google Responses' below.

### Google Responses
Some callbacks have a response from Google in the `body`. These include callbacks for `client.connect` and `client.set`. It is JSON data, and is typically something like:
 
* `{ ok: true, data: { code: 0 } }`
* `{ ok: false, error: 'Cannot complete call.' }`
* `{ ok: false, data: { code: 20 } }`. 
* `{"ok":false,"error":"Can't edit text of non-voicemail call #A#####AA$$$A$$$$A$$$$A###AAA###A#A####AAA#AA","errorCode":84}`
	
At this time, node-google-voice makes no attempts to parse these responses, because the string can change as Google makes changes to how Google Voice works. What is important to note is that there is usually a Boolean `ok` property in the JSON response. This is not parsed by node-google-voice at this time, so the `error` in callback may not reflect whether the request was 'approved' by Google. The end-user may wish to parse this response to see if `body.ok===true` in order to make final decisions on the success or failure of a request. Some of the examples show how this can be done.


### TODO
* allow changing of various Google Voice settings
* retrieve messages by id, not just by label
* retrieve contacts
* a schedule plugin to use Google Calendar to automatically place calls or send texts

### License
What? This project is [UNLICENSED](http://unlicense.org/) .

### Conclusion
Google does not have an official Google Voice API. Therefore, the nature of the requests and returned data can change without notice. It is unlikely to change often or soon, but I will make all efforts to keep up with the most current implementation. If you have any issues, please give me a shout, and I'll do my best to address them. I have not trained as a developer (I code only as a hobby), so I'm also open to any constructive criticism on best coding practices and the like. 

Enjoy!
