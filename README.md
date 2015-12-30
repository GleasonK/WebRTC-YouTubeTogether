# WebRTC - YouTube Together

WebRTC is becoming widely adopted by large companies everywhere. If you don't know it already, WebRTC is a free, open-source project that provides simple APIs for creating Real-Time Communications (RTC) for browsers and mobile devices. It powers many modern video chatting services, even Chromecast uses it to broadcast your chrome tab to the TV screen. It makes streaming any content such as video, audio, or arbitrary data simple and _fast_. Today we will be building a video chat application that allows you to watch YouTube with a friend!

## WebRTC and PubNub Signaling

WebRTC is not a standalone API, it needs a signaling service to coordinate communication. Metadata needs to be sent between callers before a connection can be established. 

This metadata includes things such as:

- Session control messages to open and close connections
- Error messages
- Codecs/Codec settings, bandwidth and media types
- Keys to establish a secure connection
- Network data such as host IP and port

Once signaling has taken place, video/audio/data is streamed directly between clients, using WebRTC's `PeerConnection` API. This peer-to-peer direct connection allows you to stream high-bandwidth robust data, like video. In addition, we will be using `DataChannels` to stream messages through our RTC connection.

PubNub makes this signaling incredibly simple, and then gives you the power to do so much more with your WebRTC applications.

### Browser Compatibility

