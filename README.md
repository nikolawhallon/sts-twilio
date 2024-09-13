# sts-twilio

sts-twilio is a server which enables calls made to your Twilio phone number to pass through to Deepgram's speech-to-speech (STS) API,
enabling the caller to talk to a voice agent/bot. The code/architecture is very similar to what is discussed in this guide:

https://deepgram.com/learn/deepgram-twilio-streaming

Namely, we will be relying on Twilio's websocket streaming API to fork audio from the phone to a 3rd party service (in our case, Deepgram).
See the following for more information on the Twilio streaming API:

https://www.twilio.com/docs/voice/twiml/stream

There are a couple of key differences between this server (sts-twilio) and the guide just mentioned. They include:
* we will be streaming audio from the Twilio phone call to Deepgram's STS API, not Deepgram's ASR/STT API
* we will only stream mono, inbound audio to sts-twilio
* we have removed the concept of subscribers - i.e. we will not be streaming Deepgram STS results anywhere but the ongoing phonecall

These differences are actually all simplifications to the code, and show how clear it can be to use the Deepgram STS API
in your applications.

## Try It!

This is currently deployed somewhere in the ether, and can be called via +1(734)371-8307!

## Pre-requisites

You will need:
* A [Twilio account](https://www.twilio.com/try-twilio) with a Twilio number (the free tier will work).
* A Deepgram API Key - [get an API Key here](https://console.deepgram.com/signup?jump=keys).
* (_Optional_) [ngrok](https://ngrok.com/) to let Twilio access a local server.

You will also need to set up a TwiML Bin like the following in your Twilio Console:
```
<?xml version="1.0" encoding="UTF-8"?>
<Response>
    <Say language="en">"This call may be monitored or recorded."</Say>
    <Connect>
        <Stream url="wss://a127-75-172-116-97.ngrok-free.app/twilio" />
    </Connect>
</Response>
```
Where you should replace the url with wherever you decide to deploy sts-twilio (in the case of the above
TwiML Bin, I used ngrok to expose the server running locally, this is the recommended way for quick
development). This TwiML Bin must then be attached to one of your Twilio phone numbers so that it gets
executed whenever someone calls that number.

## Running the Server

If your TwiML Bin is setup correctly, you should be able to just run the server with:
```
python server.py
```
and then start making calls to the phone number the TwiML Bin is attached to!

## Code Tour

Let's dive into the code. First we have some import statements:
```
import asyncio
import base64
import json
import sys
import websockets
import ssl
```
Nothing shocking here. We are using `asyncio` and `websockets` to build an asynchronous websocket server, we will use base64 to handle
messages coming from and going to Twilio, which `base64` encodes its audio, we will use `json` to deal with parsing and sending
text messages from and to Twilio and the Deepgram STS API, etc.

The next block of code is:
```
def sts_connect():
    extra_headers = {"Authorization": "Token INSERT_DEEPGRAM_API_KEY"}
    sts_ws = websockets.connect(
        "wss://sts.sandbox.deepgram.com/agent", extra_headers=extra_headers
    )

    return sts_ws
```
This function will help us connect to the Deepgram STS API. Make sure to insert your Deepgram API Key where it says
"INSERT_DEEPGRAM_API_KEY" (and feel free to use environment variables, or another key management solution instead
of hard-coding the key). Note that no query parameters are specified - this is because, unlike Deepgram's ASR/STT API,
the Deepgram STS API does all of it's configuration via structured json websocket text messages - this can be helpful
as the number of parameters to tune the API can be quite large.

The next block of code is:
```
async def twilio_handler(twilio_ws):
    audio_queue = asyncio.Queue()
    streamsid_queue = asyncio.Queue()

    async with sts_connect() as sts_ws:
```
Here we set up the asynchronous function used to handle websocket messages from Twilio. We will be defining
asynchronous functions for dealing with messages received from Twilio, messages to send to Deepgram,
and messages received from Deepgram. Some of these asynchronous tasks will need information passed to them
from other tasks, so we have set up two queues, one to pass audio from Twilio, and one to pass the stream
sid (a unique identifier of the Twilio stream) from Twilio.

Finally, we open up a connection to the Deepgram STS API.

The next code block is:
```
        config_message = {
            "type": "SettingsConfiguration",
            "audio": {
                "input": {
                    "encoding": "mulaw",
                    "sample_rate": 8000,
                },
                "output": {
                    "encoding": "mulaw",
                    "sample_rate": 8000,
                    "container": "none",
                    "buffer_size": 250,
                },
            },
            "agent": {
                "listen": {"model": "nova-2"},
                "think": {
                    "provider": {
                        "type": "anthropic",  # examples are anthropic, open_ai, groq, ollama
                    },
                    "model": "claude-3-haiku-20240307",  # examples are claude-3-haiku-20240307, gpt-3.5-turbo, mixtral-8x7b-32768, mistral
                    "instructions": "You are a helpful car seller.",
                },
                "speak": {"model": "aura-asteria-en"},
            },
        }

        await sts_ws.send(json.dumps(config_message))
```
After opening the STS websocket connection, we immediately send it a text message with the configuration we want.
The most important thing to note here is the audio format we are using - 8000 Hz, raw, uncontainerized mulaw. This
is the format Twilio will be sending, and the format we will need to send back to Twilio (plus some base64 encoding/decoing).

The next code block is:
```
        async def sts_sender(sts_ws):
            print("sts_sender started")
            while True:
                chunk = await audio_queue.get()
                await sts_ws.send(chunk)
```
This `sts_sender` simply waits for audio from Twilio (via the audio queue) and forwards it to the Deepgram STS API. Simple!

The next code block is:
```
        async def sts_receiver(sts_ws):
            print("sts_receiver started")
            # we will wait until the twilio ws connection figures out the streamsid
            streamsid = await streamsid_queue.get()
            # for each sts result received, forward it on to the call
            async for message in sts_ws:
                if type(message) is str:
                    print(message)
                    # handle barge-in
                    decoded = json.loads(message)
                    if decoded['type'] == 'UserStartedSpeaking':
                        clear_message = {
                            "event": "clear",
                            "streamSid": streamsid
                        }
                        await twilio_ws.send(json.dumps(clear_message))

                    continue

                print(type(message))
                raw_mulaw = message

                # construct a Twilio media message with the raw mulaw (see https://www.twilio.com/docs/voice/twiml/stream#websocket-messages---to-twilio)
                media_message = {
                    "event": "media",
                    "streamSid": streamsid,
                    "media": {"payload": base64.b64encode(raw_mulaw).decode("ascii")},
                }

                # send the TTS audio to the attached phonecall
                await twilio_ws.send(json.dumps(media_message))
```
This `sts_receiver` first waits until it has received a stream sid from Twilio, and then loops over messages
received from the Deepgram STS API. If we receive a text message, we see if it indicated that the user has started
speaking - if it has, we treat this as barge-in and have Twilio clear the agent audio on the call (we need that
stream sid for this!).

Other messages should be binary messages containing the text-to-speech (TTS) output of the Deepgram STS API.
These we pack up into valid Twilio messages (we also need that stream sid here!), and shoot
them off to Twilio to be played back on the phone for the caller to here. For more information
about streaming audio to Twilio, see the following:

https://www.twilio.com/docs/voice/twiml/stream#websocket-messages---to-twilio

The next code block is:
```
        async def twilio_receiver(twilio_ws):
            print("twilio_receiver started")
            # twilio sends audio data as 160 byte messages containing 20ms of audio each
            # we will buffer 20 twilio messages corresponding to 0.4 seconds of audio to improve throughput performance
            BUFFER_SIZE = 20 * 160

            inbuffer = bytearray(b"")
            async for message in twilio_ws:
                try:
                    data = json.loads(message)
                    if data["event"] == "start":
                        print("got our streamsid")
                        start = data["start"]
                        streamsid = start["streamSid"]
                        streamsid_queue.put_nowait(streamsid)
                    if data["event"] == "connected":
                        continue
                    if data["event"] == "media":
                        media = data["media"]
                        chunk = base64.b64decode(media["payload"])
                        if media["track"] == "inbound":
                            inbuffer.extend(chunk)
                    if data["event"] == "stop":
                        break

                    # check if our buffer is ready to send to our audio_queue (and, thus, then to sts)
                    while len(inbuffer) >= BUFFER_SIZE:
                        chunk = inbuffer[:BUFFER_SIZE]
                        audio_queue.put_nowait(chunk)
                        inbuffer = inbuffer[BUFFER_SIZE:]
                except:
                    break
```
This `twilio_receiver` loops over messages Twilio is sending our server. If we receive a "start" message,
we can extract the stream sid, and send it to our other async task which needs it. If we receive a "media"
message, we decode the audio from it, append it to a running buffer, and send it to the async task which
forwards it to Deepgram when it's of a reasonable size (there can be throughput issues when sending tons
of tiny chunks, so that's why we are doing this buffering approach).

The next code block is:
```
        # the async for loop will end if the ws connection from twilio dies
        # and if this happens, we should forward an some kind of message to sts
        # to signal sts to send back remaining messages before closing(?)
        # audio_queue.put_nowait(b'')

        await asyncio.wait(
            [
                asyncio.ensure_future(sts_sender(sts_ws)),
                asyncio.ensure_future(sts_receiver(sts_ws)),
                asyncio.ensure_future(twilio_receiver(twilio_ws)),
            ]
        )

        await twilio_ws.close()
```
This just runs the above asynchronous tasks, nothing special.

And finally the last code block is:
```
async def router(websocket, path):
    if path == "/twilio":
        print("twilio connection incoming")
        await twilio_handler(websocket)


def main():
    # use this if using ssl
    # ssl_context = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER)
    # ssl_context.load_cert_chain('cert.pem', 'key.pem')
    # server = websockets.serve(router, '0.0.0.0', 443, ssl=ssl_context)

    # use this if not using ssl
    server = websockets.serve(router, "localhost", 5000)

    asyncio.get_event_loop().run_until_complete(server)
    asyncio.get_event_loop().run_forever()


if __name__ == "__main__":
    sys.exit(main() or 0)
```
This just sets up and runs the server, making sure all incoming websocket connections get handled by `twilio_handler` which we've just gone over.
