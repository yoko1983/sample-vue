<!-- <script type="module" src="./sample.js"></script> -->

<script setup lang="ts">
import { ref } from 'vue';
import type { Ref } from 'vue';
import Settings from '@/components/Setting.vue'
import CreatePR from '@/components/CreatePR.vue'
import EditRepo from '@/components/EditRepo.vue'
import MergePR from '@/components/MergePR.vue'
import MergePRByRepo from '@/components/MergePRByRepo.vue'
import { useCookies } from "vue3-cookies";

const { cookies } = useCookies();

const isActive: Ref<string> = ref<string>('EditRepo');

let url:string = cookies.get('vue-ads-url');
let project:string = cookies.get('vue-ads-project');
let userId:string = cookies.get('vue-ads-userId');
let token:string = cookies.get('vue-ads-token');

if (url==null || project == null || userId ==null || token ==null) {
  isActive.value = 'Setting'
}

if (url!=null && project != null && userId !=null && token !=null) {
  cookies.set('vue-ads-url', url);
  cookies.set('vue-ads-project', project);
  cookies.set('vue-ads-userId', userId);
  cookies.set('vue-ads-token', token);
}

const change = (menu: string) => {
  isActive.value = menu

  url = cookies.get('vue-ads-url');
  project = cookies.get('vue-ads-project');
  userId = cookies.get('vue-ads-userId');
  token = cookies.get('vue-ads-token');

  if (url==null || project == null || userId ==null || token ==null) {
    isActive.value = 'Setting'
  }

}


</script>



<template>

<div class="main-grid">
  <header>
    <a v-on:click="change('EditRepo')"  v-bind:class="{'active': isActive === 'EditRepo'}">EditRepo</a>
    /
    <a v-on:click="change('CreatePR')"  v-bind:class="{'active': isActive === 'CreatePR'}">CreatePR</a>
    /
    <a v-on:click="change('MergePR')"  v-bind:class="{'active': isActive === 'MergePR'}">MergePR</a>
    /
    <a v-on:click="change('MergePRByRepo')"  v-bind:class="{'active': isActive === 'MergePRByRepo'}">MergePRByRepo</a>
    /
    <a v-on:click="change('Setting')"  v-bind:class="{'active': isActive === 'Setting'}">Setting</a>
  </header>
  <article v-if="isActive === 'EditRepo'">
    <EditRepo />
  </article>
  <article v-if="isActive === 'CreatePR'">
    <CreatePR />
  </article>
  <article v-if="isActive === 'MergePR'">
    <MergePR />
  </article>
  <article v-if="isActive === 'MergePRByRepo'">
    <MergePRByRepo />
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