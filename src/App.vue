<!-- <script type="module" src="./sample.js"></script> -->

<script setup lang="ts">
import { RouterLink, RouterView } from 'vue-router'
import HelloWorld from './components/HelloWorld.vue'
import { ref } from 'vue';
import type { Ref } from 'vue';
import { createApp, reactive, computed } from 'vue';

import { onMounted } from "vue"
import axios from "axios"
import { Buffer } from 'buffer'


const token: Ref<string> = ref<string>('');

const results = ref([])
// const results = ""
const error = ref([])
// ボタンクリック時に実行するパターン
const runapi = (token) => {



  axios.get('https://dev.azure.com/yoko1983/test/_apis/git/repositories', 
      { 
            auth: {
              password: token
            },
            params: {
              'api-version':'5.1'
            }
      })
    .then(response => results.value = response.data)
    .catch(error => error)
};
// 画面遷移時に実行するパターン
// onMounted(() => {
//   const token='';
//   axios.get('https://dev.azure.com/yoko1983/test/_apis/git/repositories', 
//       { 
//             auth: {
//               password: token
//             },
//             params: {
//               'api-version':'5.1'
//             }
//       })
//     .then(response => results.value = response.data)
//     .catch(error => error)
// })

// チェックボックスの状態です
const checkBoxState = reactive({
  aa: true,
  bb: false,
  cc: false,
  dd: false,
  ee: false,
  ff: false,
  gg: false,
});

// ボタンが押せるかどうか
const isButtonEnabled = computed(
  () =>
    Object.values(checkBoxState).filter((button: boolean) => button === true)
      .length >= 3
);




const count: Ref<number> = ref<number>(0);



</script>


<template>
  <header>
    <!-- <img alt="Vue logo" class="logo" src="@/assets/logo.svg" width="125" height="125" /> -->

    <div class="wrapper">
      <HelloWorld msg="Link repositories!" />

      <div class='labelbox'>
        <label>Token:</label>
      </div>

      <div class='tokenbox'>
        <input type="text" v-model="token">
        <button @click="runapi(token)">
          Display
        </button>
      </div>
      <div v-for="(repo, i) in results.value" :key="i" class='repobox'>
      <ul>
        <li>
        <input
          :id="'repo.name' + i"
          type="checkbox"
          :value="repo.name"
          v-model="selectedPokes"
        />
        <label :for="'repo.name' + i">{{repo.name}}</label>
        </li>
      </ul>
      </div>

      <div class='labelbox'>
        <label>WorkItem ID:</label>
      </div>
      <div class='tokenbox'>
        <input type="text" v-model="wiid">
        <button @click="linkWI(wiid)">
          Link
        </button>
      </div>


      <nav>
        <RouterLink to="/">Home</RouterLink>
        <RouterLink to="/about">About</RouterLink>
      </nav>

      <div v-if="hasError">
          <article class="message is-danger">
              <div class="message-header">
                  <p>Error</p>
              </div>
              <div class="message-body">
                  {{ errorMessage }}
              </div>
          </article>
      </div>
    </div>
  </header>

  <RouterView />
</template>


<style scoped>
header {
  line-height: 1.5;
  max-height: 100vh;
}

.logo {
  display: block;
  margin: 0 auto 2rem;
}

nav {
  width: 100%;
  font-size: 12px;
  text-align: center;
  margin-top: 2rem;
}

nav a.router-link-exact-active {
  color: var(--color-text);
}

nav a.router-link-exact-active:hover {
  background-color: transparent;
}

nav a {
  display: inline-block;
  padding: 0 1rem;
  border-left: 1px solid var(--color-border);
}

nav a:first-of-type {
  border: 0;
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

.tokenbox {
	width:500px;
	padding:5px;
}
.tokenbox input {
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

@media (min-width: 1024px) {
  header {
    display: flex;
    place-items: center;
    padding-right: calc(var(--section-gap) / 2);
  }

  .logo {
    margin: 0 2rem 0 0;
  }

  header .wrapper {
    display: flex;
    place-items: flex-start;
    flex-wrap: wrap;
  }

  nav {
    text-align: left;
    margin-left: -1rem;
    font-size: 1rem;

    padding: 1rem 0;
    margin-top: 1rem;
  }
}
</style>
