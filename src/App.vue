<script setup lang="ts">
import config from './config';
import { Ref, ref } from 'vue';
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
let innate: Ref<StatLine> = ref({ stat: 'accuracy', value: 6 });
let _rune: Ref<Rune> = ref(undefined);
const reappButtonDisabled = ref(false);
const animationTrigger = ref(false);

function createRune() {
    animationTrigger.value = !animationTrigger.value;
    const randomRune = getRandomRune();
    const upgradedRune = upgradeRune(randomRune);
    checkIfHasHighStats(upgradedRune);
    _rune.value = upgradedRune;
}

function upgradeRune(rune: Rune) {
    for (let index = 0; index < config.PROCS; index++) {
        const randomIndex = Math.floor(Math.random() * 4);
        const line = rune[randomIndex];
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

function clearConfig() {
    type.value = null;
    number.value = null;
    property.value = null;
    innate.value.stat = null;
    innate.value.value = null;
    _rune.value = null;
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
        reappButtonDisabled.value = true;

        setTimeout(() => {
            reappButtonDisabled.value = false;
        }, 1000);
    }
}

function onEnter(el: gsap.TweenTarget) {
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
                        <div class="innate-block">
                            <select v-model="innate.stat">
                                <option
                                    v-for="option in config.STAT_TYPES"
                                    :key="option"
                                    :value="option"
                                >
                                    {{ config.STAT_LABELS[option] }}
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

                        <div
                            v-if="innate.stat && innate.value"
                            class="d-flex gap-1"
                        >
                            <button
                                type="button"
                                @click="createRune()"
                                :disabled="
                                    !!_rune || !type || !number || !property
                                "
                            >
                                Generate
                            </button>
                            <button type="button" @click="clearConfig()">
                                Clear
                            </button>
                        </div>
                    </template>
                </template>
            </template>
        </div>

        <div class="rune-reapp">
            <template v-if="_rune">
                <h3 class="mb-0 text-capitalize">
                    {{ type }} rune ({{ number }})
                </h3>
                <h5 class="mb-0">
                    {{ config.STAT_LABELS[property] }}
                </h5>
                <h6>
                    {{ config.STAT_LABELS[innate.stat] }}: {{ innate.value
                    }}{{ showPourcentage(innate.stat) ? '%' : '' }}
                </h6>
            </template>

            <div class="result">
                <!-- <template v-if="_rune"> -->
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
                                    line.stat === 'spd' && line.value >= 17
                                        ? 'inherit'
                                        : '#e7c984',
                            }"
                            >{{ config.STAT_LABELS[line.stat] }}</span
                        >: {{ line.value
                        }}{{ showPourcentage(line.stat) ? '%' : '' }}
                    </p>
                </TransitionGroup>
                <!-- </template>
                <div v-else>
                    <p>--------</p>
                    <p>--------</p>
                    <p>--------</p>
                    <p>--------</p>
                </div> -->
            </div>

            <button
                type="button"
                @click="createRune()"
                :disabled="!_rune || reappButtonDisabled"
            >
                Reappraisal
            </button>
        </div>
    </div>
</template>

<style>
.container {
    align-items: center;
    width: fit-content;
    display: flex;
    gap: 10rem;
    height: 100vh;
}

.rune-config {
    min-width: 200px;
    min-height: 500px;
}

.rune-reapp {
    display: flex;
    flex-direction: column;
    justify-content: center;
}

.result {
    display: flex;
    flex-direction: column;
    margin-bottom: 2rem;
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
</style>
