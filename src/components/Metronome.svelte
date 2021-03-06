<script>
    import { BufferLoader } from "../helpers/BufferLoader.svelte";
    import { playFile } from "../helpers/PlayFile.svelte";
    import { audioCtx } from "../store.js";
    import { onDestroy, onMount } from "svelte";

    console.group("metronome");

    let vol = 50; //start volume at halfway
    let play = false;
    let showMetronome = false;

    //speed and number of beats
    let bpm = 60;
    let numOfBeats = 4;

    var startTime; // The start time of the entire sequence.
    var current16thNote; // What note is currently last scheduled?
    var lookahead = 5.0; // How frequently to call scheduling function
    //(in milliseconds)
    var scheduleAheadTime = 0.1; // How far ahead to schedule audio (sec)
    // This is calculated from lookahead, and overlaps
    // with next interval (in case the timer is late)
    var nextNoteTime = 0.0; // when the next note is due.
    var noteResolution = 2; // 0 == 16th, 1 == 8th, 2 == quarter note
    var notesInQueue = []; // the notes that have been put into the web audio,
    // and may or may not have played yet. {note, time}
    var timerWorker = null; // The Web Worker used to fire timer messages
    var unlocked = false;
    let bufferLoader;

    onMount(() => {
        audioCtx.set(new (window.AudioContext || window.webkitAudioContext)());

        // use BufferLoader to load tick audio files
        bufferLoader = new BufferLoader($audioCtx, [
            "./audio/high.wav",
            "./audio/middle.wav",
            "./audio/low.wav",
        ]);

        bufferLoader.load();

        //create timerWorker to handle setInterval calls
        timerWorker = new Worker("./metronomeworker.js");
        timerWorker.onmessage = function (e) {
            if (e.data == "tick") {
                scheduler();
            }
        };
        timerWorker.postMessage({ interval: lookahead });
    });

    onDestroy(() => {
        if (play) play = false;
    });

    $: playHandler(play);

    function nextNote() {
        // Advance current note and time by a 16th note...
        let secondsPerBeat = 60.0 / bpm; // Notice this picks up the CURRENT
        // bpm value to calculate beat length.
        nextNoteTime += 0.25 * secondsPerBeat; // Add beat length to last beat time

        current16thNote++; // Advance the beat number, wrap to zero
        if (current16thNote == numOfBeats * 4) {
            current16thNote = 0;
        }
    }

    function scheduleNote(beatNumber, time) {
        // push the note on the queue, even if we're not playing. for drawing
        notesInQueue.push({ note: beatNumber, time: time });

        if (noteResolution == 1 && beatNumber % 2) return; // we're not playing non-8th 16th notes
        if (noteResolution == 2 && beatNumber % 4) return; // we're not playing non-quarter 8th notes

        if (beatNumber % 16 === 0) {
            //on one
            playFile($audioCtx, bufferLoader, "high", vol);
        } else if (!beatNumber % 16 === 0 && beatNumber % 4 === 0) {
            //on downbeats other than one
            playFile($audioCtx, bufferLoader, "middle", vol);
        } else if (!beatNumber % 16 === 0) {
            //on eighth upbeats and any sixteenth
            playFile($audioCtx, bufferLoader, "low", vol);
        }
    }

    function scheduler() {
        // while there are notes that will need to play before the next interval,
        // schedule them and advance the pointer.
        while (nextNoteTime < $audioCtx.currentTime + scheduleAheadTime) {
            scheduleNote(current16thNote, nextNoteTime);
            nextNote();
        }
    }
    function playHandler(play) {
        if ($audioCtx) {
            if (!unlocked) {
                // play silent buffer to unlock the audio
                var buffer = $audioCtx.createBuffer(1, 1, 22050);
                var node = $audioCtx.createBufferSource();
                node.buffer = buffer;
                node.start(0);
                unlocked = true;
            }

            if (play) {
                // start playing
                current16thNote = 0;
                nextNoteTime = $audioCtx.currentTime;
                timerWorker.postMessage("start");
            } else {
                timerWorker.postMessage("stop");
            }
        }
    }
    console.groupEnd();
</script>

