<!-- <script type="module" src="./sample.js"></script> -->

<script setup lang="ts">
import { ref } from 'vue';
import type { Ref } from 'vue';
import { onMounted } from "vue"
import axios from "axios"
import { useCookies } from "vue3-cookies";
import Title from '@/components/Title.vue'

interface Repo {
  id: string
  name: string
  status: number
}

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

const repos: Ref<Repo[]> = ref([])
const gettedRepoErrorStatus: Ref<string> = ref<string>('');
const linkedSuccessStatus: Ref<string> = ref<string>('');
const linkedErrorStatus: Ref<string> = ref<string>('');
const linkedSuccessRepos: Ref<string> = ref<string>('');
const linkedErrorRepos: Ref<string> = ref<string>('');
const checkboxRepo = ref([])
const workItemId: Ref<string> = ref<string>('');

const linkWorkItem = (url: string, project: string ,token: string, workItemId: string, checkboxRepo: any, repos: any) => {
  linkedSuccessStatus.value = '';
  linkedErrorStatus.value =  '';
  linkedSuccessRepos.value = '';
  linkedErrorRepos.value = '';
  linkWorkItemSingle(url, project, token, workItemId, checkboxRepo, repos);



}

const linkWorkItemSingleWithAPI = (url: string, project: string ,token: string, workItemId: string, repo: Repo) => {
  return new Promise(async (resolve, reject) => {
    try {
      const response = await axios.patch(url + '/' + project  + '/_apis/wit/workitems/' + workItemId, 
          [{ 
            'op': 'add',
            'path': '/relations/-',
            'value': {
                'rel': 'ArtifactLink',
                'url': 'vstfs:///Git/Ref/test/'+ repo.id+ '/GBmaster',
                'attributes': {
                    'comment': repo.name,
                    'name': 'Branch'
                }
            }
          }],
          { 
                auth: {
                  username: '',
                  password: token
                },
                headers: { 
                  'Content-Type': 'application/json-patch+json'
                },
                params: {
                  'api-version':'5.1'
                },
                validateStatus: function (status) {
                  return status == 200;
                }
          })

      resolve(response);
    } catch (error) {
      reject(error);
    }
  });

};



// ボタンクリック時に実行するパターン
const linkWorkItemSingle = async (url: string, project: string ,token: string, workItemId: string, checkboxRepo: any, repos: any) => {


  for (let repoId of checkboxRepo) {


    for (let repo of repos) {
      if(repo.id == repoId) {

        try {
          const res = await linkWorkItemSingleWithAPI(url, project, token, workItemId, repo); 
          linkedSuccessRepos.value = repo.name + ','  + linkedSuccessRepos.value;
          repo.status = 1;
        } catch (error) {
          // if(linkedErrorRepos.value == '') {
          //   linkedErrorRepos.value = 'ErrorRepos: '
          // }
          linkedErrorRepos.value = repo.name + ','  + linkedErrorRepos.value;
          linkedErrorStatus.value = String(error);
          repo.status = 2;
        }
      }
    }
  }

  if(linkedErrorRepos.value == '') {
    linkedSuccessStatus.value = 'Success';
    linkedSuccessRepos.value = '';
    linkedErrorRepos.value = '';
    cookies.set('vue-ads-url', url);
    cookies.set('vue-ads-project', project);
    cookies.set('vue-ads-token', token);
 } else {
    linkedSuccessRepos.value = 'SuccessRepos: ' + linkedSuccessRepos.value.substring(0, linkedSuccessRepos.value.length-1);
    linkedErrorRepos.value = 'ErrorRepos: ' + linkedErrorRepos.value.substring(0, linkedErrorRepos.value.length-1);
 }
 


}

// ボタンクリック時に実行するパターン
const getRepos = (url :string, project :string, token: string) => {

  gettedRepoErrorStatus.value = '';

  axios.get(url + '/' + project  + '/_apis/git/repositories', 
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
      repos.value = response.data.value;
      //リポジトリ一覧を昇順ソートする
      let sortedResults = repos.value.sort(function(a:Repo, b:Repo) {
        return (a.name < b.name) ? -1 : 1;
      });

      for(let repo of repos.value) {
        repo.status = 0;
      }

      repos.value = sortedResults;

      cookies.set('vue-ads-url', url);
      cookies.set('vue-ads-project', project);
      cookies.set('vue-ads-token', token);
    })
    .catch(function (error) {
      gettedRepoErrorStatus.value = error;

    })

};

// 画面遷移時に実行するパターン
onMounted(() => {
  getRepos(url, project, token);
})
const setStatusColor = (status: number) => {
      if (status == 1) {
        return "blue";
      }
      if (status == 2) {
        return "green";
      }
      return "red";
}

</script>


<template>


  <div class="main-grid ">
<!--
      <aside>
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
          <label>Token:</label>
        </div>

        <div class='inputbox'>
          <input type="password" v-model="token">
        </div>

        <div class='inputbox'>
          <button @click="getRepos(url, project, token)">
            Display
          </button>
          <div class='errorStatusbox'>{{gettedRepoErrorStatus}}</div>
        </div>

      </aside>
-->
    <article>
        <Title title="Link repositories!" description="You can link repositories to work item."/>

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
          />
          <label :for="'repo.name'">{{repo.name}}</label>
          <label :for="'repo.name'"  :class="setStatusColor(repo.status)">●</label>
          </li>
        </ul>
      </div>


      <div class='labelbox'>
        <label>WorkItem ID:</label>
      </div>
      <div class='inputbox'>
        <input type="text" v-model="workItemId">
      </div>
      <div class='inputbox'>
        <button @click="linkWorkItem(url, project, token, workItemId, checkboxRepo, repos)">
          Link
        </button>
        <div class='statusbox'>{{linkedSuccessStatus}}</div>
        <div class='errorStatusbox'>{{linkedErrorStatus}}</div>
        <div class='statusbox'>{{linkedSuccessRepos}}</div>
        <div class='errorStatusbox'>{{linkedErrorRepos}}</div>
      </div>

    </article>
    <!-- <TokenFrom /> -->
    <!-- <RouterView /> -->
  </div>
</template>


<style scoped>

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