WebRTC is supported by popular browsers such as Chrome and Firefox, but there are many browsers on which certain features will not work. See a list of [supported browsers here](http://iswebrtcreadyyet.com/).

## Part 1: The Video Setup
Time to begin! First we will make the bare minimum WebRTC video chat. Then, in Part 2 we will be using WebRTC `DataChannels` and the YouTube API to create our application. The live demo for this tutorial [can be found here](http://kevingleason.me/WebRTC-TicTacToe/rtctactoe.html)!

### A Note on Testing and Debugging

If you try to open `file://<your-webrtc-project>` in your browser, you will likely run into Cross-Origin Resource Sharing (CORS) errors since the browser will block your requests to use video and microphone features. To test your code you have a few options. You can upload your files to a web server, like [Github Pages](https://pages.github.com/) if you prefer. In production, WebRTC requires HTTPS, so if you choose this method, visit your pages as `https://yourusername.github.io/project`. However, to keep development local, I recommend you setup a simple server using Python.

To so this, open your terminal and change directories into your current project and depending on your version of Python, run one of the following modules.

    cd <project-dir>

    # Python 2
    python -m SimpleHTTPServer <portNo>
    
    # Python 3
    python -m http.server <portNo>
    
For example, I run Python2.7 and the command I use is `python -m SimpleHTTPServer 8001`. Now I can go to `http://localhost:8001/index.html` to debug my app! Try making an `index.html` with anything in it and serve it on localhost before you continue.

### Step 1: The HTML5 Backbone

```html
<div id="player"></div>
<div style="float: left; width: 50%;">
    <div id="video-chat" hidden="true" style="margin-bottom: 10px;">
		<div id="vid-box"></div>
		<button onclick="end()">End Call</button>
    </div>
    
    <form name="loginForm" id="login" action="#" onsubmit="return login(this);">
        	<input type="text" name="username" id="username" placeholder="Enter A Username"/>            
			<input type="submit" name="login_submit" value="Log In">
    </form>

	<form name="callForm" id="call" action="#" onsubmit="return makeCall(this);">
        <input type="text" name="number" id="call" placeholder="Enter User To Call!"/>
        <input type="submit" value="Call">
	</form>
	
	<form name="cueForm" id="cue" action="#" onsubmit="return cueFromURL(this);">
        <input type="text" name="url" id="url" placeholder="Enter YouTube Video URL"/>
        <input type="submit" value="Queue Video">
	</form>
</div>
```

This should leave you with a very basic HTML backbone that looks something like this:

![HTML Backbone](img/html.png)

The `player` div will house our YouTube video, while `vid-box` will hold the video chat. We will use the forms to log-in, place calls, and enqueue videos. 

### Step 2: The JavaScript Imports

There are three libraries that you will need to include to make WebRTC operations much easier. The first thing you should include is [jQuery](https://jquery.com/) to make modifying DOM elements a breeze. Then, you will need the PubNub JavaScript SDK to facilitate the WebRTC signaling. Finally, include the PubNub WebRTC SDK which makes placing phone calls as simple as calling the `dial(number)` function.

```html
<script src="https://ajax.googleapis.com/ajax/libs/jquery/2.1.3/jquery.min.js"></script>
<script src="https://cdn.pubnub.com/pubnub.min.js"></script>
<script src="http://cdn.kevingleason.me/webrtc.js"></script>
```

We will include the YouTube API later. Now we are ready to write our calling functions for `login`, `makeCall`, and `end`!

### Part 3: Making and Receiving Calls

In order to start facilitating video calls, you will need a publish and subscribe key. To get your pub/sub keys, you’ll first need to [sign up for a PubNub account](http://www.pubnub.com/get-started/). Once you sign up, you can find your unique PubNub keys in the [PubNub Developer Dashboard](https://admin.pubnub.com). The free Sandbox tier should give you all the bandwidth you need to build and test your WebRTC Application.

```
var video_hold = document.getElementById("video-chat");
var video_out  = document.getElementById("vid-box");
var user_name = "";

function login(form) {
	user_name = form.username.value || "Anonymous";
	var phone = window.phone = PHONE({
	    number        : user_name, // listen on username line else Anonymous
	    publish_key   : 'your-pub-key', // Your Pub Key
	    subscribe_key : 'your-sub-key', // Your Sub Key
	    datachannels  : true,
	});	
	phone.ready(function(){form.username.style.background="#55ff5b"; form.login_submit.hidden="true"; });
	phone.receive(function(session){
	    session.connected(function(session) { video_hold.hidden=false; video_out.appendChild(session.video); });
	    session.ended(function(session) { video_out.innerHTML=''; });
	});
	// Configure DataChannel
	return false;
}
```

You can see we use the username as the phone's number, and instantiate PubNub using your own publish and subscribe keys. The next function `phone.ready` allows you to define a callback for when the phone is ready to place a call. I simply change the username input's background to green, but you can tailor this to your needs.

The `phone.receive` function allows you to define a callback that takes a session for when a session (call) event occurs, whether that be a new call, a call hangup, or for losing service, you attach those event handlers to the sessions in `phone.receive`. 

I defined `session.connected` which is called after receiving a call when you are ready to begin talking. I simple appended the session's video element to our video div. 

Then, I define `session.ended` which is called after invoking `phone.hangup`. This is where you place end-call logic. I simply clear the video holder's innerHTML.

We now have a phone ready to receive a call, so it is time to create a `makeCall` function.

```
function makeCall(form){
    if (!window.phone) alert("Login First!");
    else phone.dial(form.number.value);
    return false;
}
```

If `window.phone` is undefined, we cannot place a call. This will happen if the user did not log in first. If it is, we use the `phone.dial` function which takes a number and an optional list of servers to place a call.

Finally, to end a call or hangup, simply call the `phone.hangup` function and hide the video div.

```
function end(){
	if (!window.phone) return;
	window.phone.hangup();
	video_hold.hidden = true;
}
```

![Video Chat](img/basic_vid.png)

You should now have a working video chatting application! When you are ready we can move on and implement the DataChannels.

## Part 2: DataChannels and YouTube

All that's left to do is set up the YouTube API, and then use WebRTC DataChannels to synchronize playback, so lets get going!

### Step 1: The YouTube IFrame API

We will start by filling the `player` div we created in Part 1 with a YouTube iframe. The following code is all based off the [YouTube API documentation](https://developers.google.com/youtube/iframe_api_reference).

```
// This code loads the IFrame Player API code asynchronously. From YouTube API webpage.
var tag = document.createElement('script');
tag.src = "https://www.youtube.com/iframe_api";
var firstScriptTag = document.getElementsByTagName('script')[0];
firstScriptTag.parentNode.insertBefore(tag, firstScriptTag);

// This function creates an <iframe> (and YouTube player) after the API code downloads.
var player;
var vidId = "dQw4w9WgXcQ";  // The youtube Video ID for your starting video
function onYouTubeIframeAPIReady() {
	player = new YT.Player('player', {
	height: '390',
	width: '640',
	videoId: vidId, // Starting video ID
	events: {
		'onStateChange': onPlayerStateChange
		}
	});
}
```

This will look for a div with id `player` and place an iframe inside of it. You can change `vidId` to be the id of any YouTube video. `onPlayerStateChange` is a callback that we will have to implement shortly. 

### Step 2: DataChannels to Synchronize Playback

Currently, we can video chat, and we can watch YouTube. All we have left to do is synchronize all video playback. This includes play/pause, seeks, and what video is currently playing. We will accomplish all of this using the WebRTC `DataChannel` API. We need to define a few variables are used in the synchronization.

```
var done = false; // Variable to tell if the video is done or not
var seek = false; // Used to decide if we seeked elsewhere in the video.
var VID_CUE = 6;  // We define a VID_CUE event type. Used in msg.data when queueing a video.
```

We will use these variables to implement the `onPlayerStateChange` function from the previous step. The YouTube API has built in `PlayerStates` which are attached to the `onPlayerStateChange` event. Here is a quick overview of some events and when they are triggered:

- `YT.PlayerState.PLAYING` when the play button is clicked
- `YT.PlayerState.PAUSED` when the pause button is clicked
- `YT.PlayerState.BUFFERING` when the video is buffering (seeks)

The PubNub WebRTC SDK makes the DataChannel API extremely easy to use. To send data, use the `phone.sendData`. I love when it makes sense too.

```
// The API calls this function when the player's state changes. We send the event with a 
//  username attached through the WebRTC DataChannel
function onPlayerStateChange(event) {
	console.log(event);
	if (!window.phone) return;
	event.username = user_name;
	switch (event.data) {
		case YT.PlayerState.PLAYING:
			if (done) return;
			window.phone.sendData(event);
			break;
		case YT.PlayerState.PAUSED:
			window.phone.sendData(event);
			break;
		case YT.PlayerState.BUFFERING: // If they seeked, dont send this.
			if (seek) seek = false;
			else window.phone.sendData(event);
	}
}
```

__Part II Coming Soon__

[LiveDemo]:https://kevingleason.me/WebRTC-YouTubeTogether/yourtc.html
