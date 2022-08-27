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
  version: string
  linkNum: number
  rev: string
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

const pjRepos: Ref<Repo[]> = ref([]);
const wiRepos: Ref<Repo[]> = ref([]);
const linkedRepos: Ref<Repo[]> = ref([]);
const gettedRepoErrorStatus: Ref<string> = ref<string>('');
const addWiReposSuccessStatus: Ref<string> = ref<string>('');
const addWiReposErrorStatus: Ref<string> = ref<string>('');
const delWiReposSuccessStatus: Ref<string> = ref<string>('');
const delWiReposErrorStatus: Ref<string> = ref<string>('');
const wiReposCheckboxRepo = ref([])
const pjReposCheckboxRepo = ref([])
const workItemId: Ref<string> = ref<string>('');
const workItemTitle: Ref<string> = ref<string>('');
const debug: Ref<string> = ref<string>('');



const delRepoSingleWithAPI = (workItemId:string, linkNum:number) => {
  return new Promise(async (resolve, reject) => {

    try {
      const response = await axios.patch(url + '/' + project  + '/_apis/wit/workitems/' + workItemId, 
          [
            { 
              'op': 'remove',
              'path': '/relations/' +linkNum
            }
          ],
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
      debug.value = String(error);
      reject(error);
    }
  });

};

// ボタンクリック時に実行するパターン
const delWIReposSingle = async (workItemId: string) => {

  let errorCount:number = 0;
  let _wiRepos = wiRepos.value;

  await getWIReposSingleWithAPI(workItemId)
    .then((repos) => {
      _wiRepos = repos;

    })
    .catch((error) => {
      errorCount++;
      delWiReposErrorStatus.value = String(error);
    });


  for (let repoId of wiReposCheckboxRepo.value) {

    for (let repo of _wiRepos) {
      if(repo.id == repoId) {

        try {
          await delRepoSingleWithAPI(workItemId, repo.linkNum); 
          await getWIReposSingleWithAPI(workItemId)
            .then((repos) => {
              _wiRepos = repos;

            })
            .catch((error) => {
              errorCount++;
              delWiReposErrorStatus.value = String(error);
            });
        } catch (error) {
          errorCount++;
          delWiReposErrorStatus.value = String(error);
        }
        break;
      }
    }
  }

  try {
    await getWIReposSingle(workItemId);
  } catch (error) {
    delWiReposErrorStatus.value = String(error);
    errorCount++;
  }

  // if(errorCount==0) {
  //   delWiReposSuccessStatus.value="Success";
  // }

}

const delWIRepos = (workItemId: string) => {
  delWiReposSuccessStatus.value = '';
  delWiReposErrorStatus.value =  '';
  delWIReposSingle(workItemId);

}



const addRepoSingleWithAPI = (projectId:string, workItemId: string, repo: Repo) => {
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


// ボタンクリック時に実行するパターン
const addWIReposSingle = async (workItemId: string) => {

  let errorCount:number = 0;

  for (let repoId of pjReposCheckboxRepo.value) {

    for (let repo of pjRepos.value) {
      if(repo.id == repoId) {

        try {
          const projectId:string = String(await getProjectIdSingleWithAPI()); 
          const response = await addRepoSingleWithAPI(projectId, workItemId, repo); 
          repo.status = 1;
        } catch (error) {
          errorCount++;
          addWiReposErrorStatus.value = String(error);
          repo.status = 2;
        }
      }
    }
  }

  try {
    getWIReposSingle(workItemId);
  } catch (error) {
    errorCount++;
    addWiReposErrorStatus.value = String(error);
  }

  // if(errorCount == 0) {
  //   addWiReposSuccessStatus.value = 'Success';
  // }
 


}

const addWIRepos = (workItemId: string) => {
  addWiReposSuccessStatus.value = '';
  addWiReposErrorStatus.value =  '';
  addWIReposSingle(workItemId);

}


const getWIReposSingleWithAPI = (workItemId: string):Promise<Repo[]> => {
  return new Promise(async (resolve, reject) => {
    try {
      gettedRepoErrorStatus.value = '';

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
      let linkNum:number = 0;
      const _wiRepos:Repo[] = [];
      if(workItem.relations != null) {

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
              status:0,
              version:'',
              linkNum:linkNum,
              rev:workItem.rev
            };

            _wiRepos.push(repo);
          }
          linkNum++;
        }

        if(_wiRepos.length != 0){
          await getWIRepoNameForReposSingleWithAPI(_wiRepos)
            .then((repos) => {
              wiRepos.value = repos;
            })
            .catch((error) => {
              reject(error);
            });
        }


      }

      resolve(_wiRepos);

    } catch (error) {
      reject(error);
    }
  });

};

const getWIRepoNameForReposSingleWithAPI = (repos:Repo[]):Promise<Repo[]> => {
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
      let sortedRepos = repos.sort(function(a:Repo, b:Repo) {
        return (a.name < b.name) ? -1 : 1;
      });

      resolve(sortedRepos);
    } catch (error) {
      reject(error);
    }
  });

};

