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

interface PR {
  id: string
  repoId: string
  repoName: string
  url: string
  title: string
  description: string
  status: string
  mergeStatus: string
  soruceBranch: string
  targetBranch: string
}

const debug: Ref<string> = ref<string>('');

const prs: Ref<PR[]> = ref([])
const getPRErrorStatus: Ref<string> = ref<string>('');
const updatePRErrorStatus: Ref<string> = ref<string>('');
const wiPRsCheckbox = ref([]);

const { cookies } = useCookies();

let url:string = cookies.get('vue-ads-url');
let project:string = cookies.get('vue-ads-project');
let token:string = cookies.get('vue-ads-token');

const workItemId: Ref<string> = ref<string>('');

const updatePRSingleWithAPI = 
  (repo: PR, workItemId: string) => {
  return new Promise(async (resolve, reject) => {

    const request = {
      "workItemRefs": [
          {
            "id": workItemId,
            "url": url + '/' + project  + "_apis/wit/workItems/" +workItemId 
          }
        ]
      };


    try {
      const response = await axios.post(url + '/' + project  + '/_apis/git/repositories/' + repo.id +'/pullrequests',
          request, 
          { 
                auth: {
                  username: '',
                  password: token
                },
                headers: { 
                  'Content-Type': 'application/json'
                },
                params: {
                  'api-version':'5.1'
                },
                validateStatus: function (status) {
                  return status == 201;
                }
          })

      resolve(response);
    } catch (error) {
      reject(error);
    }
  });

};

const createPRsSingle = async (workItemId: string) => {

  for (let repoId of wiPRsCheckbox.value) {

    for (let repo of prs.value) {

      if(repo.id == repoId) {
        try {
          const response:any = 
            await updatePRSingleWithAPI(repo, workItemId); 

          const prUrl:string = response.data.artifactId;
          const urlBaseLen:number = "vstfs:///Git/PullRequestId/".length;
          const decodeUrl:string = decodeURIComponent(prUrl);
          const urlPart:string = decodeUrl.substring(urlBaseLen);
          const urlParts:string[] = urlPart.split('/');
          repo.prId=urlParts[2];
          repo.status=1;
        } catch(error) {
          updatePRErrorStatus.value = String(error);
          repo.status=2;
        }
        break;
      }
    }
  }




}

const createPRs = (workItemId: string) => {
  updatePRErrorStatus.value =  '';
  createPRsSingle(workItemId);

}


