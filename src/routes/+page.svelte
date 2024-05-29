<script>
    import * as Tone from "tone";

    let allowAudio = false;
    let showLogo = true;
    let sequence = 0;
    let canProgress = true;
    let sequenceDescription = "Waiting for next sequence.";
    let bassLayer1, bassLayer2, bassLayer3;

    //for using spacebar to progress sequence
    function spaceToProgressSequence() {
        function spacebar(event) {
            if (event.code === "Space" && canProgress) {
                progressSequence();
            }
        }

        window.addEventListener("keydown", spacebar);

        return () => {
            window.removeEventListener("keydown", spacebar);
        };
    }

    //for using MIDI pedal to progress sequence
    function handleMIDIMessage(message) {
        let command = message.data[0];
        let note = message.data[1];
        let velocity = message.data.length > 2 ? message.data[2] : 0;

        switch (command) {
            case 144:
                if (velocity > 0) {
                    if (note === 64 && canProgress) {
                        progressSequence();
                    }
                }
                break;
        }
    }

    //for using browser's MIDI support
    function MIDIToProgressSequence() {
        if (navigator.requestMIDIAccess) {
            navigator.requestMIDIAccess().then(onMIDISuccess, onMIDIFailure);
        } else {
            console.log("No MIDI support in your browser.");
        }

        function onMIDISuccess(midiAccess) {
            for (let input of midiAccess.inputs.values()) {
                input.onmidimessage = handleMIDIMessage;
            }
            MIDIon = true;
        }

        function onMIDIFailure() {
            console.log("Could not access your MIDI devices.");
        }
    }

    //when allow audio is toggled on, start Tone.js and set up spacebar and MIDI event listeners
    $: if (allowAudio) {
        startTone();
        spaceToProgressSequence();
        MIDIToProgressSequence();
    }

    //start Tone.js and set up bass layers
    async function startTone() {
        await Tone.start();
        //two bass layers are necessary for seamless looping in my case
        //each layer has a sounds array, a gain node, and a timeout for looping
        bassLayer1 = {
            sounds: [new Tone.Player("/static/bass-layer-1.mp3"), new Tone.Player("/static/bass-layer-1.mp3")],
            gain: new Tone.Gain().toDestination(),
            timeout: null,
        };
        bassLayer2 = {
            sounds: [new Tone.Player("/static/bass-layer-2.mp3"), new Tone.Player("/static/bass-layer-2.mp3")],
            gain: new Tone.Gain().toDestination(),
            timeout: null,
        };
        bassLayer3 = {
            sounds: [new Tone.Player("/static/bass-layer-3.mp3"), new Tone.Player("/static/bass-layer-3.mp3")],
            gain: new Tone.Gain().toDestination(),
            timeout: null,
        };
    }

    //converts the number of samples reported by the buffer to Ms
    function samplesToMs(player) {
        return (player.buffer.length / player.buffer.sampleRate) * 1000;
    }

    //random number generator for inclusive range
    function randomNumberInclusive(min, max) {
        return Math.floor(Math.random() * (max - min + 1) + min);
    }

    //for looping the first bass layer
    //the same logic is used for the other bass layers
    function loopBassLayer1(percentage) {
        //sets initial gain
        bassLayer1.gain.gain.value = 0.25;

        //connects the gain node to each bass layer and sets an initial fade in and fade out
        for (const player of bassLayer1.sounds) {
            player.connect(bassLayer1.gain);
            player.fadeIn = 10;
            player.fadeOut = 2;
        }

        //calculates the duration of the loop based on the percentage because there's no event listener for when time updates
        let duration = samplesToMs(bassLayer1.sounds[0]) * (percentage / 100);
        //counter used for tracking 100ms intervals
        let counter = 0;

        //if the duration has been surpassed, clear the interval and start the second sound
        bassLayer1.timeout = setInterval(() => {
            counter += 100;
            if (counter >= duration) {
                clearInterval(bassLayer1.timeout);
                bassLayer1.sounds[1].start();
            }
        }, 100);

        //event listener for when sounds stop
        //resets the fade because I no longer want a 10 second fade in for loops
        //starts the sound again
        bassLayer1.sounds[0].onstop = () => {
            bassLayer1.sounds[0].fadeIn = 2;
            bassLayer1.sounds[0].start();
        };

        bassLayer1.sounds[1].onstop = () => {
            bassLayer1.sounds[1].fadeIn = 2;
            bassLayer1.sounds[1].start();
        };

        //when the function is called for the first time, it will start the first sound
        if (bassLayer1.sounds[0].state === "stopped" && bassLayer1.sounds[1].state === "stopped") {
            bassLayer1.sounds[0].start();
        }
    }

    function loopBassLayer2(percentage) {
        bassLayer2.gain.gain.value = 0.35;

        for (const player of bassLayer2.sounds) {
            player.connect(bassLayer2.gain);
            player.fadeIn = 10;
            player.fadeOut = 2;
        }

        let duration = samplesToMs(bassLayer2.sounds[0]) * (percentage / 100);
        let counter = 0;

        bassLayer2.timeout = setInterval(() => {
            counter += 100;
            if (counter >= duration) {
                clearInterval(bassLayer2.timeout);
                bassLayer2.sounds[1].start();
            }
        }, 100);

        bassLayer2.sounds[0].onstop = () => {
            bassLayer2.sounds[0].fadeIn = 0;
            bassLayer2.sounds[0].start();
        };

        bassLayer2.sounds[1].onstop = () => {
            bassLayer2.sounds[1].fadeIn = 0;
            bassLayer2.sounds[1].start();
        };

        if (bassLayer2.sounds[0].state === "stopped" && bassLayer2.sounds[1].state === "stopped") {
            bassLayer2.sounds[0].start();
        }
    }

    function loopBassLayer3(percentage) {
        bassLayer3.gain.gain.value = 0.05;

        for (const player of bassLayer3.sounds) {
            player.connect(bassLayer3.gain);
            player.fadeIn = 10;
            player.fadeOut = 2;
            player.playbackRate = Math.random() * 0.2 + 0.9;
        }

        let duration = samplesToMs(bassLayer3.sounds[0]) * (percentage / 100);
        let counter = 0;

        bassLayer3.timeout = setInterval(() => {
            counter += 100;
            if (counter >= duration) {
                clearInterval(bassLayer3.timeout);
                bassLayer3.sounds[1].start();
            }
        }, 100);

        bassLayer3.sounds[0].onstop = () => {
            bassLayer3.sounds[0].fadeIn = 2;
            bassLayer3.sounds[0].playbackRate = Math.random() * 0.2 + 0.9;
            bassLayer3.sounds[0].start();
        };

        bassLayer3.sounds[1].onstop = () => {
            bassLayer3.sounds[1].fadeIn = 2;
            bassLayer3.sounds[1].playbackRate = Math.random() * 0.2 + 0.9;
            bassLayer3.sounds[1].start();
        };

        if (bassLayer3.sounds[0].state === "stopped" && bassLayer3.sounds[1].state === "stopped") {
            bassLayer3.sounds[0].start();
        }
    }

    //for making steps. Same logic applies to drips
    let stepTimeout;
    function stepMaker(volume, taps, min, max, dmin, dmax) {
        //plays a sound when the function is called for the first time, or immediately after the timeout is cleared and reassigned
        //this is unique to steps, but could be applied to drips
        if (!stepTimeout) delayMaker(volume, randomNumberInclusive(4, 6), "/static/bass-hit.mp3", dmin, dmax);
        //sets a timeout for the sounds to recur
        stepTimeout = setTimeout(
            () => {
                delayMaker(volume, taps, "/static/bass-hit.mp3", dmin, dmax);
                stepMaker(volume, taps, min, max, dmin, dmax);
            },
            randomNumberInclusive(min, max)
        );
    }

    let dripTimeout;
    function dripMaker(volume, taps, min, max, dmin, dmax) {
        dripTimeout = setTimeout(
            () => {
                delayMaker(volume, taps, "/static/drip.mp3", dmin, dmax);
                dripMaker(volume, taps, min, max, dmin, dmax);
            },
            randomNumberInclusive(min, max)
        );
    }

    //for making wind sounds
    let windTimeout;
    function windMaker(volume, rate) {
        //creates a gain node, panner, and player
        let gain = new Tone.Gain(volume).toDestination();
        let panner = new Tone.Panner(randomNumberInclusive(-1, 1)).connect(gain);
        let wind = new Tone.Player("/static/wind.mp3").connect(panner);
        //sets the fade in and fade out times and autostarts the sound
        //autostart is necessary because we don't know when the player will be loaded
        wind.fadeIn = 2;
        wind.fadeOut = 2;
        wind.autostart = true;
        //when the sound stops, destroy the nodes
        //I don't know if this is necessary, but it seems intuitive
        wind.onstop = () => {
            wind.dispose();
            gain.dispose();
            panner.dispose();
        };

        //sets the winds to recur
        windTimeout = setTimeout(() => {
            windMaker(volume, rate);
        }, rate);
    }

    //for making unique tap delays
    function delayMaker(volume, taps, path, min, max) {
        //creates a gain node, panner, and players
        //players is used because I want to play the same sound multiple times rather than making mutilple individual players
        let gain = new Tone.Gain(volume).toDestination();
        let panner = new Tone.Panner(randomNumberInclusive(-1, 1)).connect(gain);
        let players = new Tone.Players().connect(panner);
        //adds the players to the players object for each tap
        for (let i = 0; i < taps; i++) {
            players.add(i, path);
        }
        //if the players are loaded, clear the interval and start the taps
        let checkLoaded = setInterval(() => {
            if (players.loaded) {
                clearInterval(checkLoaded);
                //for each tap, set a random time and pan, then start the player at a delay with is a random time multiplied by the tap number
                for (let i = 0; i < taps; i++) {
                    let randomTime = randomNumberInclusive(min, max);
                    //starting pan is hard left or right
                    let randomPan = Math.random() * 2 - 1;
                    panner.pan.value = randomPan;
                    //ramp the pan to the opposite of the random pan over the random time
                    panner.pan.rampTo(-randomPan, (randomTime / 1000) * taps);
                    let delayTimeout = setTimeout(() => {
                        //this volume solution isn't perfect, but it scales the volume based on the tap number
                        players.player(i).volume.value = 1 * (-i * 2);
                        players.player(i).start();
                        //when the player stops, dispose of it
                        //if its the last tap, clear the timeout and dispose of the nodes
                        players.player(i).onstop = () => {
                            players.player(i).dispose();
                            if (i === taps - 1) {
                                clearTimeout(delayTimeout);
                                gain.dispose();
                                panner.dispose();
                                players.dispose();
                            }
                        };
                    }, i * randomTime);
                }
            }
        }, 20);
    }

    //logic for each sequence
    function progressSequence() {
        sequence++;
        switch (sequence) {
            case 1:
                //hides the home URL
                showLogo = false;
                //start looping bassLayer1 and bassLayer2
                loopBassLayer1(50);
                loopBassLayer2(50);
                //start looping drips with delay
                dripMaker(0.025, randomNumberInclusive(3, 5), 10000, 15000, 700, 1000);
                //makes progressing the sequence impossible during the countdown
                canProgress = false;
                //sets the sequence description for the player
                sequenceDescription = "Listen for the first droplet 💧 sound.";
                //a timer for waiting between sequences
                waitTimer(10);
                //after the timer, allow the player to progress the sequence
                setTimeout(() => {
                    canProgress = true;
                    sequenceDescription = "Waiting for next sequence.";
                }, 10000);
                break;
            case 2:
                //a bass hit with a delay on sequence trigger
                delayMaker(0.85, randomNumberInclusive(4, 6), "/static/bass-hit.mp3", 1100, 1250);
                //start looping bassLayer3
                loopBassLayer3(50);
                canProgress = false;
                sequenceDescription = "Wait for high synth to fade in entirely 🎹.";
                waitTimer(10);
                setTimeout(() => {
                    canProgress = true;
                    sequenceDescription = "Waiting for next sequence.";
                }, 10000);
                break;
            case 3:
                delayMaker(0.85, randomNumberInclusive(4, 6), "/static/bass-hit.mp3", 1100, 1250);
                canProgress = false;
                sequenceDescription = "Wait about 5 seconds.";
                waitTimer(5);
                setTimeout(() => {
                    canProgress = true;
                    sequenceDescription = "Waiting for next sequence.";
                }, 5000);
                break;
            case 4:
                //fade out all bass layers
                bassLayer1.gain.gain.rampTo(0, 10);
                bassLayer2.gain.gain.rampTo(0, 10);
                bassLayer3.gain.gain.rampTo(0, 10);
                canProgress = false;
                sequenceDescription = "Wait for the first bass drum sound 🥁.";
                waitTimer(10);
                setTimeout(() => {
                    delayMaker(0.85, randomNumberInclusive(4, 6), "/static/bass-hit.mp3", 1100, 1250);
                    canProgress = true;
                    sequenceDescription = "Waiting for next sequence.";
                }, 10000);
                break;
            case 5:
                //sets up steps to recur
                stepMaker(0.85, randomNumberInclusive(3, 4), 5000, 6000, 1100, 1200);
                canProgress = false;
                sequenceDescription = "Wait about 5 seconds.";
                waitTimer(5);
                setTimeout(() => {
                    canProgress = true;
                    sequenceDescription = "Waiting for next sequence.";
                }, 5000);
                break;
            case 6:
                //fade in all bass layers
                bassLayer1.gain.gain.rampTo(0.25, 10);
                bassLayer2.gain.gain.rampTo(0.35, 10);
                bassLayer3.gain.gain.rampTo(0.05, 10);
                //reset the step timeour and set up new steps
                clearTimeout(stepTimeout);
                stepTimeout = null;
                stepMaker(0.85, randomNumberInclusive(5, 6), 3000, 4000, 700, 1000);
                canProgress = false;
                sequenceDescription = "Wait for all synths to fade in entirely 🎹.";
                waitTimer(10);
                setTimeout(() => {
                    canProgress = true;
                    sequenceDescription = "Waiting for next sequence.";
                }, 10000);
                break;
            case 7:
                bassLayer1.gain.gain.rampTo(0, 10);
                bassLayer2.gain.gain.rampTo(0, 10);
                bassLayer3.gain.gain.rampTo(0, 10);
                //add recurring wind sounds
                windMaker(0.35, randomNumberInclusive(2000, 10000));
                canProgress = false;
                sequenceDescription = "Wait for only bass drum and wind 🥁🌬️.";
                waitTimer(10);
                setTimeout(() => {
                    canProgress = true;
                    sequenceDescription = "Waiting for next sequence.";
                }, 10000);
                break;
            case 8:
                bassLayer1.gain.gain.rampTo(0.35, 10);
                bassLayer2.gain.gain.rampTo(0.45, 10);
                bassLayer3.gain.gain.rampTo(0.05, 10);
                clearTimeout(stepTimeout);
                stepTimeout = null;
                stepMaker(0.95, randomNumberInclusive(6, 7), 2000, 3000, 500, 600);
                canProgress = false;
                sequenceDescription = "Wait for all synths to fade in entirely 🎹.";
                waitTimer(10);
                setTimeout(() => {
                    canProgress = true;
                    sequenceDescription = "Waiting for next sequence.";
                }, 10000);
                break;
            case 9:
                bassLayer1.gain.gain.rampTo(0, 30);
                bassLayer2.gain.gain.rampTo(0, 30);
                bassLayer3.gain.gain.rampTo(0, 30);
                canProgress = false;
                sequenceDescription = "Wait about 5 seconds.";
                waitTimer(5);
                setTimeout(() => {
                    sequenceDescription = "Everything will fade out in about 30 seconds.";
                    waitTimer(30);
                }, 5000);
                setTimeout(() => {
                    //clear all recurring sound timeouts
                    clearTimeout(dripTimeout);
                    dripTimeout = null;
                    clearTimeout(windTimeout);
                    windTimeout = null;
                    clearTimeout(stepTimeout);
                    stepTimeout = null;
                    stepMaker(0.85, randomNumberInclusive(3, 4), 4000, 5000, 1000, 1100);
                }, 15000);
                setTimeout(() => {
                    clearTimeout(stepTimeout);
                    stepTimeout = null;
                    sequenceDescription = "Thanks for playing! 🎉";
                    //show homepage logo
                    showLogo = true;
                }, 30000);
                break;
            default:
                break;
        }
    }

    //for waiting between sequences
    let waitCounter;
    function waitTimer(time) {
        //clears the previous waitCounter
        if (waitCounter) clearInterval(waitCounter);

        waitCounter = time;

        //decrements the waitCounter every second
        let interval = setInterval(() => {
            waitCounter--;
            if (waitCounter <= 0) {
                clearInterval(interval);
            }
        }, 1000);
    }

    //for setting up the mic
    let micOpen, mic, micGain, micSplit, micMerge, micDelay1, micDelay2, micPanner1, micPanner2, micReverb;
    function setMic() {
        //if the mic is open, close it and dispose of the nodes
        if (micOpen) {
            mic.close();
            mic.dispose();
            micGain.dispose();
            micSplit.dispose();
            micMerge.dispose();
            micDelay1.dispose();
            micDelay2.dispose();
            micPanner1.dispose();
            micPanner2.dispose();
            micOpen = false;
            //if the mic is closed, open it and set up the nodes
            //mic info
            //the gain is set to 1, all adjusts have to happen on the mic or the interface
            //the reverb is a convolution reverb with a sample of the Greek Theater at Mills College
            //the split is necessary because the mic input is stereo and I want delays to be panned in stereo
            //the merge is necessary because the delays are panned in stereo and I want to merge them back to mono. This is also necessary to include live mic input into the amplified signal
        } else {
            micGain = new Tone.Gain(1).toDestination();
            micReverb = new Tone.Convolver("/static/MillsGreekTheater.wav").connect(micGain);
            micSplit = new Tone.Split();
            micMerge = new Tone.Merge().connect(micReverb);
            micDelay1 = new Tone.FeedbackDelay(0.375, 0.3).connect(micMerge, 0, 0);
            micDelay2 = new Tone.FeedbackDelay(0.5, 0.3).connect(micMerge, 0, 1);
            micPanner1 = new Tone.Panner(-0.5).connect(micDelay1);
            micPanner2 = new Tone.Panner(0.5).connect(micDelay2);
            mic = new Tone.UserMedia().connect(micSplit);
            mic.connect(micMerge, 0, 0);
            mic.connect(micMerge, 0, 1);
            micSplit.connect(micPanner1, 0, 0);
            micSplit.connect(micPanner2, 0, 0);
            mic.open();
            micOpen = true;
        }
    }

    //for progessing the sequence with a MIDI pedal
    let MIDIon = false;
    function setMIDI() {
        MIDIon = true;
        MIDIToProgressSequence();
    }
