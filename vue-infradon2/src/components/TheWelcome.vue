<script setup lang="ts">
import { onMounted, ref, watch, computed } from 'vue';
import PouchDB from 'pouchdb'
import PouchDBFind from 'pouchdb-find'

// Activation du plugin pouchdb-find pour les recherches indexées
PouchDB.plugin(PouchDBFind)

declare interface Post {
  title: string;
  post_content: string;
  category?: string;  // indexation er recherche
  author?: string;
  attributes: {
    creation_date: any;
  };
}

declare interface Comment {
  id: string;            
  content: string;       
  created_at: string;     
}

declare interface Post {
  _id?: string;          
  _rev?: string;          
  title: string;          
  post_content: string;   
  attributes: {
    creation_date: any; 
  };
  likes: number;          
  comments: Comment[];  
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

// Variable pour la recherche
const searchCategory = ref('')
const filteredPosts = ref<Post[]>([])

// Variable pour le nombre de documents à générer
const numberOfDocs = ref(10)

// Variables pour le tri
const sortBy = ref<'likes' | 'comments' | null>(null)
const sortOrder = ref<'asc' | 'desc'>('desc')

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

// créa index sur l'attribut 'category'
const createIndex = async () => {
  try {
    await storage.value.createIndex({
      index: { fields: ['category'] }
    });
    console.log('=> Index créé sur le champ "category"');
  } catch (error) {
    console.error('=> Erreur lors de la création de l\'index :', error);
  }
}

// générer des documents de test
const generateTestDocuments = async () => {
  const categories = ['Technology', 'Science', 'Sport', 'Culture', 'Education', 'Health', 'Finance', 'Travel'];
  const titlePrefixes = ['Introduction à', 'Guide complet de', 'Découverte de', 'Analyse de', 'Étude sur', 'Rapport sur'];

  try {
    const docs = [];
    for (let i = 0; i < numberOfDocs.value; i++) {
      const category = categories[Math.floor(Math.random() * categories.length)];
      const titlePrefix = titlePrefixes[Math.floor(Math.random() * titlePrefixes.length)];

      docs.push({
        title: `${titlePrefix} ${category} #${i + 1}`,
        post_content: `Contenu généré automatiquement pour le document #${i + 1} dans la catégorie ${category}. Ceci est un texte de test créé par la factory.`,
        category: category,
        attributes: {
          creation_date: new Date().toISOString()
        }
      });
    }

    await storage.value.bulkDocs(docs);
    console.log(`=> ${numberOfDocs.value} documents générés avec succès`);
    fetchData();
  } catch (error) {
    console.error('=> Erreur lors de la génération des documents :', error);
  }
}

// Recherche avec filtre sur l'attribut indexé
const searchByCategory = async () => {
  if (!searchCategory.value) {
    filteredPosts.value = postsData.value;
    return;
  }

  try {
    const result = await storage.value.find({
      selector: {
        category: searchCategory.value
      },
      fields: ['_id', '_rev', 'title', 'post_content', 'category', 'author', 'attributes']
    });

    filteredPosts.value = result.docs;
    console.log(`=> ${result.docs.length} documents trouvés pour la catégorie "${searchCategory.value}"`);
  } catch (error) {
    console.error('=> Erreur lors de la recherche :', error);
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

onMounted(async () => {
  console.log('=> Composant initialisé');
  initDatabase()
  initSync()
  await createIndex() 
  fetchData()
});

// Watch pour mettre à jour la recherche en temps réel
watch(searchCategory, () => {
  searchByCategory();
});

watch(postsData, () => {
  if (!searchCategory.value) {
    filteredPosts.value = postsData.value;
  }
}, { immediate: true });

console.log(postsData.value)

//likes et commentaires
const toggleLike = (post: Post) => {
  post.likes = (post.likes || 0) + 1;
  updateDocument(post);
};

const addComment = (post: Post, newContent: string) => {
  if (!newContent.trim()) return;
  const newComment: Comment = {
    id: `comment_${Date.now()}_${Math.random()}`,
    content: newContent,
    created_at: new Date().toISOString()
  };
  post.comments = [...(post.comments || []), newComment];
  updateDocument(post);
};

const deleteComment = (post: Post, commentId: string) => {
  post.comments = (post.comments || []).filter(c => c.id !== commentId);
  updateDocument(post);
};

// Fonctions de tri
const setSortBy = (type: 'likes' | 'comments') => {
  if (sortBy.value === type) {
    sortOrder.value = sortOrder.value === 'asc' ? 'desc' : 'asc';
  } else {
    sortBy.value = type;
    sortOrder.value = 'desc';
  }
};

const clearSort = () => {
  sortBy.value = null;
};

// Computed property pour les posts triés
const sortedPosts = computed(() => {
  const posts = [...filteredPosts.value];

  if (!sortBy.value) {
    return posts;
  }

  return posts.sort((a, b) => {
    let valueA = 0;
    let valueB = 0;

    if (sortBy.value === 'likes') {
      valueA = a.likes || 0;
      valueB = b.likes || 0;
    } else if (sortBy.value === 'comments') {
      valueA = a.comments?.length || 0;
      valueB = b.comments?.length || 0;
    }

    if (sortOrder.value === 'asc') {
      return valueA - valueB;
    } else {
      return valueB - valueA;
    }
  });
});


</script>

<template>
  <div class="container">
    <h1>Gestion de Documents NoSQL</h1>

    
    <div class="controls">
      <button v-if="!isOffline" @click="isOffline = true">Passer en Offline</button>
      <button v-else @click="isOffline = false">Revenir en Online</button>
      <button @click="addDocument">Ajouter un document</button>
      <button @click="replicateNow">Synchroniser</button>
    </div>

    <div class="factory-section">
      <h2>Factory - Génération de données de test</h2>
      <div class="factory-controls">
        <label>
          Nombre de documents :
          <input type="number" v-model.number="numberOfDocs" min="1" max="1000" />
        </label>
        <button @click="generateTestDocuments">Générer des documents</button>
      </div>
    </div>

    <div class="search-section">
      <h2>Recherche</h2>
      <div class="search-controls">
        <label>
          Rechercher par catégorie :
          <select v-model="searchCategory">
            <option value="">-- Toutes les catégories --</option>
            <option value="Technology">Technology</option>
            <option value="Science">Science</option>
            <option value="Sport">Sport</option>
            <option value="Culture">Culture</option>
            <option value="Education">Education</option>
            <option value="Health">Health</option>
            <option value="Finance">Finance</option>
            <option value="Travel">Travel</option>
          </select>
        </label>
        <span class="result-count">{{ filteredPosts.length }} résultat(s)</span>
      </div>
    </div>

    <div class="sort-section">
      <h2>Tri</h2>
      <div class="sort-controls">
        <button
          @click="setSortBy('likes')"
          :class="{ active: sortBy === 'likes' }"
        >
          Trier par Likes {{ sortBy === 'likes' ? (sortOrder === 'asc' ? '↑' : '↓') : '' }}
        </button>
        <button
          @click="setSortBy('comments')"
          :class="{ active: sortBy === 'comments' }"
        >
          Trier par Commentaires {{ sortBy === 'comments' ? (sortOrder === 'asc' ? '↑' : '↓') : '' }}
        </button>
        <button v-if="sortBy" @click="clearSort" class="btn-clear">Réinitialiser</button>
      </div>
    </div>
    <div class="documents-list">
      <h2>Documents ({{ sortedPosts.length }})</h2>
      <article v-for="post in sortedPosts" v-bind:key="(post as any)._id" class="document-card">
        <div class="document-header">
          <input v-model="post.title" class="title-input" />
          <span v-if="post.category" class="category-badge">{{ post.category }}</span>
        </div>
        <div class="document-actions">
          <button @click="updateDocument(post)" class="btn-save">Sauvegarder</button>
          <button @click="deleteDocument(post)" class="btn-delete">Supprimer</button>
          <button @click="toggleLike(post)">Like ({{ post.likes || 0 }})</button>

     <div>
      <h4>Commentaires ({{ post.comments?.length || 0 }})</h4>
      <ul v-if="post.comments?.length">
        <li v-for="comment in post.comments" :key="comment.id">
          {{ comment.content }}
          <button @click="deleteComment(post, comment.id)">X</button>
        </li>
      </ul>

      <div>
        <input :id="'comment-input-' + (post as any)._id" placeholder="Ajouter un commentaire" />
        <button
          @click="
            addComment(post, (($event.target as HTMLElement).previousElementSibling as HTMLInputElement)?.value || '');
            ((($event.target as HTMLElement).previousElementSibling as HTMLInputElement).value = '');
          "
        >Ajouter</button>
      </div>
    </div>
        </div>
      </article>
    </div>
  </div>
</template>