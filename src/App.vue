<script setup lang="ts">
import config from './config';
import { Ref, onMounted, ref, watch } from 'vue';
import JSConfetti from 'js-confetti';
const jsConfetti = new JSConfetti();
import { gsap } from 'gsap';

type StatLine = { stat: Stat; value: number };
type Stat =
    | 'atk'
    | 'atk%'
    | 'def'
    | 'def%'
    | 'hp'
    | 'hp%'
    | 'accuracy'
    | 'res'
    | 'cr'
    | 'spd'
    | 'cd';
type Rune = [StatLine, StatLine, StatLine, StatLine] | undefined;

let type = ref('violent');
let number: Ref<1 | 2 | 3 | 4 | 5 | 6> = ref(4);
let property: Ref<Stat> = ref('cr');
let innate: Ref<StatLine> = ref({ stat: 'accuracy', value: 8 });

let _initialRune: Ref<Rune> = ref(undefined);
let _rune: Ref<Rune> = ref(undefined);

const animationTrigger = ref(false);
const enableAnimations = ref(true);

if (localStorage.getItem('enableAnimations')) {
    enableAnimations.value = JSON.parse(
        localStorage.getItem('enableAnimations') as string,
    );
}

function createInitialRune() {
    _rune.value = undefined;
    const randomRune = getRandomRune();
    const upgradedRune = upgradeRune(randomRune);
    _initialRune.value = upgradedRune;
}

function reappRune() {
    animationTrigger.value = !animationTrigger.value;
    const randomRune = getRandomRune();
    const upgradedRune = upgradeRune(randomRune);
    checkIfHasHighStats(upgradedRune);
    _rune.value = upgradedRune;
}

function upgradeRune(rune: Rune) {
    for (let index = 0; index < config.PROCS; index++) {
        const randomIndex = Math.floor(Math.random() * 4);
        const line = rune![randomIndex];
        line.value = line.value + getStatUpgrade(line.stat);
    }

    return rune;
}

function getStatUpgrade(stat: Stat) {
    return randomIntFromInterval(
        config.RANGES[stat].min,
        config.RANGES[stat].max,
    );
}

function randomIntFromInterval(min: number, max: number) {
    return Math.floor(Math.random() * (max - min + 1) + min);
}

function getRandomRune() {
    const rune = [];
    const possibleStats = [...config.STAT_TYPES] as Stat[];
    possibleStats.splice(possibleStats.indexOf(property.value), 1);

    if (innate.value) {
        possibleStats.splice(possibleStats.indexOf(innate.value.stat), 1);
    }

    for (let index = 0; index < config.PROCS; index++) {
        const statIndex = Math.floor(
            Math.random() * (possibleStats.length - 1),
        );
        const stat = possibleStats[statIndex];
        rune.push({ stat, value: getStatUpgrade(stat) });
        possibleStats.splice(statIndex, 1);
    }

    return rune as Rune;
}

function showPourcentage(stat: Stat) {
    return ['atk%', 'def%', 'hp%', 'accuracy', 'res', 'cr', 'cd'].includes(
        stat,
    );
}

function checkIfHasHighStats(rune: Rune) {
    const hasHighStat = rune?.some((line) => {
        return (
            (line.stat === 'spd' && line.value >= 23) ||
            (line.stat === 'hp%' && line.value >= 35) ||
            (line.stat === 'def%' && line.value >= 35) ||
            (line.stat === 'atk%' && line.value >= 35) ||
            (line.stat === 'cd' && line.value >= 30) ||
            (line.stat === 'cr' && line.value >= 25)
        );
    });
    if (hasHighStat) {
        jsConfetti.addConfetti();
    }
}

function onEnter(el: gsap.TweenTarget) {
    if (enableAnimations.value) {
        gsap.fromTo(
            el,
            {
                opacity: 0,
                height: 0,
            },
            {
                opacity: 1,
                height: '100%',
                delay: el.dataset.index * 0.25,
            },
        );
    }
}

onMounted(() => {
    createInitialRune();
});

watch([type, property, number, innate.value], () => {
    _initialRune.value = undefined;
    _rune.value = undefined;
});

watch(number, (newNumber) => {
    property.value = undefined;
    innate.value.stat = undefined;

    switch (newNumber) {
        case 1:
            property.value = 'atk';
            break;
        case 3:
            property.value = 'def';
            break;
        case 5:
            property.value = 'hp';
            break;
    }
});

watch(property, () => {
    innate.value.stat = undefined;
});

watch(enableAnimations, (value) => {
    localStorage.setItem('enableAnimations', JSON.stringify(value));
});
</script>

