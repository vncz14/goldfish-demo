<script setup>

import { ref,watch } from 'vue';

const OUTPUT_PLACEHOLDER = `Seed: 0xb452bf21
1N, 2E, 0NE, 4NE, 3W, 6SE, 5NW, 11S, 15SW`

const inputValue = ref('');
const outputValue = ref(OUTPUT_PLACEHOLDER);

const game = ref('og_1.1');
const useLastKnownSeed = ref(false);
const lastKnownSeed = ref('');
const numToCheck = ref(1000);
const customAdvance = ref(1);

watch(inputValue, (newValue) => {
    if (newValue === '') {
        outputValue.value = OUTPUT_PLACEHOLDER;
    }
});

const OGWS_SETTINGS = {
    wind_count: 9,
    advance_function: (seed) => (seed * 69069 + 1) >>> 0,
    advance_counts: {
        reset: 41,
        hole_load: 17,
    }
};

const WSR_SETTINGS = {
    wind_count: 18,
    advance_function: (seed) => (seed * 0x41c64e6d + 0x3039) >>> 0,
    advance_counts: {
        reset: 102,
        hole_load: 45,
    }
};

const GAME_SETTINGS = {
    'og_1.0': OGWS_SETTINGS,
    'og_1.1': OGWS_SETTINGS,
    'wsr': WSR_SETTINGS,
};

const initialize = (seed) => {
    lastKnownSeed.value = "0x" + seed.toString(16).padStart(8, '0');
}

const advance = (n = 1) => {
    let seed = lastKnownSeed.value ? parseInt(lastKnownSeed.value.slice(2), 16) : 0;
    for (let i = 0; i < n; i++) {
        seed = GAME_SETTINGS[game.value].advance_function(seed);  
    }
    initialize(seed)
}

const DIRECTIONS_TO_NUMBERS = {
    'S': 0,
    'SE': 1,
    'E': 2,
    'NE': 3,
    'N': 4,
    'NW': 5,
    'W': 6,
    'SW': 7,
    '/': 9,
};

const inputToJson = () => {
    const lines = inputValue.value.trim().split('\n');

    const json = {
        game: game.value, 
        last_known_seed: useLastKnownSeed.value && lastKnownSeed.value ? parseInt(lastKnownSeed.value.slice(2), 16) : null,
        num_to_check: useLastKnownSeed.value && lastKnownSeed.value ? parseInt(numToCheck.value) || 1000 : null,
        winds: lines.map(line => {
        const [, speed = '', direction = ''] = line.match(/^(\d+)?([A-Z]+)?$/i) || [];
        return {
            direction: direction
                ? (DIRECTIONS_TO_NUMBERS[direction.toUpperCase()] ?? 9)
                : 9,
            speed: parseInt(speed) || 0
        };
        }),
    };
    console.log(json);
    return json;
}

const jsonToOutput = (data) => {
    return data.map(s => {
        const seed = "Seed: 0x" + Number(s.seed).toString(16).padStart(8, '0');

        const winds = s.winds.slice(0, GAME_SETTINGS[game.value].wind_count).map(wind => {
            const direction = Object.keys(DIRECTIONS_TO_NUMBERS).find(key => DIRECTIONS_TO_NUMBERS[key] === wind.direction) || '';
            return `${wind.speed}${direction}`;
        }).join(', ');

        return `${seed}\n${winds}`;
    }).join('\n\n') || [];

}

const getOutput = async () => {
    // const res = await fetch("http://localhost:3000/wind2seed", {
    const res = await fetch("https://goldfish.914000.xyz/wind2seed", {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json'
        },
        body: JSON.stringify(inputToJson())
    });

    if (res.ok) {
        const data = await res.json();
        outputValue.value = `${jsonToOutput(data)}`;

        // set last known seed
        if (data.length == 1) {

            initialize(data[0].seed);

            let advancesFromReset = GAME_SETTINGS[game.value].advance_counts.reset;
            // number of holes in input - 1
            let advancesFromHoleLoads = (inputValue.value.trim().split('\n').length - 1) * GAME_SETTINGS[game.value].advance_counts.hole_load;
            advance(advancesFromReset + advancesFromHoleLoads);
        }
    } else {
        outputValue.value = await res.text();
    }

};

</script>

<template>

<div class="app">
    <textarea v-model="inputValue" placeholder="1n&#10;2e&#10;0/&#10;4ne&#10;3w&#10;6s"></textarea>
    <button class="submit-button" @click="getOutput">Submit</button>
    <textarea disabled v-model="outputValue"></textarea>
    <div class="settings">
        <div class="game-select">
            <h3>Game</h3>
            <select v-model="game">
                <option value="og_1.0">Wii Sports 1.0</option>
                <option value="og_1.1">Wii Sports 1.1/1.2</option>
                <option value="wsr">Wii Sports Resort</option>
            </select>
        </div>
        <div class="last-known-seed">
            <h3>Chaining settings</h3>
            <div class="last-known-seed-settings-list">
                <div>
                    <input type="checkbox" id="last-known-seed" v-model="useLastKnownSeed">
                    <label for="last-known-seed">Use last known seed</label>
                </div>
                <div>
                    <input type="text" :disabled="!useLastKnownSeed" v-model="lastKnownSeed" placeholder="Last known seed">        
                </div>
                <div>
                    <input type="text" :disabled="!useLastKnownSeed" v-model="numToCheck" placeholder="Number of advances ahead to check">        
                </div>
                <div class="advance-buttons">
                    <button :disabled="!useLastKnownSeed" @click="advance(GAME_SETTINGS[game].advance_counts.reset)">Reset (+{{ GAME_SETTINGS[game].advance_counts.reset}})</button>
                    <button :disabled="!useLastKnownSeed" @click="advance(GAME_SETTINGS[game.value].advance_counts.hole_load)">Load hole (+{{ GAME_SETTINGS[game].advance_counts.hole_load }})</button>
                    <div>
                        <button :disabled="!useLastKnownSeed" @click="advance(customAdvance)">Custom advance (+{{ customAdvance }})</button>
                        <input type="number" v-model="customAdvance" placeholder="Custom advance value" :disabled="!useLastKnownSeed">
                    </div>
                </div>
            </div>
        </div>
    </div>
</div>

</template>

<style scoped>

* {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
    font-family: sans-serif;
}

.app {
    display: flex;
    flex-direction: column;
    height: 100vh;
}

.submit-button {
    width: 100%;
    height: 50px;
    font-size: 20px;
    border: none;
    cursor: pointer;
}

textarea {
    width: 100%;
    font-family: monospace;
    font-size: 16px;
    padding: 10px;
    box-sizing: border-box;
    height: 40%;
}

.settings {
    display: flex;
    gap: 20px;
}

.settings > div {
    width: 300px;
}

.last-known-seed-settings-list {
    display: flex;
    flex-direction: column;
    gap: 10px;
}

.advance-buttons {
    display: flex;
    flex-direction: column;
    gap: 10px;
}

.advance-buttons button {
    width: 100%;
}

</style>
