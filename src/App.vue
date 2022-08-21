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
<<<<<<< HEAD
// ボタンクリック時に実行するパターン
const runapi = (token) => {



=======
const linkedStatus: Ref<string> = ref<string>('');
const checkboxRepo = ref([])

// ボタンクリック時に実行するパターン
const linkWorkItem = (token, workItemId, checkboxRepo, repos) => {

  for (let repo_id of checkboxRepo) {

    for (let repo of repos) {
      if(repo.id == repo_id) {
        // test.value = repo.name;
        axios.patch('https://dev.azure.com/yoko1983/test/_apis/wit/workitems/' + workItemId, 
            [{ 
              'op': 'add',
              'path': '/relations/-',
              'value': {
                  'rel': 'ArtifactLink',
                  'url': 'vstfs:///Git/Ref/test/'+ repo.id+ '/GBmain',
                  'attributes': {
                      'comment': repo.name,
                      'name': 'Branch'
                  }
              }
            }],
            { 
                  auth: {
                    password: token
                  },
                  headers: { 
                    'Content-Type': 'application/json-patch+json'
                  },
                  params: {
                    'api-version':'5.1'
                  }
            })
          .then(response => linkedStatus.value = 'success')
          .catch(error => error)
          break;
      }

    }
    

  }

};



// ボタンクリック時に実行するパターン
const runapi = (token) => {

>>>>>>> 9a3e02a... improved work item control
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
        <keep-alive><input type="text" v-model="token"></keep-alive>
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
<<<<<<< HEAD
          :value="repo.name"
          v-model="selectedPokes"
=======
          :value="repo.id"
          v-model="checkboxRepo"
>>>>>>> 9a3e02a... improved work item control
        />
        <label :for="'repo.name' + i">{{repo.name}}</label>
        </li>
      </ul>
      </div>

      <div class='labelbox'>
        <label>WorkItem ID:</label>
      </div>
      <div class='tokenbox'>
<<<<<<< HEAD
        <input type="text" v-model="wiid">
        <button @click="linkWI(wiid)">
=======
        <input type="text" v-model="workItemId">
        <button @click="linkWorkItem(token, workItemId, checkboxRepo, results.value)">
>>>>>>> 9a3e02a... improved work item control
          Link
        </button>
        <div class='linkedStatusBox'>{{linkedStatus}}</div>

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

.linkedStatusBox {
  margin-left: 1em;
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
