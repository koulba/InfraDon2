<script setup lang="ts">
import { onMounted, ref } from 'vue';
import PouchDB from 'pouchdb'
 
declare interface Post {
  title: string;
  post_content: string;
  attributes: {
    creation_date: any;
  };
}
 
// Référence à la base de données
const storage = ref()
const postsData = ref<Post[]>([])
const editingId = ref<string | null>(null)
const editTitle = ref('')
const editContent = ref('')
 
// Initialisation de la base de données
const initDatabase = () => {
  console.log('=> Connexion à la base de données');
  const db = new PouchDB('http://admin:admin123@localhost:5984/infradon2_db')
  if (db) {
    console.log("Connecté à la collection : " + db?.name)
    storage.value = db;
  } else {
    console.warn('Echec lors de la connexion à la base de données')
  }
}

const localDB = new PouchDB('infradon2_local')
 
// Récupération des données
 
   const fetchData = (): any => {
  storage.value
    .allDocs({ include_docs: true })
    .then((result: any) => {
      console.log('=> Données récupérées :', result.rows)
      postsData.value = result.rows.map((row: any) => row.doc);
    })
    .catch((error: any) => {
      console.error('=> Erreur lors de la récupération des données :', error)
    })
}
 //Ajout doc
const addDocument = () => {
  storage.value.post({
    title: 'New'
  }).then(() => {
    console.log("OK");
    fetchData();
  }).catch((error:any) => {
    console.error("Erreur :", error);
  });
};

//Supp doc
const deleteDocument = (post: any) => {
  storage.value.remove(post._id, post._rev).then(() => {
    console.log("supprimé");
    fetchData();
  }).catch((error: any) => {
    console.error("Erreur lors de la suppression :", error);
  });
};

//Update
const startEditing = (post: any) => {
  editingId.value = post._id;
  editTitle.value = post.title || '';
  editContent.value = post.post_content || '';
};

//Annuler update
const cancelEditing = () => {
  editingId.value = null;
  editTitle.value = '';
  editContent.value = '';
};

//Sauvegarder 
const saveDocument = (post: any) => {
  storage.value.put({
    _id: post._id,
    _rev: post._rev,
    title: editTitle.value,
    post_content: editContent.value
  }).then(() => {
    console.log("Document mis à jour");
    cancelEditing();
    fetchData();
  }).catch((error: any) => {
    console.error("Erreur lors de la mise à jour :", error);
  });
};
 
//Erreur db locale
const initSync = () => {
  localDB.sync(storage.value, {
    live: false,
    retry: true
  })
  .on('complete', () => console.log("Synchronisation initiale réussie"))
  .on('error', (err) => console.error("Oups, erreur de synchronisation", err));
}

onMounted(() => {
  console.log('=> Composant initialisé');
  initDatabase()
  initSync()
  fetchData()
});
console.log(postsData.value)
 
</script>

<template>
  <h1>Fetch Data</h1>
  <button v-if="!isOffline" @click="isOffline = true">Passer en Offline</button>
  <button v-else @click="isOffline = false">Revenir en Online</button>
  <button @click="addDocument">Clique ici</button>
  <button @click="replicateNow">Synchroniser maintenant</button>
  <article v-for="post in postsData" v-bind:key="(post as any).id">
  <input v-model="post.title" />
  <button @click="updateDocument(post)">Sauvegarder</button>
  <button @click="deleteDocument(post)">Supprimer</button>
</article>
</template>