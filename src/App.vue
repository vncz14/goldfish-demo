<script setup>

import { ref,watch } from 'vue';

const OUTPUT_PLACEHOLDER = `Seed: 0xb452bf21
1N
2E
0NE
4NE
3W
6SE
5NW
11S
15SW`

const inputValue = ref('');
const outputValue = ref(OUTPUT_PLACEHOLDER);

watch(inputValue, (newValue) => {
    if (newValue === '') {
        outputValue.value = OUTPUT_PLACEHOLDER;
    }
});

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
    return {winds: lines.map(line => {
        const [, speed = '', direction = ''] = line.match(/^(\d+)?([A-Z]+)?$/i) || [];
        return {
            direction: DIRECTIONS_TO_NUMBERS[direction.toUpperCase()] || 9,
            speed: parseInt(speed) || 0
        };
    })};
}

const jsonToOutput = (data) => {
    return data.map(s => {
        const seed = "Seed: 0x" + Number(s.seed).toString(16).padStart(8, '0');

        const winds = s.winds.map(wind => {
            const direction = Object.keys(DIRECTIONS_TO_NUMBERS).find(key => DIRECTIONS_TO_NUMBERS[key] === wind.direction) || '';
            return `${wind.speed}${direction}`;
        }).join('\n');

        return `${seed}\n${winds}`;
    }).join('\n\n') || [];

}

const getOutput = async () => {
    const res = await fetch("http://localhost:3000/wind2seed"/*"https://goldfish.914000.xyz/wind2seed"*/, {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json'
        },
        body: JSON.stringify(inputToJson())
    });

    if (res.ok) {
        const data = await res.json();
        console.log(data);
        outputValue.value = `${jsonToOutput(data)}`;
    } else {
        outputValue.value = await res.text();
    }

};

</script>

<template>

<div class="app">
    <textarea v-model="inputValue" placeholder="1n&#10;2e&#10;0/&#10;4ne&#10;3w&#10;6s"></textarea>
    <button @click="getOutput">Submit</button>
    <textarea disabled v-model="outputValue"></textarea>
</div>

</template>

<style scoped>

.app {
    display: flex;
    flex-direction: column;
    height: 100vh;
}

button {
    width: 100%;
    height: 50px;
    font-size: 20px;
    color: white;
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

</style>