const getPjReposSingleWithAPI = () => {
  return new Promise(async (resolve, reject) => {
    try {
      gettedRepoErrorStatus.value = '';

      const response = await axios.get(url + '/' + project  + '/_apis/git/repositories', 
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

      const resPjRepos:Repo[] = response.data.value;


      //WIにリンク付けされているレポジトリを除外する
      const _pjRepos:Repo[]  = [];
      for(let resPjRepo of resPjRepos) {
        let linkedCount = 0;
        for(let wiRepo of wiRepos.value) {
          if(resPjRepo.id == wiRepo.id) {
            linkedCount++;
            
          }
        }
        if(linkedCount==0) {
          const pjRepo:Repo = {
            id:resPjRepo.id,
            name:resPjRepo.name,
            diff:0,
            status:0,
            version:'',
            linkNum:0,
            rev:''
          };

          _pjRepos.push(pjRepo);

        }
      }

      //リポジトリ一覧を昇順ソートする
      let sortedRepos = _pjRepos.sort(function(a:Repo, b:Repo) {
        return (a.name < b.name) ? -1 : 1;
      });
      pjRepos.value = sortedRepos;

      resolve(pjRepos.value);

    } catch (error) {
      reject(error);
    }
  });

};

const getWIReposSingle = async (workItemId: string) => {

  try {
    await getWIReposSingleWithAPI(workItemId)
      .then((repos) => {
        wiRepos.value = repos;
      })
      .catch((error) => {
        gettedRepoErrorStatus.value = String(error);
      });;
    await getPjReposSingleWithAPI();

  } catch(error) {
      gettedRepoErrorStatus.value = String(error);

  }


}


const getWIRepos = (workItemId: string) => {
  gettedRepoErrorStatus.value = '';
  getWIReposSingle(workItemId);
}




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
      <Title title="Edit Repositories!" description="You can edit repositories to work item."/>
    </header>
    <aside>
      <div class='labelbox'>
        <label>WorkItem ID:</label>
      </div>
      <div class='smallInputBox'>
        <input type="text" v-model="workItemId">
        <button @click="getWIRepos(workItemId)">
          Display
        </button>
      </div>

      <div class='labelbox'>
        <label>[ {{workItemId}} ] {{workItemTitle}} :</label>
        {{debug}}
      </div>

      <div v-for="repo in wiRepos" class='repoBox'>
        <ul>
          <li>
            <input
              :id="'repo.id'"
              type="checkbox"
              :value="repo.id"
              v-model="wiReposCheckboxRepo"
              :checked=false
            />
            <div class="repoNameBox">
              <label :for="'repo.name'">{{repo.name}}</label>
            </div>
            <div class="versionBox">
              <input type="text" v-model="repo.version"/>
            </div>
          </li>
        </ul>
      </div>
      <div class='inputbox'>
        <button @click="delWIRepos(workItemId)">
          Delete
        </button>
        <button @click="linkWorkItem(workItemId)">
          Edit
        </button>
        <div class='statusbox'>{{delWiReposSuccessStatus}}</div>
        <div class='errorStatusbox'>{{delWiReposErrorStatus}}</div>
      </div>



    </aside>
    <article>

      <div class='labelbox'>
        <label>Project's Repositories </label>
      </div>
      <div class="repo2ColBox">
        <ul>
          <li v-for="repo in pjRepos">
          <div class="yoko" >
            <input
            :id="'repo.id'"
            type="checkbox"
            :value="repo.id"
            v-model="pjReposCheckboxRepo"
            :checked=false
            />
            <label>{{repo.name}}</label>
          </div>
          </li>
        </ul>
      </div>

      <div class='inputbox'>
        <button @click="addWIRepos(workItemId)">
          Add
        </button>
        <div class='statusbox'>{{addWiReposSuccessStatus}}</div>
        <div class='errorStatusbox'>{{addWiReposErrorStatus}}</div>
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
.smallInputBox {
	width:500px;
	padding:5px;
}
.smallInputBox input {
  margin-left: 1em;
	width:200px;
}

.repo2ColBox {
	padding:0px;
  width:720px; 
}
.repo2ColBox ul {
  column-count: 3;
  list-style: none;

}
.repo2ColBox ul li {
	margin-bottom:5px;
	padding-left:1em;
	text-indent:-1em;
	line-height:1.4;
  
}


.repoBox {
	padding:0px;
}
.repoBox ul {
	width:500px;
  list-style: none;
}
.repoBox ul li {
	margin-bottom:5px;
	padding-left:1em;
	text-indent:-1em;
	line-height:1.4;
  
}

.repoNameBox {
  display: inline-block; 
	width:240px;
	padding-left:1em;
}

.repoNameBox label {
	padding-left:1em;
}


.versionBox {
  display: inline-block; 
  text-align:right
}

.versionBox input {
	width:80px;
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
    grid-gap: 100px;
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
