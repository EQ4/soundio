# Soundio

Soundio provides a fast, declarative way to set up an audio graph, an API for
manipulating it and a JSONify-able structure that can be used to store patches.

A Soundio document is a <code>JSON</code>ify-able object that looks like this:

	var data = {
		objects: [{
			id: 0,
			type: "input"
		}, {
			id: 1,
			type: "flanger",
			frequency: 0.33,
			feedback: 0.9,
			delay: 0.16
		}, {
			id: 2,
			type: "output"
		}],
		
		connections: [{
			source: 0,
			destination: 1
		}, {
			source: 1,
			destination: 2
		}]
	};

It contains two arrays, <code>objects</code> and <code>connections</code>.
<code>objects</code> defines the
<a href="http://github.com/soundio/audio-object">audio objects</a> in the graph.
<code>connections</code> defines the connections between them.

Call Soundio with this data to get a live audio graph:

	var soundio = Soundio(data);

Turn your volume down a bit, enable the mic when prompted by the browser, and
you will hear your voice being flanged.

Soundio can also take an options object. There is currently one option. If there
is already an audio context on the page, pass it in to avoid creating anew one:

	var soundio = Soundio(data, {
			audio: AudioContext
		});

The returned object, <code>soundio</code>, can be thought of as a GOM – a Graph
Object Model for audio objects and connections. The GOM controls the Web Audio
API.

## Dependencies

Soundio is in development. It has three dependencies that are in separate repos:

- <a href="https://github.com/cruncher/collection">github.com/cruncher/collection</a>
- <a href="https://github.com/soundio/audio-object">github.com/soundio/audio-object</a>
- <a href="https://github.com/soundio/midi (optional)">github.com/soundio/midi</a> (optional)

## soundio

### soundio.create(data)

Create objects from data. This function is called by the Soundio constructor
when initially setting up the object graph, and can be used to add objects and
connections.

### soundio.clear()

Remove and destroy all objects and connections.

### soundio.destroy()

Removes and destroys all objects and connections, disconnects any media
inputs from soundio's input, and disconnects soundio's output from audio
destination.

### soundio.objects

A collecton of <a href="http://github.com/soundio/audio-object">audio objects</a>.
An audio object controls a graph of audio nodes. In soundio, audio objects have
an <code>id</code> and a <code>type</code>. <code>name</code> is optional. Other
properties depend on the type.

	var flanger = soundio.objects.find(1);
	
	// {
	//     id: 7,
	//     type: "flange",
	//     frequency: 256
	// }

Changes to <code>flanger.frequency</code> are reflected immediately in the
Web Audio graph.

	flanger.frequency = 480;
	
	// flanger.automate(name, value, time, curve)
	flanger.automate('frequency', 2400, audio.currentTime + 0.8, 'exponential');

For more about audio objects see
<a href="http://github.com/soundio/audio-object">github.com/soundio/audio-object</a>.

#### soundio.objects.create(type, settings)

Create an audio object.

<code>type</code> is a string:

- "input"
- "output"
- "compress"
- "filter"
- "flange"
- "loop"
- "saturate"
- "send"

<code>settings</code> depend on the type of audio object being created.

Returns the created audio object. Created objects can also be found in
<code>soundio.objects</code>, as well as in <code>soundio.inputs</code> and
<code>soundio.outputs</code> if they are of type <code>"input"</code> or
<code>"output"</code> respectively.

#### soundio.objects.delete(object || id)

Destroy an audio object in the graph. Both the object and any connections to or
from the object are destroyed.

#### soundio.objects.find(id || query)
#### soundio.objects.query(query)

### soundio.connections

A collection of connections between the audio objects in the graph. A connection
has a <code>source</code> and a <code>destination</code> that point to
<code>id</code>s of objects in <code>soundio.objects</code>:

	{
		source: 7,
		destination: 12
	}

In addition a connection can define a named output node on the source object and
a named input node on the destination object:

	{
		source: 7,
		output: "send",
		destination: 12,
		input: "default"
	}


#### soundio.connections.create(data)

Connect two objects. <code>data</code> must have <code>source</code> and
<code>destination</code> defined. Naming an <code>output</code> or
<code>input</code> is optional. They will default to <code>"default"</code>.

    soundio.connections.create({
        source: 7,
        output: "send",
        destination: 12
    });


#### soundio.connections.delete(query)

Removes all connections whose properties are equal to the properties defined in
the <code>query</code> object. For example, disconnect all connections to
object with id <code>3</code>:

    soundio.connections.query({ destination: 3 });


#### soundio.connections.query(query)

Returns an array of all objects in <code>connections</code> whose properties
are equal to the properties defined in the <code>query</code> object. For
example, get all connections from object with id <code>6</code>:

    soundio.connections.query({ source: 6 });




### soundio.midi

A collection of MIDI routes that make object properties controllable via
incoming MIDI events. A midi route looks like this:

    {
        message:   [191, 0],
        object:    AudioObject,
        property:  "gain",
        transform: "linear",
        min:       0,
        max:       1
    }


#### soundio.midi.create(data)

Create a MIDI route from data:

    soundio.midi.create({
        message:   [191, 0],
        object:    1,
        property:  "gain",
        transform: "cubic",
        min:       0,
        max:       2
    });

The properties <code>transform</code>, <code>min</code> and <code>max</code> are
optional. They default to different values depending on the type of the object.


#### soundio.midi.delete(query)

Removes all MIDI routes whose properties are equal to the properties defined in
the <code>query</code> object. For example, disconnect all routes to gain
properties:

    soundio.midi.query({ property: "gain" });


#### soundio.midi.query(query)

Returns an array of all objects in <code>soundio.midi</code> whose properties
are equal to the properties defined in the <code>query</code> object. For
example, get all connections from object with id <code>6</code>:

    soundio.connections.query({ object: 6 });


## Soundio

### Soundio.register(type, function)

Register an audio object factory function for making audio objects of type
<code>type</code>.

	Soundio.register('gain', function(audio, settings) {
		var gain = audio.createGain();

		gain.gain.value = settings.gain;

		return AudioObject(audio, gain, gain, {
			gain: gain
		});
	});

	var soundio = Soundio();

	soundio.create("gain", {
		gain: 0.25
	});

Soundio comes with the following audio object factories registered:

    // Single node audio objects 
    'biquad filter'
    'compressor'
    'convolver'
    'delay'
    'oscillator'
    'waveshaper'

    // Multi node audio objects
    'compress'
    'flange'
    'loop'
    'filter'
    'saturate'
    'send'

### Soundio.isAudioParam(object)

### Soundio.isDefined(value)

Returns <code>true</code> where <code>value</code> is not <code>undefined</code>
or <code>null</code>.
