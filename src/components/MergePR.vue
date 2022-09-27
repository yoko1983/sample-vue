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
  vote: number
  lastMergeSourceCommitId: string
  lastMergeTargetCommitId: string
}

const debug: Ref<string> = ref<string>('');

const prs: Ref<PR[]> = ref([])
const getPRErrorStatus: Ref<string> = ref<string>('');
const updatePRErrorStatus: Ref<string> = ref<string>('');
const wiPRsCheckbox = ref([]);

const { cookies } = useCookies();

let url:string = cookies.get('vue-ads-url');
let project:string = cookies.get('vue-ads-project');
let userId:string = cookies.get('vue-ads-userId');
let token:string = cookies.get('vue-ads-token');

const workItemId: Ref<string> = ref<string>('');

const reviewerId: Ref<string> = ref<string>('');


const updatePRSingleWithAPI = 
  (pr: PR, workItemId: string) => {
  return new Promise(async (resolve, reject) => {

    const request = {
      'status': 'completed',
      'lastMergeSourceCommit' : {
        'commitId': pr.lastMergeSourceCommitId,
      }
    };


    try {
      const response = await axios.patch(url + '/' + project  + '/_apis/git/repositories/' + pr.repoId +'/pullrequests/' + pr.id,
          request, 
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

    // const request = {
    //   'comment': 'Merged PR ' + pr.id + ': ' + pr.title,
    //   'parents': [pr.lastMergeSourceCommitId, pr.lastMergeTargetCommitId]
    // };


    // try {
    //   const response = await axios.post(url + '/' + project  + '/_apis/git/repositories/' + pr.repoId +'/merges',
    //       request, 
    //       { 
    //             auth: {
    //               username: '',
    //               password: token
    //             },
    //             headers: { 
    //               'Content-Type': 'application/json'
    //             },
    //             params: {
    //               'api-version':'5.1'
    //             },
    //             validateStatus: function (status) {
    //               return status == 201;
    //             }
    //       })

    //   resolve(response);
    // } catch (error) {
    //   reject(error);
    // }
  });

};

const updatePRsSingle = async (workItemId: string) => {

  for (let prId of wiPRsCheckbox.value) {

    for (let pr of prs.value) {

      if(pr.id == prId) {
        try {
          const response:any = 
            await updatePRSingleWithAPI(pr, workItemId); 
          debug.value = response.data;

          // const prUrl:string = response.data.artifactId;
          // const urlBaseLen:number = "vstfs:///Git/PullRequestId/".length;
          // const decodeUrl:string = decodeURIComponent(prUrl);
          // const urlPart:string = decodeUrl.substring(urlBaseLen);
          // const urlParts:string[] = urlPart.split('/');
          // repo.prId=urlParts[2];
          // repo.status=1;
        } catch(error) {
          updatePRErrorStatus.value = String(error);
          // repo.status=2;
        }
        break;
      }
    }
  }




}

const updatePRs = (workItemId: string) => {
  updatePRErrorStatus.value =  '';
  updatePRsSingle(workItemId);

}


const approvePRSingleWithAPI = (pr: PR, reviewerId: string) => {
  return new Promise(async (resolve, reject) => {

    try {

      const response = 
        await axios.put(
          url + '/' + project  + '/_apis/git/repositories/' + pr.repoId +'/pullrequests/' +pr.id + '/reviewers/' + reviewerId, 
          { 
            'vote': '10'
          },
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
                  return status == 200;
                }
          })

      resolve(response);
    } catch (error) {
      reject(error);
    }
  });

};

const approvePRsSingle = async () => {

  try {
    await getReviewerIdSingleWithAPI();

    for (let prId of wiPRsCheckbox.value) {
      for (let pr of prs.value) {
        if(prId==pr.id) {
          const response = await approvePRSingleWithAPI(pr, reviewerId.value); 
          break;
        }
      }
    }

  } catch (error) {
    updatePRErrorStatus.value = String(error);
  }

}

