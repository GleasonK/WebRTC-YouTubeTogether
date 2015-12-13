# WebRTC - YouTube Together

WebRTC is becoming widely adopted by large companies everywhere. If you don't know it already, WebRTC is a free, open source project that provides simple APIs for creating Real-Time Communications (RTC) for browsers and mobile devices. It powers many modern video chatting services, even Chromecast uses it to broadcast your chrome tab to the TV screen. It makes streaming any content such as video, audio, or arbitrary data simple and _fast_. Today we will be building a video chat application that allows you to watch YouTube with a friend!

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

```
&lt;div id=&quot;player&quot;&gt;&lt;/div&gt;
&lt;div style=&quot;float: left; width: 50%;&quot;&gt;
    &lt;div id=&quot;video-chat&quot; hidden=&quot;true&quot; style=&quot;margin-bottom: 10px;&quot;&gt;
		&lt;div id=&quot;vid-box&quot;&gt;&lt;/div&gt;
		&lt;button onclick=&quot;end()&quot;&gt;End Call&lt;/button&gt;
    &lt;/div&gt;
    
    &lt;form name=&quot;loginForm&quot; id=&quot;login&quot; action=&quot;#&quot; onsubmit=&quot;return login(this);&quot;&gt;
        	&lt;input type=&quot;text&quot; name=&quot;username&quot; id=&quot;username&quot; placeholder=&quot;Enter A Username&quot;/&gt;            
			&lt;input type=&quot;submit&quot; name=&quot;login_submit&quot; value=&quot;Log In&quot;&gt;
    &lt;/form&gt;

	&lt;form name=&quot;callForm&quot; id=&quot;call&quot; action=&quot;#&quot; onsubmit=&quot;return makeCall(this);&quot;&gt;
        &lt;input type=&quot;text&quot; name=&quot;number&quot; id=&quot;call&quot; placeholder=&quot;Enter User To Call!&quot;/&gt;
        &lt;input type=&quot;submit&quot; value=&quot;Call&quot;&gt;
	&lt;/form&gt;
	
	&lt;form name=&quot;cueForm&quot; id=&quot;cue&quot; action=&quot;#&quot; onsubmit=&quot;return cueFromURL(this);&quot;&gt;
        &lt;input type=&quot;text&quot; name=&quot;url&quot; id=&quot;url&quot; placeholder=&quot;Enter YouTube Video URL&quot;/&gt;
        &lt;input type=&quot;submit&quot; value=&quot;Queue Video&quot;&gt;
	&lt;/form&gt;
&lt;/div&gt;
```

This should leave you with a very basic HTML backbone that looks something like this:

![HTML Backbone](img/html.png)

The `player` div will house our YouTube video, while `vid-box` will hold the video chat. We will use the forms to log-in, place calls, and enqueue videos. 

### Step 2: The JavaScript Imports

There are three libraries that you will need to include to make WebRTC operations much easier. The first thing you should include is [jQuery](https://jquery.com/) to make modifying DOM elements a breeze. Then, you will need the PubNub JavaScript SDK to facilitate the WebRTC signaling. Finally, include the PubNub WebRTC SDK which makes placing phone calls as simple as calling the `dial(number)` function.

```
&lt;script src=&quot;https://ajax.googleapis.com/ajax/libs/jquery/2.1.3/jquery.min.js&quot;&gt;&lt;/script&gt;
&lt;script src=&quot;https://cdn.pubnub.com/pubnub.min.js&quot;&gt;&lt;/script&gt;
&lt;script src=&quot;http://cdn.kevingleason.me/webrtc.js&quot;&gt;&lt;/script&gt;
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

__Part II Coming Soon__

[LiveDemo]:https://kevingleason.me/WebRTC-YouTubeTogether/yourtc.html