const approvePRSingleWithAPI = (pr: PR) => {
  return new Promise(async (resolve, reject) => {

    try {
      const response = await axios.patch(url + '/' + project  + '/_apis/git/repositories/' + pr.repoId +'/pullrequests/' +pr.id, 
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

// ボタンクリック時に実行するパターン
const approvePRsSingle = async () => {

  try {

    for (let prId of wiPRsCheckbox.value) {
      for (let pr of prs.value) {
        if(prId==pr.id) {
          const response = await approvePRSingleWithAPI(pr); 
          break;
        }
      }
    }

  } catch (error) {
    updatePRErrorStatus.value = String(error);
  }

}

const approvePRs = () => {
  updatePRErrorStatus.value =  '';
  approvePRsSingle();

}


const getPRsSingleWithAPI = (workItemId: string) => {
  return new Promise(async (resolve, reject) => {
    try {
      getPRErrorStatus.value = '';
      prs.value = [];

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
        if(attributesName == 'Pull Request') {
          const repoUrl:string = relation.url;
          const urlBaseLen:number = "vstfs:///Git/Ref/".length;
          const decodeUrl:string = decodeURIComponent(repoUrl);
          const urlPart:string = decodeUrl.substring(urlBaseLen);
          const urlParts:string[] = urlPart.split('/');

          const pr:PR = {
            id:urlParts[3],
            repoId:urlParts[2],
            repoName:'',
            url:'',
            title:'',
            description:'',
            status:'',
            mergeStatus:'',
            soruceBranch:'',
            targetBranch:''
          };

          prs.value.push(pr);
        }
      }

      resolve(prs.value);

    } catch (error) {
      reject(error);
    }
  });

};

const setRepoNameForReposSingleWithAPI = () => {
  return new Promise(async (resolve, reject) => {
    try {
      for(let pr of prs.value) {

        const responseGit = await axios.get(url + '/' + project  + '/_apis/git/repositories/' + pr.repoId, 
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
          
        pr.repoName = responseGit.data.name;
      }
      prs.value = prs.value.sort(function(a:PR, b:PR) {
        return (a.repoName < b.repoName) ? -1 : 1;
      });

      resolve(prs.value);
    } catch (error) {
      reject(error);
    }
  });

};

const setPRDitailsSingleWithAPI = () => {
  return new Promise(async (resolve, reject) => {
      for(let pr of prs.value) {
        try {

          const responseGit = await axios.get(url + '/' + project  + '/_apis/git/repositories/' + pr.repoId + '/pullrequests/' + pr.id, 
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
          const targetBranchParts:string[] = responseGit.data.targetRefName.split('/');
          const sourceBranchParts:string[] = responseGit.data.sourceRefName.split('/');
          pr.status = responseGit.data.status;
          pr.mergeStatus = responseGit.data.mergeStatus;
          pr.title = responseGit.data.title;
          pr.description = responseGit.data.description;
          pr.soruceBranch = sourceBranchParts[2];
          pr.targetBranch = targetBranchParts[2];
          resolve(prs.value);
        } catch (error) {
          reject(error);
          
        }

    }


  });

};



const getPRsSingle = async (workItemId: string) => {

  try {
    await getPRsSingleWithAPI(workItemId);
    await setRepoNameForReposSingleWithAPI();
    await setPRDitailsSingleWithAPI();


  } catch(error) {
      getPRErrorStatus.value = String(error);

  }


}

const getPRs = (workItemId: string) => {
  getPRErrorStatus.value = '';
  getPRsSingle(workItemId);
}



const setStatusColor = (status: number) => {
      if (status == 1) {
        return "blue";
      }
      if (status == 2) {
        return "red";
      }
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
      <Title title="Merge pull requests!" description="You can merge pull requests with work item."/>
    </header>
    <aside>

      <div class='labelbox'>
        <label>WorkItem ID:</label>
      </div>

      <div class='inputbox'>
        <input type="text" v-model="workItemId">
      </div>

      <div class='inputbox'>
        <button @click="getPRs(workItemId)">
          Display
        </button>
        <div class='errorStatusbox'>{{getPRErrorStatus}}</div>
      </div>

    </aside>
    <article>
      <div class='labelbox'>
        <label>Pull Requests:</label>
      </div>

      <div v-for="pr in prs" class='repobox'>
        <ul>
          <li>
          <input
            :id="'repo.id'"
            type="checkbox"
            :value="pr.id"
            v-model="wiPRsCheckbox"
            v-bind:disabled="pr.status=='0'"
          />
          <label :for="'pr.id'">{{pr.repoName}}</label>
          <label :for="'pr.id'">{{pr.id}}</label>
          <label :for="'pr.id'">{{pr.status}}</label>
          <label :for="'pr.id'">{{pr.mergeStatus}}</label>
          <label :for="'pr.id'">{{pr.title}}</label>
          <label :for="'pr.id'">{{pr.targetBranch}}</label>
          <label :for="'pr.id'">{{pr.soruceBranch}}</label>
          <label :for="'pr.id'"  :class="setDiffColor(pr.diff)">
            {{setDiffText(pr.diff)}}
          </label>
          <label :for="'pr.id'"  :class="setStatusColor(pr.status)" >
            <div class="prStatusBox" v-show="pr.status=='1'">
              <a :href="url + project + '/_git/app/pullrequest/' + pr.id + '?_a=overview'" target="_blank">
                {{pr.id}}
              </a>
            </div>
            <div class="prStatusBox" v-show="pr.status=='2'">
              NG
            </div>
          </label>
          </li>
        </ul>
      </div>

      <div class='inputbox'>
        <button @click="approvePRs(); ">
          Approve
        </button>
        <button @click="createPRs(workItemId); ">
          Merge
        </button>
        <div class='errorStatusbox'>{{updatePRErrorStatus}}</div>
        <div class='successStatusbox'>{{debug}}</div>
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

.prStatusBox {
  margin-left: 10px;
  display: inline-block; 
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
  font-size: 90%;
}
.red {
  color: red;
  font-size: 80%;
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