<style lang="scss">
    .metronome-container {
        position: relative;

        left: 30%;

        .metronome-container-inner {
            position: absolute;
            right: -6.2rem;
            bottom: 4rem;
            width: 15rem;
            padding: 1rem;
            text-align: center;
            box-shadow: 0px 3px 3px -2px rgba(0, 0, 0, 0.2),
                0px 3px 4px 0px rgba(0, 0, 0, 0.14),
                0px 1px 8px 0px rgba(0, 0, 0, 0.12);
            background-color: #2f3437;
            border-radius: 0.5rem;
            .bpm-beat-container {
                .bpm-container,
                .beat-container {
                    width: 49%;
                    display: inline-block;
                    label {
                        display: flex;
                    }
                    input {
                        width: 3rem;
                    }
                }
            }
            .subdivision-button {
                width: 3rem;
                height: 3rem;
                font-size: 2rem;
                margin: 0rem 0.5rem;
                &.active {
                    box-shadow: 0px 0px 4px 5px rgba(20, 108, 22, 1);
                }
                .subdivision-note {
                    position: relative;
                    right: 0.25rem;
                }
            }

            .bpm-beat-container,
            .slide-container,
            .subdivision-container,
            .play-container {
                margin-bottom: 1rem;
            }

            .subdivision-container,
            .play-container {
                margin-top: 2rem;
            }
        }

        .metronome-button {
            box-shadow: 0px 0px 5px 5px rgba(0, 0, 0, 0.2),
                0px 2px 2px 0px rgba(0, 0, 0, 0.14),
                0px 1px 5px 0px rgba(0, 0, 0, 0.12);
        }
        .metronome-icon {
            width: 30px;
        }
        .volume {
            display: flex;
            justify-content: center;
            align-items: center;

            input {
                width: 60%;
                margin: 0rem 0.5rem;
            }
            img {
                width: 20px;
            }
        }
    }
    @media only screen and (max-width: 768px) {
        .metronome-container {
            left: 25%;
        }
    }
</style>

<section class="metronome-container">
    {#if showMetronome}
        <div class="metronome-container-inner">
            <div class="slide-container volume">
                <img
                    class="volume-low"
                    src={vol === 0
                        ? "../icons/volume-off-light.png"
                        : "../icons/volume-low-light.png"}
                    alt="volume"
                    on:click={() => (vol = 0)}
                />
                <input
                    type="range"
                    min="0"
                    max="100"
                    bind:value={vol}
                    class="slider volume"
                />
                <img
                    class="volume-full"
                    src="../icons/volume-full-light.png"
                    alt="volume"
                    on:click={() => (vol = 100)}
                />
            </div>
            <div class="bpm-beat-container">
                <div class="bpm-container">
                    <label>
                        BPM:
                        <input
                            type="number"
                            label="play speed"
                            bind:value={bpm}
                        />
                    </label>
                </div>
                <div class="beat-container">
                    <label>
                        Beats:
                        <input
                            type="number"
                            label="beat number"
                            bind:value={numOfBeats}
                        />
                    </label>
                </div>
            </div>
            <div class="subdivision-container">
                <button
                    class="subdivision-button"
                    class:active={noteResolution === 2}
                    on:click={() => {
                        noteResolution = 2;
                    }}><div class="subdivision-note">&#119135</div></button
                >
                <button
                    class="subdivision-button"
                    class:active={noteResolution === 1}
                    on:click={() => {
                        noteResolution = 1;
                    }}><div class="subdivision-note">&#119136</div></button
                >
                <button
                    class="subdivision-button"
                    class:active={noteResolution === 0}
                    on:click={() => {
                        noteResolution = 0;
                    }}><div class="subdivision-note">&#119137</div></button
                >
            </div>

            <div class="play-container">
                <button
                    class="play-button {play ? 'playing' : 'paused'}"
                    on:click={() => {
                        !play ? (play = true) : (play = false);
                    }}
                >
                    {play ? "Stop" : "Start"}
                </button>
            </div>
        </div>
    {/if}
    <button
        class="metronome-button"
        on:click={() => {
            showMetronome ? (showMetronome = false) : (showMetronome = true);
        }}
        ><img
            src="./icons/metronome-light.png"
            alt="metronome"
            class="metronome-icon"
        /></button
    >
</section>
