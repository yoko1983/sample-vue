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
  prId: string
  status: number
}

const debug: Ref<string> = ref<string>('');

const branchTarget1:string = 'branch1';
const branchTarget2:string = 'branch2';
const branchTarget3:string = 'branch3';
const branchTarget4:string = 'master';
const branchSource1:string = 'test';
const branchSource2:string = 'branch1';
const branchSource3:string = 'branch2';
const branchSource4:string = 'branch3';

const repos: Ref<Repo[]> = ref([])
const getRepoErrorStatus: Ref<string> = ref<string>('');
const createPRErrorStatus: Ref<string> = ref<string>('');
const wiReposCheckbox = ref([]);
const radioboxBranch: Ref<string> = ref<string>('branch2');
const prTitle: Ref<string> = ref<string>('');
const prDescription: Ref<string> = ref<string>('');
const linkingWorkItemIds: Ref<string[]> = ref([]);

const wiReposAllCheckbox: Ref<boolean> = ref<boolean>(false);

  

const { cookies } = useCookies();

let url:string = cookies.get('vue-ads-url');
let project:string = cookies.get('vue-ads-project');
let token:string = cookies.get('vue-ads-token');

const workItemId: Ref<string> = ref<string>('');

const createPRSingleWithAPI = 
  (repo: Repo, 
    sourceBranch:string, 
    targetBranch:string, 
    workItemId: string, 
    prTitle: string, 
    prDescription: string, 
    linkingWorkItemIds: string[]) => {
  return new Promise(async (resolve, reject) => {

    let linkingWorkItemUrls = [];

    linkingWorkItemUrls.push(
      {
        "id": workItemId,
        "url": url + '/' + project  + "_apis/wit/workItems/" +workItemId 
      }
    )
    prTitle = prTitle + ' #' + workItemId;

    for(let linkingWorkItemId of linkingWorkItemIds) {
      if(linkingWorkItemId != '') {
        let wiUrl = {
          "id": linkingWorkItemId,
          "url": url + '/' + project  + "_apis/wit/workItems/" +linkingWorkItemId 
        }
        linkingWorkItemUrls.push(wiUrl);
        prTitle = prTitle + ' #' + linkingWorkItemId;
      }

    }

    const request = {
      "sourceRefName": "refs/heads/" + sourceBranch,
      "targetRefName": "refs/heads/" + targetBranch,
      "title": "[" + repo.name + " / " + targetBranch + "] " + prTitle,
      "description": prDescription,
      "workItemRefs": linkingWorkItemUrls
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

  let sourceBranch:string = '';
  let targetBranch:string = '';
  if(radioboxBranch.value == "branchTarget1") {
    sourceBranch = branchSource1,
    targetBranch = branchTarget1;
  }
  else if(radioboxBranch.value == "branchTarget2") {
    sourceBranch = branchSource2,
    targetBranch = branchTarget2;
  }
  else if(radioboxBranch.value == "branchTarget3") {
    sourceBranch = branchSource3,
    targetBranch = branchTarget3;
  }
  else if(radioboxBranch.value == "branchTarget4") {
    sourceBranch = branchSource4,
    targetBranch = branchTarget4;
  }

  for (let repoId of wiReposCheckbox.value) {

    for (let repo of repos.value) {

      if(repo.id == repoId) {
        try {
          const response:any = 
            await createPRSingleWithAPI(
                                  repo, 
                                  sourceBranch, 
                                  targetBranch, 
                                  workItemId, 
                                  prTitle.value, 
                                  prDescription.value,
                                  linkingWorkItemIds.value
                                  ); 

          const prUrl:string = response.data.artifactId;
          const urlBaseLen:number = "vstfs:///Git/PullRequestId/".length;
          const decodeUrl:string = decodeURIComponent(prUrl);
          const urlPart:string = decodeUrl.substring(urlBaseLen);
          const urlParts:string[] = urlPart.split('/');
          repo.prId=urlParts[2];
          repo.status=1;
        } catch(error) {
          createPRErrorStatus.value = String(error);
          repo.status=2;
        }
        break;
      }
    }
  }
  wiReposAllCheckbox.value =false;
  wiReposCheckbox.value.length=0;




}

const createPRs = (workItemId: string) => {
  createPRErrorStatus.value =  '';
  createPRsSingle(workItemId);

}



const getReposSingleWithAPI = (workItemId: string) => {
  return new Promise(async (resolve, reject) => {
    try {
      getRepoErrorStatus.value = '';
      const repos:Repo[] = [];

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
            prId:'',
            status:0
          };

          repos.push(repo);
        }
      }

      resolve(repos);

    } catch (error) {
      getRepoErrorStatus.value = "getRepos" +String(error)
      reject(error);
    }
  });

};