const getReviewerIdSingleWithAPI = () => {
  return new Promise(async (resolve, reject) => {
    try {

      let idUrl = url;

      if(idUrl.startsWith('https://dev.azure.com')) {
        idUrl = idUrl.replace('https://dev.azure.com', 'https://vssps.dev.azure.com');
      }

      // const response = await axios.get(idUrl  + '/_apis/identities', 
      const response = await axios.get(idUrl  + '/_apis/identities', 
          { 
                auth: {
                  username: '',
                  password: token
                },
                params: {
                  'searchFilter': 'General',
                  'filterValue': userId,
                  'queryMembership': 'None',
                  'api-version':'5.1'
                },
                validateStatus: function (status) {
                  return status == 200;
                }
          })

      if(response.data.count==1) {
        reviewerId.value = response.data.value[0].id;

      } else {
        reject('User id is not correct.');

      }
      resolve(response);

    } catch (error) {
      reject(error);
    }
  });

};


const approvePRs = () => {
  updatePRErrorStatus.value =  '';
  approvePRsSingle();
  getPRsSingle(workItemId.value);

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
            targetBranch:'',
            vote:0,
            lastMergeSourceCommitId:'',
            lastMergeTargetCommitId:''
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

          if(responseGit.data.reviewers != null) {
            let reviewerUrls:string[] = [];
            for(let reviewer of responseGit.data.reviewers) {
              let isVoted: boolean = true;
              for(let reviewerUrl of reviewerUrls) {
                if(reviewer.reviewerUrl == reviewerUrl) {
                  isVoted = false;
                }
              }
              if(isVoted) {
                pr.vote += reviewer.vote;
              }
              reviewerUrls.push(reviewer.url);
            }
          }

          pr.lastMergeSourceCommitId = responseGit.data.lastMergeSourceCommit.commitId;
          pr.lastMergeTargetCommitId = responseGit.data.lastMergeTargetCommit.commitId;


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



const setStatusColor = (status: string, vote: number) => {
  if (status == 'active' && vote>0) {
        return 'green';
  }
  else if (status == 'active' && vote==0) {
        return 'blue';
  }
  else if (status == 'active' && vote<0) {
        return 'red';
  }
  else {
    return 'silver';
  }
}

const getStatus = (status: string, vote: number) => {
  if (status == 'active' && vote>0) {
        return 'approved(' + vote + ')';
  }
  else if (status == 'active' && vote==0) {
        return 'active';
  }
  else if (status == 'active' && vote<0) {
        return 'rejected';
  }
  else {
    return status;
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
            v-bind:disabled="pr.status!='active'"
          />
          <label :for="'pr.id'">{{pr.repoName}}</label>
          <label :for="'pr.id'" :class="setStatusColor(pr.status, pr.vote)">{{getStatus(pr.status, pr.vote)}}</label>
          <label :for="'pr.id'">
            <a :href="url + project + '/_git/'+ pr.repoName +'/pullrequest/' + pr.id + '?_a=overview'" target="_blank">
                {{pr.id}}
            </a>
          </label>
          <label :for="'pr.id'" class='smallFont'>({{pr.soruceBranch}} -> {{pr.targetBranch}})</label>
          <!-- <label :for="'pr.id'"  :class="setDiffColor(pr.diff)">
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
          </label> -->
          </li>
        </ul>
      </div>

      <div class='inputbox'>
        <button @click="approvePRs(); ">
          Approve
        </button>
        <button @click="updatePRs(workItemId); ">
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
	width:750px;
  list-style: none;
}
.repobox ul li {
	width:750px;
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
  font-size: 80%;
}
.green {
  color: green;
  font-size: 80%;
}
.red {
  color: red;
  font-size: 80%;
}
.silver {
  color: silver;
  font-size: 80%;
}
.smallFont {
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
