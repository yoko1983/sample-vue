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
  diff: number
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

const repos: Ref<Repo[]> = ref([]);
const linkedRepos: Ref<Repo[]> = ref([]);
const gettedRepoErrorStatus: Ref<string> = ref<string>('');
const linkedSuccessStatus: Ref<string> = ref<string>('');
const linkedErrorStatus: Ref<string> = ref<string>('');
const checkboxRepo = ref([])
const workItemId: Ref<string> = ref<string>('');
const workItemTitle: Ref<string> = ref<string>('');
const debug: Ref<string> = ref<string>('');

const linkWorkItem = (workItemId: string) => {
  linkedSuccessStatus.value = '';
  linkedErrorStatus.value =  '';
  linkWorkItemSingle(workItemId);



}

const linkWorkItemSingleWithAPI = (projectId:string, workItemId: string, repo: Repo) => {
  return new Promise(async (resolve, reject) => {

    const repoUrl:string = 'vstfs:///Git/Ref/' + encodeURIComponent(projectId + '/' + repo.id+ '/GBmaster');
    try {
      const response = await axios.patch(url + '/' + project  + '/_apis/wit/workitems/' + workItemId, 
          [{ 
            'op': 'add',
            'path': '/relations/-',
            'value': {
                'rel': 'ArtifactLink',
                'url': repoUrl,
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

const getProjectIdSingleWithAPI = () => {
  return new Promise(async (resolve, reject) => {
    try {

      const response = await axios.get(url + '/_apis/projects/' + project, 
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

      const id:string = response.data.id;
      resolve(id);
    } catch (error) {
      reject(error);
      
    }
  });

};


const getLinkedReposSingleWithAPI = (workItemId: string) => {
  return new Promise(async (resolve, reject) => {
    try {
      linkedRepos.value = [];

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

      workItemTitle.value = workItem.fields['System.Title'];
      for(let relation of workItem.relations) {
        const attributesName = relation.attributes.name;
        if(attributesName == 'Branch') {
          const repoUrl:string = relation.url;
          const urlBaseLen:number = "vstfs:///Git/Ref/".length;
          const decodeUrl:string = decodeURIComponent(repoUrl);
          const urlPart:string = decodeUrl.substring(urlBaseLen);
          const urlParts:string[] = urlPart.split('/');

          const repo:Repo = {
            id:urlParts[1],
            name:'',
            diff:0,
            status:0
          };

          linkedRepos.value.push(repo);
        }
      }

      //リポジトリ一覧を昇順ソートする
      let sortedResults = linkedRepos.value.sort(function(a:Repo, b:Repo) {
        return (a.name < b.name) ? -1 : 1;
      });
      linkedRepos.value = sortedResults;

      resolve(linkedRepos.value);

    } catch (error) {
      reject(error);
    }
  });

};

const setLinkedRepoNameForReposSingleWithAPI = () => {
  return new Promise(async (resolve, reject) => {
    try {
      for(let repo of linkedRepos.value) {

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
      }

      resolve(linkedRepos.value);
    } catch (error) {
      reject(error);
    }
  });

};


// ボタンクリック時に実行するパターン
const linkWorkItemSingle = async (workItemId: string) => {

  let errorCount:number = 0;

  for (let repoId of checkboxRepo.value) {

    for (let repo of repos.value) {
      if(repo.id == repoId) {

        try {
          const projectId:string = String(await getProjectIdSingleWithAPI()); 
          const response = await linkWorkItemSingleWithAPI(projectId, workItemId, repo); 
          repo.status = 1;
        } catch (error) {
          errorCount++;
          linkedErrorStatus.value = String(error);
          repo.status = 2;
        }
      }
    }
  }

  try {
    await getLinkedReposSingleWithAPI(workItemId);
    await setLinkedRepoNameForReposSingleWithAPI();
  } catch (error) {
    errorCount++;
    linkedErrorStatus.value = String(error);
  }

  if(errorCount == 0) {
    linkedSuccessStatus.value = 'Success';
  }
 


}

// ボタンクリック時に実行するパターン
const getRepos = () => {

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
  getRepos();
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

    <header>
      <Title title="Link repositories!" description="You can link repositories to work item."/>
    </header>
    <aside>

      <div class='labelbox'>
        <label>Repositories:</label>
        {{debug}}
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
        <button @click="linkWorkItem(workItemId)">
          Link
        </button>
        <div class='statusbox'>{{linkedSuccessStatus}}</div>
        <div class='errorStatusbox'>{{linkedErrorStatus}}</div>
      </div>

    </aside>
    <article>
      <div class='labelbox'>
        <label>Linked Repositories [ {{workItemId}}  - {{workItemTitle}}  ] :</label>
        {{debug}}
      </div>

      <div v-for="repo in linkedRepos" class='linkedrepobox'>
        <ul>
          <li>
          <!-- <input
            :id="'repo.id'"
            type="checkbox"
            :value="repo.id"
            v-model="checkboxRepo"
          /> -->
          <label :for="'repo.name'">{{repo.name}}</label>
          </li>
        </ul>
      </div>
    </article>
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

  header {
    grid-area: header;
  }

  aside {
      grid-area: left;
  }

  article {
      grid-area: main;
  }
}


</style>