const setRepoNameForReposSingleWithAPI = (repos: Repo[] ) => {
  return new Promise(async (resolve, reject) => {
    try {
      for(let repo of repos) {

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
      repos = repos.sort(function(a:Repo, b:Repo) {
        return (a.name < b.name) ? -1 : 1;
      });

      resolve(repos);
    } catch (error) {
      getRepoErrorStatus.value = "setRepoNameForRepos" +String(error)
      reject(error);
    }
  });

};

const setDiffForReposSingleWithAPI = (repos:Repo[], sourceBranchName:string, targetBranchName:string, top:number) => {
  return new Promise(async (resolve, reject) => {
      for(let repo of repos) {
        try {

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

          if(responseGit.data.aheadCount==0) {
            repo.diff = 0;
          } else {
            repo.diff = 1;
          }
          resolve(repos);
        } catch (error) {
          getRepoErrorStatus.value = "setDiffForRepos" +String(error)
          reject(error);
        }

    }

  });

};

const setActivePRForReposSingleWithAPI = (repos:Repo[], sourceBranchName:string, targetBranchName:string) => {
  return new Promise(async (resolve, reject) => {
      for(let repo of repos) {
        try {

          const responseGit = await axios.get(url + '/' + project  + '/_apis/git/repositories/' + repo.id + '/pullrequests', 
              { 
                    auth: {
                      username: '',
                      password: token
                    },
                    params: {
                      'searchCriteria.status': 'active',
                      'searchCriteria.targetRefName': 'refs/heads/' + targetBranchName,
                      'searchCriteria.sourceRefName': 'refs/heads/' + sourceBranchName,
                      'api-version':'5.1'
                    },
                    validateStatus: function (status) {
                      return status == 200;
                    }
              })
          
          if(responseGit.data.count!=0) {
            repo.prId=responseGit.data.value[0].pullRequestId;
            repo.status = 1;

          }
          
          resolve(repos);

        } catch (error) {
          getRepoErrorStatus.value = "setActivePRForRepos" +String(error)
          reject(error);
        }

    }

  });

};


const getReposSingle = async (workItemId: string) => {

  try {
    repos.value = await getReposSingleWithAPI(workItemId) as Repo[];
    await setRepoNameForReposSingleWithAPI(repos.value);
    if(radioboxBranch.value == "branchTarget1") {
      await setDiffForReposSingleWithAPI(repos.value, branchSource1, branchTarget1, 1);
      await setActivePRForReposSingleWithAPI(repos.value, branchSource1, branchTarget1);
    }
    else if(radioboxBranch.value == "branchTarget2") {
      await setDiffForReposSingleWithAPI(repos.value, branchSource2, branchTarget2, 1);
      await setActivePRForReposSingleWithAPI(repos.value, branchSource2, branchTarget2);
    }
    else if(radioboxBranch.value == "branchTarget3") {
      await setDiffForReposSingleWithAPI(repos.value, branchSource3, branchTarget3, 1);
      await setActivePRForReposSingleWithAPI(repos.value, branchSource3, branchTarget3);
    }
    else if(radioboxBranch.value == "branchTarget4") {
      await setDiffForReposSingleWithAPI(repos.value, branchSource4, branchTarget4, 1);
      await setActivePRForReposSingleWithAPI(repos.value, branchSource4, branchTarget4);
    }


  } catch(error) {
      // getRepoErrorStatus.value = String(error);

  }


}

const getRepos = (workItemId: string) => {
  getRepoErrorStatus.value = '';
  getReposSingle(workItemId);
}

const selectAllForWiReposCheckbox = () => {
  wiReposAllCheckbox.value = !wiReposAllCheckbox.value;
  wiReposCheckbox.value.length=0;
  if(wiReposAllCheckbox.value) {
    for(let repo of repos.value) {
      if(repo.diff!=0 && repo.status!=1) {
        wiReposCheckbox.value.push(repo.id as never);

      }
    }
  }

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
          v-model="radioboxBranch"
          value="branchTarget1"
        />
        <label>{{branchTarget1}}</label>
        <input
          type="radio"
          v-model="radioboxBranch"
          value="branchTarget2"
        />
        <label>{{branchTarget2}}</label>
        <input
          type="radio"
          v-model="radioboxBranch"
          value="branchTarget3"
        />
        <label>{{branchTarget3}}</label>
        <input
          type="radio"
          v-model="radioboxBranch"
          value="branchTarget4"
        />
        <label>{{branchTarget4}}</label>
      </div>

      <div class='inputbox'>
        <button @click="getRepos(workItemId)">
          Display
        </button>
        <div class='errorStatusbox'>{{getRepoErrorStatus}}</div>
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

      <div class='labelbox'>
        <label>Pull Request Linking WorkItem IDs:</label>
      </div>

      <div class='inputboxMini'>
        <input type="text" v-model="linkingWorkItemIds[0]">
        <input type="text" v-model="linkingWorkItemIds[1]">
        <input type="text" v-model="linkingWorkItemIds[2]">
        <input type="text" v-model="linkingWorkItemIds[3]">
        <input type="text" v-model="linkingWorkItemIds[4]">
      </div>

      <div class='inputbox'>
        <button @click="createPRs(workItemId); ">
          CreatePR
        </button>
        <div class='errorStatusbox'>{{createPRErrorStatus}}</div>
        <div class='successStatusbox'>{{debug}}</div>
      </div>

    </aside>
    <article>
      <div class='labelbox'>
        <label>Repositories:</label>
      </div>

      <div class='repobox'>
        <ul>
          <li>
            <input
              type="checkbox"
              v-model="wiReposAllCheckbox"
              @click="selectAllForWiReposCheckbox();"
            />
            <div class="repoNameBox">
              <label class="title">RepoName</label>
            </div>
            <div class="repoDiffBox">
              <label class="title">BranchDiff</label>
            </div>
            <div class="repoPRBox">
              <label class="title">PR</label>
            </div>
          </li>
        </ul>
      </div>      

      <div v-for="repo in repos" class='repobox'>
        <ul>
          <li>
          <input
            :id="'repo.id'"
            type="checkbox"
            :value="repo.id"
            v-model="wiReposCheckbox"
            v-bind:disabled="repo.diff==0 || repo.status==1"
          />
          <div class="repoNameBox">
            <label :for="'repo.id'">{{repo.name}}</label>
          </div>
          <div class="repoDiffBox">
            <label :for="'repo.id'"  :class="setDiffColor(repo.diff)">
              {{setDiffText(repo.diff)}}
            </label>
          </div>
          <div class="repoPRBox">
            <label :for="'repo.id'"  :class="setStatusColor(repo.status)" >
              <div class="prStatusBox" v-show="repo.status==1">
                <a :href="url + project + '/_git/' + repo.name + '/pullrequest/' + repo.prId + '?_a=overview'" target="_blank">
                  {{repo.prId}}
                </a>
              </div>
              <div class="prStatusBox" v-show="repo.status==2">
                NG
              </div>
            </label>
          </div>
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

.inputboxMini {
	width:500px;
	padding:5px;
}
.inputboxMini input {
  margin-left: 1em;
	width:60px;
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
	width:900px;
  list-style: none;
}
.repobox ul li {
	width:900px;
	margin-bottom:5px;
	padding-left:1em;
	text-indent:-1em;
	line-height:1.4;
}
.repobox ul li label {
	width:900px;
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

.prStatusBox a:hover {
    background-color: hsla(160, 100%, 37%, 0.2);
}


.repoNameBox {
  display: inline-block; 
	width:300px;
	padding-left:1em;
}

.repoNameBox label {
	padding-left:1em;
}

.repoDiffBox {
  display: inline-block; 
	width:100px;
	padding-left:1em;
}

.repoDiffBox label {
	padding-left:1em;
}

.repoPRBox {
  display: inline-block; 
	width:10px;
	padding-left:1em;
}

.repoPRBox label {
	padding-left:1em;
}

.title {
  color: gray;
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
