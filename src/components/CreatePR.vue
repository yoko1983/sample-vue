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
  diff: number
  status: number
}

const branchTarget1: Ref<string> = ref<string>('branch1');
const branchTarget2: Ref<string> = ref<string>('branch2');
const branchTarget3: Ref<string> = ref<string>('branch3');
const branchTarget4: Ref<string> = ref<string>('master');
const branchSource1: Ref<string> = ref<string>('branch1');
const branchSource2: Ref<string> = ref<string>('branch2');
const branchSource3: Ref<string> = ref<string>('branch3');
const branchSource4: Ref<string> = ref<string>('mut');

const repos: Ref<Repo[]> = ref([])
const gettedRepoErrorStatus: Ref<string> = ref<string>('');
const debug: Ref<string> = ref<string>('');
const checkboxRepo = ref([])
const radioboxBranch = ref('branchTarget2')
const prTitle: Ref<string> = ref<string>('');
const prDescription: Ref<string> = ref<string>('');



const { cookies } = useCookies();

let url:string = cookies.get('vue-ads-url');
let project:string = cookies.get('vue-ads-project');
let token:string = cookies.get('vue-ads-token');

const workItemId: Ref<string> = ref<string>('');

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
          const decodeUrl:string = decodeURIComponent(repoUrl);
          const urlPart:string = decodeUrl.substring(urlBaseLen);
          const urlParts:string[] = urlPart.split('/');

          const repo:Repo = {
            id:urlParts[1],
            name:'',
            diff:0,
            status:0
          };

          repos.value.push(repo);
        }
      }

      resolve(repos.value);

    } catch (error) {
      reject(error);
    }
  });

};

const setRepoNameForReposSingleWithAPI = () => {
  return new Promise(async (resolve, reject) => {
    try {
      for(let repo of repos.value) {

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

      resolve(repos.value);
    } catch (error) {
      reject(error);
    }
  });

};

const setDiffForReposSingleWithAPI = (sourceBranchName:string, targetBranchName:string, top:number) => {
  return new Promise(async (resolve, reject) => {
    try {
      for(let repo of repos.value) {

        const responseGit = await axios.get(url + '/' + project  + '/_apis/git/repositories/' + repo.id + '/diffs/commits', 
            { 
                  auth: {
                    username: '',
                    password: token
                  },
                  params: {
                    '$top': top,
                    'baseVersion': targetBranchName,
                    'targetVersion': sourceBranchName,
                    'api-version':'5.1'
                  },
                  validateStatus: function (status) {
                    return status == 200;
                  }
            })
          
        repo.diff = responseGit.data.aheadCount;
      }

      resolve(repos.value);
    } catch (error) {
      reject(error);
    }
  });

};



const getReposSingle = async (workItemId: string) => {

  try {
    await getReposSingleWithAPI(workItemId);
    await setRepoNameForReposSingleWithAPI();
    await setDiffForReposSingleWithAPI('mut','master',1);

    debug.value = radioboxBranch;

  } catch(error) {
      gettedRepoErrorStatus.value = String(error);

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

const setDiffText = (diff: number) => {
      if (diff > 0) {
        return "diff";
      } else {
        return "same";
      }

}

const setDiffColor = (diff: number) => {
      if (diff > 0) {
        return "diff_blue";
      } else {
        return "diff_silver";
      }

}



</script>



<template>


  <div class="main-grid ">
    <header>
      <Title title="Create pull requests!" description="You can create pull requests with work item."/>
    </header>
    <aside>

      <div class='labelbox'>
        <label>WorkItem ID:</label>
      </div>

      <div class='inputbox'>
        <input type="text" v-model="workItemId">
      </div>

      <div class='labelbox'>
        <label>Target Branch:</label>
      </div>

      <div class='radiobox'>
        <input
          type="radio"
          value="master"
          v-model="radioboxBranch"
          :checked=false
        />
        <label>{{branchTarget1}}</label>
        <input
          type="radio"
          value="branchTarget2"
          v-model="radioboxBranch"
          :checked=true
        />
        <label>{{branchTarget2}}</label>
        <input
          type="radio"
          value="test2"
          v-model="radioboxBranch"
          :checked=false
        />
        <label>{{branchTarget3}}</label>
        <input
          type="radio"
          value="test3"
          v-model="radioboxBranch"
          :checked=false
        />
        <label>{{branchTarget4}}</label>
      </div>

      <div class='inputbox'>
        <button @click="getRepos(workItemId)">
          Display
        </button>
        <div class='errorStatusbox'>{{gettedRepoErrorStatus}}</div>
      </div>

      <div class='labelbox'>
        <label>Pull Request Title:</label>
      </div>

      <div class='inputbox'>
        <input type="text" v-model="prTitle">
      </div>

      <div class='labelbox'>
        <label>Pull Request Description:</label>
      </div>

      <div class='inputbox'>
        <input type="text" v-model="prDescription">
      </div>

      <div class='inputbox'>
        <button @click="getRepos(workItemId)">
          CreatePR
        </button>
        <div class='errorStatusbox'>{{gettedRepoErrorStatus}}</div>
        <div class='successStatusbox'>{{debug}}</div>
      </div>

    </aside>
    <article>
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
            v-bind:disabled="repo.diff==0"
            :checked="repo.diff>0"
          />
          <label :for="'repo.id'">{{repo.name}}</label>
          <label :for="'repo.id'"  :class="setDiffColor(repo.diff)">{{setDiffText(repo.diff)}}</label>
          <label :for="'repo.id'"  :class="setStatusColor(repo.status)">●</label>
          </li>
        </ul>
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
.radiobox {
  margin-left: 10px
}
.radiobox input {
  margin-left: 15px
}
.radiobox label {
  margin-left: 5px
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
