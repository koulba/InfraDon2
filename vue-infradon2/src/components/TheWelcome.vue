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
  image?: string;
}

declare interface User {
  _id?: string;
  _rev?: string;
  username: string;
  email: string;
  created_at: string;
  post_count?: number;
}

// Référence aux bases de données
const storage = ref()
const usersStorage = ref()
// Données stockées
const postsData = ref<Post[]>([])
const usersData = ref<User[]>([])

// déclaration de la DB locale pour les posts
const localDB = new PouchDB('infra_don2_local')

// déclaration de la DB distante pour les posts
const remoteDB = new PouchDB('http://admin:admin123@localhost:5984/infradon2_db')

// déclaration de la DB locale pour les users
const localUsersDB = new PouchDB('infra_don2_users_local')

// déclaration de la DB distante pour les users
const remoteUsersDB = new PouchDB('http://admin:admin123@localhost:5984/infradon2_users_db')

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

// Variable pour gérer l'affichage des commentaires (clé: postId, valeur: boolean pour voir plus)
const showAllComments = ref<Record<string, boolean>>({})

// initialisation de la Database utilise isOffline pour la base
const initDatabase = () => {
  console.log('=> Connexion aux bases de données');

  // Base de données des posts
  storage.value = isOffline.value ? localDB : remoteDB;
  console.log("Mode actif Posts:", isOffline.value ? "Offline (localDB)" : "Online (remoteDB)");

  // Base de données des users
  usersStorage.value = isOffline.value ? localUsersDB : remoteUsersDB;
  console.log("Mode actif Users:", isOffline.value ? "Offline (localUsersDB)" : "Online (remoteUsersDB)");

  // synchronisation automatique locale <-> distante pour les posts
  if (!isOffline.value) {
    localDB.sync(remoteDB, {
      live: false,
      retry: true
    })
      .on('complete', () => console.log("Synchronisation posts réussie"))
      .on('error', (err) => console.error("Erreur de synchronisation posts", err));

    // synchronisation automatique locale <-> distante pour les users
    localUsersDB.sync(remoteUsersDB, {
      live: false,
      retry: true
    })
      .on('complete', () => console.log("Synchronisation users réussie"))
      .on('error', (err) => console.error("Erreur de synchronisation users", err));
  }
}

// créa index sur l'attribut 'category', 'likes' pour posts et 'username' pour users
const createIndex = async () => {
  try {
    // Index pour les posts
    await storage.value.createIndex({
      index: { fields: ['category'] }
    });
    console.log('=> Index créé sur le champ "category" (posts)');

    await storage.value.createIndex({
      index: { fields: ['likes'] }
    });
    console.log('=> Index créé sur le champ "likes" (posts)');

    await storage.value.createIndex({
      index: { fields: ['category', 'likes'] }
    });
    console.log('=> Index créé sur les champs "category" et "likes" (posts)');

    // Index pour les users
    await usersStorage.value.createIndex({
      index: { fields: ['username'] }
    });
    console.log('=> Index créé sur le champ "username" (users)');

    await usersStorage.value.createIndex({
      index: { fields: ['email'] }
    });
    console.log('=> Index créé sur le champ "email" (users)');
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
        },
        likes: Math.floor(Math.random() * 100),
        comments: []
      });
    }

    await storage.value.bulkDocs(docs);
    console.log(`=> ${numberOfDocs.value} documents générés avec succès`);
    fetchData();
  } catch (error) {
    console.error('=> Erreur lors de la génération des documents :', error);
  }
}

// Recherche avec filtre sur l'attribut indexé + tri via DB
const searchByCategory = async () => {
  try {
    const selector: any = {};

    // Filtrer par catégorie si spécifié
    if (searchCategory.value) {
      selector.category = searchCategory.value;
    }

    // Construire la requête avec tri si nécessaire
    const query: any = {
      selector: selector,
      fields: ['_id', '_rev', 'title', 'post_content', 'category', 'author', 'attributes', 'likes', 'comments']
    };

    // Ajouter le tri via la DB si activé
    if (sortBy.value === 'likes') {
      query.sort = [{ likes: sortOrder.value === 'asc' ? 'asc' : 'desc' }];
    }

    const result = await storage.value.find(query);
    filteredPosts.value = result.docs;

    const categoryMsg = searchCategory.value ? ` pour la catégorie "${searchCategory.value}"` : '';
    const sortMsg = sortBy.value ? ` (trié par ${sortBy.value} ${sortOrder.value})` : '';
    console.log(`=> ${result.docs.length} documents trouvés${categoryMsg}${sortMsg}`);
  } catch (error) {
    console.error('=> Erreur lors de la recherche :', error);
    // Fallback sur postsData si erreur
    filteredPosts.value = postsData.value;
  }
}
 
