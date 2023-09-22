<script setup lang="ts">
import config from './config';
import { Ref, computed, onMounted, ref, watch } from 'vue';
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
const reappButtonDisabled = ref(false);
const total = ref(0);

const rune_types = computed(() => {
    return config.RUNE_TYPES.sort((n) => {
        if (['violent', 'will'].includes(n)) return -1;
        return 1;
    });
});

if (localStorage.getItem('enableAnimations')) {
    enableAnimations.value = JSON.parse(
        localStorage.getItem('enableAnimations') as string,
    );
}

function createInitialRune() {
    total.value = 0;
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

    total.value++;
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
        const statIndex = Math.floor(Math.random() * possibleStats.length);
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
            (line.stat === 'spd' && line.value >= 22) ||
            (line.stat === 'hp%' && line.value >= 35) ||
            (line.stat === 'def%' && line.value >= 35) ||
            (line.stat === 'atk%' && line.value >= 35) ||
            (line.stat === 'cd' && line.value >= 30) ||
            (line.stat === 'cr' && line.value >= 25)
        );
    });
    if (hasHighStat) {
        jsConfetti.addConfetti();

        reappButtonDisabled.value = true;
        setTimeout(() => {
            reappButtonDisabled.value = false;
        }, 1000);
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

function getInnates() {
    const innates = config.STAT_TYPES.filter((innate) => {
        return (
            innate !== property.value &&
            !config.EXCLUDE_TYPES[number.value].includes(innate)
        );
    });
    return [...innates, undefined];
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
                    v-for="option in rune_types"
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
                                @change="
                                    innate.value =
                                        config.RANGES[innate.stat].max
                                "
                            >
                                <option
                                    v-for="option in getInnates()"
                                    :key="option"
                                    :value="option"
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
                                    line.stat === 'spd' && line.value >= 20,
                            }"
                        >
                            <span>{{ config.STAT_LABELS[line.stat] }}: </span
                            >{{ line.value
                            }}{{ showPourcentage(line.stat) ? '%' : '' }}
                        </p>
                    </div>
                    <button
                        class="mt-1 contrast"
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
                                        line.stat === 'spd' && line.value >= 20,
                                }"
                            >
                                <span
                                    >{{ config.STAT_LABELS[line.stat] }}: </span
                                >{{ line.value
                                }}{{ showPourcentage(line.stat) ? '%' : '' }}
                            </p>
                        </TransitionGroup>
                    </div>
                    <button
                        class="mt-1 contrast"
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
                :disabled="!_initialRune || reappButtonDisabled"
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
        <label
            for="disableAnimations"
            style="margin-left: 5px; margin-top: -3px"
        >
            Enable animations
        </label>
    </div>

    <div class="total">
        <span>Total reapps:</span> <strong>{{ total }}</strong>
        <span @click="total = 0" class="reset-icon" title="Reset total">
            ✕
        </span>
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
    border: 2px solid var(--secondary);
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

.result p span {
    font-weight: 300;
    color: var(--h4-color);
}

select {
    text-transform: capitalize;
    min-width: fit-content;
}

.text-speed {
    color: #60e857;
    font-weight: bold !important;
}

.animations-checkbox {
    position: absolute;
    left: 1.25rem;
    bottom: 1.25rem;
    display: flex;
    align-items: center;
}

.animations-checkbox:hover > label {
    color: var(--h3-color);
}

.total {
    position: absolute;
    right: 1.25rem;
    bottom: 1.25rem;
}

.reset-icon {
    cursor: pointer;
    margin-left: 0.3rem;
}

.reset-icon:hover {
    font-weight: bold;
}
</style>