<template>
    <div class="container">
        <div class="rune-config">
            <label>Select a type</label>
            <select v-model="type">
                <option
                    v-for="option in config.RUNE_TYPES"
                    :key="option"
                    :value="option"
                >
                    {{ option }}
                </option>
            </select>

            <template v-if="type">
                <label>Select a slot</label>
                <select v-model="number">
                    <option
                        v-for="option in config.RUNE_NUMBERS"
                        :key="option"
                        :value="option"
                    >
                        {{ option }}
                    </option>
                </select>

                <template v-if="number">
                    <label>Select a property</label>
                    <select v-model="property">
                        <option
                            v-for="option in config.RUNE_PROPERTIES[number]"
                            :key="option"
                            :value="option"
                        >
                            {{ config.STAT_LABELS[option] }}
                        </option>
                    </select>

                    <template v-if="property">
                        <label>Select an innate</label>
                        <div class="d-flex gap-1">
                            <select
                                v-model="innate.stat"
                                @change="innate.value = undefined"
                            >
                                <option
                                    v-for="option in [
                                        ...config.STAT_TYPES,
                                        undefined,
                                    ]"
                                    :key="option"
                                    :value="option"
                                    :disabled="option === property"
                                >
                                    {{
                                        config.STAT_LABELS[option] ||
                                        'No innate'
                                    }}
                                </option>
                            </select>
                            <input
                                v-if="innate.stat"
                                type="number"
                                v-model="innate.value"
                                :min="config.RANGES[innate.stat].min"
                                :max="config.RANGES[innate.stat].max"
                                :disabled="!innate.stat"
                            />
                        </div>

                        <button
                            type="button"
                            @click="createInitialRune()"
                            :disabled="!type || !number || !property"
                        >
                            Generate
                        </button>
                    </template>
                </template>
            </template>
        </div>

        <div>
            <div v-if="_initialRune">
                <h3 class="mb-0 text-capitalize">
                    {{ type }} rune (slot {{ number }}) -
                    {{ config.STAT_LABELS[property] }}
                </h3>
                <h6 class="mb-0" v-if="innate?.stat && innate.value">
                    {{ config.STAT_LABELS[innate.stat] }}: {{ innate.value
                    }}{{ showPourcentage(innate.stat) ? '%' : '' }}
                </h6>
            </div>

            <div class="d-flex gap-2 mt-2">
                <div class="initial-rune">
                    <div class="result">
                        <p
                            v-for="(line, index) in _initialRune"
                            :key="index"
                            :data-index="index"
                            :class="{
                                'text-speed':
                                    line.stat === 'spd' && line.value >= 17,
                                'text-speed-high':
                                    line.stat === 'spd' && line.value >= 20,
                                'text-speed-highest':
                                    line.stat === 'spd' && line.value >= 23,
                            }"
                        >
                            <span
                                :style="{
                                    color:
                                        line.stat === 'spd' && line.value >= 17
                                            ? 'inherit'
                                            : '#e7c984',
                                }"
                                >{{ config.STAT_LABELS[line.stat] }}</span
                            >: {{ line.value
                            }}{{ showPourcentage(line.stat) ? '%' : '' }}
                        </p>
                    </div>
                    <button
                        class="mt-1 secondary"
                        type="button"
                        @click="_rune = undefined"
                        :disabled="!_initialRune || !_rune"
                    >
                        Select
                    </button>
                </div>

                <div class="rune-reapp">
                    <div class="result">
                        <TransitionGroup
                            :css="false"
                            @enter="onEnter"
                            :key="animationTrigger"
                            appear
                        >
                            <p
                                v-for="(line, index) in _rune"
                                :key="index"
                                :data-index="index"
                                :class="{
                                    'text-speed':
                                        line.stat === 'spd' && line.value >= 17,
                                    'text-speed-high':
                                        line.stat === 'spd' && line.value >= 20,
                                    'text-speed-highest':
                                        line.stat === 'spd' && line.value >= 23,
                                }"
                            >
                                <span
                                    :style="{
                                        color:
                                            line.stat === 'spd' &&
                                            line.value >= 17
                                                ? 'inherit'
                                                : '#e7c984',
                                    }"
                                    >{{ config.STAT_LABELS[line.stat] }}</span
                                >: {{ line.value
                                }}{{ showPourcentage(line.stat) ? '%' : '' }}
                            </p>
                        </TransitionGroup>
                    </div>
                    <button
                        class="mt-1 secondary"
                        type="button"
                        @click="
                            _initialRune = _rune;
                            _rune = undefined;
                        "
                        :disabled="!_initialRune || !_rune"
                    >
                        Select
                    </button>
                </div>
            </div>

            <button
                type="button"
                @click="reappRune()"
                :disabled="!_initialRune"
            >
                Reappraisal
            </button>
        </div>
    </div>

    <div class="animations-checkbox">
        <input
            type="checkbox"
            id="disableAnimations"
            name="disableAnimations"
            v-model="enableAnimations"
        />
        <label for="disableAnimations">Enable animations</label>
    </div>
</template>

<style>
.container {
    align-items: center;
    width: fit-content;
    display: flex;
    gap: 12rem;
    height: 100vh;
}

.rune-config {
    min-width: 300px;
    min-height: 500px;
}

.result {
    display: flex;
    flex-direction: column;
    padding: 1.5rem;
    border: 4px solid #808185;
    border-radius: 6px;
    min-width: 300px;
    min-height: 305px;
    text-align: center;
}

.result p {
    font-size: 1.1rem;
    font-weight: 600;
}

.result p:last-child {
    margin-bottom: 0 !important;
}

select {
    text-transform: capitalize;
    min-width: fit-content;
}

.text-speed {
    color: #8ed38a;
}

.text-speed-high {
    color: #60e857;
    font-weight: bold;
}

.text-speed-highest {
    color: #46f13a;
    font-weight: bolder;
    font-size: 1.5rem;
}

.animations-checkbox {
    position: absolute;
    right: 1rem;
    bottom: 1rem;
    font-size: 0.85rem;
}
</style>
