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
 
 
 
onMounted(() => {
  console.log('=> Composant initialisé');
  initDatabase()
  fetchData()
});
console.log(postsData.value)
 
</script>
 
<template>
  <h1>On veut de la DATA</h1>
  <button @click="addDocument">Ajoute un max de data</button>
 
  <article v-for="post in postsData" v-bind:key="(post as any).id">
    <div v-if="editingId === (post as any)._id">
      
      <input v-model="editTitle" type="text" placeholder="Titre" />
      <button @click="saveDocument(post)">Sauvegarder</button>
      <button @click="cancelEditing()">Annuler</button>
    </div>
    <div v-else>
      
      <h2>{{ post.title }}</h2>
      <p>{{ post.post_content }}</p>
      <button @click="startEditing(post)">Modifier</button>
      <button @click="deleteDocument(post)">Supprimer</button>
    </div>
  </article>
</template>