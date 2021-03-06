<script>
    import { createEventDispatcher, onDestroy, onMount } from "svelte";
    import { fade } from "svelte/transition";

    import { audioCtx, allPitches } from "../store";
    import PitchSelector from "./PitchSelector.svelte";
    import Pan from "./Pan.svelte";
    import WaveType from "./WaveType.svelte";

    $: console.group("Static Oscillator");

    export let oscillator;
    export let i;
    export let isPlaying;

    let unlocked = false;
    let freq = oscillator.initVals.freq;
    let pan = oscillator.initVals.pan;
    let vol = 50;
    let waveType = "sine";

    let showPitchSelector = false;
    let showWaveSelector = false;
    let showPanSelector = false;

    let inputFrequency = false;
    let pitchName;
    let closestPitch;

    //uses logarithm to make slider with 440 as middle value
    let freqSliderVal = Math.log2(freq);

    //used for keypresses
    let key;
    let ctrl = false;
    let shift = false;

    const dispatch = createEventDispatcher();

    //turns off oscillator if destroyed
    onDestroy(() => {
        if (isPlaying)
            oscillator.onOffNode.gain.setTargetAtTime(
                0,
                $audioCtx.currentTime,
                0.01
            );
    });

    onMount(() => {
        oscillator.oscNode.frequency.setValueAtTime(
            freq,
            $audioCtx.currentTime
        );
        oscillator.onOffNode.gain.setValueAtTime(0, $audioCtx.currentTime);

        oscillator.volGainNode.gain.setValueAtTime(0.5, $audioCtx.currentTime);

        oscillator.sequenceGainNode.gain.setValueAtTime(
            1,
            $audioCtx.currentTime
        );

        oscillator.panNode.panningModel = "equalpower";
        if (oscillator.panNode.positionX) {
            oscillator.panNode.positionX.value = pan;
            oscillator.panNode.positionY.value = 0;
            oscillator.panNode.positionZ.value = -1;
        } else {
            oscillator.panNode.setPosition(0, 0, -1);
        }
        //     oscillator.oscNode.start();
    });

    function playHandler() {
        if (!unlocked && isPlaying) {
            $audioCtx
                .resume()
                .then(() => {
                    oscillator.oscNode.start();
                })
                .then(() => {
                    oscillator.onOffNode.gain.setTargetAtTime(
                        0.9,
                        $audioCtx.currentTime,
                        0.01
                    );
                });

            unlocked = true;
        }

        if (isPlaying) {
            oscillator.onOffNode.gain.setTargetAtTime(
                0.9,
                $audioCtx.currentTime,
                0.01
            );
        } else if (!isPlaying) {
            oscillator.onOffNode.gain.setTargetAtTime(
                0,
                $audioCtx.currentTime,
                0.01
            );
        }
    }

    function handlePitchSelector(event) {
        if (event.detail.text === "close") {
            showPitchSelector = false;
        }
        if (event.detail.text === "pitch") {
            showPitchSelector = false;
            pitchName = event.detail.note;
            freqSliderVal = Math.log2(event.detail.frequency);
            changeFreqSlider();
        }
    }

    //compensates for log difference between freq and freqSliderVal
    function changeFreqSlider() {
        freq = 2 ** freqSliderVal;
    }

    function handleKeydown(e) {
        if (e.ctrlKey) {
            ctrl = true;
        }
        if (e.shiftKey) {
            shift = true;
        }
        key = e.keyCode;

        switch (key) {
            //space
            case 32:
                isPlaying ? (isPlaying = false) : (isPlaying = true);
                break;
            //right arrow
            case 39:
                if (!ctrl && !shift) freq += 1;
                if (ctrl && !shift) freq += 100;
                if (ctrl && shift) freq += 0.1;
                break;
            //left arrow
            case 37:
                if (!ctrl && !shift) freq -= 1;
                if (ctrl && !shift) freq -= 100;
                if (ctrl && shift) freq -= 0.1;
                break;
        }
        key = null;
    }

    function handleKeyup() {
        ctrl ? (ctrl = false) : ctrl;
        shift ? (shift = false) : shift;
    }

    $: {
        //frequency control
        oscillator.oscNode.frequency.setValueAtTime(
            freq,
            $audioCtx.currentTime
        );
        //finds closest pitch based on current freq
        closestPitch = $allPitches.reduce((a, b) => {
            return Math.abs(b.frequency - freq) < Math.abs(a.frequency - freq)
                ? b
                : a;
        });
    }
    //pan control
    $: oscillator.panNode.positionX.setTargetAtTime(
        pan,
        $audioCtx.currentTime,
        0.01
    );
    //Wave Type Selector
    $: oscillator.oscNode.type = waveType;
    $: playHandler(isPlaying);
    //   $: changeFreqSlider(freqSliderVal);

    //volume control
    $: oscillator.volGainNode.gain.setTargetAtTime(
        vol * 0.01,
        $audioCtx.currentTime,
        0.01
    );
    console.groupEnd();
