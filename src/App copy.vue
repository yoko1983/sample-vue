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

const token = 'token';


// const headers = {
//        'Content-Type': 'application/json',
//        'Authorization': 'Basic id:token'
// };


const results = ref([])
// const results = ""
const error = ref([])
// ボタンクリック時に実行するパターン
const runapi = () => {
  axios.get('https://jsonplaceholder.typicode.com/users')
    .then(response => results.value = response.data)
    .catch(error => console.log(error))
};
// 画面遷移時に実行するパターン
onMounted(() => {
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
})

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

interface User {
  firstName: string;
  lastName: string;
  age: number;
}
// const user: User = reactive({
//   firstName: 'John',
//   lastName: 'Doe',
//   age: 25,
// });
const user: User = reactive({} as User);
user.firstName='John'
user.lastName='Doe'
user.age=25

const fullName: Ref<string> 
  = computed<string>(() =>`${user.firstName} ${user.lastName}`);

const changeName = ():void => {
  user.firstName = "Jane";
};

const changeName2: (name: string) => void = (name: string):void => {
  user.firstName = name;
};


// let users: []

// const getAds: () => void = ():void => {

//   axios.get('https://jsonplaceholder.typicode.com/users')
//             .then(response => users = response.data)
//             .catch(error => console.log(error))

// };


// changeName();

</script>


<template>
  <header>
    <img alt="Vue logo" class="logo" src="@/assets/logo.svg" width="125" height="125" />

    <div class="wrapper">
      <HelloWorld msg="You did it!" />

      <nav>
        <RouterLink to="/">Home</RouterLink>
        <RouterLink to="/about">About</RouterLink>
      </nav>
      <div>
        Count:{{ count }}
        <button @click="count++">
          Add Count
        </button>
      </div>

      <div>
        <button @click="runapi()">
          Run
        </button>
          <ul>
            <li v-for="repo in results.value">{{ repo.name }}</li>
          </ul>

          <!-- info:{{ results.value }} -->

          error:{{ error }}

          message:{{ message }}
      </div>


      <div>
        FullName: {{ fullName }}
        Count:{{ user.age }}
        <button @click="user.age++">
          Add Age
        </button>
        <button @click="changeName2('Taro')">
          ChangeName
        </button>
      </div>
      <div>
        <label><input type="checkbox" v-model="checkBoxState.aa" />aa</label>
        <label><input type="checkbox" v-model="checkBoxState.bb" />bb</label>
        <label><input type="checkbox" v-model="checkBoxState.cc" />cc</label>
        <label><input type="checkbox" v-model="checkBoxState.dd" />dd</label>
        <label><input type="checkbox" v-model="checkBoxState.ee" />ee</label>
        <label><input type="checkbox" v-model="checkBoxState.ff" />ff</label>
      </div>
      <button type="button" :disabled="!isButtonEnabled">
        送信
      </button>
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
