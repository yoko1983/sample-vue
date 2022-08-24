<script setup lang="ts">
import { ref } from 'vue';
import type { Ref } from 'vue';
import { onMounted } from "vue"
import axios from "axios"
import { useCookies } from "vue3-cookies";
import Title from '@/components/Title.vue'

interface WorkItem {
  id: string
  title: string 
}

interface Repo {
  id: string
  name: string
  diff: string
  status: number
}

const repos: Ref<Repo[]> = ref([])
const gettedRepoErrorStatus: Ref<string> = ref<string>('');
const debug: Ref<string> = ref<string>('');
const checkboxRepo = ref([])


const { cookies } = useCookies();

let url = '';
if (cookies.get('vue-ads-url') != null) {
  url = cookies.get('vue-ads-url');
}
let project = '';
if (cookies.get('vue-ads-project') != null) {
  project = cookies.get('vue-ads-project');
}
let token = '';
if (cookies.get('vue-ads-token') != null) {
  token = cookies.get('vue-ads-token');
}

const workItemId:string = '';

const getRepos = (workItemId: string) => {
  gettedRepoErrorStatus.value = '';
  getReposSingle(workItemId);



}

const getReposSingleWithAPI = (workItemId: string) => {
  return new Promise(async (resolve, reject) => {
    try {
      gettedRepoErrorStatus.value = '';
      repos.value = [];

      const response = await axios.get(url + '/' + project  + '/_apis/wit/workitems/' + workItemId, 
          { 
                auth: {
                  username: '',
                  password: token
                },
                params: {
                  '$expand': 'all',
                  'api-version':'5.1'
                },
                validateStatus: function (status) {
                  return status == 200;
                }
          })

      const workItem = response.data;
      for(let relation of workItem.relations) {
        const attributesName = relation.attributes.name;
        if(attributesName == 'Branch') {
          const repoUrl:string = relation.url;
          const urlBaseLen:number = "vstfs:///Git/Ref/".length;
          const decodeUrl:string = decodeURI(repoUrl);
          const urlPart:string = decodeUrl.substring(urlBaseLen);
          const urlParts:string[] = urlPart.split('/');

          const repo:Repo = {
            id:'',
            name:'',
            diff:'',
            status:0
          };

          repo.id = urlParts[1];
          repo.status = 0;
          repo.diff = 'diff';

          const responseGit = await axios.get(url + '/' + project  + '/_apis/git/repositories/' + repo.id, 
              { 
                    auth: {
                      username: '',
                      password: token
                    },
                    params: {
                      'api-version':'5.1'
                    },
                    validateStatus: function (status) {
                      return status == 200;
                    }
              })
            
          repo.name = responseGit.data.name;
          repos.value.push(repo);
        }
      }


      resolve(repos);
    } catch (error) {
      gettedRepoErrorStatus.value = 'Error';
      reject(error);
    }
  });

};

const getReposSingle = async (workItemId: string) => {

  try {
    const _repos = await getReposSingleWithAPI(workItemId);

    // for(repo of repos.value) {
    //   debug.value =repo.id;
    //   repo.name = getRepoName(repo.id);
    // }
  } catch(error) {

  }


}



const getRepoName = (repoId:string) => {

  gettedRepoErrorStatus.value = '';

  axios.get(url + '/' + project  + '/_apis/git/repositories/' + repoId, 
      { 
            auth: {
              username: '',
              password: token
            },
            params: {
              'api-version':'5.1'
            },
            validateStatus: function (status) {
              return status == 200;
            }
      })
    .then(function (response) {
      return response.data.name;
    })
    .catch(function (error) {
      gettedRepoErrorStatus.value = error;

    })
};


const setStatusColor = (status: number) => {
      if (status == 1) {
        return "blue";
      }
      if (status == 2) {
        return "green";
      }
      return "red";
}

const setDiffColor = (diff: string) => {
      if (diff == 'diff') {
        return "diff_blue";
      }
      if (diff == 'same') {
        return "diff_silver";
      }
      return "red";
}

const isDiff = (diff: string) => {
      if (diff == 'diff') {
        return true;
      }
      if (diff == 'same') {
        return false;
      }
      return false;
}


</script>



<template>


  <div class="main-grid ">
    <article>
      <Title title="Create pull requests!" description="You can create pull requests with work item."/>

      <div class='labelbox'>
        <label>WorkItem ID:</label>
      </div>

      <div class='inputbox'>
        <input type="text" v-model="workItemId">
      </div>

      <div class='inputbox'>
        <button @click="getRepos(workItemId)">
          Display
        </button>
        <div class='errorStatusbox'>{{gettedRepoErrorStatus}}</div>
      </div>

      <div class='labelbox'>
        <label>Repositories:</label>
      </div>

      <div v-for="repo in repos" class='repobox'>
        <ul>
          <li>
          <input
            :id="'repo.id'"
            type="checkbox"
            :value="repo.id"
            v-model="checkboxRepo"
            v-bind:disabled="repo.diff=='same'"
            :checked="isDiff(repo.diff)"
          />
          <label :for="'repo.id'">{{repo.name}}</label>
          <label :for="'repo.id'"  :class="setDiffColor(repo.diff)">{{repo.diff}}</label>
          <label :for="'repo.id'"  :class="setStatusColor(repo.status)">●</label>
          </li>
        </ul>
      </div>

      <div class='inputbox'>
        <button @click="getRepos(workItemId)">
          CreatePR
        </button>
        <div class='errorStatusbox'>{{gettedRepoErrorStatus}}</div>
        <div class='successStatusbox'>Debug: {{debug}}</div>
      </div>


</article>
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
.red {
  color: white;
}
.diff_silver {
  color: silver;
  font-size: 80%;
}
.diff_blue {
  color: blue;
  font-size: 80%;
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
    align-content: start;
  }

  aside {
    grid-area: left;
  }

  article {
    grid-area: main;
    -ms-grid-row-align: start;
    align-self: start;
  }
}


</style>
