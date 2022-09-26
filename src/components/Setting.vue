<script setup lang="ts">
import { ref } from 'vue';
import type { Ref } from 'vue';
import { useCookies, globalCookiesConfig } from "vue3-cookies";
import Title from '@/components/Title.vue'

const registedSuccessStatus: Ref<string> = ref<string>('');
const registedErrorStatus: Ref<string> = ref<string>('');

const { cookies } = useCookies();

let url = 'https://';
if (cookies.get('vue-ads-url') != null) {
  url = cookies.get('vue-ads-url');
}
let project = '';
if (cookies.get('vue-ads-project') != null) {
  project = cookies.get('vue-ads-project');
}
let userId = '';
if (cookies.get('vue-ads-userId') != null) {
  userId = cookies.get('vue-ads-userId');
}
let token = '';
if (cookies.get('vue-ads-token') != null) {
  token = cookies.get('vue-ads-token');
}

const workItemId = '';

const regist = (url: string, project: string , userId: string, token: string) => {

  cookies.set('vue-ads-url', url, "365d");
  cookies.set('vue-ads-project', project, "365d");
  cookies.set('vue-ads-userId', userId, "365d");
  cookies.set('vue-ads-token', token, "365d");

  registedSuccessStatus.value = 'Success';

}


</script>


<template>


  <div class="main-grid ">
    <aside>
      <Title title="Setting!" description="You have to set up!"/>

      <div class='labelbox'>
        <label>URL:</label>
      </div>

      <div class='inputbox'>
        <input type="text" v-model="url">
      </div>

      <div class='labelbox'>
        <label>Project:</label>
      </div>

      <div class='inputbox'>
        <input type="text" v-model="project">
      </div>

      <div class='labelbox'>
        <label>UserId:</label>
      </div>

      <div class='inputbox'>
        <input type="text" v-model="userId">
      </div>

      
      <div class='labelbox'>
        <label>Token:</label>
      </div>

      <div class='inputbox'>
        <input type="password" v-model="token">
      </div>

      <div class='inputbox'>
        <button @click="regist(url, project, userId, token)">
          Regist
        </button>
        <div class='successStatusbox'>{{registedSuccessStatus}}</div>
        <div class='errorStatusbox'>{{registedErrorStatus}}</div>
      </div>

    </aside>
  </div>

</template>


<style scoped>
header {
  line-height: 1.5;
  max-height: 100vh;
}

.labelbox {
	/* padding:10px; */
	width:500px;
	margin-top:10px;
}
.labelbox label {
  margin-left: 0.5em;
  font-size: 1rem;
}

.inputbox {
	width:500px;
	padding:5px;
}
.inputbox input {
  margin-left: 1em;
	width:360px;
}
.repobox {
	padding:0px;
}
.repobox ul {
	width:500px;
  list-style: none;
}
.repobox ul li {
	width:500px;
	margin-bottom:5px;
	padding-left:1em;
	text-indent:-1em;
	line-height:1.4;
}
.repobox ul li label {
	width:500px;
	margin-bottom:5px;
	padding-left:1em;
	text-indent:-1em;
	line-height:1.4;
}

.statusbox {
  margin-left: 1em;
}

.errorStatusbox {
  margin-left: 1em;
  color: red;
}

.memberList {
  color: wheat;
}
.blue {
  color: blue;
}
.green {
  color: red;
}
.red {
  color: white;
}


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
  }

  aside {
      grid-area: left;
  }

  article {
      grid-area: main;
  }
}


</style>