// Récupération des données posts
const fetchData = (): any => {
  storage.value
    .allDocs({ include_docs: true })
    .then((result: any) => {
      console.log('=> Posts récupérés :', result.rows.length)
      postsData.value = result.rows.map((row: any) => row.doc);
    })
    .catch((error: any) => {
      console.error('=> Erreur lors de la récupération des posts :', error)
    })
}

// Récupération des données users
const fetchUsers = (): any => {
  usersStorage.value
    .allDocs({ include_docs: true })
    .then((result: any) => {
      console.log('=> Users récupérés :', result.rows.length)
      usersData.value = result.rows.map((row: any) => row.doc);
    })
    .catch((error: any) => {
      console.error('=> Erreur lors de la récupération des users :', error)
    })
}

// Générer des users de test
const generateTestUsers = async () => {
  if (!usersStorage.value) {
    console.error('=> Users storage not initialized');
    return;
  }

  const usernames = ['Alice', 'Bob', 'Charlie', 'Diana', 'Eve', 'Frank', 'Grace', 'Henry'];

  try {
    const docs = [];
    for (let i = 0; i < usernames.length; i++) {
      const username = usernames[i];
      if (!username) continue;

      docs.push({
        username: username,
        email: `${username.toLowerCase()}@example.com`,
        created_at: new Date().toISOString(),
        post_count: Math.floor(Math.random() * 20)
      });
    }

    await usersStorage.value.bulkDocs(docs);
    console.log(`=> ${docs.length} users générés avec succès`);
    fetchUsers();
  } catch (error) {
    console.error('=> Erreur lors de la génération des users :', error);
  }
}

// Ajouter un user
const addUser = () => {
  const randomName = `User_${Math.floor(Math.random() * 1000)}`;
  usersStorage.value.post({
    username: randomName,
    email: `${randomName.toLowerCase()}@example.com`,
    created_at: new Date().toISOString(),
    post_count: 0
  }).then(() => {
    console.log("User ajouté");
    fetchUsers();
  }).catch((error: any) => {
    console.error("Erreur :", error);
  });
}

// Supprimer un user
const deleteUser = (user: any) => {
  usersStorage.value.remove(user._id, user._rev).then(() => {
    console.log("User supprimé");
    fetchUsers();
  }).catch((error: any) => {
    console.error("Erreur lors de la suppression :", error);
  });
}

