<script setup lang="ts">
import { onMounted, ref, watch } from 'vue';
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
// Données stockées
const postsData = ref<Post[]>([])

// déclaration de la DB locale
const localDB = new PouchDB('infra_don2_local')

// déclaration de la DB distante
const remoteDB = new PouchDB('http://admin:admin123@localhost:5984/infradon2_db')

// activer/désactiver le offline
const isOffline = ref(false)

// initialisation de la Database utilise isOffline pour la base
const initDatabase = () => {
  console.log('=> Connexion à la base de données');

  storage.value = isOffline.value ? localDB : remoteDB;
  console.log("Mode actif :", isOffline.value ? "Offline (localDB)" : "Online (remoteDB)");

  


 // synchronisation automatique locale <-> distante
   if (!isOffline.value) { 
    localDB.sync(remoteDB, { 
      live: false,
      retry: true
    })
      .on('complete', () => console.log("Synchronisation initiale réussie"))
      .on('error', (err) => console.error("Oups, erreur de synchronisation", err));
  } 
}
 
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
const updateDocument = (post: any) => {
  storage.value.put(post)
    .then(() => {
      console.log("Document mis à jour");
      fetchData();
    })
    .catch((error: any) => {
      console.error("Erreur lors de la mise à jour :", error);
    });
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

 //bouton manuel de réplication
const replicateNow = () => {
  localDB.replicate.to(remoteDB) 
    .on('complete', () => {
      console.log('Réplication locale -> distante réussie');
      fetchData();
    })
    .on('error', (err) => console.error("Oups, erreur de réplication", err));
};

watch(isOffline, (newVal) => {
  console.log("Mode basculé :", newVal ? "Offline" : "Online");
  storage.value = newVal ? localDB : remoteDB;

  if (!newVal) {
    // quand on repasse online, synchroniser
    localDB.sync(remoteDB, { live: false, retry: true })
      .on('complete', () => {
        console.log("Synchronisation après retour online réussie");
        fetchData();
      })
      .on('error', (err) => console.error("Erreur de sync :", err));
  } else {
    fetchData();
  }
});
 
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