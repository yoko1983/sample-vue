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
  version: string
  versionComment: string
  isDupPR: boolean
}

const debug: Ref<string> = ref<string>('');

const prs: Ref<PR[]> = ref([]);
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


const mergePRSingleWithAPI = 
  (pr: PR, workItemId: string) => {
  return new Promise(async (resolve: (value?: string) => void, reject: (reason?: any) => void) => {

    try {


      const request = {
        'status': 'completed',
        'lastMergeSourceCommit' : {
          'commitId' : pr.lastMergeSourceCommitId
        }
      };

      const response = await axios.patch(url + '/' + project  + '/_apis/git/repositories/' + pr.repoId +'/pullrequests/' + pr.id,
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
                  return status == 200;
                }
          })

          console.log('start');
          let status = '';
          for (let i = 0; i < 50; i++){
            await new Promise(s => setTimeout(s, 200));

            const responseGet = await axios.get(url + '/' + project  + '/_apis/git/repositories/' + pr.repoId +'/pullrequests/' + pr.id,
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

            status = responseGet.data.status;

            if(responseGet.data.status=='completed') {
              break;
            } 

          }
      resolve(status);
    } catch (error) {
      reject(error);
    }

  });

};

const mergePRsSingle = async (workItemId: string) => {

  for (let prId of wiPRsCheckbox.value) {

    for (let pr of prs.value) {

      if(pr.id == prId) {
        try {
          const response = 
            await mergePRSingleWithAPI(pr, workItemId);

            if(response!='completed') {
              updatePRErrorStatus.value = 'Merge may have failed.';
            }

        } catch(error) {
          updatePRErrorStatus.value = String(error);
        }
        break;
      }
    }
    await getPRsSingle(workItemId);
  }




}

const mergePRs = (workItemId: string) => {
  updatePRErrorStatus.value =  '';
  mergePRsSingle(workItemId);

}


const approvePRSingleWithAPI = (pr: PR, reviewerId: string) => {
  return new Promise(async (resolve: (value?: any) => void, reject: (reason?: any) => void) => {

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
    await getPRsSingle(workItemId.value);

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

}

const getWIReposSingleWithAPI = (workItemId: string) => {
  return new Promise(async (resolve, reject) => {
    try {

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
      let linkNum:number = 0;
      const _wiRepos:PR[] = [];
      if(workItem.relations != null) {

        for(let relation of workItem.relations) {
          const attributesName = relation.attributes.name;
          if(attributesName == 'Branch') {
            const repoUrl:string = relation.url;
            const urlBaseLen:number = "vstfs:///Git/Ref/".length;
            const decodeUrl:string = decodeURIComponent(repoUrl);
            const urlPart:string = decodeUrl.substring(urlBaseLen);
            const urlParts:string[] = urlPart.split('/');
            const comment:string = relation.attributes.comment;
            let regexp = new RegExp('(?<=\\[ver:).*?(?=\\])');
            let version:string = String(regexp.exec(comment));
            if(version=='' || version==null || version=='null') {
              version='';
            } 

            const pr:PR = {
              id:'',
              repoId:urlParts[1],
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
              lastMergeTargetCommitId:'',
              version:version,
              versionComment:'',
              isDupPR: false
            };

            _wiRepos.push(pr);
          }
          linkNum++;
        }

      }

      resolve(_wiRepos);

    } catch (error) {
      reject(error);
    }
  });

};

const getPRsSingleWithAPI = (workItemId: string, prs: PR[]) => {
  return new Promise(async (resolve, reject) => {
    try {
      getPRErrorStatus.value = '';

      const dupPRs:PR[] = [];

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
            lastMergeTargetCommitId:'',
            version:'',
            versionComment:'',
            isDupPR: true
          };

          for(let _pr of prs) {
            if(_pr.repoId == pr.repoId) {
              if(_pr.id=='') {
                _pr.id = pr.id;
              } else {
                dupPRs.push(pr);
              }
            }

          }

        }
      }

      for(let _pr of dupPRs) {
        prs.push(_pr);

      }

      resolve(prs);

    } catch (error) {
      reject(error);
    }
  });

};

const setRepoNameForReposSingleWithAPI = (prs: PR[]) => {
  return new Promise(async (resolve, reject) => {
    try {
      for(let pr of prs) {

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
      prs = prs.sort(function(a:PR, b:PR) {
        return (a.repoName < b.repoName) ? -1 : 1;
      });

      resolve(prs);
    } catch (error) {
      reject(error);
    }
  });

};

const setPRDitailsSingleWithAPI = (prs: PR[]) => {
  return new Promise(async (resolve, reject) => {
      for(let pr of prs) {
        try {

          if(pr.id == '') {
            continue;
          }

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


          resolve(prs);
        } catch (error) {
          reject(error);
          
        }

    }


  });

};



const getPRsSingle = async (workItemId: string) => {

  try {
    let _prs: PR[] = await getWIReposSingleWithAPI(workItemId) as PR[];
    _prs = await getPRsSingleWithAPI(workItemId, _prs) as PR[];
    _prs = await setPRDitailsSingleWithAPI(_prs) as PR[];
    _prs = await setRepoNameForReposSingleWithAPI(_prs) as PR[];

    prs.value = _prs;


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
          </li>
        </ul>
      </div>

      <div class='inputbox'>
        <button @click="approvePRs(); ">
          Approve
        </button>
        <button @click="mergePRs(workItemId); ">
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