// Mettre à jour un user
const updateUser = (user: any) => {
  usersStorage.value.put(user)
    .then(() => {
      console.log("User mis à jour");
      fetchUsers();
    })
    .catch((error: any) => {
      console.error("Erreur lors de la mise à jour :", error);
    });
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
  usersStorage.value = newVal ? localUsersDB : remoteUsersDB;

  if (!newVal) {
    // quand on repasse online, synchroniser les posts
    localDB.sync(remoteDB, { live: false, retry: true })
      .on('complete', () => {
        console.log("Synchronisation posts après retour online réussie");
        fetchData();
      })
      .on('error', (err) => console.error("Erreur de sync posts :", err));

    // synchroniser les users
    localUsersDB.sync(remoteUsersDB, { live: false, retry: true })
      .on('complete', () => {
        console.log("Synchronisation users après retour online réussie");
        fetchUsers();
      })
      .on('error', (err) => console.error("Erreur de sync users :", err));
  } else {
    fetchData();
    fetchUsers();
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
  fetchUsers()
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

// Fonction pour ajouter une image à un post
const addImage = (post: Post, event: Event) => {
  const input = event.target as HTMLInputElement;
  const file = input.files?.[0];

  if (file && file.type.startsWith('image/')) {
    const reader = new FileReader();
    reader.onload = (e) => {
      post.image = e.target?.result as string;
      updateDocument(post);
    };
    reader.readAsDataURL(file);
  }
};

// supp image post
const removeImage = (post: Post) => {
  post.image = undefined;
  updateDocument(post);
};

//affichage commentaires
const toggleShowComments = (postId: string) => {
  showAllComments.value[postId] = !showAllComments.value[postId];
};

const getDisplayedComments = (post: Post): Comment[] => {
  if (!post.comments || post.comments.length === 0) return [];
  const postId = (post as any)._id;

  if (showAllComments.value[postId]) {
    return post.comments;
  } else {
    const lastComment = post.comments[post.comments.length - 1];
    return lastComment ? [lastComment] : [];
  }
};

// Fonctions de tri
const setSortBy = (type: 'likes' | 'comments') => {
  if (sortBy.value === type) {
    sortOrder.value = sortOrder.value === 'asc' ? 'desc' : 'asc';
  } else {
    sortBy.value = type;
    sortOrder.value = 'desc';
  }
  searchByCategory();
};

const clearSort = () => {
  sortBy.value = null;
  // Relancer la recherche sans tri
  searchByCategory();
};

// Tri par commentaires en JS (car nombre de commentaires non indexable facilement)
const sortedPosts = computed(() => {
  if (sortBy.value === 'comments') {
    const posts = [...filteredPosts.value];
    return posts.sort((a, b) => {
      const valueA = a.comments?.length || 0;
      const valueB = b.comments?.length || 0;
      return sortOrder.value === 'asc' ? valueA - valueB : valueB - valueA;
    });
  }
  return filteredPosts.value;
});


</script>

<template>
  <div class="container">
    <div class="header-with-status">
      <h1>Gestion de Documents NoSQL</h1>
      <div class="status-indicator" :class="{ offline: isOffline, online: !isOffline }">
        <span class="status-dot"></span>
        <span class="status-text">{{ isOffline ? 'Mode Offline' : 'Mode Online' }}</span>
      </div>
    </div>


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
      <h2>Trier</h2>
      <div class="sort-controls">
        <button
          @click="setSortBy('likes')"
          :class="{ active: sortBy === 'likes' }"
        >
          Trier par likes {{ sortBy === 'likes' ? (sortOrder === 'asc' ? '↑' : '↓') : '' }}
        </button>
        <button
          @click="setSortBy('comments')"
          :class="{ active: sortBy === 'comments' }"
        >
          Trier par commentaires {{ sortBy === 'comments' ? (sortOrder === 'asc' ? '↑' : '↓') : '' }}
        </button>
        <button v-if="sortBy" @click="clearSort" class="btn-clear">Réinitialiser</button>
      </div>
    </div>

    <div class="users-section">
      <h2>Collection utilisateurs ({{ usersData.length }})</h2>
      <div class="users-controls">
        <button @click="generateTestUsers">Générer des users</button>
        <button @click="addUser">Ajouter un user</button>
      </div>
      <div class="users-list">
        <div v-for="user in usersData" :key="user._id" class="user-card">
          <div class="user-info">
            <input v-model="user.username" placeholder="Username" class="user-input" />
            <input v-model="user.email" placeholder="Email" class="user-input" />
            <span class="user-meta">Posts: {{ user.post_count || 0 }}</span>
          </div>
          <div class="user-actions">
            <button @click="updateUser(user)" class="btn-save">Sauvegarder</button>
            <button @click="deleteUser(user)" class="btn-delete">Supprimer</button>
          </div>
        </div>
      </div>
    </div>

    <div class="documents-list">
      <h2>Documents ({{ sortedPosts.length }})</h2>
      <article v-for="post in sortedPosts" v-bind:key="(post as any)._id" class="document-card">
        <div class="document-header">
          <input v-model="post.title" class="title-input" />
          <span v-if="post.category" class="category-badge">{{ post.category }}</span>
        </div>

        <div class="image-section">
          <div v-if="post.image" class="image-container">
            <img :src="post.image" alt="Image du post" class="post-image" />
            <button @click="removeImage(post)" class="btn-remove-image">Supprimer l'image</button>
          </div>
          <div v-else class="image-upload">
            <label :for="'image-upload-' + (post as any)._id" class="btn-add-image">
              Ajouter une image
            </label>
            <input
              :id="'image-upload-' + (post as any)._id"
              type="file"
              accept="image/*"
              @change="addImage(post, $event)"
              class="file-input"
            />
          </div>
        </div>

        <div class="document-actions">
          <button @click="updateDocument(post)" class="btn-save">Sauvegarder</button>
          <button @click="deleteDocument(post)" class="btn-delete">Supprimer</button>
          <button @click="toggleLike(post)">Like ({{ post.likes || 0 }})</button>

     <div class="comments-section">
      <h4>Commentaires ({{ post.comments?.length || 0 }})</h4>

      <div v-if="post.comments?.length" class="comments-container">
        <ul class="comments-list">
          <li v-for="comment in getDisplayedComments(post)" :key="comment.id" class="comment-item">
            <div class="comment-content">
              <span class="comment-text">{{ comment.content }}</span>
              <span class="comment-date">{{ new Date(comment.created_at).toLocaleString('fr-FR', { dateStyle: 'short', timeStyle: 'short' }) }}</span>
            </div>
            <button @click="deleteComment(post, comment.id)" class="btn-delete-comment">X</button>
          </li>
        </ul>

        <button
          v-if="post.comments.length > 1"
          @click="toggleShowComments((post as any)._id)"
          class="btn-toggle-comments"
        >
          {{ showAllComments[(post as any)._id] ? 'Voir moins' : `Voir plus (${post.comments.length - 1} autres)` }}
        </button>
      </div>

      <div class="add-comment-section">
        <input
          :id="'comment-input-' + (post as any)._id"
          placeholder="Ajouter un commentaire"
          class="comment-input"
        />
        <button
          @click="
            addComment(post, (($event.target as HTMLElement).previousElementSibling as HTMLInputElement)?.value || '');
            ((($event.target as HTMLElement).previousElementSibling as HTMLInputElement).value = '');
          "
          class="btn-add-comment"
        >Ajouter</button>
      </div>
    </div>
        </div>
      </article>
    </div>
  </div>
</template>