</script>

<style type="scss">
    .close-container {
        display: flex;
        justify-content: flex-end;
    }
    .play-vol-container {
        display: grid;
        grid-template-columns: 20% 80%;
        margin: 1rem;
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

    .pan {
        position: relative;
    }

    .pan-controller {
        display: grid;
        position: absolute;
        top: 3rem;
        left: -20%;
        padding: 1rem;
        background-color: rgb(221, 221, 221);
        box-shadow: 0px 3px 3px -2px rgba(0, 0, 0, 0.2),
            0px 3px 4px 0px rgba(0, 0, 0, 0.14),
            0px 1px 8px 0px rgba(0, 0, 0, 0.12);
        &::after {
            content: "";
            position: absolute;
            width: 12px;
            height: 12px;
            left: 50%;
            top: -10px;
            transform: translate(-50%, 50%) rotate(45deg);
            background-color: rgb(221, 221, 221);
        }
    }

    .wave-select-container {
        position: relative;
        .wave-select {
            position: absolute;
            top: 3rem;
            left: -100%;
            background-color: rgb(221, 221, 221);
            display: flex;
            box-shadow: 0px 3px 3px -2px rgba(0, 0, 0, 0.2),
                0px 3px 4px 0px rgba(0, 0, 0, 0.14),
                0px 1px 8px 0px rgba(0, 0, 0, 0.12);
            &::after {
                content: "";
                position: absolute;
                width: 12px;
                height: 12px;
                left: 50%;
                top: -10px;
                transform: translate(-50%, 50%) rotate(45deg);
                background-color: rgb(221, 221, 221);
                //box-shadow: 0 1px 8px rgba(0, 0, 0, 0.5);
            }

            .wave-select-box {
                margin: 0.5rem;
            }
        }

        .wave-select-button {
            background-size: contain;
            background-position: center;
            background-repeat: no-repeat;
            width: 40px;
            height: 100%;
            &.sine {
                background-image: url("/icons/sine-light.png");
            }
            &.square {
                background-image: url("/icons/square-light.png");
            }
            &.triangle {
                background-image: url("/icons/triangle-light.png");
            }
            &.sawtooth {
                background-image: url("/icons/sawtooth-light.png");
            }
        }
    }

    .frequency-container {
        margin: 0 2rem;
        .slider.frequency {
            width: 100%;
        }

        .frequency-controls {
            display: grid;
            grid-template-columns: 18% 18% 28% 18% 18%;
            justify-content: center;

            .frequency-arith-button {
                margin: 0px 5px;
                max-width: 5rem;
            }
        }

        .frequency-label {
            text-align: center;
            input {
                width: 5rem;
            }
        }
    }

    .pan-wave-container {
        display: grid;
        grid-template-columns: auto auto;
        margin: 1rem;
        justify-content: center;
        div {
            margin: 1rem;
        }
    }

    .pitch-selector-container {
        text-align: center;
        margin: 2rem 0;
    }

    @media only screen and (max-width: 768px) {
        .frequency-controls {
            .frequency-arith-button {
                font-size: 0.65rem;
            }
        }
    }
</style>

<svelte:window on:keydown={handleKeydown} on:keyup={handleKeyup} />

<section class="card oscillator-container" in:fade>
    <div class="close-container">
        <button
            on:click={() => dispatch("closeStaticOscillator", i)}
            class="close">X</button
        >
    </div>
    <div class="play-vol-container">
        <button
            class="play-button {isPlaying ? 'playing' : 'paused'}"
            on:click={() => {
                isPlaying ? (isPlaying = false) : (isPlaying = true);
            }}
            >{isPlaying ? "Pause" : "Play"}
        </button>
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
    </div>

    <div class="pitch-selector-container">
        <button
            class="pitch-selector"
            on:click={() =>
                showPitchSelector
                    ? (showPitchSelector = false)
                    : (showPitchSelector = true)}
            >Select a Pitch
        </button>
    </div>

    <!--min and max value of frequency slider are arbitrary. The slider functions logarithmically with 440 as the center-->
    <div class="frequency-container">
        <div class="slide-container Frequency">
            <input
                type="range"
                min={3}
                max={14.4}
                step={0.001}
                bind:value={freqSliderVal}
                on:input={changeFreqSlider}
                class="slider frequency"
            />
        </div>
        <div class="frequency-controls">
            <button
                class="frequency-arith-button"
                on:click={() => (freqSliderVal = Math.log2((freq *= 0.5)))}
                >&divide; 2
            </button>
            <button
                class="frequency-arith-button"
                on:click={() => (freqSliderVal = Math.log2((freq -= 1)))}
                >&minus 1
            </button>
            <div
                class="frequency-label"
                on:click={() =>
                    inputFrequency
                        ? (inputFrequency = false)
                        : (inputFrequency = true)}
            >
                {#if inputFrequency}
                    <input type="number" autofocus bind:value={freq} step={1} />
                {:else}
                    {freq.toFixed(13) != Math.round(freq)
                        ? "~"
                        : ""}{parseFloat(freq.toFixed(2))}
                {/if}

                Hz
                <div
                    class="frequency-label pitch"
                    on:click={() => (freq = closestPitch.frequency)}
                >
                    {#if Math.round(freq) != Math.round(closestPitch.frequency)}
                        {Math.round(freq) > Math.round(closestPitch.frequency)
                            ? ">"
                            : "<"}
                    {/if}
                    {closestPitch.note}
                </div>
            </div>
            <button
                class="frequency-arith-button"
                on:mousedown={() => (freqSliderVal = Math.log2((freq += 1)))}
                >&plus 1
            </button>
            <button
                class="frequency-arith-button"
                on:click={() => (freqSliderVal = Math.log2((freq *= 2)))}
                >&times 2
            </button>
        </div>
    </div>
    <div class="pan-wave-container">
        <div class=" wave-select-container">
            <button
                class="wave-select-button {waveType}"
                on:click={() =>
                    showWaveSelector
                        ? (showWaveSelector = false)
                        : (showWaveSelector = true)}
            />
            {#if showWaveSelector}
                <WaveType bind:waveType bind:showWaveSelector />
            {/if}
        </div>

        <div class="slide-container pan">
            <button
                class="pan-button"
                on:click={() =>
                    showPanSelector
                        ? (showPanSelector = false)
                        : (showPanSelector = true)}
            >
                Pan
            </button>
            {#if showPanSelector}
                <Pan bind:pan bind:showPanSelector />
            {/if}
        </div>
    </div>
</section>
{#if showPitchSelector}
    <PitchSelector
        bind:showPitchSelector
        on:message={handlePitchSelector}
        bind:pitchName
    />
{/if}