</script>

<div class="min-vh-100 bg-dark d-flex flex-column gap-3 justify-content-center align-items-center">
    {#if showLogo}
        <a class="position-absolute top-0 start-0 m-3 fs-4 text-light" href="https://forrestbalman.com">Forrest Balman</a>
    {/if}
    {#if !allowAudio}
        <h1 class="display-2 text-light text-center">Have you ever wondered why the <span class="mole-container"><img class="svg-size mole" src="/static/mole.svg" alt="Mole icon" /></span> people march?</h1>
        <button
            class="sequence-btn rounded-1 border-0 text-light"
            on:click={() => {
                allowAudio = true;
            }}>Start</button
        >
    {:else}
        <div class="d-flex flex-column align-items-center gap-2">
            {#if sequenceDescription}
                <p class="lead text-light">{sequenceDescription}</p>
            {/if}
            <h4 class="text-light">Current sequence</h4>
            <h1 class="sequence-counter text-light text-center">
                {#if sequence === 0}
                    Waiting
                {:else}
                    {sequence}
                {/if}
            </h1>
        </div>
        <button class="sequence-btn text-light border-0 rounded-1" disabled={!canProgress ? true : false} on:click={progressSequence}>
            {#if sequence === 0}
                Start
            {:else if !canProgress}
                {waitCounter}
            {:else}
                Next
            {/if}
        </button>
        <div class="d-flex flex-column position-absolute bottom-0 end-0 m-3">
            <button class="btn btn-sm btn-primary text-light border-0 rounded-1 mb-3" on:click={setMic}>Set Mic</button>
            <p class="text-light text-center">
                {#if micOpen}
                    Mic is open
                {:else}
                    Mic is closed
                {/if}
            </p>
        </div>
        <div class="d-flex flex-column position-absolute bottom-0 start-0 m-3">
            <button class="btn btn-sm btn-success text-light border-0 rounded-1 mb-3" on:click={setMIDI}>Set Pedal</button>
            <p class="text-light text-center">
                {#if MIDIon}
                    Pedal is on
                {:else}
                    Pedal is off
                {/if}
            </p>
        </div>
    {/if}
</div>

<style>
    h1,
    h4,
    p,
    a,
    button {
        font-family: "Quicksand", sans-serif;
    }

    .svg-size {
        width: 4.5rem;
        height: 4.5rem;
    }

    .sequence-btn {
        font-size: 2rem;
        padding: 1rem 2rem;
        background-color: #a0876b;
    }

    .sequence-btn:disabled {
        background-color: #6c757d;
    }

    .sequence-counter {
        font-size: 5rem;
    }
</style>
