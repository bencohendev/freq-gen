<script>
    import { fade } from "svelte/transition";

    $: console.group("Pan");
    export let pan;
    export let showPanSelector;

    console.groupEnd();
</script>

<style lang="scss">
    .pan-controller {
        display: grid;
        position: absolute;
        top: 3rem;
        left: -5rem;
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

        .pan-buttons {
            display: grid;
            grid-template-columns: 33% 33% 33%;
            justify-content: center;

            button {
                margin: 1rem;
                font-size: 10px;
                width: 25px;
                height: 25px;
            }
            .pan-left-button {
                position: relative;
                right: 25%;
            }

            .pan-right-button {
                position: relative;
                left: 25%;
            }
        }

        .pan-slider {
            display: grid;
            grid-template-columns: 15% 70% 15%;
            img {
                width: 20px;

                &.pan-left-icon {
                    -webkit-transform: scaleX(-1) scaleY(1);
                    transform: scaleX(-1) scaleY(1);
                }
            }
            input {
                &::after {
                    content: "";
                    border-right: solid 1px black;
                    position: absolute;
                    left: 49.5%;
                    top: 75px;
                    width: 1px;
                    height: 30px;
                }
            }
        }
    }
</style>

<!--toggle showPanSelector if click outside of Pan Selector-->
<svelte:window
    on:click={(e) => {
        if (!e.target.classList.value.includes("pan")) showPanSelector = false;
    }}
/>

<div class="pan-controller" transition:fade>
    <div class="pan-buttons">
        <button class="pan-left-button" on:click={() => (pan = -1)}>L</button>
        <button class="pan-center-button" on:click={() => (pan = 0)}>C</button>
        <button class="pan-right-button" on:click={() => (pan = 1)}>R</button>
    </div>
    <div class="pan-slider">
        <img
            class="pan-left-icon"
            src="../icons/pan.png"
            alt="pan left"
            on:click={() => (pan = -1)}
        />
        <input
            type="range"
            min="-1"
            max="1"
            step={0.01}
            bind:value={pan}
            class="slider pan"
        />
        <img
            class="pan-right-icon"
            src="../icons/pan.png"
            alt="pan right"
            on:click={() => (pan = 1)}
        />
    </div>
</div>
