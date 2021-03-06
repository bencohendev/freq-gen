<script>
    import { onMount } from "svelte";

    import { audioCtx, octaves, pitchObj } from "../store.js";
    import uuid from "shortid";
    import StaticOscillator from "../components/StaticOscillator.svelte";
    import { createNewOscillator } from "../helpers/NewOscillator.svelte";

    $: console.group("static");
    let unlocked = false;

    let oscillatorArray = [];
    let newNode;
    let pan = 0;
    //sequence is used in random sequence generator. For static it is always 1.
    let sequence = 1;
    let freq = 440;
    let selectedOctave;
    let selectedPitch;
    let selectedOvertones;
    let showGenerateOvertones = false;
    let id;

    let keyArray = ["C", "D", "E", "F", "G", "A", "B"];

    //Creates audio context and one oscillator on mount
    onMount(() => {
        audioCtx.set(new (window.AudioContext || window.webkitAudioContext)());
        addNewOscillator(freq, pan);
    });

    //Creates new oscillators with unique id
    function addNewOscillator(freq, pan) {
        id = uuid.generate();
        newNode = createNewOscillator($audioCtx, freq, pan, sequence);
        newNode.id = id;
        newNode.isPlaying = false;
        oscillatorArray = [...oscillatorArray, newNode];
    }

    function handleCloseStaticOscillator(e) {
        oscillatorArray.splice(e.detail, 1);
        oscillatorArray = oscillatorArray;
    }

    function playAllHandler() {
        if (!unlocked) {
            $audioCtx.resume();
            unlocked = true;
        }

        //checks if all are already playing
        let isAllPlaying = oscillatorArray.every(
            (oscillator) => oscillator.isPlaying === true
        );

        //if allPlaying is true, pause. Otherwise play
        oscillatorArray.forEach(
            (oscillator) => (oscillator.isPlaying = isAllPlaying ? false : true)
        );
        oscillatorArray = oscillatorArray;
    }
    function muteAllHandler() {
        oscillatorArray.forEach((oscillator) => (oscillator.isPlaying = false));
        oscillatorArray = oscillatorArray;
    }

    async function keyHandler() {
        keyArray = [];
        //reorders array of pitches based on key
        let pitchFront = $pitchObj.slice(selectedPitch, 12);
        let pitchBack = $pitchObj.slice(0, selectedPitch);
        keyArray = pitchFront.concat(pitchBack);

        return keyArray;
    }

    async function handleSelectedOvertones(selectedOctave) {
        keyArray = await keyHandler();
        let fundamentals = [];
        //I need a set of fundamental frequencies. I am given a note name, I create an array of notes

        switch (selectedOvertones) {
            case "1 - 3 - 5":
                oscillatorArray = [];
                fundamentals = [
                    keyArray[0].frequency * selectedOctave.multiplier,
                    keyArray[7].frequency * selectedOctave.multiplier * 2,
                    keyArray[4].frequency * selectedOctave.multiplier * 8,
                ];
                addNewOscillator(fundamentals[0], -1);
                addNewOscillator(fundamentals[1], 1);
                addNewOscillator(fundamentals[2], -1);

                selectedOvertones = "Select Overtone Set";

                break;
            case "1 - 3 - 5 - 7":
                oscillatorArray = [];
                fundamentals = [
                    keyArray[0].frequency * selectedOctave.multiplier,
                    keyArray[7].frequency * selectedOctave.multiplier * 2,
                    keyArray[4].frequency * selectedOctave.multiplier * 8,
                    keyArray[10].frequency * selectedOctave.multiplier * 16,
                ];
                addNewOscillator(fundamentals[0], -1);
                addNewOscillator(fundamentals[1], 1);
                addNewOscillator(fundamentals[2], -1);
                addNewOscillator(fundamentals[3], 1);
                selectedOvertones = "Select Overtone Set";
                break;
        }
    }
    console.groupEnd();
</script>

<style lang="scss">
    .static {
        align-items: center;
        justify-content: center;
        background-color: #4c4c4c;

        .oscillator-master-control {
            display: flex;

            button {
                margin: 1rem;
            }
            .create-oscillator {
                margin-left: 0;
            }
        }
        .overtone-preset-container {
            h3 {
                margin-bottom: 0.5rem;
            }
            margin-bottom: 1rem;
            select {
                margin-right: 1rem;
            }
        }
    }
    @media only screen and (max-width: 768px) {
        .static {
            .overtone-preset-container {
                margin: 0 0 1rem 0;
            }
        }
    }
</style>

<svelte:head>
    <title>Static</title>
</svelte:head>

<div class="static">
    <section class="oscillator-master-control">
        <button
            class="create-oscillator"
            on:click={() => addNewOscillator(freq, pan)}
            >Add Tone Generator</button
        >
        <button class="play-all paused" on:click={playAllHandler}
            >Play All</button
        >
        <button class="mute-all playing" on:click={muteAllHandler}
            >Mute All</button
        >
        <button
            class="generate-overtones"
            on:click={() => {
                showGenerateOvertones
                    ? (showGenerateOvertones = false)
                    : (showGenerateOvertones = true);
            }}
        >
            Auto Generate Overtones
        </button>
    </section>
    {#if showGenerateOvertones}
        <div class="overtone-preset-container">
            <h3>Select a Fundamental</h3>
            <div class="overtone-select-container">
                <select
                    name="pitch-select"
                    id="pitch-select"
                    bind:value={selectedPitch}
                >
                    <option value="8">Pitch</option>
                    {#each $pitchObj as pitch, j}
                        <option value={j}>{pitch.note}</option>
                    {/each}
                </select>
                <select
                    name="octave-select"
                    id="octave-select"
                    bind:value={selectedOctave}
                >
                    <option value={$octaves[3]}>Octave</option>
                    {#each $octaves as octave}
                        <option label={octave.label} value={octave} />
                    {/each}
                </select>
                <!-- svelte-ignore a11y-no-onchange -->
                <select
                    name="overtone-series-select"
                    id="overtone-series-select"
                    bind:value={selectedOvertones}
                    on:change={() =>
                        handleSelectedOvertones(selectedOctave, selectedPitch)}
                >
                    <option>Select Overtone Set</option>
                    <option>1 - 3 - 5</option>
                    <option>1 - 3 - 5 - 7</option>
                </select>
            </div>
        </div>
    {/if}
    {#each oscillatorArray as oscillator, i (oscillator.id)}
        <StaticOscillator
            {oscillator}
            {i}
            bind:isPlaying={oscillator.isPlaying}
            on:closeStaticOscillator={handleCloseStaticOscillator}
        />
    {/each}
</div>
