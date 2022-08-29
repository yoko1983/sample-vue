<!-- <script type="module" src="./sample.js"></script> -->

<script setup lang="ts">
import { ref } from 'vue';
import type { Ref } from 'vue';
import Settings from '@/components/Setting.vue'
import LinkRepo from '@/components/LinkRepo.vue'
import CreatePR from '@/components/CreatePR.vue'
import EditRepo from '@/components/EditRepo.vue'
import { useCookies } from "vue3-cookies";

const { cookies } = useCookies();

const isActive: Ref<string> = ref<string>('EditRepo');

let url:string = '';
if (cookies.get('vue-ads-url') != null) {
  url = cookies.get('vue-ads-url');
}
let project:string = '';
if (cookies.get('vue-ads-project') != null) {
  project = cookies.get('vue-ads-project');
}
let token:string = '';
if (cookies.get('vue-ads-token') != null) {
  token = cookies.get('vue-ads-token');
}



if (url==null || project == null || token ==null) {
  isActive.value = 'Setting'
}

const change = (menu: string) => {
  isActive.value = menu
}


</script>



<template>

<div class="main-grid">
  <header>
    <a v-on:click="change('EditRepo')"  v-bind:class="{'active': isActive === 'EditRepo'}">EditRepo</a>
    /
    <a v-on:click="change('LinkRepo')"  v-bind:class="{'active': isActive === 'LinkRepo'}">LinkRepo</a>
    /
    <a v-on:click="change('CreatePR')"  v-bind:class="{'active': isActive === 'CreatePR'}">CreatePR</a>
    /
    <a v-on:click="change('Setting')"  v-bind:class="{'active': isActive === 'Setting'}">Setting</a>
  </header>
  <article v-if="isActive === 'EditRepo'">
    <EditRepo />
  </article>
  <article v-if="isActive === 'LinkRepo'">
    <LinkRepo />
  </article>
  <article v-if="isActive === 'CreatePR'">
    <CreatePR />
  </article>
  <article v-if="isActive === 'Setting'">
    <Settings />
  </article>
</div>

</template>


<style scoped>
@media (min-width: 1024px) {
  .main-grid {
    display: grid;
    grid-template-areas:
    "header header"
    "left main"
    "footer footer";
    grid-template-columns: 1fr 3fr;
    grid-gap: 20px;
    height: 100vh;
    /* grid-template-rows: 70px 1fr 70px; */
    align-content: start;
  }

  header {
    grid-area: header;

  }
  header a {
    font-size: 16px;
  }

  header a:hover {
    background-color: hsla(160, 100%, 37%, 0.2);
  }

  aside {
    grid-area: left;
  }

  article {
    grid-area: main;
  }

  footer {
    grid-area: footer;
  }
}


</style>