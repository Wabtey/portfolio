<script setup lang="ts">
import GitContribGraph from './components/Contributions/git-contrib-graph.vue';
import Strudel from './components/showcase/strudel.vue';
// import HelloWorld from './components/HelloWorld.vue'
</script>

<template>
    <div>
        <a href="https://gitlab.com/Wabtey">
            <img src="./assets/avatar.png" class="avatar elevator" alt="Me In the elevator" />
        </a>

        <a href="github.com/Wabtey/">
            <img src="./assets/Monkey_III.png" class="avatar monkey" alt="Me In the elevator" />
        </a>
    </div>
    <!-- <HelloWorld msg="Vite + Vue" /> -->
    <GitContribGraph />

    <br />

    <Strudel />
</template>

<style scoped>
.avatar {
    height: 6em;
    padding: 1.5em;
    will-change: filter;
    transition: filter 300ms;
    border-radius: 50%;
}

.avatar.elevator:hover {
    filter: drop-shadow(0 0 2em #ffc092aa);
}

.avatar.monkey:hover {
    filter: drop-shadow(0 0 2em #e3c29a);
}
</style>
